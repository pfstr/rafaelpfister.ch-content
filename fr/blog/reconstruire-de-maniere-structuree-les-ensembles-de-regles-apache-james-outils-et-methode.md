---
title: "Reconstruire de manière structurée les ensembles de règles Apache James : outils et méthode"
navTitle: "Reconstruire l’ensemble de règles"
description: "Après des années, les ensembles de règles de mailets accumulés contiennent des chemins morts que plus personne ne reconnaît. Comment analyser l’ensemble de règles comme un graphe, trouver de manière fiable le code inaccessible et organiser la refonte pour qu’un seul mailet maintienne le retour en arrière possible."
date: "2026-08-11"
kategorie: "Totemomail"
timeToRead: "16 min de lecture"
themen:
  - totemomail
  - e-mail-verschluesselung
  - smtp-mailflow
hauptthema: "totemomail"
produkte:
  - "totemomail"
  - "apache-james"
protokolle:
  - "smtp"
  - "troubleshooting"
  - "verschluesselung"
related:
  - totemomail-m365
  - totemomail-licensed-user-limit-ldap-cleanup
  - mailflow-fehlersuche-kontrollierte-experimente
slug: "reconstruire-de-maniere-structuree-les-ensembles-de-regles-apache-james-outils-et-methode"
translationId: "article-b9c98459a0ff6352"
draft: false
translationOf: apache-james-ruleset-strukturiert-neu-aufbauen
url: https://rafaelpfister.ch/fr/blog/reconstruire-de-maniere-structuree-les-ensembles-de-regles-apache-james-outils-et-methode
translationSourceHash: b0274af954ad40614bc74b37b7be1e6e9bee6c856e28105336eddfb967895884
translationModel: gpt-5.6-terra
translatedAt: 2026-08-12T05:09:20.461Z
translationReview: automatic
---

# Reconstruire de manière structurée les ensembles de règles Apache James : outils et méthode

Les passerelles de messagerie basées sur Apache James, dont Totemomail, pilotent l’ensemble de leur flux de messages au moyen d’un ensemble de règles XML. Après quelques années d’exploitation, cet ensemble de règles présente une caractéristique que presque personne ne remarque : une part importante n’est jamais exécutée. Des règles ont été ajoutées, des aiguillages ont été placés avant elles, des branches n’ont plus mené nulle part et, comme rien ne cassait, tout est resté en place.

Le problème n’est pas l’espace disque. C’est que plus personne ne peut dire quelle règle s’applique réellement. Quiconque prévoit une modification lit un fichier contenant des centaines de mailets sans savoir lesquels sont encore pertinents. Or, cette question peut recevoir une réponse mécanique.

Cet article décrit la méthode et les outils nécessaires : analyser l’ensemble de règles comme un graphe orienté, trouver de manière fiable le code inaccessible et concevoir la refonte de sorte qu’un seul mailet maintienne le retour en arrière possible.

## Le modèle en quatre phrases

Un ensemble de règles se compose de **processeurs**, c’est-à-dire de chaînes nommées. Chaque chaîne contient des **mailets** qui exécutent une action, et chaque mailet possède un **matcher** qui détermine s’il correspond au message actuel. Un mailet de la classe `ToProcessor` transmet le message à une autre chaîne.

Le point d’entrée s’appelle généralement `root`. Tout le reste se ramifie à partir de là.

```xml
<processor name="root">
   <mailet class="ToProcessor" match="RecipientIs?Recipient(s)=journal@example.com">
      <processor>dropJournal</processor>
   </mailet>
   <mailet class="ToProcessor" match="HostIsLocal">
      <processor>incoming</processor>
   </mailet>
   <mailet class="ToProcessor" match="All">
      <processor>outgoing</processor>
   </mailet>
</processor>
```

La structure est donc un graphe orienté : les processeurs sont les nœuds, les destinations de `ToProcessor` sont les arêtes. Et dès lors que vous le voyez ainsi, la question du code mort devient une tâche classique : une analyse d’accessibilité.

## Deux types de code mort

Avant de mesurer, vous devez savoir ce que vous recherchez. Il en existe deux formes, et la seconde est la plus insidieuse.

**Processeurs inaccessibles.** Des chaînes entières vers lesquelles aucun `ToProcessor` ne pointe plus. Elles figurent dans le fichier, mais ne sont jamais empruntées. C’est le cas évident.

