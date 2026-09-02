---
title: "Vanlige årsaker til e-postsløyfer og hvordan de løses"
navTitle: "Løs e-postsløyfer"
description: "Slik kan SMTP-e-postsløyfer i Exchange Online, hybridmiljøer og foranliggende e-postgatewayer systematisk identifiseres og løses ved hjelp av NDR, hoder, Message Trace, mottakerobjekter og koblinger."
date: "2026-08-07"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "12 min lesetid"
themen:
  - exchange-onprem-hybrid
  - microsoft-365-exchange
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange"
protokolle:
  - "smtp"
  - "powershell"
  - "troubleshooting"
related:
  - exchange-mailboxen-ohne-remote-move-cloud-neu-erstellen
  - totemomail-m365
  - ghost-sender-exchange-online-nebeneingang
slug: "vanlige-arsaker-til-e-postsloyfer-og-hvordan-de-loses"
translationId: "article-4c91e7b2a8605fd3"
draft: false
translationOf: typische-ursachen-fuer-mail-loops-und-deren-behebung
translationSourceHash: c71063cb6e7d05a1f311a5269e4d6805d8b219e8d0fb103485738925ef99f990
translationModel: gpt-5.6-terra
translatedAt: 2026-09-01T09:05:07.297Z
translationReview: automatic
url: https://rafaelpfister.ch/no/blog/vanlige-arsaker-til-e-postsloyfer-og-hvordan-de-loses
---

# Vanlige årsaker til e-postsløyfer og hvordan de løses

En e-postsløyfe oppstår når minst to transportsystemer overleverer den samme meldingen til hverandre gjentatte ganger. Ingen av systemene oppfatter seg selv som det endelige målet, men begge kjenner til et angivelig passende neste hopp. Sløyfen avsluttes først når en server ser at det tillatte antallet transportstasjoner er overskredet og genererer en NDR.

I Exchange er to meldinger særlig informative:

- `554 5.4.6 Hop count exceeded - possible mail loop` genereres vanligvis av den lokale Exchange-serveren.
- `554 5.4.14 Hop count exceeded - possible mail loop ATTR34` genereres av Exchange Online.

Hop-grensen er ikke årsaken, men sikringen mot en endeløs gjentakelse. Å øke den løser derfor ingenting. Målet er å finne punktet der meldingen, i strid med målarkitekturen, returneres til et system den allerede har passert.

## Gjenkjenn sløyfemønsteret i hodet

NDR-en og de fullstendige opprinnelige meldingshodene bør sikres før enhver endring. `Received`-linjer leses nedenfra og opp: Den nederste linjen er det tidligste dokumenterte hoppet, den øverste er det nyeste.

En sløyfe viser seg vanligvis som en gjentakende sekvens:

```text
Internet
  → Exchange Online Protection
  → Mailgateway
  → Exchange Online Protection
  → Mailgateway
  → Exchange Online Protection
  → ...
```

Ikke alle Microsoft-vertsnavn som forekommer flere ganger, er allerede en sløyfe. Exchange Online behandler meldinger internt gjennom flere transportroller. Det som er påfallende, er den gjentatte returen mellom de samme administrative grensene, for eksempel mellom Exchange Online og en lokal gateway. Tidsstempel, sendende IP, mottakende vert og `Message-ID` bidrar til å identifisere runden entydig.

For den første analysen besvares disse spørsmålene:

1. Hvilket system genererte NDR-en?
2. Hvilke to eller tre hopp gjentar seg?
3. Hvilket system skulle ha levert meldingen endelig?
4. På grunnlag av hvilken domene-, mottaker-, koblings- eller regelbeslutning ble den videresendt?
5. Hvilken endring påvirket sist e-postflyten?

## Diagnose i Exchange Online

Med `Get-MessageTraceV2` kan behandlingen de siste 90 dagene undersøkes; maksimalt ti dager er tillatt per spørring. Et smalt tidsvindu og den konkrete mottakeradressen gir de mest nyttige resultatene:

