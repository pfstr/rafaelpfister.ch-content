---
title: "Vanlige årsaker til e-postsløyfer og hvordan de løses"
navTitle: "Løse e-postsløyfer"
description: "Slik kan SMTP-e-postsløyfer i Exchange Online, hybridmiljøer og forankoblede e-postgatewayer identifiseres og løses systematisk ved hjelp av NDR, headere, Message Trace, mottakerobjekter og connectorer."
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
url: https://rafaelpfister.ch/no/blog/vanlige-arsaker-til-e-postsloyfer-og-hvordan-de-loses
translationSourceHash: 5353684681217adafc789a3b28ec218fa927e18d801c82c437ae281e1e1017bd
translationModel: gpt-5.6-terra
translatedAt: 2026-08-08T14:01:43.874Z
translationReview: automatic
---

# Vanlige årsaker til e-postsløyfer og hvordan de løses

En e-postsløyfe oppstår når minst to transportsystemer stadig overleverer den samme meldingen til hverandre. Ingen av systemene oppfatter seg selv som det endelige målet, men begge kjenner et angivelig passende neste hopp. Sløyfen avsluttes først når en server registrerer at tillatt antall transportstasjoner er overskredet og oppretter en NDR.

I Exchange er to meldinger spesielt informative:

- `554 5.4.6 Hop count exceeded - possible mail loop` opprettes vanligvis av lokal Exchange.
- `554 5.4.14 Hop count exceeded - possible mail loop ATTR34` opprettes av Exchange Online.

Hop-grensen er ikke årsaken, men sikringen mot en uendelig gjentakelse. Å øke den løser derfor ingenting. Det som må finnes, er punktet der meldingen, i strid med målarkitekturen, returneres til et system den allerede har passert.

## Gjenkjenne sløyfemønsteret i headeren

NDR-en og de fullstendige opprinnelige meldingsheaderne bør sikres før enhver endring. `Received`-linjer leses nedenfra og opp: Den nederste linjen er det tidligste dokumenterte hoppet, den øverste er det nyeste.

En sløyfe viser seg som regel som en gjentakende sekvens:

```text
Internet
  → Exchange Online Protection
  → Mailgateway
  → Exchange Online Protection
  → Mailgateway
  → Exchange Online Protection
  → ...
```

Ikke alle Microsoft-vertsnavn som forekommer flere ganger, er allerede en sløyfe. Exchange Online behandler meldinger internt gjennom flere transportroller. Det mistenkelige er gjentatt retur mellom de samme administrative grensene, for eksempel mellom Exchange Online og en lokal gateway. Tidsstempler, avsender-IP, mottakende vert og `Message-ID` bidrar til å identifisere runden entydig.

For den første analysen besvares disse spørsmålene:

1. Hvilket system opprettet NDR-en?
2. Hvilke to eller tre hopp gjentar seg?
3. Hvilket system skulle ha levert meldingen endelig?
4. På grunnlag av hvilken domene-, mottaker-, connector- eller regelbeslutning ble den videresendt?
5. Hvilken endring påvirket sist e-postflyten?

## Diagnose i Exchange Online

Med `Get-MessageTraceV2` kan behandlingen de siste 90 dagene undersøkes; maksimalt ti dager er tillatt per spørring. Et smalt tidsvindu og den konkrete mottakeradressen gir de mest brukbare resultatene:

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

Detaljene for et treff viser individuelle transporthendelser:

```powershell
$trace | ForEach-Object {
    Get-MessageTraceDetailV2 `
        -MessageTraceId $_.MessageTraceId `
        -RecipientAddress $_.RecipientAddress
} | Format-Table Date,Event,Action,Detail -AutoSize
```

Deretter hentes domene, mottaker og connectorer sammen:

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

Det avgjørende er ikke om et enkelt objekt ser plausibelt ut. Domenetype, faktisk mottakertype og gjeldende connector må sammen beskrive samme målsted.

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

For transportbanen trengs send- og receive-connectorer samt tracking-loggene:

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

En `SEND` til Exchange Online, etterfulgt av et nytt `RECEIVE` av samme melding fra Exchange Online, synliggjør returen. Med `MessageId` og `NetworkMessageId` unngår du å forveksle ulike testmeldinger.