**Reste mort au sein d’une chaîne.** Un `ToProcessor` avec `match="All"` correspond à **chaque** message et le transmet. Tout ce qui se trouve ensuite dans la même chaîne n’est jamais atteint. Il en va de même pour les mailets avec `passThrough=false`: ils consomment le message et prennent eux-mêmes en charge le traitement ultérieur ; les mailets suivants ne le voient plus.

Cette seconde forme ne peut pas être trouvée par une simple recherche textuelle, car les lignes paraissent tout à fait normales. Vous avez besoin de l’ordre au sein de la chaîne.

## Outil 1 : extraire le graphe

Le point de départ est une analyse qui extrait les processeurs et leurs destinations. Le script suivant n’utilise que la bibliothèque standard et fonctionne avec toute installation Python :

```python
import re
import xml.dom.minidom

PFAD = "regelwerk.xml"

daten = open(PFAD, "rb").read().decode("utf-8").replace("\r\n", "\n")

# Découper les processeurs et leurs blocs
starts = [(m.start(), m.group(1))
          for m in re.finditer(r'<processor name="([^"]+)">', daten)]

bloecke = {}
for i, (pos, name) in enumerate(starts):
    ende = starts[i + 1][0] if i + 1 < len(starts) else daten.find("</spoolmanager>")
    bloecke[name] = daten[pos:ende]

def ziele(block):
    """Alle ToProcessor-Ziele einer Kette."""
    return re.findall(r"<processor>\s*([^<>]+?)\s*</processor>", block)

for name, block in bloecke.items():
    print(f"{name} -> {', '.join(ziele(block)) or '(kein Ziel)'}")
```

Notez la différence entre la **balise de définition** `<processor name="...">` et la **balise de destination** `<processor>name</processor>` au sein d’un mailet `ToProcessor`. Elles portent le même nom, mais ne désignent pas la même chose. Les confondre produit des résultats absurdes. C’est précisément sur cela que repose le piège décrit plus loin.

## Outil 2 : accessibilité depuis le point d’entrée

Avec le graphe, l’analyse consiste en un parcours en largeur à partir de `root`. Tout ce qui n’est pas visité est mort :

```python
erreichbar = set()
stapel = ["root"]

while stapel:
    knoten = stapel.pop()
    if knoten in erreichbar:
        continue
    erreichbar.add(knoten)
    for ziel in ziele(bloecke.get(knoten, "")):
        if ziel not in erreichbar:
            stapel.append(ziel)

def anzahl_mailets(block):
    return len(re.findall(r"<mailet ", block))

tot = [n for n in bloecke if n not in erreichbar]

print(f"Prozessoren gesamt: {len(bloecke)}")
print(f"Erreichbar:         {len(erreichbar)}")
print(f"Tot:                {len(tot)}")

for name in tot:
    print(f"  - {name} ({anzahl_mailets(bloecke[name])} Mailets)")
```

Voici une sortie typique pour un ensemble de règles ayant évolué au fil du temps :

```text
Prozessoren gesamt: 38
Erreichbar:         18
Tot:                20
  - addExtSender (7 Mailets)
  - decrypt (6 Mailets)
  - externalDelivery (14 Mailets)
  - outgoingProcessExceptionTriggers (12 Mailets)
  ...
```

Vingt processeurs sur 38, avec plus de 160 mailets au total, ne sont jamais exécutés. Ce n’est pas une anomalie, mais le cas normal dans un environnement ayant connu plusieurs refontes.

## Outil 3 : trouver le reste mort au sein des chaînes

Passons maintenant à la seconde forme. Parcourez chaque chaîne accessible, mailet par mailet, et marquez tout ce qui suit le premier départ inconditionnel :

```python
def toter_rest(block):
    """Index des ersten Mailets, ab dem nichts mehr erreicht wird."""
    mailets = re.findall(r"<mailet\b.*?(?:/>|</mailet>)", block, re.S)
    for i, m in enumerate(mailets):
        ist_all = 'match="All"' in m
        ist_weiche = 'class="ToProcessor"' in m
        konsumiert = "<passThrough>false</passThrough>" in m
        if ist_all and (ist_weiche or konsumiert):
            return i + 1, len(mailets)
    return None, len(mailets)

for name in sorted(erreichbar):
    ab, gesamt = toter_rest(bloecke[name])
    if ab is not None and ab < gesamt:
        print(f"{name}: Mailets {ab + 1} bis {gesamt} werden nie erreicht")
```

Ce constat est plus précieux que la liste des processeurs, car il se situe au cœur de chaînes actives. Quiconque ajoute une règle et l’insère sous un `ToProcessor match="All"` écrit une règle qui ne s’appliquera jamais, puis s’étonne de son inefficacité.

