---
title: "Planera belastningstester för e-post: verktyg för e-postburstar på 10 000 under Linux och Windows jämförda"
navTitle: "Belastningstester för e-post"
description: "Den som migrerar en gateway eller dimensionerar en e-postmiljö behöver tillförlitliga siffror i stället för magkänsla. Vilka verktyg som skapar burstar med tiotusentals e-postmeddelanden, hur en bra testplan ser ut och hur du utvärderar resultaten från loggarna."
date: "2026-08-24"
kategorie: "SMTP och e-postflöde"
timeToRead: "12 min. lästid"
themen:
  - smtp-mailflow
  - testing
produkte:
  - "uebergreifend"
protokolle:
  - "testing"
  - "smtp"
  - "tcp"
  - "tls"
  - "troubleshooting"
slug: "planera-e-postbelastningstester-verktyg-for-10-000-e-postburstar-under-linux-och-windows"
translationId: "article-14a98de0cef45565"
aiPrompt: |
  Du bist mein Assistent für Mail-Lasttests. Hilf mir Schritt für Schritt, einen Lasttest gegen mein eigenes Test-Mailgateway zu planen: Zieldefinition (Durchsatz, Latenz, Queue-Verhalten), Wahl des Lastgenerators (smtp-source, Postal, JMeter oder Skript), Aufbau einer Mail-Senke, Testablauf (Baseline, Burst, Soak) und Auswertung der Logs mit Perzentilen. Frage zuerst nach Plattform, Zielsystem und erwartetem Mailvolumen.
translationOf: mail-lasttest-tools-linux-windows-vergleich
translationSourceHash: 2fd0b1bd0748b9fb44be85907a946bbf85604b5eb7c85107170fa7443068efd7
translationModel: gpt-5.6-terra
translatedAt: 2026-08-30T09:28:02.259Z
translationReview: automatic
url: https://rafaelpfister.ch/sv/blog/planera-e-postbelastningstester-verktyg-for-10-000-e-postburstar-under-linux-och-windows
---

# Planera belastningstester för e-post: verktyg för e-postburstar på 10 000 under Linux och Windows jämförda

Om en ny e-postgateway klarar toppbelastningen från en nattlig fakturakörning går bara att kontrollera med ett belastningstest. Den som ersätter en appliance, dimensionerar en Exchange-miljö eller planerar ett nyhetsbrevsutskick via den egna infrastrukturen behöver tillförlitliga siffror i förväg: Hur många meddelanden per sekund tar systemet emot, hur beter sig kön under press och från vilken punkt börjar deferrals uppstå? Den här artikeln jämför vanliga lastgeneratorer under Linux och Windows och visar hur ett test med burstar på flera tiotusentals e-postmeddelanden planeras, genomförs och utvärderas.

Först den viktigaste regeln: Belastningstester hör uteslutande hemma i den egna infrastrukturen eller i en testmiljö som uttryckligen har godkänts för ändamålet. En burst mot externa system är en attack, och ett test med påhittade avsändaradresser mot produktionsmål skapar backscatter som leder till blocklistor. En korrekt uppsättning består av en lastgenerator, systemet som ska testas och en kontrollerad sink som slutligen tar emot och kasserar e-postmeddelandena.

## Vad ett belastningstest för e-post ska mäta

Innan ett verktyg kommer på tal är det värt att fråga vilken storhet som faktiskt är intressant. I praktiken finns det fyra olika, och de kräver olika testupplägg:

1. **Mottagningshastighet:** Hur många meddelanden per sekund tar första hoppet emot via SMTP? Detta är den klassiska genomströmningsmätningen och det värde som lastgeneratorer levererar direkt.
2. **Sessionslatens:** Hur lång tid tar en enskild SMTP-transaktion från anslutningens upprättande till `250` efter `DATA`? Under belastning ökar detta värde ofta långt innan mottagningshastigheten sjunker.
3. **End-to-end-latens:** Hur lång tid tar ett meddelande från inlämning till leverans till sinken, över alla mellanliggande stationer? Det är den storhet som användarna märker.
4. **Köbeteende:** Hur mycket växer kön under bursten och hur snabbt töms den efteråt? En gateway som tar emot 50'000 e-postmeddelanden och sedan arbetar av dem i tre timmar klarar mottagningstestet men misslyckas ändå.

