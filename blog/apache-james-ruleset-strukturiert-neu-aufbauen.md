---
title: "Apache-James-Regelwerke strukturiert neu aufbauen: Werkzeuge und Methode"
navTitle: "Regelwerk neu aufbauen"
description: "Gewachsene Mailet-Regelwerke enthalten nach Jahren tote Pfade, die niemand mehr erkennt. Wie Sie das Regelwerk als Graph auswerten, unerreichbaren Code zuverlässig finden, und den Umbau so bauen, dass ein einziges Mailet den Rückweg offen hält."
date: "2026-08-11"
kategorie: "Totemomail"
timeToRead: "16 Min. Lesezeit"
themen:
  - "totemomail"
  - "e-mail-verschluesselung"
  - "smtp-mailflow"
hauptthema: "totemomail"
produkte:
  - "totemomail"
  - "apache-james"
protokolle:
  - "smtp"
  - "troubleshooting"
  - "verschluesselung"
related:
  - "totemomail-m365"
  - "totemomail-licensed-user-limit-ldap-cleanup"
  - "mailflow-fehlersuche-kontrollierte-experimente"
slug: "apache-james-ruleset-strukturiert-neu-aufbauen"
translationId: "article-b9c98459a0ff6352"
url: "https://rafaelpfister.ch/blog/apache-james-ruleset-strukturiert-neu-aufbauen"
draft: false
---

# Apache-James-Regelwerke strukturiert neu aufbauen: Werkzeuge und Methode

Mailgateways auf Basis von Apache James, darunter Totemomail, steuern ihren gesamten Nachrichtenfluss über ein Regelwerk in XML. Nach einigen Jahren Betrieb hat dieses Regelwerk eine Eigenschaft, die kaum jemand bemerkt: Ein erheblicher Teil davon wird nie ausgeführt. Regeln kamen hinzu, Weichen wurden davorgesetzt, Zweige liefen ins Leere, und weil nichts kaputtging, blieb alles stehen.

Das Problem daran ist nicht der Plattenplatz. Es ist, dass niemand mehr sagen kann, welche Regel tatsächlich greift. Wer eine Änderung plant, liest eine Datei mit Hunderten von Mailets und weiss nicht, welche davon überhaupt relevant sind. Genau das lässt sich mechanisch beantworten.

Dieser Artikel beschreibt die Methode und die Werkzeuge dazu: das Regelwerk als gerichteten Graphen auswerten, unerreichbaren Code zuverlässig finden, und den Umbau so anlegen, dass ein einziges Mailet den Rückweg offen hält.

## Das Modell in vier Sätzen

Ein Regelwerk besteht aus **Prozessoren**, also benannten Ketten. Jede Kette enthält **Mailets**, die etwas tun, und jedes Mailet hat einen **Matcher**, der entscheidet, ob es auf die aktuelle Nachricht zutrifft. Ein Mailet der Klasse `ToProcessor` übergibt die Nachricht an eine andere Kette.

Der Einstiegspunkt heisst üblicherweise `root`. Von dort aus verzweigt alles Weitere.

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

Damit ist die Struktur ein gerichteter Graph: Prozessoren sind Knoten, `ToProcessor`-Ziele sind Kanten. Und sobald Sie das so sehen, ist die Frage nach totem Code eine Standardaufgabe, nämlich eine Erreichbarkeitsanalyse.

## Zwei Arten von totem Code

Bevor Sie messen, müssen Sie wissen, wonach Sie suchen. Es gibt zwei Formen, und die zweite ist die tückische.

**Unerreichbare Prozessoren.** Ganze Ketten, auf die kein `ToProcessor` mehr zeigt. Sie stehen in der Datei, werden aber nie betreten. Das ist der offensichtliche Fall.