```powershell
$start = (Get-Date).AddHours(-2)
$end = Get-Date
$recipient = "user01@contoso.com"

$trace = Get-MessageTraceV2 `
    -RecipientAddress $recipient `
    -StartDate $start `
    -EndDate $end `
    -ResultSize 5000

$trace |
    Select-Object Received,SenderAddress,RecipientAddress,Subject,
        Status,FromIP,ToIP,MessageTraceId,MessageId |
    Sort-Object Received
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `-RecipientAddress` | Filtrerer sporingen på den angitte mottakeradressen |
| `-StartDate` / `-EndDate` | Tidsvindu for spørringen; maksimalt ti dager er tillatt per spørring |
| `-ResultSize 5000` | Maksimalt antall returnerte oppføringer |
| `Select-Object …` | Reduserer utdataene til feltene som er relevante for sløyfeanalysen |
| `Sort-Object Received` | Sorterer treffene kronologisk etter mottakstidspunkt |

</details>

Detaljene for et treff viser enkelte transporthendelser:

```powershell
$trace | ForEach-Object {
    Get-MessageTraceDetailV2 `
        -MessageTraceId $_.MessageTraceId `
        -RecipientAddress $_.RecipientAddress
} | Format-Table Date,Event,Action,Detail -AutoSize
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `-MessageTraceId` | Entydig sporings-ID fra resultatet av `Get-MessageTraceV2` |
| `-RecipientAddress` | Mottakeradresse for treffet; kreves sammen med sporings-ID-en for detaljspørringen |
| `Format-Table … -AutoSize` | Tilpasser kolonnebreddene til innholdet slik at hendelsesdetaljene forblir lesbare |

</details>

Deretter hentes domene, mottaker og koblinger inn samlet:

```powershell
Get-AcceptedDomain |
    Format-Table Name,DomainName,DomainType,MatchSubDomains -AutoSize

Get-EXORecipient -Identity $recipient |
    Format-List DisplayName,RecipientTypeDetails,PrimarySmtpAddress,
        ExternalEmailAddress,EmailAddresses

Get-OutboundConnector -IncludeTestModeConnectors |
    Format-List Name,Enabled,ConnectorType,RecipientDomains,SmartHosts,
        UseMXRecord,RouteAllMessagesViaOnPremises,TlsSettings

Get-InboundConnector |
    Format-List Name,Enabled,ConnectorType,SenderDomains,SenderIPAddresses,
        TlsSenderCertificateName,RequireTls,RestrictDomainsToIPAddresses,
        RestrictDomainsToCertificate
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `-Identity $recipient` | Velger mottakerobjektet via adresse, alias eller navn |
| `-IncludeTestModeConnectors` | Tar også med koblinger i testmodus i utdataene |
| `Format-Table … -AutoSize` | Tabellvisning med kolonnebredder basert på innhold |
| `Format-List …` | Listevisning av de angitte egenskapene, egnet for lange verdier som adresselister |

</details>

Det avgjørende er ikke om ett enkelt objekt ser plausibelt ut. Domenetype, faktisk mottakertype og gjeldende kobling må sammen beskrive det samme målet.

## Diagnose i lokal Exchange

I et hybridmiljø kontrolleres den samme mottakeren også lokalt. Spørringene skiller mellom en ekte lokal postboks, en RemoteMailbox og en MailUser:

```powershell
Get-Recipient -Identity $recipient |
    Format-List DisplayName,RecipientType,RecipientTypeDetails,
        PrimarySmtpAddress,EmailAddresses

Get-Mailbox -Identity $recipient -ErrorAction SilentlyContinue |
    Format-List RecipientTypeDetails,ServerName,Database,PrimarySmtpAddress

Get-RemoteMailbox -Identity $recipient -ErrorAction SilentlyContinue |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,RemoteRoutingAddress

Get-MailUser -Identity $recipient -ErrorAction SilentlyContinue |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,ExternalEmailAddress
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `-Identity $recipient` | Velger objektet via adresse, alias eller navn |
| `-ErrorAction SilentlyContinue` | Undertrykker feilmeldingen dersom objektet ikke finnes i den aktuelle typen; spørringen returnerer da ganske enkelt ikke noe resultat |

</details>

For transportstien trengs send- og mottakskoblinger samt sporingsloggene:

```powershell
Get-SendConnector |
    Format-List Name,Enabled,AddressSpaces,DNSRoutingEnabled,SmartHosts,
        SourceTransportServers,CloudServicesMailEnabled,TlsDomain

Get-ReceiveConnector |
    Format-List Identity,Enabled,Bindings,RemoteIPRanges,PermissionGroups

$servers = Get-ExchangeServer |
    Where-Object { $_.IsMailboxServer -or $_.IsHubTransportServer }

$servers |
    Get-MessageTrackingLog `
        -Start $start `
        -End $end `
        -Recipients $recipient `
        -ResultSize Unlimited |
    Select-Object Timestamp,ServerHostname,ClientHostname,Source,EventId,
        ConnectorId,Sender,Recipients,MessageId,NetworkMessageId |
    Sort-Object Timestamp
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `Where-Object { … }` | Begrenser serverlisten til Mailbox- og Hub Transport-servere, altså rollene med sporingslogger |
| `-Start` / `-End` | Tidsvindu for logsøket |
| `-Recipients $recipient` | Filtrerer på sporingshendelser med denne mottakeradressen |
| `-ResultSize Unlimited` | Opphever standardgrensen på 1000 returnerte oppføringer |
| `Select-Object …` | Reduserer utdataene til feltene som er relevante for stianalysen |
| `Sort-Object Timestamp` | Sorterer hendelsene fra alle servere kronologisk |

</details>

En `SEND` til Exchange Online, etterfulgt av en ny `RECEIVE` av den samme meldingen fra Exchange Online, synliggjør returen. Med `MessageId` og `NetworkMessageId` kan man unngå å forveksle ulike testmeldinger.

## De vanligste årsakene i oversikt

| Mønster | Typisk årsak | Løsning |
| --- | --- | --- |
| Ukjente mottakere pendler mellom to systemer | Accepted Domain står på `InternalRelay`, men begge sider videresender ukjente mottakere | Definer et entydig ansvar; bruk `Authoritative` ved fullstendig EXO-levering, eller fastsett ett avsluttende hopp for Split-Domain |
| EXO sender til lokal Exchange, som umiddelbart sender tilbake til EXO | Hybrid-kobling eller Centralized Mail Transport passer ikke lenger med postboksplasseringen | Kontroller HCW-konfigurasjonen og `RouteAllMessagesViaOnPremises`; deaktiver utdatert sentral rute eller korriger lokal mottakeroppløsning |
| Meldingen pendler mellom EXO og en sikkerhets-, signatur- eller krypteringsgateway | Returnerte meldinger oppfyller utgangsregelen på nytt | Bruk en header satt av gatewayen eller en dokumentert mekanisme for forebygging av sløyfer som unntak; autentiser inn- og utgående koblinger entydig |
| Bare én mottaker er berørt | Utdatert eller feil `targetAddress`, feil RemoteMailbox-type eller motstridende proxy-adresser | Fastsett Source of Authority, korriger mottakerobjektet der og synkroniser |
| Bare videresendte meldinger går i sløyfe | Transportregel, postboksvideresending eller innboksregel adresserer den opprinnelige stien på nytt | Deaktiver regelen, korriger målet og definer et robust unntak |
| Bare ett underdomene eller én applikasjon er berørt | Overordnet domene dekker ikke underdomenet riktig i den forventede koblingsstien | Konfigurer underdomenet eksplisitt som Accepted Domain og i riktig Send Connector |
| Alle meldinger går i sløyfe etter en gateway- eller DNS-endring | Smart Host eller MX peker mot inngangen til det sendende systemet | Korriger neste hopp og kontroller DNS-, NAT- og Load Balancer-mål separat |

## Årsak 1: Feil type Accepted Domain

Et authoritative-domene betyr: Alle gyldige mottakere for dette domenet er kjent i Exchange-organisasjonen; ukjente mottakere avvises. Et Internal Relay-domene betyr: En del av mottakerne befinner seg i et annet system og må videresendes via en Send- eller Outbound-Connector.

Den problematiske konstellasjonen oppstår når Exchange Online sender ukjente mottakere til et lokalt system, og dette heller ikke behandler det samme domenet avsluttende, men sender det tilbake til Exchange Online via MX eller Smart Host.

```powershell
Get-AcceptedDomain -Identity contoso.com |
    Format-List DomainName,DomainType,MatchSubDomains
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `-Identity contoso.com` | Velger Accepted Domain som skal kontrolleres |
| `Format-List …` | Viser domenenavn, domenetype og underdomenedekning som en liste |

