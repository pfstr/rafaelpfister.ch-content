---
title: "Strukturert gjenoppbygging av Apache James-regelverk: verktøy og metode"
navTitle: "Bygg regelverket på nytt"
description: "Etablerte Mailet-regelverk inneholder etter mange år døde stier som ingen lenger oppdager. Slik evaluerer du regelverket som en graf, finner utilgjengelig kode pålitelig og organiserer ombyggingen slik at ett enkelt Mailet holder returveien åpen."
date: "2026-08-11"
kategorie: "Totemomail"
timeToRead: "16 min lesetid"
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
slug: "strukturert-gjenoppbygging-av-apache-james-regelverk-verktoy-og-metode"
translationId: "article-b9c98459a0ff6352"
draft: false
translationOf: apache-james-ruleset-strukturiert-neu-aufbauen
url: https://rafaelpfister.ch/no/blog/strukturert-gjenoppbygging-av-apache-james-regelverk-verktoy-og-metode
translationSourceHash: b0274af954ad40614bc74b37b7be1e6e9bee6c856e28105336eddfb967895884
translationModel: gpt-5.6-terra
translatedAt: 2026-08-12T05:11:42.206Z
translationReview: automatic
---

# Strukturert gjenoppbygging av Apache James-regelverk: verktøy og metode

E-postgatewayer basert på Apache James, inkludert Totemomail, styrer hele meldingsflyten gjennom et regelverk i XML. Etter noen års drift får dette regelverket en egenskap som knapt noen legger merke til: En betydelig del av det blir aldri kjørt. Regler ble lagt til, forgreninger ble plassert foran dem, grener endte blindt, og siden ingenting gikk i stykker, ble alt stående.

Problemet er ikke diskplassen. Det er at ingen lenger kan si hvilken regel som faktisk slår til. Den som planlegger en endring, leser en fil med hundrevis av Mailets og vet ikke hvilke av dem som i det hele tatt er relevante. Nettopp dette kan besvares mekanisk.

Denne artikkelen beskriver metoden og verktøyene: Evaluer regelverket som en rettet graf, finn utilgjengelig kode pålitelig, og legg opp ombyggingen slik at ett enkelt Mailet holder returveien åpen.

## Modellen i fire setninger

Et regelverk består av **prosessorer**, altså navngitte kjeder. Hver kjede inneholder **Mailets** som gjør noe, og hvert Mailet har en **Matcher** som avgjør om det gjelder den aktuelle meldingen. Et Mailet av klassen `ToProcessor` overfører meldingen til en annen kjede.

Inngangspunktet heter vanligvis `root`. Derfra forgrener alt annet seg.

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

Dermed er strukturen en rettet graf: Prosessorer er noder, `ToProcessor`-mål er kanter. Og så snart du ser det slik, er spørsmålet om død kode en standardoppgave, nemlig en tilgjengelighetsanalyse.

## To typer død kode

Før du måler, må du vite hva du leter etter. Det finnes to former, og den andre er den lumske.

**Utilgjengelige prosessorer.** Hele kjeder som ingen `ToProcessor` lenger peker til. De står i filen, men blir aldri gått inn i. Dette er det åpenbare tilfellet.

**Død rest innenfor en kjede.** Et `ToProcessor` med `match="All"` treffer **hver** melding og sender den videre. Alt som står under det i samme kjede, blir aldri nådd. Det samme gjelder Mailets med `passThrough=false`: De konsumerer meldingen og overtar den videre behandlingen selv; de påfølgende Mailets ser den ikke lenger.

Denne andre formen finner ikke et enkelt tekstsøk, for linjene ser helt normale ut. Du trenger rekkefølgen i kjeden for dette.

## Verktøy 1: Les ut grafen

Utgangspunktet er en analyse som henter ut prosessorer og målene deres. Følgende skript bruker bare standardbiblioteket og kjører på enhver Python-installasjon:

```python
import re
import xml.dom.minidom

PFAD = "regelwerk.xml"

daten = open(PFAD, "rb").read().decode("utf-8").replace("\r\n", "\n")

# Hent ut prosessorer og blokkene deres
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

Legg merke til forskjellen mellom **definisjonstagen** `<processor name="...">` og **måltagen** `<processor>name</processor>` innenfor et `ToProcessor`-Mailet. Begge heter det samme, men betyr ulike ting. Den som forveksler dem, får meningsløse resultater. Dette er også grunnlaget for fallgruven lenger ned.

## Verktøy 2: Tilgjengelighet fra inngangspunktet

Med grafen er analysen et bredde-først-søk fra `root`. Alt som ikke blir besøkt, er dødt:

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

En typisk utdata fra et etablert regelverk:

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

Tjue av 38 prosessorer med til sammen over 160 Mailets som aldri blir kjørt. Det er ikke et avvik, men normalen i et miljø som har gjennomgått flere ombygginger.

## Verktøy 3: Finn den døde resten i kjedene

Nå den andre formen. Gå gjennom hver tilgjengelige kjede Mailet for Mailet og marker alt etter den første ubetingede utgangen:

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

Dette funnet er mer verdifullt enn prosessorlisten, fordi det ligger midt i aktive kjeder. Den som legger til en regel og setter den under et `ToProcessor match="All"`, har skrevet en regel som aldri slår til, og undrer seg deretter over at den ikke har noen effekt.

## Verktøy 4: Strukturkontroll

Velformet XML er bare halve jobben. Disse fire kontrollene fanger opp feilene som en parser lar passere, men som gatewayen ikke gjør:

```python
dom = xml.dom.minidom.parseString(daten.replace("\n", "\r\n").encode("utf-8"))

namen = [p.getAttribute("name")
         for p in dom.getElementsByTagName("processor")
         if p.getAttribute("name")]

# 1. Dupliserte prosessornavn
doppelt = {n for n in namen if namen.count(n) > 1}

# 2. Hvert Mailet må være direkte barn av en prosessor og ha class + match
fehler = []
for m in dom.getElementsByTagName("mailet"):
    if m.parentNode.tagName != "processor":
        fehler.append(("falscher Elternknoten", m.getAttribute("class")))
    if not m.getAttribute("class") or not m.getAttribute("match"):
        fehler.append(("class oder match fehlt", m.parentNode.getAttribute("name")))

# 3. Hvert ToProcessor-mål må finnes
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

Et `ToProcessor` til en prosessor som ikke finnes, er den klassiske feilen etter en navneendring. XML-en forblir velformet, gatewayen snubler først over det ved kjøretid, og da som regel med en lite nyttig melding.

## En bemerkning: Dette er kompilatorbygging, ikke hobbyarbeid

Det du gjør her, har et navn og en teori bak seg. Et regelverk er en **kontrollflytgraf**, altså den samme modellen som kompilatorer har brukt til å analysere programmer i flere tiår. Det er nyttig å vite, fordi ferdige algoritmer og, enda viktigere, klare utsagn om begrensningene deres dermed er tilgjengelige.

| Spørsmål i regelverket | Modell | Metode |
|---|---|---|
| Hvilke prosessorer er døde? | Tilgjengelighet fra inngangsnoden | Bredde- eller dybde-først-søk, kompleksitet `O(V+E)` |
| Hvilke regler i en kjede er døde? | Noder etter et ubetinget hopp | samme søk på en mer finkornet graf |
| Hvor kan en e-postsløyfe oppstå? | **Syklus i grafen** | sterkt sammenhengende komponenter |
| Hvor må en regel stå for å garantert slå til? | **Dominator** for inngangsnoden | dominatortre |

De to siste linjene er de mest verdifulle i praksis. En e-postsløyfe er ikke et mystisk driftsfenomen, men en syklus i rutinggrafen; hopptelleren ved kjøretid er bare nødbremsen, strukturelt finner du sløyfen på forhånd. Og hvis du vil plassere en regel som **hver** melding må passere, for eksempel et filter for ikke-rutbare avsenderdomener, spør du etter en dominator. Det er ikke et smaksspørsmål, men kan beregnes.

### Finn sykler før de blir e-postsløyfer

Bredde-først-søket besvarer spørsmålet om død kode. For sløyfer trenger du dybde-først-søk, for der avslører en **tilbakekant** syklusen. Metoden er den klassiske trefargemarkeringen:

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

