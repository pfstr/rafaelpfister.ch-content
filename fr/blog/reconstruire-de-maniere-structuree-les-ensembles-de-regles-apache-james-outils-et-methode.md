---
title: "Reconstituer méthodiquement des ensembles de règles Apache James : outils et méthode"
navTitle: "Reconstituer l’ensemble de règles"
description: "Après des années, les ensembles de règles Mailet accumulent des chemins morts que plus personne ne reconnaît. Comment analyser l’ensemble de règles sous forme de graphe, trouver de manière fiable le code inatteignable et concevoir la transformation pour qu’un seul Mailet maintienne la possibilité de retour."
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
translationSourceHash: acf116bf560f396b8a3f679fd2ca62a600e127ccf6cd7522c4a8534bb9876c0e
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:35:03.560Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/fr/blog/reconstruire-de-maniere-structuree-les-ensembles-de-regles-apache-james-outils-et-methode
---

# Reconstituer méthodiquement des ensembles de règles Apache James : outils et méthode

Les passerelles de messagerie basées sur Apache James, dont Totemomail, pilotent l’ensemble de leur flux de messages au moyen d’un ensemble de règles XML. Après quelques années d’exploitation, cet ensemble de règles acquiert une propriété que presque personne ne remarque : une part importante n’est jamais exécutée. Des règles ont été ajoutées, des aiguillages placés en amont, des branches sont devenues des impasses et, comme rien ne cassait, tout est resté en place.

Le problème n’est pas l’espace mémoire. C’est que plus personne ne peut dire quelle règle s’applique réellement. Quiconque prévoit une modification lit un fichier comptant des centaines de Mailets sans savoir lesquels sont seulement pertinents. Or, cette question peut recevoir une réponse mécanique.

Cet article décrit la méthode et les outils nécessaires : analyser l’ensemble de règles comme un graphe orienté, trouver de manière fiable le code inatteignable et organiser la transformation afin qu’un seul Mailet maintienne la possibilité de retour.

## Le modèle en quatre phrases

Un ensemble de règles se compose de **processeurs**, c’est-à-dire de chaînes nommées. Chaque chaîne contient des **Mailets** qui exécutent une action, et chaque Mailet possède un **Matcher** qui décide s’il s’applique au message actuel. Un Mailet de la classe `ToProcessor` transmet le message à une autre chaîne.

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

La structure est donc un graphe orienté : les processeurs sont les nœuds et les destinations `ToProcessor` sont les arêtes. Dès lors que vous la considérez ainsi, la question du code mort devient une tâche standard : une analyse d’accessibilité.

## Deux types de code mort

Avant de mesurer, vous devez savoir ce que vous recherchez. Il existe deux formes, et la seconde est la plus insidieuse.

**Processeurs inatteignables.** Des chaînes entières vers lesquelles aucun `ToProcessor` ne pointe plus. Elles se trouvent dans le fichier, mais n’y entrent jamais. C’est le cas évident.

**Reste mort au sein d’une chaîne.** Un `ToProcessor` avec `match="All"` s’applique à **chaque** message et le transmet. Tout ce qui se trouve en dessous dans la même chaîne n’est jamais atteint. Il en va de même pour les Mailets avec `passThrough=false`: ils consomment le message et prennent eux-mêmes en charge le traitement ultérieur ; les Mailets suivants ne le voient pas.

Cette seconde forme ne peut pas être trouvée par une simple recherche textuelle, car les lignes semblent parfaitement normales. Il faut tenir compte de l’ordre au sein de la chaîne.

## Outil 1 : extraire le graphe

Le point de départ est une analyse qui extrait les processeurs et leurs destinations. Le script suivant n’utilise que la bibliothèque standard et s’exécute sur toute installation Python :

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

Notez la différence entre la **balise de définition** `<processor name="...">` et la **balise de destination** `<processor>name</processor>` à l’intérieur d’un Mailet `ToProcessor`. Elles portent le même nom, mais ne désignent pas la même chose. Les confondre produit des résultats absurdes. C’est précisément l’origine de l’erreur décrite plus loin.

## Outil 2 : accessibilité depuis le point d’entrée

