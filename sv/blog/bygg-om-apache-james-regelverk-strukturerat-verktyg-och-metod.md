---
title: "Bygg om Apache James-regelverk strukturerat: verktyg och metod"
navTitle: "Bygg om regelverket"
description: "Etablerade Mailet-regelverk innehåller efter åratal döda vägar som ingen längre känner igen. Så analyserar du regelverket som en graf, hittar oåtkomlig kod på ett tillförlitligt sätt och bygger om det så att ett enda Mailet håller vägen tillbaka öppen."
date: "2026-08-11"
kategorie: "TotemoMail"
timeToRead: "16 min läsning"
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
url: https://rafaelpfister.ch/sv/blog/bygg-om-apache-james-regelverk-strukturerat-verktyg-och-metod
translationSourceHash: b0274af954ad40614bc74b37b7be1e6e9bee6c856e28105336eddfb967895884
translationModel: gpt-5.6-terra
translatedAt: 2026-08-12T05:11:01.718Z
translationReview: automatic
---

# Bygg om Apache James-regelverk strukturerat: verktyg och metod

Mailgateways baserade på Apache James, däribland TotemoMail, styr hela sitt meddelandeflöde via ett regelverk i XML. Efter några års drift får detta regelverk en egenskap som knappt någon märker: En betydande del av det körs aldrig. Regler lades till, växlar sattes före dem, grenar ledde ingenstans och eftersom inget gick sönder fick allt stå kvar.

Problemet är inte diskutrymmet. Det är att ingen längre kan säga vilken regel som faktiskt träffar. Den som planerar en ändring läser en fil med hundratals Mailets och vet inte vilka av dem som ens är relevanta. Just det går att besvara mekaniskt.

Den här artikeln beskriver metoden och verktygen för detta: analysera regelverket som en riktad graf, hitta oåtkomlig kod på ett tillförlitligt sätt och utforma ombyggnaden så att ett enda Mailet håller vägen tillbaka öppen.

## Modellen i fyra meningar

Ett regelverk består av **processorer**, alltså namngivna kedjor. Varje kedja innehåller **Mailets** som gör något, och varje Mailet har en **Matcher** som avgör om det gäller det aktuella meddelandet. Ett Mailet av klassen `ToProcessor` överlämnar meddelandet till en annan kedja.

Startpunkten heter vanligtvis `root`. Därifrån förgrenas allt annat.

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

Strukturen är därmed en riktad graf: processorer är noder, `ToProcessor`-mål är kanter. Och så snart du ser det så är frågan om död kod en standarduppgift, nämligen en räckviddsanalys.

## Två typer av död kod

Innan du mäter måste du veta vad du letar efter. Det finns två former, och den andra är den lömska.

**Oåtkomliga processorer.** Hela kedjor som inget `ToProcessor` längre pekar på. De finns i filen, men nås aldrig. Det är det uppenbara fallet.

**Död återstod inom en kedja.** Ett `ToProcessor` med `match="All"` träffar **varje** meddelande och skickar det vidare. Allt som står nedanför i samma kedja nås aldrig. Detsamma gäller Mailets med `passThrough=false`: de konsumerar meddelandet och tar själva över den fortsatta hanteringen, så efterföljande Mailets ser det inte.

Den här andra formen hittar ingen enkel textsökning, eftersom raderna ser helt normala ut. För detta behöver du ordningen inom kedjan.

## Verktyg 1: Läs ut grafen

Första steget är en analys som extraherar processorer och deras mål. Följande skript använder bara standardbiblioteket och körs på varje Python-installation:

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

Observera skillnaden mellan **definitionstaggen** `<processor name="...">` och **måltaggen** `<processor>name</processor>` inom ett `ToProcessor`-Mailet. Båda heter likadant, men betyder olika saker. Den som förväxlar dem får meningslösa resultat. Det är också grunden till fallgropen längre ned.

## Verktyg 2: Räckvidd från startpunkten

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

Tjugo av 38 processorer med sammanlagt över 160 Mailets som aldrig körs. Det är inget undantag, utan normalfallet i en miljö som har genomgått flera ombyggnader.

## Verktyg 3: Hitta den döda återstoden inom kedjorna

Nu den andra formen. Gå igenom varje åtkomlig kedja Mailet för Mailet och markera allt efter den första ovillkorliga utgången:

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

Detta resultat är mer värdefullt än processorlistan, eftersom det sitter mitt i aktiva kedjor. Den som lägger till en regel och placerar den under ett `ToProcessor match="All"` har skrivit en regel som aldrig träffar och undrar sedan över att den inte har någon effekt.

## Verktyg 4: Strukturkontroll

Välformad XML är bara halva jobbet. Dessa fyra kontroller fångar felen som en parser släpper igenom men som gatewayen inte gör:

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

Ett `ToProcessor` som pekar på en processor som inte finns är det klassiska felet efter en namnändring. XML:en förblir välformad, gatewayen snubblar först över det vid körning och då oftast med ett föga hjälpsamt meddelande.

