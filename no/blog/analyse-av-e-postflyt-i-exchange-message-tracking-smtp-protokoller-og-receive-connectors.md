---
title: "Analyse av e-postflyt i Exchange: Message Tracking, SMTP-protokoller og Receive Connectors"
navTitle: "Analysere e-postflyt"
description: "Slik finner du systematisk ut hvor en melding har blitt av i Exchange OnPrem, Hybrid og Exchange Online: spørringene med eksempelutdata, hvordan du leser SMTP-protokollen riktig, og fallgruvene som jevnlig leder på feil spor."
date: "2026-08-11"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "22 min lesetid"
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
slug: "analyse-av-e-postflyt-i-exchange-message-tracking-smtp-protokoller-og-receive-connectors"
translationId: "article-ad93c41ab6cd20e6"
draft: false
translationOf: exchange-message-tracking-und-receive-connectoren-analysieren
url: https://rafaelpfister.ch/no/blog/analyse-av-e-postflyt-i-exchange-message-tracking-smtp-protokoller-og-receive-connectors
translationSourceHash: 646cb713e4dd97300a2cd068ee8f04953f2e80a99aec63ed11eddc46e1981f13
translationModel: gpt-5.6-terra
translatedAt: 2026-08-12T05:18:47.481Z
translationReview: automatic
---

# Analyse av e-postflyt i Exchange: Message Tracking, SMTP-protokoller og Receive Connectors

Det vanligste spørsmålet i e-postdrift er: En melding har ikke kommet fram, hvor ble den av? Message Tracking gir et pålitelig svar, men bare hvis du vet hva det **ikke** forteller deg. Denne artikkelen beskriver fremgangsmåten i den rekkefølgen som har vist seg best, viser typiske utdata for hver spørring og peker på fallgruvene som jevnlig koster timer fordi de antyder plausible, men feilaktige konklusjoner.

