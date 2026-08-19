---
title: "Analysera Exchange-mailflöde: meddelandespårning, SMTP-protokoll och Receive Connectors"
navTitle: "Analysera mailflöde"
description: "Så tar du systematiskt reda på var ett meddelande har tagit vägen i Exchange OnPrem, Hybrid och Exchange Online: frågor med exempelutdata, hur du läser SMTP-protokollet korrekt och fallgroparna som regelbundet leder in på fel spår."
date: "2026-08-11"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "22 min läsning"
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
url: https://rafaelpfister.ch/sv/blog/analysera-exchange-mailflode-meddelandesparning-smtp-protokoll-och-receive-connectors
translationSourceHash: 646cb713e4dd97300a2cd068ee8f04953f2e80a99aec63ed11eddc46e1981f13
translationModel: gpt-5.6-terra
translatedAt: 2026-08-12T05:17:58.597Z
translationReview: required
---

# Analysera Exchange-mailflöde: meddelandespårning, SMTP-protokoll och Receive Connectors

Den vanligaste frågan i maildriften är: Ett meddelande har inte kommit fram, var har det tagit vägen? Meddelandespårning ger ett tillförlitligt svar, men bara om du vet vad den **inte** berättar. Den här artikeln beskriver arbetssättet i den ordning som har visat sig fungera bäst, visar typisk utdata för varje fråga och pekar ut fallgroparna som regelbundet kostar timmar eftersom de inbjuder till rimliga men felaktiga slutsatser.

