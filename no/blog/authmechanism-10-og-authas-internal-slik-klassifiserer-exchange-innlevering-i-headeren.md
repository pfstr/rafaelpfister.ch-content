---
title: "AuthMechanism 10 og AuthAs Internal: Slik klassifiserer Exchange innlevering i headeren"
navTitle: "AuthMechanism 10"
description: "Headeren X-MS-Exchange-Organization-AuthMechanism dokumenterer hvordan en innleverende server har autentisert seg. Verdien 10 står for en Receive Connector med Externally Secured og klassifiserer eksterne e-poster som interne: med konsekvenser for spamfiltre, e-postflytregler og beskyttelse mot spoofing."
date: "2026-08-26"
featured: "2026-08-27"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "8 min lesetid"
themen:
  - exchange-onprem-hybrid
  - microsoft-365-exchange
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange"
  - "exchange-hybrid"
protokolle:
  - "smtp"
  - "troubleshooting"
related:
  - exchange-hybrid-header-intern-extern
  - exchange-message-tracking-und-receive-connectoren-analysieren
  - typische-ursachen-fuer-mail-loops-und-deren-behebung
slug: "authmechanism-10-og-authas-internal-slik-klassifiserer-exchange-innlevering-i-headeren"
translationId: "article-0df383d5c49016da"
translationOf: exchange-authmechanism-10-authas-internal
url: https://rafaelpfister.ch/no/blog/authmechanism-10-og-authas-internal-slik-klassifiserer-exchange-innlevering-i-headeren
translationSourceHash: 5a9335a90afc9bf7df78b908f71b679f64c29f3b9e96bd7f25bcc916123b82df
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:18:34.838Z
translationReview: automatic
---

# AuthMechanism 10 og AuthAs Internal: Slik klassifiserer Exchange innlevering i headeren

Ved analyse av spam-, spoofing- og e-postflytsaker i Exchange-miljøer er tre headerfelt avgjørende, som Exchange setter ved mottak:

```text
X-MS-Exchange-Organization-AuthAs: Internal
X-MS-Exchange-Organization-AuthSource: EX01.example.com
X-MS-Exchange-Organization-AuthMechanism: 10
```

`AuthAs` angir hvordan avsenderen fremstod overfor transporten. `AuthSource` oppgir serveren som foretok vurderingen. `AuthMechanism` dokumenterer hvilken mekanisme autentiseringen skjedde gjennom. Sammen avgjør de om Exchange behandler en melding som intern eller ekstern, og denne klassifiseringen har betydelige konsekvenser.

## Hvorfor klassifiseringen er viktig

`AuthAs` har i praksis to verdier: `Internal` og `Anonymous`. En melding klassifisert som `Internal` behandles annerledes enn ekstern e-post:

- E-postflytregler med betingelsen «avsender utenfor organisasjonen» gjelder ikke.
- Meldingen kan leveres til distribusjonsgrupper og postbokser som krever autentiserte avsendere (`RequireSenderAuthenticationEnabled`).
- Anti-spam- og anti-spoofing-kontroller blir mindre strenge eller utelates; i hybridmiljøer legges ikke den eksterne ansvarsfraskrivelsen til, og Outlook viser ingen «Ekstern»-indikasjon.
- Visningsnavnet løses opp fra adresseboken, og e-posten fremstår som intern post for mottakerne.

Derfor bør spørsmålet «AuthAs Internal eller Anonymous?» være først i enhver headeranalyse: Det gjør det mulig å avklare hvorfor en åpenbar spoofing-e-post passerte spamfilteret, eller hvorfor en e-postflytregel aldri ble utløst.

## AuthMechanism-verdiene

Microsoft dokumenterer ikke kodingen av `AuthMechanism` fullstendig offentlig. To verdier er relevante og godt dokumentert for feilsøking:

| Verdi | Betydning |
|---|---|
| `04` | Autentisert Exchange-trafikk: postboks til postboks innenfor organisasjonen samt hybridtrafikk via connectorene som er opprettet av Hybrid Configuration Wizard. |
| `10` | Receive Connector med autentiseringsalternativet `ExternalAuthoritative` («eksternt sikret» / «Externally secured»): Forbindelsen anses som sikret utenfor Exchange, og alt som leveres gjennom den, behandles som internt. |

Andre verdier forekommer i headere, men uten offisiell referanse. I praksis er skillet tilstrekkelig: `04` betyr faktisk Exchange-autentisering, `10` betyr tillit basert på connector-konfigurasjon.

## Hva Externally Secured faktisk betyr

