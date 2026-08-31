---
title: "Vad vi kan lära oss av naturvetenskapen för felsökning inom IT"
navTitle: "Kontrollerade experiment"
description: "Falsifierbarhet, kontrollgrupper, störvariabler och urvalsbias: Metoden som naturvetenskaperna har arbetat med i århundraden löser just de problem där IT-felsökning regelbundet misslyckas, illustrerat med exempel från e-postflödet."
date: "2026-08-11"
kategorie: "SMTP / e-postflöde"
timeToRead: "15 min. läsning"
themen:
  - smtp-mailflow
  - testing
  - exchange-onprem-hybrid
hauptthema: "smtp-mailflow"
produkte:
  - "uebergreifend"
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
translationSourceHash: e3fff70bc1386c28d78713ec89a35b4d6c29b7f16e809e8a84bd9850a40a261c
translationModel: gpt-5.6-terra
translatedAt: 2026-08-31T10:17:35.036Z
translationReview: automatic
url: https://rafaelpfister.ch/sv/blog/vad-vi-kan-lara-oss-av-naturvetenskapen-for-it-felsokning
---

# Vad vi kan lära oss av naturvetenskapen för felsökning inom IT

Ett meddelande kommer inte fram. Protokollet ger ett felmeddelande som omedelbart antyder en förklaring. Du kontrollerar förklaringen, hittar belägg, och efter två timmar visar det sig att förklaringen var fel och att beläggen var tillfälligheter.

Det är inget nybörjarmisstag, utan regel. Och det är anmärkningsvärt att vår bransch sällan har en metod för detta problem, trots att det finns en sedan århundraden och den fungerar synnerligen väl. Naturvetenskaperna har exakt samma uppgift: att dra slutsatser om orsaker utifrån observationer, i system som man inte har full överblick över.

Den här artikeln överför fem principer från den vetenskapliga metoden till felsökning i e-postflödet. Exemplen kommer från praktiken, men tillvägagångssättet är inte specifikt för e-post.

## Varför IT-felsökning systematiskt är sårbar

E-postflödet är en kedja av system som vart och ett har sin egen bild av samma meddelande: gatewayen, filtreringslagret, den lokala transportservern, molntjänsten, måle-postlådan. Varje meddelande är skrivet ur exakt ett lagers perspektiv.

Dessutom är feltexter samlingsbegrepp. Samma formulering beskriver ofta helt olika situationer, eftersom det avvisande systemet bara känner till ett grovt raster. De utökade statuskoderna är gjorda just för att bilda klasser, inte för att benämna enskilda fall.

Ett exempel: En molntjänst avvisade ett meddelande med uppgiften att avsändaren inte var tillåten för utgående leverans. Samma formulering förekom i samma miljö i två helt olika situationer. Ena gången försökte ett system leverera via tjänsten till en extern mottagare, alltså ett faktiskt vidarebefordringsförsök utåt. Andra gången var mottagaren en vanlig e-postlåda i tjänsten, och det var enbart avsändardomänen som anmärktes på.

Den som tar texten bokstavligt letar efter samma sak i båda fallen. Och eftersom ordet ”utgående” förekommer i den, börjar man leta i fel ände.

## Princip 1: En hypotes måste utesluta något

Karl Popper berikade vetenskapsteorin med en insikt som är direkt praktisk för felsökning: **Ett påstående är bara användbart om det kan motbevisas.** En förklaring som passar varje tänkbart observationsresultat förklarar ingenting.

Överfört innebär det: Formulera din misstanke så att den innehåller en **förutsägelse** som kan vara fel. Inte ”något med avsändardomänen stämmer inte”, utan ”om jag skickar samma meddelande med en annan avsändardomän via samma väg, kommer det fram”.

Den andra formuleringen är värdefull eftersom den kan motbevisas på fem minuter. Den första kan du mata med belägg i timmar utan att någonsin bli klokare.

Ett bra test för detta: Fråga dig före försöket vilket resultat som skulle **motbevisa** din hypotes. Om du inte kommer på något har du ingen hypotes, utan en känsla.

## Princip 2: En variabel, annars allt lika

Experimentets kärna är kontroll av störvariabler. I praktiken sker regelbundet motsatsen: Man jämför två fall som råkar finnas tillgängliga. Och de skiljer sig nästan alltid åt i flera egenskaper samtidigt.