## Outil 4 : contrôle structurel

Un XML bien formé ne fait que la moitié du travail. Ces quatre contrôles détectent les erreurs qu’un analyseur XML laisse passer, mais pas la passerelle :

```python
dom = xml.dom.minidom.parseString(daten.replace("\n", "\r\n").encode("utf-8"))

namen = [p.getAttribute("name")
         for p in dom.getElementsByTagName("processor")
         if p.getAttribute("name")]

# 1. Noms de processeurs en double
doppelt = {n for n in namen if namen.count(n) > 1}

# 2. Chaque Mailet doit être un enfant direct d’un processeur et avoir class + match
fehler = []
for m in dom.getElementsByTagName("mailet"):
    if m.parentNode.tagName != "processor":
        fehler.append(("falscher Elternknoten", m.getAttribute("class")))
    if not m.getAttribute("class") or not m.getAttribute("match"):
        fehler.append(("class oder match fehlt", m.parentNode.getAttribute("name")))

# 3. Chaque destination ToProcessor doit exister
zielnamen = set()
for m in dom.getElementsByTagName("mailet"):
    if m.getAttribute("class") == "ToProcessor":
        for el in m.getElementsByTagName("processor"):
            text = "".join(c.data for c in el.childNodes if c.nodeType == c.TEXT_NODE)
            zielnamen.add(text.strip())

print("Doppelte Namen:      ", doppelt or "keine")
print("Strukturfehler:      ", fehler or "keine")
print("Ziele ohne Definition:", sorted(zielnamen - set(namen)) or "keine")
```

Un `ToProcessor` vers un processeur qui n’existe pas est l’erreur classique après un renommage. Le XML reste bien formé, mais la passerelle échoue seulement à l’exécution, généralement avec un message peu utile.

## Une parenthèse : c’est de la construction de compilateurs, pas du bricolage

Ce que vous faites ici porte un nom et repose sur une théorie. Un ensemble de règles est un **graphe de flot de contrôle**, soit le même modèle que les compilateurs utilisent depuis des décennies pour analyser les programmes. Il est utile de le savoir, car cela met à disposition des algorithmes éprouvés et, surtout, des affirmations claires sur leurs limites.

| Question dans l’ensemble de règles | Modèle | Méthode |
|---|---|---|
| Quels processeurs sont morts ? | Accessibilité depuis le nœud d’entrée | parcours en largeur ou en profondeur, complexité `O(V+E)` |
| Quelles règles d’une chaîne sont mortes ? | Nœuds après un saut inconditionnel | même parcours sur un graphe plus fin |
| Où une boucle de messagerie peut-elle apparaître ? | **Cycle dans le graphe** | composantes fortement connexes |
| Où une règle doit-elle se trouver pour s’appliquer à coup sûr ? | **Dominateur** du nœud d’entrée | arbre des dominateurs |

Les deux dernières lignes sont les plus utiles en pratique. Une boucle de messagerie n’est pas un mystérieux phénomène d’exploitation, mais un cycle dans le graphe de routage ; le compteur de sauts à l’exécution n’est que le frein d’urgence, vous trouvez structurellement la boucle avant cela. Et si vous voulez placer une règle que **chaque** message doit traverser, par exemple un filtre pour les domaines d’expéditeur non routables, vous recherchez un dominateur. Ce n’est pas une question de préférence, c’est calculable.

### Trouver les cycles avant qu’ils ne deviennent des boucles de messagerie

Le parcours en largeur répond à la question du code mort. Pour les boucles, vous avez besoin du parcours en profondeur, car une **arête de retour** y révèle le cycle. La méthode est le marquage classique en trois couleurs :

```python
def zyklen_finden(bloecke, ziele):
    WEISS, GRAU, SCHWARZ = 0, 1, 2
    farbe = {n: WEISS for n in bloecke}
    pfad, gefunden = [], []

    def besuche(knoten):
        farbe[knoten] = GRAU
        pfad.append(knoten)
        for ziel in ziele(bloecke.get(knoten, "")):
            if ziel not in farbe:
                continue
            if farbe[ziel] == GRAU:                 # Rueckwaertskante = Zyklus
                gefunden.append(pfad[pfad.index(ziel):] + [ziel])
            elif farbe[ziel] == WEISS:
                besuche(ziel)
        farbe[knoten] = SCHWARZ
        pfad.pop()

    for knoten in bloecke:
        if farbe[knoten] == WEISS:
            besuche(knoten)
    return gefunden

for zyklus in zyklen_finden(bloecke, ziele):
    print(" -> ".join(zyklus))
```