## De vanligste årsakene i oversikt

| Mønster | Typisk årsak | Løsning |
| --- | --- | --- |
| Ukjente mottakere pendler mellom to systemer | Accepted Domain er satt til `InternalRelay`, men begge sider videresender ukjente mottakere | Definer et tydelig ansvar; bruk `Authoritative` ved fullstendig EXO-levering, eller definer ett enkelt avsluttende hopp for Split Domain |
| EXO sender til lokal Exchange, som umiddelbart returnerer til EXO | Hybrid-connector eller Centralized Mail Transport passer ikke lenger med postboksplasseringen | Kontroller HCW-konfigurasjonen og `RouteAllMessagesViaOnPremises`; deaktiver foreldet sentral rute eller korriger lokal mottakeroppløsning |
| Meldingen pendler mellom EXO og en sikkerhets-, signatur- eller krypteringsgateway | Returnerte meldinger oppfyller utgående regel på nytt | Bruk en header satt av gatewayen eller en dokumentert loop-prevention-mekanisme som unntak; autentiser inn- og utgående connectorer entydig |
| Bare én mottaker er berørt | Utdatert eller feil `targetAddress`, feil RemoteMailbox-type eller motstridende proxy-adresser | Fastslå Source of Authority, korriger mottakerobjektet der og synkroniser |
| Bare videresendte meldinger går i sløyfe | Transportregel, postboksvideresending eller Inbox-regel adresserer den opprinnelige banen på nytt | Deaktiver regelen, korriger målet og definer et robust unntak |
| Bare ett subdomene eller én applikasjon er berørt | Overordnet domene dekker ikke subdomenet korrekt i den forventede connectorbanen | Konfigurer subdomenet eksplisitt som Accepted Domain og i riktig Send Connector |
| Alle meldinger går i sløyfe etter en gateway- eller DNS-endring | Smart Host eller MX peker mot inngangen til det sendende systemet | Korriger Next Hop og kontroller DNS-, NAT- og load balancer-mål hver for seg |

## Årsak 1: Feil type Accepted Domain

Et authoritative domene betyr: Alle gyldige mottakere i dette domenet er kjent i Exchange-organisasjonen; ukjente mottakere avvises. Et Internal Relay-domene betyr: En del av mottakerne befinner seg i et annet system og må videresendes via en Send- eller Outbound-connector.

Den problematiske konfigurasjonen oppstår når Exchange Online sender ukjente mottakere til et lokalt system, og dette systemet heller ikke behandler det samme domenet endelig, men returnerer det til Exchange Online via MX eller Smart Host.

```powershell
Get-AcceptedDomain -Identity contoso.com |
    Format-List DomainName,DomainType,MatchSubDomains
```

Hvis alle mottakere befinner seg i Exchange Online etter at en migrering er fullført, er `Authoritative` som regel riktig måltilstand:

```powershell
# Kjør først etter fullstendig kontroll av mottakere og ruting.
Set-AcceptedDomain -Identity contoso.com -DomainType Authoritative
```

For et reelt Split Domain kan `InternalRelay` være korrekt. Da kreves det imidlertid en tydelig connector til systemet som kjenner de gjenværende mottakerne. Dette målet må ikke sende ukjente adresser tilbake til utgangspunktet.

## Årsak 2: Overlappende hybrid-connectorer og Centralized Mail Transport

Centralized Mail Transport leder utgående Exchange Online-meldinger bevisst via lokal Exchange. Dette er hensiktsmessig for bestemte compliance-krav, men skaper ekstra transportveier. Hvis alternativet forblir aktivt etter en migrering, selv om det lokale systemet sender meldinger tilbake til Exchange Online via sin egen MX, kan det oppstå en sirkel.

```powershell
Get-OutboundConnector -IncludeTestModeConnectors |
    Format-Table Name,Enabled,ConnectorType,RouteAllMessagesViaOnPremises,
        RecipientDomains,SmartHosts,UseMXRecord -AutoSize
```

Flere connectorer med overlappende omfang er også mistenkelige. Microsoft anbefaler en dedikert On-Premises-connector for hybrid e-postflyt; reparasjon gjennom Hybrid Configuration Wizard er ofte sikrere enn isolerte enkeltendringer.

