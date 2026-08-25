---
title: "Planera e-postbelastningstester: verktyg för 10 000 e-postburstar under Linux och Windows jämförda"
navTitle: "E-postbelastningstester"
description: "Den som migrerar en gateway eller dimensionerar en e-postmiljö behöver tillförlitliga siffror i stället för magkänsla. Vilka verktyg som skapar burstar på flera tiotusentals e-postmeddelanden, hur en välgjord testplan ser ut och hur du utvärderar resultaten från loggarna."
date: "2026-08-24"
kategorie: "SMTP och e-postflöde"
timeToRead: "12 min läsning"
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
url: https://rafaelpfister.ch/sv/blog/planera-e-postbelastningstester-verktyg-for-10-000-e-postburstar-under-linux-och-windows
translationSourceHash: c9b76f3c9887117756e07c71a3dc30d1ee99aeb8a322c50dee994a07df46cb97
translationModel: gpt-5.6-terra
translatedAt: 2026-08-25T04:12:54.362Z
translationReview: automatic
---

# Planera e-postbelastningstester: verktyg för 10 000 e-postburstar under Linux och Windows jämförda

Om en ny e-postgateway klarar toppbelastningen under en natt med fakturakörningar syns inte i databladet, utan i testet. Den som ersätter en appliance, dimensionerar en Exchange-miljö eller planerar ett nyhetsbrevsutskick via den egna infrastrukturen behöver tillförlitliga siffror i förväg: Hur många meddelanden per sekund tar systemet emot, hur beter sig kön under belastning och från vilken punkt börjar deferrals uppstå? Den här artikeln jämför vanliga belastningsgeneratorer under Linux och Windows och visar hur ett test med burstar på flera tiotusentals e-postmeddelanden planeras, genomförs och utvärderas.

Först den viktigaste regeln: Belastningstester hör uteslutande hemma i den egna infrastrukturen eller i en testmiljö som uttryckligen har godkänts för detta. En burst mot externa system är en attack, och ett test med påhittade avsändaradresser mot produktionsmål skapar backscatter som leder till blocklistor. En korrekt uppsättning består av en belastningsgenerator, systemet som ska testas och en kontrollerad sänka som slutligen tar emot och kastar bort e-postmeddelandena.

## Vad ett e-postbelastningstest ska mäta

Innan ett verktyg kommer på tal är det värt att fråga vilken storhet som egentligen är intressant. I praktiken finns det fyra olika, och de kräver olika testupplägg:

1. **Mottagningshastighet:** Hur många meddelanden per sekund tar första hoppet emot via SMTP? Det är den klassiska genomströmningsmätningen och det värde som belastningsgeneratorer levererar direkt.
2. **Sessionslatens:** Hur lång tid tar en enskild SMTP-transaktion från anslutningsupprättande till `250` efter `DATA`? Under belastning stiger detta värde ofta långt innan mottagningshastigheten faller.
3. **End-to-end-latens:** Hur lång tid behöver ett meddelande från inlämning till leverans till sänkan, över alla mellanliggande steg? Det är den storhet som användarna märker.
4. **Köbeteende:** Hur mycket växer kön under bursten och hur snabbt töms den efteråt? En gateway som tar emot 50 000 e-postmeddelanden och sedan arbetar av dem i tre timmar klarar mottagningstestet men misslyckas ändå.

Ett test som bara mäter mottagningshastigheten säger lite om en miljö i flera steg med gateway, krypteringslager och målserver. Särskilt där är end-to-end-perspektivet avgörande.

## Verktyg under Linux

**smtp-source och smtp-sink** från Postfix-paketet är standarden för rå SMTP-belastning och finns tillgängliga på praktiskt taget alla system där Postfix är installerat. `smtp-source` skapar meddelanden med inställbar storlek, parallellitet och antal, medan `smtp-sink` är motparten: en SMTP-server som tar emot och kastar bort allt. En burst med 10 000 e-postmeddelanden med 50 parallella sessioner och 5 KB-meddelanden är en enradsvariant:

```bash
time smtp-source -s 50 -m 10000 -l 5120 -c -f last@test.example -t senke@test.example gateway.test.example:25
```

