---
title: "Vad vi kan lära oss av naturvetenskapen för felsökning inom IT"
navTitle: "Kontrollerade experiment"
description: "Falsifierbarhet, kontrollgrupper, störvariabler och urvalsbias: Metoden som naturvetenskaperna har använt i århundraden löser precis de problem där IT-felsökning regelbundet misslyckas, illustrerat med exempel från e-postflödet."
date: "2026-08-11"
kategorie: "SMTP / e-postflöde"
timeToRead: "15 min. läsning"
themen:
  - smtp-mailflow
  - testing
  - exchange-onprem-hybrid
hauptthema: "smtp-mailflow"
produkte:
  - "exchange"
protokolle:
  - "smtp"
  - "troubleshooting"
related:
  - exchange-message-tracking-und-receive-connectoren-analysieren
  - einliefernde-ip-adressen-aggregieren
  - typische-ursachen-fuer-mail-loops-und-deren-behebung
slug: "vad-vi-kan-lara-oss-av-naturvetenskapen-for-it-felsokning"
translationId: "article-098ed40e6d027b8b"
draft: false
translationOf: mailflow-fehlersuche-kontrollierte-experimente
translationSourceHash: 1759c709160723c161883fede66d48cd968bb6a587cb49c73781b3a39498cd66
translationModel: gpt-5.6-terra
translatedAt: 2026-10-30T03:30:51.929Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/vad-vi-kan-lara-oss-av-naturvetenskapen-for-it-felsokning
---

# Vad vi kan lära oss av naturvetenskapen för felsökning inom IT

Ett meddelande kommer inte fram. Protokollet ger ett felmeddelande som omedelbart antyder en förklaring. Du kontrollerar den förklaringen, hittar belägg och efter två timmar visar det sig att förklaringen var felaktig och att beläggen var slumpmässiga.

Det är inget nybörjarmisstag, utan regel snarare än undantag. Och det är anmärkningsvärt att vår bransch sällan har en metod för detta problem, trots att en har funnits i århundraden och fungerar utmärkt. Naturvetenskaperna har exakt samma uppgift: att dra slutsatser om orsaker utifrån observationer i system som man inte har fullständig överblick över.

Den här artikeln överför fem principer från den vetenskapliga metoden till felsökning i e-postflödet. Exemplen kommer från praktiken, men tillvägagångssättet är inte specifikt för e-post.

## Varför IT-felsökning är systematiskt sårbar

E-postflödet är en kedja av system som vart och ett har sin egen bild av samma meddelande: gatewayen, filterlagret, den lokala transportservern, molntjänsten, målpostlådan. Varje meddelande är skrivet ur exakt ett lagers perspektiv.

Dessutom är feltexter samlingsbegrepp. Samma formulering beskriver ofta helt olika situationer, eftersom systemet som avvisar bara känner till ett grovt rutnät. De utökade statuskoderna är just till för att bilda klasser, inte för att benämna enskilda fall.

Ett exempel: En molntjänst avvisade ett meddelande med upplysningen att avsändaren inte var tillåten för utgående leverans. Samma formulering förekom i samma miljö i två helt olika konstellationer. I det ena fallet försökte ett system leverera via tjänsten till en extern mottagare, alltså ett verkligt försök att vidarebefordra utåt. I det andra fallet var mottagaren en vanlig postlåda i tjänsten, och endast avsändardomänen anmärktes.

Den som tar texten bokstavligt letar efter samma sak i båda fallen. Och eftersom ordet ”utgående” förekommer i den börjar man leta i fel ände.

## Princip 1: En hypotes måste utesluta något

Karl Popper berikade vetenskapsteorin med en insikt som är direkt praktisk för felsökning: **Ett påstående är bara användbart om det kan motbevisas.** En förklaring som passar varje tänkbart observationsresultat förklarar ingenting.

Översatt betyder det: Formulera din misstanke så att den innehåller en **förutsägelse** som kan vara fel. Inte ”något med avsändardomänen stämmer inte”, utan ”om jag skickar samma meddelande med en annan avsändardomän via samma väg kommer det fram”.

Den andra formuleringen är värdefull eftersom den kan motbevisas på fem minuter. Den första kan du mata med belägg i timmar utan att någonsin bli klokare.

Ett bra test är detta: Fråga dig före försöket vilket resultat som skulle **motbevisa** din hypotes. Om du inte kommer på något har du ingen hypotes, utan en känsla.

## Princip 2: En variabel, annars allt lika

Experimentets kärna är kontrollen av störvariabler. I praktiken händer regelbundet motsatsen: Man jämför två fall som råkar finnas tillgängliga. Och de skiljer sig nästan alltid åt i flera egenskaper samtidigt.

