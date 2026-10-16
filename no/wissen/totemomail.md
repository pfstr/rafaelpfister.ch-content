---
title: "Totemomail og Kiteworks EPG: e-postgateway og plattformbytte"
blatt: "totemomail"
description: "Totemomail og etterfølgeren Kiteworks Email Protection Gateway for meldingsadministratorer: SMTP-inline-arkitektur, policyer, S/MIME, OpenPGP og TLS, WebMail, Java-kryptografistakk, nøkkeltilstand, identitet, lisenser, migrering, høy tilgjengelighet, overvåking og gjenoppretting."
fakten:
  - label: Produktlinje
    wert: totemomail → Kiteworks Email Protection Gateway
    href: https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf
  - label: Systemrolle
    wert: policybasert krypteringsgateway for e-post
    href: https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf
  - label: Transport
    wert: SMTP i meldingsbanen
    href: https://marketplace.microsoft.com/en/marketplace/apps/totemo.totemo-azr-tm7?tab=overview
  - label: Metoder
    wert: S/MIME · OpenPGP · TLS
    href: https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf
  - label: Uten mottakernøkkel
    wert: WebMail- eller portallevering
    href: https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf
  - label: Avgjørelse
    wert: sentrale policyer og automatisk nøkkelvalg
    href: https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf
  - label: Eldre kryptokjerne
    wert: Totemo Security Platform · Java-basert TCM
    href: https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/3500
  - label: EPG-plattform
    wert: Kiteworks Hardened Virtual Appliance
    href: https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf
  - label: EPG-datalagring
    wert: integrert database som kunden ikke administrerer direkte
    href: https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf
  - label: Observerbarhet
    wert: dokumentasjon av e-postflyt · Syslog · plattformrevisjon
    href: https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf
  - label: EPG-driftsformer
    wert: lokalt · egne skyressurser · Single-Tenant-hosting
    href: https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf
  - label: Opprinnelse
    wert: totemo AG, Sveits · grunnlagt i 2001 · overtatt av Kiteworks i 2022
    href: https://www.kiteworks.com/sites/default/files/resources/on-the-hunt-kiteworks-secure-content-strategy.pdf
werbung:
  - tools
  - newsletter
ctaThemen:
  - totemomail
translationSourceHash: d949453b9a80abb64f76a8ecdee768813548ae2987b18b071cc23d33b0c5df21
translationModel: gpt-5.6-terra
translatedAt: 2026-10-15T08:57:32.886Z
translationReview: automatic
---

# Totemomail og Kiteworks EPG: e-postgateway og plattformbytte

Totemomail var en produktfamilie fra det sveitsiske selskapet totemo AG for sentral e-postkryptering. Etter oppkjøpet av Kiteworks ble koden, ifølge produsenten, integrert i Kiteworks-plattformen og videreført som **Email Protection Gateway**, EPG. Navnene betegner derfor ikke to fullstendig uavhengige produkter, men heller ikke samme driftsmiljø: En eksisterende Totemomail-appliance og en EPG i Kiteworks Hardened Virtual Appliance har ulike grenser for plattform, database, administrasjon og gjenoppretting ([Kiteworks: FAQ for totemomail customers](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf), [Kiteworks: EPG capability brief](https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf)).