```text
outgoing -> processOutgoing -> outgoing
```

Une telle découverte ne prouve pas l’existence d’une boucle, car les arêtes sont conditionnées et ne seront peut-être jamais empruntées ensemble. Mais elle fournit la liste complète des endroits où une boucle **peut** apparaître, et c’est précisément ce que vous devez connaître avant une refonte. Le compteur de sauts à l’exécution n’est que le frein d’urgence ; ici, vous voyez la construction.

La **limite** de la méthode est tout aussi importante. Les arêtes sont conditionnées par des matchers, qui dépendent du contenu des messages. L’accessibilité exacte est donc indécidable dans le cas général ; l’analyse fournit une sur-approximation. Il en résulte une valeur informative asymétrique que vous devez connaître :

- **« Inaccessible » est fiable.** Si aucun chemin n’y mène, aucun message ne peut y parvenir. Vous pouvez supprimer ce code.
- **« Accessible » signifie seulement « structurellement non exclu ».** Le graphe ne dit pas si un message réel remplira un jour les conditions.

L’analyse ne remplace donc pas les tests, elle réduit l’espace de test. En pratique, cela reste un gain énorme : de 38 processeurs, vous passez à 18 que vous devez réellement vérifier.

Vous n’avez expressément pas besoin de méthodes d’apprentissage automatique, telles que les Graph Neural Networks ou les plongements de nœuds. Elles sont utiles pour de grands graphes à structure inconnue et présentant des motifs statistiques. Un ensemble de règles comporte quelques dizaines de nœuds, une structure entièrement connue et une sémantique déterministe. Les algorithmes exacts sont ici non seulement moins coûteux, ils fournissent des preuves plutôt que des probabilités.

## Pièges du traitement automatisé

Lorsque vous modifiez un ensemble de règles par script, trois erreurs surviennent de manière fiable. Je les ai toutes commises moi-même.

**Le classique : le motif gourmand qui franchit les limites des processeurs.** Pour supprimer un processeur avec une expression régulière, on utilise naturellement :

```python
muster = re.compile(r'   <processor name="%s">.*?</processor>\n' % name, re.S)
```

C’est incorrect. Dans la chaîne, chaque mailet `ToProcessor` contient un `<processor>ziel</processor>`, et le `.*?` non gourmand s’arrête précisément là. Résultat : la moitié du processeur est supprimée, un reste composé de `</mailet>` et de `</processor>` demeure, et le XML est détruit. Ancrez plutôt le motif sur l’indentation de la balise fermante et contrôlez l’équilibre des balises avec :

```python
muster = re.compile(r'   <processor name="%s">.*?\n   </processor>\n' % name, re.S)

treffer = muster.search(daten)
assert treffer, "Prozessor nicht gefunden"
block = treffer.group(0)
assert block.count("<mailet ") == block.count("</mailet>"), "Tag-Bilanz stimmt nicht"
```

**Fins de ligne.** La configuration utilise généralement CRLF. Lisez en Python avec `rb`, normalisez en `\n` pour le traitement, puis réécrivez en CRLF à la fin. Oublier cela produit un fichier aux fins de ligne mixtes, qui peut être refusé silencieusement selon le produit.

**Caractères spéciaux.** Conservez le fichier en ASCII pur et écrivez les umlauts sous forme de références de caractères (`&#228;` pour ä). Vous éviterez ainsi toute discussion sur les encodages entre l’éditeur, le script et l’interface Web de la passerelle.

Après chaque modification, vérifiez au minimum le bon formatage, l’absence de modification des fins de ligne et l’absence de modification du nombre de processeurs. Trois lignes de contrôle épargnent un retour en arrière.

## La méthode de refonte : arbre parallèle avec un aiguillage

Venons-en maintenant à la reconstruction proprement dite. La voie évidente, qui consiste à transformer progressivement l’ensemble de règles existant, est la pire : vous ne pouvez pas revenir proprement en arrière et vous ne pouvez plus lire l’état ancien.

L’arbre parallèle a fait ses preuves :

**Étape 1 : construire le nouvel arbre à côté.** Créez les nouveaux processeurs avec un suffixe de nom, par exemple `rootV2`, `incomingV2`, `outgoingV2`. L’ancien arbre reste intégralement présent et inchangé.

**Étape 2 : un seul aiguillage.** Au début du point d’entrée existant se trouve exactement un mailet :

