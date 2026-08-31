---
title: "Strukturert nyoppbygging av Apache James-regelverk: Verktøy og metode"
navTitle: "Bygg regelverket på nytt"
description: "Mailet-regelverk som har vokst over år, inneholder døde stier som ingen lenger kjenner igjen. Slik evaluerer du regelverket som en graf, finner utilgjengelig kode pålitelig og utformer ombyggingen slik at ett enkelt Mailet holder returveien åpen."
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
translationSourceHash: ebcf5bf98f1f74aa7784c74c558da4db240e69f02de722a0251dd832d1224403
translationModel: gpt-5.6-terra
translatedAt: 2026-08-31T10:04:12.724Z
translationReview: automatic
url: https://rafaelpfister.ch/no/blog/strukturert-gjenoppbygging-av-apache-james-regelverk-verktoy-og-metode
---

# Strukturert nyoppbygging av Apache James-regelverk: Verktøy og metode

E-postgatewayer basert på Apache James, deriblant Totemomail, styrer hele meldingsflyten via et regelverk i XML. Etter noen års drift får dette regelverket en egenskap som knapt noen legger merke til: En betydelig del av det blir aldri kjørt. Regler ble lagt til, avgreninger ble plassert foran, grener endte blindt, og siden ingenting gikk i stykker, ble alt stående.

Problemet er ikke diskplassen. Det er at ingen lenger kan si hvilken regel som faktisk slår inn. Den som planlegger en endring, leser en fil med hundrevis av Mailets og vet ikke hvilke av dem som i det hele tatt er relevante. Nettopp dette kan besvares mekanisk.

Denne artikkelen beskriver metoden og verktøyene for dette: Evaluere regelverket som en rettet graf, finne utilgjengelig kode pålitelig og utforme ombyggingen slik at ett enkelt Mailet holder returveien åpen.

## Modellen i fire setninger

Et regelverk består av **prosessorer**, altså navngitte kjeder. Hver kjede inneholder **Mailets** som gjør noe, og hvert Mailet har en **Matcher** som avgjør om det gjelder for den aktuelle meldingen. Et Mailet av klassen `ToProcessor` sender meldingen videre til en annen kjede.

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

Før du måler, må du vite hva du leter etter. Det finnes to former, og den andre er den lumskeste.

**Utilgjengelige prosessorer.** Hele kjeder som ingen `ToProcessor` lenger peker til. De står i filen, men blir aldri åpnet. Dette er det åpenbare tilfellet.

**Død rest innenfor en kjede.** Et `ToProcessor` med `match="All"` gjelder for **hver** melding og sender den videre. Alt som står under det i samme kjede, blir aldri nådd. Det samme gjelder Mailets med `passThrough=false`: De konsumerer meldingen og overtar den videre behandlingen selv; de etterfølgende Mailets får ikke se den.

Denne andre formen finner ikke et enkelt tekstsøk, for linjene ser helt normale ut. Du trenger rekkefølgen innenfor kjeden.

## Verktøy 1: Lese ut grafen

Utgangspunktet er en evaluering som trekker ut prosessorer og målene deres. Følgende skript bruker bare standardbiblioteket og kjører på enhver Python-installasjon:

```python
import re
import xml.dom.minidom

PFAD = "regelwerk.xml"

daten = open(PFAD, "rb").read().decode("utf-8").replace("\r\n", "\n")

# Del opp prosessorer og blokkene deres
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

Merk forskjellen mellom **definisjonstagen** `<processor name="...">` og **måltagen** `<processor>name</processor>` innenfor et `ToProcessor`-Mailet. Begge heter det samme, men betyr ulike ting. Den som forveksler dem, får meningsløse resultater. Det er også dette feilkilden lenger ned bygger på.

## Verktøy 2: Tilgjengelighet fra inngangspunktet

Med grafen er analysen et bredde-først-søk fra `root`. Alt som ikke besøkes, er dødt:

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

En typisk utdata fra et regelverk som har vokst over tid:

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

Tjue av 38 prosessorer med til sammen over 160 Mailets som aldri blir kjørt. Det er ikke et unntak, men normalen i et miljø som har gjennomgått flere ombygginger.

## Verktøy 3: Finne den døde resten innenfor kjedene

Nå den andre formen. Gå gjennom hver tilgjengelige kjede Mailet for Mailet, og marker alt etter den første ubetingede avgangen:

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

Dette funnet er mer verdifullt enn prosessorlisten, fordi det sitter midt i aktive kjeder. Den som legger til en regel og setter den under et `ToProcessor match="All"`, har skrevet en regel som aldri slår inn, og undrer seg deretter over at den ikke virker.

## Verktøy 4: Strukturkontroll

Velformet XML alene er ikke nok. Disse fire kontrollene fanger opp feilene som en parser slipper gjennom, men gatewayen ikke gjør:

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

Et `ToProcessor` til en prosessor som ikke finnes, er den klassiske feilen etter en navneendring. XML-en forblir velformet, men gatewayen feiler først under kjøring, og da som regel med en lite hjelpsom melding.

## Kontekst: Dette er kompilatorbygging

Det du gjør her, har et navn og en teori bak seg. Et regelverk er en **kontrollflytgraf**, altså den samme modellen som kompilatorer har brukt til å analysere programmer i flere tiår. Det er nyttig å vite, fordi ferdige algoritmer og, enda viktigere, klare utsagn om grensene deres da er tilgjengelige.

| Spørsmål i regelverket | Modell | Metode |
|---|---|---|
| Hvilke prosessorer er døde? | Tilgjengelighet fra inngangsnoden | Bredde- eller dybde-først-søk, kompleksitet `O(V+E)` |
| Hvilke regler i en kjede er døde? | Noder etter et ubetinget hopp | det samme søket på en finere graf |
| Hvor kan en e-postsløyfe oppstå? | **Syklus i grafen** | sterkt sammenhengende komponenter |
| Hvor må en regel stå for at den garantert slår inn? | **Dominator** for inngangsnoden | dominatortre |

De to siste radene er de mest verdifulle i praksis. En e-postsløyfe er ikke et mystisk driftsfenomen, men en syklus i rutinggrafen; hopptelleren under kjøring er bare nødbremsen, strukturelt finner du sløyfen på forhånd. Og hvis du vil plassere en regel som **hver** melding må passere, for eksempel et filter for ikke-rutbare avsenderdomener, spør du etter en dominator. Det er ikke et spørsmål om smak, men kan beregnes.

### Finn sykluser før de blir e-postsløyfer

Bredde-først-søket besvarer spørsmålet om død kode. For sløyfer trenger du dybde-først-søk, for der viser en **tilbakekant** syklusen. Metoden er den klassiske trefargemarkeringen:

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

Et slikt funn er ikke bevis på en sløyfe, for kantene er beskyttet av betingelser og blir kanskje aldri tatt samtidig. Men det er den fullstendige listen over steder der en kan oppstå, og nettopp dem vil du kjenne før en ombygging. Hopptelleren under kjøring er bare nødbremsen; her ser du konstruksjonen.

Like viktig er **grensen** for metoden. Kantene er beskyttet av Matchers, og de avhenger av meldingsinnholdet. Eksakt tilgjengelighet er dermed generelt uavgjørbar; analysen gir en overapproksimasjon. Dette innebærer en asymmetrisk utsagnskraft som du må kjenne:

- **«Utilgjengelig» er pålitelig.** Hvis ingen sti leder dit, kan ingen melding komme dit. Denne koden kan du slette.
- **«Tilgjengelig» betyr bare «strukturelt ikke utelukket».** Grafen sier ikke om noen reell melding noen gang oppfyller betingelsene.

Analysen erstatter altså ikke testing, men den reduserer testrommet. I praksis er dette likevel en enorm gevinst: Av 38 prosessorer blir det 18 som du i det hele tatt må kontrollere.

Metoder fra maskinlæring, som Graph Neural Networks eller nodeinnleiringer, trenger du uttrykkelig ikke her. De lønner seg for store grafer med ukjent struktur og statistiske mønstre. Et regelverk har noen dusin noder, fullt kjent struktur og deterministisk semantikk. Eksakte algoritmer er her ikke bare billigere, de leverer bevis i stedet for sannsynligheter.

## Feilkilder ved maskinell bearbeiding

Hvis du endrer et regelverk med et skript, finnes det tre feil som oppstår pålitelig. Alle tre har jeg gjort selv.

**Det grådige mønsteret på tvers av prosessorgrenser.** Den som vil fjerne en prosessor med et regulært uttrykk, tyr nærliggende til:

```python
muster = re.compile(r'   <processor name="%s">.*?</processor>\n' % name, re.S)
```

Det er feil. Inne i kjeden står det i hvert `ToProcessor`-Mailet et `<processor>ziel</processor>`, og det ikke-grådige `.*?` stopper akkurat der. Resultatet: Halve prosessoren fjernes, en rest av `</mailet>` og `</processor>` blir stående, og XML-en er ødelagt. Forankre i stedet på innrykkingen til den avsluttende taggen og kontroller taggbalansen mot:

```python
muster = re.compile(r'   <processor name="%s">.*?\n   </processor>\n' % name, re.S)

treffer = muster.search(daten)
assert treffer, "Prozessor nicht gefunden"
block = treffer.group(0)
assert block.count("<mailet ") == block.count("</mailet>"), "Tag-Bilanz stimmt nicht"
```

**Linjeslutt.** Konfigurasjonen bruker vanligvis CRLF. Les i Python med `rb`, normaliser til `\n` for bearbeiding og skriv tilbake CRLF til slutt. Den som glemmer det, produserer en fil med blandede linjeslutt som avhengig av produktet avvises uten kommentar.

**Spesialtegn.** Hold filen i ren ASCII og skriv umlauter som tegnreferanser (`&#228;` for ä). Det sparer deg for enhver diskusjon om tegnkodinger mellom redigeringsprogram, skript og gatewayens webgrensesnitt.

Kontroller etter hver endring minst velformethet, uendrede linjeslutt og uendret antall prosessorer. Tre linjer med kontroll sparer en tilbakerulling.

