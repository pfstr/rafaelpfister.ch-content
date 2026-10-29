---
title: "Strukturera om Apache James-regelverk: verktyg och metod"
navTitle: "Bygg om regelverket"
description: "Vuxna Mailet-regelverk innehåller efter flera år döda vägar som ingen längre känner igen. Så analyserar du regelverket som en graf, hittar oåtkomlig kod på ett tillförlitligt sätt och utformar ombyggnaden så att ett enda Mailet håller vägen tillbaka öppen."
date: "2026-08-11"
kategorie: "Totemomail"
timeToRead: "16 min lästid"
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
slug: "bygg-om-apache-james-regelverk-strukturerat-verktyg-och-metod"
translationId: "article-b9c98459a0ff6352"
draft: false
translationOf: apache-james-ruleset-strukturiert-neu-aufbauen
translationSourceHash: acf116bf560f396b8a3f679fd2ca62a600e127ccf6cd7522c4a8534bb9876c0e
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:36:43.154Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/bygg-om-apache-james-regelverk-strukturerat-verktyg-och-metod
---

# Strukturera om Apache James-regelverk: verktyg och metod

E-postgatewayer baserade på Apache James, däribland TotemoMail, styr hela sitt meddelandeflöde via ett regelverk i XML. Efter några års drift får detta regelverk en egenskap som få märker: En betydande del av det körs aldrig. Regler tillkom, växlar placerades före dem, grenar ledde ingenstans, och eftersom inget gick sönder blev allt kvar.

Problemet är inte lagringsutrymmet. Det är att ingen längre kan säga vilken regel som faktiskt träffar. Den som planerar en ändring läser en fil med hundratals Mailets och vet inte vilka av dem som över huvud taget är relevanta. Just det kan besvaras mekaniskt.

Den här artikeln beskriver metoden och verktygen för detta: att analysera regelverket som en riktad graf, hitta oåtkomlig kod på ett tillförlitligt sätt och utforma ombyggnaden så att ett enda Mailet håller vägen tillbaka öppen.

## Modellen i fyra meningar

Ett regelverk består av **processorer**, alltså namngivna kedjor. Varje kedja innehåller **Mailets** som utför något, och varje Mailet har en **Matcher** som avgör om den gäller för det aktuella meddelandet. Ett Mailet av klassen `ToProcessor` överlämnar meddelandet till en annan kedja.

Startpunkten heter vanligtvis `root`. Därifrån förgrenar sig allt annat.

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

Därmed är strukturen en riktad graf: processorer är noder, `ToProcessor`-mål är kanter. Och så snart du ser det så är frågan om död kod en standarduppgift, nämligen en nåbarhetsanalys.

## Två typer av död kod

Innan du mäter måste du veta vad du letar efter. Det finns två former, och den andra är den lömska.

**Oåtkomliga processorer.** Hela kedjor som inget `ToProcessor` längre pekar på. De finns i filen, men används aldrig. Det är det uppenbara fallet.

**Död återstod inom en kedja.** Ett `ToProcessor` med `match="All"` träffar **varje** meddelande och skickar det vidare. Allt som står längre ned i samma kedja nås aldrig. Detsamma gäller Mailets med `passThrough=false`: de konsumerar meddelandet och tar över den fortsatta behandlingen själva, efterföljande Mailets ser det inte längre.

Den andra formen hittas inte med enkel textsökning, eftersom raderna ser helt normala ut. För den behöver du ordningen inom kedjan.

## Verktyg 1: Läs ut grafen

Utgångspunkten är en analys som extraherar processorer och deras mål. Följande skript använder bara standardbiblioteket och körs i varje Python-installation:

```python
import re
import xml.dom.minidom

PFAD = "regelwerk.xml"

daten = open(PFAD, "rb").read().decode("utf-8").replace("\r\n", "\n")

# Extrahera processorer och deras block
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

Observera skillnaden mellan **definitions-taggen** `<processor name="...">` och **mål-taggen** `<processor>name</processor>` i ett `ToProcessor`-Mailet. Båda heter likadant men betyder olika saker. Den som blandar ihop dem får meningslösa resultat. Det är också grunden till felkällan längre ned.

## Verktyg 2: Nåbarhet från startpunkten

Med grafen blir analysen en breddförstsökning från `root`. Allt som inte besöks är dött:

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

En typisk utdata från ett vuxet regelverk:

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

Tjugo av 38 processorer med sammanlagt över 160 Mailets som aldrig körs. Det är inget undantag, utan normalfallet i en miljö som har genomgått flera ombyggnader.

## Verktyg 3: Hitta den döda återstoden inom kedjorna

Nu den andra formen. Gå igenom varje nåbar kedja Mailet för Mailet och markera allt efter den första ovillkorliga utgången:

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

Det här resultatet är mer värdefullt än processorlistan, eftersom det ligger mitt i aktiva kedjor. Den som lägger till en regel och placerar den under ett `ToProcessor match="All"` har skrivit en regel som aldrig träffar och undrar sedan varför den inte har någon effekt.

## Verktyg 4: Strukturkontroll

Enbart välformad XML räcker inte. Dessa fyra kontroller fångar felen som en parser släpper igenom men som gatewayen inte gör:

```python
dom = xml.dom.minidom.parseString(daten.replace("\n", "\r\n").encode("utf-8"))

namen = [p.getAttribute("name")
         for p in dom.getElementsByTagName("processor")
         if p.getAttribute("name")]

# 1. Dubbla processornamn
doppelt = {n for n in namen if namen.count(n) > 1}

# 2. Varje Mailet måste vara ett direkt barn till en processor och ha class + match
fehler = []
for m in dom.getElementsByTagName("mailet"):
    if m.parentNode.tagName != "processor":
        fehler.append(("falscher Elternknoten", m.getAttribute("class")))
    if not m.getAttribute("class") or not m.getAttribute("match"):
        fehler.append(("class oder match fehlt", m.parentNode.getAttribute("name")))

# 3. Varje ToProcessor-mål måste finnas
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

Ett `ToProcessor` till en processor som inte finns är det klassiska felet efter ett namnbyte. XML:en förblir välformad, gatewayen misslyckas först vid körning, och då oftast med ett föga hjälpsamt meddelande.

## Klassificering: det här är kompilatorkonstruktion

Det du gör här har ett namn och en teori bakom sig. Ett regelverk är en **kontrollflödesgraf**, alltså samma modell som kompilatorer har använt för att analysera program i årtionden. Det är värt att känna till, eftersom färdiga algoritmer och, ännu viktigare, tydliga uttalanden om deras gränser då finns tillgängliga.

| Fråga i regelverket | Modell | Metod |
|---|---|---|
| Vilka processorer är döda? | Nåbarhet från startnoden | Bredd- eller djupförstsökning, komplexitet `O(V+E)` |
| Vilka regler i en kedja är döda? | Noder efter ett ovillkorligt hopp | samma sökning på en mer detaljerad graf |
| Var kan en e-postloop uppstå? | **Cykel i grafen** | starkt sammanhängande komponenter |
| Var måste en regel stå för att garanterat träffa? | **Dominator** för startnoden | dominatorträd |

De två sista raderna är de mest praktiskt värdefulla. En e-postloop är inget mystiskt driftfenomen, utan en cykel i routinggrafen; hoppräknaren vid körning är bara nödbromsen, strukturellt hittar du slingan i förväg. Och om du vill placera en regel som **varje** meddelande måste passera, exempelvis ett filter för avsändardomäner som inte kan routas, frågar du efter en dominator. Det är ingen smaksak utan beräkningsbart.

### Hitta cykler innan de blir e-postloopar

Breddförstsökningen besvarar frågan om död kod. För slingor behöver du djupförstsökning, eftersom en **bakåtkant** där visar cykeln. Metoden är den klassiska trefärgsmarkeringen:

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