Alla exempel använder generiska namn: `SRV-MAIL01` och `SRV-MAIL02` som transportservrar, `example.com` som domän. Om du vill sätta ihop kommandona för din miljö i stället för att skriva dem: [kommandogeneratorn](https://rafaelpfister.ch/tools/command-builder) innehåller vanliga kommandon för meddelandespårning och inspelning för PowerShell och Unix-skal sida vid sida, helt lokalt i webbläsaren.

## Principen: lokalisera först, förklara sedan

Reflexen är att genast leta efter orsaken. Det är effektivare att först fastställa hur långt meddelandet faktiskt har kommit. Det begränsar sökområdet drastiskt i ett enda steg, eftersom du sedan vet om du ska leta i det egna systemet, hos den föregående gatewayen eller hos mottagaren.

Ordningen är därför: hitta meddelandet, läs den senaste händelsen, läs felorsaken, avgör om det är ett enskilt fall eller ett mönster och rekonstruera först därefter inleveransvägen.

## Steg 1: Hitta meddelandet

Börja med mottagaren, eftersom den nästan alltid är känd. Det är viktigt att köra frågan mot **alla** transportservrar, inte bara en.

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

Om frågan inte hittar något, kontrollera om mottagaren har expanderats via en distributionsgrupp. Sök då hellre på avsändaren:

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

## Steg 2: Läs den senaste händelsen

Hela diagnostiken hänger på meddelandets **senaste** `EventId`. Den berättar vilket sökområde som ska undersökas härnäst.

| Senaste EventId | Betydelse | Nästa steg |
|---|---|---|
| `RECEIVE`, därefter inget | Meddelandet har fastnat | Kontrollera köer |
| `SEND` eller `SENDEXTERNAL` | Överlämnat utan problem | Fortsätt söka vid nästa hopp |
| `FAIL` | Slutgiltigt misslyckat | Läs orsaken i `RecipientStatus` |
| `DEFER` | Nytt försök pågår | Kontrollera kö och målsystem |
| `DROP` eller `POISONMESSAGE` | Förkastat | Kontrollera transportregel eller agent |
| `DELIVER` | Levererat till en lokal postlåda | Kontrollera postlåde-regler |
| `RESOLVE` | Mottagaren har skrivits om | Läs måladdressen i posten |

`RESOLVE` är det mest upplysande mellansteget i hybridmiljöer, eftersom omskrivningen till molnroutningsadressen syns där:

```text
EventId : RESOLVE
Source  : ROUTING
Sender  : absender@example.com
To      : BENUTZER@example.mail.onmicrosoft.com
```

Om den förväntade `onmicrosoft.com`-adressen står där är mottagarobjektet korrekt konfigurerat och du kan avskriva frågan. Om den ursprungliga adressen fortfarande står där saknar det lokala objektet måladdressen och Exchange försöker leverera lokalt.

Om meddelandet har fastnat visar kön oftast orsaken i klartext:

```powershell
Get-Queue -Server SRV-MAIL01 |
    Where-Object { $_.MessageCount -gt 0 } |
    Format-Table Identity, DeliveryType, Status, MessageCount, NextHopDomain, LastError `
        -AutoSize -Wrap
```

## Fallgrop 1: Spårning är serverbaserad och många poster är skuggkopior

Om du ser par av `HARECEIVE` och `HADISCARD` i utdata, ofta med tillägget `ExplicitlyDiscarded`, har den servern **inte bearbetat** meddelandet. Den höll bara en skuggkopia inom ramen för Shadow Redundancy, medan en annan server utförde den faktiska leveransen. Så snart den primära servern rapporterar framgång förkastar partnern sin kopia.

Så här ser det ut om du bara har frågat fel server:

```text
Timestamp           EventId    SourceContext
---------           -------    -------------
11.08.2026 09:47:13 HARECEIVE  1a2aae6b-f3a3-4e04-9233-7de460b92223
11.08.2026 09:48:07 HADISCARD  ExplicitlyDiscarded;1a2aae6b-f3a3-4e04-9233-7de460b92223
```

Två rader, inget fel, ingen leverans. Den som drar slutsatsen att meddelandet har försvunnit letar på fel ställe. Den faktiska bearbetningen finns i partnerserverns spårning.

I praktiken innebär detta två saker. För det första är sådana rader inte ett tecken på problem utan normal drift. För det andra måste du ovillkorligen fråga alla transportservrar.

## Fallgrop 2: `Format-Table` klipper bort just de avgörande kolumnerna

Felorsaken står i `RecipientStatus`, och det fältet är långt. I en tabell utelämnas det helt eller klipps av. Det leder just till att man ser `FAIL` men inte orsaken och i stället börjar gissa.

Så snart du har hittat ett fel ska du därför växla till `Format-List` och expandera samlingsfälten:

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

Så här ser skillnaden ut. Först tabellvyn, som inte förklarar något:

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

Diagnosen är därmed klar utan att du har behövt någon enda gissning: Motparten invänder mot avsändaren. `LED` innehåller det fullständiga SMTP-svaret, `FQDN` och `IP` anger systemet som svarade, och `LRT` tidpunkten för det senaste försöket.

## Steg 3: Enskilt fall eller mönster?

Innan du fördjupar dig i ett enskilt fall ska du klargöra omfattningen. Den här enda frågan avgör om du har att göra med en randanmärkning eller en incident:

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

Ersätt `5.1.8` med den statuskod du undersöker. Utdata besvarar frågan på en rad:

```text
Count Name
----- ----
    9 example-test.com
```

En enda avsändardomän innebär: ett snävt avgränsat problem, ingen incident, och du kan fortsätta leta i lugn och ro. Om det stod tjugo olika domäner där skulle du ha ett pågående avbrott, och allt annat skulle få vänta. Att göra denna åtskillnad så tidigt sparar enligt erfarenhet mest tid.

## Fallgrop 3: `ConnectorId` avslöjar inte den verkliga Receive Connectorn

Detta är den dyraste fallgropen eftersom utdata ser seriös ut. Mail som en klient eller ett tredjepartssystem levererar på port 25 träffar först **Front End Transport**. Den vidarebefordrar meddelandet till **Transport Service** på port 2525. Meddelandespårningen skrivs först där; Front End Transport skriver ingen egen spårning.

Konsekvensen syns på den här raden:

```text
EventId        : RECEIVE
ConnectorId    : SRV-MAIL01\Default SRV-MAIL01
ClientIp       : 10.0.1.11
ClientHostname : srv-mail01.intern.example.com
```

`ConnectorId` anger den interna connectorn på port 2525, och `ClientIp` är adressen till **den proxande servern**, inte den ursprungliga inlevereraren. Vilken av de konfigurerade connectorerna på port 25 ett system faktiskt träffade framgår helt enkelt inte av spårningen. Den som tror på uppgiften letar efter felet i en connector som inte ens var inblandad.

Det finns två vägar till svaret. Den första är rekonstruktion via konfigurationen:

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object { Get-ReceiveConnector -Server $_ } |
    Format-List Identity, Enabled,
        @{n='Bindings';       e={$_.Bindings -join ','}},
        @{n='RemoteIPRanges'; e={$_.RemoteIPRanges -join ','}},
        PermissionGroups, AuthMechanism
```

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

Fastställ käll-IP-adressen för det inlevererande systemet och sök efter connectorn vars `RemoteIPRanges` innehåller den. Om den inte faller inom någon av de begränsade connectorerna återstår standard-frontend-connectorn, som vanligtvis accepterar hela adressutrymmet. Använd även här `Format-List`, eftersom `RemoteIPRanges` och `PermissionGroups` regelbundet klipps av i tabeller.

Den andra vägen är SMTP-protokollet, som förtjänar ett eget avsnitt.

## SMTP-protokollet: den enda platsen med hela sanningen

Protokollet för Front End Transport registrerar hela SMTP-sessionen: vilken connector som kontaktades, vilken IP som anslöt och vad klient och server sade till varandra. Det är den enda källan som löser fallgropen ovan.

### Aktivera loggning

Som standard är loggning **avstängd** på de flesta connectorer. Du aktiverar den per connector:

```powershell
Set-ReceiveConnector -Identity "SRV-MAIL01\Default Frontend SRV-MAIL01" `
    -ProtocolLoggingLevel Verbose
```

För utgående anslutningar gör du motsvarande med `Set-SendConnector`. Kom ihåg att återställa värdet till `None` efter analysen, eftersom detaljerad loggning tar diskutrymme och skriver betydande datamängder vid hög belastning.

### Var filerna finns

Exchange separerar protokollen efter tjänst och riktning. Det är onödigt att hårdkoda sökvägarna; fråga efter dem:

```powershell
Get-FrontendTransportService SRV-MAIL01 |
    Format-List ReceiveProtocolLogPath, SendProtocolLogPath,
        ReceiveProtocolLogMaxAge, ReceiveProtocolLogMaxDirectorySize

Get-TransportService SRV-MAIL01 |
    Format-List ReceiveProtocolLogPath, SendProtocolLogPath
```

Vanligtvis ligger de under installationssökvägen i `TransportRoles\Logs\FrontEnd\ProtocolLog\SmtpReceive` för Front End Transport och i `TransportRoles\Logs\Hub\ProtocolLog\SmtpReceive` för Transport Service. **Detta är kärnan:** Klientanslutningar på port 25 finns uteslutande i sökvägen `FrontEnd`, medan sökvägen `Hub` endast innehåller den interna vidarebefordringstrafiken på 2525.

Observera lagringstiden. `ReceiveProtocolLogMaxAge` är ofta inställd på 30 dagar, medan `ReceiveProtocolLogMaxDirectorySize` dessutom begränsar diskutrymmet. Vid hög belastning träder storleksbegränsningen i kraft långt före åldersgränsen, och då är dina protokoll bara några dagar gamla.

### Förstå formatet

Filerna är CSV-filer med rubrikrader som börjar med `#`. De viktigaste kolumnerna är `date-time`, `connector-id`, `session-id`, `sequence-number`, `local-endpoint`, `remote-endpoint`, `event` och `data`.

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

För enskilda fall är ett textfilter betydligt snabbare än objektutvärdering. Sök efter avsändaradressen eller `EHLO`-namnet och hämta sessionsidentifieraren:

```powershell
$pfad = (Get-FrontendTransportService SRV-MAIL01).ReceiveProtocolLogPath
Select-String -Path "$pfad\*.log" -Pattern "dienst@example-test.com" -SimpleMatch |
    Select-Object -First 5 Line
```

Med den hittade `session-id` hämtar du hela sessionen:

```powershell
Select-String -Path "$pfad\*.log" -Pattern "08DEF44EC454A414" -SimpleMatch |
    ForEach-Object { $_.Line } | Select-Object -First 40
```

Om du bara vill veta vilka connectorer som överhuvudtaget ser trafik räknar du anslutningsetableringarna. Det är storleksordningar snabbare för stora filer än att parsa varje rad:

```powershell
Select-String -Path "$pfad\*.log" -Pattern ',\+,' |
    ForEach-Object { ($_.Line -split ',')[1] } |
    Group-Object | Sort-Object Count -Descending |
    Format-Table Count, Name -AutoSize
```

```text
Count Name
----- ----
51479 SRV-MAIL01\Default Frontend SRV-MAIL01
50756 SRV-MAIL01\smtp-auth SRV-MAIL01
19405 SRV-MAIL01\smtp-intern SRV-MAIL01
15789 SRV-MAIL01\smtp-noauth SRV-MAIL01
```

Denna fördelning besvarar en fråga som inte går att besvara med meddelandespårning: Vilka vägar använder dina applikationer i praktiken? Före en connector-ändring är detta den viktigaste siffran av alla.

### Om inget loggades

Om det saknas varje rad vid den aktuella tidpunkten finns tre vanliga orsaker: Loggningen var avstängd på berörd connector, lagringsgränsen har redan trängt undan filen eller så tittar du i fel sökväg, alltså i katalogen `Hub` i stället för `FrontEnd`. Kontrollera i denna ordning.

## Steg 4: Kontrollera behörigheter

Om en inleverans avvisas eller omvänt mer är tillåtet än väntat går vägen via connectorns behörigheter. Här finns en teknisk fallgrop: `Get-ADPermission` kräver **DistinguishedName**. Om du skickar den vanliga identiteten i formatet `Server\Connectorname` misslyckas anropet i en fjärrsession med det missvisande meddelandet att objektet inte kan hittas.

```powershell
$dn = (Get-ReceiveConnector "SRV-MAIL01\Default Frontend SRV-MAIL01").DistinguishedName
Get-ADPermission -Identity $dn -User "NT AUTHORITY\ANONYMOUS LOGON" |
    Where-Object { $_.ExtendedRights -like "*ms-Exch-SMTP*" } |
    Format-Table User, @{n='Rights'; e={$_.ExtendedRights}} -AutoSize
```

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
| `ms-Exch-SMTP-Submit` | Får överhuvudtaget leverera |
| `ms-Exch-SMTP-Accept-Any-Sender` | Får använda valfria avsändaradresser |
| `ms-Exch-SMTP-Accept-Authoritative-Domain-Sender` | Får uppträda som egen domän |
| `ms-Exch-SMTP-Accept-Any-Recipient` | **Får vidarebefordra till externa domäner** |

De första tre är standarduppsättningen som krävs för anonym inleverans och mottagning av internetmail. Det är först den fjärde rättigheten som gör en inkommande connector till ett relä. På en connector som accepterar hela adressutrymmet är det ett öppet relä. På en connector med strikt IP-begränsning är det däremot den vanliga och avsedda vägen för att applikationsservrar ska kunna skicka externt.

Förväxla inte `Accept-Any-Sender` med `Accept-Any-Recipient`. Den första är ofarlig och nödvändig, den andra är den säkerhetsrelevanta inställningen.

## Steg 5: Kontroll genom egen inleverans

Om utvärderingen förblir tvetydig levererar du själv. Då kontrollerar du avsändare, mottagare och inleveranspunkt helt:

```powershell
Send-MailMessage -SmtpServer 10.0.1.11 -Port 25 `
    -From 'test@example.com' `
    -To 'empfaenger@example.net' `
    -Subject 'Diagnose, bitte ignorieren' `
    -Body 'Testeinlieferung' `
    -Encoding UTF8
```

`Send-MailMessage` är officiellt utfasat, men är fortfarande det snabbaste verktyget för diagnos och finns på alla Windows-servrar. Vid lyckat resultat visas ingen utdata, vilket kan kännas ovant.

Om du testar en TLS-sträcka på port 587 och motparten presenterar ett certifikat som inte stämmer överens med namnet som används, exempelvis eftersom du adresserar IP-adressen, avbryts anropet med ett certifikatfel. För testet kan du inaktivera kontrollen i sessionen:

```powershell
[Net.ServicePointManager]::ServerCertificateValidationCallback = { $true }
```

Detta gäller endast den aktuella PowerShell-sessionen. Gör det medvetet och aldrig i skript som körs i drift.

Om testmeddelandet kommer fram och du vill veta vad som hände med det på vägen hjälper [Mail Header Analyzer](https://rafaelpfister.ch/tools/header-analyzer): den delar upp rubrikerna, ritar vägen över hoppen och visar resultaten av autentiseringskontrollerna, helt lokalt i webbläsaren utan att meddelandet lämnar din enhet.

## Exchange Online: samma fråga, ett annat verktyg

I tenant gäller andra regler, och det är här invanda arbetssätt misslyckas. Räkna med dessa skillnader:

| | Exchange OnPrem | Exchange Online |
|---|---|---|
| Fråga | `Get-MessageTrackingLog` | `Get-MessageTraceV2` |
| Detaljnivå | Varje transporthändelse | En rad per meddelande och mottagare |
| Connector synlig | Ja (med begränsning, se ovan) | **Nej** |
| Serverberoende | Ja, fråga per server | Faller bort |
| SMTP-protokoll | Finns | **Inte tillgängligt** |
| Lagring | Din konfiguration | Cirka 10 dagar via cmdleten |
| Fördröjning | Nästan omedelbart | Några minuter |

De tre viktigaste praktiska konsekvenserna: Det finns **ingen connector-tilldelning**, så du får använda `FromIP` och `ToIP`. Det finns **inget SMTP-protokoll**, och SMTP-konversationen går inte att rekonstruera. Och data visas **fördröjt**; ett nyss skickat meddelande syns inte genast.

### Grundfrågan

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-4) `
    -EndDate (Get-Date) `
    -RecipientAddress "empfaenger@example.com" `
    -ResultSize 1000 |
  Sort-Object Received |
  Format-Table Received, SenderAddress, RecipientAddress, Status, FromIP, Size -AutoSize
```

```text
Received            SenderAddress          RecipientAddress          Status    FromIP
--------            -------------          ----------------          ------    ------
11.08.2026 08:27:16 emma@partner.example   empfaenger@example.com    Delivered 10.0.20.23
11.08.2026 09:05:24 dienst@example-test.com empfaenger@example.com   Failed    10.0.20.23
```

De viktigaste värdena för `Status`: `Delivered`, `Failed`, `Pending`, `Quarantined`, `FilteredAsSpam` och `Expanded` för expanderade distributionsgrupper. `Pending` betyder att leveransförsök fortfarande pågår, inte att något är trasigt.

### Detaljerna för ett meddelande

Enbart statusen säger inget om orsaken. För det behöver du detaljvyn, och den kräver meddelandeidentifieraren från grundfrågan:

```powershell
$n = Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-4) -EndDate (Get-Date) `
        -RecipientAddress "empfaenger@example.com" -ResultSize 100 |
     Where-Object { $_.Status -eq 'Failed' } | Select-Object -First 1

Get-MessageTraceDetailV2 -MessageTraceId $n.MessageTraceId `
    -RecipientAddress $n.RecipientAddress |
  Format-List Date, Event, Action, Detail