Hvis Centralized Mail Transport dokumentert ikke lenger er nødvendig, kan innstillingen deaktiveres målrettet:

```powershell
# Kun etter kontroll av compliance- og gateway-krav.
Set-OutboundConnector `
    -Identity "Outbound to On-Premises" `
    -RouteAllMessagesViaOnPremises:$false
```

## Årsak 3: En gateway behandler returmeldinger på nytt

I et in-and-out-scenario sender Exchange Online en melding til en tilleggstjeneste for signering, kryptering eller arkivering. Denne returnerer deretter meldingen til Exchange Online. Den utgående regelen må gjenkjenne returmeldingen; ellers sendes den til tjenesten på nytt.

Kontrollen begynner med alle regler som velger connectorer, omdirigerer mottakere eller evaluerer headere:

```powershell
Get-TransportRule |
    Sort-Object Priority |
    Format-List Name,State,Mode,Priority,FromScope,SentToScope,
        RedirectMessageTo,RouteMessageOutboundConnector,
        SetHeaderName,SetHeaderValue,ExceptIfHeaderContainsMessageHeader,
        ExceptIfHeaderContainsWords
```

Det konkrete unntaket må følge dokumentasjonen fra gateway-produsenten. Vanlig er en header som settes av tjenesten og som ikke kan forfalskes pålitelig av Internett. I tillegg bør inbound-connectorer identifisere tjenesten via sertifikat eller fast avsender-IP. Et generelt unntak for alle meldinger som fremstår som «interne», er for omfattende.

## Årsak 4: Mottakerobjekt og faktisk postboks befinner seg ikke på samme sted

Et objekt kan vises i Exchange Online som `MailUser`, selv om den aktive postboksen befinner seg lokalt. I et synkronisert hybridmiljø er dette ikke automatisk et duplikat. Heller ikke en `ExternalEmailAddress`, som tilsvarer den primære SMTP-adressen, beviser i seg selv en feilkonfigurasjon.

Det avgjørende er kombinasjonen av alle spørringene:

- `Get-Mailbox` lokalt gir et resultat: Den aktive postboksen befinner seg lokalt.
- `Get-RemoteMailbox` lokalt gir et resultat: Det administrerte målet befinner seg i Exchange Online.
- `Get-EXOMailbox` gir et resultat: Det finnes en ekte postboks i skyen.
- `Get-EXORecipient` gir bare en MailUser: Objektet er et rutingsmål, ikke en skylagringspostboks.

Utdaterte objekter etter en migrering, feil Remote Routing-domener eller manuelt angitte `targetAddress`-verdier der domenet leder tilbake via samme transportbane, er problematiske. Endringer gjøres ved Source of Authority: I synkroniserte miljøer altså med Exchange-administrasjonsverktøy lokalt og ikke ved direkte redigering av enkeltattributter i Exchange Online.

## Årsak 5: Videresendinger og transportregler danner en sirkel

En regel kan omdirigere fra adresse A til B, mens B sender tilbake til A via en annen regel, en postboksvideresending eller et eksternt system. Slike sløyfer rammer ofte bare enkelte mottakere eller meldingstyper.

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

Løsningen består ikke bare i å deaktivere en regel midlertidig. Hele kjeden må løses opp, og regler for eksterne tjenester trenger et unntak som sikkert gjenkjenner allerede behandlede meldinger.

## Årsak 6: MX, Smart Host eller subdomene peker tilbake

En gateway kan internt trenge et annet Next Hop enn eksterne avsendere. Hvis den bare bruker offentlig MX for videresending, kan dette i enkelte tilfeller peke tilbake på selve gatewayen. Det samme problemet oppstår når en Smart Host leder tilbake til sin egen listener gjennom NAT eller load balancing.

```powershell
Resolve-DnsName -Type MX contoso.com
Resolve-DnsName -Type MX app.contoso.com

Get-SendConnector |
    Format-List Name,AddressSpaces,DNSRoutingEnabled,SmartHosts
```

Subdomener fortjener en egen kontroll. Microsoft dokumenterer tilfeller der et applikasjonssubdomene eksplisitt må opprettes som Internal Relay-domene og synkroniseres til Edge-systemene:

```powershell
New-AcceptedDomain `
    -Name "app.contoso.com" `
    -DomainName app.contoso.com `
    -DomainType InternalRelay

Start-EdgeSynchronization
```

Disse kommandoene er ingen universell løsning. De passer bare når `app.contoso.com` faktisk leveres utenfor Exchange-organisasjonen og Send Connector har et entydig neste hopp.

## Sikker fremgangsmåte ved en aktiv sløyfe

Under forstyrrelsen bør spredningen først stanses. Avhengig av arkitekturen deaktiveres den utløsende transportregelen eller den spesifikke connectoren kontrollert, eller gatewayen holder den berørte køen tilbake. Konfigurasjon og meldingseksempler eksporteres på forhånd.

Deretter følger en test med nøyaktig én avsender, én mottaker og en tydelig gjenkjennelig emnelinje. Meldingen spores uten avbrudd via headere, Message Trace og lokale tracking-logger. Først når den ender på det planlagte målet, åpnes e-postflyten gradvis igjen.

Ikke anbefalt er:

- å øke hop-grenser
- å endre flere connectorer samtidig
- å bytte Accepted Domains på mistanke mellom `Authoritative` og `InternalRelay`
- å mate inn en problematisk kø på nytt gjentatte ganger uten kontroll
- å korrigere synkroniserte Exchange-attributter direkte i AD eller Exchange Online
- å deaktivere TLS-, IP- eller sertifikatkontroller som en angivelig hurtigfiks

## Sluttkontroll

Etter korrigeringen bør dokumentasjonen inneholde nøyaktig én opplysning for hvert relevant domene: Hvilket system kjenner mottakeren, hvilken connector gjelder, og hvilken vert er det endelige neste hoppet?

Den tekniske godkjenningen omfatter minst:

- ekstern og intern testmelding
- ukjent mottaker i samme domene
- mottaker på hver side av et reelt Split Domain
- utgående melding med aktiv gateway eller Centralized Mail Transport
- headere uten gjentakende hoppsekvens
- Message Trace med `Delivered` eller forventet overlevering
- lokal tracking uten nytt `RECEIVE` etter et `SEND` til samme mål
- connector-validering for alle connectorer som fortsatt er nødvendige

En løst e-postsløyfe er først ferdig håndtert når ikke bare test-e-posten kommer frem, men også ukjente mottakere og alternative e-postflytbaner avsluttes som definert. Det er nettopp her de fleste tilbakefall oppstår.

## Kilder

1. [Microsoft Learn – Fix NDR error 5.4.6 or 5.4.14 in Exchange Online](https://learn.microsoft.com/en-us/troubleshoot/exchange/email-delivery/ndr/fix-error-code-5-4-6-through-5-4-20-in-exchange-online): Betydningen av Exchange-NDR-er og typiske årsaker i Accepted Domains og hybrid-connectorer.

2. [Microsoft Learn – Manage accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains): Forskjeller mellom `Authoritative` og `InternalRelay`.

3. [Microsoft Learn – Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains): Ansvar, relay-domener og Recipient Lookup i lokal Exchange.

4. [Microsoft Learn – Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing): Forventede transportveier med og uten Centralized Mail Transport.

5. [Microsoft Learn – Validate connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/validate-connectors): Connector-validering og merknader om flere connectorer som passer samtidig.

6. [Microsoft Learn – Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud): Støttede e-postflytmønstre med forankoblede tredjepartstjenester.

7. [Microsoft Learn – Mail flow rules in Exchange Online](https://learn.microsoft.com/en-us/exchange/security-and-compliance/mail-flow-rules/mail-flow-rules): Behandling, prioritet, handlinger og unntak for transportregler.

8. [Microsoft Learn – Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2?view=exchange-ps): Søk etter meldinger i Exchange Online-transporten.

9. [Microsoft Learn – Search message tracking logs](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/search-message-tracking-logs): Lokal meldingssporing på tvers av alle Exchange-servere.

10. [Microsoft Learn – Hop count exceeded for an on-premises application subdomain](https://learn.microsoft.com/en-us/troubleshoot/exchange/mailflow/hop-count-exceeded-possible-mail-loop): Dokumentert subdomene-/EdgeSync-scenario med eksplisitt Internal Relay-domene.