Alternativet `-c` räknar skickade meddelanden löpande, `time` ger den totala tiden och därmed hastigheten. Viktiga begränsningar: `smtp-source` mäter inga latenspercentiler och meddelandena är syntetiskt likformiga. För frågan ”hur mycket systemet maximalt tar emot” är det ändå förstahandsvalet, eftersom det även på svag hårdvara skapar tiotusentals meddelanden per minut och generatorn i praktiken nästan aldrig blir flaskhalsen.

**Postal** är den klassiska dedikerade benchmarken för e-postservrar under Linux. Den varierar automatiskt avsändare, mottagare och meddelandestorlek, håller en målhastighet under långa tidsperioder och skriver statistik per minut. Därmed lämpar den sig bättre än `smtp-source` för soak-tester, alltså kontinuerlig belastning under flera timmar. Tillhörande `bhm` (Black Hole Mailer) tar rollen som sänka. Postal är gammalt, men byggt just för detta och finns i paketkällorna för de flesta distributioner.

**swaks** är ingen belastningsgenerator, men hör hemma i varje testplan. Det utför en enskild SMTP-transaktion med full kontroll över varje steg: autentisering, STARTTLS, valfria rubriker och bilagor. Före varje belastningstest bör en swaks-körning göras som funktionstest, så att bursten inte misslyckas på grund av en felaktig mottagare eller ett TLS-problem och gör mätningen värdelös. I en slinga med `xargs -P` kan swaks även missbrukas som en liten belastningsgenerator, men för tiotusentals e-postmeddelanden är processoverheaden för stor.

**Egna skript** i Python (smtplib, aiosmtplib) eller Go är rätt väg när testet behöver realistiska meddelandemixar: olika storlekar, riktiga bilagor, varierande antal mottagare per transaktion och riktade felscenarier. Arbetsinsatsen är större, men skriptet mäter exakt det som den egna miljön senare ser och kan skriva tidsstämplar per meddelande för latensutvärderingen.

## Verktyg under Windows

**Apache JMeter** är förstahandsrekommendationen under Windows. Den inbyggda SMTP Sampler hanterar autentisering, STARTTLS, bilagor och EML-filer som meddelandekälla, och JMeter-mekanismen erbjuder det som Postfix-verktygen saknar: trådgrupper för stegvisa belastningsprofiler, svarstidspercentiler, felfrekvenser och rapporter. För burstar över några tusen e-postmeddelanden per minut gäller den vanliga JMeter-regeln: använd GUI:t endast för att skapa testplanen och kör själva mätningen i CLI-läge, annars mäter du gränssnittet.

**PowerShell med MailKit** är vägen med inbyggda medel. Det tidigare vanliga `Send-MailMessage` markerar Microsoft självt som föråldrat och rekommenderar en övergång; MailKit kan laddas via NuGet och parallelliseras från PowerShell 7 med Runspaces. Realistiskt sett går det att nå några hundra till några tusen e-postmeddelanden per minut, tillräckligt för funktions- och regressionstester men för lite för mätning av maxbelastning. Fördelen är att skriptet körs utan extra installation på varje administratörsarbetsplats och kan skriva resultat direkt som CSV för utvärdering.

**Microsoft Exchange Load Generator (LoadGen)** var i många år det officiella verktyget för att belasta Exchange-miljöer med simulerade användarprofiler (Outlook, ActiveSync, OWA). Microsoft har inte vidareutvecklat det efter Exchange 2013 och har tagit bort nedladdningen. För ren SMTP-belastning var LoadGen ändå fel verktyg; den som i dag vill simulera Exchange-postlådebelastning står utan officiellt verktyg och testar SMTP-sökvägen bättre direkt.

**WSL** förtjänar en egen punkt: Den som sitter vid en Windows-dator men behöver Linux-verktyg installerar `smtp-source` och Postal i en WSL-distribution och får därmed hela Linux-verktygslådan utan en separat test-VM. För de belastningar som diskuteras här är WSL-nätverkssökvägen ingen relevant flaskhals.

## Jämförelse

