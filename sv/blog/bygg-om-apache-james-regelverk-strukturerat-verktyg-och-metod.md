---
title: "Bygg om Apache James-regelverk strukturerat: verktyg och metod"
navTitle: "Bygg om regelverk"
description: "Etablerade Mailet-regelverk innehåller efter flera år döda vägar som ingen längre känner igen. Så utvärderar du regelverket som en graf, hittar otillgänglig kod på ett tillförlitligt sätt och utformar ombyggnaden så att ett enda Mailet håller vägen tillbaka öppen."
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
translationSourceHash: ebcf5bf98f1f74aa7784c74c558da4db240e69f02de722a0251dd832d1224403
translationModel: gpt-5.6-terra
translatedAt: 2026-08-31T10:03:29.613Z
translationReview: automatic
url: https://rafaelpfister.ch/sv/blog/bygg-om-apache-james-regelverk-strukturerat-verktyg-och-metod
---

# Bygg om Apache James-regelverk strukturerat: verktyg och metod

Mailgateways baserade på Apache James, däribland Totemomail, styr hela sitt meddelandeflöde via ett regelverk i XML. Efter några års drift får detta regelverk en egenskap som få lägger märke till: En betydande del av det körs aldrig. Regler tillkom, växlar placerades före dem, grenar ledde ingenstans och eftersom inget gick sönder blev allt kvar.

Problemet är inte diskutrymmet. Det är att ingen längre kan säga vilken regel som faktiskt träffar. Den som planerar en ändring läser en fil med hundratals Mailets och vet inte vilka som över huvud taget är relevanta. Just det kan besvaras mekaniskt.

Den här artikeln beskriver metoden och verktygen för det: utvärdera regelverket som en riktad graf, hitta otillgänglig kod på ett tillförlitligt sätt och utforma ombyggnaden så att ett enda Mailet håller vägen tillbaka öppen.

## Modellen i fyra meningar

Ett regelverk består av **processorer**, alltså namngivna kedjor. Varje kedja innehåller **Mailets** som utför något, och varje Mailet har en **Matcher** som avgör om det gäller det aktuella meddelandet. Ett Mailet av klassen `ToProcessor` överlämnar meddelandet till en annan kedja.

Ingångspunkten heter vanligen `root`. Därifrån förgrenas allt annat.

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

Därmed är strukturen en riktad graf: processorer är noder, `ToProcessor`-mål är kanter. Och så snart du ser det på det sättet blir frågan om död kod en standarduppgift, nämligen en nåbarhetsanalys.

## Två typer av död kod

Innan du mäter måste du veta vad du söker. Det finns två former, och den andra är den lömska.

**Otillgängliga processorer.** Hela kedjor som inget `ToProcessor` längre pekar på. De finns i filen, men används aldrig. Det är det uppenbara fallet.

**Död rest inom en kedja.** Ett `ToProcessor` med `match="All"` träffar **varje** meddelande och skickar det vidare. Allt som står längre ned i samma kedja nås aldrig. Detsamma gäller Mailets med `passThrough=false`: de konsumerar meddelandet och tar själva över den fortsatta behandlingen, så efterföljande Mailets ser det inte längre.

Den andra formen hittas inte med en enkel textsökning, eftersom raderna ser helt normala ut. Du behöver ordningen inom kedjan för det.

## Verktyg 1: Läs ut grafen

Utgångspunkten är en utvärdering som extraherar processorer och deras mål. Följande skript använder endast standardbiblioteket och körs på varje Python-installation:

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

Observera skillnaden mellan **definitionstaggen** `<processor name="...">` och **måltaggen** `<processor>name</processor>` i ett `ToProcessor`-Mailet. Båda heter likadant, men betyder olika saker. Den som blandar ihop dem får meningslösa resultat. Det är också precis detta som ligger bakom felkällan längre ned.

## Verktyg 2: Nåbarhet från ingångspunkten

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

En typisk utdata från ett etablerat regelverk:

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

Tjugo av 38 processorer med sammanlagt över 160 Mailets som aldrig körs. Det är ingen avvikelse, utan normalfallet i en miljö som har genomgått flera ombyggnader.

## Verktyg 3: Hitta den döda resten inom kedjorna

Nu till den andra formen. Gå igenom varje nåbar kedja Mailet för Mailet och markera allt efter den första ovillkorliga övergången:

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

Detta fynd är mer värdefullt än processorlistan, eftersom det finns mitt i aktiva kedjor. Den som lägger till en regel och placerar den under ett `ToProcessor match="All"` skriver en regel som aldrig träffar och undrar sedan varför den inte har någon effekt.

## Verktyg 4: Strukturkontroll

Välformad XML räcker inte i sig. Dessa fyra kontroller fångar upp de fel som en parser släpper igenom, men som gatewayen inte gör:

```python
dom = xml.dom.minidom.parseString(daten.replace("\n", "\r\n").encode("utf-8"))

namen = [p.getAttribute("name")
         for p in dom.getElementsByTagName("processor")
         if p.getAttribute("name")]

# 1. Dubbla processornamn
doppelt = {n for n in namen if namen.count(n) > 1}

# 2. Varje Mailet måste vara direkt barn till en processor och ha class + match
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

Ett `ToProcessor` till en processor som inte finns är det klassiska felet efter ett namnbyte. XML:en förblir välformad, gatewayen misslyckas först vid körning och då oftast med ett föga hjälpsamt meddelande.

## Kontext: detta är kompilatorbygge

Det du gör här har ett namn och en teori bakom sig. Ett regelverk är en **kontrollflödesgraf**, alltså samma modell som kompilatorer har använt för att analysera program i årtionden. Det är värt att känna till, eftersom det ger tillgång till färdiga algoritmer och, viktigare, tydliga utsagor om deras gränser.

| Fråga i regelverket | Modell | Metod |
|---|---|---|
| Vilka processorer är döda? | Nåbarhet från ingångsnoden | Bredd- eller djupförstsökning, komplexitet `O(V+E)` |
| Vilka regler i en kedja är döda? | Noder efter ett ovillkorligt hopp | samma sökning på en finare graf |
| Var kan en mail-loop uppstå? | **Cykel i grafen** | starkt sammanhängande komponenter |
| Var måste en regel placeras för att garanterat träffa? | **Dominator** för ingångsnoden | dominatorträd |

De två sista raderna är de mest praktiskt värdefulla. En mail-loop är inte ett mystiskt driftsfenomen utan en cykel i routinggrafen; hoppräknaren vid körning är bara nödbromsen, strukturellt hittar du slingan i förväg. Och om du vill placera en regel som **varje** meddelande måste passera, exempelvis ett filter för icke-routbara avsändardomäner, ska du fråga efter en dominator. Det är ingen smaksak, utan beräkningsbart.

### Hitta cykler innan de blir mail-loops

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

Ett sådant fynd är inget bevis på en loop, eftersom kanterna bevakas och kanske aldrig tas tillsammans. Men det är den fullständiga listan över ställen där en kan uppstå, och just dem vill du känna till före en ombyggnad. Hoppräknaren vid körning är bara nödbromsen; här ser du konstruktionen.

Lika viktig är **gränsen** för metoden. Kanterna bevakas av Matchers, och de beror på meddelandeinnehållet. Exakt nåbarhet är därför i allmänhet oavgörbar, analysen ger en överapproximation. Av detta följer en asymmetrisk beviskraft som du måste känna till:

- **”Otillgänglig” är tillförlitligt.** Om ingen väg leder dit kan inget meddelande komma dit. Denna kod får du ta bort.
- **”Nåbar” betyder bara ”inte strukturellt utesluten”.** Grafen säger inte om något verkligt meddelande någonsin uppfyller villkoren.

Analysen ersätter alltså inte testet, men den minskar testutrymmet. I praktiken är det ändå en enorm vinst: Av 38 processorer återstår 18 som du alls behöver kontrollera.

Metoder från maskininlärning, exempelvis Graph Neural Networks eller nodinbäddningar, behöver du uttryckligen inte här. De lönar sig för stora grafer med okänd struktur och statistiska mönster. Ett regelverk har några dussin noder, fullständigt känd struktur och deterministisk semantik. Exakta algoritmer är här inte bara billigare, de ger bevis i stället för sannolikheter.

## Felkällor vid maskinell bearbetning

När du ändrar ett regelverk med skript finns det tre fel som uppträder pålitligt. Jag har själv gjort alla tre.

**Det giriga mönstret över processorgränser.** Den som vill ta bort en processor med ett reguljärt uttryck väljer naturligtvis:

```python
muster = re.compile(r'   <processor name="%s">.*?</processor>\n' % name, re.S)
```

Det är fel. Inom kedjan finns i varje `ToProcessor`-Mailet ett `<processor>ziel</processor>`, och det icke-giriga `.*?` stannar precis där. Resultatet: Halva processorn tas bort, en rest av `</mailet>` och `</processor>` blir kvar och XML:en förstörs. Förankra i stället vid indraget för den avslutande taggen och kontrollera taggbalansen mot:

```python
muster = re.compile(r'   <processor name="%s">.*?\n   </processor>\n' % name, re.S)

treffer = muster.search(daten)
assert treffer, "Prozessor nicht gefunden"
block = treffer.group(0)
assert block.count("<mailet ") == block.count("</mailet>"), "Tag-Bilanz stimmt nicht"
```

**Radslut.** Konfigurationen använder vanligen CRLF. Läs i Python med `rb`, normalisera till `\n` för bearbetningen och skriv tillbaka CRLF i slutet. Den som glömmer detta producerar en fil med blandade radslut, som beroende på produkt avvisas utan kommentar.

**Specialtecken.** Håll filen i ren ASCII och skriv umlauter som teckenreferenser (`&#228;` för ä). Det besparar dig varje diskussion om kodningar mellan redigerare, skript och gatewayens webbgränssnitt.