Faglig sett er kjerneoppgaven fortsatt en e-postgateway: Den behandler e-post etter sentrale retningslinjer og bruker S/MIME, OpenPGP eller TLS; for mottakere uten egnede nøkler finnes en WebMail- eller portalløsning. Produsentdokumentasjonen beskriver automatisk nøkkel- og sertifikatadministrasjon samt bruk med vanlige e-postklienter uten EPG-spesifikt klientprogramtillegg som grunnmodell ([Kiteworks: Enterprise packaging and EPG features](https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf), [Microsoft Marketplace: totemomail Encryption Gateway](https://marketplace.microsoft.com/en/marketplace/apps/totemo.totemo-azr-tm7?tab=overview)).

For meldingsadministratorer er produktlinjen først og fremst en kjede av tillitsgrenser. Gatewayen mottar en melding via SMTP, klassifiserer retning og identiteter, bestemmer en policy, søker etter nøkler eller en reservekanal, endrer eventuelt MIME-innholdet og overleverer meldingen til neste hop. Ved WebMail kommer egen tilstand for web, konto, nøkler og lagring i tillegg. Utsagnet «meldingen er kryptert» er ufullstendig uten angitt metode, beskyttet strekning, nøkkelholder og tidspunkt ([RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321), [RFC 8551](https://datatracker.ietf.org/doc/html/rfc8551), [RFC 9580](https://datatracker.ietf.org/doc/html/rfc9580)).

Forklaringen følger en melding gjennom EPG-regler, nøkkelvalg og krypteringsmetoder frem til direkte levering eller WebMail. Byttet fra Totemomail til Kiteworks, drift, diagnose og gjenoppretting behandles deretter separat.

Totemomail og Kiteworks EPG ligger som en inline-gateway i SMTP-banen. En melding mottas, tilordnes en identitet og en policy, beskyttes med en egnet metode og leveres deretter via SMTP eller portal. Plattformbyttet endrer implementering og drift, ikke denne grunnoppgaven.

## Arkitekturtilnærming: Inline-gateway med sentral policy

I gatewaydrift sender det interne e-postsystemet utgående meldinger via [SMTP](/kb/smtp) til Totemomail eller EPG. Innkommende meldinger når gatewayen direkte via MX eller gjennom et foranliggende e-postsikkerhetssystem. Gatewayen avslutter SMTP-økten, behandler meldingen og åpner en ny økt for videresending. En vellykket TCP- eller TLS-handshake er ennå ikke meldingsmottak; ansvarsövergangen skjer først med det positive SMTP-svaret på slutten av DATA-fasen ([RFC 5321, avsnitt 4.1.1.4](https://datatracker.ietf.org/doc/html/rfc5321#section-4.1.1.4), [RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207)).

Den offentlige EPG-produktbeskrivelsen skiller mellom flere beskyttelsesbaner. «Encryption Gateway» dekker ekstern gatewaykryptering og -dekryptering. «Internal Encryption» beskytter meldinger på vei gjennom og ved lagring i en skybasert e-postplattform. Andre dokumenterte komponenter er Directory Proxy, Application Integration Proxy og en kobling for RMS eller Microsoft Information Protection. Hvilke av disse rollene som er lisensiert og faktisk aktivert, må fremgå av egen konfigurasjon og kontrakt; produktnavnet alene dokumenterer dem ikke ([Kiteworks: Enterprise packaging and licensing](https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf)).

For Microsoft 365 forblir transportintegrasjonen et connector-spørsmål. Exchange Online må rute meldinger målrettet til gatewayen og akseptere returtrafikk basert på en pålitelig motpartsidentitet. En MX foran en tredjepartsgateway endrer dessuten hvilken kilde-IP Exchange Online ser for spam- og spoofingvurdering. Microsoft dokumenterer connectorer og Enhanced Filtering for dette; begge er atskilt fra EPG-kryptopolicyen ([Microsoft: Configure mail flow using connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow), [Microsoft: Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)).

## Teknologistakk og dokumenterbare produktgrenser

For den historiske Totemomail-plattformen er den kryptografiske kjernen godt offentlig dokumentert. NIST-CMVP-dokumentasjonen beskriver **Totemo Cryptographic Module**, TCM, som et felles Java-bibliotek for Totemo Security Platform, TSP. Valideringen fra 2019 nevner Totemo Appliance OS 2.3, JRE 8 og en programvare-kryptomodul med AES, RSA, ECDSA, SHA-familien, HMAC og tilfeldighetsgenerator; oppføringen har nå historisk status og er derfor ikke dokumentasjon for et i dag tillatt anskaffelsesvalg eller en konkret installert algoritmemodus ([NIST CMVP Certificate 3500](https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/3500), [Totemo TCM Security Policy](https://csrc.nist.gov/CSRC/media/projects/cryptographic-module-validation-program/documents/security-policies/140sp3500.pdf)).

NIST-sikkerhetspolicyen avgrenser uttrykkelig TCM som en komponent i TSP. Den dokumenterer ikke en fullstendig Totemomail-e-postflyt og heller ikke en universelt gyldig implementering av kø, database eller MTA. Slike detaljer må derfor ikke overføres fra en fremmed Apache-James-installasjon eller én enkelt feltobservasjon til alle Totemomail- eller EPG-versjoner. For et driftsdokument er eksportert produktkonfigurasjon, supportmatrise, pakke- eller tjenesteinventar og produsentmanualen for den faktisk installerte generasjonen avgjørende.

For EPG flyttes grensen til Kiteworks Hardened Virtual Appliance. Migrations-FAQ-en beskriver at operativsystem, webserver og database er inkludert i appliance-en, og at kundeadministratorer ikke får direkte tilgang til operativsystemet eller databasen. Oppdateringer installeres som en testet appliance-enhet. Dette reduserer kundens drift av enkeltkomponenter, men gjør appliance-prosedyrer for sikkerhetskopi, klynge, oppdatering og support til den primære kontrollflaten ([Kiteworks: FAQ for totemomail customers](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf)).

| Tilstandsrom | Eldre Totemomail | Kiteworks EPG | Administrativt bevis |
|---|---|---|---|
| SMTP og ruting | Gateway-, MTA- og målkonfigurasjon for installasjonen | EPG pluss Kiteworks-nettverks- og rollemodell | Testmelding, SMTP-svar, kø-/e-postflyt-ID, neste hop |
| Policy | Totemomail-regler og utløsere | integrert EPG-policy pluss plattformkontroller | eksportert policy, prioritet, aktivert status, testmatrise |
| Nøkler | TSP/TCM, sertifikater og OpenPGP-nøkler | migrert EPG-nøkkel- og sertifikatbeholdning | fingeravtrykk, innehaver, utløp, historikk, dekrypteringstest |
| Weblevering | Totemomail WebMail og mottakerkontoer | EPG-portal i Kiteworks-plattformen | registrering, innlogging, oppbevaring, åpning av gammel melding |
| Database | installasjonsavhengig ekstern eller lokal database | integrert database uten direkte administratortilgang | sikkerhetskopieringsjobb, gjenopprettingsprosedyre, produsentens supportvei |
| Logger | Produkt-, e-postflyt- og systemlogger | normaliserte plattform- og EPG-logger, Syslog-eksport | korrelerbar meldings-/hendelses-ID, tidsgrunnlag, SIEM-mottak |

Sammenligningen følger produsentens migrerings-FAQ og publiserte EPG-funksjonsbeskrivelser. Der Totemomail-detaljer er installasjonsavhengige, står dette bevisst som et kontrollpunkt i stedet for en oppdiktet produktegenskap ([Kiteworks: totemomail-to-EPG migration FAQ](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf), [Kiteworks: Top reasons for EPG](https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-totemomail.svg?v=20260813" title="Interaktive Infografik: Totemomail- und Kiteworks-EPG-Mailflow, Policy, Schlüssel, WebMail, Plattformmigration, Monitoring und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-totemomail.svg?v=20260813">Åpne den interaktive grafikken direkte</a>.
</iframe>

## SMTP-e-postflyt og sløyferisiko

En gatewaybane trenger entydig ansvar i hver retning. Utgående må e-postsystemet ikke levere valgfritt direkte eller via EPG dersom bare gatewaybanen utfører påkrevd policy. Innkommende må det være klart om MX, foranliggende filtre eller EPG autentiserer den opprinnelige avsenderen, og hvilket hop som når den interne e-postserveren. Flere toveis connectorer uten klar mottaker- og domeneautoritet kan få meldinger til å sirkulere mellom systemer ([RFC 5321: Trace information and loop detection](https://datatracker.ietf.org/doc/html/rfc5321#section-6.3), [Microsoft: Mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud)).

For hver produksjonsrute bør administratorer dokumentere følgende verdier samlet:

- kildenettverk eller sertifikatbasert peer-identitet og forventet EHLO-navn;
- lokal mottaksadresse, port, STARTTLS- og klientsertifikatkrav;
- autoritativt, relay- eller split-domain-ansvar;
- policyretning, bruker- og mottakerklassifisering;
- neste hop, DNS- eller smarthost-oppløsning og TLS-krav;
- oppførsel når policy-, nøkkel-, katalog- eller portalkomponenten ikke er tilgjengelig;
- maksimalt antall hop, køalder samt ansvar for retry og bounce.

SMTP-standarden krever at Received-linjer legges til ved hver videresending, og beskriver sløyfegjenkjenning basert på sporingsbanen. For diagnose og revisjon må gatewayens hendelses-ID kunne korreleres med Internet `Message-ID`, envelope, tidsvindu og etterfølgende spor. Et synlig emne er ingen stabil identifikator og kan endres av policy eller weblevering ([RFC 5321, avsnitt 4.4](https://datatracker.ietf.org/doc/html/rfc5321#section-4.4), [RFC 5322: Message-ID](https://datatracker.ietf.org/doc/html/rfc5322#section-3.6.4)).

Etter SMTP-mottak avgjør policy engine basert på avsender, mottaker, katalog og tilgjengelige nøkler hvilken beskyttelsesbane som er tillatt og mulig.

## Policy engine og identitetsavgjørelse

Kiteworks beskriver EPG som et automatisk, policystyrt beslutningslag. Regler kan, avhengig av lisensiert funksjonsomfang, baseres på domener, brukere, steder, innholdsfunksjoner eller Microsoft Information Protection-etiketter. Automatisering fjerner likevel ikke behovet for å stole på inndataene: En avsenderheader, en emneutløser eller en etikett satt utenfra må ikke ukontrollert få samme effekt som en autentisert intern identitet ([Kiteworks: EPG capability brief](https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf), [Kiteworks: totemo and Kiteworks better together](https://www.kiteworks.com/sites/default/files/resources/kiteworks-product-brief-top-5-reasons-totemo-kiteworks-better-together.pdf)).

En robust regelmodell skiller mellom fire spørsmål:

1. **Retning:** internt til eksternt, eksternt til internt, internt til internt eller applikasjon til eksternt.
2. **Identitet:** SMTP-envelope, synlig `From:`, autentisert økt, connector, katalogobjekt og tenant er ikke det samme.
3. **Handling:** signere, kryptere, omkryptere, portallevering, videresende uendret, midlertidig avvise eller permanent bounce.
4. **Feilbane:** manglende nøkkel, ugyldig kjede, utilgjengelig katalog, policyfeil og utilgjengelig neste hop krever separate resultater.

Før endringer bør en liten testmatrise med nøyaktig én avsender og mottaker per identitetsklasse gjennomføres. For hver rad dokumenteres forventet regel, metode, synlig avsender, mottakerkanal, loggkjennetegn og feiloppførsel. Først når disse resultatene er reproduserbare, er policyen ikke bare konfigurert, men dokumentert.

## Beskyttelsesmetoder og omkryptering

[S/MIME](https://datatracker.ietf.org/doc/html/rfc8551) bruker CMS og X.509-sertifikater. Gatewayen trenger en privat nøkkel for signatur eller dekryptering og et egnet offentlig sertifikat for mottakerkryptering. Sertifikatbane, navn eller e-postadresse, Key Usage, gyldighet og tilbakekallingsinformasjon inngår i kontrollen ([RFC 5652](https://datatracker.ietf.org/doc/html/rfc5652), [RFC 5280](https://datatracker.ietf.org/doc/html/rfc5280), [RFC 6960](https://datatracker.ietf.org/doc/html/rfc6960)).

[OpenPGP](https://datatracker.ietf.org/doc/html/rfc9580) bruker en selvstendig modell for nøkler, signaturer og pakker. Fingeravtrykk, User ID, funksjoner, utløp og tilbakekalling må inventariseres uavhengig av X.509. En automatisk innlært offentlig nøkkel er bare så tillitsverdig som den dokumenterte prosessen for innlæring og bekreftelse. Produktdokumentasjonen nevner S/MIME og OpenPGP side om side, men gjør dem ikke til utskiftbare tillitsmodeller ([Kiteworks: Enterprise EPG feature matrix](https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf)).

[SMTP-TLS](/kb/tls#tls-policy-bei-smtp) beskytter én enkelt transportforbindelse. Ved opportunistisk STARTTLS kan leveringen falle tilbake til klartekst uten ytterligere policy. En bindende peer-identitet kan, avhengig av miljøet, fastsettes gjennom sertifikatkontroll, DANE eller MTA-STS; dette beskytter likevel ikke meldingen mot de involverte MTA-ene ([RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207), [RFC 7672](https://datatracker.ietf.org/doc/html/rfc7672), [RFC 8461](https://datatracker.ietf.org/doc/html/rfc8461)).

Ved omkryptering dekrypterer gatewayen en innkommende metode og oppretter en ny for neste delstrekning. Dette muliggjør innholdskontroll eller protokollbytte, men skaper et klartekstøyeblikk innenfor gatewaygrensen. EPG-funksjonskatalogen beskriver uttrykkelig scenarioer der internt beskyttede meldinger kort dekrypteres ved gatewayen for å kontrolleres eller konverteres til mottakerens standard ([Kiteworks: Internal Encryption and Encryption Gateway](https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf)). DLP, antivirus, midlertidige filer, minnedumper, feilsøkingslogger og administratortilgang hører da til kryptogrensen.

Hvis en direkte brukbar mottakernøkkel mangler, flytter portallevering klarteksttilgangen til en webapplikasjon. Dermed blir webidentitet, økt og lagret melding en del av driftsmodellen.

## WebMail og portallevering

Den offisielle migrerings-FAQ-en bekrefter at EPG tilbyr en funksjon tilsvarende Totemomail WebMail for mottakere uten egnet e-postkryptering. Offentlige operatørveiledninger for Totemomail viser den typiske brukerflyten: Varsling, førstegangspassord eller registreringslenke, passordendring, innlogging på portalen og valg mellom WebMail, S/MIME eller OpenPGP ([Kiteworks: Migration FAQ, spørsmål 7](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf), [L-Bank: Sikker e-postkommunikasjon](https://www.l-bank.info/allgemein/rechtliches/e-secure/sichere-e-mail-kommunikation.html)).

Portallevering er en egen lagrings- og identitetsbane. Varslings-e-posten inneholder ikke nødvendigvis den konfidensielle meldingen; innholdet kan ligge på portalen eller i en passordbeskyttet envelope. For administratorer er derfor minst vertsnavn, TLS-sertifikat, kontostatus, passord- og tilbakestillingsprosedyre, oppbevaringsfrist, meldingsnøkler, leveringshistorikk og slettingsprosess relevant. Valgt variant må fremgå av egen policy; eksterne operatørveiledninger dokumenterer mulige Totemomail-funksjoner, men ingen globale standardinnstillinger.

En helsecheck av portalen må ikke stanse ved HTTP 200. Den syntetiske testen må omfatte registrering eller eksisterende testkonto, innlogging, åpning av en forhåndskjent melding, nedlasting av et vedlegg, svar og utlogging. Etter en gjenoppretting må det særlig kontrolleres om eldre portalinnhold fortsatt kan leses med sine historiske nøkler.

## Katalog og nøkler med lisenstelling

Kiteworks dokumenterer LDAP/AD-integrasjon, roller og sentrale policyer på plattformnivå. For en gateway kan katalogen klassifisere avsendere og mottakere, levere gruppebetingelser og knytte identiteter til sertifikater eller nøkler. Så snart en regel avhenger av et katalogoppslag, hører [LDAP](/kb/ldap#bind-sasl-und-tls-vertrauensgrenzen)-endepunkt, Search Base, filter, bind-konto, TLS-tillit, timeout, cache og failover til meldingsbanen ([Kiteworks: security and integration overview](https://www.kiteworks.com/sites/default/files/resources/accellion-brochure-external-file-sharing-security-and-governance-platform_redirected_20191210.pdf), [RFC 4511](https://datatracker.ietf.org/doc/html/rfc4511), [RFC 4513](https://datatracker.ietf.org/doc/html/rfc4513)).

Nøkkelbeholdningen er ikke et biprodukt av brukerimporten. Det må inventariseres private bruker- og domenenøkler, offentlige mottakernøkler, CA- og mellomliggende sertifikater, tillitsstatus, tilbakekallingsinformasjon, keyserver- eller MPKI-forbindelser og tilordning til historiske meldinger. En opprydding i katalogen må aldri ukontrollert fjerne en privat nøkkel som trengs for gamle meldinger.

Et publisert Kiteworks-dokument om paketering definerte EPG-seter slik at en intern bruker forbruker et sete ved sending eller mottak av en kryptert melding eller ved sending av en signert melding; interne brukere som passerte ukryptert og usignert, ble behandlet annerledes der. Dokumentet er en datert kontraktsbeskrivelse, ikke en tidløs lisensgaranti ([Kiteworks: Enterprise 8.0 licensing and packaging](https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf)). For drift må derfor tre verdier føres separat: teknisk kjente identiteter, faktisk kryptografisk aktive identiteter og kontraktsmessig lisensiert metrikk for egen bestilling.

Ved overgangen til EPG må derfor ikke bare SMTP-ruter, men også regler, identiteter, nøkler, portaltilstander og bevis overtas eller bevisst bygges opp på nytt.

## Migrering fra Totemomail til EPG

Kiteworks beskriver ikke migreringen som en in-place-omdøping. En automatisert overføring kopierer informasjon fra den eksisterende Totemomail-databasen til EPG-databasen. Den gamle eksterne databasen skal avvikles etter at Totemomail-systemet tas ut av drift, fordi den deretter ikke lenger er synkronisert. EPG inneholder selv database, operativsystem og webtjenester i Hardened Virtual Appliance ([Kiteworks: Migration FAQ](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf)).

Et fullstendig migreringsomfang består minst av:

- domener, ruter, relay- og connectoridentiteter samt TLS-policyer;
- aktive regler, prioriteter, utløsere, unntak og feilbaner;
- S/MIME-, OpenPGP-, domene-, TLS- og CA-nøkler inkludert historikk;
- WebMail-kontoer, portalmeldinger, oppbevaring, maler og offentlig navn;
- integrasjoner for katalog, SSO, SIEM, Syslog, DLP, arkiv og applikasjoner;
- roller, administratorer, revisjons- og supporttilganger;
- lisensmetrikk, aktiv brukerbestand og kontraktsgrenser;
- sikkerhetskopi, gjenoppretting, HA, DNS, overvåking og driftsprosedyrer.

Aksepttesten må kontrollere historiske data i tillegg til nye e-poster. For hver gammel krypteringstype åpnes eller dekrypteres en melding opprettet før migreringen. For hver retning sendes en ny testmelding med og uten tilgjengelig mottakernøkkel. I tillegg kontrolleres negative tilfeller som ugyldig sertifikat, utilgjengelig neste hop og utilgjengelig katalog. Først deretter kan den gamle gatewayen fjernes fra DNS, connectorer, tillatlister og overvåking.

## Høy tilgjengelighet med sikkerhetskopi og gjenoppretting

EPG-migrerings-FAQ-en nevner integrert klynging, sikkerhetskopi/gjenoppretting, disaster recovery, høy tilgjengelighet og distribusjon som egenskaper ved Hardened Virtual Appliance. Den sier imidlertid ikke at hver enkelt EPG-funksjon automatisk får samme RPO og RTO. E-postkø, database, nøkler, portalinnhold, revisjonslogg, konfigurasjon og ekstern objekt- eller fillagring må tilordnes separat avhengig av arkitekturen ([Kiteworks: Migration FAQ – Hardened Virtual Appliance](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf)).

| Gjenopprettingsobjekt | Minimumsbevis | Typisk gap |
|---|---|---|
| Gatewaykonfigurasjon | eksportert eller sikret ønsketilstand og gjenoppretting på erstatningssystem | konfigurasjon finnes, men hemmeligheter eller sertifikater mangler |
| Private nøkler | fingeravtrykksliste før og etter gjenoppretting; historisk dekryptering | gjeldende nøkler finnes, gammel rotasjon er tapt |
| Policy | aktivert status pluss positiv og negativ testmatrise | eksporten samsvarer ikke med den faktisk aktive regelen |
| E-post- og portaltilstand | forhåndskjente kø- og WebMail-testobjekter etter gjenoppretting | plattformen starter, gammel melding er uleselig |
| Klynge | failover med SMTP-, HTTPS-, nøkkel- og loggtest | VIP bytter, men backendtilstand eller utgående IP gjør det ikke |
| Revisjon og Syslog | ekstern hendelseskopi med tids- og meldingskorrelasjon | lokal gjenoppretting sletter undersøkelseshistorikken |

For eldre Totemomail er sikkerhetskopieringsprosedyren versjons- og installasjonsavhengig. EPG-utsagnet om integrert sikkerhetskopi må ikke overføres tilbakevirkende til en gammel ekstern database eller en separat WebMail-lagring. Produsentens supportvei, installert dokumentasjon og en isolert gjenopprettingstest er primærkildene her.

## Overvåking og sikkerhetsdrift

Kiteworks bekrefter for EPG eksport til ett eller flere Syslog-mål samt Splunk-integrasjon. Plattformdokumentasjonen beskriver normaliserte logger og metadata om e-post og andre kommunikasjonskanaler. For meldingsadministratorer er den faglige korrelasjonen fortsatt avgjørende: Plattformrevisjon, SMTP-hendelse, policyavgjørelse, nøkkeloperasjon, portaltilgang og etterfølgende Message Trace må samles ved hjelp av tid, identitet og meldingsidentifikator ([Kiteworks: Migration FAQ – syslog](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf), [Kiteworks: EPG logging and audit](https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf)).

Et administratordashbord bør minst vise mottak og avvisninger, køalder per neste hop, policyfeil, antall og type kryptooperasjoner, manglende eller ugyldige nøkler, sertifikatutløp, LDAP-latens, mislykkede portalinnlogginger, disk- og databasetilstand, klyngereplikering, alder på sikkerhetskopier og Syslog-tap. En tom kø er bare et positivt funn når meldinger samtidig mottas og overleveres vellykket til neste hop.

Syslog over UDP tilbyr ingen garantert levering. For beskyttelsesverdige revisjonsdata må TLS-transport, buffering, målautentisering, korrekt tid og alarm ved bortfall planlegges ([RFC 5424](https://datatracker.ietf.org/doc/html/rfc5424), [RFC 5425](https://datatracker.ietf.org/doc/html/rfc5425)). Logger kan inneholde avsender, mottaker, emne, sertifikatinformasjon eller portalaktiviteter; dataminimering og tilgangsbeskyttelse hører derfor til loggpolicyen.

Historiske sikkerhetsmeldinger viser at også webgrensesnittet er en del av angrepsflaten. NVD-oppføringer dokumenterer blant annet tidligere CSRF-, objektaksess- og XSS-sårbarheter i Totemomail-webkomponenter. Dette sier ikke noe om en vilkårlig ukontrollert patchstatus, men innebærer en varig plikt til å inkludere produsenthenvisninger, eksponering, supportstatus og webportaltester i patchadministrasjonen ([NVD: CVE-2018-6563](https://nvd.nist.gov/vuln/detail/CVE-2018-6563), [NVD: CVE-2020-7918](https://nvd.nist.gov/vuln/detail/CVE-2020-7918), [NVD: CVE-2024-28063](https://nvd.nist.gov/vuln/detail/CVE-2024-28063)).

Diagnosen følger en testmelding fra mottak via policy- og nøkkelavgjørelse til SMTP-utgang eller portaltilgang.

## Diagnoseverktøy

Diagnosen starter ved den publiserte e-postbanen og følger deretter EPG-hendelsene frem til levering eller WebMail. Plattformmigrering og lisensstatus kontrolleres først når det berørte behandlingstrinnet er fastslått.

### DNS og offentlige e-postbaner

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Totemomail- und EPG-DNS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName -Type MX example.ch
Resolve-DnsName epg.example.ch -Type A,AAAA
Resolve-DnsName _25._tcp.epg.example.ch -Type TLSA
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig +short MX example.ch
dig +short A epg.example.ch
dig +short AAAA epg.example.ch
dig +short TLSA _25._tcp.epg.example.ch
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) og [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) viser offentlig MX, gatewayadresse og eventuelt DANE-TLSA. Internt og eksternt resolverperspektiv må kontrolleres separat. I Microsoft 365 beskriver MX bare det første internetthopet; connectorer og transportregler bestemmer den videre ruten.

### TCP-, SMTP- og TLS-tilgjengelighet

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Totemomail- und EPG-TLS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Test-NetConnection epg.example.ch -Port 25 -InformationLevel Detailed
Test-NetConnection epg.example.ch -Port 443 -InformationLevel Detailed
curl.exe --verbose --ssl-reqd smtp://epg.example.ch:25
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nc -vz epg.example.ch 25
nc -vz epg.example.ch 443
openssl s_client -starttls smtp -connect epg.example.ch:25 \
  -servername epg.example.ch -showcerts
```

  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) og [`nc`](https://man.openbsd.org/nc) dokumenterer bare TCP-banen. Windows-[`curl.exe`](https://curl.se/docs/manpage.html) ber om SMTP-STARTTLS; [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) viser funksjoner, handshake, SNI og sertifikatkjede. Først en fullstendig SMTP-transaksjon med positivt DATA-svar dokumenterer mottak.

### Autorisert SMTP-testmelding

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Totemomail- und EPG-SMTP-Test">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
curl.exe --verbose --ssl-reqd --url smtp://epg.example.ch:25 `
  --mail-from test-sender@example.ch `
  --mail-rcpt test-recipient@example.net `
  --upload-file .\epg-test.eml
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
swaks --server epg.example.ch --port 25 --tls \
  --from test-sender@example.ch --to test-recipient@example.net \
  --data epg-test.eml
```

  </div>
</div>

[`curl`](https://curl.se/docs/manpage.html) og [`swaks`](https://jetmore.org/john/code/swaks/) sender her en kontrollert testmelding. Avsender, mottaker og mål må være uttrykkelig autorisert for dette. Det skal loggføres SMTP-sluttsvar, gateway-ID, utløst policy, benyttet nøkkel eller portalbane, neste hop og faktisk ankomst. En `250` på `RCPT TO` er ennå ikke mottak av meldingsinnholdet.

### WebMail- eller portalbane

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Totemomail- und EPG-Portaldiagnose">
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

[`Invoke-WebRequest`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest), [`curl`](https://curl.se/docs/manpage.html) og [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) kontrollerer HTTP- og TLS-grensesnittet. En vellykket statuskode dokumenterer verken innlogging eller nøkkeltilgang. En syntetisk nettlesertest må dekke innlogging, åpning av en kjent melding, vedlegg og svar.

### Pakkebane ved riktig målepunkt

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Totemomail- und EPG-Paketerfassung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
pktmon filter remove
pktmon filter add EPG-SMTP -p 25
pktmon start --capture --pkt-size 0 --file-name epg.etl
pktmon stop
pktmon pcapng epg.etl -o epg.pcapng
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
tcpdump -ni any -s 0 -w epg.pcap 'tcp port 25 or tcp port 443'
```

  </div>
</div>

[`pktmon`](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon) og [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) ser bare trafikk ved det valgte målepunktet. En administratorklient observerer ikke automatisk banen mellom Exchange Online, load balancer og EPG. En autorisert vert, port mirror eller skybaserte flowlogger egner seg til analysen; SMTP før STARTTLS og webmetadata må lagres med tilsvarende beskyttelse.

## Teknisk historie

Totemo ble grunnlagt i Sveits i 2001 og utviklet løsninger for e-postkryptering, Managed File Transfer og mobil kommunikasjon. NIST-valideringene fra 2012 og 2019 dokumenterer utviklingen av det Java-baserte Totemo Cryptographic Module og Totemo Security Platform på egne appliance-operativsystemer ([Entrust Partner Directory: Totemo](https://www.entrust.com/partners/directory/totemo), [NIST: TCM Security Policy 2012](https://csrc.nist.gov/csrc/media/projects/cryptographic-module-validation-program/documents/security-policies/140sp1884.pdf), [NIST CMVP Certificate 3500](https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/3500)).

Kiteworks overtok totemo i januar 2022. Produsentdokumentasjonen beskriver integreringen av Totemomail-koden i Kiteworks-plattformen og omdøpingen til Email Protection Gateway. EPG kombinerte de eksisterende funksjonene for e-postkryptering med Hardened Virtual Appliance, felles roller, normaliserte logger og andre Kiteworks-kommunikasjonskanaler ([Kiteworks: acquisition and EPG integration](https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf), [Kiteworks: Migration FAQ](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf), [Kiteworks: January 2022 acquisition](https://www.kiteworks.com/sites/default/files/resources/on-the-hunt-kiteworks-secure-content-strategy.pdf)).

Overgangen er teknisk relevant: Det gamle produktet kunne driftes med en ekstern database; EPG overtar dataene til en integrert database og flytter vedlikehold av operativsystem og database inn i appliance-grensen. Driftsprosedyrer, overvåking og gjenoppretting må derfor ikke bare erstatte produktbegreper, men må tilordne kontrollflater og lagrede data og konfigurasjoner på nytt.

## Kilder

- [Kiteworks – FAQ for totemomail Customers Updating to EPG](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf)
- [Kiteworks – Top 6 Reasons to Add Email Protection Gateway](https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf)
- [Kiteworks – Enterprise 8.0 Licensing and Packaging](https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf)
- [Microsoft Marketplace – totemomail Encryption Gateway](https://marketplace.microsoft.com/en/marketplace/apps/totemo.totemo-azr-tm7?tab=overview)
- [IETF RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321)
- [IETF RFC 8551 – S/MIME 4.0](https://datatracker.ietf.org/doc/html/rfc8551)
- [IETF RFC 9580 – OpenPGP](https://datatracker.ietf.org/doc/html/rfc9580)
- [IETF RFC 3207 – SMTP STARTTLS](https://datatracker.ietf.org/doc/html/rfc3207)
- [Microsoft – Configure mail flow using connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow)
- [Microsoft – Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)
- [NIST CMVP – Certificate 3500, Totemo Cryptographic Module](https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/3500)
- [NIST – Totemo Cryptographic Module Security Policy 2019](https://csrc.nist.gov/CSRC/media/projects/cryptographic-module-validation-program/documents/security-policies/140sp3500.pdf)
- [Microsoft – Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud)
- [RFC 5322: Message-ID](https://datatracker.ietf.org/doc/html/rfc5322)
- [Kiteworks – Top 5 Reasons totemo and Kiteworks Are Better Together](https://www.kiteworks.com/sites/default/files/resources/kiteworks-product-brief-top-5-reasons-totemo-kiteworks-better-together.pdf)
- [IETF RFC 5652 – Cryptographic Message Syntax](https://datatracker.ietf.org/doc/html/rfc5652)
- [IETF RFC 5280 – X.509 PKI Certificate and CRL Profile](https://datatracker.ietf.org/doc/html/rfc5280)
- [IETF RFC 6960 – Online Certificate Status Protocol](https://datatracker.ietf.org/doc/html/rfc6960)
- [IETF RFC 7672 – SMTP Security via DANE](https://datatracker.ietf.org/doc/html/rfc7672)
- [IETF RFC 8461 – MTA Strict Transport Security](https://datatracker.ietf.org/doc/html/rfc8461)
- [L-Bank – Sikker e-postkommunikasjon med Totemomail](https://www.l-bank.info/allgemein/rechtliches/e-secure/sichere-e-mail-kommunikation.html)
- [Kiteworks – Security and Integration Overview](https://www.kiteworks.com/sites/default/files/resources/accellion-brochure-external-file-sharing-security-and-governance-platform_redirected_20191210.pdf)
- [IETF RFC 4511 – LDAP Protocol](https://datatracker.ietf.org/doc/html/rfc4511)
- [IETF RFC 4513 – LDAP Authentication Methods and Security Mechanisms](https://datatracker.ietf.org/doc/html/rfc4513)
- [IETF RFC 5424 – Syslog Protocol](https://datatracker.ietf.org/doc/html/rfc5424)
- [IETF RFC 5425 – TLS Transport Mapping for Syslog](https://datatracker.ietf.org/doc/html/rfc5425)
- [NVD – CVE-2018-6563](https://nvd.nist.gov/vuln/detail/CVE-2018-6563)
- [NVD – CVE-2020-7918](https://nvd.nist.gov/vuln/detail/CVE-2020-7918)
- [NVD – CVE-2024-28063](https://nvd.nist.gov/vuln/detail/CVE-2024-28063)
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
- [Entrust Partner Directory – Totemo](https://www.entrust.com/partners/directory/totemo)
- [NIST – Totemo Cryptographic Module Security Policy 2012](https://csrc.nist.gov/csrc/media/projects/cryptographic-module-validation-program/documents/security-policies/140sp1884.pdf)
- [Kiteworks – The Deal: On the Hunt](https://www.kiteworks.com/sites/default/files/resources/on-the-hunt-kiteworks-secure-content-strategy.pdf)