</details>

Når alle mottakere befinner seg i Exchange Online etter fullført migrering, er `Authoritative` vanligvis riktig måltilstand:

```powershell
# Kjør bare etter fullstendig kontroll av mottakere og ruting.
Set-AcceptedDomain -Identity contoso.com -DomainType Authoritative
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `-Identity contoso.com` | Accepted Domain som skal endres |
| `-DomainType Authoritative` | Setter domenet til authoritative: ukjente mottakere avvises i stedet for å videresendes |

</details>

Ved et reelt Split-Domain kan `InternalRelay` være riktig. Da kreves det imidlertid en tydelig kobling til systemet som kjenner de gjenværende mottakerne. Dette målet må ikke sende ukjente adresser tilbake til utgangspunktet.

## Årsak 2: Overlappende hybridkoblinger og Centralized Mail Transport

Centralized Mail Transport ruter utgående meldinger fra Exchange Online bevisst via lokal Exchange. Dette er nyttig for bestemte samsvarskrav, men skaper ekstra transportveier. Hvis alternativet forblir aktivt etter en migrering, selv om det lokale systemet sender meldinger tilbake til Exchange Online via sin egen MX, kan det oppstå en sirkel.

```powershell
Get-OutboundConnector -IncludeTestModeConnectors |
    Format-Table Name,Enabled,ConnectorType,RouteAllMessagesViaOnPremises,
        RecipientDomains,SmartHosts,UseMXRecord -AutoSize
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `-IncludeTestModeConnectors` | Tar også med koblinger i testmodus i utdataene |
| `Format-Table … -AutoSize` | Tabellvisning av rutingegenskapene med kolonnebredder basert på innhold |

</details>

Flere koblinger med overlappende omfang bør også kontrolleres. Microsoft anbefaler en dedikert lokal kobling for hybrid e-postflyt; reparasjon med Hybrid Configuration Wizard er ofte sikrere enn isolerte enkeltendringer.

Hvis Centralized Mail Transport dokumentert ikke lenger er nødvendig, kan innstillingen deaktiveres målrettet:

```powershell
# Bare etter kontroll av samsvars- og gatewaykrav.
Set-OutboundConnector `
    -Identity "Outbound to On-Premises" `
    -RouteAllMessagesViaOnPremises:$false
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `-Identity "Outbound to On-Premises"` | Outbound Connector som skal endres |
| `-RouteAllMessagesViaOnPremises:$false` | Deaktiverer Centralized Mail Transport: utgående meldinger fra Exchange Online går ikke lenger via lokal Exchange |

</details>

## Årsak 3: En gateway behandler returmeldinger på nytt

I et inn-og-ut-scenario sender Exchange Online en melding til en tilleggstjeneste for signering, kryptering eller arkivering. Denne returnerer den deretter til Exchange Online. Utgangsregelen må gjenkjenne returmeldingen; ellers sendes den til tjenesten på nytt.

Kontrollen begynner med alle regler som velger koblinger, omdirigerer mottakere eller evaluerer headere:

```powershell
Get-TransportRule |
    Sort-Object Priority |
    Format-List Name,State,Mode,Priority,FromScope,SentToScope,
        RedirectMessageTo,RouteMessageOutboundConnector,
        SetHeaderName,SetHeaderValue,ExceptIfHeaderContainsMessageHeader,
        ExceptIfHeaderContainsWords
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `Sort-Object Priority` | Sorterer reglene i evalueringsrekkefølgen deres |
| `Format-List …` | Viser egenskapene som velger koblinger, omdirigerer mottakere eller angir headere eller evaluerer dem som unntak |

</details>

Det konkrete unntaket må følge dokumentasjonen fra gateway-produsenten. Vanlig er en header satt av tjenesten som ikke kan forfalskes troverdig fra Internett. I tillegg bør innkommende koblinger identifisere tjenesten ved hjelp av sertifikat eller fast avsender-IP. Et generelt unntak for alle meldinger som fremstår som «interne», er for bredt.

## Årsak 4: Mottakerobjektet og den faktiske postboksen befinner seg ikke på samme sted