**Toter Rest innerhalb einer Kette.** Ein `ToProcessor` mit `match="All"` trifft auf **jede** Nachricht zu und gibt sie weiter. Alles, was in derselben Kette darunter steht, wird nie erreicht. Dasselbe gilt für Mailets mit `passThrough=false`: Sie konsumieren die Nachricht und übernehmen die weitere Behandlung selbst, die nachfolgenden Mailets sehen sie nicht mehr.

Diese zweite Form findet keine einfache Textsuche, denn die Zeilen sehen völlig normal aus. Sie brauchen dafür die Reihenfolge innerhalb der Kette.

## Werkzeug 1: Den Graphen auslesen

Der Einstieg ist eine Auswertung, die Prozessoren und ihre Ziele extrahiert. Das folgende Skript nutzt nur die Standardbibliothek und läuft auf jeder Python-Installation:

```python
import re
import xml.dom.minidom

PFAD = "regelwerk.xml"

daten = open(PFAD, "rb").read().decode("utf-8").replace("\r\n", "\n")

# Prozessoren und ihre Blöcke schneiden
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

Beachten Sie den Unterschied zwischen dem **Definitionstag** `<processor name="...">` und dem **Zieltag** `<processor>name</processor>` innerhalb eines `ToProcessor`-Mailets. Beide heissen gleich, meinen aber Verschiedenes. Wer sie verwechselt, bekommt sinnlose Ergebnisse. Genau darauf beruht auch der Fallstrick weiter unten.

## Werkzeug 2: Erreichbarkeit ab dem Einstiegspunkt

Mit dem Graphen ist die Analyse eine Breitensuche ab `root`. Alles, was dabei nicht besucht wird, ist tot:

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

Eine typische Ausgabe bei einem gewachsenen Regelwerk:

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

Zwanzig von 38 Prozessoren mit zusammen über 160 Mailets, die nie ausgeführt werden. Das ist kein Ausreisser, sondern der Normalfall in einer Umgebung, die mehrere Umbauten hinter sich hat.

## Werkzeug 3: Den toten Rest innerhalb der Ketten finden

Jetzt die zweite Form. Gehen Sie jede erreichbare Kette Mailet für Mailet durch und markieren Sie alles nach dem ersten unbedingten Abgang:

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

Dieser Befund ist wertvoller als die Prozessorliste, weil er mitten in aktiven Ketten sitzt. Wer eine Regel ergänzt und sie unterhalb eines `ToProcessor match="All"` einfügt, hat eine Regel geschrieben, die niemals greift, und wundert sich anschliessend über die Wirkungslosigkeit.

## Werkzeug 4: Strukturprüfung

Wohlgeformtes XML ist nur die halbe Miete. Diese vier Prüfungen fangen die Fehler ab, die ein Parser durchgehen lässt, das Gateway aber nicht:

```python
dom = xml.dom.minidom.parseString(daten.replace("\n", "\r\n").encode("utf-8"))

namen = [p.getAttribute("name")
         for p in dom.getElementsByTagName("processor")
         if p.getAttribute("name")]

# 1. Doppelte Prozessornamen
doppelt = {n for n in namen if namen.count(n) > 1}

# 2. Jedes Mailet muss direktes Kind eines Prozessors sein und class + match haben
fehler = []
for m in dom.getElementsByTagName("mailet"):
    if m.parentNode.tagName != "processor":
        fehler.append(("falscher Elternknoten", m.getAttribute("class")))
    if not m.getAttribute("class") or not m.getAttribute("match"):
        fehler.append(("class oder match fehlt", m.parentNode.getAttribute("name")))

# 3. Jedes ToProcessor-Ziel muss existieren
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

Ein `ToProcessor` auf einen Prozessor, den es nicht gibt, ist der klassische Fehler nach einer Umbenennung. Das XML bleibt wohlgeformt, das Gateway stolpert erst zur Laufzeit darüber, und dann meist mit einer wenig hilfreichen Meldung.

## Ein Zwischenruf: das ist Compilerbau, nicht Bastelei

