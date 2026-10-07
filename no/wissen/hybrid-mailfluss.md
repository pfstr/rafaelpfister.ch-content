---
title: "Hybrid e-postflyt: ruting mellom Exchange Online og lokalt miljø"
blatt: "hybrid-mailfluss"
description: "Hybrid e-postflyt trinn for trinn: direkte og sentralisert levering, connectorer, TLS-sertifikater, eksterne domener, delte SMTP-domener, e-postgatewayer, Message Trace og Message Tracking."
fakten:
  - label: Oppgave
    wert: SMTP-ruting mellom Exchange Online og lokal Exchange-organisasjon
    href: https://learn.microsoft.com/en-us/exchange/transport-routing
  - label: Oppsett
    wert: Hybrid Configuration Wizard oppretter og vedlikeholder transportkonfigurasjonen
    href: https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard
  - label: Transport
    wert: SMTP over TCP 25 med TLS
    href: https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites
  - label: Kontroll av motpart
    wert: Sertifikatnavn og connectorbetingelser
    href: https://learn.microsoft.com/en-us/exchange/hybrid-deployment/hybrid-mail-flow
  - label: Cloudconnectorer
    wert: Inbound- og Outbound Connectors
    href: https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail
  - label: Lokale connectorer
    wert: Receive- og Send Connectors
    href: https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors
  - label: Mottakerruting
    wert: Remote Mailbox, Target Address og sameksistensdomene
    href: https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox
  - label: Standardruting
    wert: Cloud- og lokal organisasjon kan hver for seg sende internett-e-post direkte
    href: https://learn.microsoft.com/en-us/exchange/transport-routing
  - label: Centralized Mail Transport
    wert: Internett-e-post fra Exchange Online går via den lokale organisasjonen
    href: https://learn.microsoft.com/en-us/exchange/transport-routing
  - label: Eksterne gatewayer
    wert: Ekstra connector- og filterkjede foran eller bak Microsoft 365
    href: https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud
  - label: Cloud-diagnose
    wert: Message Trace og connectorvalidering
    href: https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message
  - label: Lokal diagnose
    wert: Message Tracking, køer og protokollogger
    href: https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking
werbung:
  - tools
  - newsletter
ctaThemen:
  - exchange-onprem-hybrid
  - smtp-mailflow
translationSourceHash: 442ec532c64e034403d3449ffe1b46d4b6de05a2337d3c3210da2d83cb5f618f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-07T11:06:02.366Z
translationReview: automatic
---

# Hybrid e-postflyt: ruting mellom Exchange Online og lokalt miljø

**Hybrid e-postflyt** er SMTP-veien mellom en lokal Exchange-organisasjon og Exchange Online. Den sørger for at postbokser på begge sider kan bruke samme SMTP-domene, samtidig som meldinger kommer til riktig sted. Hybrid Configuration Wizard konfigurerer connectorer og TLS-parametere for dette ([Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing), [Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)).

E-postflyt er bare én del av Exchange Hybrid. Katalogsynkronisering, ledig/opptatt, OAuth og postboksflyttinger bruker andre veier. Denne artikkelen holder seg derfor bevisst til ett enkelt spørsmål: **Hvilke SMTP-hopp går en konkret melding gjennom, og hvilken beslutning tas ved hvert hopp?**

**Protokollstakken** er oversiktlig: DNS angir de offentlig tilgjengelige målene, SMTP over TCP 25 transporterer meldingen, TLS beskytter og identifiserer forbindelsen, og Exchange-connectorer fastsetter hvilken motpart som brukes for hvert domene. Mottakerobjekter leverer rutingsadressen; Message Trace og lokale sporingslogger viser deretter hva hver organisasjon gjorde med meldingen ([Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing), [Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)).

## Grunnmodellen: to Exchange-organisasjoner, ett adresserom

Et hybridmiljø har minst to transportorganisasjoner. Den lokale Exchange-organisasjonen kjenner lokale postbokser og eksterne postboksobjekter. Exchange Online kjenner skypostbokser og synkroniserte representasjoner av lokale mottakere. Begge sider kan bruke samme primære domene som `example.com` ([Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)).

For at en melding ikke skal ende på feil sted, trenger hver side en indikasjon på mottakerens faktiske plassering. For en skypostboks inneholder det lokale Remote Mailbox-objektet en ekstern rutingsadresse i sameksistensdomenet, vanligvis `tenant.mail.onmicrosoft.com`. Omvendt kjenner Exchange Online synkroniserte lokale mottakere som e-postaktiverte objekter ([Enable-RemoteMailbox](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox)).

Den normale prosessen er dermed enkel:

1. Den første Exchange-organisasjonen mottar meldingen.
2. Den slår opp mottakeren i katalogen sin.
3. Mottakerobjektet viser om postboksen er lokal eller ligger på den andre siden.
4. Den riktige hybridconnectoren sender via SMTP/TLS til den andre organisasjonen.
5. Der slås mottakeren opp på nytt, og meldingen leveres.

Eksperter kontrollerer i tillegg om transportregler endrer ruten, om en gateway er satt inn, og hvilken domene- eller connectorprioritet som forklarer valgt neste hopp.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-hybrid-mailfluss.svg?v=20260813" title="Interaktive Infografik: direkter und zentraler Hybrid-Mailfluss zwischen Internet, Exchange Online, Exchange On-Premises und Mail-Gateway" loading="lazy">
  <a href="/images/kb-interaktiv-hybrid-mailfluss.svg?v=20260813">Åpne den interaktive grafikken for hybrid e-postflyt direkte</a>.
</iframe>

## Standardruten uten sentralisert internetttransport

I den vanlige desentraliserte modellen sender hver side sin egen internett-e-post. En lokal postboks bruker den lokale Exchange-transportorganisasjonen for utgående internett-e-post. En skypostboks sender via Exchange Online Protection. Bare meldinger mellom lokale postbokser og skypostbokser går gjennom hybridconnectorene ([Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing)).

Innkommende internett-e-post følger den publiserte MX-posten. Hvis MX-posten peker til Exchange Online, mottar EOP meldingen først. For en skypostboks leverer Exchange Online lokalt; for en synkronisert lokal mottaker bruker den Hybrid Outbound Connector. Hvis MX-posten derimot peker til det lokale miljøet eller en foranstilt gateway, tas den første mottakerbeslutningen der.

Denne modellen holder internettveiene korte, men fører til flere mulige egress-IP-er og filtreringssteder. SPF, DKIM, DMARC, tillatlisting og partnerregler må ta hensyn til begge utgående veier. Dette er ikke en feil ved hybridmodellen, men en konsekvens av distribuert levering.

## Sett Centralized Mail Transport i riktig sammenheng

**Centralized Mail Transport**, CMT, endrer nettopp denne utgående veien. Meldinger fra Exchange Online-postbokser til internett sendes først til den lokale Exchange-organisasjonen. Først der forlater de organisasjonen. Den lokale siden kan dermed fortsatt bruke sentrale transportregler, løsninger eller faste egress-IP-er ([Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing)).

Fordelen er felles kontroll med utgående trafikk. Prisen er ekstra hopp og avhengigheter. Hvis lokal transport eller internettforbindelsen svikter, påvirker det nå også utgående sky-e-post. Ventetid, plassering av køen, egress-IP og stedet for siste filtrering endres.

For avanserte administratorer er beslutningen derfor ikke «CMT på eller av», men: Hvilken konkret policy krever det lokale hoppet, hvilken kapasitet må det håndtere, og hvordan rutes det ved feil? Eksperter dokumenterer i tillegg løkkebeskyttelse, TLS-krav, connectorprioritet og beviset på at hver tiltenkte melding faktisk tar den sentrale veien.

Dermed er temaet CMT avsluttet. Autentisering av Outlook-klienter eller Hybrid Modern Authentication hører ikke hjemme her, fordi det ikke velger et SMTP-hopp.

## Hvordan connectorer gjenkjenner motparten

Når ruten er valgt, må hver side kunne stole på motparten. Hybrid Configuration Wizard oppretter lokale Send-/Receive-connectorer og passende Inbound-/Outbound-connectorer i Exchange Online. Transporten går via SMTP på TCP 25 og bruker TLS ([Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites), [Hybrid mail flow](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/hybrid-mail-flow)).

Den lokale Send Connector fastsetter mål og TLS-krav for sameksistensdomenet. Cloud Outbound Connector beskriver den lokale organisasjonen som mål. I motsatt retning aksepterer Receive- eller Inbound Connector trafikken basert på dokumenterte betingelser, blant annet sertifikatidentitet og opprinnelse.

Sertifikatet har her en konkret oppgave: Det identifiserer SMTP-endepunktet i TLS-håndtrykket. Subject eller Subject Alternative Name, connectorparametere, presentert sertifikatkjede og faktisk vertsnavn må samsvare. Et gyldig sertifikat i sertifikatlageret er ikke tilstrekkelig dersom transporttjenesten presenterer et annet.

Eksperter kontrollerer derfor begge retninger hver for seg. Retning A kan fungere, mens retning B mislykkes på grunn av en annen connector, et annet DNS-mål eller sertifikatnavn.

## Mottakerruting og delte domener

