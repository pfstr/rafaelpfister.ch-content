---
title: "Hur länge förblir en SMTP-session öppen? ConnectionTimeout 00:10:00 i Exchange och de system där det är för kort"
navTitle: "SMTP-sessionens varaktighet"
description: "Exchange avslutar varje inkommande SMTP-session efter tio minuter, även om den just då överför data. Vilka avsändare som stannar så länge på en anslutning, hur du läser ut den faktiska sessionslängden från protokollloggen och när ConnectionTimeout och ConnectionInactivityTimeout bör justeras på en reläanslutning."
date: "2026-09-03"
kategorie: "SMTP och e-postflöde"
timeToRead: "10 min läsning"
themen:
  - smtp-mailflow
  - exchange-onprem-hybrid
produkte:
  - "exchange-on-premises"
  - "exchange"
protokolle:
  - "smtp"
  - "troubleshooting"
slug: "hur-lange-forblir-en-smtp-session-oppen-connectiontimeout-00-10-00-i-exchange-och-de-system-dar"
translationId: "article-b40497933bbe0a88"
aiPrompt: |
  Du bist mein Exchange- und Mailflow-Assistent. Hilf mir, die SMTP-Session-Dauer auf einem Exchange-Receive-Connector zu beurteilen: 1. Frage mich, welche Systeme (Relays, Gateways, Applikationen, Scanner) über den Connector einliefern und ob sie Verbindungen über mehrere Nachrichten hinweg offen halten. 2. Lass dir die Ausgabe der Session-Auswertung aus dem Protokoll-Log geben (IP, Mails, Dauer, Timeout-Kennzeichen) und erkläre mir, welche Sessions am ConnectionTimeout abgebrochen wurden. 3. Empfiehl pro Connector konkrete Werte für ConnectionTimeout und ConnectionInactivityTimeout und begründe, warum der internetseitige Connector unverändert bleibt. 4. Nenne mir, was ich stattdessen auf der Client-Seite ändern kann, damit die Verbindung nach einer festen Anzahl Nachrichten neu aufgebaut wird.
translationOf: smtp-session-dauer-exchange-connectiontimeout
url: https://rafaelpfister.ch/sv/blog/hur-lange-forblir-en-smtp-session-oppen-connectiontimeout-00-10-00-i-exchange-och-de-system-dar
translationSourceHash: a107c4edd960dabb30ba1b6f263a693808a5edf6815747d81f5d446c103a7e79
translationModel: gpt-5.6-terra
translatedAt: 2026-09-04T08:22:51.440Z
translationReview: automatic
---

# Hur länge förblir en SMTP-session öppen? ConnectionTimeout 00:10:00 i Exchange och de system där det är för kort

Kort slutsats: En SMTP-session har inget naturligt slut. RFC 5321 begränsar endast väntetiden för nästa steg, och en klient får fortsätta leverera meddelanden på en öppen anslutning så länge servern håller anslutningen öppen. Exchange håller den som standard öppen i tio minuter på Receive-anslutningar, sedan avslutar servern anslutningen oavsett om data just då överförs. För trafik mellan Exchange-servrar och för de flesta MTA:er saknar detta betydelse, eftersom dessa avsändare själva återansluter efter några sekunder. För applikationer, gateways och lastgeneratorer som använder en enda anslutning för en hel sändningskörning är värdet däremot orsaken till avbrott som visas som anslutningsfel i klienten och som `421 4.4.1 Connection timed out` i Exchange-protokollloggen.

## Två timeouter med olika betydelse

En Receive-anslutning har två tidsgränser som ofta förväxlas:

| Parameter | Betydelse | Standard för Mailbox-server | Standard för Edge Transport |
|---|---|---|---|
| `ConnectionInactivityTimeout` | maximal inaktiv tid utan klientaktivitet, därefter stängs anslutningen | 00:05:00 | 00:01:00 |
| `ConnectionTimeout` | maximal total varaktighet för anslutningen, även när den aktivt överför data | 00:10:00 | 00:05:00 |

Båda värdena accepterar 1 sekund till 1 dag (`1.00:00:00`), och `ConnectionTimeout` måste vara större än `ConnectionInactivityTimeout`. Värdena gäller per anslutning, alltså separat för den internetanslutna `Default Frontend <Server>`, anslutningen för transporttjänsten `Default <Server>` på port 2525 och varje reläanslutning som skapats separat.