Avec le graphe, l’analyse consiste en une recherche en largeur depuis `root`. Tout ce qui n’est pas visité est mort :

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

Voici une sortie typique pour un ensemble de règles qui a évolué au fil du temps :

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

Vingt processeurs sur 38, totalisant plus de 160 Mailets, ne sont jamais exécutés. Ce n’est pas une exception, mais le cas normal dans un environnement ayant connu plusieurs transformations.

## Outil 3 : trouver le reste mort à l’intérieur des chaînes

Passons à la seconde forme. Parcourez chaque chaîne accessible, Mailet par Mailet, et marquez tout ce qui suit la première sortie inconditionnelle :

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

Ce constat est plus précieux que la liste des processeurs, car il se situe au milieu de chaînes actives. Quiconque ajoute une règle et l’insère sous un `ToProcessor match="All"` a écrit une règle qui ne s’appliquera jamais, puis s’étonne de son inefficacité.

## Outil 4 : vérification structurelle

Du XML bien formé ne suffit pas. Ces quatre contrôles détectent les erreurs qu’un analyseur laisse passer, mais que la passerelle n’accepte pas :

```python
dom = xml.dom.minidom.parseString(daten.replace("\n", "\r\n").encode("utf-8"))

namen = [p.getAttribute("name")
         for p in dom.getElementsByTagName("processor")
         if p.getAttribute("name")]

# 1. Noms de processeur en double
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

Un `ToProcessor` vers un processeur inexistant est l’erreur classique après un renommage. Le XML reste bien formé, mais la passerelle échoue seulement à l’exécution, généralement avec un message peu utile.

## Mise en perspective : c’est de la construction de compilateurs

Ce que vous faites ici a un nom et une théorie. Un ensemble de règles est un **graphe de flux de contrôle**, soit le même modèle que les compilateurs utilisent depuis des décennies pour analyser les programmes. Il est utile de le savoir, car des algorithmes éprouvés sont disponibles et, plus important encore, des énoncés clairs existent quant à leurs limites.

| Question dans l’ensemble de règles | Modèle | Méthode |
|---|---|---|
| Quels processeurs sont morts ? | Accessibilité depuis le nœud d’entrée | recherche en largeur ou en profondeur, coût `O(V+E)` |
| Quelles règles d’une chaîne sont mortes ? | nœuds après un saut inconditionnel | même recherche sur un graphe plus fin |
| Où une boucle de messagerie peut-elle se former ? | **cycle dans le graphe** | composantes fortement connexes |
| Où une règle doit-elle se trouver pour s’appliquer à coup sûr ? | **dominateur** du nœud d’entrée | arbre des dominateurs |

Les deux dernières lignes sont les plus utiles en pratique. Une boucle de messagerie n’est pas un mystérieux phénomène d’exploitation, mais un cycle dans le graphe de routage ; le compteur de sauts à l’exécution n’est qu’un frein d’urgence, tandis que la structure permet de trouver la boucle en amont. Et si vous souhaitez placer une règle que **chaque** message doit traverser, par exemple un filtre pour les domaines expéditeurs non routables, recherchez un dominateur. Ce n’est pas une question de préférence, mais un calcul.

### Trouver les cycles avant qu’ils ne deviennent des boucles de messagerie

La recherche en largeur répond à la question du code mort. Pour les boucles, il faut une recherche en profondeur, car une **arête de retour** y indique un cycle. La méthode est le classique marquage en trois couleurs :

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

Une telle découverte ne prouve pas l’existence d’une boucle, car les arêtes sont conditionnées par des Matcher et ne sont peut-être jamais empruntées ensemble. Elle constitue toutefois la liste complète des endroits où une boucle **peut** apparaître, et c’est précisément ce que vous devez connaître avant une transformation. Le compteur de sauts à l’exécution n’est qu’un frein d’urgence ; ici, vous voyez la construction.

La **limite** de la méthode est tout aussi importante. Les arêtes sont conditionnées par des Matcher, lesquels dépendent du contenu du message. L’accessibilité exacte est donc généralement indécidable ; l’analyse fournit une sur-approximation. Il en résulte une valeur informative asymétrique que vous devez connaître :

- **« Inatteignable » est fiable.** Si aucun chemin n’y mène, aucun message ne peut y parvenir. Vous pouvez supprimer ce code.
- **« Accessible » signifie seulement « non exclu structurellement ».** Le graphe ne dit pas si un message réel remplira un jour les conditions.

L’analyse ne remplace donc pas les tests, elle réduit l’espace de test. En pratique, c’est néanmoins un gain considérable : sur 38 processeurs, il n’en reste plus que 18 à vérifier.

Vous n’avez explicitement pas besoin ici de méthodes issues de l’apprentissage automatique, telles que les Graph Neural Networks ou les représentations vectorielles de nœuds. Elles sont utiles pour de grands graphes à structure inconnue et aux motifs statistiques. Un ensemble de règles comporte quelques dizaines de nœuds, une structure entièrement connue et une sémantique déterministe. Les algorithmes exacts sont ici non seulement moins coûteux, mais fournissent des preuves plutôt que des probabilités.

## Sources d’erreur lors du traitement automatisé

Lorsque vous modifiez un ensemble de règles par script, trois erreurs surviennent de façon fiable. Je les ai toutes commises moi-même.

**Le motif gourmand qui traverse les limites des processeurs.** Pour supprimer un processeur au moyen d’une expression régulière, on est naturellement tenté d’utiliser :

```python
muster = re.compile(r'   <processor name="%s">.*?</processor>\n' % name, re.S)
```

C’est incorrect. Au sein de la chaîne, chaque Mailet `ToProcessor` contient un `<processor>ziel</processor>`, et le `.*?` non gourmand s’arrête précisément à cet endroit. Résultat : la moitié du processeur est supprimée, un fragment composé de `</mailet>` et `</processor>` subsiste, et le XML est détruit. Ancrez plutôt l’expression sur l’indentation de la balise fermante et vérifiez l’équilibre des balises par rapport à :

```python
muster = re.compile(r'   <processor name="%s">.*?\n   </processor>\n' % name, re.S)