Ett test som bara mäter mottagningshastigheten säger lite om en flerstegsmiljö med gateway, krypteringsnivå och målserver. Det är särskilt där som end-to-end-perspektivet avgör.

## Belastningsbilden avgör verktyget

Utöver mätstorheten avgör en andra fråga valet av verktyg, och den hoppas ofta över: Vilket anslutningsbeteende har belastningen som ska simuleras? Två belastningsbilder behöver skiljas åt.

En **bulkavsändare med öppna sessioner** är belastningsbilden för fakturakörningar, lönebesked och nyhetsbrevssystem: Ett enskilt system upprättar få anslutningar och skickar hundratals till tusentals meddelanden i följd över dem. Anslutningsöverhänget uppstår en gång per session, inte en gång per meddelande, och gatewayen ser få anslutningar med många transaktioner.

**Många oberoende inlämnare** är belastningsbilden för applikationslandskap och användartrafik: Många system lämnar vardera in enskilda meddelanden via egna anslutningar. Här ingår anslutningsupprättandet inklusive TLS och AUTH i varje meddelande.

För dimensionering av massutskick är den första belastningsbilden avgörande, och då måste lastgeneratorn kunna hålla sessioner öppna: `smtp-source` gör det (många meddelanden fördelade på få sessioner), liksom Postal och egna skript med beständig anslutning. JMeter kan inte det; bakgrunden förklaras i Windows-avsnittet. För toppbelastningen vid en fakturakörning är därför detta sessionskriterium avgörande, inte plattformen; under Windows går vägen då via WSL.

## Verktyg under Linux

**smtp-source och smtp-sink** från Postfix-paketet är standarden för rå SMTP-belastning och finns på praktiskt taget alla system där Postfix är installerat. `smtp-source` skapar meddelanden med inställbar storlek, parallellitet och antal, `smtp-sink` är motsvarigheten: en SMTP-server som tar emot och kasserar allt. En burst med 10'000 e-postmeddelanden med 50 parallella sessioner och meddelanden på 5 KB är en enda rad:

```bash
time smtp-source -s 50 -m 10000 -l 5120 -c -f last@test.example -t senke@test.example gateway.test.example:25
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `time` | Mäter anropets totala tid; därifrån fås hastigheten i e-postmeddelanden per sekund |
| `-s 50` | 50 parallella SMTP-sessioner |
| `-m 10000` | Totalt antal meddelanden, fördelade mellan sessionerna |
| `-l 5120` | Storleken på meddelandetexten i byte (utan rubriker), här 5 KB |
| `-c` | Löpande räknare över skickade meddelanden som förloppsindikator |
| `-f last@test.example` | Avsändaradress |
| `-t senke@test.example` | Mottagaradress |
| `gateway.test.example:25` | Målhost och port för inlämning |

</details>

Viktiga begränsningar: `smtp-source` mäter inga latenspercentiler och meddelandena är syntetiskt likformiga. För frågan ”hur mycket tar systemet som mest emot” är det ändå förstahandsvalet, eftersom det även på svag hårdvara skapar tiotusentals meddelanden per minut och generatorn i praktiken aldrig blir flaskhalsen.

**Postal** är det klassiska dedikerade riktmärket för e-postservrar under Linux. Det varierar avsändare, mottagare och meddelandestorlek automatiskt, håller en målhastighet under lång tid och skriver statistik per minut. Därmed passar det bättre än `smtp-source` för soak-tester, alltså kontinuerlig belastning under timmar. Tillhörande `bhm` (Black Hole Mailer) tar rollen som sink. Postal är gammalt, men byggt just för detta och ingår i paketkällorna för de flesta distributioner.

**swaks** är ingen lastgenerator, men hör hemma i varje testplan. Det genomför en enskild SMTP-transaktion med full kontroll över varje steg: autentisering, STARTTLS, valfria rubriker och bilagor. Före varje belastningstest bör en swaks-körning fungera som funktionstest, så att bursten inte misslyckas på grund av en felaktig mottagare eller ett TLS-problem och gör mätningen värdelös. I en slinga med `xargs -P` kan swaks även missbrukas som en liten lastgenerator, men för tiotusentals e-postmeddelanden är processöverhänget för stort.

**Egna skript** i Python (smtplib, aiosmtplib) eller Go är vägen när testet behöver realistiska meddelandemixar: olika storlekar, riktiga bilagor, varierande antal mottagare per transaktion och riktade felscenarier. Insatsen är större, men skriptet mäter då exakt det som den egna miljön senare ser och kan skriva tidsstämplar per meddelande för latensutvärderingen.

## Verktyg under Windows

**Apache JMeter** är rätt verktyg under Windows när belastningsbilden består av många oberoende inlämnare eller när percentiler, meddelandemix och rapporter står i fokus. Den inbyggda SMTP Sampler hanterar Auth, STARTTLS, bilagor och EML-filer som meddelandekälla, och JMeter-mekaniken tillhandahåller det som Postfix-verktygen saknar: trådgrupper för stegvisa belastningsprofiler, svarstidspercentiler, felfrekvenser och rapporter. För burstar på mer än några tusen e-postmeddelanden per minut gäller den vanliga JMeter-regeln: GUI endast för att skapa testplanen, kör själva mätningen i CLI-läge, annars mäter du gränssnittet också.

En begränsning hos SMTP Sampler måste vara känd: JMeter kan inte hålla SMTP-sessioner öppna. Varje sample-körning öppnar en ny anslutning, går igenom hela dialogen med TCP-handshake, EHLO, eventuellt STARTTLS och AUTH, skickar exakt ett meddelande och avslutar anslutningen med QUIT. Flera meddelanden över samma öppna anslutning, som massavsändare med återanvändning av sessioner gör, kan inte avbildas; `smtp-source` fördelar däremot många meddelanden på få öppna sessioner. Orsaken ligger i arkitekturen: JMeter är ett protokollöverskridande ramverk för belastningstestning, inte ett SMTP-verktyg. Dess körmodell behandlar varje sampler som en fristående, oberoende mätt enhet, eftersom bara detta gör att timers, assertions och percentilutvärdering fungerar enhetligt för alla protokoll som stöds. SMTP Sampler är därför ett tunt lager ovanpå JavaMail-biblioteket, som som klient-API upprättar och stänger en anslutning för varje sändning; återanvändning av anslutningar mellan samples, såsom HTTP Sampler erbjuder med Keep-Alive, har aldrig implementerats för SMTP. För mätningen innebär det: JMeter genererar belastningsbilden med många enskilda inlämnare, inte den hos en bulkavsändare med öppen session. Den uppmätta genomströmningen inkluderar det fulla anslutnings- och TLS-överhänget per meddelande, och anslutningsgränser på gatewayen slår därmed till tidigare än vid återanvändning av sessioner. För bulkavsändarens belastningsbild vid en fakturakörning är JMeter därför inte rätt verktyg; under Windows är WSL-vägen med `smtp-source` ett bättre val.

**PowerShell med MailKit** är vägen med inbyggda verktyg. Det tidigare vanliga `Send-MailMessage` är av Microsoft självt markerat som föråldrat och man rekommenderar en övergång; MailKit kan laddas via NuGet och parallelliseras från PowerShell 7 med Runspaces. Realistiskt kan man uppnå några hundra till några tusen e-postmeddelanden per minut, tillräckligt för funktions- och regressionstester men för lite för mätning av maximal belastning. Fördelen: Skriptet körs utan extra installation på varje administratörsarbetsplats och kan skriva resultat direkt som CSV för utvärdering.

**Microsoft Exchange Load Generator (LoadGen)** var under många år det officiella verktyget för att belasta Exchange-miljöer med simulerade användarprofiler (Outlook, ActiveSync, OWA). Microsoft har inte fortsatt underhålla det efter Exchange 2013 och har tagit bort nedladdningen. För ren SMTP-belastning var LoadGen ändå fel verktyg; den som i dag vill simulera belastning på Exchange-brevlådor står utan officiellt verktyg och testar SMTP-vägen bättre direkt.

**WSL** förtjänar en egen punkt: Den som sitter vid en Windows-dator men behöver Linux-verktyg kan installera `smtp-source` och Postal i en WSL-distribution och får därmed tillgång till alla Linux-verktyg utan en separat test-VM. För de belastningar som diskuteras här är WSL-nätverkssökvägen ingen relevant flaskhals.

## Jämförelse

| Verktyg | Plattform | Styrka | Begränsning |
|---|---|---|---|
| smtp-source / smtp-sink | Linux (Postfix) | Maximal råbelastning med minimal insats, generator och sink från samma källa | Inga latenspercentiler, likformiga meddelanden |
| Postal / bhm | Linux | Kontinuerlig belastning med målhastighet, varierande meddelanden, minutstatistik | Ålderdomliga verktyg, bygg utvärderingen själv |
| swaks | Linux, Windows (Perl) | Fullt styrbart enskilt test, idealiskt som funktionskontroll före bursten | Ingen lastgenerator |
| JMeter (SMTP Sampler) | Windows, Linux (Java) | Belastningsprofiler, percentiler, rapporter, EML-meddelandekällor | Java-överhäng, GUI-läge olämpligt för höga hastigheter, en anslutning per meddelande (ingen återanvändning av sessioner) |
| PowerShell + MailKit | Windows | Utan extra installation på varje administratörsdator, CSV-utdata | Begränsad genomströmning, bygg parallelliseringen själv |
| Eget skript (Python/Go) | båda | Realistisk meddelandemix, egna mätpunkter | Utvecklingsinsats, validera generatorn själv |

## Sinken: vart ska e-postmeddelandena ta vägen

Den underskattade hälften av testuppsättningen är målet. Tre varianter har visat sig fungera väl:

- **smtp-sink** eller `bhm` som svart hål: tar emot allt, kasserar allt och mäter den rena transportkedjan. `smtp-sink` kan vid behov skapa artificiella svarsfördröjningar och felkoder och därmed även kontrollera hur testsystemet beter sig vid ett långsamt eller felaktigt svarande mål.
- **Postfix med discard-transport** som mer realistisk sink, när målet självt ska vara en fullständig SMTP-server med köhantering.
- **Några få riktiga seed-brevlådor** utöver sinken, för att stickprovsvis kontrollera att meddelanden kommer fram innehållsmässigt intakta, inklusive krypterings- eller signaturnivå.

Verktyg med webbgränssnitt som Mailpit är avsedda för utveckling och blir snabbt själva flaskhalsen vid tiotusentals e-postmeddelanden. De är olämpliga som sink för ett belastningstest; mätningen skulle mäta analysverktyget i stället för testsystemet.

## Planera testet

Ett tillförlitligt test körs i tre steg, vart och ett med sin egen frågeställning:

1. **Baslinje:** En måttlig, känd belastning (ungefär 10 procent av den förväntade toppen) under några minuter. Den ger referensvärden för latens och resursförbrukning och avslöjar konfigurationsfel innan de försvinner i burstmätningen.
2. **Burst:** Själva mätningen av toppbelastningen, exempelvis 10'000 till 50'000 e-postmeddelanden så snabbt som möjligt eller med en definierad målhastighet. Flera körningar med ökande parallellitet visar var mottagningshastigheten planar ut och latensen tippar över.
3. **Soak:** Den förväntade dagliga belastningen under flera timmar. Först här syns minnesläckor, spool-partitioner som fylls, loggrotation under belastning och anslutningsgränser som en kort burst aldrig når.

För meddelandemixen gäller: så realistisk som behövs. En mätning med enbart textmeddelanden på 5 KB överskattar genomströmningen i en miljö där vardagen består av PDF-bilagor med flera gånger om. En blandning från det egna beståndet är lämplig, exempelvis 70 procent små, 25 procent med typisk bilaga och 5 procent stora. TLS ska också ingå i testet om produktionen använder TLS: Handshaken kostar betydligt mer per anslutning än själva meddelandeöverföringen, och generatorer som öppnar en ny anslutning per e-postmeddelande mäter annars främst TLS-termineringen.

För den senare utvärderingen får varje testmeddelande en unik markör, enklast en egen rubrik som `X-Loadtest-Id` med körnummer och tidsstämpel samt en igenkännbar ämneskonvention. Då kan testkörningar tydligt skiljas från varandra och från övrig trafik i loggarna, och testmeddelandena kan rensas riktat från karantäner och journaler efter körningen.

Tre punkter som regelbundet glöms bort i planeringen: För det första hastighetsgränser och tarpitting på testvägen; en gateway som stryper efter 100 e-postmeddelanden per minut och käll-IP testar annars bara sin egen strypning (undantag den medvetet för mätning av maximal belastning, låt den medvetet vara kvar för realitetskontrollen). För det andra DNS: Om testsystemet slår upp mottagardomäner eller gör DNSBL-frågor för varje meddelande, ska en resolver ingå i testmiljön, annars mäter testet uppströms-DNS. För det tredje generatorn själv: Före första körningen mot målsystemet ska generatorn köras direkt mot sinken för att visa att generatorn över huvud taget kan skapa målhastigheten.

## Utvärdera

Lastgeneratorns mätvärden är bara halva sanningen, eftersom de visar inlämnarens perspektiv. Den andra hälften finns i testsystemets loggar.

I Postfix ger maillog per meddelande fälten `delay` och `delays`, där det senare delas upp efter tid i kön, anslutningsupprättande och överföring. En utvärdering över en testkörning kan göras med inbyggda verktyg:

```bash
grep "status=sent" /var/log/mail.log |
  grep -o "delay=[0-9.]*" | cut -d= -f2 | sort -n |
  awk '{a[NR]=$1} END {print "n="NR, "p50="a[int(NR*0.5)], "p95="a[int(NR*0.95)], "p99="a[int(NR*0.99)], "max="a[NR]}'
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `grep "status=sent" /var/log/mail.log` | Filtrerar mailloggen till meddelanden som levererats utan fel |
| `grep -o "delay=[0-9.]*"` | `-o` visar bara själva träffen, här fältet `delay` med dess värde |
| `cut -d= -f2` | Delar vid `=` (`-d`) och behåller det andra fältet (`-f2`), alltså det numeriska värdet |
| `sort -n` | Sorterar numeriskt i stället för alfabetiskt; förutsättning för percentilberäkning |
| `awk '…'` | Samlar de sorterade värdena i en array och ger ut antal, p50, p95, p99 och maximum |