Et slikt funn er ikke bevis på en sløyfe, for kantene er bevoktet og blir kanskje aldri tatt samtidig. Men det er den komplette listen over steder der en kan oppstå, og det er nettopp dem du vil kjenne før en ombygging. Hopptelleren ved kjøretid er bare nødbremsen; her ser du konstruksjonen.

Like viktig er **grensen** for metoden. Kantene er bevoktet av Matchers, og de avhenger av meldingsinnholdet. Eksakt tilgjengelighet er derfor generelt uavgjørbar; analysen gir en overapproksimasjon. Dette gir en asymmetrisk utsagnskraft som du må kjenne:

- **«Utilgjengelig» er pålitelig.** Hvis ingen sti leder dit, kan ingen melding komme dit. Denne koden kan du slette.
- **«Tilgjengelig» betyr bare «strukturelt ikke utelukket».** Grafen sier ikke om en reell melding noen gang oppfyller betingelsene.

Analysen erstatter altså ikke testen, den reduserer testrommet. I praksis er det likevel en enorm gevinst: Av 38 prosessorer blir det 18 som du i det hele tatt må kontrollere.

Metoder fra maskinlæring, som Graph Neural Networks eller nodeinnbygginger, trenger du uttrykkelig ikke her. De lønner seg ved store grafer med ukjent struktur og statistiske mønstre. Et regelverk har noen dusin noder, fullstendig kjent struktur og deterministisk semantikk. Eksakte algoritmer er her ikke bare billigere, de gir bevis i stedet for sannsynligheter.

## Fallgruver ved maskinell behandling

Når du endrer et regelverk med skript, finnes det tre feil som oppstår pålitelig. Alle tre har jeg gjort selv.

**Klassikeren: det grådige mønsteret på tvers av prosessorgrenser.** Den som vil fjerne en prosessor med et regulært uttrykk, tyr nærliggende til:

```python
muster = re.compile(r'   <processor name="%s">.*?</processor>\n' % name, re.S)
```

Det er feil. I kjeden står det i hvert `ToProcessor`-Mailet et `<processor>ziel</processor>`, og det ikke-grådige `.*?` stopper akkurat der. Resultatet: Halve prosessoren blir fjernet, en rest av `</mailet>` og `</processor>` blir stående, og XML-en er ødelagt. Forankre i stedet på innrykket til sluttagen og kontroller tag-balansen mot:

```python
muster = re.compile(r'   <processor name="%s">.*?\n   </processor>\n' % name, re.S)

treffer = muster.search(daten)
assert treffer, "Prozessor nicht gefunden"
block = treffer.group(0)
assert block.count("<mailet ") == block.count("</mailet>"), "Tag-Bilanz stimmt nicht"
```

**Linjeslutt.** Konfigurasjonen bruker vanligvis CRLF. Les i Python med `rb`, normaliser til `\n` for behandlingen og skriv til slutt tilbake med CRLF. Den som glemmer dette, produserer en fil med blandede linjeslutt som, avhengig av produktet, avvises uten kommentar.

**Spesialtegn.** Hold filen i ren ASCII og skriv umlauter som tegnreferanser (`&#228;` for ä). Det sparer deg for enhver diskusjon om koding mellom redigeringsprogram, skript og gatewayens nettgrensesnitt.

Kontroller etter hver endring minst velformethet, uendrede linjeslutt og uendret antall prosessorer. Tre kontrollinjer sparer en tilbakerulling.

## Metoden for ombyggingen: Parallelltre med én veksel

Nå til selve nyoppbyggingen. Den nærliggende veien, å bygge om det eksisterende regelverket steg for steg, er den dårligste: Du kan ikke gå rent tilbake, og du kan ikke lenger lese den gamle tilstanden.

I stedet har parallelltreet vist seg å fungere godt:

**Trinn 1: Bygg opp et nytt tre ved siden av.** Opprett de nye prosessorene med et navnesuffiks, for eksempel `rootV2`, `incomingV2`, `outgoingV2`. Det gamle treet forblir fullstendig og uendret.

**Trinn 2: Én eneste veksel.** I begynnelsen av det eksisterende inngangspunktet står nøyaktig ett Mailet:

```xml
<processor name="root">
   <mailet class="ToProcessor" match="All">
      <processor>rootV2</processor>
   </mailet>
   <!-- alles Bisherige bleibt hier stehen, ist aber nun unerreichbar -->
</processor>
```

