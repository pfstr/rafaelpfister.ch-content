---
title: "Cisco Secure Email: Gateway, AsyncOS og SMA"
blatt: "cisco"
description: "Cisco Secure Email for meldingsadministratorer: SEG-/ESA-e-postpipeline, lyttere, HAT og RAT, arbeidskø og levering, e-postpolicyer, AsyncOS, SMA-tjenester, klyngegrenser, teknologistakk, overvåking, gjenoppretting og diagnose."
fakten:
  - label: Produktroller
    wert: Secure Email Gateway (SEG/ESA) · Secure Email and Web Manager (SMA)
    href: https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/datasheet-c78-742868.html
  - label: Systemrolle
    wert: SMTP-e-postgateway foran eller mellom e-postsystemer
    href: https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html
  - label: Operativsystem
    wert: Cisco AsyncOS
    href: https://www.cisco.com/c/en/us/support/security/email-security-appliance/series.html
  - label: Pipeline
    wert: Receipt → Work Queue → Delivery
    href: https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html
  - label: Godkjenning
    wert: Listener · HAT · Sender Groups · RAT · LDAP
    href: https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html
  - label: Behandling
    wert: Message Filters · Mail Policies · Content Filters · Scan-Engines
    href: https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html
  - label: Sentrale tjenester
    wert: Tracking · Reporting · Spam- og policykarantener på SMA
    href: https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0100.html
  - label: Konfigurasjonsklynge
    wert: peer-to-peer; ingen kø- eller trafikk-HA
    href: https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0101000.html
  - label: Formfaktorer
    wert: virtuell appliance · Public Cloud · Cisco Cloud Gateway
    href: https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/datasheet-c78-742868.html
  - label: Administrasjon
    wert: webgrensesnitt · CLI via SSH · REST-API
    href: https://docs.ces.cisco.com/docs/api
  - label: Primære tilstander
    wert: konfigurasjon · kø · karantener · sporing/rapportering · nøkler og sertifikater
    href: https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0100010.html
  - label: Opprinnelse
    wert: IronPort-teknologi; kjøpt av Cisco i 2007
    href: https://newsroom.cisco.com/c/r/newsroom/en/us/a/y2007/m01/cisco-announces-agreement-to-acquire-ironport.html
werbung:
  - tools
  - newsletter
ctaThemen:
  - cisco-esa-sma
translationSourceHash: c796a2c50226bbdcf5a0d7a7562913a5126b28cdbd05a5b72065afcf531bfd42
translationModel: gpt-5.6-terra
translatedAt: 2026-10-03T09:45:49.938Z
translationReview: automatic
---

# Cisco Secure Email: Gateway, AsyncOS og SMA

**Cisco Secure Email Gateway**, forkortet SEG og historisk **Email Security Appliance** eller ESA, er en tilstandsbevarende [SMTP-e-postgateway](/kb/smtp). Den avslutter innkommende SMTP-økter, avgjør om de skal godtas, behandler meldinger i en intern Work Queue og oppretter en ny SMTP-økt for levering. Den tekniske ansvarsgrensen ligger dermed ikke ved vellykket TCP- eller TLS-handshake, men ved det positive SMTP-svaret etter meldingsinnholdet: Fra dette tidspunktet må gatewayen levere eller generere en standardkonform feil ([Cisco: Email Pipeline](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html), [RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321)).

Den andre klassiske rollen er **Cisco Secure Email and Web Manager**, SMA. Den står normalt ikke som en ordinær MTA i den produktive e-postflyten. Den håndterer sentrale sporings- og rapporteringsdata samt – avhengig av design – spam-, policy-, virus- og utbruddskarantener fra flere gatewayer. En feil på SMA kan derfor la leveringen på SEG-nodene fortsette upåvirket, samtidig som søk, sluttbrukerkarantene eller behandling av tilbakeholdte meldinger avbrytes ([Cisco: SMA Message Tracking](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0100.html), [Cisco: Centralized Quarantines](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0110.html)).