Ett sådant fynd är inget bevis för en loop, eftersom kanterna bevakas och kanske aldrig tas tillsammans. Men det är den fullständiga listan över ställen där en loop **kan** uppstå, och det är precis dessa du vill känna till före en ombyggnad. Hoppräknaren vid körning är bara nödbromsen; här ser du konstruktionen.

Lika viktig är metodens **gräns**. Kanterna bevakas av Matchers, och de är beroende av meddelandeinnehållet. Exakt nåbarhet är därför i allmänhet oavgörbar, analysen ger en överapproximation. Därav följer en asymmetrisk utsagekraft som du måste känna till:

- **”Oåtkomlig” är tillförlitligt.** Om ingen väg leder dit kan inget meddelande komma dit. Den koden kan du ta bort.
- **”Nåbar” betyder bara ”strukturellt inte utesluten”.** Grafen säger inget om huruvida något verkligt meddelande någonsin uppfyller villkoren.

Analysen ersätter alltså inte testet, den minskar testutrymmet. I praktiken är det ändå en enorm vinst: Av 38 processorer återstår 18 som du över huvud taget behöver kontrollera.

Metoder från maskininlärning, såsom Graph Neural Networks eller nodinbäddningar, behöver du uttryckligen inte här. De lönar sig för stora grafer med okänd struktur och statistiska mönster. Ett regelverk har några dussin noder, helt känd struktur och deterministisk semantik. Exakta algoritmer är här inte bara billigare, de ger bevis i stället för sannolikheter.

## Felkällor vid maskinell bearbetning

Om du ändrar ett regelverk med skript finns det tre fel som uppstår pålitligt. Jag har själv gjort alla tre.

**Det giriga mönstret över processorgränser.** Den som vill ta bort en processor med ett reguljärt uttryck väljer naturligtvis:

```python
muster = re.compile(r'   <processor name="%s">.*?</processor>\n' % name, re.S)
```

Det är fel. Inom kedjan finns i varje `ToProcessor`-Mailet en `<processor>ziel</processor>`, och det icke-giriga `.*?` stannar exakt där. Resultatet: Halva processorn tas bort, en rest av `</mailet>` och `</processor>` blir kvar, och XML:en förstörs. Förankra i stället mot indragningen för den avslutande taggen och kontrollera taggbalansen mot:

```python
muster = re.compile(r'   <processor name="%s">.*?\n   </processor>\n' % name, re.S)

treffer = muster.search(daten)
assert treffer, "Prozessor nicht gefunden"
block = treffer.group(0)
assert block.count("<mailet ") == block.count("</mailet>"), "Tag-Bilanz stimmt nicht"
```

**Radslut.** Konfigurationen använder vanligtvis CRLF. Läs i Python med `rb`, normalisera till `\n` för bearbetningen och skriv tillbaka CRLF i slutet. Den som glömmer detta producerar en fil med blandade radslut, som beroende på produkt avvisas utan kommentar.

**Specialtecken.** Håll filen i ren ASCII och skriv umlauter som teckenreferenser (`&#228;` för ä). Det besparar dig varje diskussion om kodningar mellan editor, skript och gatewayens webbgränssnitt.

Kontrollera efter varje ändring åtminstone välformning, oförändrade radslut och oförändrat antal processorer. Tre rader kontroll sparar en återställning.

## Metoden för ombyggnaden: parallellt träd med en växel

Nu till själva ombyggnaden. Den närliggande vägen, att bygga om det befintliga regelverket steg för steg, är den sämsta: Du kan inte enkelt återgå och du kan inte längre läsa det gamla tillståndet.

I stället har det parallella trädet visat sig fungera:

**Steg 1: Bygg det nya trädet bredvid.** Skapa de nya processorerna med ett namnsuffix, exempelvis `rootV2`, `incomingV2`, `outgoingV2`. Det gamla trädet förblir helt och oförändrat.

**Steg 2: En enda växel.** I början av den tidigare startpunkten finns exakt ett Mailet:

```xml
<processor name="root">
   <mailet class="ToProcessor" match="All">
      <processor>rootV2</processor>
   </mailet>
   <!-- alles Bisherige bleibt hier stehen, ist aber nun unerreichbar -->
</processor>
```

