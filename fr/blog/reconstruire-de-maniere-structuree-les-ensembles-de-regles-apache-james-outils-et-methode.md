---
title: "Reconstruire de manière structurée les ensembles de règles Apache James : outils et méthode"
navTitle: "Reconstruire l’ensemble de règles"
description: "Après des années, les ensembles de règles Mailet développés au fil du temps contiennent des chemins morts que plus personne ne repère. Comment analyser l’ensemble de règles comme un graphe, trouver de manière fiable le code inaccessible et concevoir la refonte de sorte qu’un seul Mailet préserve le retour en arrière."
date: "2026-08-11"
kategorie: "Totemomail"
timeToRead: "16 min de lecture"
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
translationSourceHash: ebcf5bf98f1f74aa7784c74c558da4db240e69f02de722a0251dd832d1224403
translationModel: gpt-5.6-terra
translatedAt: 2026-08-31T10:01:39.390Z
translationReview: automatic
url: https://rafaelpfister.ch/fr/blog/reconstruire-de-maniere-structuree-les-ensembles-de-regles-apache-james-outils-et-methode
---

# Reconstruire de manière structurée les ensembles de règles Apache James : outils et méthode

Les passerelles de messagerie basées sur Apache James, dont Totemomail, pilotent l’ensemble de leur flux de messages au moyen d’un ensemble de règles XML. Après quelques années d’exploitation, cet ensemble de règles acquiert une propriété que presque personne ne remarque : une part considérable n’est jamais exécutée. Des règles ont été ajoutées, des aiguillages ont été placés avant elles, des branches n’aboutissaient nulle part et, comme rien ne cassait, tout est resté en place.

Le problème n’est pas l’espace disque. C’est que plus personne ne peut dire quelle règle s’applique réellement. Toute personne qui prépare une modification lit un fichier contenant des centaines de Mailets sans savoir lesquels sont seulement pertinents. Or, cette question peut recevoir une réponse mécanique.

Cet article décrit la méthode et les outils correspondants : analyser l’ensemble de règles comme un graphe orienté, trouver de manière fiable le code inaccessible et concevoir la refonte de sorte qu’un seul Mailet préserve le retour en arrière.

## Le modèle en quatre phrases

Un ensemble de règles se compose de **processeurs**, c’est-à-dire de chaînes nommées. Chaque chaîne contient des **Mailets** qui effectuent une action, et chaque Mailet possède un **Matcher** qui décide s’il correspond au message en cours. Un Mailet de classe `ToProcessor` transmet le message à une autre chaîne.

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

La structure est donc un graphe orienté : les processeurs sont des nœuds, les destinations `ToProcessor` sont des arêtes. Dès lors que vous le voyez ainsi, la question du code mort devient une tâche standard : une analyse d’accessibilité.

## Deux types de code mort

Avant de mesurer, vous devez savoir ce que vous recherchez. Il en existe deux formes, dont la seconde est la plus sournoise.

**Processeurs inaccessibles.** Des chaînes entières vers lesquelles aucun `ToProcessor` ne pointe plus. Elles figurent dans le fichier, mais ne sont jamais empruntées. C’est le cas évident.

**Reste mort au sein d’une chaîne.** Un `ToProcessor` avec `match="All"` correspond à **chaque** message et le transmet. Tout ce qui le suit dans la même chaîne n’est jamais atteint. Il en va de même pour les Mailets avec `passThrough=false`: ils consomment le message et prennent eux-mêmes en charge son traitement ultérieur ; les Mailets suivants ne le voient pas.

Cette seconde forme ne peut pas être trouvée par une simple recherche textuelle, car les lignes paraissent tout à fait normales. Il vous faut pour cela tenir compte de l’ordre au sein de la chaîne.

## Outil 1 : lire le graphe

Le point de départ est une analyse qui extrait les processeurs et leurs destinations. Le script suivant n’utilise que la bibliothèque standard et s’exécute sur toute installation Python :

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

Notez la différence entre la **balise de définition** `<processor name="...">` et la **balise de destination** `<processor>name</processor>` au sein d’un Mailet `ToProcessor`. Elles portent le même nom, mais n’ont pas la même signification. Les confondre produit des résultats absurdes. C’est précisément la source de l’erreur décrite plus loin.