```

Där anges bearbetningsstegen i tjänsten, till exempel regelanvändningar, filterbeslut och orsaken till ett avvisande.

### Längre tillbaka än tio dagar

Cmdleten når cirka tio dagar tillbaka. För äldre perioder finns den historiska sökningen, som körs asynkront och tillhandahåller resultatet som CSV, med ett intervall på upp till 90 dagar:

```powershell
Start-HistoricalSearch -ReportTitle "Analyse Nachtlauf" `
    -StartDate (Get-Date).AddDays(-45) `
    -EndDate (Get-Date).AddDays(-30) `
    -ReportType MessageTrace `
    -SenderAddress "dienst@example-test.com" `
    -NotifyAddress "admin@example.com"

Get-HistoricalSearch | Format-Table JobId, ReportTitle, Status, SubmitDate -AutoSize
```

Planera in tid; sådana uppdrag tar timmar beroende på omfattningen.

### Fallgrop 4: Avsaknad av träffar är inte bevis på avsaknad av trafik

Detta är den mest subtila fallgropen i tenant. `Get-MessageTraceV2` returnerar resultat sidvis, högst 5 000 rader per anrop. Vid hög belastning kan en sida bara täcka några minuter trots att du har frågat efter sju dagar. Om du sedan filtrerar lokalt, exempelvis på en käll-IP, filtrerar du över ett mycket litet utdrag.