Från ett verkligt fall: Meddelanden från `example-test.com` avvisades, medan meddelanden från `partner.example` kom fram. De båda domänerna skilde sig åt i minst fyra egenskaper: tillhörighet till organisationen, var e-posten hostas, om en strikt autentiseringspolicy finns konfigurerad och leveransvägen. Av två datapunkter med fyra skillnader går det inte att dra någon slutsats alls. Alla fyra förklaringar passar.

Bygg därför jämförelsen själv. Samma inlämningspunkt, samma mottagare, samma väg, samma tid och **exakt en** ändrad egenskap. Om du misstänker avsändardomänen ändrar du bara den.

## Princip 3: Utan kontrollförsök är resultatet värdelöst

Det här är den del man helst utelämnar, och den viktigaste. Inom klinisk forskning är kontrollgruppen självklar; inom IT avstår man oftast från den och förundras över motstridiga resultat.

**Din testuppställning måste först reproducera felet.** Om du inte kan återskapa felsituationen med dina egna verktyg säger ett lyckat motförsök ingenting. Din testpost kanske bara fungerar eftersom du lämnar in den på ett annat ställe än originalsystemet, eller eftersom en kontroll inte alls tillämpas på din väg.

Ett användbart test består därför av minst två meddelanden:

| | Syfte | Förväntning |
|---|---|---|
| Försök 1 | Kontroll, replikerar originalfallet | **måste misslyckas** |
| Försök 2 | Hypotes, en variabel ändrad | bör lyckas |

Om försök 1 inte misslyckas är din uppställning inte representativ. Då har du inte lärt dig något om originalfallet, utan bara om din testuppställning, och måste lämna in närmare originalet.

## Ett genomgånget exempel

Tillbaka till fallet ovan, anonymiserat. Meddelanden från ett system nådde inte mottagare i molnet, medan andra meddelanden till samma mottagare kom fram utan problem. Tre försök via samma väg, till samma mottagare, med några minuters mellanrum:

| Försök | Avsändardomän | Hypotes som det prövar | Resultat |
|---|---|---|---|
| 1 (kontroll) | `example-test.com` | Uppställningen är representativ | Avvisning, identisk med originalet |
| 2 | `example.com`, målets egen domän | det beror på avsändardomänen | levererat |
| 3 | `other-test.com`, extern domän i samma organisation | det beror på organisationstillhörigheten | levererat |

Försök 1 reproducerade felet, så uppställningen var giltig. Försök 2 visade att det hänger på avsändardomänen och inte på mottagare, postlåda, routning eller behörigheter. Försök 3 var det verkligt eleganta: Det prövade riktat den mest närliggande alternativa förklaringen och **motbevisade den**, eftersom `other-test.com` tillhörde samma organisation och ändå släpptes igenom.

Tre meddelanden, tio minuter, och orsaken var belagd i stället för antagen. Innan dess hade flera timmar lagts på förklaringsförsök, som inget av höll i slutändan.

## Princip 4: Att motbevisa är det verkliga framsteget

En motbevisad hypotes känns som ett steg bakåt. I själva verket är den det enda du vet säkert. Bekräftelser är svaga, eftersom en observation kan passa flera förklaringar. Ett tydligt motbevis eliminerar en hel gren ur sökrymden, och det permanent.

Det är just här bekräftelsebias verkar som starkast. När du har en misstanke hittar du nästan alltid något som passar den. I analysen som beskrivs ovan fanns en korrelation mellan avvisningen och frågan om var avsändardomänen har sin e-post hostad. Den såg övertygande ut, men byggde på två datapunkter som skilde sig åt i flera egenskaper. Det tredje försöket motbevisade den.

Anteckna därför de motbevisade förklaringarna tillsammans med skälet till att de avfärdades. Det är inget annat än en laboratoriejournal. Det har två effekter: Den som senare tar över fallet går inte in i samma återvändsgränder. Och du märker själv när du tänker i cirklar, eftersom en redan förkastad idé återkommer under ett nytt namn.

I dokumentationen hör de motbevisade punkterna uttryckligen hemma bredvid de belagda. En rapport som bara innehåller rätt svar döljer hälften av arbetet och inbjuder till att det upprepas.

## Princip 5: Känn till ditt urval

Den mest subtila felkällan är urvalsbias, och inom IT drabbar den främst frågor som levererar resultat sida för sida.

Du frågar efter sju dagars meddelandespårning, filtrerar lokalt efter en egenskap och får inget resultat. Slutsatsen ligger nära till hands att denna trafik inte förekom. I själva verket har du bara filtrerat den första sidan, och vid hög belastning täcker den bara några minuter.