## Outil 2 : accessibilité depuis le point d’entrée

Avec le graphe, l’analyse consiste en un parcours en largeur à partir de `root`. Tout ce qui n’est pas visité est mort :

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

Une sortie typique pour un ensemble de règles développé au fil du temps :

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

Vingt processeurs sur 38, totalisant plus de 160 Mailets, qui ne sont jamais exécutés. Ce n’est pas une exception, mais le cas normal dans un environnement ayant connu plusieurs refontes.

## Outil 3 : trouver le reste mort au sein des chaînes

Passons maintenant à la seconde forme. Parcourez chaque chaîne accessible, Mailet par Mailet, et marquez tout ce qui suit la première sortie inconditionnelle :

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

Ce constat est plus précieux que la liste des processeurs, car il se trouve au milieu de chaînes actives. Quiconque ajoute une règle et l’insère sous un `ToProcessor match="All"` écrit une règle qui ne s’appliquera jamais, puis s’étonne de son inefficacité.

## Outil 4 : contrôle de structure

Du XML bien formé ne suffit pas. Ces quatre contrôles détectent les erreurs qu’un analyseur XML laisse passer, mais que la passerelle n’accepte pas :

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

Un `ToProcessor` qui pointe vers un processeur inexistant est l’erreur classique après un renommage. Le XML reste bien formé, mais la passerelle échoue seulement à l’exécution, généralement avec un message peu utile.

## Mise en perspective : il s’agit de construction de compilateurs

Ce que vous faites ici a un nom et une théorie. Un ensemble de règles est un **graphe de flux de contrôle**, soit le même modèle que les compilateurs utilisent depuis des décennies pour analyser les programmes. Il est utile de le savoir, car cela fournit des algorithmes éprouvés et, plus important encore, des affirmations claires sur leurs limites.

| Question dans l’ensemble de règles | Modèle | Méthode |
|---|---|---|
| Quels processeurs sont morts ? | Accessibilité depuis le nœud d’entrée | parcours en largeur ou en profondeur, complexité `O(V+E)` |
| Quelles règles d’une chaîne sont mortes ? | Nœuds après un saut inconditionnel | même recherche sur un graphe plus fin |
| Où une boucle de messagerie peut-elle apparaître ? | **Cycle dans le graphe** | composantes fortement connexes |
| Où une règle doit-elle se trouver pour qu’elle s’applique à coup sûr ? | **Dominateur** du nœud d’entrée | arbre des dominateurs |

Les deux dernières lignes sont les plus utiles en pratique. Une boucle de messagerie n’est pas un mystérieux phénomène d’exploitation, mais un cycle dans le graphe de routage ; le compteur de sauts à l’exécution n’est que le frein d’urgence, alors que vous pouvez trouver structurellement la boucle auparavant. Et si vous souhaitez placer une règle que **chaque** message doit traverser, par exemple un filtre pour les domaines d’expéditeur non routables, recherchez un dominateur. Ce n’est pas une question de préférence : c’est calculable.

### Trouver les cycles avant qu’ils ne deviennent des boucles de messagerie

Le parcours en largeur répond à la question du code mort. Pour les boucles, vous avez besoin du parcours en profondeur, car une **arête de retour** y indique le cycle. La méthode est le marquage classique à trois couleurs :

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

Une telle détection ne prouve pas l’existence d’une boucle, car les arêtes sont gardées par des conditions qui ne seront peut-être jamais empruntées ensemble. Elle fournit toutefois la liste complète des endroits où une boucle **peut** apparaître, et c’est précisément ce que vous devez connaître avant une refonte. Le compteur de sauts à l’exécution n’est que le frein d’urgence ; ici, vous voyez la construction.

La **limite** de la méthode est tout aussi importante. Les arêtes sont gardées par des Matchers, qui dépendent du contenu des messages. L’accessibilité exacte est donc indécidable dans le cas général ; l’analyse fournit une sur-approximation. Il en découle une valeur informative asymétrique que vous devez connaître :

