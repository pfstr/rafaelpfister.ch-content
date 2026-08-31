---
title: "Analyse av Exchange-e-postflyt: Message Tracking, SMTP-protokoller og Receive Connectors"
navTitle: "Analysere e-postflyt"
description: "Slik finner du systematisk ut hvor en melding har blitt av i Exchange OnPrem, Hybrid og Exchange Online: spørringene med eksempelutdata, hvordan du leser SMTP-protokollen riktig, og punktene som regelmessig fører til feilslutninger."
date: "2026-08-11"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "22 min. lesetid"
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
translationSourceHash: da923f7fa45ee5c38ea52e96d56781f7c3806556245a5f071242e7f02473a71c
translationModel: gpt-5.6-terra
translatedAt: 2026-08-31T10:14:53.436Z
translationReview: automatic
url: https://rafaelpfister.ch/no/blog/analyse-av-e-postflyt-i-exchange-message-tracking-smtp-protokoller-og-receive-connectors
---

# Analyse av Exchange-e-postflyt: Message Tracking, SMTP-protokoller og Receive Connectors

Det vanligste spørsmålet i e-postdrift er: En melding har ikke kommet frem – hvor ble den av? Message Tracking gir et pålitelig svar, men bare hvis du vet hva det **ikke** forteller deg. Denne artikkelen beskriver fremgangsmåten i rekkefølgen som har vist seg å fungere, viser typiske utdata for hver spørring og nevner feilkildene som regelmessig koster timer, fordi de peker mot plausible, men feilaktige konklusjoner.