Was Sie hier tun, hat einen Namen und eine Theorie dahinter. Ein Regelwerk ist ein **Kontrollflussgraph**, also dasselbe Modell, mit dem Compiler seit Jahrzehnten Programme analysieren. Das lohnt sich zu wissen, weil damit fertige Algorithmen und, wichtiger, klare Aussagen über deren Grenzen zur Verfügung stehen.

| Frage im Regelwerk | Modell | Verfahren |
|---|---|---|
| Welche Prozessoren sind tot? | Erreichbarkeit ab dem Einstiegsknoten | Breiten- oder Tiefensuche, Aufwand `O(V+E)` |
| Welche Regeln in einer Kette sind tot? | Knoten nach einem unbedingten Sprung | dieselbe Suche auf feinerem Graphen |
| Wo kann ein Mail-Loop entstehen? | **Zyklus im Graphen** | starke Zusammenhangskomponenten |
| Wo muss eine Regel stehen, damit sie garantiert greift? | **Dominator** des Einstiegsknotens | Dominatorbaum |

Die letzten beiden Zeilen sind die praktisch wertvollsten. Ein Mail-Loop ist kein mysteriöses Betriebsphänomen, sondern ein Zyklus im Routing-Graphen; der Hop-Zähler zur Laufzeit ist nur die Notbremse, strukturell finden Sie die Schleife vorher. Und wenn Sie eine Regel platzieren wollen, die **jede** Nachricht passieren muss, etwa einen Filter für nicht routbare Absenderdomänen, dann fragen Sie nach einem Dominator. Das ist keine Geschmacksfrage, sondern berechenbar.

### Zyklen finden, bevor sie zu Mail-Loops werden

Die Breitensuche beantwortet die Frage nach totem Code. Für Schleifen brauchen Sie die Tiefensuche, denn dort verrät eine **Rückwärtskante** den Zyklus. Das Verfahren ist die klassische Dreifarbenmarkierung:

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

Ein solcher Fund ist kein Beweis für einen Loop, denn die Kanten sind bewacht und werden vielleicht nie gemeinsam genommen. Er ist aber die vollständige Liste der Stellen, an denen einer entstehen **kann**, und genau die wollen Sie vor einem Umbau kennen. Der Hop-Zähler zur Laufzeit ist nur die Notbremse; hier sehen Sie die Konstruktion.

Ebenso wichtig ist die **Grenze** des Verfahrens. Die Kanten sind durch Matcher bewacht, und die hängen vom Nachrichteninhalt ab. Exakte Erreichbarkeit ist damit im Allgemeinen unentscheidbar, die Analyse liefert eine Über-Approximation. Daraus folgt eine asymmetrische Aussagekraft, die Sie kennen müssen:

- **„Unerreichbar" ist verlässlich.** Wenn kein Pfad hinführt, kann dort keine Nachricht ankommen. Diesen Code dürfen Sie löschen.
- **„Erreichbar" heisst nur „strukturell nicht ausgeschlossen".** Ob je eine reale Nachricht die Bedingungen erfüllt, sagt der Graph nicht.

Die Analyse ersetzt den Test also nicht, sie verkleinert den Testraum. Für die Praxis ist das trotzdem ein enormer Gewinn: Aus 38 Prozessoren werden 18, die Sie überhaupt prüfen müssen.

Verfahren aus dem maschinellen Lernen, etwa Graph Neural Networks oder Knoteneinbettungen, brauchen Sie hier ausdrücklich nicht. Die lohnen sich bei grossen Graphen mit unbekannter Struktur und statistischen Mustern. Ein Regelwerk hat einige Dutzend Knoten, vollständig bekannte Struktur und deterministische Semantik. Exakte Algorithmen sind hier nicht nur billiger, sie liefern Beweise statt Wahrscheinlichkeiten.

## Fallstricke beim maschinellen Bearbeiten

Wenn Sie ein Regelwerk per Skript ändern, gibt es drei Fehler, die zuverlässig auftreten. Alle drei habe ich selbst gemacht.

