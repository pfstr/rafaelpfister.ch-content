---
title: "HIN: Tillitsområde, e-postgateway og Stargate"
blatt: "hin"
description: "HIN for meldings- og plattformadministratorer: tillits- og identitetsmodell, HIN Mail, klassisk e-post og Access Gateway, HIN Client, PKI, SMTP- og portalveier, e-postlager, Stargate-arkitektur, migrering, overvåking, gjenoppretting og diagnose."
fakten:
  - label: Plattform
    wert: Tillitsområde for det sveitsiske helsevesenet
    href: https://www.hin.ch/de/services/hin-mail/hin-mail.cfm
  - label: Operatør
    wert: Health Info Net AG · grunnlagt i 1996
    href: https://www.hin.ch/de/ueber-hin/unternehmen/geschichte.cfm
  - label: E-postmodell
    wert: HIN til HIN automatisk · eksternt ved konfidensialitetsmerking
    href: https://support.hin.ch/de/service/hin-mail-und-mobile.cfm
  - label: Klassisk edge
    wert: E-post og Access som virtuelle apparater
    href: https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm
  - label: Målarkitektur
    wert: Postfix → MXEngine → Postfix
    href: https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf
  - label: Kjernestack
    wert: OPA/Rego · PostgreSQL · Vault · MinIO
    href: https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf
  - label: Mesh-transport
    wert: WireGuard · IDAgent · Port 19818
    href: https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf
  - label: Tillitsanker
    wert: HIN-identitet · S/MIME · nøkler og CSR
    href: https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf
  - label: Webtilgang
    wert: HIN Client · Access Gateway · SAML · OAuth 2.0
    href: https://download.hin.ch/oauth2/doku/de/
  - label: E-postlager
    wert: separat fra gatewaytransport · IMAP/POP/webmail
    href: https://support.hin.ch/de/service/hin-gateway.cfm
  - label: Utrulling
    wert: Linux · Docker Compose · VM-avbilder
    href: https://health-info-net-ag.github.io/Stargate-deployment/de/
  - label: Observability
    wert: Prometheus · Promtail/Loki · Node Exporter
    href: https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf
werbung:
  - stargate
  - newsletter
ctaThemen:
  - hin-gateway
translationSourceHash: 4b3c48b30e1dfb939e082bba0637c401f51d48a6dc5ed8e71b90c9d8114b2c8c
translationModel: gpt-5.6-terra
translatedAt: 2026-10-07T10:46:39.273Z
translationReview: automatic
---

# HIN: Tillitsområde, e-postgateway og Stargate

HIN er ikke et enkelt krypteringsprogram, men et bransjerelatert tillitsområde bestående av verifiserte identiteter, sentrale plattformtjenester og tilgangs- eller gatewaykomponenter. **HIN Mail** beskytter e-postkommunikasjon, **HIN Access** formidler tilgang til beskyttede webapplikasjoner, og en **HIN-identitet** kobler den autoriserte personen eller organisasjonen med kryptografisk nøkkelmateriale. Hos institusjoner ligger e-post- og Access-komponentene i grensen mellom egen infrastruktur og HIN-plattformen ([HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm), [HIN kollektivmedlemskap med gateway](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm)).

For meldingsadministratorer må fire tilstandsrom holdes atskilt. Den lokale e-postserveren har postbokser, connectorer og køer. Gatewayen tar beslutninger om transport, tillit og beskyttelse. HIN-plattformen stiller katalog-, identitets-, nøkkel-, e-post- og Access-tjenester til rådighet. For mottakere utenfor HIN Community kommer det i tillegg en web- og autentiseringsvei. En feil i ett rom er ikke automatisk en feil i alle de andre; en tilgjengelig SMTP-port beviser for eksempel verken en gyldig HIN-identitet eller vellykket portallevering ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm), [HIN Mail til ikke-medlemmer](https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm)).

Også produktnavnet trenger tidsmessig plassering. Den klassiske gatewaygenerasjonen omfatter en Mail Gateway, MGW, og avhengig av avtalen en Access Gateway, AGW. HIN beskriver som etterfølger den e-postkompatible mesh-noden som er utviklet i prosjektet **Stargate**. Den forblir SMTP-kompatibel overfor lokale e-postsystemer, men endrer arkitektur, nøkkeldistribusjon, utrulling og transporten mellom organisasjoner. Utsagn om målplattformen må derfor ikke uten videre overføres til en eksisterende MGW, og omvendt ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm)).

Forklaringen følger en melding fra avsenderen via HIN-identitet og gateway til mottakeren. Deretter behandles plattformbytte, avhengigheter, diagnose, nøkkelmateriale og gjenoppretting.

## Arkitekturtilnærming: tillitsområde med edge-komponenter

Den klassiske kollektivmodellen leverer HIN Mail og HIN Access som virtuelle apparater. HIN dokumenterer S/MIME-kryptering og -signatur på e-postdomenenivå samt et revisjonsspor; Access Gateway fungerer som lokal identitetsleverandør, tilordner HIN-eID-er til verifiserte forespørsler og kan integrere eksisterende katalog- og autentiseringstjenester ([HIN kollektivmedlemskap med gateway](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm)).