treffer = muster.search(daten)
assert treffer, "Prozessor nicht gefunden"
block = treffer.group(0)
assert block.count("<mailet ") == block.count("</mailet>"), "Tag-Bilanz stimmt nicht"
```

**Fins de ligne.** La configuration utilise généralement CRLF. Lisez en Python avec `rb`, normalisez en `\n` pour le traitement, puis réécrivez à la fin en CRLF. Si vous l’oubliez, vous produisez un fichier aux fins de ligne mixtes, qui peut être refusé sans message selon le produit.

**Caractères spéciaux.** Conservez le fichier en ASCII pur et écrivez les umlauts sous forme de références de caractères (`&#228;` pour ä). Vous éviterez ainsi toute discussion sur les encodages entre l’éditeur, le script et l’interface web de la passerelle.

Après chaque modification, vérifiez au minimum que le fichier est bien formé, que les fins de ligne sont inchangées et que le nombre de processeurs est inchangé. Trois lignes de contrôle épargnent un retour en arrière.

## La méthode de transformation : arbre parallèle avec un aiguillage

Venons-en à la véritable reconstruction. La voie évidente, qui consiste à transformer l’ensemble de règles existant étape par étape, est la pire : vous ne pouvez pas revenir proprement en arrière et vous ne pouvez plus lire l’état précédent.

L’arbre parallèle a fait ses preuves :

**Étape 1 : construire le nouvel arbre à côté.** Créez les nouveaux processeurs avec un suffixe de nom, par exemple `rootV2`, `incomingV2`, `outgoingV2`. L’ancien arbre reste entièrement présent et inchangé.

**Étape 2 : un seul aiguillage.** Au début du point d’entrée existant se trouve exactement un Mailet :

```xml
<processor name="root">
   <mailet class="ToProcessor" match="All">
      <processor>rootV2</processor>
   </mailet>
   <!-- alles Bisherige bleibt hier stehen, ist aber nun unerreichbar -->
</processor>
```

Ainsi, tout le trafic passe par le nouvel arbre. L’ancien est inatteignable, mais reste entièrement présent. **Le retour en arrière consiste à supprimer ces trois lignes**, ce qui reste compréhensible dans toute situation, y compris pour une personne n’ayant pas effectué la transformation.