- **« Inaccessible » est fiable.** Si aucun chemin n’y mène, aucun message ne peut y arriver. Vous pouvez supprimer ce code.
- **« Accessible » signifie seulement « non exclu structurellement ».** Le graphe ne dit pas si un message réel satisfera un jour les conditions.

L’analyse ne remplace donc pas les tests ; elle réduit l’espace de test. En pratique, c’est néanmoins un gain immense : sur 38 processeurs, il n’en reste plus que 18 à vérifier.

Vous n’avez expressément pas besoin ici de méthodes issues de l’apprentissage automatique, telles que les Graph Neural Networks ou les plongements de nœuds. Elles sont utiles pour de grands graphes à structure inconnue et présentant des motifs statistiques. Un ensemble de règles compte quelques dizaines de nœuds, possède une structure entièrement connue et une sémantique déterministe. Les algorithmes exacts sont ici non seulement moins coûteux, ils fournissent des preuves plutôt que des probabilités.

## Sources d’erreur lors du traitement automatisé

Si vous modifiez un ensemble de règles par script, trois erreurs surviennent systématiquement. Je les ai toutes commises moi-même.

**Le motif gourmand au-delà des frontières de processeur.** Pour supprimer un processeur à l’aide d’une expression régulière, on utilise naturellement :

```python
muster = re.compile(r'   <processor name="%s">.*?</processor>\n' % name, re.S)
```

C’est incorrect. Dans la chaîne, chaque Mailet `ToProcessor` contient un `<processor>ziel</processor>`, et le `.*?` non gourmand s’arrête précisément là. Résultat : la moitié du processeur est supprimée, un fragment constitué de `</mailet>` et de `</processor>` demeure, et le XML est détruit. Ancrez plutôt l’expression sur l’indentation de la balise fermante et vérifiez l’équilibre des balises avec :

```python
muster = re.compile(r'   <processor name="%s">.*?\n   </processor>\n' % name, re.S)

treffer = muster.search(daten)
assert treffer, "Prozessor nicht gefunden"
block = treffer.group(0)
assert block.count("<mailet ") == block.count("</mailet>"), "Tag-Bilanz stimmt nicht"
```

**Fins de ligne.** La configuration utilise habituellement CRLF. Lisez-la dans Python avec `rb`, normalisez en `\n` pour le traitement, puis réécrivez-la en CRLF à la fin. Oublier cela produit un fichier aux fins de ligne mixtes, que le produit peut refuser sans le signaler selon le cas.

**Caractères spéciaux.** Conservez le fichier en ASCII pur et écrivez les umlauts sous forme de références de caractères (`&#228;` pour ä). Vous éviterez ainsi toute discussion sur les encodages entre l’éditeur, le script et l’interface web de la passerelle.

Après chaque modification, vérifiez au minimum la bonne formation du XML, l’absence de modification des fins de ligne et le nombre inchangé de processeurs. Trois lignes de contrôle vous épargnent un retour en arrière.

## La méthode de refonte : arbre parallèle avec un aiguillage

Venons-en à la véritable reconstruction. La voie évidente, qui consiste à transformer progressivement l’ensemble de règles existant, est la pire : vous ne pouvez pas revenir proprement en arrière et vous ne pouvez plus lire l’ancien état.

La méthode de l’arbre parallèle a fait ses preuves :

**Étape 1 : construire le nouvel arbre à côté.** Créez les nouveaux processeurs avec un suffixe de nom, par exemple `rootV2`, `incomingV2`, `outgoingV2`. L’ancien arbre reste intégralement en place et inchangé.

**Étape 2 : un seul aiguillage.** Au début du point d’entrée existant se trouve exactement un Mailet :

```xml
<processor name="root">
   <mailet class="ToProcessor" match="All">
      <processor>rootV2</processor>
   </mailet>
   <!-- alles Bisherige bleibt hier stehen, ist aber nun unerreichbar -->
</processor>
```

Ainsi, tout le trafic passe par le nouvel arbre. L’ancien est inaccessible, mais toujours entièrement présent. **Le retour en arrière consiste à supprimer ces trois lignes**, ce qui est compréhensible dans toute situation, même pour une personne qui n’a pas réalisé la refonte.