En fungerende connector sier ennå ikke hvilke meldinger som bruker den. Denne beslutningen begynner med mottakerobjektet. En lokal Remote Mailbox peker til skyen. Et synkronisert lokalt postboksobjekt i Exchange Online peker tilbake til den lokale organisasjonen.

Accepted Domains fastsetter i tillegg om en organisasjon er ansvarlig for alle mottakere i et domene, eller kan videresende ukjente mottakere. I delte domener er en Internal Relay-konfigurasjon bare sikker når neste hopp kjenner eller avviser ukjente mottakere korrekt ([Accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains), [Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)).

Et utdatert Remote Mailbox-objekt kan derfor føre til feilruting til tross for sunn TLS. Omvendt hjelper ikke en korrekt Target Address dersom cloudconnectoren er deaktivert. Diagnosen knytter objekt og transport sammen, i stedet for å se på bare én side.

For eksperter blir videresendinger, e-postkontakter, distribusjonsgruppeutvidelse og transportregler viktige. De kan endre den opprinnelige mottakeradressen eller opprette flere mottakere. Hver resulterende melding får sin egen rutingsbeslutning.

## E-postgatewayer foran eller bak Exchange Online

Mange organisasjoner utvider hybrid med en Secure Email Gateway eller en skybasert filtreringsplattform. Dette legger til minst ett SMTP-hopp. Veien må tegnes separat for innkommende og utgående meldinger ([Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud)).

Hvis gatewayen står foran Exchange Online, peker MX-posten til gatewayen. EOP ser da først kilde-IP-en dens. Enhanced Filtering for Connectors kan inkludere den opprinnelige avsenderinformasjonen i Microsofts filtervurdering når connector og hoppede over IP-er er riktig konfigurert ([Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)).

For utgående e-post må det være fastsatt om Exchange Online sender direkte, via gatewayen eller ved CMT først lokalt og deretter via gatewayen. Flere tillatte veier kan omgå policyer og skape ulike DKIM-signaturer, egress-IP-er og logger.

Ekspertkontrollen er en graf over tillatte veier: Hver pil angir initiativtaker, mål, port, TLS-kontroll, tillatte domener, filtreringsoppgave, køeier og loggkilde. Et gatewaynavn uten disse opplysningene er ennå ikke en arkitektur.

## Spor en melding ende til ende

Feilsøkingen begynner med en testmelding med kjent avsender, mottaker og tidspunkt. Først kontrolleres den offentlige veien, deretter hendelsene på hver involverte Exchange-side.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für DNS- und SMTP-Test">
    <button type="button" role="tab" data-os-tab="windows" aria-selected="true">Windows</button>
    <button type="button" role="tab" data-os-tab="unix" aria-selected="false">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName -Type MX example.com