**Der Klassiker: das gierige Muster über Prozessorgrenzen hinweg.** Wer einen Prozessor per regulärem Ausdruck entfernen will, greift naheliegend zu:

```python
muster = re.compile(r'   <processor name="%s">.*?</processor>\n' % name, re.S)
```

Das ist falsch. Innerhalb der Kette steht in jedem `ToProcessor`-Mailet ein `<processor>ziel</processor>`, und das nicht-gierige `.*?` stoppt genau dort. Das Ergebnis: Der halbe Prozessor wird entfernt, ein Rumpf aus `</mailet>` und `</processor>` bleibt stehen, und das XML ist zerstört. Verankern Sie stattdessen auf der Einrückung des schliessenden Tags und prüfen Sie die Tag-Bilanz gegen:

```python
muster = re.compile(r'   <processor name="%s">.*?\n   </processor>\n' % name, re.S)

treffer = muster.search(daten)
assert treffer, "Prozessor nicht gefunden"
block = treffer.group(0)
assert block.count("<mailet ") == block.count("</mailet>"), "Tag-Bilanz stimmt nicht"
```

**Zeilenenden.** Die Konfiguration verwendet üblicherweise CRLF. Lesen Sie in Python mit `rb`, normalisieren Sie auf `\n` für die Bearbeitung und schreiben Sie am Ende wieder CRLF zurück. Wer das vergisst, produziert eine Datei mit gemischten Zeilenenden, die je nach Produkt kommentarlos abgelehnt wird.

**Sonderzeichen.** Halten Sie die Datei in reinem ASCII und schreiben Sie Umlaute als Zeichenreferenzen (`&#228;` für ä). Das erspart Ihnen jede Diskussion über Kodierungen zwischen Editor, Skript und Weboberfläche des Gateways.

Prüfen Sie nach jeder Änderung mindestens auf Wohlgeformtheit, unveränderte Zeilenenden und unveränderte Prozessorzahl. Drei Zeilen Kontrolle sparen einen Rückbau.

## Die Methode für den Umbau: Parallelbaum mit einer Weiche

Nun zum eigentlichen Neuaufbau. Der naheliegende Weg, das bestehende Regelwerk Schritt für Schritt umzubauen, ist der schlechteste: Sie können nicht sauber zurück, und Sie können den alten Zustand nicht mehr lesen.

Bewährt hat sich stattdessen der Parallelbaum:

**Schritt 1: Neuen Baum daneben aufbauen.** Legen Sie die neuen Prozessoren mit einem Namenssuffix an, etwa `rootV2`, `incomingV2`, `outgoingV2`. Der alte Baum bleibt vollständig und unverändert bestehen.

**Schritt 2: Eine einzige Weiche.** Am Anfang des bisherigen Einstiegspunkts steht genau ein Mailet:

```xml
<processor name="root">
   <mailet class="ToProcessor" match="All">
      <processor>rootV2</processor>
   </mailet>
   <!-- alles Bisherige bleibt hier stehen, ist aber nun unerreichbar -->
</processor>
```

Damit läuft aller Verkehr durch den neuen Baum. Der alte ist unerreichbar, aber vollständig vorhanden. **Der Rückweg besteht aus dem Entfernen dieser drei Zeilen**, und das ist in jeder Situation nachvollziehbar, auch für eine Person, die den Umbau nicht gemacht hat.

**Schritt 3: Erreichbarkeit als Abnahme.** Lassen Sie die Analyse aus Werkzeug 2 laufen und prüfen Sie drei Punkte: Der neue Einstiegspunkt wird genau einmal referenziert, alle neuen Prozessoren sind erreichbar, und der alte Baum ist vollständig unerreichbar. Das ist ein objektives Abnahmekriterium statt einer Sichtprüfung.

**Schritt 4: Erst nach der Bewährung aufräumen.** Wenn der neue Baum im Betrieb bestätigt ist, entfernen Sie den alten und streichen die Suffixe. Erst dann verlieren Sie den Rückweg in der Datei, und bis dahin haben Sie ihn nicht gebraucht.

