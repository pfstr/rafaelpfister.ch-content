---
title: "Vad vi kan lära oss av naturvetenskapen för IT-felsökning"
navTitle: "Kontrollerade experiment"
description: "Falsifierbarhet, kontrollgrupper, störvariabler och urvalsbias: Metoden som naturvetenskaperna har arbetat med i århundraden löser just de problem där IT-felsökning regelbundet misslyckas. Med genomspelade exempel från e-postflödet."
date: "2026-08-11"
kategorie: "SMTP / e-postflöde"
timeToRead: "15 min läsning"
themen:
  - smtp-mailflow
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
url: https://rafaelpfister.ch/sv/blog/vad-vi-kan-lara-oss-av-naturvetenskapen-for-it-felsokning
translationSourceHash: d2466d0e63e5b08052fe7a47766ec2500b94c84097bfcfe91f8f6348cd6d1cc2
translationModel: gpt-5.6-terra
translatedAt: 2026-08-12T05:21:15.088Z
translationReview: automatic
---

# Vad vi kan lära oss av naturvetenskapen för IT-felsökning

Ett meddelande kommer inte fram. Protokollet ger ett felmeddelande som genast antyder en förklaring. Du granskar denna förklaring, hittar belägg och efter två timmar visar det sig att förklaringen var felaktig och att beläggen var en slump.

Detta är inget nybörjarmisstag, utan regel. Och det är anmärkningsvärt att vår bransch sällan har en metod för detta problem, trots att en har funnits i århundraden och fungerar mycket bra. Naturvetenskaperna har exakt samma uppgift: att dra slutsatser om orsaker utifrån observationer i system som man inte kan överblicka helt.

Den här artikeln överför fem principer från den vetenskapliga metoden till felsökning av e-postflöden. Exemplen kommer från praktiken, men tillvägagångssättet är inte specifikt för e-post.

## Varför IT-felsökning är systematiskt sårbar

E-postflödet är en kedja av system som vart och ett har sin egen syn på samma meddelande: gatewayen, filtreringslagret, den lokala transportservern, molntjänsten, måle-postlådan. Varje meddelande är skrivet ur exakt ett lags perspektiv.

Dessutom är feltexter samlingsbegrepp. Samma formulering beskriver ofta helt olika situationer, eftersom det avvisande systemet bara känner till ett grovt raster. De utökade statuskoderna är gjorda just för att bilda klasser, inte för att ange enskilda fall.

Ett exempel: En molntjänst avvisade ett meddelande med uppgiften att avsändaren inte var tillåten för utgående leverans. Samma formulering förekom i samma miljö i två helt olika konstellationer. I ena fallet försökte ett system leverera via tjänsten till en extern mottagare, alltså ett verkligt vidarebefordringsförsök utåt. I det andra fallet var mottagaren en vanlig e-postlåda i tjänsten, och endast avsändardomänen anmärktes.

Den som tolkar texten bokstavligt söker efter samma sak i båda fallen. Och eftersom ordet ”utgående” förekommer i den börjar man leta i fel ände.

## Princip 1: En hypotes måste utesluta något

Karl Popper berikade vetenskapsteorin med en insikt som är direkt praktisk för felsökning: **Ett påstående är bara användbart om det går att motbevisa.** En förklaring som passar alla tänkbara observationsresultat förklarar ingenting.

Överfört betyder det: Formulera din misstanke så att den innehåller en **förutsägelse** som kan vara fel. Inte ”något med avsändardomänen stämmer inte”, utan ”om jag skickar samma meddelande med en annan avsändardomän via samma väg, kommer det fram”.

Den andra formuleringen har ett värde eftersom den kan kullkastas på fem minuter. Den första kan du mata med belägg i timmar utan att någonsin bli klokare.

Ett bra test: Fråga dig före försöket vilket resultat som skulle **motbevisa** din hypotes. Om du inte kommer på något har du ingen hypotes, utan en känsla.

## Princip 2: En variabel, annars allt lika

Experimentets kärna är kontrollen av störvariabler. I praktiken händer regelbundet motsatsen: Man jämför två fall som råkar finnas tillgängliga. Och de skiljer sig nästan alltid åt i flera egenskaper samtidigt.

Från ett verkligt fall: Meddelanden från `example-test.com` avvisades, medan meddelanden från `partner.example` kom fram. De två domänerna skilde sig åt i minst fyra egenskaper: tillhörighet till organisationen, var e-posten hostas, om en strikt autentiseringspolicy finns konfigurerad och inlämningsvägen. Av två datapunkter med fyra skillnader går det att dra exakt inga slutsatser. Alla fyra förklaringarna passar.