```xml
<processor name="root">
   <mailet class="ToProcessor" match="All">
      <processor>rootV2</processor>
   </mailet>
   <!-- alles Bisherige bleibt hier stehen, ist aber nun unerreichbar -->
</processor>
```

Ainsi, tout le trafic passe par le nouvel arbre. L’ancien est inaccessible, mais reste entièrement présent. **Le retour en arrière consiste à supprimer ces trois lignes**, ce qui est compréhensible dans toute situation, même pour une personne qui n’a pas effectué la refonte.

**Étape 3 : l’accessibilité comme réception.** Exécutez l’analyse de l’outil 2 et vérifiez trois points : le nouveau point d’entrée est référencé exactement une fois, tous les nouveaux processeurs sont accessibles et l’ancien arbre est entièrement inaccessible. C’est un critère de réception objectif plutôt qu’un contrôle visuel.

**Étape 4 : ne nettoyer qu’après validation.** Lorsque le nouvel arbre est confirmé en exploitation, supprimez l’ancien et retirez les suffixes. Ce n’est qu’à ce moment que vous perdez le retour en arrière dans le fichier, et jusque-là vous n’en avez pas eu besoin.

Pour les étapes intermédiaires que vous souhaitez observer sans encore les activer, les mailets d’observation purs conviennent : ils consignent les informations, mais ne modifient pas le routage. Vous collectez ainsi les données nécessaires à la décision sans risque.

## Intégrer également la visibilité

Lors de la reconstruction, il est utile de prendre en compte deux éléments qui feront ensuite la différence en exploitation.

**Ne rejetez jamais directement dans la chaîne principale.** Un mailet qui rejette un message ne laisse dans l’historique du message que l’indication qu’il a été supprimé, sans raison. Bifurquez plutôt vers un processeur explicitement nommé, par exemple `dropNonRoutable`. Son seul nom apparaît dans l’historique et indique déjà ce qui s’est produit.

**Toutes les journalisations n’apparaissent pas dans l’historique du message.** De nombreux produits proposent deux mécanismes : l’un pour le journal du serveur et l’autre pour l’historique, que le support peut également consulter. Seul le second est visible dans l’historique. Si vous ne définissez que le premier, vous avez bien journalisé, mais la trace affichera toujours uniquement « message supprimé ». Rédigez les entrées d’historique en langage clair et nommez la règle : « rejeté délibérément par la règle pour les domaines d’expéditeur non routables, pas d’erreur de distribution » évite énormément de questions en exploitation.

## Le cluster fait partie de la tâche

Un point régulièrement sous-estimé : si la passerelle fonctionne sur plusieurs nœuds, la configuration doit être enregistrée **à l’identique sur tous les nœuds et de manière persistante après redémarrage**. Si elle n’est active que sur un nœud, le comportement dépend du nœud qui traite le message, et vos tests mesurent le hasard.

Le cas où une modification fonctionne mais n’a pas été persistée est particulièrement désagréable. Le nœud fonctionne alors correctement jusqu’à son redémarrage, puis revient à l’ancien état. Après chaque déploiement, vérifiez donc les deux points : même état sur tous les nœuds, et persistance de cet état après un redémarrage.

## En résumé

Traitez l’ensemble de règles comme un graphe, et non comme un fichier texte. Un parcours en largeur depuis le point d’entrée sépare en quelques lignes de code ce qui est vivant de ce qui est mort, et l’analyse au sein des chaînes identifie en outre les règles qui sont présentes mais ne sont jamais atteintes après un départ inconditionnel.

Pour la refonte elle-même, l’arbre parallèle avec un seul aiguillage est la méthode offrant le meilleur rapport entre effort et sécurité. Et l’analyse d’accessibilité vous fournit en même temps le critère de réception correspondant.

## Sources

1.  [Apache James: Mailet container configuration](https://james.apache.org/server/config-mailetcontainer.html): structure du gestionnaire de spool, des processeurs, des mailets et des matchers, ainsi que l’ordre de traitement.

2.  [Apache James: Provided mailets](https://james.apache.org/server/dev-provided-mailets.html): référence des mailets fournis, y compris ToProcessor et les paramètres de transmission et de consommation.

3.  [Apache James: Provided matchers](https://james.apache.org/server/dev-provided-matchers.html): référence des matchers, notamment All, HostIsLocal et les variantes liées aux destinataires.

4.  [Apache James: Mailet API](https://james.apache.org/server/dev-mailet-api.html): contrat entre le mailet et le conteneur, base pour comprendre la consommation et la transmission.
