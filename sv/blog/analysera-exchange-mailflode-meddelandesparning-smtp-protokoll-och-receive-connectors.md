---
title: "Analysera e-postflödet i Exchange: meddelandespårning, SMTP-loggar och Receive Connectors"
navTitle: "Analysera e-postflödet"
description: "Så tar du systematiskt reda på var ett meddelande har tagit vägen i Exchange OnPrem, Hybrid och Exchange Online: frågor med exempelutdata, hur du läser SMTP-loggen korrekt och fallgroparna som regelbundet leder till felaktiga slutsatser."
date: "2026-08-11"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "22 minuters läsning"
themen:
  - exchange-onprem-hybrid
  - smtp-mailflow
  - microsoft-365-exchange
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange"
  - "exchange-on-premises"
  - "exchange-hybrid"
  - "exchange-online"
protokolle:
  - "smtp"
  - "powershell"
  - "troubleshooting"
related:
  - typische-ursachen-fuer-mail-loops-und-deren-behebung
  - einliefernde-ip-adressen-aggregieren
  - mailflow-fehlersuche-kontrollierte-experimente
slug: "analysera-exchange-mailflode-meddelandesparning-smtp-protokoll-och-receive-connectors"
translationId: "article-ad93c41ab6cd20e6"
draft: false
translationOf: exchange-message-tracking-und-receive-connectoren-analysieren
translationSourceHash: 867f3a9a27b8db05d2721cce6282149f4616c4170576d0ab3f39a3cbde3701e3
translationModel: gpt-5.6-terra
translatedAt: 2026-10-30T03:27:43.663Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/analysera-exchange-mailflode-meddelandesparning-smtp-protokoll-och-receive-connectors
---

# Analysera e-postflödet i Exchange: meddelandespårning, SMTP-loggar och Receive Connectors

Den vanligaste frågan i e-postdriften är: Ett meddelande har inte kommit fram, var har det tagit vägen? Meddelandespårning besvarar detta tillförlitligt, men bara om du vet vad den **inte** berättar. Den här artikeln beskriver arbetsgången i den ordning som har visat sig fungera, visar typisk utdata för varje fråga och nämner felkällorna som regelbundet kostar timmar eftersom de antyder rimliga men felaktiga slutsatser.