Test-NetConnection mail.example.com -Port 25
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig MX example.com
nc -vz mail.example.com 25
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) og [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) viser det publiserte MX-målet. [`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) og [`nc`](https://man.openbsd.org/nc) kontrollerer om TCP 25 er tilgjengelig fra målepunktet. Det beviser ennå ikke at TLS- eller connectorkontrollen lykkes.

Deretter følger SMTP-håndtrykket. OpenSSL kan brukes på begge administratorplattformene.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-STARTTLS-Test">
    <button type="button" role="tab" data-os-tab="windows" aria-selected="true">Windows</button>
    <button type="button" role="tab" data-os-tab="unix" aria-selected="false">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
openssl s_client -starttls smtp -connect mail.example.com:25 `
  -servername mail.example.com -showcerts
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
openssl s_client -starttls smtp -connect mail.example.com:25 \
  -servername mail.example.com -showcerts
```

  </div>
</div>

[`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) viser sertifikatkjede, navn og TLS-forhandling. For en fullstendig autorisert testdialog egner [`swaks`](https://jetmore.org/john/code/swaks/) seg. Produksjonsmeldinger testes ikke med fritt oppdiktede avsendere; testidentitet og forventet rute fastsettes på forhånd.

I Exchange Online leverer [`Get-MessageTraceV2`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2) skyhendelsene. Lokalt viser [`Get-MessageTrackingLog`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog) behandlingen på Exchange-servere, og [`Get-Queue`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue) viser ventende neste hopp. Tidsstempler bringes til en felles tidssone; Internet Message ID og Network Message ID hjelper med å koble sammen delene ([Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq), [Message tracking](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking)).

## Typiske feilbilder uten temahopp

En **TLS-feil** behandles først som et transportproblem: Hvilken vert ble koblet til, hvilket sertifikat presenterte den, og hvilken connectorbetingelse forventet motparten? Mottakersynkronisering er først relevant når meldingen rutes feil etter vellykket mottak.

En **NDR for ukjent mottaker** leder derimot først til mottakerobjektet og typen Accepted Domain. Først når objektet er korrekt, kontrolleres det om valgt rute transporterer det til riktig side.

En **ventende kø** krever neste hopp, tidspunkt for nytt forsøk og `LastError`. Den åpne porten til målverten er bare neste test. En **sløyfe** vises i gjentatte `Received`-headere, hopp eller sporingshendelser og oppstår som regel når begge sider videresender ukjente mottakere til hverandre ([RFC 5321: Trace information and loop detection](https://datatracker.ietf.org/doc/html/rfc5321#section-6.3)).

En **forbindelse som bare fungerer i én retning** er ingen selvmotsigelse. Motretningene bruker ulike avsendere, connectorer og sertifikatkontroller. De registreres og testes separat.

## Sikkerhet, drift og endringer

Hybrid-SMTP åpner en bevisst tillatt transportvei. Denne veien bør begrenses til dokumenterte kilde- og målsystemer, sertifikater og domener. Åpne reléer, for brede IP-områder eller connectorer som klassifiserer enhver melding som pålitelig, strider mot denne modellen.

Sertifikatbytter planlegges som rutingsendringer. Før utløp kontrolleres nytt sertifikat, tjenestetilordning, presentert kjede og connectorforventning på begge sider. Deretter følger testmeldinger i begge retninger og en kontrollert tilbakeføringsplan.

For løpende drift overvåkes minst hybridconnectorer, sertifikatutløp, vekst i køer, Message Trace-feil, DNS-mål og gatewaytilstand. Ved Centralized Mail Transport kommer kapasiteten til den lokale utgående veien i tillegg.

## Sikkerhetskopi og gjenoppbygging av e-postveien

SMTP-meldinger ligger under en driftsforstyrrelse i køer på systemene som til enhver tid er ansvarlige. En konfigurasjonssikkerhetskopi gjenoppretter ikke disse ventende meldingene. Derfor vurderes konfigurasjon og transporttilstand separat.

Gjenoppbyggingen omfatter connectorparametere på begge sider, Accepted og Remote Domains, transportregler, offentlige DNS-oppføringer, sertifikater med private nøkler, gatewaykonfigurasjon og HCW-valg. Hemmeligheter lagres beskyttet; lesbare eksporter dokumenterer struktur og avhengigheter.

Etter gjenoppbygging testes ikke bare en port. En merket melding går gjennom den forventede veien i hver retning. Message Trace, lokale sporingslogger, gatewaylogger og målpostboksen bekrefter hvert hopp. Først denne ende-til-ende-godkjenningen viser at ruting og filtrering igjen er korrekt.

## Teknisk utvikling og avveininger

Hybrid Configuration Wizard automatiserte forbindelsen til Exchange Online gjennom flere Exchange-generasjoner. Transporten forble SMTP/TLS, mens cloudconnectorer, støttede sertifikater og rutingsalternativer ble videreutviklet ([Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)).

Direkte internetttransport holder veiene korte og bruker den aktuelle plattformen der postboksen befinner seg. Centralized Mail Transport samler kontrollen, men gjør sky-e-post avhengig av den lokale utgående veien. En tredjepartsgateway tilfører spesialisert filtrering eller kryptering, men øker antallet hopp og loggkilder. Riktig valg følger av et dokumenterbart krav, ikke av ønsket om at alt i diagrammet skal gå gjennom samme boks.

## Kilder

- [Microsoft Learn – Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing)
- [Microsoft Learn – Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)
- [Microsoft Learn – Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)
- [Microsoft Learn – Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)
- [Microsoft Learn – Hybrid mail flow](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/hybrid-mail-flow)
- [Microsoft Learn – Enable-RemoteMailbox](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox)
- [Microsoft Learn – Set up connectors to route mail](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail)
- [Microsoft Learn – Connectors on Exchange servers](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors)
- [Microsoft Learn – Accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains)
- [Microsoft Learn – Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)
- [Microsoft Learn – Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud)
- [Microsoft Learn – Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)
- [Microsoft Learn – Trace an email message](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message)
- [Microsoft Learn – Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq)
- [Microsoft Learn – Message tracking](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking)
- [Microsoft Learn – Queues in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues)
- [Microsoft Learn – Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2)
- [Microsoft Learn – Get-MessageTrackingLog](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog)
- [Microsoft Learn – Get-Queue](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [BIND 9 – dig manual](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc(1)](https://man.openbsd.org/nc)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Swaks – SMTP test tool](https://jetmore.org/john/code/swaks/)
- [RFC 5321 – Trace information and loop detection](https://datatracker.ietf.org/doc/html/rfc5321#section-6.3)