| Verktyg | Plattform | Styrka | Begränsning |
|---|---|---|---|
| smtp-source / smtp-sink | Linux (Postfix) | Maximal råbelastning med minimal insats, generator och sänka från samma källa | Inga latenspercentiler, likformiga meddelanden |
| Postal / bhm | Linux | Kontinuerlig belastning med målhastighet, varierande meddelanden, minutstatistik | Ålderstiget verktygsstöd, bygg utvärderingen själv |
| swaks | Linux, Windows (Perl) | Fullt kontrollerbart enskilt test, idealiskt som funktionskontroll före bursten | Ingen belastningsgenerator |
| JMeter (SMTP Sampler) | Windows, Linux (Java) | Belastningsprofiler, percentiler, rapporter, EML-meddelandekällor | Java-overhead, GUI-fälla vid höga hastigheter |
| PowerShell + MailKit | Windows | Utan extra installation på varje administratörsdator, CSV-utdata | Begränsad genomströmning, bygg parallelliseringen själv |
| Eget skript (Python/Go) | båda | Realistisk meddelandemix, egna mätpunkter | Utvecklingsinsats, validera generatorn själv |

## Sänkan: vart tar e-postmeddelandena vägen

Den underskattade halvan av testuppsättningen är målet. Tre varianter har visat sig fungera väl:

- **smtp-sink** eller `bhm` som svart hål: tar emot allt, kastar bort allt och mäter den rena transportkedjan. `smtp-sink` kan på begäran skapa artificiella svarsfördröjningar och felkoder och därmed även testa testssystemets beteende vid ett långsamt eller motsträvigt mål.
- **Postfix med discard-transport** som mer realistisk sänka när målet självt ska vara en fullfjädrad SMTP-server med köhantering.
- **Ett fåtal riktiga seed-postlådor** utöver sänkan för att stickprovsvis kontrollera att meddelanden kommer fram innehållsmässigt oskadda, inklusive krypterings- eller signaturnivå.

Verktyg med webbgränssnitt som Mailpit är avsedda för utveckling och blir snabbt själva flaskhalsen vid tiotusentals e-postmeddelanden. De är olämpliga som sänka för ett belastningstest; mätningen skulle mäta analysverktyget i stället för testsystemet.

## Planera testet

Ett tillförlitligt test genomförs i tre steg, vart och ett med sin egen frågeställning:

1. **Baslinje:** En måttlig, känd belastning (omkring 10 procent av den förväntade toppen) under några minuter. Den ger referensvärden för latens och resursförbrukning och upptäcker konfigurationsfel innan de försvinner i burstmätningen.
2. **Burst:** Själva mätningen av toppbelastning, till exempel 10 000 till 50 000 e-postmeddelanden så snabbt som möjligt eller med en definierad målhastighet. Flera körningar med ökande parallellitet visar var mottagningshastigheten planar ut och latensen kollapsar.
3. **Soak:** Den förväntade dagsbelastningen under flera timmar. Först här visar sig minnesläckor, fulla spool-partitioner, loggrotation under belastning och anslutningsgränser som en kort burst aldrig når.

För meddelandemixen gäller: så realistisk som nödvändigt. En mätning med enbart 5 KB stora textmeddelanden överskattar genomströmningen i en miljö där vardagen består av PDF-bilagor med flera gånger om. En mix från det egna beståndet är lämplig, exempelvis 70 procent små, 25 procent med typisk bilaga och 5 procent stora. TLS ska också ingå i testet om produktionen använder TLS: handskakningen kostar betydligt mer per anslutning än själva meddelandeöverföringen, och generatorer som öppnar en ny anslutning för varje e-postmeddelande mäter annars främst TLS-termineringen.

För den senare utvärderingen får varje testmeddelande en unik markör, enklast en egen rubrik som `X-Loadtest-Id` med körningsnummer och tidsstämpel samt en igenkännbar ämneskonvention. Därmed kan testkörningar tydligt skiljas från varandra och från övrig trafik i loggarna, och testmeddelandena kan rensas selektivt från karantäner och journaler efter körningen.

Tre punkter som regelbundet glöms bort i planeringen: För det första hastighetsgränser och tarpitting i testsökvägen; en gateway som begränsar efter 100 e-postmeddelanden per minut och käll-IP testar annars bara sin egen begränsning (undantag den medvetet för maxbelastningsmätningen, låt den medvetet vara kvar för verklighetskontrollen). För det andra DNS: Om testsystemet slår upp mottagardomäner eller gör DNSBL-frågor för varje meddelande måste en resolver ingå i testmiljön, annars mäter testet upstream-DNS. För det tredje generatorn själv: Före den första körningen mot målsystemet bör generatorn köras direkt mot sänkan för att visa att den över huvud taget kan skapa målhastigheten.

## Utvärdera

Belastningsgeneratorns mätvärden är bara halva sanningen, eftersom de visar inlämnarens perspektiv. Den andra halvan finns i testsystemets loggar.