Kontrollera efter varje ändring minst välformning, oförändrade radslut och oförändrat antal processorer. Tre rader kontroll sparar en återställning.

## Metoden för ombyggnaden: parallellt träd med en växel

Nu till själva ombyggnaden. Den närliggande vägen, att bygga om det befintliga regelverket steg för steg, är den sämsta: Du kan inte återgå rent, och du kan inte längre läsa det gamla tillståndet.

I stället har det parallella trädet visat sig fungera väl:

**Steg 1: Bygg upp det nya trädet bredvid.** Skapa de nya processorerna med ett namnsuffix, exempelvis `rootV2`, `incomingV2`, `outgoingV2`. Det gamla trädet finns kvar helt och oförändrat.

**Steg 2: En enda växel.** I början av den tidigare ingångspunkten finns exakt ett Mailet:

```xml
<processor name="root">
   <mailet class="ToProcessor" match="All">
      <processor>rootV2</processor>
   </mailet>
   <!-- alles Bisherige bleibt hier stehen, ist aber nun unerreichbar -->
</processor>
```

Därmed går all trafik genom det nya trädet. Det gamla är otillgängligt, men fortfarande helt kvar. **Vägen tillbaka består i att ta bort dessa tre rader**, och det är begripligt i varje situation, även för en person som inte har gjort ombyggnaden.

**Steg 3: Nåbarhet som godkännande.** Kör analysen från verktyg 2 och kontrollera tre punkter: Den nya ingångspunkten refereras exakt en gång, alla nya processorer är nåbara och det gamla trädet är helt otillgängligt. Det är ett objektivt godkännandekriterium i stället för en visuell kontroll.

**Steg 4: Rensa först efter beprövad drift.** När det nya trädet har bekräftats i drift tar du bort det gamla och stryker suffixen. Först då förlorar du vägen tillbaka i filen, och fram till dess har du inte behövt den.

För mellansteg som du vill observera men ännu inte aktivera fullt ut passar rena observations-Mailets: De loggar, men ändrar inte routingen. På så sätt samlar du in de data som saknas för beslutet utan risk.

## Bygg in synlighet samtidigt

Vid nyuppbyggnaden är det värt att beakta två saker som senare gör skillnad i drift.

**Kasta aldrig direkt i huvudkedjan.** Ett Mailet som kastar ett meddelande lämnar bara informationen att det raderades i meddelandets historik, utan orsak. Förgrena i stället till en särskilt namngiven processor, exempelvis `dropNonRoutable`. Namnet i sig visas i historiken och talar redan om vad som har hänt.

**All loggning hamnar inte i meddelandehistoriken.** Många produkter har två mekanismer: en för serverloggen och en för historiken som även supporten ser. Endast den andra syns i historiken. Den som enbart använder den första har visserligen loggat, men i spårningen står fortfarande bara ”Meddelande raderat”. Formulera historikposterna på klarspråk och ange regeln: ”medvetet kastat genom regeln för icke-routbara avsändardomäner, inget leveransfel” sparar mycket efterfrågan i drift.

## Klustret är en del av uppgiften

En punkt som regelbundet underskattas: Kör gatewayen på flera noder måste konfigurationen vara **identisk på alla noder och beständig över omstarter**. Om den bara är aktiv på en nod beror beteendet på vilken nod som behandlar meddelandet, och dina tester mäter slumpen.

Särskilt obehagligt är fallet då en ändring körs men inte har sparats beständigt. Då fungerar noden korrekt tills den startas om och återgår därefter till det gamla läget. Kontrollera därför två saker efter varje driftsättning: samma version på alla noder, och att versionen överlever en omstart.

## Sammanfattning

Behandla regelverket som en graf, inte som en textfil. En breddförstsökning från ingångspunkten skiljer på några rader kod levande från dött, och analysen inom kedjorna hittar dessutom reglerna som visserligen står där men aldrig nås efter en ovillkorlig övergång.

För själva ombyggnaden är det parallella trädet med en enda växel metoden med bäst förhållande mellan arbetsinsats och säkerhet. Och nåbarhetsanalysen ger dig samtidigt godkännandekriteriet för den.

## Källor

1.  [Apache James: Mailet container configuration](https://james.apache.org/server/config-mailetcontainer.html): Struktur för spoolmanagern, processorer, Mailets och Matchers samt bearbetningsordningen.

2.  [Apache James: Provided mailets](https://james.apache.org/server/dev-provided-mailets.html): Referens för de medföljande Mailets, inklusive ToProcessor och parametrarna för vidarebefordran och konsumtion.

3.  [Apache James: Provided matchers](https://james.apache.org/server/dev-provided-matchers.html): Referens för Matchers, bland annat All, HostIsLocal och de mottagarrelaterade varianterna.

4.  [Apache James: Mailet API](https://james.apache.org/server/dev-mailet-api.html): Avtal mellan Mailet och container, grund för förståelsen av konsumtion och vidarebefordran.