**Étape 3 : l’accessibilité comme réception.** Exécutez l’analyse de l’outil 2 et vérifiez trois points : le nouveau point d’entrée est référencé exactement une fois, tous les nouveaux processeurs sont accessibles et l’ancien arbre est entièrement inaccessible. Il s’agit d’un critère de réception objectif, plutôt que d’un contrôle visuel.

**Étape 4 : ne nettoyer qu’après validation.** Une fois que le nouvel arbre a fait ses preuves en exploitation, supprimez l’ancien et retirez les suffixes. Ce n’est qu’à ce moment que vous perdez le retour en arrière dans le fichier, et jusque-là vous n’en aurez pas eu besoin.

Pour les étapes intermédiaires que vous souhaitez observer sans encore les activer, les Mailets de pure observation conviennent bien : ils journalisent, mais ne modifient pas le routage. Vous recueillez ainsi les données nécessaires à la décision sans prendre de risque.

## Concevoir aussi la visibilité

Lors de la reconstruction, il vaut la peine de tenir compte de deux aspects qui feront ensuite la différence en exploitation.

**Ne rejetez jamais directement dans la chaîne principale.** Un Mailet qui rejette un message ne laisse dans l’historique du message que l’indication qu’il a été supprimé, sans motif. Bifurquez plutôt vers un processeur explicitement nommé, par exemple `dropNonRoutable`. Le seul nom apparaît dans l’historique et indique déjà ce qui s’est passé.

**Toute journalisation n’apparaît pas dans l’historique du message.** De nombreux produits proposent deux mécanismes : l’un pour le journal du serveur, l’autre pour l’historique que le support peut également consulter. Seul le second est visible dans l’historique. Si vous ne définissez que le premier, vous avez certes journalisé l’événement, mais la trace affiche toujours seulement « Message supprimé ». Formulez les entrées d’historique en langage clair et nommez la règle : « rejeté délibérément par la règle relative aux domaines d’expéditeur non routables, aucune erreur de remise » évite énormément de questions en exploitation.

## Le cluster fait partie de la tâche

Un point régulièrement sous-estimé : si la passerelle s’exécute sur plusieurs nœuds, la configuration doit être déposée **de manière identique et persistante après redémarrage sur tous les nœuds**. Si elle n’est active que sur un nœud, le comportement dépend du nœud qui traite le message, et vos tests mesurent le hasard.

Le cas particulièrement désagréable est celui où une modification fonctionne, mais n’a pas été persistée. Le nœud fonctionne alors correctement jusqu’à son redémarrage, puis revient à l’ancien état. Après chaque déploiement, vérifiez donc les deux aspects : même état sur tous les nœuds, et maintien de cet état après un redémarrage.

## En résumé

Traitez l’ensemble de règles comme un graphe, et non comme un fichier texte. Un parcours en largeur depuis le point d’entrée sépare en quelques lignes de code ce qui est vivant de ce qui est mort, et l’analyse à l’intérieur des chaînes identifie en plus les règles qui sont présentes, mais ne sont jamais atteintes après une sortie inconditionnelle.

Pour la reconstruction elle-même, l’arbre parallèle avec un seul aiguillage est la méthode offrant le meilleur rapport entre effort et sécurité. L’analyse d’accessibilité vous fournit en même temps le critère de réception correspondant.

## Sources

1.  [Apache James: Mailet container configuration](https://james.apache.org/server/config-mailetcontainer.html): structure du gestionnaire de spool, des processeurs, des Mailets et des Matchers, ainsi que l’ordre de traitement.

2.  [Apache James: Provided mailets](https://james.apache.org/server/dev-provided-mailets.html): référence des Mailets fournis, y compris ToProcessor et les paramètres de transmission et de consommation.

3.  [Apache James: Provided matchers](https://james.apache.org/server/dev-provided-matchers.html): référence des Matchers, notamment All, HostIsLocal et les variantes liées aux destinataires.

4.  [Apache James: Mailet API](https://james.apache.org/server/dev-mailet-api.html): contrat entre le Mailet et le conteneur, fondement de la compréhension de la consommation et de la transmission.