## Metoden for ombyggingen: Parallelltre med én veksling

Så til selve nyoppbyggingen. Den nærliggende veien, å bygge om det eksisterende regelverket trinn for trinn, er den dårligste: Du kan ikke enkelt gå tilbake, og du kan ikke lenger lese den gamle tilstanden.

I stedet har parallelltreet vist seg å fungere godt:

**Trinn 1: Bygg opp et nytt tre ved siden av.** Opprett de nye prosessorene med et navnesuffiks, for eksempel `rootV2`, `incomingV2`, `outgoingV2`. Det gamle treet blir stående helt og uendret.

**Trinn 2: Én eneste veksling.** I starten av det hittilværende inngangspunktet står nøyaktig ett Mailet:

```xml
<processor name="root">
   <mailet class="ToProcessor" match="All">
      <processor>rootV2</processor>
   </mailet>
   <!-- alles Bisherige bleibt hier stehen, ist aber nun unerreichbar -->
</processor>
```

Dermed går all trafikk gjennom det nye treet. Det gamle er utilgjengelig, men finnes fortsatt i sin helhet. **Returveien består i å fjerne disse tre linjene**, og det er forståelig i enhver situasjon, også for en person som ikke har utført ombyggingen.

**Trinn 3: Tilgjengelighet som godkjenning.** Kjør analysen fra verktøy 2 og kontroller tre punkter: Det nye inngangspunktet refereres nøyaktig én gang, alle nye prosessorer er tilgjengelige, og det gamle treet er fullstendig utilgjengelig. Dette er et objektivt godkjenningskriterium i stedet for en visuell kontroll.

**Trinn 4: Rydd opp først etter at det har bevist seg.** Når det nye treet er bekreftet i drift, fjern det gamle og stryk suffiksene. Først da mister du returveien i filen, og frem til da har du ikke hatt behov for den.

For mellomtrinn som du vil observere, men ennå ikke aktivere, passer rene observasjons-Mailets: De logger, men endrer ikke rutingen. Slik samler du dataene som mangler for beslutningen, uten risiko.

## Bygg inn synlighet samtidig

Ved nyoppbyggingen lønner det seg å ta hensyn til to ting som senere utgjør forskjellen i drift.

**Kast aldri direkte i hovedkjeden.** Et Mailet som forkaster en melding, etterlater bare en indikasjon i meldingshistorikken på at den ble slettet, uten begrunnelse. Forgren i stedet til en særskilt navngitt prosessor, for eksempel `dropNonRoutable`. Navnet alene vises i historikken og sier allerede hva som skjer.

**Ikke all logging havner i meldingshistorikken.** Mange produkter kjenner to mekanismer: én for serverloggen og én for historikken som også support ser. Bare den andre er synlig i historikken. Den som utelukkende bruker den første, har riktignok logget, men i sporet står det fortsatt bare «Melding slettet». Formuler historikkoppføringene i klartekst og nevn regelen: «bevisst forkastet av regelen for ikke-rutbare avsenderdomener, ingen leveringsfeil» sparer svært mange oppfølgingsspørsmål i drift.

## Klyngen er en del av oppgaven

Et punkt som regelmessig undervurderes: Kjører gatewayen på flere noder, må konfigurasjonen være **identisk på alle noder og vedvarende etter omstart**. Hvis den bare er aktiv på én node, avhenger oppførselen av hvilken node som behandler meldingen, og testene dine måler tilfeldigheter.

Særlig ubehagelig er tilfellet der en endring riktignok kjører, men ikke ble persistert. Da arbeider noden korrekt til den startes på nytt, og faller deretter tilbake til den gamle tilstanden. Kontroller derfor begge ting etter hver utrulling: Samme tilstand på alle noder, og at tilstanden overlever en omstart.

## Oppsummert

Behandle regelverket som en graf, ikke som en tekstfil. Et bredde-først-søk fra inngangspunktet skiller levende fra dødt på noen få kodelinjer, og analysen innenfor kjedene finner i tillegg reglene som riktignok står der, men aldri nås etter en ubetinget avgang.

For selve ombyggingen er parallelltreet med én enkelt veksling metoden med best forhold mellom innsats og sikkerhet. Og tilgjengelighetsanalysen gir deg samtidig godkjenningskriteriet for dette.

## Kilder

1.  [Apache James: Mailet container configuration](https://james.apache.org/server/config-mailetcontainer.html): Oppbygning av spoolmanageren, prosessorer, Mailets og Matchers samt behandlingsrekkefølgen.

2.  [Apache James: Provided mailets](https://james.apache.org/server/dev-provided-mailets.html): Referanse for de medfølgende Mailets, inkludert ToProcessor og parameterne for videresending og konsum.

3.  [Apache James: Provided matchers](https://james.apache.org/server/dev-provided-matchers.html): Referanse for Matchers, blant annet All, HostIsLocal og de mottakerrelaterte variantene.

4.  [Apache James: Mailet API](https://james.apache.org/server/dev-mailet-api.html): Kontrakten mellom Mailet og containeren, grunnlaget for å forstå konsum og videresending.