Et objekt kan vises i Exchange Online som `MailUser`, selv om den aktive postboksen ligger lokalt. I et synkronisert hybridmiljø er dette ikke automatisk et duplikat. Heller ikke en `ExternalEmailAddress`, som tilsvarer den primære SMTP-adressen, beviser alene en feilkonfigurasjon.

Avgjørende er kombinasjonen av alle spørringene:

- `Get-Mailbox` lokalt gir et resultat: Den aktive postboksen ligger lokalt.
- `Get-RemoteMailbox` lokalt gir et resultat: Det administrerte målet ligger i Exchange Online.
- `Get-EXOMailbox` gir et resultat: Det finnes en ekte postboks i skyen.
- `Get-EXORecipient` gir bare en MailUser: Objektet er et rutingmål, ikke en skypostboks.

Problematiske er utdaterte objekter etter en migrering, feil eksterne rutingdomener eller manuelt angitte `targetAddress`-verdier, der domenet fører tilbake via samme transportvei. Endringer utføres ved Source of Authority: I synkroniserte miljøer altså med Exchange-administrasjonsverktøy lokalt og ikke ved å redigere enkeltattributter direkte i Exchange Online.

## Årsak 5: Videresendinger og transportregler danner en sirkel

En regel kan omdirigere fra adresse A til B, mens B sender tilbake til A via en annen regel, en postboksvideresending eller et eksternt system. Slike sløyfer berører ofte bare enkelte mottakere eller meldingstyper.

```powershell
Get-TransportRule |
    Sort-Object Priority |
    Select-Object Name,State,Mode,Priority,RedirectMessageTo,
        BlindCopyTo,AddToRecipients,RouteMessageOutboundConnector

Get-Mailbox -ResultSize Unlimited |
    Select-Object DisplayName,PrimarySmtpAddress,
        ForwardingAddress,ForwardingSmtpAddress,DeliverToMailboxAndForward

Get-InboxRule -Mailbox user01@contoso.com |
    Select-Object Name,Enabled,Priority,ForwardTo,RedirectTo,ForwardAsAttachmentTo
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `Sort-Object Priority` | Sorterer transportreglene i evalueringsrekkefølgen deres |
| `-ResultSize Unlimited` | Opphever standardgrensen på 1000 returnerte postbokser |
| `-Mailbox user01@contoso.com` | Postboks der innboksreglene hentes ut |
| `Select-Object …` | Reduserer utdataene til målene for videresending og omdirigering |

</details>

Løsningen består ikke bare i å slå av en regel midlertidig. Hele kjeden må løses opp, og regler for eksterne tjenester trenger et unntak som pålitelig gjenkjenner allerede behandlede meldinger.

## Årsak 6: MX, Smart Host eller underdomene peker tilbake

En gateway kan internt trenge et annet neste hopp enn eksterne avsendere. Hvis den bruker den offentlige MX-en for videresending, kan denne under visse omstendigheter peke tilbake på selve gatewayen. Det samme problemet oppstår dersom en Smart Host føres tilbake til sin egen lytter via NAT eller lastbalansering.

```powershell
Resolve-DnsName -Type MX contoso.com
Resolve-DnsName -Type MX app.contoso.com

Get-SendConnector |
    Format-List Name,AddressSpaces,DNSRoutingEnabled,SmartHosts
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `-Type MX` | Spør etter MX-poster i stedet for standard A-poster |
| `contoso.com` / `app.contoso.com` | Domenet som skal spørres etter som posisjonsargument (parameter `-Name`) |
| `Format-List …` | Viser adresserom, rutingmodus og Smart Hosts per Send Connector |

</details>

Underdomener fortjener en egen kontroll. Microsoft dokumenterer tilfeller der et applikasjonsunderdomene må opprettes eksplisitt som et Internal Relay-domene og synkroniseres til Edge-systemene:

```powershell
New-AcceptedDomain `
    -Name "app.contoso.com" `
    -DomainName app.contoso.com `
    -DomainType InternalRelay

Start-EdgeSynchronization
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `-Name "app.contoso.com"` | Visningsnavn for det nye Accepted Domain-objektet |
| `-DomainName app.contoso.com` | SMTP-domenet som Exchange godtar meldinger for |
| `-DomainType InternalRelay` | En del av mottakerne befinner seg utenfor organisasjonen; ukjente mottakere videresendes via en Send Connector i stedet for å avvises |