Alternativet `ExternalAuthoritative` på en Receive Connector forteller Exchange: Noen andre sørger for sikringen av denne forbindelsen, for eksempel en brannmur, et dedikert nettverkssegment eller IPsec. Exchange kontrollerer da ikke lenger noe, men behandler enhver innlevering via denne connectoren som autentisert og intern, inkludert retten til å bruke vilkårlige interne avsenderadresser.

Dette er ment for få scenarioer, for eksempel en fullt ut pålitelig applikasjonsserver i eget datasenter. Det blir problematisk dersom connectoren peker mot en foranliggende e-postgateway eller et spamfilter i DMZ-en, som også mottar internettpost. Da får hver ekstern e-post etter innleveringen:

```text
X-MS-Exchange-Organization-AuthAs: Internal
X-MS-Exchange-Organization-AuthMechanism: 10
```

Konsekvensene er at eksterne e-poster anses som interne, e-postflytregler for eksterne avsendere ikke gjelder, spoofing-beskyttelsen for eget domene er virkningsløs, og enhver som når gatewayen, kan levere med interne avsenderadresser til mottakere som egentlig krever autentiserte avsendere.

## Finn berørte connectorer

Exchange Management Shell viser hvilke Receive Connectorer som er konfigurert med `ExternalAuthoritative`:

```powershell
Get-ReceiveConnector | Where-Object {
  $_.AuthMechanism -match "ExternalAuthoritative"
} | Format-Table Identity, RemoteIPRanges, AuthMechanism, PermissionGroups
```

Kontroller for hvert treff hvilke `RemoteIPRanges` som er angitt, og om systemene bak faktisk trenger denne tilliten. En gateway som bare skal videresende e-post, trenger den ikke.

## Alternativet for relay-scenarioer

Dersom et system bare skal relayere anonymt via Exchange (skrivere, applikasjoner, overvåking), er en anonym relay-connector en ryddigere løsning: anonym innlevering pluss retten til å levere til vilkårlige mottakere, men uten Internal-klassifisering.

```powershell
New-ReceiveConnector -Name "Anonymous Relay" -TransportRole FrontendTransport `
  -RemoteIPRanges 192.0.2.10 -Bindings 0.0.0.0:25 -Usage Custom -PermissionGroups AnonymousUsers

Get-ReceiveConnector "EX01\Anonymous Relay" | Add-ADPermission `
  -User "NT AUTHORITY\ANONYMOUS LOGON" -ExtendedRights "ms-Exch-SMTP-Accept-Any-Recipient"
```

E-poster via denne connectoren forblir `AuthAs: Anonymous`, gjennomgår de vanlige kontrollene og kan ikke utgi seg for å ha interne avsendere. `ExternalAuthoritative` er forbeholdt systemene du bevisst vil gi rett til å bruke interne avsenderadresser.

## Les headeren i sammenheng

Om en konkret melding ble klassifisert som intern eller ekstern, og hvilken vei den kom via, kan raskest leses av i den fullstendige headeren: `AuthAs`, `AuthMechanism` og `AuthSource` sammen med `Received`-kjeden. [Mail Header Analyzer](/tools/header-analyzer) på dette nettstedet analyserer disse feltene direkte i nettleseren og markerer hybridklassifiseringen i leveringsveien; headeren forlater ikke nettleseren.

Artikkelen [Intern eller ekstern? Klassifisering av Exchange-hybrid-e-post i headeren](/blog/exchange-hybrid-header-intern-extern) beskriver hvordan klassifiseringen bevares mellom Exchange Online og OnPrem i hybridmiljøer, og hvordan feil klassifisering kan gjenkjennes.

## Kilder

1.  [Microsoft Q&A: Exchange 2016 mail flow rule, which header is checked for "outside the organization"?](https://learn.microsoft.com/en-us/answers/questions/54418/exchange-2016-mail-flow-rule-which-header-is-check): Tilordning av AuthAs og AuthMechanism 10 til Externally-Secured-konfigurasjonen og dens virkning på e-postflytregler.

2.  [Microsoft Tech Community: Demystifying and troubleshooting hybrid mail flow: when is a message internal?](https://techcommunity.microsoft.com/blog/exchange/demystifying-and-troubleshooting-hybrid-mail-flow-when-is-a-message-internal/1420838): Offisiell beskrivelse av Internal-klassifiseringen og konsekvensene i hybrid e-postflyt.

3.  [msxfaq: X-MS-Exchange-Organization-AuthAs](https://www.msxfaq.de/cloud/exchangeonline/transport/x-ms-exchange-organization-authas.htm): Observerte AuthAs-, AuthSource- og AuthMechanism-verdier i ulike innleveringsscenarioer.

4.  [Microsoft Learn: Allow anonymous relay on Exchange servers](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/allow-anonymous-relay): Oppsett av den anonyme relay-connectoren som alternativ til Externally Secured.