Denne konstruksjonen er en **edge-tillitsmodell**: Organisasjonen kontrollerer e-postservere, DNS, brannmur, intern levering og den lokale gatewayressursen; HIN driver de overordnede tillits- og plattformtjenestene. En positiv SMTP-avslutning overfører ansvaret for en konkret melding til neste hop; den sier ingenting om hvorvidt den etterfølgende HIN-, portal- eller mottakerveien allerede er fullført. For den nye gatewaystakken tilordner HIN uttrykkelig kunden DNS og e-postomdømme ([HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf), [RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321)).

Stargate flytter denne grensen til en organisasjonsrelatert mesh-node. HIN beskriver en skyinnfødt mikrotjenestearkitektur med REST-API-er, desentraliserte identiteter, distribuert nøkkelhåndtering og programmerbare policyer. Det er planlagt en egen instans per organisasjon; HIN nevner virtuelle avbilder og containere som utrullingsformer, samt også OpenShift- og Kubernetes-miljøer i produktbeskrivelsen. Dette er en publisert målarkitektur, ikke bevis for at enhver eksisterende HIN-organisasjon allerede drives slik ([HIN Gateway: arkitektur og utrulling](https://support.hin.ch/de/service/hin-gateway.cfm), [HIN Gateway-produktbeskrivelse](https://www.hin.ch/de/services/hin-mail/hin-gateway.cfm)).

## Teknologistack og ansvarsområder

Den offentlig dokumenterte stacken består av flere generasjoner og må ikke blandes til én enkelt monolitt:

| Nivå | Implementering av den nye gatewayen | Tilstands- og feilområde | Administrativt bevis |
|---|---|---|---|
| SMTP-rand | **Postfix Relay** for mottak, retry, DNS-ruting og levering | Transport er atskilt fra innholdsbeslutning | SMTP-sluttrespons, kø-ID, neste hop, MX og PTR |
| Behandling | **MXEngine** for HTTP-/SMTP-inntak, transformasjon og leveringsstrategi | Behandlingsfeil kan skilles fra Postfix-retryer | Message-ID, MXEngine-hendelse, transformasjon, retur til Postfix |
| Policy | **Open Policy Agent med Rego**, eventuelt synkronisert via Git | Regeltilstand er data- og versjonstilstand | Policyrevisjon, inndata, resultat, godkjenning og tilbakeføring |
| Identitet og krypto | **S/MIME Keys Client**, **IDAgent**, Issuer og Verifier | CSR, sertifikat, privat nøkkel og peeridentitet er separate objekter | Fingeravtrykk, innehaver, utløp, Issuer, peer og rotasjon |
| Persistens | **PostgreSQL** per tjeneste, **Vault** for hemmeligheter, **MinIO** for meldinger og vedlegg | Database, Secret Store og objektlagring har egne gjenopprettingsgrenser | Volum, sikkerhetskopieringstid, gjenopprettingstest og konsistenskontroll per tjeneste |
| Mesh-transport | **WireGuard** via IDAgent | Kanaltilstand er ikke det samme som SMTP-levering | Peer-nøkkel, endepunkt, håndtrykk, port 19818 og etterfølgende SMTP-hendelse |
| Observability | **Promtail → Loki**, **Node Exporter**, Prometheus-kompatible måledata og Version Collector | Loggtransport, vertsmetrikk og tjenestehelse kan svikte separat | Liveness, scrape-alder, loggmottak, vertsressurser og tidsgrunnlag |
| Utrulling | Linux, **Docker Compose**, persistente Docker-volumer; VM-avbilder som installasjonsvei | Vert, containere, avbilder og volumer har ulike livssykluser | godkjent avbildning, Compose-konfigurasjon, voluminventar og omstartstest |

HIN dokumenterer meldingsflyten uttrykkelig som `External SMTP → Postfix → MXEngine → Postfix → External SMTP`. Den tvungne overleveringen til MXEngine forhindrer en policy-bypass; Postfix forblir ansvarlig for levering og retryer. OPA/Rego holder forretningsregler utenfor applikasjonskoden. PostgreSQL, Vault og MinIO lagrer ulike tilstandsklasser og må ikke behandles som ett enkelt filsystem i sikkerhetskopieringen ([HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf), [HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf)).

Den tekniske installasjonsdokumentasjonen nevner Linux-distribusjoner fra RHEL-kompatible familier samt Ubuntu og Debian, Docker Compose-drift på én enkelt vert og VM-avbilder for flere plattformer. Denne støtteerklæringen er versjons- og utrullingsrelatert; det avgjørende er HIN-dokumentasjonen som er godkjent på installasjonstidspunktet, ikke en statisk overført distribusjonsliste ([HIN Stargate Deployment](https://health-info-net-ag.github.io/Stargate-deployment/de/)).

E-postlageret forblir en annen grense: Ifølge HIN fungerer Stargate som Mail Transport Agent, mens IMAP fortsatt er lagt til den eksisterende Zimbra-plattformen. HIN Access utgjør i tillegg en selvstendig autentiserings- og autoriseringsvei via klient, AGW eller Access Control Service ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm), [HIN Client 3-håndbok](https://download.hin.ch/documentation/HIN_Client3_Handbuch_de_v1.1.pdf)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-hin.svg?v=20260813" title="Interaktive Infografik: HIN Vertrauensraum mit lokalem Mailserver, klassischem Mail und Access Gateway, HIN Identität, Mailplattform, Nichtmitglieder-Portal sowie Stargate-Zielarchitektur und Betriebssignalen" loading="lazy">
  <a href="/images/kb-interaktiv-hin.svg?v=20260813">Åpne den interaktive grafikken direkte</a>.
</iframe>

## Identitet og PKI med autorisering

En HIN-identitet er mer enn en e-postadresse. Ved aktivering oppretter HIN Client et nøkkelpar; passordet låser opp det lokale nøkkelmaterialet. I den nye gatewayen genererer S/MIME Keys Client kryptografiske nøkler og Certificate Signing Requests, mens Vault lagrer private nøkler, legitimasjon og sensitiv konfigurasjon. Et sertifikat knytter en offentlig nøkkel til en navngitt identitet, men erstatter ikke applikasjonsrelatert autorisering ([HIN Client 3-håndbok](https://download.hin.ch/documentation/HIN_Client3_Handbuch_de_v1.1.pdf), [HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf), [RFC 5280](https://datatracker.ietf.org/doc/html/rfc5280)).

Livssyklusen hører hjemme i IAM-runbooken: registrering, aktivering, enhetsbytte, rolle- eller navneendring, sperring, ny registrering ved mistanke om nøkkelkompromittering og uttreden. HIN krever en identitetskontroll etter bestilling og skiller mellom personlige eID-er og organisasjons- eller Device-ID i kollektivmodellen. En fungerende e-posttransport må ikke anses som bevis på at en tidligere eller feiltilordnet identitet ikke lenger har tilgang ([HIN-identitet](https://support.hin.ch/de/service/hin-identitaet.cfm), [HIN kollektivmedlemskap med gateway](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm)).

HIN Access og HIN Mail deler tillitsområdet, men ikke den samme protokollflyten. Access Control Service kan be om et autentiseringsnivå via SAML `RequestedAuthnContext`; HIN dokumenterer profiler for passord og MFA. For integrasjoner tilbyr HIN også OAuth 2.0-flyter for Authorization Code og Client Credentials. Autentisering, tokenutstedelse og autorisering gjennom målapplikasjonen skal loggføres separat ([HIN Authentication Context](https://support.hin.ch/de/thema/hin-access/pwd.cfm), [HIN OAuth2-integrasjon](https://download.hin.ch/oauth2/doku/de/), [SAML 2.0 Core](https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf), [RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749)).

Først når identitet og PKI er avklart, kan e-postveien vurderes. Gateway og policy avgjør, avhengig av avsender, mottaker og mål, hvilken beskyttelsesmetode som brukes.

## E-postveier og beskyttelsesbeslutninger

Med de involverte HIN-komponentene i mente kan den konkrete meldingsveien nå forklares. Følgende tilfeller skiller seg etter hvor avsender og mottaker befinner seg, og hvilken tjeneste som treffer beskyttelsesbeslutningen.

### Mellom HIN-deltakere

HIN beskriver meldinger mellom HIN-adresser som automatisk overført i samsvar med personvernet. Plattformen markerer integritetsstatusen i emnefeltet med `[HIN secured]` eller `[Not secured by HIN]`. Denne markeringen er et brukersignal, men ikke tilstrekkelig teknisk korrelasjon: For en hendelse trengs i tillegg envelope-avsender og -mottaker, Internet `Message-ID`, `Received`-kjeden, gatewayhendelsen og tidsvinduet ([HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm), [HIN Mail og Mobile](https://support.hin.ch/de/service/hin-mail-und-mobile.cfm), [RFC 5322](https://datatracker.ietf.org/doc/html/rfc5322)).

En klassisk organisasjonsvei kan beskrives som `Mailserver → SMTP → MGW → HIN → MGW → SMTP → Mailserver`. Hver stasjon avslutter en økt og kan ha sin egen kø. HIN beskriver kollektivmodellen som S/MIME-kryptering og -signatur på e-postdomenenivå; den lokale leveringen før og etter denne gatewaygrensen forblir et eget beskyttelses- og driftsområde ([HIN kollektivmedlemskap med gateway](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm), [RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321)).

### Til personer uten HIN-medlemskap

For mottakere uten HIN-adresse må avsenderen ifølge HIN-dokumentasjonen uttrykkelig merke meldingen som konfidensiell, for eksempel med `(Vertraulich)` i emnefeltet. Mottakeren åpner det beskyttede innholdet via en webvei og autentiserer seg med mobilnummer og SMS-kode; et sikkert svar er mulig. Dermed oppstår ytterligere tilstander: varslings-e-post, portalobjekt, mottakerregistrering, andre faktor, oppbevaring og svarkanal ([HIN Mail til ikke-medlemmer](https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm), [HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm)).

Et levert varsel er ennå ikke en lest melding. Syntetiske tester bør derfor dekke hele veien frem til pålogging, åpning, vedlegg og svar. Videresendinger fra portalen kan forlate beskyttelsesveien; HIN påpeker uttrykkelig at visse videresendingstyper sendes ukryptert. Slike brukerhandlinger hører hjemme i opplæring, DLP-modell og hendelsesanalyse ([HIN Mail til ikke-medlemmer](https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm)).

### Enheter, applikasjoner og masseutsendelse

Den klassiske Mail Gateway kan fungere som SMTP-grensesnitt for interne enheter; HIN dokumenterer denne muligheten også for Stargate. HIN Mail-tjenestesiden påpeker at systemutsendelse via gatewayen krever separat lisensiering. Skannere, KIS, laboratorieapplikasjoner og batchprosesser trenger derfor en egen, dokumentert avsender-, relay-, mengde- og feilvei i stedet for skjult medbruk av brukerens e-postflyt ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm), [HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm)).

## SMTP-ruting og mottaksgrenser

Gatewayen må for hver retning vite nøyaktig hvilke domener som er lokale, autoritative, skal videresendes eller avvises. Uklar ansvarsfordeling mellom Exchange, cloud connector, Secure Mail Gateway og HIN-edge fører enten til bypasser eller løkker. [SMTP](/kb/smtp) krever sporlinjer og beskriver løkkedeteksjon; den faktiske ansvarsoverføringen skjer først med et positivt svar etter hele meldingsinnholdet ([RFC 5321, Trace Information](https://datatracker.ietf.org/doc/html/rfc5321#section-4.4), [RFC 5321, DATA](https://datatracker.ietf.org/doc/html/rfc5321#section-4.1.1.4)).

For hver rute må minst disse verdiene inngå i driftsdokumentasjonen:

- lokal listener, port og forventede kildenettverk eller peeridentiteter;
- EHLO-navn, envelope-domener og autoriserte relayområder;
- rekkefølge på spam-/skadevarefilter, HIN-gateway og internt e-postsystem;
- neste hop, DNS- eller smarthost-oppløsning og [TLS](/kb/tls)-krav;
- atferd når HIN-plattformen, identitets- eller nøkkelkomponenten ikke er tilgjengelig;
- køalder, retryplan, maksimalt hopptall og bounce-ansvar;
- unntaksveier for enheter, systemutsendelse og migreringssameksistens.

For Exchange Online er dette et spørsmål om connectorer og ikke bare DNS. Microsoft dokumenterer ruting til tredjepartsgatewayer samt sertifikat- eller IP-baserte connectorbetingelser. HIN-FAQ-en bekrefter grunnleggende støtte for hybride og Microsoft 365-nære arkitekturer, men viser for konkret konfigurasjon til kundespesifikk migreringsdokumentasjon ([Microsoft: Mail flow using connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow), [Microsoft: Third-party cloud mail flow](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud), [HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm)).

## E-postlager og klienttilgang med token

Postbokstilgang og gatewaytransport er forskjellige driftsmodeller. HIN publiserer IMAP på port 993 med TLS og Message Submission på port 587 med STARTTLS for personlige HIN-e-postkontoer; et generert e-posttoken brukes som passord. POP på port 995 er også dokumentert, men laster vanligvis ned meldinger lokalt og kan fjerne dem på serversiden. Protokollrollene samsvarer med [IMAP](/kb/apache-james#protokolle-tls-und-ports), POP3 og Message Submission, ikke gateway-til-gateway-transport ([HIN Mail Token Service](https://support.hin.ch/de/service/hin-mail-und-mobile/token-service-mail-client.cfm), [HIN POP-konfigurasjon](https://support.hin.ch/de/service/hin-mail-und-mobile/mail-clients-einrichten-pop.cfm), [RFC 9051](https://datatracker.ietf.org/doc/html/rfc9051), [RFC 1939](https://datatracker.ietf.org/doc/html/rfc1939), [RFC 6409](https://datatracker.ietf.org/doc/html/rfc6409)).

HIN Client-håndboken dokumenterer at den historiske lokale e-postproxyen erstattes av tokenbasert tilgang. HIN påpeker uttrykkelig for terminalservere at sending og mottak skjer via e-posttoken når proxyen er deaktivert. Dette er relevant for den tekniske historien: Klienten var opprinnelig kommunikasjonsformidler for HTTP, SMTP, POP og IMAP; senere driftsmodeller kobler tilgang til e-postklient og HIN-identitetsklient sterkere fra hverandre ([HIN Client 3-håndbok](https://download.hin.ch/documentation/HIN_Client3_Handbuch_de_v1.1.pdf), [HIN Client på terminalservere](https://support.hin.ch/de/thema/hin-client/hin-client-auf-terminalserver.cfm)).

HIN beskriver den eksisterende Mail Storage Agent i Stargate-FAQ-en som Zimbra-basert og foreløpig uberørt av den nye gatewaytransporten. En vellykket Stargate-e-postflyt beviser derfor verken IMAP-tilgjengelighet eller postbokkonsistens. Omvendt kan webmail fungere mens organisasjonsconnectoren eller mesh-transporten er forstyrret ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm)).

Den klassiske e-postlagerveien er ikke den eneste driftsformen. Stargate forskyver funksjoner og ansvar, og må derfor forstås som en egen meldings- og administrasjonsvei.

## Stargate som målarkitektur

Stargate skal bevare e-postkompatibiliteten ved kanten og samtidig muliggjøre mer generell, desentralisert datautveksling. HIN nevner Self-Sovereign Identity, Data Mesh, mikrotjenester, RESTful API-er, open source-komponenter og en e-postkompatibel mesh-node. For kanalen mellom Stargate-instanser er en [WireGuard](https://www.wireguard.com/protocol/)-basert applikasjon-til-applikasjon-transport annonsert; HIN avgrenser den uttrykkelig fra en generell VPN-tunnel ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm)).

For administratorer følger dette tredelte sporet:

1. Lokalt forblir SMTP med mottakssvar, kø og connector den kontrollerbare grensen.
2. Mellom mesh-nodene kommer identitets-, discovery-, nøkkel- og WireGuard-kanaltilstander i tillegg.
3. Ved målet oppstår igjen en lokal SMTP-vei til det mottakende e-postsystemet.

De publiserte systemoversiktene dokumenterer Postfix, MXEngine, OPA/Rego, PostgreSQL, Vault, MinIO, IDAgent, Promtail/Loki og Prometheus-kompatible måledata. Et programmeringsspråk eller et komplett kildekodetre for produkttjenestene er ikke oppgitt der; denne informasjonen skal derfor ikke gjettes fra containeravbilder eller fremmedprosjekter med samme navn ([HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf), [HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf)).

Nettverksgrensene for den nye gatewayen er offentlig dokumentert på en uvanlig konkret måte:

| Port og retning | Rolle | Driftsbetydning |
|---|---|---|
| TCP 25 innkommende og utgående | SMTP-mottak og MX-basert levering | Internetteksponering, omdømme, kø og neste-hop-ruting |
| TCP 8084 innkommende | HTTP-callback fra ekstern Sealer | ifølge HIN med hensikt uten ekstra TLS-lag, fordi nyttelasten selv er kryptert |
| TCP og UDP 19818 i begge retninger | WireGuard mellom IDAgent-er | kontroller peer-nøkkel, endepunkt, NAT og brannmur samlet |
| TCP 443 og 4433 utgående | Registry, S/MIME-CA, Sealer, Issuer, Logging og Verifier | plattformavhengighet til tross for lokal gatewaydrift |
| TCP og UDP 53 utgående | MX-, SPF-, A/AAAA- og PTR-oppløsning | Ruting- og sikkerhetsbeslutninger avhenger av [DNS](/kb/dns) |

De lokalt eksponerte diagnose- og tjenesteportene, herunder PostgreSQL, Vault, MinIO og metrikkendepunkter, skal ifølge HIN ikke være tilgjengelige fra det offentlige internettet. Administrasjonstilgang på 22, 443, 8180 eller 8190 hører til et definert administrasjonsnettverk og ikke en generell internettåpning ([HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf), [HIN Stargate Deployment](https://health-info-net-ag.github.io/Stargate-deployment/de/)).

HIN beskriver migreringen som en parallell oppbygging av den nye gatewayen. Først etter konfigurasjon, tester og bekreftet driftsberedskap bestemmer kunden om omkoblingen. Eksisterende IP-adresser kan i utgangspunktet gjenbrukes, men konfigurasjonen må oversettes. En sikker cutover-runbook inneholder derfor minst inventar, eksport, parallellrute, testmatrise, omkoblingskriterium, returrute, køhåndtering og tydelig rollback ([HIN Gateway: migrering](https://support.hin.ch/de/service/hin-gateway.cfm)).

## Høy tilgjengelighet med backup og gjenoppretting

HIN dokumenterer en redundant OpenShift-klynge for sin egen Stargate-side, men planlegger ikke automatisk to redundante virtuelle maskiner på kundesiden. Disse utsagnene må ikke trekkes sammen til en generell ende-til-ende-høytilgjengelighet. En redundant plattformtjeneste beskytter ikke mot én enkelt lokal hypervisor, en feil connectorregel, utløpt nøkkel eller blokkert brannmur ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm)).

I den nye gatewayen persisterer tjenestene via Docker-volumer. Vault forsegler seg automatisk etter en containeromstart og krever en kontrollert unseal-prosess. Den tekniske oversikten nevner som standard daglige sikkerhetskopier av databaser, Vault-nøkler og -hemmeligheter, konfigurasjonsfiler samt sertifikater; ansvar og oppbevaring skal avtales med kunden ([HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf)).

Et gjenopprettingsinventar bør per generasjon minst inneholde:

| Objekt | Virkning ved tap | Dokumenterbar gjenopprettingshandling |
|---|---|---|
| Vert, Compose- og containerdefinisjon | Edge-node starter ikke | klargjør nytt mål fra godkjent avbildning og opprett deterministisk runtime |
| Kundekonfigurasjon og OPA/Rego-policyer | feil rute, policy eller domene | legg inn versjonert tilstand, last policy og kjør testmatrise |
| PostgreSQL-databaser | policy-, metadata- eller agenttilstand mangler | utfør databaserestore per tjeneste og referensiell kontroll |
| Vault-nøkler, hemmeligheter og sertifikater | dekryptering, peer- eller organisasjonsidentitet mangler | utfør godkjent restore, unseal og kryptografisk funksjonstest |
| MinIO-meldinger og vedlegg | melding eller arkivobjekt mangler | avklar omfang og oppbevaring separat med HIN og test objektgjenoppretting |
| Connectorer og DNS | bypass, løkke eller manglende levering | kontroller ruten i begge retninger med entydig Message-ID |
| Kø eller overleveringsbevis | duplikater eller meldingstap | avklar åpent ansvar per melding før omkobling |
| Postboks og e-posttoken | klienttilgang forstyrret | valider separat via webmail og IMAP/Submission |
| Revisjons- og driftslogger | hendelse kan ikke rekonstrueres | test tidsgrunnlag, eksport, oppbevaring og SIEM-inngang |

Den offentlige backuplisten nevner database, Vault, konfigurasjon og sertifikater, men ikke uttrykkelig MinIO-meldinger og -vedlegg. Derav kan verken sikkerhetskopiering eller bevisst utelatelse avledes; nettopp dette punktet hører skriftlig hjemme i en avtale om retention, backup og restore før produksjonssetting. Et generisk VM-snapshot beviser dessuten verken en konsistent PostgreSQL-tilstand eller en gjenopprettbar Vault ([HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf), [HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf)).

Ved en feil følges meldingen fra mottak via identitets- og policybeslutning til valgt leveringsvei; først deretter startes eller omgås enkeltkomponenter på nytt.

## Overvåking og hendelsestriage

Et brukbart driftsbilde kombinerer lokale og sentrale signaler:

- tilgjengelighet og køalder per SMTP-neste-hop;
- mottaks-, overleverings- og bouncekvoter med korrelerbar Message-ID;
- HIN-, portal-, identitets-, nøkkel- og policyfeil separat;
- sertifikatutløp, registrerings- og tokenstatus;
- postbokstilgang via webmail og IMAP uavhengig av gatewayen;
- DNS- og brannmuroppløsning via navn i stedet for fastkodede HIN-IP-adresser;
- plattformmeldinger på [HIN Status](https://status.hin.ch/) pluss lokal telemetri.

Den nye stacken leverer Prometheus-kompatible tjenestemåledata, vertsmetrikk via Node Exporter, sentral loggvideresending via Promtail til Loki og en Version Collector som spør liveness-endepunkter. For varsling må minst manglende scrape, uteblivende loggmottak, Postfix-køalder, MXEngine-feil, Vault-seal-tilstand, PostgreSQL- og MinIO-kapasitet, WireGuard-peer-tilstand samt sertifikatutløp behandles separat ([HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf), [HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf)).

HINs brannmurdokumentasjon anbefaler DNS-navn fordi IP-adresser kan endres, og nevner blant annet HTTPS- og SMTP-veier for klienttjenester. En global statusside kan ikke oppdage en lokal DNS-, NAT-, MTU-, connector- eller nøkkelfeil. Triage begynner derfor med omfang: én bruker, én identitet, ett domene, én retning, én gateway eller plattformen ([HIN brannmurtilpasninger](https://support.hin.ch/de/thema/hin-client/anpassung-firewall-hin-client.cfm), [HIN Status](https://status.hin.ch/)).

Det klassiske kollektivtilbudet nevner et revisjonsspor for e-postflyten; den nye gatewayen supplerer dette med strukturerte sentrale logger. For en sammenhengende e-postanalyse må disse bevisene korreleres med lokale SMTP- og e-postsystemlogger. Tidssynkronisering og ensartede tidssoner er driftskrav, ikke kosmetiske innstillinger ([HIN kollektivmedlemskap med gateway](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm), [HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf)).

## Diagnoseverktøy

Diagnosen starter med det offentlige eller interne navnet og følger deretter den faktiske e-postveien. Først når DNS, forbindelse og sertifikat er riktige, evalueres gatewaytilstand, kø og HIN-spesifikke hendelser.

### DNS og HIN-endepunkter

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für HIN-DNS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName gateway.hin.ch -Type A
Resolve-DnsName gateway.hin.ch -Type AAAA
Resolve-DnsName smtp.mail.hin.ch -Type A
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig +short A gateway.hin.ch
dig +short AAAA gateway.hin.ch
dig +short A smtp.mail.hin.ch
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) og [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) viser om de dokumenterte navnene kan løses opp fra perspektivet til resolveren som faktisk brukes. Dette er viktigere enn en kopiert IP-verdi, særlig ved split-DNS og proxyer; HIN anbefaler uttrykkelig DNS-navn i stedet for langsiktig fastlagte adresser ([HIN brannmurtilpasninger](https://support.hin.ch/de/thema/hin-client/anpassung-firewall-hin-client.cfm)).

### TCP- og TLS-tilgjengelighet

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für HIN-TCP- und TLS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Test-NetConnection hin-gateway.example.ch -Port 25 -InformationLevel Detailed
Test-NetConnection hin-gateway.example.ch -Port 19818 -InformationLevel Detailed
curl.exe --verbose --ssl-reqd smtp://hin-gateway.example.ch:25
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nc -vz hin-gateway.example.ch 25
nc -vz hin-gateway.example.ch 19818
nc -vzu hin-gateway.example.ch 19818
openssl s_client -starttls smtp -connect hin-gateway.example.ch:25 \
  -servername hin-gateway.example.ch -showcerts
```

  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) og [`nc`](https://man.openbsd.org/nc) dokumenterer TCP-veien. UDP-kallet fra `nc` kan høyst indikere tilgjengelighet; fordi WireGuard forkaster uautoriserte pakker uten svar, er det først det autentiserte peer-håndtrykket som er et pålitelig bevis for port 19818. Windows-[`curl.exe`](https://curl.se/docs/manpage.html) og [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) kontrollerer SMTP-/STARTTLS-randen. Et vellykket håndtrykk beviser ennå ikke policybehandling eller meldingslevering ([HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf), [RFC 8446](https://datatracker.ietf.org/doc/html/rfc8446)).

### Kontrollert SMTP-transaksjon

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für einen autorisierten HIN-Gateway-SMTP-Test">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
curl.exe --verbose --url smtp://hin-gateway.intern.example:25 `
  --mail-from hin-test@example.ch `
  --mail-rcpt test-recipient@example.net `
  --upload-file .\hin-test.eml
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
swaks --server hin-gateway.intern.example --port 25 \
  --from hin-test@example.ch --to test-recipient@example.net \
  --data hin-test.eml
```

  </div>
</div>

[`curl`](https://curl.se/docs/manpage.html) og [`swaks`](https://jetmore.org/john/code/swaks/) må bare brukes mot en uttrykkelig autorisert listener og testmottaker. Det registreres sluttrespons etter `DATA`, lokal kø-ID, gatewayhendelse, valgt beskyttelsesvei, neste hop og faktisk ankomst. En `250` på `RCPT TO` er ennå ikke et mottak av meldingsinnholdet ([RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321)).

### Lokale sockettilstander

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für lokale HIN-Gateway-Socketdiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-NetTCPConnection -State Listen,Established |
  Where-Object LocalPort -In 25,443,587,993
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
ss -tanp '( sport = :25 or sport = :443 or sport = :587 or sport = :993 )'
```

  </div>
</div>

[`Get-NetTCPConnection`](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection) og [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) viser lokale listenere og etablerte TCP-økter. De er bare nyttige der administratoren har tilgang til den aktuelle verten; et administrert apparat- eller containerprodukt må ikke endres gjennom udokumentert shell-tilgang.

### Pakkebane ved riktig målepunkt

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für HIN-Paketerfassung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
pktmon filter remove
pktmon filter add HIN-SMTP -p 25
pktmon start --capture --pkt-size 0 --file-name hin.etl
pktmon stop
pktmon pcapng hin.etl -o hin.pcapng
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
tcpdump -ni any -s 0 -w hin.pcap \
  'tcp port 25 or tcp port 443 or tcp port 587 or tcp port 993'
```

  </div>
</div>

[`pktmon`](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon) og [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) ser bare trafikken ved det valgte målepunktet. For Stargate kan et lokalt SMTP-spor ikke fullt ut forklare mesh-kanalen; det trengs i tillegg gateway- og plattformhendelser. Opptak kan inneholde adresser, emnelinjer eller ukrypterte protokolldeler og må behandles som sensitive driftsdata.

## Teknisk historie

FMH og Ärztekasse grunnla Health Info Net AG i 1996, da e-post i helsevesenet vokste frem og sending av sensitive data via vanlig internett-e-post ble ansett som utilstrekkelig beskyttet. HIN startet dermed som leverandør av beskyttet e-postkommunikasjon for leger og utviklet seg til et bredere tillits- og tilgangsområde ([HIN selskapshistorie](https://www.hin.ch/de/ueber-hin/unternehmen/geschichte.cfm)).

Klientarkitekturen viser den tekniske endringen. HIN Client 1 og 2 ble erstattet av HIN Client 3. Av kompatibilitetshensyn fortsatte klienten å fungere som lokal proxy for nettlesere og e-postprogrammer; for webtilgang dokumenterte HIN senere Challenge/Response, og for e-postkontoer overgangen til uavhengige token via standardporter. Historikken forklarer hvorfor eldre installasjonsveiledninger nevner lokale proxyporter, mens nyere dokumentasjon bruker direkte IMAP-, POP- og Submission-endepunkter ([HIN Client 3-håndbok](https://download.hin.ch/documentation/HIN_Client3_Handbuch_de_v1.1.pdf), [HIN Mail Token Service](https://support.hin.ch/de/service/hin-mail-und-mobile/token-service-mail-client.cfm)).

Den klassiske HIN-kollektivmodellen samlet Mail- og Access-apparater i kundenettverket. Den nåværende tjenestesiden dokumenterer S/MIME på e-postdomenenivå, revisjonsspor, en lokal identitetsleverandør og tilkobling av eksisterende autentiserings- og katalogtjenester for denne generasjonen. Disse funksjonene forklarer det utviklede skillet mellom e-posttransport og webtilgang ([HIN kollektivmedlemskap med gateway](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm)).

Fra 2025 innførte HIN en ny levering til ikke-medlemmer; samtidig ble plattform- og Access-infrastrukturen fornyet. Gateway-dokumentasjonen som ble publisert i 2026, beskriver Stargate som neste generasjonsskifte: fra en utelukkende e-postkrypteringsgateway til en desentralisert, skyinnfødt node for e-post og strukturert utveksling av helsedata. Den tekniske dokumentasjonen konkretiserer denne endringen med Postfix, MXEngine, OPA/Rego, PostgreSQL, Vault, MinIO og containerisert drift. For en migreringsplan er det derfor ikke nok å bytte ut en VM, men identitet, nøkler, transport, observability og recovery må måles på nytt ([HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm), [HIN Access](https://support.hin.ch/de/thema/hin-access.cfm), [HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm), [HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf)).

## Kilder

- [HIN – HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm)
- [HIN – Kollektivmedlemskap med gateway](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm)
- [HIN Support – HIN Gateway og Stargate](https://support.hin.ch/de/service/hin-gateway.cfm)
- [HIN Support – HIN Mail til ikke-medlemmer](https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm)
- [HIN – Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf)
- [RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321)
- [HIN – HIN Gateway-produktbeskrivelse](https://www.hin.ch/de/services/hin-mail/hin-gateway.cfm)
- [HIN – Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf)
- [HIN – Stargate Deployment](https://health-info-net-ag.github.io/Stargate-deployment/de/)
- [HIN – HIN Client 3-håndbok](https://download.hin.ch/documentation/HIN_Client3_Handbuch_de_v1.1.pdf)
- [RFC 5280 – Internet X.509 PKI](https://datatracker.ietf.org/doc/html/rfc5280)
- [HIN Support – HIN-identitet](https://support.hin.ch/de/service/hin-identitaet.cfm)
- [HIN Support – SAML Authentication Context](https://support.hin.ch/de/thema/hin-access/pwd.cfm)
- [HIN – OAuth2-integrasjon](https://download.hin.ch/oauth2/doku/de/)
- [OASIS – SAML 2.0 Core](https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf)
- [RFC 6749 – OAuth 2.0](https://datatracker.ietf.org/doc/html/rfc6749)
- [HIN Support – HIN Mail og Mobile](https://support.hin.ch/de/service/hin-mail-und-mobile.cfm)
- [RFC 5322 – Internet Message Format](https://datatracker.ietf.org/doc/html/rfc5322)
- [Microsoft – Mail flow using connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow)
- [Microsoft – Mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud)
- [HIN Support – HIN Mail Token Service](https://support.hin.ch/de/service/hin-mail-und-mobile/token-service-mail-client.cfm)
- [HIN Support – POP-konfigurasjon](https://support.hin.ch/de/service/hin-mail-und-mobile/mail-clients-einrichten-pop.cfm)
- [RFC 9051 – IMAP4rev2](https://datatracker.ietf.org/doc/html/rfc9051)
- [RFC 1939 – POP3](https://datatracker.ietf.org/doc/html/rfc1939)
- [RFC 6409 – Message Submission](https://datatracker.ietf.org/doc/html/rfc6409)
- [HIN Support – HIN Client på terminalservere](https://support.hin.ch/de/thema/hin-client/hin-client-auf-terminalserver.cfm)
- [WireGuard – Protocol and Cryptography](https://www.wireguard.com/protocol/)
- [HIN Status](https://status.hin.ch/)
- [HIN Support – Brannmurtilpasninger for HIN Client](https://support.hin.ch/de/thema/hin-client/anpassung-firewall-hin-client.cfm)
- [Microsoft – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [ISC BIND – dig-manual](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc-manual](https://man.openbsd.org/nc)
- [curl – kommandolinjemanual](https://curl.se/docs/manpage.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [RFC 8446 – TLS 1.3](https://datatracker.ietf.org/doc/html/rfc8446)
- [swaks – Swiss Army Knife for SMTP](https://jetmore.org/john/code/swaks/)
- [Microsoft – Get-NetTCPConnection](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection)
- [Linux man-pages – ss](https://man7.org/linux/man-pages/man8/ss.8.html)
- [Microsoft – Packet Monitor](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon)
- [tcpdump – tcpdump(1)](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [HIN – Selskapshistorie](https://www.hin.ch/de/ueber-hin/unternehmen/geschichte.cfm)
- [HIN Support – HIN Access](https://support.hin.ch/de/thema/hin-access.cfm)