Alle eksemplene bruker generiske navn: `SRV-MAIL01` og `SRV-MAIL02` som transportservere, `example.com` som domene. Hvis du vil sette sammen kommandoene for ditt miljø i stedet for å skrive dem inn: [Kommando-generatoren](https://rafaelpfister.ch/tools/command-builder) inneholder vanlige Message Tracking- og opptakskommandoer for PowerShell og Unix-skall side om side, helt lokalt i nettleseren.

## Grunnprinsippet: lokaliser først, forklar deretter

Refleksen er å lete etter årsaken umiddelbart. Det er mer effektivt å først fastslå hvor langt meldingen faktisk har kommet. Det begrenser søkerommet drastisk i ett steg, fordi du deretter vet om du må lete i ditt eget system, hos den oppstrøms gatewayen eller hos målet.

Rekkefølgen er derfor: Finn meldingen, les siste hendelse, les feilårsaken, avgjør om det er et enkelttilfelle eller et mønster, og rekonstruer først deretter innleveringsveien.

## Trinn 1: Finn meldingen

Start med mottakeren, for den kjenner du nesten alltid. Det er viktig å kjøre spørringen mot **alle** transportservere, ikke bare én.

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

Typiske utdata for en melding som har gått gjennom uten problemer:

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

Hvis spørringen ikke finner noe, kontroller om mottakeren ble utvidet via en distribusjonsliste. Da er det bedre å søke via avsenderen:

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

## Trinn 2: Les siste hendelse

Hele diagnosen avhenger av meldingens **siste** `EventId`. Den forteller deg hvilket søkerom som skal undersøkes videre.

| Siste EventId | Betydning | Neste trinn |
|---|---|---|
| `RECEIVE`, deretter ingenting | Meldingen sitter fast | Kontroller køer |
| `SEND` eller `SENDEXTERNAL` | Vellykket overlevert | Søk videre ved neste hop |
| `FAIL` | Endelig mislykket | Les årsaken i `RecipientStatus` |
| `DEFER` | Nytt forsøk pågår | Kontroller kø og målsystem |
| `DROP` eller `POISONMESSAGE` | Forkastet | Transportregel eller agent |
| `DELIVER` | Levert til en lokal postboks | Kontroller postboksregler |
| `RESOLVE` | Mottakeren ble omskrevet | Les mål-adressen i oppføringen |

`RESOLVE` er det mest opplysende mellomtrinnet i hybridmiljøer, fordi omskrivingen til skyens rutingadresse er synlig der:

```text
EventId : RESOLVE
Source  : ROUTING
Sender  : absender@example.com
To      : BENUTZER@example.mail.onmicrosoft.com
```

Hvis den forventede `onmicrosoft.com`-adressen står der, er mottakerobjektet riktig konfigurert, og du kan avslutte saken. Hvis den opprinnelige adressen fortsatt står der, mangler mål-adressen på det lokale objektet, og Exchange forsøker å levere lokalt.

Hvis meldingen sitter fast, viser køen vanligvis årsaken i klartekst:

```powershell
Get-Queue -Server SRV-MAIL01 |
    Where-Object { $_.MessageCount -gt 0 } |
    Format-Table Identity, DeliveryType, Status, MessageCount, NextHopDomain, LastError `
        -AutoSize -Wrap
```

## Fallgruve 1: Tracking er serverbasert, og mange oppføringer er skyggekopier

Hvis du ser par med `HARECEIVE` og `HADISCARD` i utdataene, ofte med tillegget `ExplicitlyDiscarded`, har denne serveren **ikke behandlet** meldingen. Den holdt bare en skyggekopi som del av Shadow Redundancy, mens en annen server utførte selve leveringen. Så snart den primære serveren melder om suksess, forkaster partneren sin kopi.

Slik ser det ut dersom du bare har spurt feil server:

```text
Timestamp           EventId    SourceContext
---------           -------    -------------
11.08.2026 09:47:13 HARECEIVE  1a2aae6b-f3a3-4e04-9233-7de460b92223
11.08.2026 09:48:07 HADISCARD  ExplicitlyDiscarded;1a2aae6b-f3a3-4e04-9233-7de460b92223
```

To linjer, ingen feil, ingen levering. Den som konkluderer med at meldingen har forsvunnet, leter på feil sted. Den faktiske behandlingen står i sporingen på partnerserveren.

I praksis betyr dette to ting. For det første er slike linjer ikke tegn på et problem, men normal drift. For det andre må du alltid spørre alle transportservere.

## Fallgruve 2: `Format-Table` kutter bort de avgjørende kolonnene

Feilårsaken står i `RecipientStatus`, og dette feltet er langt. I en tabell faller det enten helt bort eller blir avkortet. Nettopp dette fører til at man ser `FAIL`, men ikke årsaken, og begynner å gjette i stedet.

Så snart du har funnet et feiltilfelle, bytter du derfor til `Format-List` og utvider samlingsfeltene:

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

Slik ser forskjellen ut. Først tabellvisningen, som ikke forklarer noe:

```text
Timestamp           EventId ConnectorId
---------           ------- -----------
11.08.2026 09:47:13 FAIL    Outbound-to-O365
```

Deretter samme melding som liste:

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

Diagnosen er dermed fastslått uten at du har trengt én eneste antakelse: Motparten innvender mot avsenderen. `LED` inneholder det fullstendige SMTP-svaret, `FQDN` og `IP` angir systemet som svarte, og `LRT` tidspunktet for siste forsøk.

## Trinn 3: Enkelttilfelle eller mønster?

Før du går i dybden på ett enkelt tilfelle, avklar omfanget. Denne ene spørringen avgjør om du har å gjøre med en mindre detalj eller en hendelse:

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

Erstatt `5.1.8` med statuskoden du undersøker. Utdataene besvarer spørsmålet på én linje:

```text
Count Name
----- ----
    9 example-test.com
```

Én enkelt avsenderdomene betyr: et avgrenset problem, ikke en hendelse, og du kan undersøke videre i ro og mak. Hvis det sto tjue forskjellige domener der, ville du hatt et pågående utfall, og alt annet måtte vente. Erfaringen viser at det sparer mest tid å gjøre dette skillet så tidlig.

## Fallgruve 3: `ConnectorId` avslører ikke den faktiske Receive Connectoren

Dette er den dyreste fallgruven, fordi utdataene ser seriøse ut. E-post som en klient eller et eksternt system leverer på port 25, treffer først **Front End Transport**. Denne videresender meldingen til **Transport Service** på port 2525. Message Tracking skrives først der; Front End Transport skriver ikke egen sporing.

Konsekvensen ser du på denne linjen:

```text
EventId        : RECEIVE
ConnectorId    : SRV-MAIL01\Default SRV-MAIL01
ClientIp       : 10.0.1.11
ClientHostname : srv-mail01.intern.example.com
```

`ConnectorId` angir den interne connectoren på port 2525, og `ClientIp` er adressen til **serveren som proxyer**, ikke den opprinnelige avsenderen. Hvilken av de konfigurerte connectorene på port 25 et system faktisk traff, står rett og slett ikke i sporingen. Den som stoler på denne opplysningen, leter etter feilen i en connector som ikke var involvert.

Det finnes to veier til svaret. Den første er rekonstruksjon via konfigurasjonen:

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

Fastslå kilde-IP-en til systemet som leverer inn, og finn connectoren som inneholder den i `RemoteIPRanges`. Hvis den ikke faller innenfor noen av de begrensede connectorene, gjenstår standard-frontend-connectoren, som vanligvis aksepterer hele adresserommet. Bruk også her `Format-List`, fordi `RemoteIPRanges` og `PermissionGroups` regelmessig avkortes i tabeller.

Den andre veien er SMTP-protokollen, og den fortjener et eget avsnitt.

## SMTP-protokollen: det eneste stedet med hele sannheten

Protokollen fra Front End Transport registrerer hele SMTP-økten: hvilken connector som ble kontaktet, hvilken IP som koblet til, og hva klient og server sa til hverandre. Det er den eneste kilden som løser fallgruven over.

### Slå på logging

Som standard er logging **slått av** på de fleste connectorer. Du slår den på per connector:

```powershell
Set-ReceiveConnector -Identity "SRV-MAIL01\Default Frontend SRV-MAIL01" `
    -ProtocolLoggingLevel Verbose
```

For utgående forbindelser tilsvarende via `Set-SendConnector`. Husk å sette verdien tilbake til `None` etter analysen, fordi detaljert logging bruker diskplass og skriver betydelige datamengder ved høy trafikk.

### Hvor filene ligger

Exchange skiller protokollene etter tjeneste og retning. Det er unødvendig å hardkode stiene; spør etter dem:

```powershell
Get-FrontendTransportService SRV-MAIL01 |
    Format-List ReceiveProtocolLogPath, SendProtocolLogPath,
        ReceiveProtocolLogMaxAge, ReceiveProtocolLogMaxDirectorySize

Get-TransportService SRV-MAIL01 |
    Format-List ReceiveProtocolLogPath, SendProtocolLogPath
```

Vanligvis ligger de under installasjonsstien i `TransportRoles\Logs\FrontEnd\ProtocolLog\SmtpReceive` for Front End Transport og i `TransportRoles\Logs\Hub\ProtocolLog\SmtpReceive` for Transport Service. **Dette er kjernen:** Klientforbindelser på port 25 finner du kun i `FrontEnd`-stien, mens `Hub`-stien bare inneholder den interne videresendingstrafikken på 2525.

Vær oppmerksom på oppbevaringen. `ReceiveProtocolLogMaxAge` er ofte satt til 30 dager, og `ReceiveProtocolLogMaxDirectorySize` begrenser i tillegg plassforbruket. Ved høy trafikk slår størrelsesbegrensningen inn lenge før aldersgrensen, og da er protokollene dine bare noen få dager gamle.

### Forstå formatet

Filene er CSV med overskriftslinjer som begynner med `#`. De viktigste kolonnene er `date-time`, `connector-id`, `session-id`, `sequence-number`, `local-endpoint`, `remote-endpoint`, `event` og `data`.

Avgjørende er kolonnen `event`, et enkelt tegn:

| Tegn | Betydning |
|---|---|
| `+` | Forbindelse opprettet |
| `-` | Forbindelse avsluttet |
| `>` | Server sender til klient |
| `<` | Klient sender til server |
| `*` | Serverinformasjon, ingen SMTP-trafikk |

Du kjenner igjen en økt på felles `session-id`; `sequence-number` angir rekkefølgen innenfor økten. Et typisk utdrag ser slik ut:

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

Her står alt som manglet i Message Tracking: den **faktiske** connectoren (`smtp-noauth`), den **faktiske** kilde-IP-en (`10.0.20.22`) og navnet systemet oppgir i `EHLO`.

### Målrettet søk

For enkelttilfeller er et tekstfilter langt raskere enn objektanalyse. Søk etter avsenderadressen eller `EHLO`-navnet og få oppgitt øktidentifikatoren:

```powershell
$pfad = (Get-FrontendTransportService SRV-MAIL01).ReceiveProtocolLogPath
Select-String -Path "$pfad\*.log" -Pattern "dienst@example-test.com" -SimpleMatch |
    Select-Object -First 5 Line
```

Med den funne `session-id` henter du hele økten:

```powershell
Select-String -Path "$pfad\*.log" -Pattern "08DEF44EC454A414" -SimpleMatch |
    ForEach-Object { $_.Line } | Select-Object -First 40
```

Hvis du bare vil vite hvilke connectorer som i det hele tatt har trafikk, teller du forbindelsesopprettelsene. Dette er størrelsesordener raskere i store filer enn å analysere hver linje:

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

Denne fordelingen besvarer et spørsmål som ikke kan besvares i Message Tracking: Hvilke veier bruker applikasjonene dine faktisk? Før en connector-omlegging er dette det viktigste tallet av alle.

### Når ingenting ble logget

Hvis det mangler enhver linje på det aktuelle tidspunktet, finnes det tre vanlige årsaker: Logging var avslått på den aktuelle connectoren, oppbevaringsgrensen har allerede fjernet filen, eller du ser i feil sti, altså i `Hub`- i stedet for `FrontEnd`-katalogen. Kontroller i denne rekkefølgen.

## Trinn 4: Kontroller tillatelser

Hvis en innlevering blir avvist, eller omvendt mer er tillatt enn antatt, går veien via connectorens tillatelser. Her finnes en teknisk fallgruve: `Get-ADPermission` krever **DistinguishedName**. Hvis du sender inn den vanlige identiteten på formen `Server\Connectorname`, mislykkes kallet i en ekstern økt med den misvisende meldingen om at objektet ikke ble funnet.

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

Tolkningen er enklere enn den ser ut hvis du skiller mellom fire rettigheter:

| Rettighet | Betydning |
|---|---|
| `ms-Exch-SMTP-Submit` | Har i det hele tatt lov til å levere inn |
| `ms-Exch-SMTP-Accept-Any-Sender` | Har lov til å bruke vilkårlige avsenderadresser |
| `ms-Exch-SMTP-Accept-Authoritative-Domain-Sender` | Har lov til å opptre som eget domene |
| `ms-Exch-SMTP-Accept-Any-Recipient` | **Har lov til å videresende til eksterne domener** |

De tre første er standardsettet som trengs for anonym innlevering og mottak av e-post fra Internett. Først den fjerde rettigheten gjør en inngangsconnector til et relé. På en connector som aksepterer hele adresserommet, er det et åpent relé. På en connector med streng IP-begrensning er det derimot den vanlige og tilsiktede måten applikasjonsservere kan sende eksternt på.

Ikke forveksle `Accept-Any-Sender` med `Accept-Any-Recipient`. Den første er ufarlig og nødvendig, den andre er den sikkerhetsrelevante innstillingen.

## Trinn 5: Kontrolltest med egen innlevering

Hvis analysen fortsatt er tvetydig, lever inn en melding selv. Da kontrollerer du avsender, mottaker og innleveringspunkt fullstendig:

```powershell
Send-MailMessage -SmtpServer 10.0.1.11 -Port 25 `
    -From 'test@example.com' `
    -To 'empfaenger@example.net' `
    -Subject 'Diagnose, bitte ignorieren' `
    -Body 'Testeinlieferung' `
    -Encoding UTF8
```

`Send-MailMessage` er offisielt avviklet, men er fortsatt det raskeste verktøyet for diagnoseformål og finnes på alle Windows-servere. Ved suksess kommer det ingen utdata, noe som kan være uvant.

Hvis du tester en TLS-forbindelse på port 587 og motparten presenterer et sertifikat som ikke passer til navnet som brukes, for eksempel fordi du henvender deg til IP-adressen, avbrytes kallet med en sertifikatfeil. For testen kan du deaktivere kontrollen i økten:

```powershell
[Net.ServicePointManager]::ServerCertificateValidationCallback = { $true }
```

Dette gjelder bare den aktive PowerShell-økten. Sett det bevisst, og aldri i skript som kjører i drift.

Hvis testmeldingen kommer fram og du vil vite hva som har skjedd med den underveis, hjelper [Mail Header Analyzer](https://rafaelpfister.ch/tools/header-analyzer): Den bryter ned meldingshodene, tegner veien gjennom hopene og viser resultatene av autentiseringskontrollene, helt lokalt i nettleseren uten at meldingen forlater enheten din.

## Exchange Online: samme spørsmål, et annet verktøy

Andre regler gjelder i leieren, og dette er punktet der vante fremgangsmåter mislykkes. Regn med disse forskjellene:

| | Exchange OnPrem | Exchange Online |
|---|---|---|
| Spørring | `Get-MessageTrackingLog` | `Get-MessageTraceV2` |
| Detaljnivå | Hver transporthendelse | Én linje per melding og mottaker |
| Connector synlig | Ja (med begrensning, se over) | **Nei** |
| Servertilknytning | Ja, spør per server | Utgår |
| SMTP-protokoll | Tilgjengelig | **Ikke tilgjengelig** |
| Oppbevaring | Din konfigurasjon | Rundt 10 dager via cmdleten |
| Forsinkelse | Nesten umiddelbart | Noen minutter |

De tre viktigste praktiske konsekvensene: Det finnes **ingen connector-tilordning**, så du må bruke `FromIP` og `ToIP`. Det finnes **ingen SMTP-protokoll**, så SMTP-samtalen kan ikke rekonstrueres. Og dataene vises **forsinket**, slik at en nettopp sendt melding ikke dukker opp med én gang.

### Grunnspørringen

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

De viktigste verdiene for `Status`: `Delivered`, `Failed`, `Pending`, `Quarantined`, `FilteredAsSpam` og `Expanded` for utvidede distribusjonslister. `Pending` betyr at leveringsforsøk fortsatt pågår, ikke at noe er ødelagt.

### Detaljene for en melding

Statusen alene sier ingenting om årsaken. Til det trenger du detaljvisningen, og den krever meldingsidentifikatoren fra grunnspørringen:

```powershell
$n = Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-4) -EndDate (Get-Date) `
        -RecipientAddress "empfaenger@example.com" -ResultSize 100 |
     Where-Object { $_.Status -eq 'Failed' } | Select-Object -First 1

Get-MessageTraceDetailV2 -MessageTraceId $n.MessageTraceId `
    -RecipientAddress $n.RecipientAddress |
  Format-List Date, Event, Action, Detail
