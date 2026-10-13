---
title: "SEPPmail: e-postgateway, nøkler og GINA-levering"
blatt: "seppmail"
description: "SEPPmail for meldingsadministratorer: SMTP-inline-arkitektur, Postfix- og regelsettspipeline, S/MIME, OpenPGP og GINA, nøkkel- og sertifikattilstand, katalogintegrasjon, klynger, overvåking, sikkerhetskopiering og gjenoppretting."
fakten:
  - label: Produkt
    wert: SEPPmail Secure E-Mail Gateway
    href: https://www.seppmail.com/products/
  - label: Systemrolle
    wert: SMTP-gateway i meldingsbanen
    href: https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html
  - label: Transportkjerne
    wert: Postfix MTA
    href: https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html
  - label: Systemgrunnlag
    wert: Appliance-stakk basert på OpenBSD
    href: https://docs.seppmail.com/ch/20_revision-history.html
  - label: Meldingskryptografi
    wert: S/MIME · OpenPGP · domenenøkler
    href: https://docs.seppmail.com/ch/07_mi_07_mp_01_ep_01_addedit-encryption-policy.html
  - label: Uten mottakernøkkel
    wert: GINA-weblevering via HTTPS
    href: https://docs.seppmail.com/en/07_mi_17_gd__gina-domains.html
  - label: Regelmodell
    wert: generert regelsett · policyer · egendefinerte kommandoer
    href: https://docs.seppmail.com/ch/08_cr_01_gi_04_macros.html
  - label: Katalog
    wert: LDAP/LDAPS for nøkkel-, identitets- og regeloppslag
    href: https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html
  - label: Webgrensesnitt
    wert: GINA 443 · Admin 8443 · REST 8445
    href: https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html
  - label: Overvåking
    wert: Postmaster · SNMP · NRPE · Syslog
    href: https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html
  - label: Klyngetransport
    wert: SSH mellom appliance-er · NTP påkrevd
    href: https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html
  - label: Opprinnelse
    wert: Sveitsisk produktutvikling siden 2001
    href: https://www.seppmail.com/de/die-seppmail-story/
werbung:
  - tools
  - newsletter
ctaThemen:
  - seppmail
translationSourceHash: f01b161a8f003ca453398dd3d393ca671b7346ad63164cd311dabb11055526fe
translationModel: gpt-5.6-terra
translatedAt: 2026-10-12T11:51:56.995Z
translationReview: automatic
---

# SEPPmail: e-postgateway, nøkler og GINA-levering