Det märks på varningen som anger att fler resultat finns:

```text
WARNING: There are more results, use the following command to get more.
Get-MessageTraceV2 -StartDate "2026-08-11T07:25:19Z" -EndDate "2026-08-11T09:05:46Z"
  -StartingRecipientAddress "naechster@example.com" -ResultSize 5000
```

Om den visas är din utvärdering **ofullständig**. Om ingen träff returneras är det korrekta resultatet: inte hittat i utdraget. Det betyder inte: finns inte.

Det finns två rena utvägar. Antingen minskar du tidsfönstret tills en sida täcker det helt, vilket märks genom att varningen uteblir. Eller så går du igenom alla sidor med fortsättningsuppgifterna i varningen. För frågan om något **aldrig** förekommer är en konfigurationskontroll ändå överlägsen: Om ett system inte har någon väg till ett mål kan det inte leverera dit, oberoende av något observationsfönster.

Den fullständiga utvärderingen av alla inlevererande adresser är ett eget ämne, med egna fallgropar vid tolkningen. Den finns i [Vem levererar egentligen till din tenant? Aggregera inlevererande IP-adresser](https://rafaelpfister.ch/blog/einliefernde-ip-adressen-aggregieren).

## Ett arbetssätt som har visat sig fungera

Sammanfattningsvis har denna ordning visat sig vara den snabbaste. Sök efter meddelandet på alla servrar och fastställ den senaste händelsen. Vid ett misslyckande, växla direkt till `Format-List` och läs det fullständiga SMTP-svaret i stället för att dra slutsatser från händelsetypen. Klargör därefter omfattningen genom att gruppera och räkna. Först när fallet är snävt avgränsat rekonstruerar du inleveransvägen via connector-konfiguration och SMTP-protokoll. Slutligen kontrollerar du vid behov med en egen inleverans.

De vanligaste tidstjuvarna är däremot alltid desamma: Man läser en avklippt tabell i stället för det fullständiga felmeddelandet, man tar skuggkopior för bearbetningssteg, man tror på `ConnectorId` i spårningen och man betraktar ett tomt stickprov som bevis. Den som känner till dessa fyra kommer i regel till rätt nivå på några minuter.

## Källor

1.  [Message tracking in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): Fältbeskrivning och fullständig lista över händelsetyper i meddelandespårning.

2.  [Protocol logging in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/protocol-logging): Lagringsplatser, format och lagringstid för SMTP-protokoll, inklusive Front End Transport.

3.  [Shadow redundancy in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/shadow-redundancy): förklarar händelserna kring skuggkopior och deras förkastande.

4.  [Mail routing in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing): Samspelet mellan Front End Transport och Transport Service, grunden för proxybeteendet.

5.  [Receive connectors in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/receive-connectors): Bindningar, behörighetsgrupper och autentiseringsmekanismer.

6.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2): Efterföljaren till Get-MessageTrace inklusive sidlogik och fältlista.

7.  [Start-HistoricalSearch](https://learn.microsoft.com/en-us/powershell/module/exchange/start-historicalsearch): Asynkron meddelandespårning över upp till 90 dagar.