```

Der finner du behandlingstrinnene i tjenesten, som regelanvendelser, filteravgjørelser og årsaken til en avvisning.

### Utover ti dager

Cmdleten går omtrent ti dager tilbake. For eldre perioder finnes historisk søk, som kjører asynkront og leverer resultatet som CSV, med et område på opptil 90 dager:

```powershell
Start-HistoricalSearch -ReportTitle "Analyse Nachtlauf" `
    -StartDate (Get-Date).AddDays(-45) `
    -EndDate (Get-Date).AddDays(-30) `
    -ReportType MessageTrace `
    -SenderAddress "dienst@example-test.com" `
    -NotifyAddress "admin@example.com"

Get-HistoricalSearch | Format-Table JobId, ReportTitle, Status, SubmitDate -AutoSize
```

Sett av tid; slike jobber kan ta timer, avhengig av omfanget.

### Fallgruve 4: Manglende treff er ikke bevis på manglende trafikk

Dette er den mest subtile fallgruven i leieren. `Get-MessageTraceV2` leverer resultater sidevis, maksimalt 5000 linjer per kall. Ved høy trafikk kan én side dekke bare noen få minutter, selv om du har spurt etter syv dager. Hvis du deretter filtrerer lokalt, for eksempel på en kilde-IP, filtrerer du over et svært lite utdrag.