Für Zwischenschritte, die Sie beobachten, aber noch nicht scharf schalten wollen, eignen sich reine Beobachtungsmailets: Sie protokollieren, ändern aber das Routing nicht. Damit sammeln Sie die Daten, die für die Entscheidung fehlen, ohne Risiko.

## Sichtbarkeit gleich mitbauen

Beim Neuaufbau lohnt es sich, zwei Dinge zu berücksichtigen, die im Betrieb später den Unterschied machen.

**Verwerfen Sie nie direkt in der Hauptkette.** Ein Mailet, das eine Nachricht verwirft, hinterlässt im Nachrichtenverlauf nur den Hinweis, dass gelöscht wurde, ohne Grund. Verzweigen Sie stattdessen in einen eigens benannten Prozessor, etwa `dropNonRoutable`. Der Name allein erscheint im Verlauf und sagt bereits, was passiert ist.

**Nicht jede Protokollierung landet im Nachrichtenverlauf.** Viele Produkte kennen zwei Mechanismen: einen für das Serverprotokoll und einen für den Verlauf, den auch der Support sieht. Nur der zweite ist im Verlauf sichtbar. Wer ausschliesslich den ersten setzt, hat zwar geloggt, aber im Trace steht weiterhin nur „Nachricht gelöscht". Formulieren Sie die Verlaufseinträge im Klartext und nennen Sie die Regel: „durch die Regel für nicht routbare Absenderdomänen bewusst verworfen, kein Zustellfehler" spart im Betrieb sehr viel Rückfrage.

## Der Cluster ist Teil der Aufgabe

Ein Punkt, der regelmässig unterschätzt wird: Läuft das Gateway auf mehreren Knoten, muss die Konfiguration **auf allen Knoten identisch und neustartfest** hinterlegt sein. Ist sie nur auf einem Knoten aktiv, hängt das Verhalten davon ab, welcher Knoten die Nachricht bearbeitet, und Ihre Tests messen Zufall.

Besonders unangenehm ist der Fall, dass eine Änderung zwar läuft, aber nicht persistiert wurde. Dann arbeitet der Knoten korrekt, bis er neu startet, und fällt danach auf den alten Stand zurück. Prüfen Sie deshalb nach jedem Deployment beide Dinge: gleicher Stand auf allen Knoten, und der Stand überlebt einen Neustart.

## Zusammengefasst

Behandeln Sie das Regelwerk als Graphen, nicht als Textdatei. Eine Breitensuche ab dem Einstiegspunkt trennt in wenigen Zeilen Code Lebendes von Totem, und die Analyse innerhalb der Ketten findet zusätzlich die Regeln, die zwar dastehen, aber nach einem unbedingten Abgang nie erreicht werden.

Für den Umbau selbst ist der Parallelbaum mit einer einzigen Weiche die Methode mit dem besten Verhältnis von Aufwand zu Sicherheit. Und die Erreichbarkeitsanalyse liefert Ihnen gleich das Abnahmekriterium dazu.

## Quellen

1.  [Apache James: Mailet container configuration](https://james.apache.org/server/config-mailetcontainer.html): Aufbau des Spoolmanagers, Prozessoren, Mailets und Matcher sowie die Verarbeitungsreihenfolge.

2.  [Apache James: Provided mailets](https://james.apache.org/server/dev-provided-mailets.html): Referenz der mitgelieferten Mailets inklusive ToProcessor und der Parameter für Weitergabe und Konsum.

3.  [Apache James: Provided matchers](https://james.apache.org/server/dev-provided-matchers.html): Referenz der Matcher, unter anderem All, HostIsLocal und die empfängerbezogenen Varianten.

4.  [Apache James: Mailet API](https://james.apache.org/server/dev-mailet-api.html): Vertrag zwischen Mailet und Container, Grundlage für das Verständnis von Konsum und Weitergabe.