**Étape 3 : l’accessibilité comme réception.** Exécutez l’analyse de l’outil 2 et vérifiez trois points : le nouveau point d’entrée est référencé exactement une fois, tous les nouveaux processeurs sont accessibles et l’ancien arbre est entièrement inatteignable. Il s’agit d’un critère de réception objectif, plutôt que d’un contrôle visuel.

**Étape 4 : ne nettoyer qu’après validation.** Une fois le nouvel arbre confirmé en exploitation, supprimez l’ancien et retirez les suffixes. Ce n’est qu’à ce moment que vous perdez la possibilité de retour dans le fichier, et jusque-là vous n’en aurez pas eu besoin.

Pour les étapes intermédiaires que vous souhaitez observer sans encore les activer, les Mailets purement d’observation conviennent : ils consignent les informations, mais ne modifient pas le routage. Vous recueillez ainsi les données nécessaires à la décision, sans risque.

## Intégrer également la visibilité

Lors de la reconstruction, il vaut la peine de tenir compte de deux éléments qui feront ensuite toute la différence en exploitation.

**Ne rejetez jamais directement dans la chaîne principale.** Un Mailet qui rejette un message ne laisse dans l’historique du message que l’indication d’une suppression, sans motif. Bifurquez plutôt vers un processeur spécifiquement nommé, par exemple `dropNonRoutable`. Le nom seul apparaît dans l’historique et indique déjà ce qui se passe.

**Toutes les journalisations n’apparaissent pas dans l’historique du message.** De nombreux produits connaissent deux mécanismes : l’un pour le journal du serveur et l’autre pour l’historique consulté également par le support. Seul le second est visible dans l’historique. Si vous ne définissez que le premier, vous avez certes journalisé l’événement, mais la trace indique toujours uniquement « message supprimé ». Rédigez les entrées d’historique en langage clair et mentionnez la règle : « rejeté intentionnellement par la règle pour les domaines expéditeurs non routables, aucune erreur de livraison » évite de nombreuses demandes de précisions en exploitation.

## Le cluster fait partie de la tâche

Un point régulièrement sous-estimé : si la passerelle s’exécute sur plusieurs nœuds, la configuration doit être enregistrée **de manière identique et persistante après redémarrage sur tous les nœuds**. Si elle n’est active que sur un seul nœud, le comportement dépend du nœud qui traite le message et vos tests mesurent le hasard.

Le cas où une modification fonctionne, mais n’a pas été persistée, est particulièrement désagréable. Le nœud fonctionne alors correctement jusqu’à son redémarrage, puis revient à l’ancien état. Après chaque déploiement, vérifiez donc ces deux éléments : même état sur tous les nœuds et persistance de cet état après un redémarrage.

## En résumé

Traitez l’ensemble de règles comme un graphe, non comme un fichier texte. Une recherche en largeur depuis le point d’entrée sépare en quelques lignes de code ce qui est vivant de ce qui est mort, et l’analyse à l’intérieur des chaînes détecte en outre les règles qui sont présentes, mais ne sont jamais atteintes après une sortie inconditionnelle.

Pour la transformation elle-même, l’arbre parallèle avec un seul aiguillage est la méthode offrant le meilleur rapport entre effort et sécurité. L’analyse d’accessibilité vous fournit également le critère de réception correspondant.

## Historique des modifications

- **25 octobre 2026 :** liens des Sources mis à jour vers les nouvelles adresses de la documentation Apache James ; les références aux Mailets et aux Matchers sont désormais regroupées dans une seule source.

## Sources

1.  [Apache James: Mailet container configuration](https://james.apache.org/server/config-mailetcontainer.html): structure du gestionnaire de spool, processeurs, Mailets et Matchers, ainsi que l’ordre de traitement.

2.  [Apache James: Mailets and Matchers Reference](https://james.apache.org/mailet/standard/mailet-report.html): référence des Mailets fournis, y compris ToProcessor, et des Matchers, notamment All, HostIsLocal et RecipientIs.

3.  [Apache James: Mailet API](https://james.apache.org/mailet/api/index.html): description de l’interface Mailet et de l’exécution dans le conteneur Mailet, base pour comprendre la consommation et la transmission.