Alla exempel använder generiska namn: `SRV-MAIL01` och `SRV-MAIL02` som transportservrar, `example.com` som domän. Om du vill sätta ihop kommandona för din miljö i stället för att skriva dem: [Kommandogeneratorn](https://rafaelpfister.ch/tools/command-builder) innehåller vanliga kommandon för meddelandespårning och insamling för PowerShell och Unix-skal sida vid sida, helt lokalt i webbläsaren.

## Grundprincipen: lokalisera först, förklara sedan

Reflexen är att genast söka efter orsaken. Det är effektivare att först avgöra hur långt meddelandet över huvud taget har kommit. Det begränsar sökområdet drastiskt i ett enda steg, eftersom du därefter vet om du behöver söka i ditt eget system, hos den föregående gatewayen eller vid målet.

Ordningen är därför: hitta meddelandet, läs den senaste händelsen, läs felorsaken, avgör om det är ett enskilt fall eller ett mönster och rekonstruera först därefter inskickningsvägen.

## Steg 1: Hitta meddelandet

Börja med mottagaren, eftersom du nästan alltid känner till den. Det är viktigt att köra frågan mot **alla** transportservrar, inte bara en.

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object {
    Get-MessageTrackingLog -Server $_ `
        -Start (Get-Date).AddHours(-6) `
        -ResultSize Unlimited `
        -Recipients "empfaenger@example.com"
} | Sort-Object Timestamp |
    Format-Table Timestamp, ServerHostname, EventId, Source, ConnectorId, MessageId `
        -AutoSize -Wrap
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-Server` | Transportserver vars spårningslogg frågas; här båda servrarna i följd via pipelinen |
| `-Start` | Nedre tidsgräns för sökningen, här de senaste sex timmarna |
| `-ResultSize Unlimited` | Tar bort standardgränsen på 1000 poster |
| `-Recipients` | Filtrerar efter meddelanden till denna mottagaradress |
| `Sort-Object Timestamp` | Sorterar de sammanfogade resultaten från båda servrarna kronologiskt |
| `-AutoSize -Wrap` | Anpassar kolumnbredden efter innehållet och radbryter långa värden i stället för att klippa av dem |

</details>

Typisk utdata för ett meddelande som har passerat utan problem:

```text
Timestamp           ServerHostname EventId      Source  ConnectorId
---------           -------------- -------      ------  -----------
11.08.2026 08:27:15 SRV-MAIL02     HARECEIVE    SMTP
11.08.2026 08:27:15 SRV-MAIL01     RECEIVE      SMTP    SRV-MAIL01\Default SRV-MAIL01
11.08.2026 08:27:15 SRV-MAIL01     HAREDIRECT   SMTP
11.08.2026 08:27:15 SRV-MAIL01     RESOLVE      ROUTING
11.08.2026 08:27:15 SRV-MAIL01     AGENTINFO    AGENT
11.08.2026 08:27:16 SRV-MAIL01     SENDEXTERNAL SMTP    Outbound-to-O365
11.08.2026 08:27:53 SRV-MAIL02     HADISCARD    SMTP
```

Om frågan inte hittar något, kontrollera om mottagaren har expanderats via en distributionsgrupp. Sök då hellre via avsändaren:

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object {
    Get-MessageTrackingLog -Server $_ `
        -Start (Get-Date).AddHours(-6) `
        -ResultSize Unlimited
} | Where-Object { $_.Sender -like "*@example.com" } |
    Sort-Object Timestamp |
    Format-Table Timestamp, EventId, Sender,
        @{n='To'; e={$_.Recipients -join ','}}, MessageSubject -AutoSize -Wrap
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-Server` | Transportserver vars spårningslogg frågas |
| `-Start` | Nedre tidsgräns för sökningen |
| `-ResultSize Unlimited` | Tar bort standardgränsen på 1000 poster |
| `Where-Object` | Filtrerar på klientsidan efter avsändare i den egna domänen, eftersom `-Sender` endast accepterar exakta adresser |
| `@{n=…; e=…}` | Beräknad kolumn: sammanfattar samlingsfältet `Recipients` till en kommaseparerad sträng |

</details>

## Steg 2: Läs den senaste händelsen

Hela diagnosen hänger på meddelandets **senaste** `EventId`. Den talar om för dig vilket sökområde som ska undersökas härnäst.

| Senaste EventId | Betydelse | Nästa steg |
|---|---|---|
| `RECEIVE`, därefter ingenting | Meddelandet har fastnat | Kontrollera köer |
| `SEND` eller `SENDEXTERNAL` | Överlämnat utan problem | Sök vidare vid nästa hopp |
| `FAIL` | Slutgiltigt misslyckat | Läs orsaken i `RecipientStatus` |
| `DEFER` | Nytt försök pågår | Kontrollera kön och målsystemet |
| `DROP` eller `POISONMESSAGE` | Kasserat | Kontrollera transportregel eller agent |
| `DELIVER` | Levererat till en lokal postlåda | Kontrollera postlåderegler |
| `RESOLVE` | Mottagaren har skrivits om | Läs måladressen i posten |

`RESOLVE` är det mest informativa mellansteget i hybridmiljöer, eftersom omskrivningen till molnroutningsadressen syns där:

```text
EventId : RESOLVE
Source  : ROUTING
Sender  : absender@example.com
To      : BENUTZER@example.mail.onmicrosoft.com
```

Om den förväntade `onmicrosoft.com`-adressen står där är mottagarobjektet korrekt konfigurerat och du kan avsluta ärendet. Om den ursprungliga adressen fortfarande står där saknar det lokala objektet måladressen, och Exchange försöker leverera lokalt.

Om meddelandet har fastnat visar kön oftast orsaken i klartext:

```powershell
Get-Queue -Server SRV-MAIL01 |
    Where-Object { $_.MessageCount -gt 0 } |
    Format-Table Identity, DeliveryType, Status, MessageCount, NextHopDomain, LastError `
        -AutoSize -Wrap
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-Server` | Server vars transportköer frågas |
| `Where-Object` | Döljer tomma köer och visar endast köer med väntande meddelanden |
| `-AutoSize -Wrap` | Förhindrar att den långa kolumnen `LastError` klipps av |

</details>

## Felkälla 1: Spårning är serverbaserad och många poster är skuggkopior

Om du ser par av `HARECEIVE` och `HADISCARD` i utdata, ofta med tillägget `ExplicitlyDiscarded`, har denna server **inte behandlat** meddelandet. Den höll endast en skuggkopia inom Shadow Redundancy, medan en annan server hanterade den faktiska leveransen. Så snart den primära servern rapporterar framgång kasserar partnern sin kopia.

Så här ser det ut när du bara har frågat fel server:

```text
Timestamp           EventId    SourceContext
---------           -------    -------------
11.08.2026 09:47:13 HARECEIVE  1a2aae6b-f3a3-4e04-9233-7de460b92223
11.08.2026 09:48:07 HADISCARD  ExplicitlyDiscarded;1a2aae6b-f3a3-4e04-9233-7de460b92223
```

Två rader, inget fel, ingen leverans. Den som drar slutsatsen att meddelandet har försvunnit söker på fel plats. Den faktiska behandlingen finns i partnerserverns spårning.

I praktiken betyder det två saker. För det första är sådana rader inte en indikation på ett problem, utan normal drift. För det andra måste du alltid fråga alla transportservrar.

## Felkälla 2: `Format-Table` klipper av just de avgörande kolumnerna

Felorsaken finns i `RecipientStatus`, och det fältet är långt. I en tabell försvinner det antingen helt eller klipps av. Det leder just till att man ser `FAIL` men inte orsaken och i stället börjar gissa.

Så snart du har hittat ett fel fall bör du därför byta till `Format-List` och expandera samlingsfälten:

```powershell
Get-MessageTrackingLog -Server SRV-MAIL01 `
    -Start (Get-Date).AddHours(-6) `
    -ResultSize Unlimited `
    -Recipients "empfaenger@example.com" `
    -EventId FAIL |
  Format-List Timestamp, Sender,
    @{n='To';     e={$_.Recipients -join ','}},
    @{n='Status'; e={$_.RecipientStatus -join ' | '}},
    MessageSubject, MessageId, SourceContext
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-Server` | Transportserver vars spårningslogg frågas |
| `-Start` | Nedre tidsgräns för sökningen |
| `-ResultSize Unlimited` | Tar bort standardgränsen på 1000 poster |
| `-Recipients` | Filtrerar efter meddelanden till denna mottagaradress |
| `-EventId FAIL` | Endast poster med slutgiltigt leveransfel |
| `Format-List` | Visar varje fält på en egen rad i full längd, inget klipps av |
| `@{n=…; e=…}` | Beräknade fält: expanderar samlingsfälten `Recipients` och `RecipientStatus` till läsbara strängar |

</details>

Så här ser skillnaden ut. Först tabellvyn som inte förklarar någonting:

```text
Timestamp           EventId ConnectorId
---------           ------- -----------
11.08.2026 09:47:13 FAIL    Outbound-to-O365
```

Sedan samma meddelande som lista:

```text
Timestamp      : 11.08.2026 09:47:13
Sender         : dienst@example-test.com
To             : BENUTZER@example.mail.onmicrosoft.com
Status         : [{LED=550 5.1.8 Access denied, bad outbound sender AS(42000001)
                 [XX1PEPF00000000.eurprd02.prod.outlook.com]};{MSG=};
                 {FQDN=10.0.0.40};{IP=10.0.0.40};{LRT=11.08.2026 09:47:13}]
MessageSubject : Statusmeldung Nachtlauf
MessageId      : <1897281176.1319@app01.intern.example.com>
```

Diagnosen är därmed klar, utan att du behövde göra en enda gissning: Motparten invänder mot avsändaren. `LED` innehåller det fullständiga SMTP-svaret, `FQDN` och `IP` anger systemet som svarade, och `LRT` tidpunkten för det senaste försöket.

## Steg 3: Enskilt fall eller mönster?

Innan du fördjupar dig i ett enskilt fall ska du klargöra omfattningen. Denna enda fråga avgör om du har att göra med en randanmärkning eller en incident:

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object {
    Get-MessageTrackingLog -Server $_ `
        -Start (Get-Date).AddHours(-8) `
        -EventId FAIL -ResultSize Unlimited
} | Where-Object { ($_.RecipientStatus -join '') -like "*5.1.8*" } |
    Group-Object { ($_.Sender -split '@')[-1] } |
    Sort-Object Count -Descending |
    Format-Table Count, Name -AutoSize
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-Start` | Nedre tidsgräns, här de senaste åtta timmarna |
| `-EventId FAIL` | Endast slutgiltigt misslyckade leveranser |
| `-ResultSize Unlimited` | Tar bort standardgränsen på 1000 poster |
| `Where-Object` | Filtrerar efter den undersökta SMTP-statuskoden i fältet `RecipientStatus` |
| `Group-Object` | Grupperar efter avsändardomän (delen efter `@`) |
| `Sort-Object Count -Descending` | Vanligaste domänen överst |

</details>

Ersätt `5.1.8` med statuskoden som du undersöker. Utdata besvarar frågan på en rad:

```text
Count Name
----- ----
    9 example-test.com
```

En enda avsändardomän innebär: ett snävt avgränsat problem, ingen incident, du kan fortsätta söka i lugn och ro. Om det stod tjugo olika domäner där skulle du ha ett pågående avbrott och allt annat behöva vänta. Att göra denna åtskillnad så tidigt sparar enligt erfarenhet mest tid.

## Felkälla 3: `ConnectorId` anger inte den verkliga Receive Connectorn

Det här är den dyraste felkällan, eftersom utdata ser seriös ut. E-post som en klient eller ett externt system skickar in på port 25 träffar först **Front End Transport**. Den vidarebefordrar meddelandet till **Transport Service** på port 2525. Meddelandespårningen skrivs först där; Front End Transport skriver ingen egen spårning.

Följden syns på denna rad:

```text
EventId        : RECEIVE
ConnectorId    : SRV-MAIL01\Default SRV-MAIL01
ClientIp       : 10.0.1.11
ClientHostname : srv-mail01.intern.example.com
```

`ConnectorId` anger den interna connectorn på port 2525, och `ClientIp` är adressen till **den proxande servern**, inte den ursprungliga avsändaren. Vilken av de konfigurerade connectorerna på port 25 ett system faktiskt träffade framgår helt enkelt inte av spårningen. Den som litar på denna uppgift söker felet vid en connector som inte ens var inblandad.

Det finns två sätt att få svaret. Det första är rekonstruktion via konfigurationen:

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object { Get-ReceiveConnector -Server $_ } |
    Format-List Identity, Enabled,
        @{n='Bindings';       e={$_.Bindings -join ','}},
        @{n='RemoteIPRanges'; e={$_.RemoteIPRanges -join ','}},
        PermissionGroups, AuthMechanism
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-Server` | Server vars Receive Connectors listas |
| `Format-List` | Fullständiga fältlängder; `RemoteIPRanges` och `PermissionGroups` skulle klippas av i tabeller |
| `@{n=…; e=…}` | Beräknade fält: sammanfattar samlingsfälten `Bindings` och `RemoteIPRanges` till kommaseparerade strängar |

</details>

```text
Identity         : SRV-MAIL01\Default Frontend SRV-MAIL01
Bindings         : 10.0.1.11:25
RemoteIPRanges   : 0.0.0.0-255.255.255.255
PermissionGroups : AnonymousUsers, ExchangeServers, ExchangeLegacyServers
AuthMechanism    : Tls, Integrated, BasicAuth, BasicAuthRequireTLS, ExchangeServer

Identity         : SRV-MAIL01\smtp-noauth SRV-MAIL01
Bindings         : 10.0.1.13:25
RemoteIPRanges   : 10.0.20.22,10.0.21.11,10.0.21.12
PermissionGroups : AnonymousUsers, Custom
AuthMechanism    : Tls
```

Fastställ käll-IP-adressen för det inskickande systemet och leta efter connectorn vars `RemoteIPRanges` innehåller den. Om den inte omfattas av någon av de begränsade connectorerna återstår standard-frontend-connectorn, som vanligtvis accepterar hela adressutrymmet. Använd även här `Format-List`, eftersom `RemoteIPRanges` och `PermissionGroups` regelbundet klipps av i tabeller.

Det andra sättet är SMTP-loggen, och den förtjänar ett eget avsnitt.

## SMTP-loggen: den enda kompletta källan

Loggen för Front End Transport registrerar hela SMTP-sessionen: vilken connector som kontaktades, vilken IP-adress som anslöt och vad klient och server sade till varandra. Det är den enda källan som löser det ovan beskrivna problemet med `ConnectorId`.

### Aktivera loggning

Som standard är loggning **avstängd** på de flesta connectorer. Du aktiverar den per connector:

```powershell
Set-ReceiveConnector -Identity "SRV-MAIL01\Default Frontend SRV-MAIL01" `
    -ProtocolLoggingLevel Verbose
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-Identity` | Connectorn som ska ändras i formatet `Server\Connectorname` |
| `-ProtocolLoggingLevel Verbose` | Aktiverar SMTP-loggning; `None` stänger av den igen |

</details>

För utgående anslutningar används motsvarande `Set-SendConnector`. Kom ihåg att återställa värdet till `None` efter analysen, eftersom detaljerad loggning kostar diskutrymme och skriver stora mängder data vid hög belastning.

### Var filerna finns

Exchange separerar loggarna efter tjänst och riktning. Det är onödigt att hårdkoda sökvägarna, fråga efter dem:

```powershell
Get-FrontendTransportService SRV-MAIL01 |
    Format-List ReceiveProtocolLogPath, SendProtocolLogPath,
        ReceiveProtocolLogMaxAge, ReceiveProtocolLogMaxDirectorySize

Get-TransportService SRV-MAIL01 |
    Format-List ReceiveProtocolLogPath, SendProtocolLogPath
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `SRV-MAIL01` | Positionsparametern `-Identity`: servern som ska frågas |
| `ReceiveProtocolLogPath`, `SendProtocolLogPath` | Sökvägar till loggarna för inkommande respektive utgående anslutningar |
| `ReceiveProtocolLogMaxAge` | Maximal ålder för loggfilerna; äldre filer tas bort |
| `ReceiveProtocolLogMaxDirectorySize` | Övre gräns för loggkatalogens diskutrymmesanvändning |

</details>

Vanligtvis finns de under installationssökvägen i `TransportRoles\Logs\FrontEnd\ProtocolLog\SmtpReceive` för Front End Transport och i `TransportRoles\Logs\Hub\ProtocolLog\SmtpReceive` för Transport Service. **Detta är kärnan:** Klientanslutningar på port 25 hittar du uteslutande i sökvägen `FrontEnd`; i sökvägen `Hub` finns endast den interna vidarebefordringstrafiken på 2525.

Observera lagringen. `ReceiveProtocolLogMaxAge` är ofta inställt på 30 dagar, medan `ReceiveProtocolLogMaxDirectorySize` dessutom begränsar diskutrymmet. Vid hög belastning träder storleksgränsen i kraft betydligt tidigare än åldersgränsen, och då är dina loggar bara några dagar gamla.

### Förstå formatet

Filerna är CSV med rubrikrader som börjar med `#`. De viktigaste kolumnerna är `date-time`, `connector-id`, `session-id`, `sequence-number`, `local-endpoint`, `remote-endpoint`, `event` och `data`.

Avgörande är kolumnen `event`, ett enda tecken:

| Tecken | Betydelse |
|---|---|
| `+` | Anslutning upprättad |
| `-` | Anslutning avslutad |
| `>` | Servern skickar till klienten |
| `<` | Klienten skickar till servern |
| `*` | Serverinformation, ingen SMTP-trafik |

Du känner igen en session på den gemensamma `session-id`; `sequence-number` anger ordningen inom sessionen. Ett typiskt utdrag ser ut så här:

```text
2026-08-11T09:47:10.4Z,SRV-MAIL01\smtp-noauth SRV-MAIL01,08DEF44EC454A414,0,
  10.0.1.13:25,10.0.20.22:51244,+,,
2026-08-11T09:47:10.4Z,SRV-MAIL01\smtp-noauth SRV-MAIL01,08DEF44EC454A414,1,
  10.0.1.13:25,10.0.20.22:51244,>,"220 srv-mail01.intern.example.com Microsoft ESMTP",
2026-08-11T09:47:10.5Z,SRV-MAIL01\smtp-noauth SRV-MAIL01,08DEF44EC454A414,2,
  10.0.1.13:25,10.0.20.22:51244,<,EHLO app01.intern.example.com,
2026-08-11T09:47:10.6Z,SRV-MAIL01\smtp-noauth SRV-MAIL01,08DEF44EC454A414,6,
  10.0.1.13:25,10.0.20.22:51244,<,MAIL FROM:<dienst@example-test.com>,
2026-08-11T09:47:10.7Z,SRV-MAIL01\smtp-noauth SRV-MAIL01,08DEF44EC454A414,8,
  10.0.1.13:25,10.0.20.22:51244,>,"250 2.1.5 Recipient OK",
```

Här finns allt som saknades i meddelandespårningen: den **verkliga** connectorn (`smtp-noauth`), den **verkliga** käll-IP-adressen (`10.0.20.22`) och namnet som systemet anmäler sig med i `EHLO`.

### Sök riktat

För enskilda fall är ett textfilter betydligt snabbare än objektutvärdering. Sök efter avsändaradressen eller namnet i `EHLO` och låt sessionens identifierare visas:

```powershell
$pfad = (Get-FrontendTransportService SRV-MAIL01).ReceiveProtocolLogPath
Select-String -Path "$pfad\*.log" -Pattern "dienst@example-test.com" -SimpleMatch |
    Select-Object -First 5 Line
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-Path "$pfad\*.log"` | Söker igenom alla loggfiler i den tidigare hämtade sökvägen |
| `-Pattern` | Söktermen, här avsändaradressen |
| `-SimpleMatch` | Behandlar mönstret som text i stället för reguljärt uttryck; punkten i adressen behöver då inte escapas |
| `-First 5` | Begränsar utdata till de första fem träffarna |

</details>

Med den hittade `session-id` hämtar du hela sessionen:

```powershell
Select-String -Path "$pfad\*.log" -Pattern "08DEF44EC454A414" -SimpleMatch |
    ForEach-Object { $_.Line } | Select-Object -First 40
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-Pattern` | Sessionsidentifieraren från den första träffen |
| `-SimpleMatch` | Bokstavlig sökning utan regex-utvärdering |
| `-First 40` | Begränsar utdata till de första 40 raderna i sessionen |

</details>

Om du bara vill veta vilka connectorer som över huvud taget ser trafik räknar du anslutningsuppbyggnaderna. Det är storleksordningar snabbare för stora filer än att parsa varje rad:

```powershell
Select-String -Path "$pfad\*.log" -Pattern ',\+,' |
    ForEach-Object { ($_.Line -split ',')[1] } |
    Group-Object | Sort-Object Count -Descending |
    Format-Table Count, Name -AutoSize
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-Pattern ',\+,'` | Reguljärt uttryck för händelsen `+` (anslutningsuppbyggnad) mellan två CSV-kommatecken; plustecknet är escapat |
| `ForEach-Object { … -split ',' }` | Delar träffraden vid kommatecknen och hämtar den andra kolumnen, `connector-id` |
| `Group-Object` | Räknar anslutningsuppbyggnader per connector |
| `Sort-Object Count -Descending` | Mest använda connector överst |

</details>

```text
Count Name
----- ----
51479 SRV-MAIL01\Default Frontend SRV-MAIL01
50756 SRV-MAIL01\smtp-auth SRV-MAIL01
19405 SRV-MAIL01\smtp-intern SRV-MAIL01
15789 SRV-MAIL01\smtp-noauth SRV-MAIL01
```

Denna fördelning besvarar en fråga som inte går att besvara i meddelandespårningen: Vilka vägar använder dina applikationer faktiskt? Före en connectorändring är det den viktigaste siffran av alla.

### Om inget loggades

Om varje rad saknas vid den aktuella tidpunkten finns det tre vanliga orsaker: Loggning var avstängd på berörd connector, lagringsgränsen har redan trängt undan filen eller du tittar i fel sökväg, alltså i katalogen `Hub` i stället för `FrontEnd`. Kontrollera i denna ordning.

## Steg 4: Kontrollera behörigheter

Om en inskickning avvisas eller omvänt mer tillåts än väntat går vägen via connectorns behörigheter. Här finns en teknisk egenhet: `Get-ADPermission` behöver **DistinguishedName**. Om du skickar den vanliga identiteten i formatet `Server\Connectorname` misslyckas anropet i en fjärrsession med det missvisande meddelandet att objektet inte kan hittas.

```powershell
$dn = (Get-ReceiveConnector "SRV-MAIL01\Default Frontend SRV-MAIL01").DistinguishedName
Get-ADPermission -Identity $dn -User "NT AUTHORITY\ANONYMOUS LOGON" |
    Where-Object { $_.ExtendedRights -like "*ms-Exch-SMTP*" } |
    Format-Table User, @{n='Rights'; e={$_.ExtendedRights}} -AutoSize
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-Identity $dn` | Objektet som ska kontrolleras som DistinguishedName; formatet `Server\Connectorname` misslyckas i fjärrsessioner |
| `-User` | Begränsar utdata till behörigheterna för denna säkerhetsprincipal, här anonym åtkomst |
| `Where-Object` | Filtrerar efter SMTP-relevanta Extended Rights |

</details>

```text
User                         Rights
----                         ------
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Submit
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Accept-Any-Sender
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Accept-Authoritative-Domain-Sender
```

Utvärderingen är enklare än den ser ut om du skiljer mellan fyra rättigheter:

| Rättighet | Betydelse |
|---|---|
| `ms-Exch-SMTP-Submit` | får över huvud taget skicka in |
| `ms-Exch-SMTP-Accept-Any-Sender` | får använda godtyckliga avsändaradresser |
| `ms-Exch-SMTP-Accept-Authoritative-Domain-Sender` | får uppträda som egen domän |
| `ms-Exch-SMTP-Accept-Any-Recipient` | **får vidarebefordra till externa domäner** |

De första tre är standarduppsättningen som krävs för anonym inskickning och mottagning av e-post från Internet. Det är först den fjärde rättigheten som gör en inkommande connector till en relay. På en connector som accepterar hela adressutrymmet är det en öppen relay. På en connector med snäv IP-begränsning är det däremot den vanliga och avsedda vägen för att applikationsservrar ska kunna skicka externt.

Förväxla inte `Accept-Any-Sender` med `Accept-Any-Recipient`. Den första är ofarlig och nödvändig, den andra är den säkerhetsrelevanta inställningen.

## Steg 5: Kontrolltest med egen inskickning

Om utvärderingen fortfarande är tvetydig, skicka in ett meddelande själv. Då kontrollerar du avsändare, mottagare och inskickningspunkt helt:

```powershell
Send-MailMessage -SmtpServer 10.0.1.11 -Port 25 `
    -From 'test@example.com' `
    -To 'empfaenger@example.net' `
    -Subject 'Diagnose, bitte ignorieren' `
    -Body 'Testeinlieferung' `
    -Encoding UTF8
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-SmtpServer` | Målhost för inskickningen, här medvetet som IP-adress för att träffa en viss slutpunkt |
| `-Port 25` | Målport; 25 för oautentiserad server-till-server-inskickning |
| `-From` | Envelope- och headeravsändare för testmeddelandet |
| `-To` | Mottagaradress |
| `-Subject` | Ämnesrad |
| `-Body` | Meddelandetext |
| `-Encoding UTF8` | Teckenkodning för ämne och text, undviker problem med specialtecken |

</details>

`Send-MailMessage` är officiellt utfasat men är fortfarande det snabbaste verktyget för diagnosändamål och finns på varje Windows-server. Vid lyckat resultat visas ingen utdata, vilket kan kännas ovant.

Om du testar en TLS-sträcka på port 587 och motparten presenterar ett certifikat som inte matchar det använda namnet, exempelvis för att du använder IP-adressen, avbryts anropet med ett certifikatfel. För testet kan du inaktivera kontrollen i sessionen:

```powershell
[Net.ServicePointManager]::ServerCertificateValidationCallback = { $true }
```

Detta gäller endast den aktuella PowerShell-sessionen. Gör det medvetet och aldrig i skript som körs i produktion.

Om testmeddelandet kommer fram och du vill veta vad som hände med det på vägen hjälper [Mail Header Analyzer](https://rafaelpfister.ch/tools/header-analyzer): Den delar upp rubrikerna, ritar vägen över hoppen och visar resultaten av autentiseringskontrollerna, helt lokalt i webbläsaren utan att meddelandet lämnar din enhet.

## Exchange Online: samma fråga, ett annat verktyg

Andra regler gäller i klientorganisationen, och det är här invanda arbetssätt misslyckas. Räkna med följande skillnader:

| | Exchange OnPrem | Exchange Online |
|---|---|---|
| Fråga | `Get-MessageTrackingLog` | `Get-MessageTraceV2` |
| Detaljnivå | varje transporthändelse | en rad per meddelande och mottagare |
| Connector synlig | ja (med begränsning, se ovan) | **nej** |
| Serverberoende | ja, fråga per server | bortfaller |
| SMTP-logg | tillgänglig | **inte tillgänglig** |
| Lagring | din konfiguration | 90 dagar via cmdleten, högst 10 dagar per fråga |
| Fördröjning | nästan omedelbar | några minuter |

De tre viktigaste praktiska konsekvenserna: Det finns **ingen connectorkoppling**, du får använda `FromIP` och `ToIP`. Det finns **ingen SMTP-logg**, SMTP-konversationen går inte att rekonstruera. Och data visas **med fördröjning**, ett just skickat meddelande visas inte omedelbart.

### Grundfrågan

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-4) `
    -EndDate (Get-Date) `
    -RecipientAddress "empfaenger@example.com" `
    -ResultSize 1000 |
  Sort-Object Received |
  Format-Table Received, SenderAddress, RecipientAddress, Status, FromIP, Size -AutoSize
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-StartDate` | Nedre tidsgräns för frågan, här de senaste fyra timmarna |
| `-EndDate` | Övre tidsgräns; cmdleten kräver båda gränserna |
| `-RecipientAddress` | Filtrerar efter meddelanden till denna mottagaradress |
| `-ResultSize 1000` | Maximalt antal rader på denna sida; övre gränsen är 5000 |

</details>

```text
Received            SenderAddress          RecipientAddress          Status    FromIP
--------            -------------          ----------------          ------    ------
11.08.2026 08:27:16 emma@partner.example   empfaenger@example.com    Delivered 10.0.20.23
11.08.2026 09:05:24 dienst@example-test.com empfaenger@example.com   Failed    10.0.20.23
```

De viktigaste värdena för `Status`: `Delivered`, `Failed`, `Pending`, `Quarantined`, `FilteredAsSpam` och `Expanded` för expanderade distributionsgrupper. `Pending` betyder att leveransförsök fortfarande pågår, inte att något är trasigt.

### Detaljerna för ett meddelande

Statusen ensam säger ingenting om orsaken. För det behöver du detaljvyn, och den kräver meddelandeidentifieraren från grundfrågan:

```powershell
$n = Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-4) -EndDate (Get-Date) `
        -RecipientAddress "empfaenger@example.com" -ResultSize 100 |
     Where-Object { $_.Status -eq 'Failed' } | Select-Object -First 1

Get-MessageTraceDetailV2 -MessageTraceId $n.MessageTraceId `
    -RecipientAddress $n.RecipientAddress |
  Format-List Date, Event, Action, Detail
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-MessageTraceId` | Unik identifierare för meddelandet från grundfrågan; obligatorisk |
| `-RecipientAddress` | Mottagaren vars behandlingssteg visas; också obligatorisk eftersom ett meddelande kan ha flera mottagare |

</details>

Där hittar du tjänstens behandlingssteg, exempelvis regelanvändning, filterbeslut och orsaken till ett avvisande.

### Längre tillbaka än tio dagar

`Get-MessageTraceV2` levererar data för de senaste 90 dagarna, men högst 10 dagar per fråga; dela upp längre perioder i flera frågor. Alternativt finns den historiska sökningen, som körs asynkront och tillhandahåller resultatet som CSV, med ett intervall på upp till 90 dagar:

```powershell
Start-HistoricalSearch -ReportTitle "Analyse Nachtlauf" `
    -StartDate (Get-Date).AddDays(-45) `
    -EndDate (Get-Date).AddDays(-30) `
    -ReportType MessageTrace `
    -SenderAddress "dienst@example-test.com" `
    -NotifyAddress "admin@example.com"

Get-HistoricalSearch | Format-Table JobId, ReportTitle, Status, SubmitDate -AutoSize
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-ReportTitle` | Fritt valbart namn på jobbet, under vilket resultatet senare kan hittas |
| `-StartDate`, `-EndDate` | Undersökt tidsperiod, upp till 90 dagar tillbaka |
| `-ReportType MessageTrace` | Typ av rapport; `MessageTrace` levererar meddelandeöversikten som CSV |
| `-SenderAddress` | Filtrerar efter denna avsändaradress |
| `-NotifyAddress` | Mottagare av färdigmeddelandet; måste vara en adress i en Accepted Domain i klientorganisationen |

</details>

Avsätt tid; sådana jobb körs i timmar beroende på omfattning.

### Felkälla 4: Saknade träffar är inget bevis för att trafik saknas

Detta är den mest subtila felkällan i klientorganisationen. `Get-MessageTraceV2` levererar sida för sida, högst 5000 rader per anrop. Vid hög belastning kan en sida bara täcka några minuter trots att du har frågat efter sju dagar. Om du sedan filtrerar lokalt, exempelvis efter en käll-IP, filtrerar du över ett mycket litet utsnitt.

Det syns på varningen som hänvisar till ytterligare resultat:

```text
WARNING: There are more results, use the following command to get more.
Get-MessageTraceV2 -StartDate "2026-08-11T07:25:19Z" -EndDate "2026-08-11T09:05:46Z"
  -StartingRecipientAddress "naechster@example.com" -ResultSize 5000
```

Om den visas är din utvärdering **ofullständig**. Om ingen träff returneras är det korrekta resultatet: inte hittad i utsnittet. Det är inte: existerar inte.

Det finns två rena utvägar. Antingen minskar du tidsfönstret tills en sida täcker det helt, vilket märks genom att varningen uteblir. Eller så följer du fortsättningsuppgifterna från varningen genom alla sidor. För frågan om något **aldrig** förekommer är en konfigurationskontroll ändå överlägsen: Om ett system inte har någon rutt till ett mål kan det inte leverera dit, oavsett observationsfönster.

Den fullständiga utvärderingen av alla inskickande adresser är ett ämne i sig, med egna känsliga tolkningspunkter. Den behandlas i [Vem skickar egentligen in till din klientorganisation? Aggregera inskickande IP-adresser](https://rafaelpfister.ch/blog/einliefernde-ip-adressen-aggregieren).

## En arbetsgång som har visat sig fungera

Sammanfattningsvis har denna ordning visat sig vara den snabbaste. Sök efter meddelandet på alla servrar och fastställ den senaste händelsen. Vid ett misslyckande byter du genast till `Format-List` och läser det fullständiga SMTP-svaret, i stället för att dra slutsatser från händelsetypen. Klargör sedan omfattningen, alltså gruppera och räkna. Först när fallet är snävt avgränsat rekonstruerar du inskickningsvägen via connector-konfiguration och SMTP-logg. Och kontrollera slutligen vid behov med en egen inskickning.

De vanligaste tidstjuvarna är däremot alltid desamma: Man läser en avklippt tabell i stället för det fullständiga felmeddelandet, man misstar skuggkopior för behandlingssteg, man tror på `ConnectorId` i spårningen och man håller ett tomt stickprov för ett bevis. Den som känner till dessa fyra kommer vanligtvis till rätt nivå på några minuter.

## Ändringshistorik

- **25 oktober 2026:** Korrigerat: Get-MessageTraceV2 levererar data för de senaste 90 dagarna, men högst 10 dagar per fråga. Källänkar uppdaterade till de nya adresserna på Microsoft Learn.

## Källor

1.  [Message tracking in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): Fältbeskrivning och fullständig lista över händelsetyper i meddelandespårning.

2.  [Protocol logging in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/protocol-logging): Lagringsplatser, format och bevarande av SMTP-loggar, inklusive Front End Transport.

3.  [Shadow redundancy in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/shadow-redundancy): förklarar händelserna kring skuggkopior och hur de kasseras.

4.  [Mail routing in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing): Samspelet mellan Front End Transport och Transport Service, grunden för proxybeteendet.

5.  [Receive connectors in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/receive-connectors): Bindningar, behörighetsgrupper och autentiseringsmekanismer.

6.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2): Efterträdaren till Get-MessageTrace inklusive sidlogik, fältlista och frågeintervall (90 dagar, högst 10 dagar per fråga).

7.  [Start-HistoricalSearch](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/start-historicalsearch): Asynkron meddelandespårning över upp till 90 dagar.