Dermed går all trafikk gjennom det nye treet. Det gamle er utilgjengelig, men fortsatt komplett til stede. **Returveien består i å fjerne disse tre linjene**, og det er forståelig i enhver situasjon, også for en person som ikke utførte ombyggingen.

**Trinn 3: Tilgjengelighet som godkjenning.** Kjør analysen fra verktøy 2 og kontroller tre punkter: Det nye inngangspunktet refereres nøyaktig én gang, alle nye prosessorer er tilgjengelige, og det gamle treet er fullstendig utilgjengelig. Det er et objektivt godkjenningskriterium i stedet for en visuell kontroll.

**Trinn 4: Rydd først opp etter at løsningen har bevist seg.** Når det nye treet er bekreftet i drift, fjern det gamle og stryk suffiksene. Først da mister du returveien i filen, og frem til da har du ikke hatt bruk for den.

For mellomtrinn som du vil observere, men ennå ikke aktivere, passer rene observasjons-Mailets: De logger, men endrer ikke rutingen. Slik samler du dataene som mangler for beslutningen, uten risiko.

## Bygg inn synlighet samtidig

Ved nyoppbyggingen lønner det seg å ta hensyn til to ting som senere utgjør forskjellen i drift.

**Kast aldri direkte i hovedkjeden.** Et Mailet som forkaster en melding, etterlater i meldingshistorikken bare opplysningen om at den ble slettet, uten begrunnelse. Forgren i stedet til en særskilt navngitt prosessor, for eksempel `dropNonRoutable`. Navnet alene vises i historikken og forteller allerede hva som skjer.

**Ikke all logging havner i meldingshistorikken.** Mange produkter har to mekanismer: Én for serverloggen og én for historikken som også supporten ser. Bare den andre er synlig i historikken. Den som kun setter den første, har riktignok logget, men i sporet står det fortsatt bare «melding slettet». Formuler historikkoppføringene i klartekst og nevn regelen: «bevisst forkastet av regelen for ikke-rutbare avsenderdomener, ingen leveringsfeil» sparer svært mange oppfølgingsspørsmål i drift.

## Klyngen er en del av oppgaven

Et punkt som regelmessig undervurderes: Kjører gatewayen på flere noder, må konfigurasjonen være lagret **identisk på alle noder og bestandig etter omstart**. Hvis den bare er aktiv på én node, avhenger atferden av hvilken node som behandler meldingen, og testene dine måler tilfeldigheter.

Særlig ubehagelig er tilfellet der en endring riktignok kjører, men ikke ble persistert. Da arbeider noden korrekt helt til den starter på nytt, og faller deretter tilbake til den gamle tilstanden. Kontroller derfor begge ting etter hver utrulling: Samme tilstand på alle noder, og at tilstanden overlever en omstart.

## Oppsummert

Behandle regelverket som en graf, ikke som en tekstfil. Et bredde-først-søk fra inngangspunktet skiller på noen få kodelinjer levende fra dødt, og analysen i kjedene finner i tillegg reglene som riktignok står der, men aldri nås etter en ubetinget utgang.

For selve ombyggingen er parallelltreet med én eneste veksel metoden med det beste forholdet mellom innsats og sikkerhet. Og tilgjengelighetsanalysen gir deg samtidig godkjenningskriteriet.

## Kilder

1.  [Apache James: Mailet container configuration](https://james.apache.org/server/config-mailetcontainer.html): Oppbygging av spoolmanageren, prosessorer, Mailets og Matchers samt behandlingsrekkefølgen.

2.  [Apache James: Provided mailets](https://james.apache.org/server/dev-provided-mailets.html): Referanse for de medfølgende Mailets, inkludert ToProcessor og parametrene for videresending og konsum.

3.  [Apache James: Provided matchers](https://james.apache.org/server/dev-provided-matchers.html): Referanse for Matchers, blant annet All, HostIsLocal og de mottakerrelaterte variantene.

4.  [Apache James: Mailet API](https://james.apache.org/server/dev-mailet-api.html): Kontrakten mellom Mailet og container, grunnlaget for å forstå konsum og videresending.