</details>

Disse kommandoene er ingen universell løsning. De passer bare dersom `app.contoso.com` faktisk leveres utenfor Exchange-organisasjonen og Send Connector har et entydig neste hopp.

## Sikker fremgangsmåte ved en aktiv sløyfe

Under driftsforstyrrelsen bør først oppformeringen stanses. Avhengig av arkitekturen deaktiveres den utløsende transportregelen eller den spesifikke koblingen kontrollert, eller gatewayen holder den berørte køen tilbake. Før dette eksporteres konfigurasjon og meldingseksempler.

Deretter følger en test med nøyaktig én avsender, én mottaker og en tydelig gjenkjennelig emnelinje. Meldingen følges kontinuerlig gjennom headere, Message Trace og lokale sporingslogger. Først når den ender på det planlagte målet, åpnes e-postflyten trinnvis igjen.

Ikke anbefalt er:

- å øke hop-grenser
- å endre flere koblinger samtidig
- å bytte Accepted Domains på mistanke mellom `Authoritative` og `InternalRelay`
- å legge en problematisk kø inn igjen gjentatte ganger uten kontroll
- å korrigere synkroniserte Exchange-attributter direkte i AD eller Exchange Online
- å deaktivere TLS-, IP- eller sertifikatkontroller som en tilsynelatende hurtigløsning

## Sluttkontroll

Etter korrigeringen bør dokumentasjonen inneholde nøyaktig én uttalelse for hvert relevant domene: Hvilket system kjenner mottakeren, hvilken kobling gjelder, og hvilken vert er det endelige neste hoppet?

Den tekniske godkjenningen omfatter minst:

- ekstern og intern testmelding
- ukjent mottaker i samme domene
- mottaker på hver side av et reelt Split-Domain
- utgående melding med aktivert gateway eller Centralized Mail Transport
- headere uten gjentakende hoppsekvens
- Message Trace med `Delivered` eller forventet overlevering
- lokal sporing uten ny `RECEIVE` etter en `SEND` til samme mål
- koblingsvalidering for alle koblinger som fortsatt trengs

En løst e-postsløyfe er først avsluttet når ikke bare test-e-posten kommer frem, men også ukjente mottakere og alternative e-postflytstier avsluttes definert. Det er nettopp der de fleste tilbakefallene ligger.

## Kilder

1. [Microsoft Learn – Fix NDR error 5.4.6 or 5.4.14 in Exchange Online](https://learn.microsoft.com/en-us/troubleshoot/exchange/email-delivery/ndr/fix-error-code-5-4-6-through-5-4-20-in-exchange-online): Betydningen av Exchange-NDR-er og typiske årsaker i Accepted Domains og hybridkoblinger.

2. [Microsoft Learn – Manage accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains): Forskjeller mellom `Authoritative` og `InternalRelay`.

3. [Microsoft Learn – Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains): Ansvar, relay-domener og Recipient Lookup i lokal Exchange.

4. [Microsoft Learn – Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing): Forventede transportveier med og uten Centralized Mail Transport.

5. [Microsoft Learn – Validate connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/validate-connectors): Koblingsvalidering og merknader om flere koblinger som passer samtidig.

6. [Microsoft Learn – Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud): Støttede e-postflytmønstre med foranliggende tredjepartstjenester.

7. [Microsoft Learn – Mail flow rules in Exchange Online](https://learn.microsoft.com/en-us/exchange/security-and-compliance/mail-flow-rules/mail-flow-rules): Behandling, prioritet, handlinger og unntak for transportregler.

8. [Microsoft Learn – Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2?view=exchange-ps): Søk etter meldinger i Exchange Online-transporten.

9. [Microsoft Learn – Search message tracking logs](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/search-message-tracking-logs): Lokal meldingssporing på tvers av alle Exchange-servere.

10. [Microsoft Learn – Hop count exceeded for an on-premises application subdomain](https://learn.microsoft.com/en-us/troubleshoot/exchange/mailflow/hop-count-exceeded-possible-mail-loop): Dokumentert scenario for underdomene/EdgeSync med eksplisitt Internal Relay-domene.