Begge rollene kjører på **AsyncOS**, en programvareplattform Cisco vedlikeholder som en appliance-enhet. Administratorer administrerer ikke underliggende pakker som på en generell Linux-server; den pålitelige tekniske flaten består av AsyncOS-konfigurasjon, CLI, webgrensesnitt, REST-API, loggabonnementer, MIB-er, oppdateringskanaler og dokumenterte integrasjoner. Ciscos oversikter over åpen kildekode dokumenterer en rekke innebygde komponenter, men ingen offentlig vedlikeholdbar stykklistе over den proprietære e-postpipelinen. Enkelte biblioteker må derfor ikke sidestilles med totalarkitekturen ([Cisco: Open Source Used in AsyncOS](https://www.cisco.com/c/dam/en_us/about/doing_business/open_source/docs/CiscoEmailSecurityAppliance-AsyncOS1253-1688323737.pdf), [Cisco: AsyncOS API](https://docs.ces.cisco.com/docs/api)).

Forklaringen følger en melding gjennom Cisco Secure Email: fra Listener via HAT, RAT og Work Queue til levering. Deretter følger SMA, klyngedrift, avhengigheter, diagnose og gjenoppretting.

## Produktroller og tillitsgrenser

Et typisk lokalt design plasserer minst to SEG-noder i DMZ og én SMA i et internt administrasjonsnettverk. DNS-MX eller en foranliggende tjeneste fordeler innkommende forbindelser til gatewayene; utgående bestemmer smarthost-koblinger i e-postsystemet gatewaybanen. Flere SEG-noder er først høyt tilgjengelige når DNS, lastbalanserer eller den sendende MTA-en kan bruke alternative mål. AsyncOS-konfigurasjonsklyngen alene utfører ikke denne trafikkstyringen ([Cisco: Centralized Management Using Clusters](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0101000.html), [RFC 5321, Address Resolution](https://datatracker.ietf.org/doc/html/rfc5321)).

Cisco dokumenterer virtuelle appliances, Public Cloud-distribusjoner og en driftet Secure Email Cloud Gateway. Disse variantene deler produktbegreper, men flytter ansvar: For den virtuelle appliancen har kunden ansvar for hypervisor, nettverk, kapasitet og gjenoppretting; med Cloud Gateway leverer Cisco gatewayinfrastrukturen. **Secure Email Threat Defense** er derimot en skybasert analyse- og beskyttelsesplattform som kan integreres via gateway, journaling eller Microsoft-API. Den er verken et synonym for den lokale Work Queue eller et alternativt navn for SMA ([Cisco: Secure Email Data Sheet](https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/datasheet-c78-742868.html), [Cisco: Email Threat Defense Data Sheet](https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/secure-email-threat-defense-ds.html)).

| Rolle | I SMTP-banen | Persistент tilstand | Konsekvens ved feil |
|---|---:|---|---|
| Secure Email Gateway | ja | Kø, lokale karantener, konfigurasjon, sertifikater, logger | Godkjenning eller levering på denne noden forstyrres |
| Secure Email and Web Manager | normalt nei | Sporing, rapportering, sentrale karantener, Safe-/Blocklists, egen konfigurasjon | Synlighet og sentrale karantenetjenester påvirkes |
| Email Threat Defense | avhenger av integrasjon | Skybasert telemetri, undersøkelse og retningslinjer | Ekstra analyse eller utbedring påvirkes |
| E-postsystem | foran eller bak gatewayen | Postbokser, transportkøer, koblinger | Brukertilgang eller ende-til-ende-levering påvirkes |

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-cisco.svg?v=20260813" title="Interaktive Infografik: Cisco Secure Email mit SEG-Mailpipeline, Listener, HAT und RAT, Work Queue, Delivery, SMA-Diensten, Konfigurationscluster und Admin-Kontrollpunkten" loading="lazy">
  <a href="/images/kb-interaktiv-cisco.svg?v=20260813">Åpne den interaktive grafikken direkte</a>.
</iframe>

## Receipt: Listener, HAT og RAT

En **Listener** binder SMTP til et IP-grensesnitt og utgjør den første policygrensen. Public Listeners mottar vanligvis internettrafikk for lokale domener; Private Listeners mottar utgående meldinger fra kontrollerte nettverk. Disse rollene er konfigurasjon, ikke en iboende tillitsegenskap ved porten. En Private Listener med for bred relaytillatelse er et Open Relay, selv om den har et internt navn.

**Host Access Table**, HAT, tilordner tilkoblende verter til Sender Groups. Deres Mail Flow Policies bestemmer blant annet om en forbindelse skal godtas, avvises, begrenses eller behandles uten enkelte skanninger. **Recipient Access Table**, RAT, definerer lokale mottakerdomener for innkommende meldinger. Valgfritt kontrollerer [LDAP](/kb/ldap) konkrete mottakere under SMTP-økten eller senere i Work Queue; alternativt kan SMTP Call-Ahead spørre den etterfølgende serveren. Cisco skiller dermed mellom fire identiteter som ikke må blandes ved en feil: kilde-IP, konvoluttavsender, konvoluttmottaker og headeridentiteter ([Cisco: Email Pipeline, Incoming](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html)).

En ryddig Listener-dokumentasjon inneholder minst bind-IP og port per retning, tillatte kildenettverk, forventede EHLO-navn, TLS-modus, krav til klientsertifikat, HAT-rekkefølge, RAT-domener, mottakerkontroll, maksimal meldingsstørrelse, rate limits og bounce-profil. Rekkefølgen er særlig kritisk: En tidlig HAT-avvisning oppretter ingen Message-ID-post slik en senere godtatt melding gjør; helpdesk kan derfor ikke finne den med samme søk.

Etter SMTP-godkjenning begynner den egentlige innholds- og policybehandlingen. Rekkefølgen er viktig fordi et tidlig resultat kan påvirke senere kontroller, mottakergrupper eller leveringsbaner.

## Work Queue: rekkefølge, splintering og policy

Etter godkjenning kommer meldingen inn i **Work Queue**. Cisco dokumenterer der routing og masquerading, Message Filters, Safe-/Blocklists, Anti-Spam, Anti-Virus, Graymail, filomdømme og -analyse, Content Filters, Outbreak Filters og karantener. Rekkefølgen er en del av sikkerhetsmodellen. En policyendring virker vanligvis ikke tilbake på meldinger som allerede er lagt i kø; senere aktivering av en skanner reparerer derfor ikke automatisk en tidligere bypass ([Cisco: Email Pipeline, Work Queue](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html)).

**Message Filters** arbeider før den mottakerrelaterte Mail Policy og kan endre, arkivere, sette i karantene, bounce eller forkaste meldinger basert på konvolutt, headere, innhold, vedlegg eller forbindelsesdata. Deretter kan AsyncOS **splitte** en melding med flere mottakere: Ulike mottakerpolicyer oppretter separate Message IDs og dermed ulike sluttilstander. Én opprinnelig inject- eller ICID-verdi kan derfor forgrene seg til flere MIDs og leveringsresultater. Sporing må vise dette treet, ikke bare søke etter emne.

**Mail Policies** styrer mottaker- eller avsenderrelaterte skanne- og innholdsfiltre. Ifølge Cisco er DLP begrenset til utgående behandling. Lisenser, oppdateringsstatus for motorene og skytilkobling avgjør i tillegg hvilke kontroller som faktisk utføres. For hver policy trenger en robust test en ufarlig positiv test, en målrettet negativ test og forventet sluttilstand – levering, endring, karantene, drop eller bounce.

## Delivery: SMTP Routes, Destination Controls og kø

I Delivery-fasen velger AsyncOS rute, kildegrensesnitt og mål. **SMTP Routes** overstyrer normal MX-oppløsning for konfigurerte domener; **Destination Controls** begrenser parallelle forbindelser og mottakere per mål. Virtual Gateways kan tilby ulike kilde-IP-adresser, vertsnavn og leveringskøer. Disse innstillingene påvirker omdømme, SPF, allowlisting hos motparter og stedet der en melding venter ([Cisco: Email Pipeline, Delivery](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html), [RFC 7208](https://datatracker.ietf.org/doc/html/rfc7208)).

Utgående TLS er hop-by-hop. AsyncOS kan bruke [STARTTLS](/kb/tls) med motparter; suksess beskytter denne transportstrekningen, men sier ingenting om foregående eller påfølgende hopp. For tvungne policyer må målmønstre, sertifikatkontroll, navnerelasjon og feiloppførsel dokumenteres. Opportunistisk TLS kan falle tilbake til klartekst ved handshakefeil; en obligatorisk policy må i stedet sette i kø eller feile ([Cisco: Verify and Troubleshoot TLS Certificates](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118844-technote-esa-00.html), [RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207)).

Køalder er viktigere enn ren kølengde. Et høyt volum kan være sunt ved høy gjennomstrømning; få svært gamle meldinger peker på en vedvarende mål-, DNS-, TLS- eller policyblokkering. For hver rute skal den eldste meldingen, retryårsak, neste forsøk, målsvar og ansvarlig motpart inngå i hendelsesbildet.

Så snart ESA har videresendt eller satt en melding i karantene, flyttes deler av administratorens oversikt til SMA. Den erstatter imidlertid ikke lokale kø- og systemdata på ESA.

## SMA: sporing, rapportering og karantener

SMA samler sporings- og rapporteringsdata fra flere SEG-noder. Message Tracking kan vise sluttilstander som `Delivered`, `Dropped`, `Bounced`, `Quarantined`, `Queued`, `Processing` og `Splintered`. Det er imidlertid en avledet indeks: Hvis eksportdata mangler, en tjeneste er forsinket eller meldingen ligger utenfor oppbevaringstiden, beviser ikke et tomt treff at meldingen aldri ble behandlet. Primærbeviset er fortsatt de relevante Mail Logs og MID-kjeden på SEG ([Cisco: Tracking Messages](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0100.html)).

Spamkarantene og policy-/virus-/utbruddskarantener er separate tjenester med ulike brukere, frigivelsesprosesser og oppbevaringstider. Sentraliserte karantener lagrer meldinger på SMA bak brannmuren og kan inkluderes i standardbackupen. Ved 75, 85 og 95 prosent kapasitetsbruk genererer AsyncOS dokumenterte terskelvarsler. Hvis en sentral karantenetjeneste blir utilgjengelig, trenger driften en forhåndstestet beslutning: midlertidig sette i kø, endre til lokal behandling eller kontrollert deaktivere den tilhørende policyen ([Cisco: Centralized Quarantines](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0110.html), [Cisco: Centralizing Services](https://www.cisco.com/c/en/us/td/docs/security/esa/esa13-7-0/user_guide/b_ESA_Admin_Guide_13-7/b_ESA_Admin_Guide_12_1_chapter_0101011.html)).

## Konfigurasjonsklynge er ikke e-post-HA

AsyncOS kan koble flere gatewayer til en peer-to-peer-**konfigurasjonsklynge**. Innstillinger kan holdes på klynge-, gruppe- eller maskinnivå; det finnes ingen primær klyngenode. Medlemmene må bruke en kompatibel AsyncOS-versjon og kommuniserer via SSH eller Cluster Communication Service. Klyngen replikerer konfigurasjon, ikke aktive SMTP-økter, køinnhold, lokale karantener eller leveringsfremdrift ([Cisco: Centralized Management Using Clusters](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0101000.html)).

Derfor finnes tre separate mekanismer:

- **Trafikkfordeling:** flere MX-mål, lastbalanserer eller smarthost-failover;
- **Konfigurasjonskonsistens:** AsyncOS-klynge med klare overstyringer på klynge-, gruppe- og maskinnivå;
- **Datatilgjengelighet:** køtilstand per SEG samt sporings- og karantenedata på SMA.

Tap av en node etter positiv SMTP-godkjenning kan påvirke meldinger som bare ligger i den lokale køen. Avsenderen må ikke bare sende dem på nytt så lenge den opprinnelige leveringsstatusen er uklar; ellers oppstår duplikater. En gjenopprettingstest må derfor ikke bare laste konfigurasjonen, men følge godtatte testmeldinger gjennom en kontrollert nodefeil.

## Teknologistakk og administrasjonsflater

AsyncOS er en lukket applianceplattform. Cisco publiserer merknader om åpen kildekode for medfølgende komponenter, men ingen fullstendig kilde- eller språkplan for proprietære tjenester. Påstander som «skrevet i Python» eller «basert på FreeBSD» er ikke pålitelig driftsinformasjon uten versjonsspesifikk produsentdokumentasjon. For administratorer er følgende dokumenterbare stakk mer relevant ([Cisco: Open Source Used in AsyncOS](https://www.cisco.com/c/dam/en_us/about/doing_business/open_source/docs/CiscoEmailSecurityAppliance-AsyncOS1253-1688323737.pdf)):

| Nivå | Dokumenterbar teknologi | Driftsrelevans |
|---|---|---|
| E-posttransport | SMTP-Listener, Receipt, Work Queue, Delivery Queue | Godkjenningsgrense, policyrekkefølge, retry og bounce |
| Policy og analyse | HAT/RAT, Message Filters, Mail Policies, Content Filters, Scan-Engines | Rekkefølge, lisenser, motoroppdateringer, splintering |
| Data og søk | lokale køer/karantener; SMA-sporing, rapportering og sentrale karantener | Kapasitet, oppbevaring, backup og personvern |
| Administrasjon | HTTPS-GUI, interaktiv CLI via SSH, XML-konfigurasjon | Endring, commit, eksport, gjenoppretting og revisjon |
| Automatisering | RESTful AsyncOS API med Swagger | Rapportering, sporing og karantenetilgang; ingen ukontrollert fullkonfigurasjon |
| Telemetri | Mail Logs, andre Log Subscriptions, Syslog, Alerts, SNMP/MIB, API | Korrelasjon via ICID/MID/DCID og ressurstilstand |
| Plattform | maskinvare-, virtuell og sky-appliance | Ansvar for compute, lagring, nettverk og livssyklus |

CLI-endringer følger et transaksjonelt mønster: Kommandoer endrer først en kjørende konfigurasjon, `commit` aktiverer den, `clearchanges` forkaster den. En runbook må angi hele dialogen og konfigurasjonsmodusen; rene copy-and-paste-fragmenter er farlige på grunn av forskjeller mellom utgivelser og klynger. REST-API-et gir sikkert autentisert tilgang til rapporter, tellere, sporings- og karantenedata; det lokale Swagger-grensesnittet dokumenterer det faktisk installerte API-omfanget ([Cisco: AsyncOS API](https://docs.ces.cisco.com/docs/api), [Cisco: SEG Support Documentation](https://www.cisco.com/c/en/us/support/security/email-security-appliance/series.html)).

## Nettverks-, identitets- og tidsavhengigheter

Når e-postpipelinen er fastlagt, kan eksterne forbindelser kontrolleres. Hver rad viser hvilket system som starter en forbindelse, hva den trengs til og hvordan feil viser seg.

| Forbindelse | Vanlig port | Initiativtaker | Formål og feilbilde |
|---|---:|---|---|
| SMTP | TCP 25 | ekstern MTA, internt e-postsystem eller SEG | Godkjenning og videresending; timeout, 4xx/5xx, køvekst |
| HTTPS | TCP 443 eller konfigurert | Admin, sluttbruker eller API-klient | GUI, API, karantene; kontroller sertifikat, SSO og roller separat |
| SSH | TCP 22 eller konfigurert | Admin eller SEG-medlem | CLI og valgfri klyngekommunikasjon |
| CCS | TCP 2222 som standard, konfigurerbar | SEG-medlem | Konfigurasjonsklynge; ingen e-postflyt |
| DNS | UDP/TCP 53 | SEG/SMA | MX, A/AAAA, PTR, omdømme og oppdateringer |
| LDAP/LDAPS | TCP 389/636 | SEG/SMA | Mottakere, ruting, grupper og administratorautentisering |
| Syslog | UDP/TCP 514 eller TLS 6514 etter design | SEG/SMA | Ekstern loggtransport; fastsett modell for tap og backpressure |
| SNMP | UDP 161/162 | Overvåking eller appliance | Statusspørring og traps; foretrekk SNMPv3 |

Portnumre alene dokumenterer ingen aktiv funksjon. Tilordningene kommer fra [IANA Service Name and Port Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml); Cisco dokumenterer CCS og konfigurerbarheten i klyngekapitlet. Brannmurer bør inkludere kilde, mål, retning, protokoll, TLS- eller autentiseringskrav og forretningsformål.

[LDAP](/kb/ldap) kan forsyne mottakergodkjenning, ruting, gruppemedlemskap og administratorautentisering. Disse spørringene har ulike skjemaer, tidsavbrudd og feilkonsekvenser. Hvis Recipient Acceptance svikter, kan systemet avhengig av konfigurasjon bounce forsinket eller forkaste; en autentiseringsfeil i GUI er derfor ikke bevis på feil i SMTP-mottakerkontrollen. Tjenestekontoer, Base DNs, filtre, referral-oppførsel, sertifikatkjede og failoverrekkefølge skal dokumenteres per spørring ([Cisco: Email Pipeline, LDAP Recipient Acceptance](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html), [RFC 4511](https://datatracker.ietf.org/doc/html/rfc4511)).

DNS og korrekt tid er systemavhengigheter. MX- og vertsuppløsning styrer levering og klyngenåbarhet; PTR og omdømme påvirker klassifisering. NTP holder logg-, Received- og sporingstider korrelerbare. Cisco krever oppløsbare vertsnavn eller konsekvent brukte IP-adresser for klynger og beskriver systemtid samt NTP som del av grunnkonfigurasjonen ([Cisco: Setup and Installation](https://www.cisco.com/c/en/us/td/docs/security/esa/esa13-7-0/user_guide/b_ESA_Admin_Guide_13-7/b_ESA_Admin_Guide_12_1_chapter_010.html)).

Ved feil følges samme bane bakover: leveringsstatus, Work Queue-beslutning, Receipt-policy, Listener og nettverksavhengigheter.

## Overvåking og hendelsestriage

Det sentrale spørsmålet er: **Godtok gatewayen meldingen, behandlet den og overleverte den til hvilket hopp?** Til dette kobles forbindelses-, meldings- og Delivery-ID-er fra Mail Logs sammen. Message Tracking på SMA fremskynder søket, men erstatter ikke råloggene. Nyttige tekniske signaler er:

- Godkjenningsrate, 4xx-/5xx-svar og avviste forbindelser per Listener og Sender Group;
- Work- og Delivery-kø, alder på eldste melding samt gjentakende målsvar;
- Behandlings- og splinteringstid, Scan-Engine-feil og oppdateringsalder;
- Lokal og sentral karantenefylling, frigivelses- og slettingshendelser;
- Resource-Conservation-verdi, CPU, minne, diskbruk og kritiske varsler;
- Tilgjengelighet og latens for DNS, LDAP, SMA, oppdaterings- og skytjenester;
- Klyngekonsistens og utilsiktede Machine Overrides;
- Utløp og bruk av hvert TLS-sertifikat samt endringer i Truststore.

I **Resource Conservation Mode** begrenser AsyncOS godkjenning trinnvis slik at leveringen kan redusere etterslepet; ved ekstrem ressursmangel godtas ingen nye meldinger. Symptomet er derfor ofte synkende innkommende gjennomstrømning, mens den egentlige utløsende faktoren er en langsom målroute eller full ressurs. Cisco tilbyr status og varsler i GUI/CLI; SNMPv3 og `ASYNCOS-MAIL-MIB` muliggjør ekstern overvåking ([Cisco: Resource Conservation](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/117834-qanda-esa-00.html), [Cisco: SNMP Monitoring](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/117831-qanda-esa-00.html)).

## Backup, gjenoppretting og oppgradering

En eksportert XML-konfigurasjonsfil er nødvendig, men ingen fullstendig systembackup. Cisco dokumenterer `saveconfig`, `mailconfig` og `loadconfig`; maskerte passfraser kan ikke lastes inn igjen. Sertifikater og nøkler, klyngetilstand, Feature Keys, lokale køer, lokale karantener, SMA-data samt eksterne avhengigheter trenger egne bevis ([Cisco: System Administration](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0100010.html), [Cisco: Automated Configuration Backup](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118403-technote-esa-00.html)).

| Gjenopprettingsobjekt | Sikring eller rekonstruksjon | Akseptansetest |
|---|---|---|
| SEG-konfigurasjon | umaskert, beskyttet lagret eksport pluss dokumenterte passfraser | last på erstatningsinstans, diff og Listener-/policytest |
| Sertifikater og private nøkler | kryptert nøkkelbackup, CA-kjede og rollematrise | HTTPS- og SMTP-TLS-handshake med navnekontroll |
| lokal kø | kan normalt ikke rekonstrueres fra konfigurasjonsbackup | nodefeil med godtatt test-e-post og duplikatkontroll |
| SMA-data | SMA-backup for sporing, rapportering, karantener og lister | kontroller søk, frigivelse av test-e-post og oppbevaring |
| klynge | eksport per nivå pluss dokumenterte Machine Overrides | koble til medlemmet på nytt og kontroller konsistens |
| eksterne tjenester | DNS-, LDAP-, Syslog-, NTP-, oppdaterings- og skykonfigurasjon | syntetisk ende-til-ende-kontroll |

Oppgraderinger er appliance-migreringer. Først må målbane, kompatible mellomtilstander, hypervisor- eller skykrav, funksjonsendringer, klyngerekkefølge, ledig lagring, nedetid og rollbackgrense kontrolleres. Ciscos utgivelseskategorier GD og MD er ingen automatisk anbefaling for ethvert miljø; avgjørende er Security Advisories, supportmatrise og det testede egne policyomfanget. Supportsiden og livssyklusforklaringene hører hjemme i patchprosedyren, ikke som et statisk versjonsnummer i artikkelen ([Cisco: SEG Release Notes](https://www.cisco.com/c/en/us/support/security/email-security-appliance/products-release-notes-list.html), [Cisco: Software Lifecycle Support Statement](https://www.cisco.com/c/dam/en/us/td/docs/security/esa/lifecycle_support_statement/Secure_Email_Gateway_Software_Lifecycle_Support_Statement.pdf)).

## Diagnoseverktøy

Feilsøkingen følger meldingsveien utenfra og innover. Først kontrolleres navn og tilgjengelighet, deretter SMTP-godkjenning, pipelinehendelser, kø og eventuelt SMA-evaluering.

### DNS, MX og målopp­løsning

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Cisco-Secure-Email-DNS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName -Type MX example.ch
Resolve-DnsName seg1.example.ch -Type A,AAAA
Resolve-DnsName 192.0.2.25 -Type PTR
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig +short MX example.ch
dig +short A seg1.example.ch
dig +short AAAA seg1.example.ch
dig +short -x 192.0.2.25
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) og [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) viser MX-, fremover- og reversoppløsning. Spørringen må gjentas fra interne og eksterne resolverperspektiver; AsyncOS SMTP Routes kan overstyre det synlige MX-resultatet.

### TCP og SMTP-TLS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Cisco-Secure-Email-TLS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Test-NetConnection seg1.example.ch -Port 25 -InformationLevel Detailed
curl.exe --verbose --ssl-reqd smtp://seg1.example.ch:25
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nc -vz seg1.example.ch 25
openssl s_client -starttls smtp -connect seg1.example.ch:25 \
  -servername seg1.example.ch -showcerts
```

  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) og [`nc`](https://man.openbsd.org/nc) dokumenterer kun TCP-banen. [`curl`](https://curl.se/docs/manpage.html) og [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) ber om STARTTLS og viser handshake samt sertifikatkjede; først forventet navne- og tillitskontroll dokumenterer den konfigurerte TLS-policyen.

### Autorisert SMTP-testmelding

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Cisco-Secure-Email-SMTP-Test">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
curl.exe --verbose --ssl-reqd --url smtp://seg1.example.ch:25 `
  --mail-from test-sender@example.ch `
  --mail-rcpt test-recipient@example.net `
  --upload-file .\seg-test.eml
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
swaks --server seg1.example.ch --port 25 --tls \
  --from test-sender@example.ch --to test-recipient@example.net \
  --data seg-test.eml
```

  </div>
</div>

[`curl`](https://curl.se/docs/manpage.html) og [`swaks`](https://jetmore.org/john/code/swaks/) sender en kontrollert testmelding. Avsender, mottaker og mål må være autorisert. SMTP-sluttsvar, ICID/MID, Splinter-MIDs, policy, karantene- eller Delivery-status og faktisk ankomst skal registreres.

### Webgrensesnitt og REST-API

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Cisco-AsyncOS-API-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Invoke-WebRequest -Method Head -Uri https://sma.example.ch/
Invoke-WebRequest -Method Head -Uri https://seg1.example.ch/swagger
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
curl --head --verbose https://sma.example.ch/
curl --head --verbose https://seg1.example.ch/swagger
```

  </div>
</div>

[`Invoke-WebRequest`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest) og [`curl`](https://curl.se/docs/manpage.html) kontrollerer HTTP- og TLS-tilgjengelighet. En statuskode dokumenterer verken pålogging eller rolleautorisasjon, sporingsimport eller karantenefunksjon. Swagger-siden beskriver bare API-et til instansen som adresseres; produktive API-tester bruker en minimal skrivebeskyttet konto og lagrer ikke tokens i shellhistorikken.

### Pakkebane ved et autorisert målepunkt

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Cisco-Secure-Email-Paketerfassung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
pktmon filter remove
pktmon filter add SEG-SMTP -p 25
pktmon start --capture --pkt-size 0 --file-name seg.etl
pktmon stop
pktmon pcapng seg.etl -o seg.pcapng
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
tcpdump -ni any -s 0 -w seg.pcap 'tcp port 25 or tcp port 443'
```

  </div>
</div>

[`pktmon`](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon) og [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) ser bare trafikken ved det valgte målepunktet. En administratorklient observerer ikke automatisk banen mellom lastbalanserer, SEG, SMA og mål-MTA. Pakkedata kan inneholde SMTP-innhold før STARTTLS og personrelaterte metadata, og må beskyttes tilsvarende.

## Teknisk historie

IronPort Systems utviklet spesialiserte meldingsgatewayer og AsyncOS-produktlinjen. Cisco annonserte oppkjøpet av selskapet i januar 2007 og innlemmet e-post- og web-sikkerhetsteknologien i sin egen sikkerhetsportefølje ([Cisco: Agreement to Acquire IronPort](https://newsroom.cisco.com/c/r/newsroom/en/us/a/y2007/m01/cisco-announces-agreement-to-acquire-ironport.html)). Produktnavnene skiftet deretter fra Cisco IronPort Email Security Appliance via Cisco Email Security Appliance til **Cisco Secure Email Gateway**; historiske begreper som ESA, C-Series og M-Series er fortsatt synlige i runbooks, loggmeldinger, lisenser og dokumentasjonsstier.

Arkitekturideen forble gjenkjennelig gjennom disse omdøpingene: en spesialisert gateway med tretrinnspipelinen Receipt, Work Queue og Delivery samt et separat administrasjonssystem for aggregerte data og karantener. Senere kom virtuelle og Public Cloud-appliances, Cloud Gateway, REST-API-er og skybaserte analysetjenester. Email Threat Defense utvider porteføljen med API-, journaling- og gatewaymodeller; den endrer ikke i ettertid tilstandsgrensene til en eksisterende ESA-/SMA-installasjon ([Cisco: Secure Email Data Sheet](https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/datasheet-c78-742868.html), [Cisco: Email Threat Defense](https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/secure-email-threat-defense-ds.html)).

Navnet på en installert appliance er derfor ikke tilstrekkelig livssyklusinformasjon. Maskinvaremodell, virtuell plattform, AsyncOS-gren, aktiverte lisenser, motor- og regeloppdateringer samt avhengige skytjenester har egne livssykluser. Ciscos support-, utgivelses- og End-of-Life-sider er dynamiske driftskilder; en statisk artikkel bør lenke til dem, men ikke fastslå en angivelig varig aktuell versjonsstatus ([Cisco: SEG End-of-Life Notices](https://www.cisco.com/c/en/us/products/security/email-security-appliance/eos-eol-notice-listing.html), [Cisco: SEG Support](https://www.cisco.com/c/en/us/support/security/email-security-appliance/series.html)).

## Kilder

- [Cisco – AsyncOS User Guide: Understanding the Email Pipeline](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html)
- [IETF RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321)
- [Cisco – SMA User Guide: Tracking Messages](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0100.html)
- [Cisco – SMA User Guide: Centralized Quarantines](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0110.html)
- [Cisco – Open Source Used in Email Security Appliance AsyncOS](https://www.cisco.com/c/dam/en_us/about/doing_business/open_source/docs/CiscoEmailSecurityAppliance-AsyncOS1253-1688323737.pdf)
- [Cisco – AsyncOS API](https://docs.ces.cisco.com/docs/api)
- [Cisco – AsyncOS User Guide: Centralized Management Using Clusters](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0101000.html)
- [Cisco – Secure Email Gateway and Secure Email and Web Manager Data Sheet](https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/datasheet-c78-742868.html)
- [Cisco – Secure Email Threat Defense Data Sheet](https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/secure-email-threat-defense-ds.html)
- [IETF RFC 7208 – Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [Cisco – Verify and Troubleshoot TLS Certificates on ESA](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118844-technote-esa-00.html)
- [IETF RFC 3207 – SMTP Service Extension for Secure SMTP over TLS](https://datatracker.ietf.org/doc/html/rfc3207)
- [Cisco – Centralizing Services on a Secure Email and Web Manager](https://www.cisco.com/c/en/us/td/docs/security/esa/esa13-7-0/user_guide/b_ESA_Admin_Guide_13-7/b_ESA_Admin_Guide_12_1_chapter_0101011.html)
- [Cisco – Secure Email Gateway Support Documentation](https://www.cisco.com/c/en/us/support/security/email-security-appliance/series.html)
- [IANA – Service Name and Transport Protocol Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml)
- [IETF RFC 4511 – Lightweight Directory Access Protocol](https://datatracker.ietf.org/doc/html/rfc4511)
- [Cisco – Setup and Installation](https://www.cisco.com/c/en/us/td/docs/security/esa/esa13-7-0/user_guide/b_ESA_Admin_Guide_13-7/b_ESA_Admin_Guide_12_1_chapter_010.html)
- [Cisco – Resource Conservation Mode](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/117834-qanda-esa-00.html)
- [Cisco – SNMP Monitoring on ESA](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/117831-qanda-esa-00.html)
- [Cisco – AsyncOS User Guide: System Administration](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0100010.html)
- [Cisco – Automate Configuration Backup of a Clustered ESA](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118403-technote-esa-00.html)
- [Cisco – Secure Email Gateway Release Notes](https://www.cisco.com/c/en/us/support/security/email-security-appliance/products-release-notes-list.html)
- [Cisco – Secure Email Gateway Software Lifecycle Support Statement](https://www.cisco.com/c/dam/en/us/td/docs/security/esa/lifecycle_support_statement/Secure_Email_Gateway_Software_Lifecycle_Support_Statement.pdf)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [ISC BIND 9 – dig manpage](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc manpage](https://man.openbsd.org/nc)
- [curl – command line manpage](https://curl.se/docs/manpage.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [swaks – SMTP test tool](https://jetmore.org/john/code/swaks/)
- [Microsoft Learn – Invoke-WebRequest](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest)
- [Microsoft Learn – pktmon](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon)
- [tcpdump – manual page](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [Cisco – Agreement to Acquire IronPort](https://newsroom.cisco.com/c/r/newsroom/en/us/a/y2007/m01/cisco-announces-agreement-to-acquire-ironport.html)
- [Cisco – Secure Email Gateway End-of-Life Notices](https://www.cisco.com/c/en/us/products/security/email-security-appliance/eos-eol-notice-listing.html)