## En invändning: detta är kompilatorkonstruktion, inte pyssel

Det du gör här har ett namn och en teori bakom sig. Ett regelverk är en **kontrollflödesgraf**, alltså samma modell som kompilatorer har använt för att analysera program i årtionden. Det är värt att veta, eftersom färdiga algoritmer och, viktigare, tydliga utsagor om deras begränsningar då finns tillgängliga.

| Fråga i regelverket | Modell | Metod |
|---|---|---|
| Vilka processorer är döda? | Räckvidd från startnoden | Bredd- eller djupförstsökning, komplexitet `O(V+E)` |
| Vilka regler i en kedja är döda? | Noder efter ett ovillkorligt hopp | samma sökning på en finare graf |
| Var kan en e-postloop uppstå? | **Cykel i grafen** | starkt sammanhängande komponenter |
| Var måste en regel stå för att garanterat träffa? | **Dominator** för startnoden | dominatorträd |

De två sista raderna är de praktiskt mest värdefulla. En e-postloop är inte ett mystiskt driftfenomen utan en cykel i routningsgrafen; hoppräknaren under körning är bara nödbromsen, strukturellt hittar du loopen i förväg. Och om du vill placera en regel som **varje** meddelande måste passera, till exempel ett filter för avsändardomäner som inte kan routas, frågar du efter en dominator. Det är inte en smaksak, utan beräkningsbart.

### Hitta cykler innan de blir e-postloopar

Breddförstsökningen besvarar frågan om död kod. För loopar behöver du djupförstsökning, eftersom en **bakåtkant** där avslöjar cykeln. Metoden är den klassiska trefärgsmarkeringen:

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

Ett sådant fynd är inget bevis för en loop, eftersom kanterna bevakas och kanske aldrig tas tillsammans. Men det är den fullständiga listan över ställen där en kan uppstå **kan**, och just dem vill du känna till före en ombyggnad. Hoppräknaren under körning är bara nödbromsen; här ser du konstruktionen.

Lika viktig är **begränsningen** hos metoden. Kanterna bevakas av Matchers och de beror på meddelandeinnehållet. Exakt räckvidd är därför i allmänhet obestämd, analysen ger en överapproximation. Det följer en asymmetrisk beviskraft som du måste känna till:

- **”Oåtkomlig” är tillförlitligt.** Om ingen väg leder dit kan inget meddelande komma dit. Den koden kan du ta bort.
- **”Åtkomlig” betyder bara ”strukturellt inte utesluten”.** Grafen säger inte om något verkligt meddelande någonsin uppfyller villkoren.

Analysen ersätter alltså inte testning, men den minskar testutrymmet. I praktiken är det ändå en enorm vinst: av 38 processorer återstår 18 som du över huvud taget behöver kontrollera.

Metoder från maskininlärning, som Graph Neural Networks eller nodinbäddningar, behöver du uttryckligen inte här. De är värda det för stora grafer med okänd struktur och statistiska mönster. Ett regelverk har några dussin noder, en helt känd struktur och deterministisk semantik. Exakta algoritmer är här inte bara billigare, de ger bevis i stället för sannolikheter.

## Fallgropar vid maskinell bearbetning

När du ändrar ett regelverk med skript finns det tre fel som uppstår pålitligt. Jag har själv gjort alla tre.

**Klassikern: det giriga mönstret över processorgränser.** Den som vill ta bort en processor med ett reguljärt uttryck väljer naturligt:

```python
muster = re.compile(r'   <processor name="%s">.*?</processor>\n' % name, re.S)
```

Detta är fel. Inom kedjan finns i varje `ToProcessor`-Mailet en `<processor>ziel</processor>`, och det icke-giriga `.*?` stannar precis där. Resultatet: halva processorn tas bort, en rest av `</mailet>` och `</processor>` blir kvar och XML:en förstörs. Förankra i stället vid indragningen för den avslutande taggen och kontrollera taggbalansen mot:

```python
muster = re.compile(r'   <processor name="%s">.*?\n   </processor>\n' % name, re.S)

treffer = muster.search(daten)
assert treffer, "Prozessor nicht gefunden"
block = treffer.group(0)
assert block.count("<mailet ") == block.count("</mailet>"), "Tag-Bilanz stimmt nicht"
```

**Radslut.** Konfigurationen använder vanligtvis CRLF. Läs i Python med `rb`, normalisera till `\n` för bearbetningen och skriv tillbaka CRLF i slutet. Den som glömmer det producerar en fil med blandade radslut, som beroende på produkt avvisas utan kommentar.

**Specialtecken.** Håll filen i ren ASCII och skriv umlauter som teckenreferenser (`&#228;` för ä). Det sparar dig varje diskussion om kodningar mellan redigerare, skript och gatewayens webbgränssnitt.