SEPPmail er en spesialisert e-postgateway som mottar meldinger i SMTP-banen, behandler dem etter sentrale regler og videresender dem til et internt e-postsystem, en smarthost eller neste motpart på Internett. De dokumenterte behandlingsmetodene omfatter S/MIME- og OpenPGP-signering eller -kryptering, S/MIME-basert domene-kryptering, tilkoblingsrelaterte TLS-krav og GINA-weblevering for mottakere uten brukbar offentlig nøkkel ([SEPPmail: Products](https://www.seppmail.com/products/), [SEPPmail: Mail System](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html), [SEPPmail: Encryption Policy](https://docs.seppmail.com/ch/07_mi_07_mp_01_ep_01_addedit-encryption-policy.html)).

Den sentrale behandlingen forenkler klienten: Brukere kan sende fra sitt vanlige e-postsystem, mens gatewayen søker etter nøkler, vurderer retningslinjer og utfører kryptografiske operasjoner. Samtidig blir appliance-en dermed en tillits- og tilgjengelighetsgrense. Den ser meldingsinnholdet før utgående kryptering eller etter innkommende dekryptering, oppbevarer privat nøkkelmateriale og kan bli det nye ansvarspunktet for meldingen etter vellykket SMTP-mottak. En administrator må derfor ikke bare dokumentere «kryptering aktiv», men kunne etterspore hele banen med ruting, regel, nøkkelvalg, leveringsmetode, kø, logging og gjenoppretting ([RFC 5321: SMTP responsibility](https://datatracker.ietf.org/doc/html/rfc5321#section-6.1), [RFC 8551: S/MIME](https://datatracker.ietf.org/doc/html/rfc8551), [SEPPmail: Frontend/Backend Cluster](https://docs.seppmail.com/en/04_com_09_cl_05_frontend-backend-cluster.html)).

Forklaringen følger en melding gjennom SMTP-mottak, regelsett og valgt krypteringsmetode frem til levering eller GINA-portalen. Nøkler, klynger, diagnose og gjenoppretting forklares deretter langs denne banen.

SEPPmail ligger som en tilstandsbevarende gateway i meldingsveien. Den mottar SMTP, fastsetter policy og mottakerkapasitet, velger S/MIME, OpenPGP, TLS eller GINA og videresender deretter. Arkitektur, drift og gjenoppretting er innrettet etter denne prosessen.

## Arkitekturtilnærming: tilstandsbevarende inline-gateway

I det enkleste oppsettet sender det interne e-postsystemet utgående meldinger via [SMTP](/kb/smtp) til SEPPmail; for innkommende meldinger peker den offentlige [MX-oppføringen](/kb/dns#dns-im-mail--und-identitätsbetrieb) direkte eller via en foranliggende smarthost til gatewayen. For hvert administrert domene er det definert en intern videresendingsserver. Utgående kan appliance-en selv levere via DNS eller bruke en fastsatt SMTP-smarthost. Brannmurmatrisen dokumentert av produsenten viser disse retningene separat og angir TCP 25 mellom e-postserver, appliance, Internett eller smarthost ([SEPPmail: Setting Up The Firewall/Router](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html), [SEPPmail: Mail System](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)).

Gatewayen er ikke en transparent pakkefiltreringsenhet. Den avslutter en SMTP-økt og åpner en ny for videresending. Etter `250` ved slutten av DATA-fasen ligger leveringsansvaret hos det mottakende SMTP-systemet; en vellykket tilkobling eller `STARTTLS` alene dokumenterer ikke denne overføringen ([RFC 5321, avsnitt 4.1.1.4](https://datatracker.ietf.org/doc/html/rfc5321#section-4.1.1.4), [RFC 3207: SMTP STARTTLS](https://datatracker.ietf.org/doc/html/rfc3207)). Fra et driftsperspektiv oppstår minst fem separate kontrollobjekter: innkommende SMTP-økt, lokal behandling, neste SMTP-hopp, nøkkel- og policytilstand samt eventuelt GINA-webtilstanden.

For Exchange Online beskriver SEPPmail en eksplisitt modell for connectorer og e-postflytregler. I miljøer med flere leietakere anbefaler produsentens veiledning sertifikatbaserte connectorer; flere gatewayverter må i tillegg tas med i Microsoft-connectorene og eventuelt i antispam-konfigurasjonen. Microsoft dokumenterer også connectorer som en definert tillits- og rutingsrelasjon, ikke som en generell deaktivering av alle andre leveringsbaner ([SEPPmail: Setup with Exchange Online](https://docs.seppmail.com/en/04_com_powershell_setup-exo.html), [Microsoft: Configure mail flow using connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow)).

## Teknologistakk og varige tilstander

Den offentlig tilgjengelige revisjonshistorikken dokumenterer en appliance-stakk basert på OpenBSD med Postfix som MTA samt blant annet Apache HTTP Server, OpenSSL eller LibreSSL, GnuPG, OpenLDAP, Dovecot, Fetchmail, ClamAV, OpenSSH og Perl. Disse komponentene fyller forskjellige roller; versjonsnumrene som er angitt i en bestemt release note, er ikke tidløse produktegenskaper. For drift er det avgjørende hvilket grensesnitt produsenten støtter: SMTP- og TLS-parametere administreres for eksempel via «Mail System», mens «Extended Postfix MTA settings» uttrykkelig beskrives som risikofylt og ikke klyngesynkronisert ([SEPPmail: Revision History](https://docs.seppmail.com/ch/20_revision-history.html), [SEPPmail: Mail System – Extended Postfix settings](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)).

Den synlige e-postflyten er bare en del av tilstanden. For en robust inventering må minst følgende objekter inventariseres separat:

| Tilstandsrom | Eksempler | Hvorfor det kontrolleres separat |
|---|---|---|
| SMTP og ruting | Administrerte domener, videresendingsserver, smarthost, relénettverk, TLS-mål | avgjør mottak og neste hopp |
| Behandling | generert regelsett, Encryption Policies, Custom Commands, header- og emneutløsere | avgjør signatur, kryptering, GINA, bounce eller klartekstbane |
| Kryptografi | private S/MIME-/OpenPGP-nøkler, offentlige mottakernøkler, CA-tillit, OCSP/CRL, domenenøkler | avgjør om og for hvem en operasjon er mulig og kontrollerbar |
| GINA | domener, layout, kontoer, passord, leveringshistorikk, lagrede meldinger og nøkler | utgjør en ekstra HTTPS-, identitets- og oppbevaringsbane |
| Plattform | appliance-konfigurasjon, systemnavn, nettverk, sertifikater, lisenser, klyngerolle | avgjør tilgjengelighet, administrasjon og oppstart på nytt |
| Observerbarhet | e-post-, system- og revisjonslogger, postmastermeldinger, SNMP/NRPE/Syslog | gir dokumentasjon og varsling, men er ikke fullt inkludert i sikkerhetskopien |

Inndelingen følger den dokumenterte meny- og komponentstrukturen, GINA-kontoadministrasjonen, klyngebeskrivelsen og Quick Setup Guide. Den er en inventarmodell for administratorer, ikke en påstand om en udokumentert intern databasestruktur ([SEPPmail: Mail System](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html), [SEPPmail: GINA User Details](https://docs.seppmail.com/en/07_mi_18_ga_01_user-x.html), [SEPPmail: Frontend/Backend Cluster](https://docs.seppmail.com/en/04_com_09_cl_05_frontend-backend-cluster.html), [SEPPmail: Quick Setup Guide](https://downloads.seppmail.com/wp-content/uploads/docu/Quick-Setup-Guide-en.pdf)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-seppmail.svg?v=20260813" title="Interaktive Infografik: SEPPmail-SMTP-Pfad, Ruleset, Verschlüsselungsverfahren, GINA, Schlüsselzustand, Cluster, Monitoring und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-seppmail.svg?v=20260813">Åpne interaktiv grafikk direkte</a>.
</iframe>

## SMTP-mottak, ruting og relégrense

Et administrert domene knytter et mottatt SMTP-navnerom til den interne videresendingsserveren og andre domenespesifikke innstillinger. Gatewayen kan valgfritt kontrollere mottakeradressen mot denne serveren via SMTP-oppslag. Den utgående banen bestemmes enten av den innebygde MTA-en via DNS eller gjennom en konfigurert SMTP-server; domenespesifikke TLS- og senderbaserte rutingsinnstillinger kan overstyre denne standardbanen ([SEPPmail: Mail System – Managed Domains and Outgoing Server](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)).

«Relaying allowed» definerer hvilke IP-adresser eller nettverk appliance-en mottar meldinger til eksterne mottakere fra. Produsentens dokumentasjon advarer uttrykkelig mot å legge inn den utgående serveren generelt, fordi dette kan føre til et åpent relé. Portåpning, relétillatelse, administrert avsenderdomene og eventuelt SMTP-autentisering er derfor ulike kontroller. En positiv TCP-test sier ingenting om reléavgjørelsen etter `MAIL FROM` og `RCPT TO` ([SEPPmail: Mail System – Relaying](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html), [RFC 5321: SMTP commands](https://datatracker.ietf.org/doc/html/rfc5321#section-4.1)).

Ved flere gatewayer må den foranliggende lastbalansereren eller den sendende tjenesten kunne nå hver produksjonsvert. Samtidig må etterfølgende systemer godta alle forventede gatewayidentiteter. I Exchange Online gjelder dette connectorsertifikater eller IP-områder; i et klyngemiljø dessuten virtuelle IP-er og den dokumenterte egenskapen at mottak skjer med IP-alias, men sending skjer fra den fysiske adressen til det sendende systemet ([SEPPmail: Firewall and IP aliases](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html), [SEPPmail: Exchange Online clustering and multi-host](https://docs.seppmail.com/en/04_com_powershell_setup-exo.html)).

Etter SMTP-mottak avgjør regelsettet hvilken beskyttelses- og leveringsbane som gjelder. Denne avgjørelsen må senere kunne spores ut fra melding, identitet, katalogdata og regelversjon.

## Regelsett, policyer og dokumentasjon av beslutninger

SEPPmail genererer kjørbare behandlingsregler fra innstillingene i Ruleset Generator. Ytterligere Encryption Policies kan tvinge frem metoder som kun S/MIME, kun OpenPGP, kun GINA eller et hierarkisk valg for utvalgte administrerte domener og valgfrie flagg. Etter en policyendring må regelsettet ifølge produsenten genereres på nytt før endringen er aktiv. «Lagret i GUI-et» og «virksomt i den aktive meldingsbanen» er dermed to ulike tilstander ([SEPPmail: Add/Edit Encryption Policy](https://docs.seppmail.com/ch/07_mi_07_mp_01_ep_01_addedit-encryption-policy.html)).

Custom Commands utvider denne pipelinen med betingelser og handlinger. Produsenten dokumenterer blant annet LDAP-spørringer, flagg, bruker- og nøkkeloppretting, headeroperasjoner, signering, kryptering, levering og avvisning. Makroer utvides når regelsettet genereres; en syntaksfeil kan hindre genereringen ([SEPPmail: Macros](https://docs.seppmail.com/ch/08_cr_01_gi_04_macros.html), [SEPPmail: LDAP-based user creation](https://docs.seppmail.com/en/09_ht_user_creation-with-ldap.html)).

Header- eller emneutløsere er inndata i en sikkerhetsbeslutning. De må bare evalueres fra en pålitelig retning eller renses før evalueringen. SEPPmail-eksemplene for signering på tvers av leietakere advarer uttrykkelig mot å forutsette avsenderidentiteten ukontrollert. For hver utløser må derfor kilde, tillatt connector, autentisering, normalisering, prioritet og fail-closed-atferd dokumenteres ([SEPPmail: Signing emails between clients](https://docs.seppmail.com/en/09_ht_tenant_signing-e-mails-between-tenants.html), [SEPPmail: Internal X-header processing](https://docs.seppmail.com/en/08_cr_01_gi_06_x-header.html)).

En reproduserbar dokumentasjon av beslutningen inneholder minst: envelope-avsender og -mottaker, synlig `From:`- og eventuelt `Sender:`-header, retning og administrert domene, utløsende policy eller regel, funne nøkler, valgt metode, endelig handling og kø-ID. Dette er særlig viktig fordi S/MIME-signaturidentitet vanligvis kontrolleres mot den synlige avsenderen, mens SMTP-ruting først arbeider med envelope ([SEPPmail: Sender identity selection](https://docs.seppmail.com/en/09_ht_user_creation-based-on-other-criteria-than-from-header.html), [RFC 5322: Originator fields](https://datatracker.ietf.org/doc/html/rfc5322#section-3.6.2)).

## S/MIME, OpenPGP og TLS som separate beskyttelseslag

[S/MIME](https://datatracker.ietf.org/doc/html/rfc8551) kapsler signert eller kryptert MIME-innhold med Cryptographic Message Syntax. Identiteter og offentlige nøkler knyttes typisk gjennom X.509-sertifikater; banekontroll, gyldighetstid, nøkkelbruk og tilbakekallingsinformasjon inngår derfor i beslutningen ([RFC 5652: CMS](https://datatracker.ietf.org/doc/html/rfc5652), [RFC 5280: PKIX certificate validation](https://datatracker.ietf.org/doc/html/rfc5280), [RFC 6960: OCSP](https://datatracker.ietf.org/doc/html/rfc6960)). SEPPmail kan administrere bruker- og domenesertifikater og hente dem gjennom MPKI-grensesnitt; brannmurdokumentasjonen angir HTTPS for dette, samt HTTP eller HTTPS for OCSP/CRL ([SEPPmail: Firewall – MPKI and revocation](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html)).

[OpenPGP](https://datatracker.ietf.org/doc/html/rfc9580) bruker en annen pakke-, nøkkel- og tillitsmodell. En eksisterende offentlig nøkkel dokumenterer ikke automatisk at den tilhører den tiltenkte mottakeren eller fortsatt skal brukes. Fingeravtrykk, opprinnelse, User ID, utløp, tilbakekalling, algoritme og privat dekrypterbarhet må behandles som en livssyklus. Encryption Policy skiller mellom PGP/MIME og Inline PGP og kan konfigurere signatur og kryptering separat ([SEPPmail: Encryption Policy – OpenPGP options](https://docs.seppmail.com/ch/07_mi_07_mp_01_ep_01_addedit-encryption-policy.html), [RFC 9580: Key structures](https://datatracker.ietf.org/doc/html/rfc9580#section-5.5)).

[SMTP-TLS](/kb/tls#tls-policy-bei-smtp) beskytter en forbindelse mellom to hopp. Uten en bindende policy er STARTTLS opportunistisk: Hvis det ikke er tilgjengelig, kan den sendende MTA-en levere ukryptert, avhengig av konfigurasjonen. SEPPmail kan konfigurere TLS per mål med nivåer, fast server, sertifikatfingeravtrykk eller andre krav. Dette er verken S/MIME eller OpenPGP og beskytter ikke meldingen mot de involverte MTA-ene ([RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207), [RFC 7672: SMTP DANE](https://datatracker.ietf.org/doc/html/rfc7672), [SEPPmail: Mail System – TLS Settings](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)).

Ved gatewaykryptering ligger klartekst og private avsendernøkler innenfor SEPPmails tillitsgrense. Det muliggjør sentrale policyer, men betyr også at administratorroller, sikkerhetskopier, klyngeforbindelser, feilsøkingseksporter, logger og etterfølgende innholdsbehandling må beskyttes som om de kan berøre konfidensielle kommunikasjonsdata eller nøkler. Om en konkret eksport faktisk inneholder innhold, må verifiseres ut fra produsentens dokumentasjon og en kontrollert test.

Hvis ingen egnet mottakernøkkel eller direkte beskyttelsesbane er tilgjengelig, flytter GINA tilgangen til en nettportal. Dermed kommer webidentitet, økt, lagring og varslings-e-post i tillegg til SMTP-banen.

## GINA som webtjeneste med identitet og lagring

GINA er den dokumenterte reservebanen når en melding ikke leveres med en egnet mottakernøkkel, eller en policy uttrykkelig tvinger frem GINA. Kommunikasjonen med portalen skjer over HTTPS på TCP 443. GINA-domener styrer blant annet portalnavn, layout, konto- og passordatferd, første passordoverføring og oppbevaringsparametere ([SEPPmail: GINA Domains](https://docs.seppmail.com/en/07_mi_17_gd__gina-domains.html), [SEPPmail: Firewall – GINA](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html)).

For administratorer er GINA ikke bare et visningsformat, men en egen tjenestetilstand. En GINA-konto har status, passord- og eventuelt eksterne autentiseringsinnstillinger, brukerlogger, e-posthistorikk og en nøkkel. Produsentens dokumentasjon advarer om at sletting av kontoen også sletter nøkkelen ugjenkallelig, slik at gjenværende GINA-meldinger kan bli uleselige ([SEPPmail: GINA User Details](https://docs.seppmail.com/en/07_mi_18_ga_01_user-x.html)).

GINA-drift omfatter derfor minst:

- et stabilt, sertifikatkompatibelt offentlig vertsnavn og dets [DNS](/kb/dns)-livssyklus;
- HTTPS-sertifikat, komplett kjede, fornyelse og ekstern tilgjengelighet;
- separat overføring eller avledning av den første hemmeligheten;
- kontosperre, tilbakestilling, ekstern identitet og støtteprosess;
- oppbevarings- og sletteregler for melding, metadata, logger og nøkler;
- en gjenopprettingstest som faktisk åpner en GINA-melding opprettet før sikkerhetskopien etter gjenoppretting.

Quick Setup Guide påpeker at endring av vertsnavnet kan føre til at tidligere GINA-meldinger ikke lenger er tilgjengelige. Den dokumenterer også at en gjenoppretting til et nytt system krever passordet som ble angitt ved sikkerhetskopieringen ([SEPPmail: Quick Setup Guide](https://downloads.seppmail.com/wp-content/uploads/docu/Quick-Setup-Guide-en.pdf)). Vertsnavn, sertifikat, GINA-domene, konto og nøkkel må derfor betraktes samlet.

## Katalog og livssyklus for nøkler og sertifikater

LDAP/LDAPS kan tilby offentlige nøkler, evaluere bruker- eller gruppebetingelser og begrense automatisk konto- eller nøkkeloppretting. Appliance-en kan dessuten tilby egne LDAP-endepunkter for distribusjon av innsamlede eller interne offentlige nøkler. Nødvendige retninger og porter varierer etter rolle; den offisielle brannmurmatrisen angir 389/636 for eksterne eller interne katalogoppslag samt separate porter for de integrerte keyserver-endepunktene ([SEPPmail: Firewall – LDAP and Key server](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html), [SEPPmail: LDAP-based user creation](https://docs.seppmail.com/en/09_ht_user_creation-with-ldap.html)).

Et LDAP-oppslag er en del av meldingsbanen så snart en regel ikke kan ta en beslutning uten resultatet. Search Base, filter, objektklasse, bind-identitet, TLS-tillit, timeout, cacheatferd og atferd ved utilgjengelighet inngår derfor i driftsmodellen. Et tomt søkeresultat, timeout og et ikke-pålitelig sertifikat må ikke behandles som samme resultat. De generelle protokoll- og TLS-kravene beskrives i [LDAP-artikkelen](/kb/ldap#bind-sasl-und-tls-vertrauensgrenzen).

For nøkler og sertifikater kreves minst følgende, separat målbare livssykluser:

| Objekt | Oppretting eller innhenting | Løpende kontroll | Gjenopprettingsbevis |
|---|---|---|---|
| S/MIME-brukernøkkel | lokalt, import eller MPKI | utløp, CA-bane, tilbakekallingsstatus, adressebinding, algoritme | dekryptere gammel melding med historisk privat nøkkel |
| OpenPGP-nøkkel | lokalt, import eller keyserver | fingeravtrykk, User ID, utløp/tilbakekalling, kapabiliteter | dekryptere gammel melding med passende Secret Key |
| Domenenøkkel | Managed Domain Service eller manuell utveksling | publisert motpartstilstand, rotasjon, komplett e-postbane | teste gammel og ny domenemelding over rotasjonsgrensen |
| SMTP-/HTTPS-sertifikat | import eller lokal/offentlig CA | SAN, kjede, utløp, private-key match, peer-policy | starte tjenesten med gjenopprettet nøkkel og forventet kjede |
| CA-tillit | Root-/Intermediate-import | formål, omfang, fjerning av gamle CA-er, OCSP/CRL | kontrollteste kjent gyldig og tilbakekalt kjede |

Kontrollkriteriene følger de respektive standardene og de dokumenterte import-, policy- og signaturkontrollene. SEPPmail nevner spesielt CA-tillit, komplett kjede, avsenderbinding og etterfølgende innholdsendring som årsaker til en ugyldig S/MIME-signatur ([SEPPmail: Reasons for an invalid signature](https://docs.seppmail.com/en/09_ht_signature_reasons-for-an-invalid-signature.html), [RFC 5280](https://datatracker.ietf.org/doc/html/rfc5280), [RFC 9580](https://datatracker.ietf.org/doc/html/rfc9580)).

Siden gateway, nøkler, katalog, GINA og kø virker sammen, må høy tilgjengelighet dekke mer enn en virtuell lytter. Frontend-/backend-inndelingen avgjør hvilke tilstander som må være tilgjengelige sammen.

## Klynge, frontend/backend-separasjon og feilområder

Den generelle klyngekommunikasjonen bruker ifølge brannmurmatrisen SSH mellom appliance-er; tidssynkronisering via NTP er obligatorisk i klyngen. Matrisen nevner også virtuelle IP-er og påpeker at ved IP-alias skjer sending fra den fysiske adressen til det aktive systemet. Tillatelseslister for brannmur, smarthost og connectorer må derfor gjenspeile den faktiske avsenderbanen, ikke bare den virtuelle mottaksadressen ([SEPPmail: Firewall – Cluster and NTP](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html)).

En spesialisert frontend/backend-klynge skiller e-postbehandling og varige konfigurasjonsdata. Ifølge produsentens beskrivelse ligger det ingen konfigurasjonsdatabase på frontend; nødvendig konfigurasjon og nøkkelmateriale leveres av backend for en handling og forkastes deretter på frontend. Når Large File Transfer er aktivert, er det likevel nødvendig med ytterligere filtilstand på frontend ([SEPPmail: Frontend/Backend Cluster](https://docs.seppmail.com/en/04_com_09_cl_05_frontend-backend-cluster.html)).

Denne arkitekturen reduserer varig nøkkellagring i DMZ, men fjerner ikke alle avhengigheter. Backend, SSH-bane, navneoppløsning, tid, sertifikater og eventuelt LFT-lagring forblir feilområder. «Frontend mottar SMTP» dokumenterer verken tilgjengelig backend eller en gjennomførbar kryptooperasjon. Omvendt dokumenterer ikke en fungerende backend at virtuell IP, ekstern rute og neste SMTP-hopp fungerer.

Ikke alle innstillinger gjelder for hele klyngen. Dokumentasjonen for Mail System beskriver flere maskinspesifikke parametere og særlig Extended Postfix-innstillinger samt bestemte TLS-mål som lokalt vedlikeholdt. Før failovertester er derfor en planlagt/faktisk-matrise per node nyttig: systemnavn, IP, sertifikat, utgående identitet, TLS-ruter, Extended Settings, lokale tjenester, overvåkingsmål og forventet klyngereplikering ([SEPPmail: Mail System – cluster notes](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)).

## Overvåking, logging, sikkerhetskopiering og gjenoppretting

Postmasteradressen mottar ifølge produsenten viktige Watchdog- og Daily Report-meldinger; uten en konfigurert og faktisk tilgjengelig adresse mangler denne varslingsbanen. I tillegg dokumenterer SEPPmail SNMP over UDP 161, NRPE over TCP 5666 samt Syslog over UDP 514 eller TLS-beskyttet TCP 6514. Hvilke verdier og hendelser som overvåkes, må fastsettes per installasjon; et åpent overvåkingsgrensesnitt alene er ingen helsesjekk ([SEPPmail: Mail System – Postmaster](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html), [SEPPmail: Firewall – Monitoring and Syslog](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html)).

Et meningsfullt dashboard skiller minst mellom SMTP-mottak og -avvisninger, køalder per Next Hop, regelsettfeil, kryptosuksess og -feil, sertifikatutløp, OCSP/CRL- og MPKI-tilgjengelighet, LDAP-latens, GINA-påloggings- og leveringsfeil, disktilstand, klyngekommunikasjon og alder på sikkerhetskopi. Terskelverdier trenger sammenheng med normalt gjennomløp og avtalt RPO/RTO. En høy kø kan indikere en langsom motpart; en tom kø kan også bety at ingen meldinger lenger mottas.

Quick Setup Guide beskriver en passordbeskyttet sikkerhetskopi som sikrer appliance-data, inkludert registrerte brukere og sertifikater, men unntar logger. Passordet kreves for gjenoppretting til et nytt system. Veiledningen beskriver også nattlig levering til «Backup Operator» ([SEPPmail: Quick Setup Guide](https://downloads.seppmail.com/wp-content/uploads/docu/Quick-Setup-Guide-en.pdf)). Dette medfører tre separate kontroller:

1. Sikkerhetskopifilen må ankomme utenfor appliance-en, og alderen må overvåkes.
2. Sikkerhetskopipassordet må oppbevares separat, tilgangsbeskyttet og være tilgjengelig i en nødsituasjon.
3. Logger trenger en egen eksport- og oppbevaringsbane dersom de skal bevares for forensikk eller dokumentasjon.

En gjenoppretting er først testet når målsystemet starter, nettverk og lisenser passer, regler er aktive, SMTP fungerer i begge retninger, historisk S/MIME-/OpenPGP- og GINA-innhold kan dekrypteres og en dokumentert testtilfelle kjører den forventede policyen. Klyngereplikering erstatter ikke denne testen: Den ivaretar tilgjengelighet, ikke en uavhengig kopi mot feilkonfigurasjon, sletting eller kompromitterte nøkler.

Diagnosen følger en melding fra SMTP-mottak via regelmatch og nøkkelbeslutning frem til valgt utgang eller GINA-portal.

## Diagnoseverktøy

Feilsøkingen følger meldingen fra det offentlige navnet via SMTP og TLS til regelsettet og den valgte krypteringsbanen. Slik dokumenteres transporten først, før nøkkel- eller portalfunksjoner endres.

### DNS, MX og offentlige navn

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SEPPmail-DNS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName -Type MX example.ch
Resolve-DnsName securemail.example.ch -Type A,AAAA
Resolve-DnsName _25._tcp.securemail.example.ch -Type TLSA
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig +short MX example.ch
dig +short A securemail.example.ch
dig +short AAAA securemail.example.ch
dig +short TLSA _25._tcp.securemail.example.ch
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) og [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) kontrollerer MX, gatewayadresse og eventuelt DANE-TLSA. Ekstern og intern resolvervisning må testes separat. GINA-vertsnavnet må dessuten samsvare med sertifikatet; MX alene sier ingenting om den etterfølgende videresendingsserveren eller Exchange Online-connectoren ([RFC 5321: Address resolution and mail routing](https://datatracker.ietf.org/doc/html/rfc5321#section-5), [RFC 7672](https://datatracker.ietf.org/doc/html/rfc7672)).

### TCP-, SMTP- og STARTTLS-tilgjengelighet

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SEPPmail-SMTP- und TLS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Test-NetConnection securemail.example.ch -Port 25 -InformationLevel Detailed
Test-NetConnection securemail.example.ch -Port 443 -InformationLevel Detailed
curl.exe --verbose --ssl-reqd smtp://securemail.example.ch:25
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nc -vz securemail.example.ch 25
nc -vz securemail.example.ch 443
openssl s_client -starttls smtp -connect securemail.example.ch:25 \
  -servername securemail.example.ch -showcerts
```

  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) og [`nc`](https://man.openbsd.org/nc) dokumenterer bare TCP-tilgjengelighet. Windows-[`curl.exe`](https://curl.se/docs/manpage.html) kan be om STARTTLS; [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) viser SMTP-kapabiliteter, handshake og sertifikatkjede på Unix-systemer. For en kontrollert SMTP-transaksjon med envelope og DATA egner [`swaks`](https://jetmore.org/john/code/swaks/) seg; den må bare brukes mot autoriserte testmottakere. En leveringstest er først komplett med kø-ID, logghendelse og ankomst ved neste hopp.

### GINA-HTTPS og sertifikatnavn

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SEPPmail-GINA-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Invoke-WebRequest -Method Head -Uri https://securemail.example.ch/
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
curl --head --verbose https://securemail.example.ch/
openssl s_client -connect securemail.example.ch:443 \
  -servername securemail.example.ch -showcerts
```

  </div>
</div>

[`Invoke-WebRequest`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest) og [`curl`](https://curl.se/docs/manpage.html) kontrollerer HTTP-tilgjengelighet og responsatferd; [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) synliggjør den forhandlede TLS-økten og presentert kjede. En `200`, `301` eller `302` dokumenterer fortsatt verken vellykket GINA-pålogging eller tilgang til en historisk melding. Det krever en syntetisk brukerflyt med registrering, pålogging, lesing, svar og tilbakestilling i henhold til det konfigurerte GINA-domenet.

### Pakkebane ved et autorisert målepunkt

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SEPPmail-Paketerfassung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
pktmon filter remove
pktmon filter add SEPPmail-SMTP -p 25
pktmon start --capture --pkt-size 0 --file-name seppmail.etl
pktmon stop
pktmon pcapng seppmail.etl -o seppmail.pcapng
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
tcpdump -ni any -s 0 -w seppmail.pcap 'tcp port 25 or tcp port 443'
```

  </div>
</div>

[`pktmon`](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon) og [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) registrerer bare trafikk som passerer det valgte målepunktet. På en administrasjonsklient er dette ikke automatisk e-posttrafikken mellom e-postsystem og gateway; det kreves en autorisert vert, portspeiling eller et annet egnet observasjonspunkt. SMTP før STARTTLS kan inneholde personopplysninger, HTTPS og STARTTLS gir i det minste metadata. Innsamling, tilgang, oppbevaring og sletting må derfor inngå i incident-runbooken.

## Teknisk historie

SEPPmail beskriver selskapets opprinnelse i 2001: Grunnlegger Stefan Klein utviklet først en løsning for kryptert kommunikasjon mellom et advokatfirma i Zürich og dets klienter. Dette behovet førte til det senere produktselskapet ([SEPPmail: Die SEPPmail-Story](https://www.seppmail.com/de/die-seppmail-story/)).

Den offentlig dokumenterte produkthistorien viser utviklingen fra en sentral krypteringsgateway til en modulær appliance: Historiske release notes nevner allerede i 2007 en intern CA/RA, S/MIME-domene-kryptering, webmail-funksjoner, antispam/antivirus og en begrenset sikkerhetskopibruker. Senere generasjoner la blant annet til policyer for flere leietakere, REST-grensesnitt, MPKI-connectorer, Large File Transfer, DKIM/ARC, moderne S/MIME-metoder og spesialiserte klyngeroller ([SEPPmail: Revision History](https://docs.seppmail.com/ch/20_revision-history.html)).

Også underlaget utviklet seg videre. Revisjonshistorikken nevner gjennom årene OpenBSD, Postfix, Apache, Perl, GnuPG, OpenLDAP, OpenSSL/LibreSSL, Dovecot, Fetchmail og ClamAV. For administratorer er denne bakgrunnen relevant fordi loggbegreper, MTA-parametere, sertifikatformat og nettverksatferd stammer fra disse komponentene. Direkte inngrep under produsentgrensesnittet støttes likevel ikke automatisk av den grunn; dokumentasjonen advarer særlig mot Extended Postfix-endringer og manglende klyngesynkronisering av dem ([SEPPmail: Mail System](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)).

## Kilder

- [SEPPmail – Products](https://www.seppmail.com/products/)
- [SEPPmail – Mail System](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)
- [SEPPmail – Add/Edit Encryption Policy](https://docs.seppmail.com/ch/07_mi_07_mp_01_ep_01_addedit-encryption-policy.html)
- [RFC 5321: SMTP responsibility](https://datatracker.ietf.org/doc/html/rfc5321)
- [IETF RFC 8551 – S/MIME 4.0 Message Specification](https://datatracker.ietf.org/doc/html/rfc8551)
- [SEPPmail – Frontend/Backend Cluster](https://docs.seppmail.com/en/04_com_09_cl_05_frontend-backend-cluster.html)
- [SEPPmail – Setting Up The Firewall/Router](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html)
- [IETF RFC 3207 – SMTP Service Extension for Secure SMTP over TLS](https://datatracker.ietf.org/doc/html/rfc3207)
- [SEPPmail – Setup with Exchange Online](https://docs.seppmail.com/en/04_com_powershell_setup-exo.html)
- [Microsoft – Configure mail flow using connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow)
- [SEPPmail – Revision History](https://docs.seppmail.com/ch/20_revision-history.html)
- [SEPPmail – GINA User Details](https://docs.seppmail.com/en/07_mi_18_ga_01_user-x.html)
- [SEPPmail – Quick Setup Guide](https://downloads.seppmail.com/wp-content/uploads/docu/Quick-Setup-Guide-en.pdf)
- [SEPPmail – Macros](https://docs.seppmail.com/ch/08_cr_01_gi_04_macros.html)
- [SEPPmail – LDAP-based user creation](https://docs.seppmail.com/en/09_ht_user_creation-with-ldap.html)
- [SEPPmail – Signing Emails Between Clients](https://docs.seppmail.com/en/09_ht_tenant_signing-e-mails-between-tenants.html)
- [SEPPmail – Internal X-header processing](https://docs.seppmail.com/en/08_cr_01_gi_06_x-header.html)
- [SEPPmail – Sender identity selection](https://docs.seppmail.com/en/09_ht_user_creation-based-on-other-criteria-than-from-header.html)
- [RFC 5322: Originator fields](https://datatracker.ietf.org/doc/html/rfc5322)
- [IETF RFC 5652 – Cryptographic Message Syntax](https://datatracker.ietf.org/doc/html/rfc5652)
- [IETF RFC 5280 – Internet X.509 PKI Certificate and CRL Profile](https://datatracker.ietf.org/doc/html/rfc5280)
- [IETF RFC 6960 – Online Certificate Status Protocol](https://datatracker.ietf.org/doc/html/rfc6960)
- [IETF RFC 9580 – OpenPGP](https://datatracker.ietf.org/doc/html/rfc9580)
- [IETF RFC 7672 – SMTP Security via DANE](https://datatracker.ietf.org/doc/html/rfc7672)
- [SEPPmail – GINA Domains](https://docs.seppmail.com/en/07_mi_17_gd__gina-domains.html)
- [SEPPmail – Reasons For An Invalid Signature](https://docs.seppmail.com/en/09_ht_signature_reasons-for-an-invalid-signature.html)
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
- [SEPPmail – Die SEPPmail-Story](https://www.seppmail.com/de/die-seppmail-story/)