</details>

På Exchange-sidan är Message Tracking Log den centrala källan. För en testkörning med ämneskonvention:

```powershell
$p = @{
    Start          = "24.08.2026 14:00"
    End            = "24.08.2026 15:00"
    MessageSubject = "LOADTEST"
    ResultSize     = "Unlimited"
}
Get-MessageTrackingLog @p | Group-Object EventId |
    Sort-Object Count -Descending | Format-Table Name, Count
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `Start` / `End` | Tidsfönster för loggsökningen; överlämnas här via splatting (`@p`) |
| `MessageSubject "LOADTEST"` | Filtrerar efter meddelanden vars ämne innehåller markören |
| `ResultSize Unlimited` | Tar bort standardgränsen på 1000 returnerade poster |
| `Group-Object EventId` | Grupperar spårningshändelserna efter typ (RECEIVE, DELIVER, DEFER, …) |
| `Sort-Object Count -Descending` | Sorterar händelsegrupperna fallande efter frekvens |
| `Format-Table Name, Count` | Visar antalet per händelsetyp |

</details>

Skillnaden mellan tidsstämplarna för RECEIVE- och DELIVER-händelsen för samma MessageId ger end-to-end-latensen per meddelande; exporterat som CSV kan percentilfördelningen beräknas utifrån detta.

Vid tolkningen räknas tre principer. För det första: percentiler i stället för medelvärden. Ett genomsnitt på två sekunder kan innebära att allt tar två sekunder, eller att 95 procent är klara på en halv sekund medan resten fastnade i kön; p50, p95 och p99 skiljer dessa fall åt. För det andra: pivoter SMTP-svarskoder. Fördelningen av 4xx-svar över tid visar när systemet börjar strypa, och vilka koder det gäller (anslutningsgräns, köskydd, greylisting) visar vilken mekanism som griper in först. För det tredje: rita upp ködjupet över tid, under Postfix med `qshape` respektive `postqueue -j`, på Exchange med `Get-Queue` varje minut. Arean under denna kurva, inte mottagningshastigheten, avgör om miljön klarar en burst eller bara lagrar den.

Parallellt med e-postloggarna ska testsystemets systemmetriker ingå i utvärderingen: CPU, I/O-väntetider på spool-partitionen, filbeskrivare, anslutningsräknare. Det vanligaste fyndet i flerstegsmiljöer är att det inte är e-postprocessen som begränsar, utan ett content-inspection-steg (virusskanner, krypteringsmodul, DLP) med ett fast antal workers. Just sådana fynd är testets egentliga värde: De anger reglaget innan produktionen hittar det.

## Slutsats

För snabb mätning av maximal belastning under Linux går det inte att komma runt `smtp-source` med `smtp-sink`; Postal kompletterar för kontinuerlig belastning. Under Windows ger JMeter den mest fullständiga mätningen, PowerShell med MailKit täcker funktions- och regressionstester, och WSL hämtar vid behov Linux-verktygen till administratörsarbetsplatsen. Viktigare än verktyget är planen: separat mätning av mottagning, latens och köbeteende, en realistisk meddelandemix, en markerad testkörning och en utvärdering som inkluderar percentiler och målsystemets loggar i stället för enbart generatorns räknare.

## Källor

1.  [smtp-source(1), Postfix-manual](https://www.postfix.org/smtp-source.1.html): Referens för lastgeneratorn med alla alternativ för parallellitet, meddelandestorlek och TLS.

2.  [smtp-sink(1), Postfix-manual](https://www.postfix.org/smtp-sink.1.html): Referens för e-postsinken, inklusive artificiella fördröjningar och felsvar.

3.  [Postal-dokumentation, Russell Coker](https://doc.coker.com.au/projects/postal/): Beskrivning av e-postserverbenchmarken med målhastighet, meddelandevariation och bhm-sink.

4.  [swaks, John Jetmore](https://www.jetmore.org/john/code/swaks/): SMTP-funktionstestaren för kontroll i förväg av varje testväg.

5.  [Apache JMeter Component Reference: SMTP Sampler](https://jmeter.apache.org/usermanual/component_reference.html): Funktionalitet hos SMTP Sampler, inklusive Auth, TLS och EML-källor.

6.  [Send-MailMessage, Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/send-mailmessage): Officiellt meddelande från Microsoft om att cmdleten är föråldrad, med hänvisning till alternativ som MailKit.

7.  [MailKit, Jeffrey Stedfast](https://github.com/jstedfast/MailKit): .NET-e-postbiblioteket för egna sändskript under PowerShell 7.

8.  [Get-MessageTrackingLog, Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog): Referens för utvärdering av Exchange Message Tracking Log efter en testkörning.

9.  [qshape(1), Postfix-manual](https://www.postfix.org/qshape.1.html): Verktyg för analys av köfördelningen under och efter bursten.