Från ett verkligt fall: Meddelanden från `example-test.com` avvisades, medan meddelanden från `partner.example` kom fram. De två domänerna skilde sig åt i minst fyra egenskaper: tillhörighet till organisationen, var e-posten hostas, om en strikt autentiseringspolicy finns konfigurerad och insändningsvägen. Av två datapunkter med fyra skillnader går det inte att dra exakt några slutsatser. Alla fyra förklaringarna passar.

Bygg därför jämförelsen själv. Samma insändningspunkt, samma mottagare, samma väg, samma tid och **exakt en** ändrad egenskap. Om du misstänker avsändardomänen, ändra bara den.

## Princip 3: Utan kontrollförsök är resultatet värdelöst

Detta är den del man helst utelämnar, och den viktigaste. Inom klinisk forskning är kontrollgruppen självklar; inom IT avstår man oftast från den och förundras över motsägelsefulla resultat.

**Din testuppställning måste först reproducera felet.** Om du inte kan återskapa felutfallet med dina egna verktyg säger ett lyckat motförsök ingenting. Kanske fungerar ditt testmeddelande bara för att du skickar in det på ett annat ställe än originalsystemet, eller för att en kontroll över huvud taget inte tillämpas på din väg.

Ett användbart test består därför av minst två meddelanden:

| | Syfte | Förväntning |
|---|---|---|
| Försök 1 | Kontroll, replikerar originalfallet | **måste misslyckas** |
| Försök 2 | Hypotes, en variabel ändrad | ska lyckas |

Om försök 1 inte misslyckas är din uppställning inte representativ. Då har du inte lärt dig något om originalfallet, utan bara om din testuppställning, och måste skicka in närmare originalet.

## Ett genomarbetat exempel

Tillbaka till fallet ovan, anonymiserat. Meddelanden från ett system nådde inte mottagare i molnet, medan andra meddelanden till samma mottagare kom fram utan problem. Tre försök via samma väg, till samma mottagare, med några minuters mellanrum:

| Försök | Avsändardomän | Hypotes som det prövar | Resultat |
|---|---|---|---|
| 1 (kontroll) | `example-test.com` | Uppställningen är representativ | Avvisning, identisk med originalet |
| 2 | `example.com`, målets egen domän | det beror på avsändardomänen | levererat |
| 3 | `other-test.com`, extern domän i samma organisation | det beror på organisationstillhörigheten | levererat |

Försök 1 reproducerade felet, så uppställningen var giltig. Försök 2 visade att det hänger på avsändardomänen och inte på mottagare, e-postlåda, routning eller behörigheter. Försök 3 var det verkligt eleganta: Det prövade målmedvetet den mest närliggande alternativa förklaringen och **motbevisade den**, eftersom `other-test.com` tillhörde samma organisation och ändå släpptes igenom.

Tre meddelanden, tio minuter, och orsaken var belagd i stället för antagen. Dessförinnan hade flera timmar lagts på försök att förklara, och i slutändan höll ingen av dem.

## Princip 4: Att motbevisa är det egentliga framsteget

En motbevisad hypotes känns som ett steg tillbaka. I själva verket är den det enda du vet säkert. Bekräftelser är svaga, eftersom en observation kan passa flera förklaringar. Ett rent motbevis eliminerar en hel gren ur sökutrymmet, permanent.

Det är just här bekräftelsebias verkar som starkast. När du har en misstanke hittar du nästan alltid något som passar den. I analysen som beskrivs ovan fanns en korrelation mellan avvisningen och frågan om var avsändardomänen låter sin e-post hostas. Den såg övertygande ut, men byggde på två datapunkter som skilde sig åt i flera egenskaper. Det tredje försöket motbevisade den.

Notera därför de motbevisade förklaringarna tillsammans med skälet till att de förkastades. Det är inget annat än en laboratoriejournal. Det har två effekter: Den som senare tar över fallet hamnar inte i samma återvändsgränder. Och du märker själv när du tänker i cirklar, eftersom en redan förkastad idé återkommer under ett nytt namn.

I dokumentationen hör de motbevisade punkterna uttryckligen hemma bredvid de belagda. En rapport som bara innehåller rätt svar döljer hälften av arbetet och inbjuder till att upprepa det.

## Princip 5: Känn till ditt urval

Den mest subtila felkällan är urvalsbias, och inom IT drabbar den framför allt frågor som levererar sida för sida.

Du frågar efter sju dagars meddelandespårning, filtrerar lokalt efter en egenskap och får inget resultat. Slutsatsen ligger nära till hands att denna trafik inte fanns. I själva verket har du bara filtrerat den första sidan, och vid hög volym täcker den bara några minuter.