I Postfix ger mailloggen per meddelande fälten `delay` och `delays`, där det senare delas upp efter tid i kön, anslutningsupprättande och överföring. En utvärdering över en testkörning kan göras med inbyggda verktyg:

```bash
grep "status=sent" /var/log/mail.log |
  grep -o "delay=[0-9.]*" | cut -d= -f2 | sort -n |
  awk '{a[NR]=$1} END {print "n="NR, "p50="a[int(NR*0.5)], "p95="a[int(NR*0.95)], "p99="a[int(NR*0.99)], "max="a[NR]}'
```

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

Skillnaden mellan tidsstämplarna för RECEIVE- och DELIVER-händelsen för samma MessageId ger end-to-end-latensen per meddelande; när den exporteras som CSV kan percentilfördelningen beräknas.

Vid tolkningen räknas tre grundprinciper. För det första: percentiler i stället för medelvärden. Ett genomsnitt på två sekunder kan betyda att allt tar två sekunder, eller att 95 procent går igenom på en halv sekund och resten fastnar i kön; p50, p95 och p99 skiljer dessa fall åt. För det andra: pivotera SMTP-svarskoder. Fördelningen av 4xx-svar över tid visar när systemet börjar begränsa, och vilka koder det är (anslutningsgräns, köskydd, greylisting) visar vilken mekanism som griper in först. För det tredje: rita upp ködjupet över tid, i Postfix med `qshape` respektive `postqueue -j`, i Exchange med `Get-Queue` varje minut. Arean under denna kurva, inte mottagningshastigheten, avgör om miljön klarar en burst eller bara lagrar den.

Parallellt med e-postloggarna ska testsystemets systemmått ingå i utvärderingen: CPU, I/O-väntetider på spool-partitionen, filbeskrivare och anslutningsräknare. Det vanligaste resultatet i miljöer med flera steg är att det inte är e-postprocessen som begränsar, utan ett content-inspection-lager (virusskanner, krypteringsmodul, DLP) med ett fast antal arbetstrådar. Just sådana resultat är testets egentliga värde: De identifierar reglaget innan produktionen gör det.

## Slutsats

För snabb mätning av maxbelastning under Linux finns ingen väg runt `smtp-source` med `smtp-sink`; Postal kompletterar för kontinuerlig belastning. Under Windows ger JMeter den mest kompletta mätningen, PowerShell med MailKit täcker funktions- och regressionstester, och WSL hämtar vid behov Linux-verktygen till administratörsarbetsplatsen. Viktigare än verktyget är planen: separat mätning av mottagning, latens och köbeteende, en realistisk meddelandemix, en markerad testkörning och en utvärdering som inkluderar percentiler och målsystemets loggar i stället för enbart generatorns räknare.

## Källor

1.  [smtp-source(1), Postfix-manual](https://www.postfix.org/smtp-source.1.html): Referens för belastningsgeneratorn med alla alternativ för parallellitet, meddelandestorlek och TLS.

2.  [smtp-sink(1), Postfix-manual](https://www.postfix.org/smtp-sink.1.html): Referens för e-postsänkan, inklusive artificiella fördröjningar och felsvar.

3.  [Postal-dokumentation, Russell Coker](https://doc.coker.com.au/projects/postal/): Beskrivning av benchmarken för e-postservrar med målhastighet, meddelandevariation och bhm-sänka.

4.  [swaks, John Jetmore](https://www.jetmore.org/john/code/swaks/): SMTP-funktionstestaren för förhandskontrollen av varje testsökväg.

5.  [Apache JMeter Component Reference: SMTP Sampler](https://jmeter.apache.org/usermanual/component_reference.html): Funktionaliteten i SMTP Sampler, inklusive autentisering, TLS och EML-källor.

6.  [Send-MailMessage, Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/send-mailmessage): Officiellt meddelande från Microsoft om att cmdleten är föråldrad, med hänvisning till alternativ som MailKit.

7.  [MailKit, Jeffrey Stedfast](https://github.com/jstedfast/MailKit): .NET-e-postbiblioteket för egna sändningsskript under PowerShell 7.

8.  [Get-MessageTrackingLog, Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog): Referens för utvärdering av Exchange Message Tracking Log efter en testkörning.

9.  [qshape(1), Postfix-manual](https://www.postfix.org/qshape.1.html): Verktyg för analys av köfördelningen under och efter bursten.