Därmed går all trafik genom det nya trädet. Det gamla är oåtkomligt men finns kvar i sin helhet. **Vägen tillbaka består i att ta bort dessa tre rader**, och det är begripligt i varje situation, även för en person som inte har gjort ombyggnaden.

**Steg 3: Nåbarhet som acceptanskontroll.** Kör analysen från verktyg 2 och kontrollera tre punkter: Den nya startpunkten refereras exakt en gång, alla nya processorer är nåbara och det gamla trädet är helt oåtkomligt. Det är ett objektivt acceptanskriterium i stället för en visuell kontroll.

**Steg 4: Städa först efter beprövad drift.** När det nya trädet har bekräftats i drift tar du bort det gamla och stryker suffixen. Först då förlorar du vägen tillbaka i filen, och fram till dess har du inte behövt den.

För mellansteg som du vill observera men ännu inte aktivera skarpt passar rena observations-Mailets: De loggar, men ändrar inte routingen. På så sätt samlar du in de data som saknas för beslutet utan risk.

## Bygg in synlighet direkt

Vid ombyggnaden är det värt att ta hänsyn till två saker som senare gör stor skillnad i drift.

**Kassera aldrig direkt i huvudkedjan.** Ett Mailet som kasserar ett meddelande lämnar i meddelandehistoriken bara en uppgift om att det raderades, utan orsak. Förgrena i stället till en särskilt namngiven processor, exempelvis `dropNonRoutable`. Bara namnet visas i historiken och säger redan vad som har hänt.

**All loggning hamnar inte i meddelandehistoriken.** Många produkter har två mekanismer: en för serverloggen och en för historiken som även supporten ser. Bara den andra syns i historiken. Den som enbart använder den första har visserligen loggat, men i spårningen står fortfarande bara ”Meddelande raderat”. Formulera historikposterna tydligt och ange regeln: ”medvetet kasserat av regeln för avsändardomäner som inte kan routas, inget leveransfel” sparar mycket återkoppling i driften.

## Klustret är en del av uppgiften

En punkt som regelbundet underskattas: Om gatewayen körs på flera noder måste konfigurationen finnas **identiskt på alla noder och vara beständig efter omstart**. Om den bara är aktiv på en nod beror beteendet på vilken nod som behandlar meddelandet, och dina tester mäter slumpen.

Särskilt obehagligt är fallet då en ändring visserligen körs men inte har sparats permanent. Noden fungerar då korrekt tills den startas om och återgår därefter till det gamla läget. Kontrollera därför båda sakerna efter varje driftsättning: samma status på alla noder och att statusen överlever en omstart.

## Sammanfattning

Behandla regelverket som en graf, inte som en textfil. En breddförstsökning från startpunkten skiljer på några få kodrader det levande från det döda, och analysen inom kedjorna hittar dessutom reglerna som visserligen står där men aldrig nås efter en ovillkorlig utgång.

För själva ombyggnaden är det parallella trädet med en enda växel metoden med bäst förhållande mellan arbete och säkerhet. Och nåbarhetsanalysen ger dig samtidigt acceptanskriteriet för den.

## Ändringshistorik

- **25 oktober 2026:** Källänkar uppdaterade till de nya adresserna för Apache James-dokumentationen; referenserna till Mailets och Matchers har nu sammanfattats i en källa.

## Källor

1.  [Apache James: Mailet container configuration](https://james.apache.org/server/config-mailetcontainer.html): Struktur för spoolhanteraren, processorer, Mailets och Matchers samt bearbetningsordningen.

2.  [Apache James: Mailets and Matchers Reference](https://james.apache.org/mailet/standard/mailet-report.html): Referens för medföljande Mailets, inklusive ToProcessor, och Matchers, bland annat All, HostIsLocal och RecipientIs.

3.  [Apache James: Mailet API](https://james.apache.org/mailet/api/index.html): Beskrivning av Mailet-gränssnittet och körningen i Mailet-containern, som grund för att förstå konsumtion och vidarebefordran.