Bygg därför jämförelsen själv. Samma inlämningspunkt, samma mottagare, samma väg, samma tid och **exakt en** ändrad egenskap. Om du misstänker avsändardomänen ändrar du bara den.

## Princip 3: Utan kontrollförsök är resultatet värdelöst

Detta är den del man helst utelämnar, och den viktigaste. I klinisk forskning är kontrollgruppen självklar; inom IT avstår man oftast från den och förundras sedan över motsägelsefulla resultat.

**Din testuppställning måste först reproducera felet.** Om du inte kan skapa felfallet med dina egna medel säger ett lyckat motförsök ingenting. Kanske fungerar ditt testmeddelande bara för att du lämnar in det någon annanstans än originalsystemet, eller för att en kontroll inte alls tillämpas på din väg.

Ett användbart test består därför av minst två meddelanden:

| | Syfte | Förväntning |
|---|---|---|
| Försök 1 | Kontroll, replikerar originalfallet | **måste misslyckas** |
| Försök 2 | Hypotes, en variabel ändrad | bör lyckas |

Om försök 1 inte misslyckas är din uppställning inte representativ. Då har du inte lärt dig något om originalfallet, utan bara om din testuppställning, och måste lämna in närmare originalet.

## Ett genomspelat exempel

Tillbaka till fallet ovan, anonymiserat. Meddelanden från ett system nådde inte mottagare i molnet, medan andra meddelanden till samma mottagare kom fram utan problem. Tre försök via samma väg, till samma mottagare, med några minuters mellanrum:

| Försök | Avsändardomän | Hypotes som det prövar | Resultat |
|---|---|---|---|
| 1 (kontroll) | `example-test.com` | Uppställningen är representativ | Avvisning, identisk med originalet |
| 2 | `example.com`, målets egen domän | Det beror på avsändardomänen | levererat |
| 3 | `other-test.com`, extern domän i samma organisation | Det beror på organisationstillhörigheten | levererat |

Försök 1 reproducerade felet, alltså var uppställningen giltig. Försök 2 visade att det hänger på avsändardomänen och inte på mottagare, e-postlåda, routning eller behörigheter. Försök 3 var det verkligt eleganta: Det prövade den mest närliggande alternativa förklaringen specifikt och **motbevisade den**, eftersom `other-test.com` tillhörde samma organisation och ändå släpptes igenom.

Tre meddelanden, tio minuter, och orsaken var belagd i stället för antagen. Innan dess hade flera timmar lagts på förklaringsförsök, av vilka inget höll i slutändan.

## Princip 4: Att motbevisa är det egentliga framsteget

En motbevisad hypotes känns som ett steg tillbaka. I själva verket är det det enda du vet säkert. Bekräftelser är svaga, eftersom en observation kan passa flera förklaringar. Ett rent motbevis tar bort en hel gren ur sökutrymmet, permanent.

Det är just här bekräftelsebias har störst verkan. När du har en misstanke hittar du nästan alltid något som passar den. I analysen som beskrivs ovan fanns en korrelation mellan avvisningen och frågan om var avsändardomänen låter hosta sin e-post. Den såg övertygande ut, men byggde på två datapunkter som skilde sig åt i flera egenskaper. Det tredje försöket slog sönder den.

Skriv därför ner de motbevisade förklaringarna tillsammans med skälet till att de förkastades. Det är inget annat än en laboratoriejournal. Det har två effekter: Den som senare tar över fallet hamnar inte i samma återvändsgränder. Och du märker själv när du tänker i cirklar, eftersom en redan förkastad idé återkommer under ett nytt namn.

I dokumentationen ska de motbevisade punkterna uttryckligen stå bredvid de belagda. En rapport som bara innehåller rätt svar döljer hälften av arbetet och inbjuder till att det upprepas.

## Princip 5: Känn ditt urval

Den mest subtila felkällan är urvalsbias, och inom IT drabbar den framför allt frågor som returnerar resultat sida för sida.

Du frågar efter meddelandespårning för sju dagar, filtrerar lokalt efter en egenskap och får inget resultat. Slutsatsen ligger nära till hands att denna trafik inte förekom. I själva verket filtrerade du bara den första sidan, som vid hög belastning täcker några få minuter.

Det korrekta resultatet är: inte hittat i utdraget. Det är inte: finns inte. Skillnaden är densamma som mellan ”ingen effekt kunde påvisas i vår studie” och ”det finns ingen effekt”.

