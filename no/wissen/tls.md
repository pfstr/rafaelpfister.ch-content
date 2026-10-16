---
title: "TLS: protokoll, sertifikatvalidering og drift"
blatt: "tls"
description: "Transport Layer Security fra et administratorperspektiv: Handshake og Record Layer, nøkkelplanen i TLS 1.3, validering av X.509- og PKIX-stier, SAN og SNI, truststores, SMTP- og LDAP-integrasjon, nøkkeldrift samt lagvis diagnose."
fakten:
  - label: Fullt navn
    wert: Transport Layer Security
    href: https://datatracker.ietf.org/doc/html/rfc9846
  - label: Beskyttelsesmål
    wert: Autentisering · konfidensialitet · integritet
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-1
  - label: Protokolldeler
    wert: Handshake Protocol · Record Protocol · Alerts
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-1
  - label: Underlag
    wert: pålitelig, ordnet bytestrøm
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-1
  - label: Autentisering
    wert: Server alltid · klient valgfritt
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-1
  - label: Nøkkelutveksling
    wert: (EC)DHE · PSK · kombinert
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-2
  - label: Record-beskyttelse
    wert: AEAD med separate Traffic Secrets
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-5
  - label: Sertifikatprofil
    wert: X.509 v3 og PKIX-stivalidering
    href: https://datatracker.ietf.org/doc/html/rfc5280
  - label: Tjenesteidentitet
    wert: Referanse-ID mot subjectAltName
    href: https://datatracker.ietf.org/doc/html/rfc9525
  - label: Startmodeller
    wert: implisitt TLS eller protokollspesifikk STARTTLS
    href: https://datatracker.ietf.org/doc/html/rfc8314
  - label: Gjenopptakelse
    wert: PSK-basert Session Resumption
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-2.2
  - label: Driftsanbefaling
    wert: BCP 195
    href: https://www.rfc-editor.org/info/bcp195
werbung:
  - tools
  - newsletter
ctaThemen:
  - e-mail-verschluesselung
  - smtp-mailflow
translationSourceHash: 50ea371b919edcc4811a303d26ec6c575ef8dc334ce6e681fdc3f74de9ca2058
translationModel: gpt-5.6-terra
translatedAt: 2026-10-15T08:51:56.656Z
translationReview: required
---

# TLS: protokoll, sertifikatvalidering og drift