Inaktivitetstimeouten är okritisk: Fem minuter motsvarar exakt den minimitid som RFC 5321 anger att en server ska vänta på nästa kommando, och en klient som inte skickar något på fem minuter har i regel själv glömt anslutningen. Den totala timeouten är en egenhet hos Exchange: Den räknas från anslutningens upprättande och fortsätter att löpa medan klienten levererar meddelande efter meddelande. Efter tio minuter stänger Exchange anslutningen där dialogen just då befinner sig, vid behov mitt i ett `DATA`-block.

På sändsidan finns ingen motsvarighet: En Send-anslutning har endast `ConnectionInactivityTimeOut` (standard tio minuter) och begränsar i stället sessioner via `SmtpMaxMessagesPerConnection`, som som standard är 20 meddelanden. Exchange avslutar alltså själv som klient varje anslutning senast efter 20 meddelanden och upprättar en ny. Det är anledningen till att den totala timeouten mellan Exchange-servrar aldrig märks: Sessionerna varar i sekunder.

## Vad RFC 5321 föreskriver

Standarden definierar i avsnitt 4.5.3.2 minsta väntetider som en klient ska iaktta för varje protokollsteg innan den ger upp anslutningen:

| Steg | Minsta timeout på klientsidan |
|---|---|
| Vänta på `220`-bannern | 5 minuter |
| Svar på `MAIL` | 5 minuter |
| Svar på `RCPT` | 5 minuter |
| Svar på `DATA` (den `354`) | 2 minuter |
| Skicka ett datablock | 3 minuter |
| Svar på den avslutande punkten | 10 minuter |
| Server: Vänta på nästa kommando | minst 5 minuter |

Det finns ingen övre gräns för en sessions totala varaktighet i RFC:en. En klient som levererar meddelanden på samma anslutning i trettio minuter och aldrig är tyst längre än några sekunder följer standarden. Det sista klientvärdet är anmärkningsvärt: Tio minuters väntetid på svaret efter den avslutande punkten, eftersom servern i denna fas accepterar och tar över meddelandet. Om klienten avbryter för tidigt här har meddelandet redan levererats och levereras en andra gång vid nästa försök. Samma situation uppstår spegelvänt när servern stänger anslutningen i detta ögonblick på grund av den totala timeouten.

Om en server stänger anslutningen med `421` ska klienten enligt avsnitt 3.8 behandla den pågående transaktionen som om den hade fått en `451`, alltså som ett tillfälligt fel med nytt försök. En MTA med kö gör just det. En applikation utan kö rapporterar i stället ett undantag och lämnar resten till anroparen.

## Hur länge avsändare faktiskt håller sina sessioner öppna

Sessionslängden bestäms av klienten, och skillnaderna mellan avsändartyperna är stora:

| Avsändare | Typisk sessionslängd | Begränsas av |
|---|---|---|
| Exchange Send-anslutning | Sekunder | `SmtpMaxMessagesPerConnection` = 20 |
| Postfix med anslutningscache | högst 5 minuter | `smtp_connection_reuse_time_limit` = 300s |
| Postfix utan anslutningscache | ett meddelande per anslutning | Standardbeteendet hos `smtp`-klienten |
| Applikation med `.NET SmtpClient`, `JavaMail Transport`, Python `smtplib` | så länge objektet lever: för en batchkörning hela körningen | endast programkoden |
| Karantännotifieringar från e-postgateways | en session per notifieringskörning | Produktbeteende, delvis med `NOOP`-keepalive |
| Multifunktionsenheter, skanna-till-e-post | ett meddelande per anslutning, flera minuter för stora skanningar över långsamma anslutningar | Filstorlek och bandbredd |
| Lastgeneratorer som `smtp-source -d` | till slutet av körningen | Anropsparametrar |

De två första raderna förklarar varför värdet inte märks av någon i klassiska miljöer under många år: MTA:er håller själva anslutningarna korta. Postfix använder till exempel en cachad anslutning i högst fem minuter och öppnar sedan en ny, medan Exchange kopplar ned efter 20 meddelanden. Båda ligger därmed under alla standardvärden i Exchange.

Applikationsraden är det vanligaste problemfallet. Ett batchjobb som skickar fakturor, lönebesked eller systemmeddelanden skapar vanligtvis ett klientobjekt, anropar sändmetoden på det i en slinga och stänger det i slutet. `System.Net.Mail.SmtpClient` använder sedan .NET Framework 4 samma anslutning för efterföljande `Send`-anrop och skickar `QUIT` först vid `Dispose`; JavaMail beter sig på samma sätt med en en gång öppnad `Transport`. Om jobbet kör längre än tio minuter inträffar `421` någonstans mitt i processen, och jobbet avbryts med ett undantag, till exempel med texten `Service not available, closing transmission channel. The server response was: 4.4.1 Connection timed out` i .NET. Vilket meddelande som påverkas beror på körtiden, därför verkar felet slumpmässigt: Ibland sker avbrottet efter 800 meddelanden, ibland efter 1200, beroende på meddelandestorlek och serverbelastning.