Alle eksemplene bruker generiske navn: `SRV-MAIL01` og `SRV-MAIL02` som transportservere, `example.com` som domene. Hvis du vil sette sammen kommandoene for ditt miljø i stedet for å skrive dem inn: [Kommandogeneratoren](https://rafaelpfister.ch/tools/command-builder) inneholder vanlige kommandoer for Message Tracking og opptak for PowerShell og Unix-skall side om side, helt lokalt i nettleseren.

## Grunnprinsippet: lokaliser først, forklar deretter

Refleksen er å lete etter årsaken med én gang. Det er mer effektivt å først fastslå hvor langt meldingen i det hele tatt har kommet. Det avgrenser søkeområdet drastisk i ett steg, for da vet du om du må lete i ditt eget system, hos en foranstilt gateway eller hos målet.

Rekkefølgen er derfor: Finn meldingen, les siste hendelse, les feilårsaken, avgjør om det er et enkelttilfelle eller et mønster, og rekonstruer først deretter innleveringsveien.

## Trinn 1: Finn meldingen

Begynn med mottakeren, for den kjenner du nesten alltid. Det er viktig å kjøre spørringen mot **alle** transportservere, ikke bare én.

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
<summary>Forklaring av alternativene</summary>

| Alternativ | Virkning |
|---|---|
| `-Server` | Transportserveren som Tracking-loggen hentes fra; her begge serverne etter tur via pipelinen |
| `-Start` | Nedre tidsgrense for søket, her de siste seks timene |
| `-ResultSize Unlimited` | Fjerner standardgrensen på 1000 oppføringer |
| `-Recipients` | Filtrerer på meldinger til denne mottakeradressen |
| `Sort-Object Timestamp` | Sorterer de sammenslåtte resultatene fra begge servere kronologisk |
| `-AutoSize -Wrap` | Tilpasser kolonnebredden til innholdet og bryter lange verdier i stedet for å kutte dem |

</details>

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

Hvis spørringen ikke finner noe, må du kontrollere om mottakeren ble utvidet via en distribusjonsgruppe. Da er det bedre å søke på avsenderen:

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
<summary>Forklaring av alternativene</summary>

| Alternativ | Virkning |
|---|---|
| `-Server` | Transportserveren som Tracking-loggen hentes fra |
| `-Start` | Nedre tidsgrense for søket |
| `-ResultSize Unlimited` | Fjerner standardgrensen på 1000 oppføringer |
| `Where-Object` | Filtrerer på klientsiden etter avsendere i eget domene, siden `-Sender` bare godtar eksakte adresser |
| `@{n=…; e=…}` | Beregnet kolonne: samler samlefeltet `Recipients` til en kommaseparert tekststreng |

</details>

## Trinn 2: Les siste hendelse

Hele diagnosen avhenger av meldingens **siste** `EventId`. Den forteller deg hvilket søkeområde som skal undersøkes videre.

| Siste EventId | Betydning | Neste trinn |
|---|---|---|
| `RECEIVE`, deretter ingenting | Meldingen sitter fast | Kontroller køer |
| `SEND` eller `SENDEXTERNAL` | Levert videre | Fortsett søket ved neste hopp |
| `FAIL` | Endelig mislykket | Les årsaken i `RecipientStatus` |
| `DEFER` | Nytt forsøk pågår | Kontroller kø og målsystem |
| `DROP` eller `POISONMESSAGE` | Forkastet | Kontroller transportregel eller agent |
| `DELIVER` | Levert til en lokal postkasse | Kontroller postkasseregler |
| `RESOLVE` | Mottakeren ble omskrevet | Les måladresse i oppføringen |

`RESOLVE` er det mest informative mellomtrinnet i hybridmiljøer, fordi omskrivingen til skyens rutingadresse blir synlig der:

```text
EventId : RESOLVE
Source  : ROUTING
Sender  : absender@example.com
To      : BENUTZER@example.mail.onmicrosoft.com
```

Hvis forventet `onmicrosoft.com`-adresse står der, er mottakerobjektet konfigurert riktig, og du kan avslutte saken. Hvis den opprinnelige adressen fortsatt står der, mangler mål-adressen på det lokale objektet, og Exchange forsøker å levere lokalt.

Hvis meldingen sitter fast, viser køen som regel årsaken i klartekst:

```powershell
Get-Queue -Server SRV-MAIL01 |
    Where-Object { $_.MessageCount -gt 0 } |
    Format-Table Identity, DeliveryType, Status, MessageCount, NextHopDomain, LastError `
        -AutoSize -Wrap
```

<details class="options-details">
<summary>Forklaring av alternativene</summary>

| Alternativ | Virkning |
|---|---|
| `-Server` | Serveren som transportkøene hentes fra |
| `Where-Object` | Skjuler tomme køer og viser bare køer med ventende meldinger |
| `-AutoSize -Wrap` | Hindrer at den lange kolonnen `LastError` blir avkortet |

</details>

## Feilkilde 1: Tracking er serverbasert, og mange oppføringer er skyggekopier

Hvis du ser par med `HARECEIVE` og `HADISCARD` i utdataene, ofte med tillegget `ExplicitlyDiscarded`, har denne serveren **ikke behandlet** meldingen. Den holdt bare en skyggekopi som del av Shadow Redundancy, mens en annen server tok seg av selve leveringen. Så snart primærserveren melder suksess, forkaster partneren sin kopi.

Slik ser det ut hvis du bare har spurt feil server:

```text
Timestamp           EventId    SourceContext
---------           -------    -------------
11.08.2026 09:47:13 HARECEIVE  1a2aae6b-f3a3-4e04-9233-7de460b92223
11.08.2026 09:48:07 HADISCARD  ExplicitlyDiscarded;1a2aae6b-f3a3-4e04-9233-7de460b92223
```

To linjer, ingen feil, ingen levering. Den som konkluderer med at meldingen har forsvunnet, leter på feil sted. Den faktiske behandlingen står i tracking-loggen til partnerserveren.

I praksis betyr det to ting. For det første er slike linjer ikke tegn på et problem, men normal drift. For det andre må du alltid spørre alle transportservere.

## Feilkilde 2: `Format-Table` kutter akkurat de avgjørende kolonnene

Feilårsaken står i `RecipientStatus`, og dette feltet er langt. I en tabell faller det enten helt bort eller blir avkortet. Nettopp dette fører til at man ser `FAIL`, men ikke årsaken, og i stedet begynner å gjette.

Så snart du har funnet et feiltilfelle, bør du derfor bytte til `Format-List` og løse opp samlefeltene:

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
<summary>Forklaring av alternativene</summary>

| Alternativ | Virkning |
|---|---|
| `-Server` | Transportserveren som Tracking-loggen hentes fra |
| `-Start` | Nedre tidsgrense for søket |
| `-ResultSize Unlimited` | Fjerner standardgrensen på 1000 oppføringer |
| `-Recipients` | Filtrerer på meldinger til denne mottakeradressen |
| `-EventId FAIL` | Bare oppføringer med endelig leveringsfeil |
| `Format-List` | Viser hvert felt på egen linje i full lengde; ingenting blir avkortet |
| `@{n=…; e=…}` | Beregnede felt: løser opp samlefeltene `Recipients` og `RecipientStatus` til lesbare tekststrenger |

</details>

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

Diagnosen er dermed klar uten at du har trengt én eneste antakelse: Motparten avviser avsenderen. `LED` inneholder hele SMTP-svaret, `FQDN` og `IP` angir systemet som svarte, og `LRT` tidspunktet for siste forsøk.

## Trinn 3: Enkelttilfelle eller mønster?

Før du går i dybden på én enkelt sak, må du avklare omfanget. Denne ene spørringen avgjør om du har en bagatell eller en hendelse:

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
<summary>Forklaring av alternativene</summary>

| Alternativ | Virkning |
|---|---|
| `-Start` | Nedre tidsgrense, her de siste åtte timene |
| `-EventId FAIL` | Bare endelig mislykkede leveringer |
| `-ResultSize Unlimited` | Fjerner standardgrensen på 1000 oppføringer |
| `Where-Object` | Filtrerer på undersøkt SMTP-statuskode i feltet `RecipientStatus` |
| `Group-Object` | Grupperer etter avsenderdomene, delen etter `@` |
| `Sort-Object Count -Descending` | Vanligste domene øverst |

</details>

Erstatt `5.1.8` med statuskoden du undersøker. Utdataene besvarer spørsmålet på én linje:

```text
Count Name
----- ----
    9 example-test.com
```

Ett enkelt avsenderdomene betyr: et avgrenset problem, ingen hendelse, og du kan fortsette å lete i ro. Hvis det sto tjue forskjellige domener der, ville du hatt et pågående utfall, og alt annet måtte vente. Å gjøre dette skillet så tidlig sparer erfaringsmessig mest tid.

## Feilkilde 3: `ConnectorId` oppgir ikke den faktiske Receive Connectoren

Dette er den dyreste feilkilden, fordi utdataene ser seriøse ut. E-post som leveres av en klient eller et eksternt system på port 25, treffer først **Front End Transport**. Denne sender meldingen videre til **Transport Service** på port 2525. Message Tracking skrives først der; Front End Transport skriver ikke egen tracking.

Konsekvensen ser du på denne linjen:

```text
EventId        : RECEIVE
ConnectorId    : SRV-MAIL01\Default SRV-MAIL01
ClientIp       : 10.0.1.11
ClientHostname : srv-mail01.intern.example.com
```

`ConnectorId` angir den interne connectoren på port 2525, og `ClientIp` er adressen til **serveren som proxyer**, ikke den opprinnelige avsenderen. Hvilken av de konfigurerte connectorene på port 25 et system faktisk traff, står ganske enkelt ikke i tracking. Den som stoler på denne opplysningen, leter etter feilen i en connector som ikke var involvert.

Det finnes to veier til svaret. Den første er rekonstruksjon via konfigurasjonen:

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object { Get-ReceiveConnector -Server $_ } |
    Format-List Identity, Enabled,
        @{n='Bindings';       e={$_.Bindings -join ','}},
        @{n='RemoteIPRanges'; e={$_.RemoteIPRanges -join ','}},
        PermissionGroups, AuthMechanism
```

<details class="options-details">
<summary>Forklaring av alternativene</summary>

| Alternativ | Virkning |
|---|---|
| `-Server` | Serveren som Receive Connectorene listes for |
| `Format-List` | Full feltlengde; `RemoteIPRanges` og `PermissionGroups` ville blitt avkortet i tabeller |
| `@{n=…; e=…}` | Beregnede felt: samler samlefeltene `Bindings` og `RemoteIPRanges` til kommaseparerte tekststrenger |

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

Fastslå kilde-IP-en til systemet som leverer inn, og finn connectoren der `RemoteIPRanges` inneholder den. Hvis den ikke faller innenfor noen av de begrensede connectorene, gjenstår Default Frontend Connectoren, som vanligvis godtar hele adresserommet. Bruk også her `Format-List`, fordi `RemoteIPRanges` og `PermissionGroups` regelmessig blir avkortet i tabeller.

Den andre veien er SMTP-protokollen, og den fortjener en egen seksjon.

## SMTP-protokollen: den eneste komplette kilden

Protokollen til Front End Transport registrerer hele SMTP-økten: hvilken connector som ble kontaktet, hvilken IP som koblet til, og hva klient og server sa til hverandre. Det er den eneste kilden som løser problemet med `ConnectorId` beskrevet ovenfor.

### Slå på logging

Som standard er logging **slått av** på de fleste connectorer. Du slår den på per connector:

```powershell
Set-ReceiveConnector -Identity "SRV-MAIL01\Default Frontend SRV-MAIL01" `
    -ProtocolLoggingLevel Verbose
```

<details class="options-details">
<summary>Forklaring av alternativene</summary>

| Alternativ | Virkning |
|---|---|
| `-Identity` | Connectoren som skal endres, i formen `Server\Connectorname` |
| `-ProtocolLoggingLevel Verbose` | Slår på SMTP-logging; `None` slår den av igjen |

</details>

For utgående forbindelser gjøres det tilsvarende med `Set-SendConnector`. Husk å sette verdien tilbake til `None` etter analysen, fordi detaljert logging bruker diskplass og skriver betydelige datamengder ved høy trafikk.

### Hvor filene ligger

Exchange skiller protokollene etter tjeneste og retning. Det er unødvendig å hardkode stiene; hent dem heller:

```powershell
Get-FrontendTransportService SRV-MAIL01 |
    Format-List ReceiveProtocolLogPath, SendProtocolLogPath,
        ReceiveProtocolLogMaxAge, ReceiveProtocolLogMaxDirectorySize

Get-TransportService SRV-MAIL01 |
    Format-List ReceiveProtocolLogPath, SendProtocolLogPath
```

<details class="options-details">
<summary>Forklaring av alternativene</summary>

| Alternativ | Virkning |
|---|---|
| `SRV-MAIL01` | Posisjonsparameteren `-Identity`: serveren det skal spørres mot |
| `ReceiveProtocolLogPath`, `SendProtocolLogPath` | Lagringsstier for protokollene for henholdsvis innkommende og utgående forbindelser |
| `ReceiveProtocolLogMaxAge` | Maksimal alder på protokollfilene; eldre filer slettes |
| `ReceiveProtocolLogMaxDirectorySize` | Øvre grense for diskplassen protokollkatalogen kan bruke |

</details>

Vanligvis ligger de under installasjonsstien i `TransportRoles\Logs\FrontEnd\ProtocolLog\SmtpReceive` for Front End Transport og i `TransportRoles\Logs\Hub\ProtocolLog\SmtpReceive` for Transport Service. **Dette er kjernen:** Klientforbindelser på port 25 finner du bare i `FrontEnd`-stien; i `Hub`-stien står bare intern videresendingstrafikk på 2525.

Vær oppmerksom på oppbevaringen. `ReceiveProtocolLogMaxAge` står ofte på 30 dager, og `ReceiveProtocolLogMaxDirectorySize` begrenser i tillegg plassforbruket. Ved høy trafikk trer størrelsesbegrensningen i kraft betydelig før aldersgrensen, og da er protokollene dine bare noen få dager gamle.

### Forstå formatet

Filene er CSV-filer med overskriftslinjer som begynner med `#`. De viktigste kolonnene er `date-time`, `connector-id`, `session-id`, `sequence-number`, `local-endpoint`, `remote-endpoint`, `event` og `data`.

Avgjørende er kolonnen `event`, ett enkelt tegn:

| Tegn | Betydning |
|---|---|
| `+` | Forbindelse opprettet |
| `-` | Forbindelse avsluttet |
| `>` | Server sender til klient |
| `<` | Klient sender til server |
| `*` | Serverinformasjon, ikke SMTP-trafikk |

Du kjenner igjen en økt på felles `session-id`; `sequence-number` angir rekkefølgen i økten. Et typisk utdrag ser slik ut:

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

Her står alt som manglet i Message Tracking: den **faktiske** connectoren (`smtp-noauth`), den **faktiske** kilde-IP-en (`10.0.20.22`) og navnet systemet melder seg med i `EHLO`.

### Målrettet søk

For enkelttilfeller er et tekstfilter betydelig raskere enn objektanalyse. Søk etter avsenderadressen eller `EHLO`-navnet og hent øktidentifikatoren:

```powershell
$pfad = (Get-FrontendTransportService SRV-MAIL01).ReceiveProtocolLogPath
Select-String -Path "$pfad\*.log" -Pattern "dienst@example-test.com" -SimpleMatch |
    Select-Object -First 5 Line
```

<details class="options-details">
<summary>Forklaring av alternativene</summary>

| Alternativ | Virkning |
|---|---|
| `-Path "$pfad\*.log"` | Søker gjennom alle protokollfiler i stien som ble hentet tidligere |
| `-Pattern` | Søkeordet, her avsenderadressen |
| `-SimpleMatch` | Behandler mønsteret som tekst i stedet for regulært uttrykk; punktumet i adressen trenger dermed ikke escaping |
| `-First 5` | Begrenser utdata til de første fem treffene |

</details>

Med den funne `session-id` henter du hele økten:

```powershell
Select-String -Path "$pfad\*.log" -Pattern "08DEF44EC454A414" -SimpleMatch |
    ForEach-Object { $_.Line } | Select-Object -First 40
```

<details class="options-details">
<summary>Forklaring av alternativene</summary>

| Alternativ | Virkning |
|---|---|
| `-Pattern` | Øktidentifikatoren fra første treff |
| `-SimpleMatch` | Ordrett søk uten regex-evaluering |
| `-First 40` | Begrenser utdata til de første 40 linjene i økten |

</details>

Hvis du bare vil vite hvilke connectorer som i det hele tatt har trafikk, teller du forbindelsesopprettelsene. Dette er mange størrelsesordener raskere enn å parse hver linje i store filer:

```powershell
Select-String -Path "$pfad\*.log" -Pattern ',\+,' |
    ForEach-Object { ($_.Line -split ',')[1] } |
    Group-Object | Sort-Object Count -Descending |
    Format-Table Count, Name -AutoSize
```

<details class="options-details">
<summary>Forklaring av alternativene</summary>

| Alternativ | Virkning |
|---|---|
| `-Pattern ',\+,'` | Regulært uttrykk for hendelsen `+` (forbindelsesopprettelse) mellom to CSV-kommaer; plusstegnet er escaped |
| `ForEach-Object { … -split ',' }` | Deler trefflinjen på kommaene og henter andre kolonne, `connector-id` |
| `Group-Object` | Teller forbindelsesopprettelser per connector |
| `Sort-Object Count -Descending` | Mest brukte connector øverst |

</details>

```text
Count Name
----- ----
51479 SRV-MAIL01\Default Frontend SRV-MAIL01
50756 SRV-MAIL01\smtp-auth SRV-MAIL01
19405 SRV-MAIL01\smtp-intern SRV-MAIL01
15789 SRV-MAIL01\smtp-noauth SRV-MAIL01
```

Denne fordelingen besvarer et spørsmål som ikke kan besvares i Message Tracking: Hvilke veier bruker applikasjonene dine faktisk? Før en connector-endring er dette det viktigste tallet av alle.

### Når ingenting ble logget

Hvis det mangler enhver linje på det aktuelle tidspunktet, finnes det tre vanlige årsaker: Logging var slått av på den aktuelle connectoren, oppbevaringsgrensen har allerede fjernet filen, eller du ser i feil sti – altså i `Hub`- i stedet for `FrontEnd`-katalogen. Kontroller i denne rekkefølgen.

## Trinn 4: Kontroller rettigheter

Når en innlevering avvises, eller omvendt når mer er tillatt enn antatt, går veien via connectorens rettigheter. Her finnes en teknisk særegenhet: `Get-ADPermission` krever **DistinguishedName**. Hvis du sender inn den vanlige identiteten i formen `Server\Connectorname`, mislykkes kallet i en ekstern økt med den misvisende meldingen om at objektet ikke ble funnet.

```powershell
$dn = (Get-ReceiveConnector "SRV-MAIL01\Default Frontend SRV-MAIL01").DistinguishedName
Get-ADPermission -Identity $dn -User "NT AUTHORITY\ANONYMOUS LOGON" |
    Where-Object { $_.ExtendedRights -like "*ms-Exch-SMTP*" } |
    Format-Table User, @{n='Rights'; e={$_.ExtendedRights}} -AutoSize
```

<details class="options-details">
<summary>Forklaring av alternativene</summary>

| Alternativ | Virkning |
|---|---|
| `-Identity $dn` | Objektet som skal kontrolleres som DistinguishedName; formen `Server\Connectorname` mislykkes i eksterne økter |
| `-User` | Begrenser utdata til rettighetene for denne sikkerhetsidentiteten, her anonym tilgang |
| `Where-Object` | Filtrerer på SMTP-relevante Extended Rights |

</details>

```text
User                         Rights
----                         ------
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Submit
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Accept-Any-Sender
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Accept-Authoritative-Domain-Sender
```

Vurderingen er enklere enn den ser ut når du skiller mellom fire rettigheter:

| Rettighet | Betydning |
|---|---|
| `ms-Exch-SMTP-Submit` | Kan i det hele tatt levere inn |
| `ms-Exch-SMTP-Accept-Any-Sender` | Kan bruke vilkårlige avsenderadresser |
| `ms-Exch-SMTP-Accept-Authoritative-Domain-Sender` | Kan opptre som eget domene |
| `ms-Exch-SMTP-Accept-Any-Recipient` | **Kan videresende til eksterne domener** |

De første tre er standardsettet for anonym innlevering og nødvendig for mottak av Internett-e-post. Først den fjerde rettigheten gjør en inngående connector til et relay. På en connector som godtar hele adresserommet, er det et åpent relay. På en connector med streng IP-begrensning er det derimot den vanlige og tilsiktede løsningen for at applikasjonsservere skal kunne sende eksternt.

Ikke forveksle `Accept-Any-Sender` med `Accept-Any-Recipient`. Den første er ufarlig og nødvendig, den andre er den sikkerhetsrelevante innstillingen.

## Trinn 5: Kontrolltest med egen innlevering

Hvis vurderingen forblir tvetydig, leverer du selv inn en melding. Da kontrollerer du avsender, mottaker og innleveringspunkt fullt ut:

```powershell
Send-MailMessage -SmtpServer 10.0.1.11 -Port 25 `
    -From 'test@example.com' `
    -To 'empfaenger@example.net' `
    -Subject 'Diagnose, bitte ignorieren' `
    -Body 'Testeinlieferung' `
    -Encoding UTF8
```

<details class="options-details">
<summary>Forklaring av alternativene</summary>

| Alternativ | Virkning |
|---|---|
| `-SmtpServer` | Målvert for innleveringen, her bevisst som IP-adresse for å treffe et bestemt endepunkt |
| `-Port 25` | Målport; 25 for uautentisert server-til-server-innlevering |
| `-From` | Envelope- og header-avsender for testmeldingen |
| `-To` | Mottakeradresse |
| `-Subject` | Emnelinje |
| `-Body` | Meldingstekst |
| `-Encoding UTF8` | Tegnkoding for emne og tekst, unngår problemer med spesialtegn |

</details>

`Send-MailMessage` er offisielt avviklet, men er fortsatt det raskeste verktøyet til diagnoseformål og finnes på alle Windows-servere. Ved suksess gis det ingen utdata, noe som kan virke uvant.

Hvis du tester en TLS-strekning på port 587 og motparten presenterer et sertifikat som ikke passer til navnet som brukes, for eksempel fordi du bruker IP-adressen, avbrytes kallet med en sertifikatfeil. Til testen kan du deaktivere kontrollen i økten:

```powershell
[Net.ServicePointManager]::ServerCertificateValidationCallback = { $true }
```

Dette gjelder bare den aktive PowerShell-økten. Bruk det bevisst og aldri i skript som kjører i drift.

Hvis testmeldingen kommer frem og du vil vite hva som skjedde med den underveis, hjelper [Mail Header Analyzer](https://rafaelpfister.ch/tools/header-analyzer): Den deler opp headerne, tegner veien via hoppene og viser resultatene av autentiseringskontrollene, helt lokalt i nettleseren, uten at meldingen forlater enheten din.

## Exchange Online: samme spørsmål, et annet verktøy

I tenant-en gjelder andre regler, og dette er punktet der vante fremgangsmåter feiler. Regn med disse forskjellene:

| | Exchange OnPrem | Exchange Online |
|---|---|---|
| Spørring | `Get-MessageTrackingLog` | `Get-MessageTraceV2` |
| Detaljnivå | Hver transporthendelse | Én linje per melding og mottaker |
| Connector synlig | Ja, med begrensning (se over) | **Nei** |
| Servertilknytning | Ja, spør per server | Ikke relevant |
| SMTP-protokoll | Tilgjengelig | **Ikke tilgjengelig** |
| Oppbevaring | Din konfigurasjon | Omtrent 10 dager via cmdleten |
| Forsinkelse | Nesten umiddelbart | Noen minutter |

De tre viktigste praktiske konsekvensene: Det finnes **ingen connector-tilordning**, du må bruke `FromIP` og `ToIP`. Det finnes **ingen SMTP-protokoll**, SMTP-samtalen kan ikke rekonstrueres. Og dataene vises **forsinket**, en nettopp sendt melding dukker ikke opp umiddelbart.

### Grunnspørringen

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-4) `
    -EndDate (Get-Date) `
    -RecipientAddress "empfaenger@example.com" `
    -ResultSize 1000 |
  Sort-Object Received |
  Format-Table Received, SenderAddress, RecipientAddress, Status, FromIP, Size -AutoSize
```

<details class="options-details">
<summary>Forklaring av alternativene</summary>

| Alternativ | Virkning |
|---|---|
| `-StartDate` | Nedre tidsgrense for spørringen, her de siste fire timene |
| `-EndDate` | Øvre tidsgrense; cmdleten krever begge grensene |
| `-RecipientAddress` | Filtrerer på meldinger til denne mottakeradressen |
| `-ResultSize 1000` | Maksimalt antall linjer på denne siden; øvre grense er 5000 |

</details>

```text
Received            SenderAddress          RecipientAddress          Status    FromIP
--------            -------------          ----------------          ------    ------
11.08.2026 08:27:16 emma@partner.example   empfaenger@example.com    Delivered 10.0.20.23
11.08.2026 09:05:24 dienst@example-test.com empfaenger@example.com   Failed    10.0.20.23
```

De viktigste verdiene for `Status`: `Delivered`, `Failed`, `Pending`, `Quarantined`, `FilteredAsSpam` og `Expanded` for utvidede distribusjonsgrupper. `Pending` betyr at leveringsforsøk fortsatt pågår, ikke at noe er ødelagt.

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

<details class="options-details">
<summary>Forklaring av alternativene</summary>

| Alternativ | Virkning |
|---|---|
| `-MessageTraceId` | Entydig identifikator for meldingen fra grunnspørringen; obligatorisk |
| `-RecipientAddress` | Mottakeren hvis behandlingssteg vises; også obligatorisk, siden en melding kan ha flere mottakere |

</details>

Der står behandlingsstegene i tjenesten, for eksempel regelanvendelser, filteravgjørelser og årsaken til en avvisning.

### Lenger tilbake enn ti dager

Cmdleten går omtrent ti dager tilbake. For eldre perioder finnes det et historisk søk som kjører asynkront og leverer resultatet som CSV, med et tidsrom på opptil 90 dager:

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
<summary>Forklaring av alternativene</summary>

| Alternativ | Virkning |
|---|---|
| `-ReportTitle` | Fritt valgt navn på oppgaven, som resultatet senere kan finnes under |
| `-StartDate`, `-EndDate` | Undersøkt tidsrom, opptil 90 dager tilbake |
| `-ReportType MessageTrace` | Rapporttype; `MessageTrace` leverer meldingsoversikten som CSV |
| `-SenderAddress` | Filtrerer på denne avsenderadressen |
| `-NotifyAddress` | Mottaker av ferdigmeldingen; må være en adresse i en Accepted Domain i tenant-en |

</details>

Beregn tid til dette; slike oppgaver kan kjøre i timer, avhengig av omfanget.

### Feilkilde 4: Manglende treff er ikke bevis på manglende trafikk

Dette er den mest subtile feilkilden i tenant-en. `Get-MessageTraceV2` leverer sidevis, maksimalt 5000 linjer per kall. Ved høy trafikk kan én side bare dekke noen få minutter, selv om du har spurt etter syv dager. Filtrerer du deretter lokalt, for eksempel etter en kilde-IP, filtrerer du bare et svært lite utdrag.

Dette ser du på advarselen som viser at det finnes flere resultater:

```text
WARNING: There are more results, use the following command to get more.
Get-MessageTraceV2 -StartDate "2026-08-11T07:25:19Z" -EndDate "2026-08-11T09:05:46Z"
  -StartingRecipientAddress "naechster@example.com" -ResultSize 5000
```

Hvis den vises, er analysen din **ufullstendig**. Hvis det ikke kommer tilbake noen treff, er korrekt resultat: Ikke funnet i utdraget. Det er ikke: Finnes ikke.

Det finnes to ryddige utveier. Enten reduserer du tidsvinduet til én side dekker det helt, noe du ser ved at advarselen uteblir. Eller du går gjennom alle sidene ved hjelp av fortsettelsesopplysningene i advarselen. For spørsmålet om noe **aldri** forekommer, er en konfigurasjonskontroll uansett bedre: Hvis et system ikke har en rute til et mål, kan det ikke levere dit, uavhengig av ethvert observasjonsvindu.

Den fullstendige analysen av alle innleverende adresser er et eget tema, med egne krevende punkter ved tolkningen. Den finner du i [Hvem leverer egentlig inn i tenant-en din? Aggregere innleverende IP-adresser](https://rafaelpfister.ch/blog/einliefernde-ip-adressen-aggregieren).

## En fremgangsmåte som har vist seg å fungere

Oppsummert har denne rekkefølgen vist seg å være den raskeste. Søk etter meldingen på alle servere og fastslå siste hendelse. Ved feil, bytt straks til `Format-List` og les hele SMTP-svaret, i stedet for å trekke slutninger fra hendelsestypen. Avklar deretter omfanget, altså grupper og tell. Først når saken er avgrenset, rekonstruerer du innleveringsveien via connector-konfigurasjon og SMTP-protokoll. Til slutt, ved behov, kontrollerer du med egen innlevering.

De vanligste tidstyvene er derimot alltid de samme: Man leser en avkortet tabell i stedet for hele feilmeldingen, man oppfatter skyggekopier som behandlingssteg, man stoler på `ConnectorId` i tracking, og man anser et tomt utvalg som bevis. Den som kjenner disse fire, kommer som regel til riktig nivå i løpet av få minutter.

## Kilder

1.  [Message tracking in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): Feltbeskrivelse og fullstendig liste over hendelsestyper i Message Tracking.

2.  [Protocol logging in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/protocol-logging): Lagringssteder, format og oppbevaring av SMTP-protokoller, inkludert Front End Transport.

3.  [Shadow redundancy in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/shadow-redundancy): Forklarer hendelsene rundt skyggekopier og forkastingen av dem.

4.  [Mail routing in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing): Samspillet mellom Front End Transport og Transport Service, grunnlaget for proxy-atferden.

5.  [Receive connectors in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/receive-connectors): Bindinger, rettighetsgrupper og autentiseringsmekanismer.

6.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2): Etterfølgeren til Get-MessageTrace, inkludert sidelogikk og feltliste.

7.  [Start-HistoricalSearch](https://learn.microsoft.com/en-us/powershell/module/exchange/start-historicalsearch): Asynkron meldingssporing over opptil 90 dager.