Det korrekta resultatet är: inte hittat i urvalet. Det är inte: finns inte. Skillnaden är densamma som mellan ”ingen effekt kan påvisas i vår studie” och ”det finns ingen effekt”.

Två utvägar fungerar. Minska tidsfönstret så mycket att en sida täcker det helt, vilket märks genom att hänvisningen till fler resultat uteblir. Eller bläddra igenom alla sidor och utvärdera sedan.

Och en tredje, som ofta förbises: För frågan om något **aldrig** förekommer är en konfigurationskontroll överlägsen varje observation. Om ett system inte har någon rutt till ett mål kan det inte leverera dit, oberoende av observationsfönster. Det är skillnaden mellan ett empiriskt och ett strukturellt argument, och där du kan ha det strukturella ska du använda det.

## Överföringen: Koppla beviskravet till reversibiliteten

Här slutar analogin med vetenskapen, och ingenjörsperspektivet tar över. Forskning vill ha sanning, drift vill ha en fungerande anläggning. Därav följer en måttstock som vetenskapen inte känner: **Ansträngningen för belägget styrs av hur reversibelt ingreppet är.**

Att inaktivera en connector är ett kommando, och det är också att återställa det. För det räcker välgrundade indicier, eftersom ett misstag kan rättas till på en minut och märks genast. Att radera samma connector är inte reversibelt; för det är det värt den extra verifieringen genom motpartens konfiguration eller en serverbaserad användningsrapport.

Detsamma gäller regeländringar. Ett rent observationssteg, som loggar och inte omdirigerar något, får du införa på tunt faktaunderlag. Det är utan konsekvenser och samlar in exakt de data som saknas för det skarpa steget. Först en ändring som kan hålla kvar meddelanden kräver robusta belägg.

Den som inte tillämpar denna måttstock gör regelbundet båda felen samtidigt: kräver veckolånga bevis för en ändring som kan återställas på sekunder och aktiverar utan säkring något som kan stoppa e-posttrafiken.

## När du får sluta

Det finns en punkt där fortsatt grävande inte längre skapar något värde: när åtgärden står klar, men mekanismen förblir oklar.

I exemplet ovan var det efter tre försök belagt att avsändardomänen är utlösaren, att allt annat i e-postvägen fungerar och att inget bredare problem föreligger. Varför molntjänsten internt fattar exakt detta beslut förblev öppet. För korrigeringen saknade det betydelse, eftersom den låg hos den sändande applikationen.

Separera därför medvetet två frågor. Vad måste jag ändra för att det ska fungera? Och varför beter sig systemet så? Den första måste du besvara, den andra får du lämna till tillverkaren. Ett supportärende med tre kontrollerade försök, tidsstämplar, meddelandeidentifierare och ett fungerande motexempel är ändå mångdubbelt mer värdefullt än en beskrivning av symptomet.

Det är för övrigt också punkten där vetenskap och drift kan skiljas åt tydligt. Vetenskapen får inte ge upp frågan om mekanismen. Driften måste prioritera den.

## Kortversionen

Formulera hypoteser så att de kan misslyckas, och fråga dig i förväg vilket resultat som skulle motbevisa dem. Jämför aldrig två fall som råkar finnas tillgängliga, utan bygg jämförelsen med exakt en ändrad variabel. Reproducera felet i kontrollförsöket innan du tror på motförsöket. Behandla motbevis som framsteg och dokumentera dem skriftligt. Kontrollera vid varje fråga om du ser helheten eller ett urval. Och anpassa det begärda bevisdjupet efter hur lätt det planerade ingreppet kan återställas.

De konkreta frågorna finns i [Analysera Exchange-e-postflöde: Message Tracking, SMTP-protokoll och Receive-Connectors](https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren). Den som hellre klickar ihop kommandona än skriver in dem hittar dem i [Kommandogeneratorn](https://rafaelpfister.ch/tools/command-builder).

## Källor

1.  [Karl Popper: Logik der Forschung](https://www.mohrsiebeck.com/buch/logik-der-forschung-9783161584350): Ursprunget till falsifikationsprincipen, enligt vilken ett påstående bara är vetenskapligt om det förblir motbevisbart.

2.  [RFC 3463: Enhanced Mail System Status Codes](https://www.rfc-editor.org/rfc/rfc3463): förklarar varför utökade statuskoder medvetet är grova klasser och tillåter samma kod för olika orsaker.

3.  [Message tracking in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): händelsetyper och fält, grund för att fastställa det sista bearbetningssteget.

4.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2): sidlogiken för meddelandespårning, som främjar urvalsfel.