Gateway-raden beskriver ett dokumenterat fall: Symantec (numera Broadcom) Messaging Gateway skickar spamkarantännotifieringar via en enda anslutning och skickar `NOOP` som keepalive mellan meddelandena. Exchange besvarar `NOOP` med en Tarpit-fördröjning på fem sekunder, så att högst cirka 120 notifieringar hinner skickas på tio minuter innan sessionen avslutas med `421 4.4.1` och gatewayen måste återansluta.

Scanner-raden är ett storleksproblem snarare än ett mängdproblem: En skanning på 60 MB över en 2-Mbit/s-anslutning kräver omkring fyra minuter ren överföringstid, vid 100 MB nästan sju minuter. På en Edge Transport-server med fem minuters total timeout räcker detta redan för ett avbrott, på en Mailbox-server finns marginal men inte mycket.

## Vad som händer vid avbrottet

När den totala timeouten löper ut skriver Exchange svaret `421 4.4.1 Connection timed out` i protokollloggen, skickar det till klienten och stänger anslutningen. För den transaktion som pågår gäller följande: Om den avslutande punkten ännu inte har skickats har meddelandet inte accepterats och måste upprepas helt. Om punkten har skickats och anslutningen stängs före svaret `250` har klienten ingen information om huruvida Exchange har tagit över meddelandet; en korrekt implementerad klient upprepar det, och mottagaren kan då få det två gånger. Sannolikheten är liten, men inte noll vid tusentals meddelanden per körning.

Observera även proxysökvägen: Front End-transporttjänsten tar emot anslutningen på port 25 och vidarebefordrar den som en egen SMTP-session till transporttjänsten på port 2525, där anslutningen `Default <Server>` gäller med samma standardvärden. En lång session syns därför i båda loggarna, och en justering måste omfatta båda anslutningarna.

## Läs ut den faktiska sessionslängden från protokollloggen

Innan du ändrar ett värde är det värt att titta på de verkliga sessionerna. Förutsättningen är detaljerad protokollloggning på den berörda anslutningen; på `Default Frontend <Server>` är den redan aktiv, men inte på övriga anslutningar:

```powershell
Set-ReceiveConnector -Identity 'EX01\Relay Applikationen' -ProtocolLoggingLevel Verbose
```

Loggarna finns under `%ExchangeInstallPath%TransportRoles\Logs\FrontEnd\ProtocolLog\SmtpReceive` (Front End) och `%ExchangeInstallPath%TransportRoles\Logs\Hub\ProtocolLog\SmtpReceive` (transporttjänsten), och namnges efter UTC-timme som `RECVyyyyMMddhh-nnnn.log`. Varje rad är en protokollhändelse med fälten `date-time`, `connector-id`, `session-id`, `sequence-number`, `local-endpoint`, `remote-endpoint`, `event`, `data` och `context`. Alla rader i en session har samma `session-id`, så sessionslängden är skillnaden mellan den första och den sista tidsstämpeln för detta ID.

Följande skript utvärderar dagens senaste loggfil för en anslutning, sammanfattar raderna per session och visar de 15 längsta sessionerna med antal meddelanden, varaktighet och information om huruvida Exchange avslutade dem med `421 4.4.1`:

```powershell
$logPfad = Join-Path $env:ExchangeInstallPath 'TransportRoles\Logs\FrontEnd\ProtocolLog\SmtpReceive'
$connector = 'Relay Applikationen'
$tag = (Get-Date).ToUniversalTime().ToString('yyyyMMdd')
$datei = Get-ChildItem $logPfad -Filter "RECV$tag*.log" |
    Sort-Object Name -Descending |
    Select-Object -First 1

$sessions = @{}
Get-Content $datei.FullName |
    Where-Object { $_ -notlike '#*' -and $_ -like "*$connector*" } |
    ForEach-Object {
        $c = $_ -split ','
        $s = $c[2]
        if (-not $sessions[$s]) {
            $sessions[$s] = [pscustomobject]@{
                IP = ($c[5] -split ':')[0]; Start = $c[0]; Ende = $c[0]
                Zeilen = 0; Mails = 0; Timeout = $false
            }
        }
        $sessions[$s].Ende = $c[0]
        $sessions[$s].Zeilen++
        if ($c[7] -like 'MAIL FROM*') { $sessions[$s].Mails++ }
        if ($c[7] -like '421 4.4.1*') { $sessions[$s].Timeout = $true }
    }

$sessions.Values |
    Sort-Object Zeilen -Descending |
    Select-Object -First 15 IP, Mails, Zeilen, Timeout,
        @{ n = 'Dauer_s'
           e = { [math]::Round(([datetime]$_.Ende - [datetime]$_.Start).TotalSeconds, 1) } } |
    Format-Table -AutoSize
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Element | Effekt |
|---|---|
| `$logPfad` | Loggkatalog för Front End-transporttjänsten; använd `Hub` i stället för `FrontEnd` för transporttjänsten |
| `$connector` | Namndel för anslutningen; filtrerar via fältet `connector-id`, som loggas som `Server\Name` |
| `$tag` | UTC-datum, eftersom loggfilerna namnges efter UTC-timme |
| `-Filter "RECV$tag*.log"` | endast Receive-loggar från dagens datum |
| `Sort-Object Name -Descending`, `Select-Object -First 1` | den senaste filen (högsta timme, högsta instansnummer) |
| `$_ -notlike '#*'` | hoppar över rubrikraderna `#Software`, `#Version`, `#Log-Type`, `#Date`, `#Fields` |
| `$_ -split ','` | delar upp CSV-raden; de använda fälten 0, 2, 5 och 7 ligger före den första fritexten och är därför stabila |
| `$c[2]` | `session-id`, grupperingsnyckeln |
| `($c[5] -split ':')[0]` | IPv4-adress från `remote-endpoint` (för IPv6-slutpunkter måste uppdelningen anpassas) |
| `$c[0]` som `Start` och `Ende` | första och sista tidsstämpeln i sessionen; `Ende` skrivs över med varje rad |
| `$c[7] -like 'MAIL FROM*'` | räknar meddelanden via det mottagna kommandot `MAIL FROM` |
| `$c[7] -like '421 4.4.1*'` | markerar sessioner som Exchange avslutade på grund av den totala timeouten |
| `Sort-Object Zeilen -Descending` | de mest aktiva sessionerna först; alternativt sortera efter `Dauer_s` |
| `Dauer_s` | skillnaden mellan ISO-8601-tidsstämplarna i sekunder, avrundad till en decimal |

</details>

I utdata känner du igen de berörda systemen på att `Timeout` är satt till `True` och att `Dauer_s` ligger strax under 600: Sessionen har levt exakt så länge som anslutningen tillåter. Sessioner med många meddelanden och en varaktighet tydligt under 600 sekunder är okritiska, även om de för närvarande är de längsta. För en översikt över vilka källor som berörs räcker det att gruppera de markerade sessionerna:

```powershell
$sessions.Values |
    Where-Object { $_.Timeout } |
    Group-Object IP |
    Select-Object Name, Count |
    Sort-Object Count -Descending
```

Två begränsningar med metoden: En session som löper över en timgräns fördelas över två loggfiler och visas förkortad i en enskild fil; för en dagsutvärdering ska du läsa in alla filer från dagen. Och värdet `Mails` räknar `MAIL FROM`-kommandon, alltså försök, inte accepterade meddelanden.

## Justera värden: på vilken anslutning och hur mycket

Standardvärdena är ett skydd för den internetanslutna anslutningen, där godtyckliga motparter kan uppta anslutningar. Där förblir de oförändrade; en legitim extern MTA återansluter ändå. Justeringen görs på den dedikerade anslutning genom vilken de identifierade interna systemen levererar. Om en sådan anslutning saknas kan den skapas med `RemoteIPRanges` begränsad till avsändarnas IP-adresser; det är bättre än att höja värdet på `Default Frontend`. Aktuellt läge för alla anslutningar visas med:

```powershell
Get-ReceiveConnector |
    Format-Table Name, TransportRole, ConnectionTimeout, ConnectionInactivityTimeout, TarpitInterval -AutoSize
```

Själva justeringen, här med en timmes total varaktighet och oförändrad inaktivitetstimeout:

```powershell
$werte = @{
    ConnectionTimeout           = '01:00:00'
    ConnectionInactivityTimeout = '00:05:00'
}
Set-ReceiveConnector -Identity 'EX01\Relay Applikationen' @werte
Set-ReceiveConnector -Identity 'EX01\Default EX01' @werte
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Parameter | Effekt |
|---|---|
| `ConnectionTimeout` | Total varaktighet för en anslutning; tillåtet 00:00:01 till 1.00:00:00, måste vara högre än `ConnectionInactivityTimeout` |
| `ConnectionInactivityTimeout` | Inaktiv tid före stängning; fem minuter motsvarar RFC-minimumet och kan behållas |
| `-Identity 'EX01\Relay Applikationen'` | Front End-anslutningen för interna avsändare |
| `-Identity 'EX01\Default EX01'` | Transporttjänstens anslutning på port 2525, till vilken Front End vidarebefordrar sessionen |
| `@werte` | Splatting: skickar båda parametrarna från hashtabellen till cmdleten |

</details>

För värdet gäller följande: Det ska ligga över den längsta legitima session som utvärderingen har visat, med marginal för belastningstoppar. En timme täcker de flesta batchkörningar; för en nattlig körning på två timmar behövs motsvarande mer, upp till högsta värdet på en dag. Värdet bör dock inte sättas godtyckligt högt ens på en intern anslutning, eftersom `MaxInboundConnectionPerSource` (standard 20) och `MaxInboundConnection` (standard 5000) också räknas: En klient som fortsätter att öppna nya anslutningar utöver en anslutning som har hängt sig når gränsen per källa desto tidigare ju längre de gamla anslutningarna förblir öppna.

För avsändare som skickar `NOOP` mellan meddelanden bör `TarpitInterval` på samma anslutning ställas till `00:00:00`. Tarpit-fördröjningen har ingen nytta för autentiserade eller IP-begränsade interna avsändare och förlänger varje session artificiellt.

Ändringen på Exchange-sidan åtgärdar symptomet. Den stabilare lösningen finns i klienten: Den upprättar anslutningen på nytt efter ett fast antal meddelanden, på samma sätt som Exchange gör efter 20 och Postfix efter fem minuter. Med `.NET SmtpClient` innebär det att skapa och kassera objektet för varje block på exempelvis 100 meddelanden; i JavaMail stängs och öppnas `Transport` på nytt på motsvarande sätt. Då fungerar sändningen även mot mål vars timeouter inte kan justeras, särskilt Exchange Online, vars inkommande anslutningar saknar timeoutparametrar.

## Ytterligare tidsgränser längs vägen

Exchange-värdet är inte den enda gränsen. Brandväggar och lastbalanserare har egna inaktivitetstimers för TCP-anslutningar: En FastL4-profil på en F5 BIG-IP är som standard inställd på 300 sekunder, en Azure Load Balancer på fyra minuter. Dessa timers mäter inaktivitet, inte total varaktighet, och träder därför i kraft vid sändningspauser, till exempel när ett batchjobb läser data från databasen mellan två block. Det lägsta värdet på hela vägen är alltid avgörande. Hur du dimensionerar timeouter på en lastbalanserare för bestående SMTP-anslutningar beskrivs i artikeln [F5 BIG-IP som utgående proxy för massutskick av e-post](https://rafaelpfister.ch/blog/f5-big-ip-outbound-smtp-massenversand).

## Källor

1.  [Microsoft Learn: Set-ReceiveConnector](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-receiveconnector): Referens med standardvärden och värdeintervall för `ConnectionTimeout`, `ConnectionInactivityTimeout`, `TarpitInterval`, `MaxInboundConnection` och `MaxInboundConnectionPerSource` för Mailbox- och Edge Transport-servrar.

2.  [Microsoft Learn: Set-SendConnector](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-sendconnector): `ConnectionInactivityTimeOut` och `SmtpMaxMessagesPerConnection` på sändsidan.

3.  [Microsoft Learn: Protocol logging](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/protocol-logging): Lagringsplatser, filnamn och fältstruktur för SMTP-protokollloggar för Front End och transporttjänsten.

4.  [Microsoft Learn: Mail routing in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing): Front End-transporttjänsten som tillståndslös proxy framför transporttjänsten.

5.  [RFC 5321, avsnitt 4.5.3.2 Timeouts](https://www.rfc-editor.org/rfc/rfc5321.html#section-4.5.3.2): Minsta väntetider per protokollsteg, motiveringen för de tio minuterna efter den avslutande punkten och beteendet vid `421` i avsnitt 3.8.

6.  [Postfix: postconf(5)](https://www.postfix.org/postconf.5.html): `smtp_connection_reuse_time_limit` (300s) och `smtpd_timeout` som exempel på en MTA som själv håller sessioner korta.

7.  [Broadcom Knowledge Base: Quarantine notification process appears to be failing, logs may show 421 4.4.1 Connection timed out](https://knowledge.broadcom.com/external/article/154389/quarantine-notification-process-appears.html): Dokumenterat fall av en gateway som med `NOOP`-keepalive och Tarpit når Exchange-totaltimeouten.

8.  [Microsoft Learn: SmtpClient Class](https://learn.microsoft.com/en-us/dotnet/api/system.net.mail.smtpclient): Återanvändning av anslutningar över flera `Send`-anrop och `QUIT` först vid `Dispose`.