Du kjenner det igjen på advarselen som viser til flere resultater:

```text
WARNING: There are more results, use the following command to get more.
Get-MessageTraceV2 -StartDate "2026-08-11T07:25:19Z" -EndDate "2026-08-11T09:05:46Z"
  -StartingRecipientAddress "naechster@example.com" -ResultSize 5000
```

Hvis den vises, er analysen din **ufullstendig**. Hvis det ikke kommer tilbake noe treff, er korrekt resultat: ikke funnet i utdraget. Det er ikke: finnes ikke.

Det finnes to ryddige utveier. Enten reduserer du tidsvinduet til én side dekker det fullstendig, noe du ser ved at advarselen uteblir. Eller du følger fortsettelsesopplysningene i advarselen gjennom alle sidene. For spørsmålet om noe **aldri** forekommer, er en konfigurasjonskontroll uansett bedre: Hvis et system ikke har noen rute til et mål, kan det ikke levere dit, uavhengig av ethvert observasjonsvindu.

Den fullstendige analysen av alle innleverende adresser er et eget tema, med egne fallgruver i tolkningen. Den finner du i [Hvem leverer egentlig inn til leieren din? Aggregering av innleverende IP-adresser](https://rafaelpfister.ch/blog/einliefernde-ip-adressen-aggregieren).

## En fremgangsmåte som har vist seg effektiv

Oppsummert har denne rekkefølgen vist seg å være den raskeste. Søk etter meldingen på alle servere og fastslå siste hendelse. Ved feil, bytt umiddelbart til `Format-List` og les det fullstendige SMTP-svaret, i stedet for å trekke slutninger fra hendelsestypen. Avklar deretter omfanget, altså grupper og tell. Først når saken er tydelig avgrenset, rekonstruerer du innleveringsveien via connector-konfigurasjon og SMTP-protokoll. Til slutt, ved behov, kontrollerer du med en egen innlevering.

De vanligste tidstyvene er derimot alltid de samme: Du leser en avkortet tabell i stedet for den fullstendige feilmeldingen, du holder skyggekopier for behandlingstrinn, du stoler på `ConnectorId` i sporingen, og du anser en tom prøve for å være bevis. Den som kjenner disse fire, kommer vanligvis til riktig nivå i løpet av få minutter.

## Kilder

1.  [Message tracking in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): Feltbeskrivelse og fullstendig liste over hendelsestyper i Message Tracking.

2.  [Protocol logging in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/protocol-logging): Lagringssteder, format og oppbevaring av SMTP-protokoller, inkludert Front End Transport.

3.  [Shadow redundancy in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/shadow-redundancy): Forklarer hendelsene rundt skyggekopier og forkasting av dem.

4.  [Mail routing in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing): Samspillet mellom Front End Transport og Transport Service, grunnlaget for proxy-atferden.

5.  [Receive connectors in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/receive-connectors): Bindinger, tillatelsesgrupper og autentiseringsmekanismer.

6.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2): Etterfølgeren til Get-MessageTrace, inkludert sidelogikk og feltliste.

7.  [Start-HistoricalSearch](https://learn.microsoft.com/en-us/powershell/module/exchange/start-historicalsearch): Asynkron meldingssporing over opptil 90 dager.