Det korrekta resultatet lyder: inte hittat i urvalet. Det lyder inte: finns inte. Skillnaden är densamma som mellan ”ingen effekt kunde påvisas i vår studie” och ”det finns ingen effekt”.

Två utvägar fungerar. Minska tidsfönstret så mycket att en sida täcker det helt, vilket märks genom att hänvisningen till fler resultat uteblir. Eller bläddra igenom alla sidor och utvärdera först därefter.

Och ett tredje alternativ, som ofta förbises: För frågan om något **aldrig** förekommer är en konfigurationskontroll överlägsen varje observation. Om ett system saknar en rutt till ett mål kan det inte leverera dit, oavsett observationsfönster. Det är skillnaden mellan ett empiriskt och ett strukturellt argument, och där du kan använda det strukturella ska du göra det.

## Överföringen: Knyt bevisbördan till reversibiliteten

Här slutar analogin med vetenskapen och ingenjörsperspektivet tar över. Forskning vill ha sanning, drift vill ha en fungerande anläggning. Därav följer en måttstock som vetenskapen inte känner till: **Insatsen för belägg ska stå i proportion till ingreppets reversibilitet.**

Att inaktivera en connector är ett kommando, och detsamma gäller att återställa det. Motiverade indicier räcker för detta, eftersom ett misstag är åtgärdat på en minut och märks direkt. Att radera samma connector är inte reversibelt; då är det värt det extra belägget genom motpartens konfiguration eller en användningsrapport på serversidan.

Detsamma gäller regeländringar. Ett rent observationssteg som loggar och inte omdirigerar något kan du införa med tunt faktaunderlag. Det är ofarligt och samlar in precis de data som saknas för det skarpa steget. Först ändringen som kan hålla kvar meddelanden kräver robusta belägg.

Den som inte använder denna måttstock gör regelbundet båda felen samtidigt: kräver veckolånga bevis för en ändring som kan återställas på sekunder och aktiverar utan säkring något som kan stoppa e-posttrafiken.

## När du får sluta

Det finns en punkt där ytterligare grävande inte längre skapar något värde: när lösningen är fastställd men mekanismen förblir oklar.

I exemplet ovan var det efter tre försök belagt att avsändardomänen är utlösaren, att allt annat i e-postvägen fungerar och att inget bredare problem föreligger. Varför molntjänsten internt beslutar på just det sättet förblev öppet. För korrigeringen saknade det betydelse, eftersom den låg hos den sändande applikationen.

Skilj därför medvetet på två frågor. Vad måste jag ändra för att det ska fungera? Och varför beter sig systemet så? Den första måste du besvara, den andra får du lämna till tillverkaren. Ett supportärende med tre kontrollerade försök, tidsstämplar, meddelandeidentifierare och ett fungerande motexempel är ändå vida mer värdefullt än en beskrivning av symptomet.

Det är för övrigt också den punkt där vetenskap och drift kan skiljas åt på ett tydligt sätt. Vetenskapen får inte ge upp frågan om mekanismen. Driften måste prioritera den.

## Kortversionen

Formulera hypoteser så att de kan misslyckas och fråga dig i förväg vilket resultat som skulle motbevisa dem. Jämför aldrig två fall som råkar finnas tillgängliga, utan bygg jämförelsen med exakt en ändrad variabel. Reproducera felet i kontrollförsöket innan du litar på motförsöket. Behandla motbevis som framsteg och dokumentera dem skriftligt. Kontrollera vid varje fråga om du ser hela mängden eller ett urval. Anpassa dessutom det efterfrågade bevisdjupet efter hur lätt det planerade ingreppet kan återställas.

De konkreta frågorna finns i [Analysera Exchange-e-postflöde: Message Tracking, SMTP-protokoll och Receive Connectors](https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren). Den som hellre klickar ihop kommandona än skriver in dem hittar dem i [Kommandogeneratorn](https://rafaelpfister.ch/tools/command-builder).

## Ändringshistorik

- **25 oktober 2026:** Källänkar uppdaterade till de nya adresserna hos Mohr Siebeck och Microsoft Learn.

## Källor

1.  [Karl Popper: Logik der Forschung](https://www.mohrsiebeck.com/buch/logik-der-forschung-9783161484100/): Ursprung till falsifikationsprincipen, enligt vilket ett påstående bara är vetenskapligt om det förblir möjligt att motbevisa.

2.  [RFC 3463: Enhanced Mail System Status Codes](https://www.rfc-editor.org/rfc/rfc3463): förklarar varför utökade statuskoder medvetet är grova klasser och tillåter samma kod för olika orsaker.

3.  [Message tracking in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): händelsetyper och fält, grund för att fastställa det sista bearbetningssteget.

4.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2): sidlogiken för meddelandespårning, som gynnar urvalsfel.