Kontrollera efter varje ändring åtminstone välformning, oförändrade radslut och oförändrat antal processorer. Tre rader kontroll sparar en återställning.

## Metoden för ombyggnaden: parallellt träd med en växel

Nu till själva ombyggnaden. Den närliggande vägen, att bygga om det befintliga regelverket steg för steg, är den sämsta: du kan inte återgå rent och du kan inte längre läsa det gamla tillståndet.

I stället har det parallella trädet visat sig fungera väl:

**Steg 1: Bygg upp det nya trädet bredvid.** Skapa de nya processorerna med ett namnsuffix, till exempel `rootV2`, `incomingV2`, `outgoingV2`. Det gamla trädet förblir helt och oförändrat.

**Steg 2: En enda växel.** I början av den befintliga startpunkten finns exakt ett Mailet:

```xml
<processor name="root">
   <mailet class="ToProcessor" match="All">
      <processor>rootV2</processor>
   </mailet>
   <!-- alles Bisherige bleibt hier stehen, ist aber nun unerreichbar -->
</processor>
```

Därmed går all trafik genom det nya trädet. Det gamla är oåtkomligt, men fortfarande fullständigt närvarande. **Vägen tillbaka består i att ta bort dessa tre rader**, och den är begriplig i varje situation, även för en person som inte gjorde ombyggnaden.

**Steg 3: Räckvidd som acceptans.** Kör analysen från verktyg 2 och kontrollera tre punkter: Den nya startpunkten refereras exakt en gång, alla nya processorer är åtkomliga och det gamla trädet är helt oåtkomligt. Det är ett objektivt acceptanskriterium i stället för en visuell kontroll.

**Steg 4: Rensa först efter beprövad drift.** När det nya trädet har bekräftats i drift tar du bort det gamla och stryker suffixen. Först då förlorar du vägen tillbaka i filen, och fram till dess har du inte behövt den.

För mellansteg som du vill observera men ännu inte aktivera skarpt passar rena observations-Mailets: de loggar, men ändrar inte routningen. På så sätt samlar du in de data som saknas för beslutet utan risk.

## Bygg in synlighet direkt

Vid ombyggnaden lönar det sig att ta hänsyn till två saker som senare gör stor skillnad i drift.

**Kasta aldrig direkt i huvudkedjan.** Ett Mailet som kastar ett meddelande lämnar i meddelandeförloppet bara uppgiften att det raderades, utan anledning. Förgrena i stället till en särskilt namngiven processor, exempelvis `dropNonRoutable`. Bara namnet visas i förloppet och säger redan vad som sker.

**All loggning hamnar inte i meddelandeförloppet.** Många produkter har två mekanismer: en för serverloggen och en för förloppet som även supporten ser. Endast den andra är synlig i förloppet. Den som enbart sätter den första har visserligen loggat, men i spårningen står fortfarande bara ”Meddelande raderat”. Formulera förloppsposterna i klartext och ange regeln: ”medvetet kastat av regeln för avsändardomäner som inte kan routas, inget leveransfel” sparar mycket följdfrågor i drift.

## Klustret är en del av uppgiften

En punkt som regelbundet underskattas: Kör gatewayen på flera noder måste konfigurationen vara **identisk på alla noder och beständig efter omstart**. Om den bara är aktiv på en nod beror beteendet på vilken nod som behandlar meddelandet, och dina tester mäter slumpen.

Särskilt obehagligt är fallet där en ändring visserligen körs, men inte har gjorts beständig. Då fungerar noden korrekt tills den startas om, och återgår sedan till det gamla läget. Kontrollera därför efter varje distribution båda sakerna: samma version på alla noder och att versionen överlever en omstart.

## Sammanfattning

Behandla regelverket som en graf, inte som en textfil. En breddförstsökning från startpunkten skiljer på några rader kod levande från dött, och analysen inom kedjorna hittar dessutom regler som visserligen finns där men aldrig nås efter en ovillkorlig utgång.

För själva ombyggnaden är det parallella trädet med en enda växel metoden med bäst förhållande mellan insats och säkerhet. Och räckviddsanalysen ger dig samtidigt acceptanskriteriet för den.

## Källor

1.  [Apache James: Mailet container configuration](https://james.apache.org/server/config-mailetcontainer.html): Struktur för spoolmanagern, processorer, Mailets och Matchers samt bearbetningsordningen.

2.  [Apache James: Provided mailets](https://james.apache.org/server/dev-provided-mailets.html): Referens för de medföljande Mailets, inklusive ToProcessor och parametrarna för vidarebefordran och konsumtion.

3.  [Apache James: Provided matchers](https://james.apache.org/server/dev-provided-matchers.html): Referens för Matchers, bland annat All, HostIsLocal och mottagarrelaterade varianter.

4.  [Apache James: Mailet API](https://james.apache.org/server/dev-mailet-api.html): Avtalet mellan Mailet och container, grunden för att förstå konsumtion och vidarebefordran.