**Transport Layer Security** (TLS) beskytter én enkelt forbindelse mellom to endepunkter. Først blir klient og server enige om kryptografiske metoder og nøkler, deretter går SMTP, LDAP, HTTP eller en annen applikasjonsprotokoll gjennom denne beskyttede kanalen. TLS hindrer avlytting og uoppdagede endringer på denne forbindelsen og autentiserer minst serveren. Det bekrefter imidlertid verken avsenderen av en e-post eller garanterer at neste SMTP-hopp også bruker TLS ([RFC 9846, avsnitt 1](https://datatracker.ietf.org/doc/html/rfc9846#section-1), [RFC 3207, avsnitt 6](https://datatracker.ietf.org/doc/html/rfc3207#section-6)).

I drift må derfor flere kontroller lykkes etter hverandre: Riktig vert og port må være tilgjengelige, klient og server må ha felles TLS-parametere, sertifikatkjeden må føre til et klarert anker, og det forespurte navnet må stå i sertifikatet. Først deretter begynner selve applikasjonsprotokollen. Artikkelen følger nettopp denne rekkefølgen og viser på hvert punkt hva en administrator må observere, konfigurere og sikre.

## Plassering i protokollstakken

I vanlig serverdrift ligger TLS mellom [TCP](/kb/tcp) og applikasjonsprotokollen. TCP leverer den ordnede datastrømmen; TLS beskytter den; [SMTP](/kb/smtp), [LDAP](/kb/ldap), IMAP eller HTTP fastsetter når TLS starter og hvilket servernavn som forventes. Denne innbyggingen forklarer hvorfor en åpen port ennå ikke utgjør en vellykket TLS-test, og hvorfor en vellykket handshake ennå ikke beviser en fungerende e-post- eller katalogtjeneste ([RFC 9846, avsnitt 1](https://datatracker.ietf.org/doc/html/rfc9846#section-1)).

| Lag | Synlig objekt | Typisk administrasjonsspørsmål |
|---|---|---|
| Navne- og tjenestevalg | DNS-navn, MX- eller SRV-mål, port | Hvilket endepunkt ble faktisk valgt? |
| Transport | TCP-forbindelse og tilstandsautomat | Kommer SYN, SYN-ACK og ACK frem i begge retninger? |
| TLS-handshake | Versjon, Key Share, signatur, sertifikater, Finished | Finnes det felles parametere, og godtas peeren? |
| TLS Record Layer | krypterte records med AEAD-beskyttelse | Overføres eller forkastes records, eller avsluttes de med Alert? |
| Applikasjonsprotokoll | SMTP-, LDAP-, IMAP- eller HTTP-dialog | Hvilke kommandoer og identiteter gjelder etter handshaken? |

QUIC er et viktig unntak fra det klassiske bildet «TLS over TCP»: Det bruker TLS-1.3-handshaken, men ikke TLS Record Layer; QUIC beskytter sine egne pakker med nøkler avledet fra TLS ([RFC 9001](https://datatracker.ietf.org/doc/html/rfc9001)). For e-post- og katalogprotokollene som behandles her, forblir den TCP-baserte TLS-kanalen den relevante modellen.

## Arkitektur: Handshake, Record Layer og Alerts

TLS utfører to oppgaver etter hverandre. I **Handshake Protocol** forhandler motpartene parametere, autentiserer hverandre og genererer felles nøkkelmateriale. Deretter deler **Record Protocol** nyttedataene opp i beskyttede records. **Alerts** rapporterer protokollfeil eller en ordnet avslutning; ved en brå nettverks- eller prosessfeil kan TCP-forbindelsen imidlertid også avsluttes uten en brukbar Alert ([RFC 9846, avsnitt 4 til 6](https://datatracker.ietf.org/doc/html/rfc9846#section-4)).

Dette skillet er avgjørende i drift. En sertifikatfeil hører til autentiseringsfasen av handshaken. En `bad_record_mac` eller en `decrypt_error` gjelder kryptografisk behandling. En timeout uten ServerHello tyder derimot ofte på protokollforveksling, filtrering, feil port eller en server som først forventer en klartekstdialog og en STARTTLS-kommando.

## TLS-1.3-handshake trinn for trinn

Den fullstendige TLS-1.3-handshaken kan leses som en samtale: Klienten kommer med et tilbud, serveren velger fra det, dokumenterer identiteten sin og begge sider bekrefter protokollen så langt. Allerede etter **ServerHello** er de videre handshake-meldingene beskyttet. Med de gjensidige **Finished**-meldingene er det endelig fastslått at begge parter har det samme transkriptet og de samme hemmelighetene ([RFC 9846, avsnitt 2](https://datatracker.ietf.org/doc/html/rfc9846#section-2)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1056" src="/images/kb-interaktiv-tls-handshake.svg?v=20260813" title="Interaktive Infografik: TLS-1.3-Handshake, Zertifikatskette, Identitätsprüfung und Admin-Diagnosepunkte" loading="lazy">
  <a href="/images/kb-interaktiv-tls-handshake.svg?v=20260813">Åpne infografikk om TLS-handshake og sertifikatvalidering</a>
</iframe>

Forløpet i detalj:

1. **ClientHello** tilbyr blant annet støttede TLS-versjoner, Cipher Suites, signaturalgoritmer og kryptografiske grupper. En `key_share` inneholder vanligvis allerede klientens efemere offentlige andel. SNI kan overføre ønsket servernavn, ALPN mulige applikasjonsprotokoller.
2. **ServerHello** velger versjon, Cipher Suite og Key Share. Klientens og serverens andeler danner den felles (EC)DHE-hemmeligheten. Dersom ingen egnet forhåndssendt gruppe finnes, kan serveren kreve en ny ClientHello med **HelloRetryRequest**.
3. **EncryptedExtensions** bekrefter ytterligere forhandlede utvidelser. Dersom serveren ber om et klientsertifikat, følger også **CertificateRequest**.
4. **Certificate** overfører Leaf-sertifikatet og normalt Intermediate-sertifikatene klienten trenger. **CertificateVerify** signerer handshake-transkriptet med den private nøkkelen som hører til Leaf-sertifikatet. Slik beviser serveren nøkkelbesittelse; sertifikatet alene beviser den ikke.
5. **Finished** er en MAC over transkriptet så langt og bekrefter de avledede handshake-hemmelighetene. Klienten validerer parametere, sertifikatsti og tjenesteidentitet og sender deretter sitt Finished, ved forespurt gjensidig autentisering i tillegg Certificate og CertificateVerify.
6. For applikasjonsdata avledes separate Traffic Secrets. Senere **NewSessionTicket**-meldinger tillater PSK-basert gjenopptakelse uten at serveren må sende det opprinnelige sertifikatet på nytt i hver påfølgende handshake.

Det observerte antallet Round Trips avhenger derfor av DNS, TCP, fullstendig eller gjenopptatt TLS-handshake, HelloRetryRequest og applikasjonen. En pakkefangst må måle disse fasene separat; «TLS var tregt» er ingen diagnose uten denne oppdelingen.

## Nøkkelplan og Record Layer

Fra det avtalte (EC)DHE- eller PSK-materialet genererer TLS 1.3 ikke én enkelt sesjonsnøkkel, men flere separate hemmeligheter med **HKDF**. Handshake, applikasjonsdata, eksporter og senere gjenopptakelse bruker eget materiale; i tillegg har klient og server separate nøkler og initialiseringsvektorer for hver retning. Denne oppdelingen er viktig for analyse og Incident Response, fordi en avslørt Record-nøkkel ikke automatisk dekrypterer alle faser og retninger ([RFC 9846, avsnitt 7.1](https://datatracker.ietf.org/doc/html/rfc9846#section-7.1)).

Record Layer bruker utelukkende **Authenticated Encryption with Associated Data** (AEAD). Cipher Suite angir i TLS 1.3 AEAD-algoritmen og hashen for HKDF; sertifikattype, signaturalgoritme og Key-Exchange-gruppe forhandles separat. Dette er annerledes enn i TLS 1.2, der navn som `TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256` samler flere beslutninger i én suite.

| Egenskap | TLS 1.2 | TLS 1.3 |
|---|---|---|
| Cipher-Suite-semantikk | Nøkkelutveksling, autentisering, kryptering og hash samlet | AEAD-metode og HKDF-hash; Key Exchange og signatur separat |
| Forward Secrecy | avhenger av valgt suite | offentlig nøkkelutveksling kun med efemer (EC)DHE |
| Handshake-synlighet | store deler i klartekst | kryptert etter ServerHello |
| Gjenopptakelse | Session ID eller Ticket etter eldre modell | ensartet PSK-modell |
| 0-RTT | ikke del av protokollen | valgfritt, men kan gjentas og er ikke egnet for ikke-idempotente operasjoner |

0-RTT sparer én Round Trip ved gjenopptakelse, men har svakere Replay-egenskaper. En server må derfor avgjøre applikasjonsavhengig hvilke tidlige data den aksepterer. For tilstandsendrede administrasjons-, e-post- eller katalogoperasjoner må lav forsinkelse ikke forveksles med Replay-sikkerhet ([RFC 9846, avsnitt 2.3 og 8](https://datatracker.ietf.org/doc/html/rfc9846#section-2.3)).

## X.509-sertifikat og privat nøkkel

Et X.509-sertifikat er den signerte beskrivelsen av en offentlig nøkkel. Det angir blant annet utsteder, gyldighetsperiode, tillatte bruksområder og navnene nøkkelen kan brukes for. Den tilhørende private nøkkelen er ikke en del av sertifikatet og må holdes beskyttet hos operatøren. RFC 5280 definerer PKIX-profilen som brukes til dette, med felt som Key Usage, Extended Key Usage, Basic Constraints og Subject Alternative Name ([RFC 5280, avsnitt 4](https://datatracker.ietf.org/doc/html/rfc5280#section-4)).

X.509 kodes binært med ASN.1 DER. De ofte synlige linjene `BEGIN CERTIFICATE` og `END CERTIFICATE` er en Base64-basert tekstinnpakning som RFC 7468 beskriver. En **PKCS #12**-container kan derimot overføre privat nøkkel, sertifikatkjede og ytterligere attributter samlet ([RFC 7468](https://datatracker.ietf.org/doc/html/rfc7468), [RFC 7292](https://datatracker.ietf.org/doc/html/rfc7292)). Filendelser som `.cer`, `.crt`, `.pem`, `.p12` eller `.pfx` er konvensjoner, men ikke et pålitelig bevis på innhold eller koding.

Viktige utvidelser har ulike oppgaver:

| Utvidelse | Kontrollspørsmål | Vanlig driftsfeil |
|---|---|---|
| Subject Alternative Name | inneholder den DNS-, IP-, URI- eller SRV-identiteten klienten forventer? | Administrasjonsnavn eller Load-Balancer-alias mangler |
| Basic Constraints | kan sertifikatet opptre som CA, og hvor dyp kan stien være? | Intermediate uten `CA=TRUE` eller feil Path Length |
| Key Usage | hvilke grunnleggende kryptografiske operasjoner kan nøkkelen brukes til? | CA kan ikke signere eller Leaf passer ikke til signaturbruken |
| Extended Key Usage | er nøkkelen tillatt for server- eller klientautentisering? | S/MIME-sertifikat brukes feilaktig som TLS-serversertifikat |
| Authority / Subject Key Identifier | hvilke nøkkelvarianter hjelper ved stibygging? | feil Intermediate med samme Subject valgt |
| CRL Distribution Points / AIA | hvor finner klienten tilbakekallings- eller utstederinformasjon? | målet er ikke tilgjengelig fra servernettverket eller proxy mangler |

## Sertifikatsti og tillitsgrense

Sammen med sitt **Leaf-sertifikat** sender en server normalt de nødvendige **Intermediate-sertifikatene**. Klienten forsøker å bygge en sti derfra til en Root-CA den allerede stoler på lokalt. Nettopp dette siste punktet er avgjørende: En Root-CA sendt av serveren blir ikke klarert bare av den grunn. Truststore i prosessen som faktisk kobler til, fastsetter hvilke tillitsankre som gjelder ([RFC 5280, avsnitt 6](https://datatracker.ietf.org/doc/html/rfc5280#section-6)).

Stibygging og stivalidering er mer enn tre enkle avkryssinger. Klienten må blant annet kontrollere signaturer, gyldighetsintervaller, Basic Constraints, Path Length, Name Constraints, Key Usage, Extended Key Usage, kritiske utvidelser, retningslinjer og eventuelt tilbakekalling. Deretter følger den protokollspesifikke tjenesteidentiteten. Ulike klienter kan konstruere ulike gyldige stier fra samme Leaf-sertifikat, fordi truststores, AIA-nedlasting, Cross-Signatures og algoritmeregler varierer.

Begrepet **Truststore** betegner derfor ikke ett universelt sted. En Windows-applikasjon kan bruke Schannel og Windows-sertifikatlageret, en Java-applikasjon JSSE med egen `cacerts`- eller applikasjonsfil, en container et innebygd CA-bundle og en appliance et proprietært lager. Selv på samme vert kan to prosesser stole ulikt på samme server. Ved hver feil må den faktisk validerende prosessen og dens Truststore identifiseres.

Tilbakekalling er en ekstra policybeslutning. CRL-er distribuerer signerte lister over tilbakekalte sertifikater; OCSP besvarer statusforespørsler for enkeltsertifikater. Om en klient kontrollerer, hvordan den reagerer ved timeout, og om den bruker OCSP Stapling, avhenger av bibliotek og applikasjon ([RFC 5280, avsnitt 5](https://datatracker.ietf.org/doc/html/rfc5280#section-5), [RFC 6960](https://datatracker.ietf.org/doc/html/rfc6960)). «Sertifikatet er tidsmessig gyldig» betyr derfor ikke automatisk «sertifikatet godtas av denne klienten».

## Tjenesteidentitet: SAN, referanse-ID og SNI

Etter at stien er godkjent, kontrollerer klienten navnet på tjenesten. Utgangspunktet er **referanseidentiteten**, som vanligvis er det konfigurerte vertsnavnet eller vertsdelen av en URL. Den må samsvare med en passende typet oppføring i **Subject Alternative Name** i Leaf-sertifikatet. For DNS-navn gjelder `dNSName`, for direkte konfigurerte IP-adresser `iPAddress`; det fritt formulerte Common Name er ingen moderne reserveutvei. Wildcards kan bare erstatte den komplette venstre etiketten ([RFC 9525, avsnitt 4 og 6](https://datatracker.ietf.org/doc/html/rfc9525#section-4)).

**Server Name Indication** (SNI) må skilles fra dette. Klienten sender ønsket servernavn i ClientHello, slik at en delt listener kan velge riktig sertifikat og riktig virtuell konfigurasjon. SNI forteller serveren hva klienten vil nå; det erstatter verken SAN-sammenligningen eller stivalideringen ([RFC 6066, avsnitt 3](https://datatracker.ietf.org/doc/html/rfc6066#section-3)). En test uten SNI kan derfor vise standardsertifikatet, selv om produksjonsklienten mottar et annet.

**Application-Layer Protocol Negotiation** (ALPN) lar klient og server velge en høyere protokoll innenfor TLS-handshaken. Den er særlig synlig for HTTP og QUIC, men er konseptuelt heller ingen sertifikatkontroll ([RFC 7301](https://datatracker.ietf.org/doc/html/rfc7301)).

## Implisitt TLS og STARTTLS

Applikasjonsprotokollen bestemmer når handshaken starter. Ved **implisitt TLS** følger ClientHello umiddelbart etter TCP-opprettelsen. Ved **STARTTLS** snakker begge sider først klartekst og bytter først til TLS etter en protokollspesifikk kommando. Den som sender en ClientHello direkte til en STARTTLS-port, bruker derfor TLS, men på feil tidspunkt i dialogen.

| Tjenestevei | Port | Startmodell | Protokollregel |
|---|---:|---|---|
| SMTP-relé mellom MTA-er | 25 | ESMTP `STARTTLS` | etter vellykket handshake tilbakestilles SMTP; klienten sender EHLO på nytt |
| Message Submission | 587 | STARTTLS | Submission kan kreve TLS før autentisering |
| Implicit TLS Submission | 465 | TLS umiddelbart | beskrevet for Mail Submission i RFC 8314 |
| IMAP / IMAPS | 143 / 993 | STARTTLS / TLS umiddelbart | klarteksttilgang anses som foreldet for moderne bruk |
| POP3 / POP3S | 110 / 995 | STLS / TLS umiddelbart | også separate klartekst- og Implicit-TLS-veier |
| LDAP / LDAPS | 389 / 636 | LDAP StartTLS Extended Operation / TLS umiddelbart | Bind og identitetskontroll følger LDAP-sikkerhetspolicyen |
| HTTP / HTTPS | 80 / 443 | separate URI-skjemaer; på 443 TLS umiddelbart | Host/SNI, sertifikatidentitet og HTTP-versjon er separate beslutninger |

Porttilordningene stammer fra [IANA Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml); semantikken følger den aktuelle applikasjonsstandarden. Port 465 er derfor ikke generelt «SMTPS for server-til-server-levering», men den registrerte Implicit-TLS-veien for Submission.

SMTP STARTTLS definerer en særlig viktig tilstandsovergang. Før oppgraderingen kan EHLO-svaret og `STARTTLS`-tilbudet manipuleres. Etter handshaken må begge sider forkaste tidligere lært SMTP-kunnskap, og klienten sender EHLO på nytt ([RFC 3207, avsnitt 4.2](https://datatracker.ietf.org/doc/html/rfc3207#section-4.2)). For LDAP er StartTLS en Extended Operation innenfor LDAP-sesjonen; det er ikke det samme som en ny TCP-port ([RFC 4511, avsnitt 4.14](https://datatracker.ietf.org/doc/html/rfc4511#section-4.14)).

## TLS-policy for SMTP

For SMTP på port 25 er TLS ofte opportunistisk: Den sendende MTA-en krypterer når motparten tilbyr STARTTLS og lokal policy godtar handshaken. Uten ekstra policy kan en angriper fjerne tilbudet eller forstyrre handshaken. Avsender-MTA-en må da avgjøre om den skal levere ukryptert, prøve på nytt senere eller avbryte endelig. RFC 3207 tydeliggjør dermed at et beskyttet SMTP-hopp ikke er ende-til-ende-beskyttelse av meldingen.

To mekanismer kan binde mottakerdomenet og TLS-kravet sterkere:

- **DANE for SMTP** publiserer TLSA-data under DNSSEC og avleder identiteten som skal kontrolleres fra MX-veien. En validerende resolver og en intakt DNSSEC-kjede blir dermed en del av tillitsgrunnlaget ([RFC 7672](https://datatracker.ietf.org/doc/html/rfc7672)).
- **MTA-STS** publiserer en policy via DNS og HTTPS. Den angir tillatte MX-mønstre og krever PKIX-validert TLS i policyens levetid ([RFC 8461](https://datatracker.ietf.org/doc/html/rfc8461)). **SMTP TLS Reporting** rapporterer aggregerte suksess- og feilmønstre for disse policyene ([RFC 8460](https://datatracker.ietf.org/doc/html/rfc8460)).

Disse modellene kan ikke byttes ut med hverandre. DANE forankrer utsagnet i DNSSEC og TLSA; MTA-STS bruker Web-PKIX og en policy hentet via HTTPS. En administrator må derfor, i tillegg til sertifikat og Cipher, overvåke [DNS](/kb/dns), MX-valg, policycache og rapporteringskanal.

## Gjensidig TLS-autentisering

**Mutual TLS** (mTLS) utvider vanlig serverbevis med et klientsertifikat. Serveren ber om det med CertificateRequest; klienten sender sin kjede og beviser med CertificateVerify besittelse av den private nøkkelen. Deretter kontrollerer serveren sti, gyldighet, Extended Key Usage og sin lokale tilordning. Sertifikatet identifiserer først en nøkkelinnehaver. Hvilke applikasjonskonto-, LDAP- eller SMTP-relérettigheter som følger av det, bestemmer fortsatt applikasjonen.

I heterogene miljøer oppstår ofte asymmetriske tillitsproblemer: Klienten stoler på server-CA-en, men serveren stoler ikke på klient-CA-en; serveren ber bare om bestemte Issuers; klientsertifikatet har bare `serverAuth`; eller en Load Balancer terminerer TLS og videresender ingen verifisert klientidentitet til backend. En vellykket serverautentisert handshake er derfor ikke bevis på fungerende mTLS.

## Drift av nøkler, sertifikater og fornyelse

Et sertifikatbytte er mer enn å erstatte en fil. Nøkkel, sertifikat, Intermediate-kjede, listenerbinding og utrullingsstatus må passe sammen. En robust livssyklus omfatter minst:

1. **Inventar:** Registrer tjenestenavn, port, startmodell, terminerende prosess, forventede SAN-er, Issuer, serienummer, Fingerprint, `notBefore`/`notAfter`, Key Store, Truststore og ansvarlig automatisering.
2. **Nøkkelgenerering:** Generer et nytt nøkkelpar i den planlagte programvare-, TPM- eller HSM-konteksten. Private nøkler forblir konfidensielle og krever beskyttelse, tilgangskontroll og, avhengig av Recovery-modell, en dokumentert strategi for sikkerhetskopiering eller ny utstedelse ([NIST SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)).
3. **Søknad og utstedelse:** Generer CSR med riktige identiteter og parametere; oppfyll CA-policy og nødvendig dokumentasjon. ACME standardiserer automatisk bestilling og fornyelse, men garanterer ennå ingen riktig tjenestebinding ([RFC 8555](https://datatracker.ietf.org/doc/html/rfc8555)).
4. **Installasjon:** Plasser Leaf, passende Intermediate-bundle og privat nøkkel i formatet prosessen faktisk leser. Kontroller filrettigheter, passordtilgang og Key-Pair-match.
5. **Aktivering:** Last inn tjenesten på nytt eller bytt binding atomisk. I klynger, Load Balancere og med flere protokollroller må hver TLS-terminator behandles separat.
6. **Validering:** Kontroller eksternt og internt med produktivt SNI, port og STARTTLS-modus; registrer levert kjede, SAN, gyldighet, algoritme, policy og applikasjonstest.
7. **Utfasing:** Fjern gammelt sertifikat først når alle noder leverer det nye og ingen avhengighet lenger viser til Fingerprint eller gammel Issuer. Ved kompromittering må det i tillegg tilbakekalles, og den private nøkkelen behandles som kompromittert.

En sikkerhetskopi må passe til denne modellen. En eksportert offentlig sertifikatdel gjenoppretter ingen tjeneste dersom privat nøkkel, passord, providerbinding, ACL eller HSM-objekt mangler. Omvendt øker hver ubeskyttet kopi av den private nøkkelen angrepsflaten. Root- og Issuing-CA-Recovery, Server-Key-Recovery og rask ny utstedelse er ulike prosedyrer og må ikke samles under «sikkerhetskopiere sertifikater».

Offentlig Web-PKI, intern Enterprise-PKI og selvforvaltede tillitsankre har ulike regler for utstedelse og levetid. I stedet for å bruke en statisk maksimalverdi som driftsmål bør overvåkingen måle den konkrete `notAfter` for hvert levert sertifikat og beregne tilstrekkelig tid til utstedelse, utrulling, validering og rollback.

## Implementerings- og teknologistack

Hvilken TLS-stack som faktisk arbeider, avhenger av prosessen. Windows-tjenester kan bruke Schannel, Java-applikasjoner JSSE, Unix-tjenester OpenSSL eller GnuTLS og appliances et eget bibliotek. Dermed endres Truststore, støttede metoder, diagnoseverktøy og konfigurasjonssted, selv om alle motparter snakker «TLS».

| Stack | Kjøretid og API | Typisk Trust- og Key-kilde | Administrativ relevans |
|---|---|---|---|
| [OpenSSL](https://docs.openssl.org/master/) ([Kildekode](https://github.com/openssl/openssl)) | native-bibliotek og CLI, hovedsakelig C | CA-fil/-katalog, PEM, DER, PKCS #12, Provider | mange Unix-tjenester og appliances; `s_client`, `x509`, `verify` |
| [Schannel](https://learn.microsoft.com/windows-server/security/tls/tls-ssl-schannel-ssp-overview) | Windows Security Support Provider via SSPI | Windows-sertifikatlager, CNG/CSP-nøkler | System- og .NET-applikasjoner kan bruke OS-policy og lagre |
| [JSSE](https://docs.oracle.com/en/java/javase/24/security/java-secure-socket-extension-jsse-reference-guide.html) | Java-API-er `SSLSocket`, `SSLEngine`, `SSLContext` og Provider | KeyStore, TrustStore, `cacerts`, PKCS #12 | JVM-applikasjon kan avvike fra operativsystemets tillit |
| [GnuTLS](https://www.gnutls.org/documentation.html) | native TLS-bibliotek, C-API og CLI | filer, PKCS #11, systemtillit etter integrasjon | alternativt bibliotek i Unix- og GNU-programvare |

Over dette ligger applikasjonskonfigurasjonen. En e-postserver kan velge sertifikat og policy per listener eller SNI-navn; en Reverse Proxy terminerer TLS foran backend; et Java-produkt leverer med en egen JRE med Truststore; et container-image fryser et CA-bundle på byggetidspunktet. «Sertifikatet er installert i hele operativsystemet» er derfor ingen pålitelig påstand uten bevis på prosess og binding.

Algoritmestøtte er også et snitt av protokollversjon, bibliotek, kryptoleverandør, sertifikat, klienttilbud og lokal policy. Et ECDSA-sertifikat hjelper ikke hvis klienten ikke støtter signaturen; en TLS-1.3-Cipher-streng endrer ingen TLS-1.2-suiter; en systemomfattende Schannel-innstilling styrer ingen separat levert OpenSSL- eller Java-kjøretid. [BCP 195](https://www.rfc-editor.org/info/bcp195) tilbyr IETF-sikkerhetsveiledningen, men den konkrete aktiveringen er fortsatt implementerings- og livssyklusavhengig.

## Diagnose fra port til applikasjon

En TLS-diagnose begynner derfor ikke med sertifikatet, men med den nøyaktige forbindelsesveien. Hver test må omfatte målnavn, oppløst adresse, port, SNI, STARTTLS-modus, klientbibliotek, Truststore, tidspunkt og gjeldende policy. Bare tester med de samme parameterne kan sammenlignes.

### Kontroller TCP-tilgjengelighet separat

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für TCP-Erreichbarkeit vor TLS">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection mail.example.ch -Port 25 -InformationLevel Detailed
Test-NetConnection directory.example.ch -Port 636 -InformationLevel Detailed</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">nc -vz mail.example.ch 25
nc -vz directory.example.ch 636</code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) og [`nc`](https://man.openbsd.org/nc) kontrollerer her bare navneoppløsning og TCP-opprettelse. En vellykket test sier ennå ingenting om TLS-versjon, sertifikat, SNI eller applikasjon.

### Test SMTP STARTTLS med produktivt SNI

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-STARTTLS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">openssl.exe s_client -connect mail.example.ch:25 -starttls smtp `
  -servername mail.example.ch -verify_hostname mail.example.ch `
  -showcerts -verify_return_error</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">openssl s_client -connect mail.example.ch:25 -starttls smtp \
  -servername mail.example.ch -verify_hostname mail.example.ch \
  -showcerts -verify_return_error</code></pre>
  </div>
</div>

[`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) er en diagnoseklient, ikke bevis på policyen til produksjons-MTA-en. `-starttls smtp` leder klartekstdialogen frem til oppgraderingen, `-servername` setter SNI, `-showcerts` viser sertifikatlisten serveren sender, og `-verify_return_error` avbryter ved en valideringsfeil. For en fullstendig identitetstest bør dessuten `-verify_hostname mail.example.ch` og den faktisk brukte CA-stien angis. OpenSSL er ikke en innebygd systemkomponent i Windows; `openssl.exe` må komme fra en kontrollert installasjon.

### Skill mellom LDAP StartTLS og implisitt TLS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für LDAP-TLS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">openssl.exe s_client -connect directory.example.ch:636 `
  -servername directory.example.ch -verify_hostname directory.example.ch `
  -verify_return_error
openssl.exe s_client -connect directory.example.ch:389 -starttls ldap `
  -servername directory.example.ch -verify_hostname directory.example.ch `
  -verify_return_error</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">openssl s_client -connect directory.example.ch:636 \
  -servername directory.example.ch -verify_hostname directory.example.ch \
  -verify_return_error
openssl s_client -connect directory.example.ch:389 -starttls ldap \
  -servername directory.example.ch -verify_hostname directory.example.ch \
  -verify_return_error</code></pre>
  </div>
</div>

Det første kallet forventer TLS umiddelbart på 636, det andre utfører LDAP StartTLS Extended Operation på 389. En vellykket OpenSSL-test beviser ennå ikke at det produktive LDAP-biblioteket bruker samme Truststore, samme tilbakekallingspolicy eller samme referanseidentitet.

### Undersøk sertifikatfil og sti lokalt

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für lokale Zertifikatsprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">certutil.exe -dump server.cer
certutil.exe -verify -urlfetch -sslpolicy mail.example.ch server.cer
certutil.exe -store My</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">openssl x509 -in server.pem -noout -subject -issuer -serial -dates \
  -fingerprint -sha256 -ext subjectAltName,extendedKeyUsage
openssl verify -show_chain -CAfile roots.pem \
  -untrusted intermediates.pem server.pem</code></pre>
  </div>
</div>

[`certutil`](https://learn.microsoft.com/windows-server/administration/windows-commands/certutil) bruker Windows-sertifikatlogikken for `-verify` og kan inkludere et servernavn med `-sslpolicy`. [`openssl x509`](https://docs.openssl.org/master/man1/openssl-x509/) viser felter i et sertifikat; [`openssl verify`](https://docs.openssl.org/master/man1/openssl-verify/) bygger og kontrollerer en sti mot eksplisitt angitte Trust Anchors og Intermediates. [`keytool`](https://docs.oracle.com/en/java/javase/25/docs/specs/man/keytool.html) er det riktige offisielle administrasjons- og inspeksjonsgrensesnittet for JVM-Key- og Truststores; en test med OpenSSL erstatter ikke dette perspektivet.

### Tilordne feilmønsteret til et kontrolltrinn

| Observasjon | Sannsynlig lag | Neste kontroll |
|---|---|---|
| TCP-timeout | DNS, rute, brannmur eller listener | oppløst adresse, SYN/SYN-ACK, returvei, listenerbinding |
| Klartekstbanner i stedet for TLS-svar | feil startmodell | port og `-starttls smtp` eller `-starttls ldap` |
| umiddelbar TLS-Alert etter ClientHello | ingen felles parametere eller policy | versjoner, grupper, signaturalgoritmer, Cipher, SNI |
| feil sertifikat | SNI eller listenerbinding | sendt SNI, Default-VHost, Load-Balancer-konfigurasjon |
| `unable to get local issuer certificate` | manglende Intermediate eller Trust Anchor | serverliste, AIA, klient-Truststore, stibygging |
| `hostname mismatch` | referanse-ID passer ikke til SAN | faktisk brukt navn, SAN-type, alias og wildcard-grense |
| bare Java-applikasjon mislykkes | JSSE-policy eller egen Truststore | JVM, `cacerts`, App-Truststore, disabledAlgorithms |
| bare enkelte klyngenoder leverer gammelt sertifikat | utrulling eller binding | test hvert endepunkt og hver IP med samme SNI |
| SMTP faller tidvis tilbake til klartekst | opportunistisk policy eller nedgradering | MTA-logger, STARTTLS-tilbud, MTA-STS/DANE-status, TLS-RPT |
| mTLS-handshake lykkes, tilgang forblir forbudt | autorisasjon etter TLS | Client-EKU, sertifikatmapping, proxy-videresending, applikasjonsrolle |

[`Wireshark`](https://www.wireshark.org/docs/wsug_html/index.html) kan gjøre ClientHello, ServerHello, utvidelser og Alerts synlige; uten sesjonsnøkler forblir applikasjonsdata og mange TLS-1.3-handshake-meldinger kryptert. En opptak viser heller ikke automatisk hvilken Truststore applikasjonen bruker eller hvorfor policyen avviser en sti.

## Overvåking og driftskriterier

En portsjekk og en advarsel før utløpsdato er ikke tilstrekkelig. Overvåkingen bør kontrollere samme vei som en ekte klient: velge mål, etablere handshake, validere sertifikat og navn og deretter nå applikasjonsprotokollen. Følgende signaler er særlig nyttige:

- målvalg og SNI per tjeneste, listener og klyngenode;
- forhandlet protokoll, Cipher Suite, gruppe og signaturalgoritme;
- Leaf-Fingerprint, serienummer, SAN-er, Issuer samt `notBefore` og `notAfter`;
- faktisk sendt Intermediate-kjede og stivalidering fra relevante Truststores perspektiv;
- OCSP-/CRL-tilgjengelighet og faktisk Fail-open-/Fail-closed-atferd;
- handshake-latens, Resumption-andel, Alert-klasser og feilrate;
- STARTTLS-tilbud, vellykkede oppgraderinger og klartekst-fallbacker per SMTP-mål;
- MTA-STS-, DANE- og TLS-RPT-tilstand der disse policy-mekanismene brukes;
- samsvar mellom alle TLS-terminatorer etter fornyelse og vellykket applikasjonstest bak TLS-laget.

Overvåking bare innenfra overser offentlig kjede, eksternt DNS eller Edge-terminatorer; en ekstern sjekk alene ser ikke interne Truststores og mTLS. For meldings- og katalogplattformer er derfor minst én ekstern klientvei, én intern applikasjonsvei og en direkte kontroll av hver terminator nødvendig.

## Teknisk historie

Historien begynner med Netscapes SSL på 1990-tallet. SSL 2.0 ble senere uttrykkelig forbudt ([RFC 6176](https://datatracker.ietf.org/doc/html/rfc6176)); SSL 3.0 er bare dokumentert som en historisk protokoll og har vært ulovlig siden RFC 7568. IETF utviklet derfra **TLS 1.0** i 1999, **TLS 1.1** i 2006 og **TLS 1.2** i 2008. RFC 8996 erklærte TLS 1.0 og 1.1 som foreldet i 2021. For administratorer forklarer denne avstamningen hvorfor gamle produktnavn, Cipher-syntaks og kompatibilitetsbrytere fortsatt forekommer i konfigurasjoner.

**TLS 1.3** kom i 2018 med RFC 8446. Det fjernet statisk RSA-nøkkelutveksling, gamle ikke-AEAD-metoder og flere Legacy-funksjoner, krypterte handshaken tidligere og innførte den HKDF-baserte nøkkelplanen samt PSK-gjenopptakelsesmodellen. RFC 9846 erstattet RFC 8446 i juli 2026 som konsolidert TLS-1.3-spesifikasjon og integrerte rettelser og krav som frem til da var dokumentert separat. Protokollverdien `1.3` forble den samme; det normative standarddokumentet ble endret ([RFC 9846, avsnitt 1.2](https://datatracker.ietf.org/doc/html/rfc9846#section-1.2)).

Parallelt utviklet identitets- og driftssiden seg: X.509-PKIX ble konsolidert i RFC 5280, SNI og andre utvidelser i RFC 6066, ALPN i RFC 7301, automatisert utstedelse i ACME/RFC 8555 og moderne kontroll av tjenesteidentitet i RFC 9525. TLS-historien er derfor ikke bare en versjonsrekkefølge, men den felles utviklingen av kryptografi, sertifikatstier, applikasjonsintegrasjon og operativ automatisering.

## Kilder

- [IETF RFC 8314 – Cleartext Considered Obsolete](https://datatracker.ietf.org/doc/html/rfc8314)
- [RFC 9846 – The Transport Layer Security (TLS) Protocol Version 1.3](https://datatracker.ietf.org/doc/html/rfc9846) – Handshake, Record Layer, nøkkelplan, gjenopptakelse og sikkerhetsmodell.
- [RFC 3207 – SMTP Service Extension for Secure SMTP over TLS](https://datatracker.ietf.org/doc/html/rfc3207) – STARTTLS, SMTP-tilstandsovergang og opportunistisk policy.
- [RFC 9001 – Using TLS to Secure QUIC](https://datatracker.ietf.org/doc/html/rfc9001) – avgrensning mellom TLS-handshake og QUIC-pakkebeskyttelse.
- [RFC 5280 – Internet X.509 Public Key Infrastructure Certificate and CRL Profile](https://datatracker.ietf.org/doc/html/rfc5280) – sertifikatfelt, utvidelser og stivalidering.
- [RFC 7468 – Textual Encodings of PKIX, PKCS, and CMS Structures](https://datatracker.ietf.org/doc/html/rfc7468) – PEM-tekstinnpakning.
- [RFC 7292 – PKCS #12](https://datatracker.ietf.org/doc/html/rfc7292) – container for nøkler og sertifikater.
- [RFC 6960 – Online Certificate Status Protocol](https://datatracker.ietf.org/doc/html/rfc6960) – OCSP-statuskontroll.
- [RFC 9525 – Service Identity in TLS](https://datatracker.ietf.org/doc/html/rfc9525) – referanseidentiteter, subjectAltName, wildcards og navnekontroll.
- [RFC 6066, avsnitt 3](https://datatracker.ietf.org/doc/html/rfc6066)
- [RFC 7301 – TLS Application-Layer Protocol Negotiation](https://datatracker.ietf.org/doc/html/rfc7301) – ALPN-forhandling.
- [IANA – Service Name and Transport Protocol Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) – registrerte porter og tjenestenavn.
- [RFC 4511, avsnitt 4.14](https://datatracker.ietf.org/doc/html/rfc4511)
- [RFC 7672 – SMTP Security via DANE TLS](https://datatracker.ietf.org/doc/html/rfc7672) – DNSSEC-/TLSA-basert SMTP-autentisering.
- [RFC 8461 – SMTP MTA Strict Transport Security](https://datatracker.ietf.org/doc/html/rfc8461) – MTA-STS-policy og MX-kontroll.
- [RFC 8460 – SMTP TLS Reporting](https://datatracker.ietf.org/doc/html/rfc8460) – aggregerte TLS-feilrapporter.
- [NIST SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final) – nøkkeladministrasjon og beskyttelsesbehov.
- [RFC 8555 – Automatic Certificate Management Environment](https://datatracker.ietf.org/doc/html/rfc8555) – automatisert utstedelse og fornyelse.
- [OpenSSL](https://docs.openssl.org/master/)
- [Kildekode](https://github.com/openssl/openssl)
- [Microsoft Learn – Schannel SSP](https://learn.microsoft.com/windows-server/security/tls/tls-ssl-schannel-ssp-overview) – Windows-TLS-implementering og SSPI-kontekst.
- [Oracle – JSSE Reference Guide](https://docs.oracle.com/en/java/javase/24/security/java-secure-socket-extension-jsse-reference-guide.html) – Java-TLS-API-er, KeyManager og TrustManager.
- [GnuTLS Documentation](https://www.gnutls.org/documentation.html) – alternativt native TLS-bibliotek og verktøyfamilie.
- [BCP 195 – Recommendations for Secure Use of TLS and DTLS](https://www.rfc-editor.org/info/bcp195) – videreført IETF-sikkerhetsveiledning for TLS-konfigurasjon.
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) – TCP-tilgjengelighet i Windows.
- [OpenBSD Manual – nc](https://man.openbsd.org/nc) – TCP-forbindelsestest i Unix.
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/) – generell TLS- og STARTTLS-diagnoseklient.
- [Microsoft Learn – certutil](https://learn.microsoft.com/windows-server/administration/windows-commands/certutil) – Windows-sertifikatlager og Chain Engine.
- [OpenSSL – x509](https://docs.openssl.org/master/man1/openssl-x509/) – sertifikatfelt og utdataalternativer.
- [OpenSSL – verify](https://docs.openssl.org/master/man1/openssl-verify/) – eksplisitt stibygging og stivalidering.
- [Oracle – keytool](https://docs.oracle.com/en/java/javase/25/docs/specs/man/keytool.html) – administrasjon av Java-Key- og Truststores.
- [Wireshark User’s Guide](https://www.wireshark.org/docs/wsug_html/index.html) – analyse av Handshake, Record og Session Key.
- [RFC 6176 – Prohibiting SSL Version 2.0](https://datatracker.ietf.org/doc/html/rfc6176) – formell utfasing av SSL 2.0.