Två utvägar fungerar. Minska tidsfönstret så mycket att en sida täcker det helt, vilket märks genom att indikeringen om ytterligare resultat uteblir. Eller bläddra igenom alla sidor och utvärdera sedan.

Och en tredje, som ofta förbises: För frågan om något **aldrig** förekommer är en konfigurationskontroll överlägsen varje observation. Om ett system inte har någon rutt till ett mål kan det inte leverera dit, oberoende av varje observationsfönster. Det är skillnaden mellan ett empiriskt och ett strukturellt argument, och där du kan få det strukturella ska du använda det.

## Överföringen: Koppla bevisbördan till reversibiliteten

Här slutar analogin med vetenskapen, och ingenjörsperspektivet tar över. Forskning vill ha sanning, drift vill ha en fungerande anläggning. Därav följer ett mått som vetenskapen inte känner till: **Insatsen för belägget beror på hur reversibelt ingreppet är.**

Att inaktivera en connector är ett kommando, och att återställa det likaså. För det räcker välgrundade indicier, eftersom ett misstag kan rättas till på en minut och märks omedelbart. Att radera samma connector är inte reversibelt; då är det värt den extra verifieringen via motpartens konfiguration eller en användningsrapport på serversidan.

Detsamma gäller regeländringar. Ett rent observationssteg, som loggar och inte omdirigerar något, får du införa med tunt faktaunderlag. Det är konsekvensfritt och samlar in just de data som saknas för det skarpa steget. Först omställningen som kan hålla kvar meddelanden kräver robusta belägg.

Den som inte tillämpar denna måttstock gör regelbundet båda misstagen samtidigt: kräver veckolånga bevis för en ändring som kan återställas på sekunder, och aktiverar utan skydd något som kan stoppa e-posttrafiken.

## När du får sluta

Det finns en punkt där fortsatt grävande inte längre skapar något värde: när lösningen står klar men mekanismen förblir oklar.

I exemplet ovan var det efter tre försök belagt att avsändardomänen var utlösaren, att allt annat i e-postvägen fungerade och att inget bredare problem förelåg. Varför molntjänsten internt fattar just detta beslut förblev öppet. För korrigeringen saknade det betydelse, eftersom den låg hos den sändande applikationen.

Separera därför medvetet två frågor. Vad måste jag ändra för att det ska fungera? Och varför beter sig systemet så? Den första måste du besvara, den andra får du lämna till tillverkaren. Ett supportärende med tre kontrollerade försök, tidsstämplar, meddelandeidentifierare och ett fungerande motexempel är ändå mångdubbelt mer värdefullt än en beskrivning av symtomet.

Detta är för övrigt också punkten där vetenskap och drift kan skiljas åt på ett rent sätt. Vetenskapen får inte ge upp frågan om mekanismen. Driften måste prioritera den.

## Kortversionen

Formulera hypoteser så att de kan misslyckas och fråga dig i förväg vilket resultat som skulle motbevisa dem. Jämför aldrig två fall som råkar finnas tillgängliga, utan bygg jämförelsen med exakt en ändrad variabel. Reproducera felet i kontrollförsöket innan du tror på motförsöket. Behandla motbevis som framsteg och dokumentera dem skriftligt. Kontrollera vid varje fråga om du ser helheten eller ett urval. Och anpassa det bevisdjup som krävs efter hur lätt det planerade ingreppet kan återställas.

De konkreta frågorna för detta finns i [Analysera Exchange-e-postflöde: Message Tracking, SMTP-protokoll och Receive Connectors](https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren). Den som hellre klickar ihop kommandona än skriver dem hittar dem i [Kommandogeneratorn](https://rafaelpfister.ch/tools/command-builder).

## Källor

1.  [Karl Popper: Logik der Forschung](https://www.mohrsiebeck.com/buch/logik-der-forschung-9783161584350): Ursprung till falsifikationsprincipen, enligt vilket ett påstående bara är vetenskapligt om det förblir motbevisbart.

2.  [RFC 3463: Enhanced Mail System Status Codes](https://www.rfc-editor.org/rfc/rfc3463): förklarar varför utökade statuskoder medvetet är grova klasser och tillåter samma kod för olika orsaker.

3.  [Message tracking in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): händelsetyper och fält, grund för att fastställa det sista bearbetningssteget.

4.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2): sidlogiken i meddelandespårningen, som gynnar urvalsfel.
