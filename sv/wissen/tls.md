---
title: "TLS: Protokoll, certifikatvalidering och drift"
blatt: "tls"
description: "Transport Layer Security ur ett administratörsperspektiv: handshake och Record Layer, TLS 1.3-nyckelschema, X.509- och PKIX-sökvägsvalidering, SAN och SNI, truststores, SMTP- och LDAP-integration, nyckelhantering samt skiktad felsökning."
fakten:
  - label: Fullständigt namn
    wert: Transport Layer Security
    href: https://datatracker.ietf.org/doc/html/rfc9846
  - label: Skyddsmål
    wert: Autentisering · Konfidentialitet · Integritet
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-1
  - label: Protokolldelar
    wert: Handshake Protocol · Record Protocol · Alerts
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-1
  - label: Underlag
    wert: tillförlitlig, ordnad byteström
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-1
  - label: Autentisering
    wert: Server alltid · klient valfritt
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-1
  - label: Nyckelutbyte
    wert: (EC)DHE · PSK · kombinerat
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-2
  - label: Record-skydd
    wert: AEAD med separata Traffic Secrets
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-5
  - label: Certifikatprofil
    wert: X.509 v3 och PKIX-sökvägsvalidering
    href: https://datatracker.ietf.org/doc/html/rfc5280
  - label: Tjänsteidentitet
    wert: Referens-ID mot subjectAltName
    href: https://datatracker.ietf.org/doc/html/rfc9525
  - label: Startmodeller
    wert: implicit TLS eller protokollspecifikt STARTTLS
    href: https://datatracker.ietf.org/doc/html/rfc8314
  - label: Återupptagning
    wert: PSK-baserad Session Resumption
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-2.2
  - label: Driftrekommendation
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
translatedAt: 2026-10-14T09:45:50.717Z
translationReview: required
---

# TLS: Protokoll, certifikatvalidering och drift

**Transport Layer Security** (TLS) skyddar en enskild anslutning mellan två ändpunkter. Först kommer klient och server överens om kryptografiska metoder och nycklar, därefter går SMTP, LDAP, HTTP eller ett annat applikationsprotokoll genom denna skyddade kanal. TLS förhindrar avlyssning och obemärkta ändringar på denna anslutning och autentiserar minst servern. Det bekräftar dock varken avsändaren av ett e-postmeddelande eller garanterar att nästa SMTP-hopp också använder TLS ([RFC 9846, avsnitt 1](https://datatracker.ietf.org/doc/html/rfc9846#section-1), [RFC 3207, avsnitt 6](https://datatracker.ietf.org/doc/html/rfc3207#section-6)).

I drift måste därför flera kontroller lyckas i följd: Rätt värd och port måste vara nåbara, klient och server behöver gemensamma TLS-parametrar, certifikatkedjan måste leda till ett betrott ankare och det adresserade namnet måste finnas i certifikatet. Först därefter börjar det egentliga applikationsprotokollet. Artikeln följer exakt detta förlopp och visar vid varje punkt vad en administratör måste observera, konfigurera och säkra.

## Position i protokollstacken

I vanlig serverdrift ligger TLS mellan [TCP](/kb/tcp) och applikationsprotokollet. TCP levererar den ordnade dataströmmen; TLS skyddar den; [SMTP](/kb/smtp), [LDAP](/kb/ldap), IMAP eller HTTP anger när TLS startar och vilket servernamn som förväntas. Denna inbäddning förklarar varför en öppen port ännu inte utgör ett lyckat TLS-test och varför en lyckad handshake ännu inte bevisar en fungerande e-post- eller katalogtjänst ([RFC 9846, avsnitt 1](https://datatracker.ietf.org/doc/html/rfc9846#section-1)).

| Nivå | Synligt objekt | Typisk adminfråga |
|---|---|---|
| Namn- och tjänsteval | DNS-namn, MX- eller SRV-mål, port | Vilken ändpunkt valdes faktiskt? |
| Transport | TCP-anslutning och tillståndsautomat | Kommer SYN, SYN-ACK och ACK fram i båda riktningarna? |
| TLS-handshake | Version, Key Share, signatur, certifikat, Finished | Finns gemensamma parametrar och accepteras peeren? |
| TLS Record Layer | krypterade records med AEAD-skydd | Överförs, förkastas eller avslutas records med Alert? |
| Applikationsprotokoll | SMTP-, LDAP-, IMAP- eller HTTP-dialog | Vilka kommandon och identiteter gäller efter handshaken? |

QUIC är ett viktigt undantag från den klassiska bilden av «TLS över TCP»: det använder TLS 1.3-handshaken men inte TLS Record Layer; QUIC skyddar sina egna paket med nycklar som härleds från TLS ([RFC 9001](https://datatracker.ietf.org/doc/html/rfc9001)). För de e-post- och katalogprotokoll som behandlas här förblir den TCP-baserade TLS-kanalen den relevanta modellen.

## Arkitektur: Handshake, Record Layer och Alerts

TLS utför två uppgifter i följd. I **Handshake Protocol** förhandlar motparterna fram parametrar, autentiserar sig och skapar gemensamt nyckelmaterial. Därefter delar **Record Protocol** upp nyttolasten i skyddade records. **Alerts** rapporterar protokollfel eller ett ordnat avslut; vid ett abrupt nätverks- eller processfel kan TCP-anslutningen dock även avslutas utan användbar Alert ([RFC 9846, avsnitt 4 till 6](https://datatracker.ietf.org/doc/html/rfc9846#section-4)).

Denna uppdelning är avgörande i drift. Ett certifikatfel hör till autentiseringsfasen i handshaken. Ett `bad_record_mac` eller `decrypt_error` avser kryptografisk bearbetning. En timeout utan ServerHello tyder däremot ofta på protokollförväxling, filtrering, fel port eller en server som först förväntar sig en klartextdialog och ett STARTTLS-kommando.

## TLS 1.3-handshake steg för steg

Den fullständiga TLS 1.3-handshaken kan läsas som ett samtal: Klienten lämnar ett erbjudande, servern väljer ur detta, styrker sin identitet och båda sidor bekräftar det hittillsvarande protokollet. Redan efter **ServerHello** är de fortsatta handshake-meddelandena skyddade. Med de ömsesidiga **Finished**-meddelandena står det slutligen klart att båda parter har samma transkript och samma hemligheter ([RFC 9846, avsnitt 2](https://datatracker.ietf.org/doc/html/rfc9846#section-2)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1056" src="/images/kb-interaktiv-tls-handshake.svg?v=20260813" title="Interaktive Infografik: TLS-1.3-Handshake, Zertifikatskette, Identitätsprüfung und Admin-Diagnosepunkte" loading="lazy">
  <a href="/images/kb-interaktiv-tls-handshake.svg?v=20260813">Öppna infografik om TLS-handshake och certifikatvalidering</a>
</iframe>

Förloppet i detalj:

1. **ClientHello** erbjuder bland annat TLS-versioner, Cipher Suites, signaturalgoritmer och kryptografiska grupper som stöds. En `key_share` innehåller vanligtvis redan klientens efemära publika andel. SNI kan överföra det önskade servernamnet och ALPN möjliga applikationsprotokoll.
2. **ServerHello** väljer version, Cipher Suite och Key Share. Klientens och serverns andelar skapar den gemensamma (EC)DHE-hemligheten. Om ingen lämplig i förväg skickad grupp finns kan servern med **HelloRetryRequest** kräva ett nytt ClientHello.
3. **EncryptedExtensions** bekräftar ytterligare förhandlade utökningar. Om servern begär ett klientcertifikat följer dessutom **CertificateRequest**.
4. **Certificate** överför Leaf-certifikatet och normalt de Intermediate-certifikat som klienten behöver. **CertificateVerify** signerar handshake-transkriptet med den privata nyckel som hör till Leaf-certifikatet. Därmed bevisar servern nyckelinnehav; certifikatet ensamt bevisar det inte.
5. **Finished** är en MAC över det hittillsvarande transkriptet och bekräftar de härledda handshake-hemligheterna. Klienten validerar parametrar, certifikatsökväg och tjänsteidentitet och skickar sedan sitt Finished, vid begärd ömsesidig autentisering dessutom Certificate och CertificateVerify.
6. För applikationsdata härleds separata Traffic Secrets. Senare **NewSessionTicket**-meddelanden möjliggör PSK-baserad återupptagning utan att servern måste skicka det ursprungliga certifikatet igen vid varje efterföljande handshake.

Det observerade antalet Round Trips beror därför på DNS, TCP, fullständig eller återupptagen TLS-handshake, HelloRetryRequest och applikationen. En Packet Capture måste mäta dessa faser separat; «TLS var långsamt» är ingen diagnos utan denna uppdelning.

## Nyckelschema och Record Layer

Av överenskommet (EC)DHE- eller PSK-material skapar TLS 1.3 inte en enda sessionsnyckel utan flera separata hemligheter med **HKDF**. Handshake, applikationsdata, exporter och senare återupptagning använder eget material; dessutom har klient och server separata nycklar och initialiseringsvektorer för varje riktning. Denna uppdelning är viktig för analys och Incident Response, eftersom en exponerad Record-nyckel inte automatiskt dekrypterar alla faser och riktningar ([RFC 9846, avsnitt 7.1](https://datatracker.ietf.org/doc/html/rfc9846#section-7.1)).

Record Layer använder uteslutande **Authenticated Encryption with Associated Data** (AEAD). Cipher Suite anger i TLS 1.3 AEAD-algoritmen och hashfunktionen för HKDF; certifikattyp, signaturalgoritm och Key Exchange-grupp förhandlas separat. Detta skiljer sig från TLS 1.2, där namn som `TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256` samlar flera beslut i en Suite.

| Egenskap | TLS 1.2 | TLS 1.3 |
|---|---|---|
| Cipher Suite-semantik | Nyckelutbyte, autentisering, kryptering och hash sammanbundna | AEAD-metod och HKDF-hash; Key Exchange och signatur separat |
| Forward Secrecy | beror på vald Suite | asymmetrisk nyckelförhandling endast med efemär (EC)DHE |
| Handshake-synlighet | stora delar i klartext | krypterad efter ServerHello |
| Återupptagning | Session ID eller Ticket enligt äldre modell | enhetlig PSK-modell |
| 0-RTT | inte del av protokollet | valfritt, men kan återupprepas och är inte lämpligt för icke-idempotenta operationer |

0-RTT sparar en Round Trip vid återupptagning men har svagare Replay-egenskaper. En server måste därför beroende på applikationen avgöra vilka tidiga data den accepterar. För tillståndsförändrande admin-, e-post- eller katalogoperationer får låg latens inte förväxlas med Replay-säkerhet ([RFC 9846, avsnitt 2.3 och 8](https://datatracker.ietf.org/doc/html/rfc9846#section-2.3)).

## X.509-certifikat och privat nyckel

Ett X.509-certifikat är den signerade beskrivningen av en publik nyckel. Det anger bland annat utfärdare, giltighetsperiod, tillåtna användningar och de namn för vilka nyckeln får användas. Den tillhörande privata nyckeln ingår inte i certifikatet och måste förbli skyddad hos operatören. RFC 5280 definierar PKIX-profilen som används för detta, med fält som Key Usage, Extended Key Usage, Basic Constraints och Subject Alternative Name ([RFC 5280, avsnitt 4](https://datatracker.ietf.org/doc/html/rfc5280#section-4)).

X.509 kodas binärt med ASN.1 DER. De ofta synliga raderna `BEGIN CERTIFICATE` och `END CERTIFICATE` är ett Base64-baserat textomslag som RFC 7468 beskriver. En **PKCS #12**-container kan däremot transportera privat nyckel, certifikatkedja och ytterligare attribut tillsammans ([RFC 7468](https://datatracker.ietf.org/doc/html/rfc7468), [RFC 7292](https://datatracker.ietf.org/doc/html/rfc7292)). Filändelser som `.cer`, `.crt`, `.pem`, `.p12` eller `.pfx` är konventioner, men inget tillförlitligt bevis på innehåll eller kodning.

Viktiga utökningar har olika uppgifter:

| Utökning | Kontrollfråga | Vanligt driftfel |
|---|---|---|
| Subject Alternative Name | innehåller det DNS-, IP-, URI- eller SRV-identiteten som klienten förväntar sig? | Managementnamn eller Load Balancer-alias saknas |
| Basic Constraints | får certifikatet agera som CA och hur djupt får sökvägen bli? | Intermediate utan `CA=TRUE` eller felaktig Path Length |
| Key Usage | för vilken grundläggande kryptografisk operation får nyckeln användas? | CA får inte signera eller Leaf passar inte för signaturanvändningen |
| Extended Key Usage | är nyckeln tillåten för server- eller klientautentisering? | S/MIME-certifikat används av misstag som TLS-servercertifikat |
| Authority / Subject Key Identifier | vilka nyckelvarianter hjälper vid sökvägsbyggandet? | fel Intermediate med samma Subject väljs |
| CRL Distribution Points / AIA | var hittar klienten information om återkallelse eller Issuer? | målet är inte nåbart från servernätet eller proxy saknas |

## Certifikatsökväg och förtroendegräns

Med sitt **Leaf-certifikat** skickar en server normalt de nödvändiga **Intermediate-certifikaten**. Klienten försöker därifrån bygga en sökväg till en Root CA som den redan litar på lokalt. Just denna sista punkt är avgörande: En Root CA som medföljer från servern blir inte betrodd enbart av den anledningen. Truststoren för den process som faktiskt ansluter avgör vilka trust anchors som gäller ([RFC 5280, avsnitt 6](https://datatracker.ietf.org/doc/html/rfc5280#section-6)).

Sökvägsbyggande och sökvägsvalidering är mer än tre enkla kryssrutor. Klienten måste bland annat kontrollera signaturer, giltighetsintervall, Basic Constraints, Path Length, Name Constraints, Key Usage, Extended Key Usage, kritiska utökningar, policyer och vid behov återkallelse. Därefter följer den protokollspecifika tjänsteidentiteten. Olika klienter kan konstruera olika giltiga sökvägar från samma Leaf-certifikat, eftersom deras truststores, AIA-hämtning, Cross-Signatures och algoritmregler skiljer sig åt.

Begreppet **Truststore** betecknar därför ingen universell plats. En Windows-applikation kan använda Schannel och Windows certifikatslager, en Java-applikation JSSE med en egen `cacerts`- eller applikationsfil, en container ett inbyggt CA-bundle och en appliance en proprietär store. Även på samma värd kan två processer lita olika på samma server. Vid varje störning måste den process som faktiskt validerar och dess Truststore identifieras.

Återkallelse är ett ytterligare policybeslut. CRL:er distribuerar signerade listor över återkallade certifikat; OCSP besvarar statusförfrågningar för enskilda certifikat. Om en klient kontrollerar, hur den reagerar vid timeout och om den använder OCSP Stapling beror på biblioteket och applikationen ([RFC 5280, avsnitt 5](https://datatracker.ietf.org/doc/html/rfc5280#section-5), [RFC 6960](https://datatracker.ietf.org/doc/html/rfc6960)). «Certifikatet är tidsmässigt giltigt» innebär därför inte automatiskt «certifikatet accepteras av denna klient».

## Tjänsteidentitet: SAN, referens-ID och SNI

Efter att sökvägen har accepterats kontrollerar klienten tjänstens namn. Utgångspunkten är **referensidentiteten**, vanligen det konfigurerade värdnamnet eller värddelen i en URL. Den måste matcha en lämpligt typad post i **Subject Alternative Name** i Leaf-certifikatet. För DNS-namn gäller `dNSName`, för direkt konfigurerade IP-adresser `iPAddress`; det fritt formulerade Common Name är ingen modern reservlösning. Wildcards får endast ersätta den fullständiga vänstra labeln ([RFC 9525, avsnitt 4 och 6](https://datatracker.ietf.org/doc/html/rfc9525#section-4)).

**Server Name Indication** (SNI) måste särskiljas från detta. Klienten skickar önskat servernamn i ClientHello så att en gemensam listener kan välja lämpligt certifikat och rätt virtuell konfiguration. SNI berättar för servern vad klienten vill nå; det ersätter varken SAN-jämförelsen eller sökvägsvalideringen ([RFC 6066, avsnitt 3](https://datatracker.ietf.org/doc/html/rfc6066#section-3)). Ett test utan SNI kan därför visa standardcertifikatet, trots att produktionsklienten får ett annat.

**Application-Layer Protocol Negotiation** (ALPN) låter klient och server välja ett högre protokoll inom TLS-handshaken. Det syns främst för HTTP och QUIC, men är konceptuellt inte heller någon certifikatvalidering ([RFC 7301](https://datatracker.ietf.org/doc/html/rfc7301)).

## Implicit TLS och STARTTLS

När handshaken startar bestäms av applikationsprotokollet. Vid **implicit TLS** följer ClientHello direkt efter TCP-uppbyggnaden. Vid **STARTTLS** talar båda sidor först i klartext och byter först efter ett protokollspecifikt kommando till TLS. Den som skickar ClientHello direkt till en STARTTLS-port använder därför visserligen TLS, men vid fel punkt i dialogen.

| Tjänsteväg | Port | Startmodell | Protokollregel |
|---|---:|---|---|
| SMTP-relä mellan MTA:er | 25 | ESMTP `STARTTLS` | efter lyckad handshake återställs SMTP; klienten skickar EHLO igen |
| Message Submission | 587 | STARTTLS | Submission kan kräva TLS före autentisering |
| Implicit TLS Submission | 465 | TLS omedelbart | beskrivs av RFC 8314 för Mail Submission |
| IMAP / IMAPS | 143 / 993 | STARTTLS / TLS omedelbart | klartextåtkomst anses föråldrad för modern användning |
| POP3 / POP3S | 110 / 995 | STLS / TLS omedelbart | även här separata klartext- och Implicit-TLS-vägar |
| LDAP / LDAPS | 389 / 636 | LDAP StartTLS Extended Operation / TLS omedelbart | Bind och identitetskontroll följer LDAP-säkerhetspolicyn |
| HTTP / HTTPS | 80 / 443 | separata URI-scheman; TLS omedelbart på 443 | Host/SNI, certifikatidentitet och HTTP-version är separata beslut |

Porttilldelningarna kommer från [IANA Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml); semantiken följer respektive applikationsstandard. Port 465 är därför inte generellt «SMTPS för server-till-server-leverans», utan den registrerade Implicit-TLS-vägen för Submission.

SMTP STARTTLS definierar en särskilt viktig tillståndsövergång. Före uppgraderingen kan EHLO-svaret och `STARTTLS`-erbjudandet manipuleras. Efter handshaken måste båda sidor förkasta tidigare inlärd SMTP-information, och klienten skickar EHLO på nytt ([RFC 3207, avsnitt 4.2](https://datatracker.ietf.org/doc/html/rfc3207#section-4.2)). För LDAP är StartTLS en Extended Operation inom LDAP-sessionen; den är inte samma sak som en ny TCP-port ([RFC 4511, avsnitt 4.14](https://datatracker.ietf.org/doc/html/rfc4511#section-4.14)).

## TLS-policy för SMTP

För SMTP på port 25 är TLS ofta opportunistiskt: Den sändande MTA:n krypterar när motparten erbjuder STARTTLS och den lokala policyn accepterar handshaken. Utan ytterligare policy kan en angripare ta bort erbjudandet eller störa handshaken. Den avsändande MTA:n måste då avgöra om den ska leverera okrypterat, försöka igen senare eller avbryta definitivt. RFC 3207 klargör därmed att ett skyddat SMTP-hopp inte är ett end-to-end-skydd för meddelandet.

Två mekanismer kan binda mottagardomänen och TLS-kravet starkare:

- **DANE for SMTP** publicerar TLSA-data under DNSSEC och härleder den identitet som ska kontrolleras från MX-sökvägen. En validerande resolver och en intakt DNSSEC-kedja blir därmed en del av förtroendebasen ([RFC 7672](https://datatracker.ietf.org/doc/html/rfc7672)).
- **MTA-STS** publicerar en policy via DNS och HTTPS. Den anger tillåtna MX-mönster och kräver PKIX-validerad TLS under policyns giltighetstid ([RFC 8461](https://datatracker.ietf.org/doc/html/rfc8461)). **SMTP TLS Reporting** rapporterar aggregerade framgångs- och felbilder för dessa policyer ([RFC 8460](https://datatracker.ietf.org/doc/html/rfc8460)).

Dessa modeller är inte utbytbara. DANE förankrar uttalandet i DNSSEC och TLSA; MTA-STS använder Web-PKIX och en policy som hämtas via HTTPS. En administratör måste därför utöver certifikat och Cipher även övervaka [DNS](/kb/dns), MX-val, policycache och rapportkanal.

## Ömsesidig TLS-autentisering

**Mutual TLS** (mTLS) kompletterar det vanliga serverbeviset med ett klientcertifikat. Servern begär det med CertificateRequest; klienten skickar sin kedja och bevisar med CertificateVerify innehavet av den privata nyckeln. Därefter kontrollerar servern sökväg, giltighet, Extended Key Usage och sin lokala mappning. Certifikatet identifierar först en nyckelinnehavare. Vilka rättigheter för applikationskonto, LDAP eller SMTP-relä som följer av detta avgörs fortfarande av applikationen.

I heterogena miljöer uppstår ofta asymmetriska trustproblem: Klienten litar på server-CA:n, men servern litar inte på klient-CA:n; servern begär endast vissa Issuers; klientcertifikatet har endast `serverAuth`; eller en Load Balancer terminerar TLS och vidarebefordrar ingen verifierad klientidentitet till backend. En lyckad serverautentiserad handshake är därför inget belägg för fungerande mTLS.

## Drift av nycklar, certifikat och förnyelse

Ett certifikatbyte är mer än att ersätta en fil. Nyckel, certifikat, Intermediate-kedja, listenerbindning och utrullningsstatus måste passa ihop. En robust livscykel omfattar minst:

1. **Inventering:** Registrera tjänstenamn, port, startmodell, terminerande process, förväntade SAN:er, Issuer, serienummer, fingerprint, `notBefore`/`notAfter`, Key Store, Truststore och ansvarig automation.
2. **Nyckelgenerering:** Skapa ett nytt nyckelpar i avsedd mjukvaru-, TPM- eller HSM-kontext. Privata nycklar förblir konfidentiella och behöver skydd, åtkomstkontroll och, beroende på återställningsmodell, en dokumenterad strategi för säkerhetskopiering eller ny utfärdning ([NIST SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)).
3. **Ansökan och utfärdande:** Skapa en CSR med korrekta identiteter och parametrar; uppfyll CA-policy och erforderliga bevis. ACME standardiserar automatiserad beställning och förnyelse, men garanterar ännu ingen korrekt tjänstebindning ([RFC 8555](https://datatracker.ietf.org/doc/html/rfc8555)).
4. **Installation:** Placera Leaf, passande Intermediate-bundle och privat nyckel i det format som processen faktiskt läser. Kontrollera filrättigheter, lösenordsåtkomst och Key-Pair-matchning.
5. **Aktivering:** Ladda om tjänsten eller växla bindningen atomärt. Hantera varje TLS-terminator separat i kluster, Load Balancer och flera protokollroller.
6. **Validering:** Kontrollera externt och internt med produktivt SNI, port och STARTTLS-läge; registrera levererad kedja, SAN, giltighet, algoritm, policy och applikationstest.
7. **Avveckling:** Ta bort det gamla certifikatet först när alla noder levererar det nya och inget beroende längre hänvisar till fingerprint eller gammal Issuer. Vid kompromettering ska det dessutom återkallas och den privata nyckeln behandlas som komprometterad.

En säkerhetskopia måste passa denna modell. En exporterad offentlig certifikatdel återställer inte en tjänst om privat nyckel, lösenord, providerbindning, ACL eller HSM-objekt saknas. Omvänt ökar varje oskyddad kopia av den privata nyckeln attackytan. Återställning av Root- och Issuing-CA, återställning av servernyckel och snabb nyutfärdning är olika förfaranden och får inte sammanföras under «säkerhetskopiera certifikat».

Offentlig Web-PKI, intern Enterprise-PKI och självhanterade trust anchors har olika regler för utfärdande och giltighetstid. I stället för att använda ett statiskt maxvärde som driftmål bör övervakningen mäta det konkreta `notAfter` för varje levererat certifikat och planera tillräckligt med tid för utfärdande, utrullning, validering och rollback.

## Implementerings- och teknikstack

Vilken TLS-stack som faktiskt arbetar beror på processen. Windows-tjänster kan använda Schannel, Java-applikationer JSSE, Unix-tjänster OpenSSL eller GnuTLS och appliances ett eget bibliotek. Därmed ändras Truststore, stödda metoder, diagnosverktyg och konfigurationsplats, trots att alla motparter talar «TLS».

| Stack | Körtid och API | Typisk trust- och nyckelkälla | Adminrelevans |
|---|---|---|---|
| [OpenSSL](https://docs.openssl.org/master/) ([Källkod](https://github.com/openssl/openssl)) | native bibliotek och CLI, huvudsakligen C | CA-fil/-katalog, PEM, DER, PKCS #12, Provider | många Unix-tjänster och appliances; `s_client`, `x509`, `verify` |
| [Schannel](https://learn.microsoft.com/windows-server/security/tls/tls-ssl-schannel-ssp-overview) | Windows Security Support Provider via SSPI | Windows certifikatslager, CNG/CSP-nycklar | System- och .NET-applikationer kan använda OS-policy och stores |
| [JSSE](https://docs.oracle.com/en/java/javase/24/security/java-secure-socket-extension-jsse-reference-guide.html) | Java-API:er `SSLSocket`, `SSLEngine`, `SSLContext` och Provider | KeyStore, TrustStore, `cacerts`, PKCS #12 | JVM-applikation kan avvika från operativsystemets trust |
| [GnuTLS](https://www.gnutls.org/documentation.html) | native TLS-bibliotek, C-API och CLI | filer, PKCS #11, system-trust beroende på integration | alternativt bibliotek i Unix- och GNU-programvara |

Ovanpå detta ligger applikationskonfigurationen. En e-postserver kan välja certifikat och policy per listener eller SNI-namn; en Reverse Proxy terminerar TLS framför backend; en Java-produkt levereras med egen JRE och Truststore; en container-image fryser ett CA-bundle vid byggtid. «Certifikatet är installerat i hela operativsystemet» är därför inget robust påstående utan bevis för process och bindning.

Även algoritmstöd är en skärningsmängd av protokollversion, bibliotek, kryptoprovider, certifikat, klienterbjudande och lokal policy. Ett ECDSA-certifikat hjälper inte om klienten inte stöder signaturen; en TLS 1.3-Cipher-string ändrar inga TLS 1.2-Suites; en systemomfattande Schannel-inställning styr inte en separat medföljande OpenSSL- eller Java-körtid. [BCP 195](https://www.rfc-editor.org/info/bcp195) erbjuder IETF:s säkerhetsriktlinje, men den konkreta aktiveringen förblir implementations- och livscykelberoende.

## Diagnos från port till applikation

En TLS-diagnos börjar därför inte med certifikatet utan med den exakta anslutningsvägen. Varje test omfattar målnamn, upplöst adress, port, SNI, STARTTLS-läge, klientbibliotek, Truststore, tidpunkt och gällande policy. Endast tester med samma parametrar är jämförbara.

### Kontrollera TCP-nåbarhet separat

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

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) och [`nc`](https://man.openbsd.org/nc) kontrollerar här endast namnuppslagning och TCP-uppbyggnad. Ett lyckat test säger ännu inget om TLS-version, certifikat, SNI eller applikation.

### Testa SMTP STARTTLS med produktivt SNI

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

[`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) är en diagnosklient, inte ett bevis på produktions-MTA:ns policy. `-starttls smtp` utför klartextdialogen fram till uppgraderingen, `-servername` anger SNI, `-showcerts` visar certifikatlistan som servern skickar och `-verify_return_error` avbryter vid ett valideringsfel. För ett fullständigt identitetstest bör dessutom `-verify_hostname mail.example.ch` och den faktiskt använda CA-sökvägen anges. OpenSSL är inte en inbyggd systemkomponent i Windows; `openssl.exe` måste komma från en kontrollerad installation.

### Skilj LDAP StartTLS och implicit TLS åt

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

Det första anropet förväntar sig TLS direkt på 636, det andra utför LDAP StartTLS Extended Operation på 389. Ett lyckat OpenSSL-test bevisar ännu inte att produktions-LDAP-biblioteket använder samma Truststore, samma återkallelsepolicy eller samma referensidentitet.

### Undersök certifikatfil och sökväg lokalt

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

[`certutil`](https://learn.microsoft.com/windows-server/administration/windows-commands/certutil) använder Windows certifikatlogik för `-verify` och kan inkludera ett servernamn med `-sslpolicy`. [`openssl x509`](https://docs.openssl.org/master/man1/openssl-x509/) visar fält i ett certifikat; [`openssl verify`](https://docs.openssl.org/master/man1/openssl-verify/) bygger och kontrollerar en sökväg mot de uttryckligen angivna Trust Anchors och Intermediates. [`keytool`](https://docs.oracle.com/en/java/javase/25/docs/specs/man/keytool.html) är det lämpliga officiella hanterings- och inspektionsgränssnittet för JVM-Key- och Truststores; ett test med OpenSSL ersätter inte dess perspektiv.

### Koppla felbild till kontrollsteg

| Observation | Trolig nivå | Nästa kontroll |
|---|---|---|
| TCP-timeout | DNS, rutt, brandvägg eller listener | upplöst adress, SYN/SYN-ACK, returväg, listenerbindning |
| Klartextbanner i stället för TLS-svar | fel startmodell | port och `-starttls smtp` respektive `-starttls ldap` |
| omedelbar TLS-Alert efter ClientHello | inga gemensamma parametrar eller policy | versioner, grupper, signaturalgoritmer, Cipher, SNI |
| fel certifikat | SNI eller listenerbindning | skickat SNI, Default-VHost, Load Balancer-konfiguration |
| `unable to get local issuer certificate` | saknat Intermediate eller Trust Anchor | serverlista, AIA, klient-Truststore, sökvägsbyggande |
| `hostname mismatch` | referens-ID matchar inte SAN | faktiskt använt namn, SAN-typ, alias och wildcardgräns |
| endast Java-applikation misslyckas | JSSE-policy eller egen Truststore | JVM, `cacerts`, app-Truststore, disabledAlgorithms |
| endast enskilda klusternoder levererar gammalt certifikat | utrullning eller bindning | testa varje ändpunkt och varje IP med samma SNI |
| SMTP faller periodvis tillbaka till klartext | opportunistisk policy eller downgrade | MTA-loggar, STARTTLS-erbjudande, MTA-STS/DANE-status, TLS-RPT |
| mTLS-handshake lyckas, åtkomst är fortfarande förbjuden | auktorisering efter TLS | klient-EKU, certifikatmappning, proxyvidarebefordran, applikationsroll |

[`Wireshark`](https://www.wireshark.org/docs/wsug_html/index.html) kan synliggöra ClientHello, ServerHello, utökningar och Alerts; utan sessionsnycklar förblir applikationsdata och många TLS 1.3-handshake-meddelanden krypterade. En inspelning visar dessutom inte automatiskt vilken Truststore applikationen använder eller varför dess policy förkastar en sökväg.

## Övervakning och driftkriterier

En portkontroll och en varning före utgångsdatum räcker inte. Övervakning bör kontrollera samma väg som en verklig klient: välja mål, upprätta handshake, validera certifikat och namn och sedan nå applikationsprotokollet. Följande signaler är särskilt användbara:

- målval och SNI per tjänst, listener och klusternod;
- förhandlat protokoll, Cipher Suite, grupp och signaturalgoritm;
- Leaf-fingerprint, serienummer, SAN:er, Issuer samt `notBefore` och `notAfter`;
- faktiskt skickad Intermediate-kedja och sökvägsvalidering ur relevanta Truststores perspektiv;
- OCSP-/CRL-nåbarhet och det verkliga Fail-open-/Fail-closed-beteendet;
- handshake-latens, andel återupptagning, Alert-klasser och felfrekvens;
- STARTTLS-erbjudande, lyckade uppgraderingar och klartext-fallbacks per SMTP-mål;
- MTA-STS-, DANE- och TLS-RPT-status där dessa policymechanismer används;
- överensstämmelse mellan alla TLS-terminatorer efter förnyelse och lyckat applikationstest bakom TLS-lagret.

Övervakning enbart inifrån missar offentlig kedja, externt DNS eller Edge-terminatorer; en extern kontroll ensam ser inte interna Truststores och mTLS. För meddelande- och katalogplattformar krävs därför minst en extern klientväg, en intern applikationsväg och en direkt kontroll av varje terminator.

## Teknisk historia

Historien börjar med Netscapes SSL på 1990-talet. SSL 2.0 förbjöds senare uttryckligen ([RFC 6176](https://datatracker.ietf.org/doc/html/rfc6176)); SSL 3.0 är numera endast dokumenterat som ett historiskt protokoll och har varit otillåtet sedan RFC 7568. IETF utvecklade därefter **TLS 1.0** 1999, **TLS 1.1** 2006 och **TLS 1.2** 2008. RFC 8996 förklarade TLS 1.0 och 1.1 för föråldrade 2021. För administratörer förklarar detta ursprung varför gamla produktnamn, Cipher-syntax och kompatibilitetsväxlar fortfarande förekommer i konfigurationer.

**TLS 1.3** kom 2018 med RFC 8446. Det tog bort statiskt RSA-nyckelutbyte, äldre icke-AEAD-metoder och ytterligare Legacy-funktioner, krypterade handshaken tidigare och införde det HKDF-baserade nyckelschemat samt PSK-modellen för återupptagning. RFC 9846 ersatte RFC 8446 i juli 2026 som konsoliderad TLS 1.3-specifikation och integrerade korrigeringar och krav som fram till dess hade dokumenterats separat. Protokollvärdet `1.3` förblev detsamma; det normativa standarddokumentet ändrades ([RFC 9846, avsnitt 1.2](https://datatracker.ietf.org/doc/html/rfc9846#section-1.2)).

Parallellt utvecklades identitets- och driftssidan: X.509-PKIX konsoliderades i RFC 5280, SNI och andra utökningar i RFC 6066, ALPN i RFC 7301, automatiserad utfärdning i ACME/RFC 8555 och modern tjänsteidentitetskontroll i RFC 9525. TLS-historien är därför inte bara en versionsföljd, utan den gemensamma utvecklingen av kryptografi, certifikatsökvägar, applikationsinbäddning och operativ automation.

## Källor

- [IETF RFC 8314 – Cleartext Considered Obsolete](https://datatracker.ietf.org/doc/html/rfc8314)
- [RFC 9846 – The Transport Layer Security (TLS) Protocol Version 1.3](https://datatracker.ietf.org/doc/html/rfc9846) – Handshake, Record Layer, nyckelschema, återupptagning och säkerhetsmodell.
- [RFC 3207 – SMTP Service Extension for Secure SMTP over TLS](https://datatracker.ietf.org/doc/html/rfc3207) – STARTTLS, SMTP-tillståndsövergång och opportunistisk policy.
- [RFC 9001 – Using TLS to Secure QUIC](https://datatracker.ietf.org/doc/html/rfc9001) – avgränsning mellan TLS-handshake och QUIC-paketskydd.
- [RFC 5280 – Internet X.509 Public Key Infrastructure Certificate and CRL Profile](https://datatracker.ietf.org/doc/html/rfc5280) – certifikatfält, utökningar och sökvägsvalidering.
- [RFC 7468 – Textual Encodings of PKIX, PKCS, and CMS Structures](https://datatracker.ietf.org/doc/html/rfc7468) – PEM-textomslag.
- [RFC 7292 – PKCS #12](https://datatracker.ietf.org/doc/html/rfc7292) – container för nycklar och certifikat.
- [RFC 6960 – Online Certificate Status Protocol](https://datatracker.ietf.org/doc/html/rfc6960) – OCSP-statuskontroll.
- [RFC 9525 – Service Identity in TLS](https://datatracker.ietf.org/doc/html/rfc9525) – referensidentiteter, subjectAltName, wildcards och namnkontroll.
- [RFC 6066, avsnitt 3](https://datatracker.ietf.org/doc/html/rfc6066)
- [RFC 7301 – TLS Application-Layer Protocol Negotiation](https://datatracker.ietf.org/doc/html/rfc7301) – ALPN-förhandling.
- [IANA – Service Name and Transport Protocol Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) – registrerade portar och tjänstenamn.
- [RFC 4511, avsnitt 4.14](https://datatracker.ietf.org/doc/html/rfc4511)
- [RFC 7672 – SMTP Security via DANE TLS](https://datatracker.ietf.org/doc/html/rfc7672) – DNSSEC-/TLSA-baserad SMTP-autentisering.
- [RFC 8461 – SMTP MTA Strict Transport Security](https://datatracker.ietf.org/doc/html/rfc8461) – MTA-STS-policy och MX-kontroll.
- [RFC 8460 – SMTP TLS Reporting](https://datatracker.ietf.org/doc/html/rfc8460) – aggregerade TLS-felrapporter.
- [NIST SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final) – nyckelhantering och skyddsbehov.
- [RFC 8555 – Automatic Certificate Management Environment](https://datatracker.ietf.org/doc/html/rfc8555) – automatiserad utfärdning och förnyelse.
- [OpenSSL](https://docs.openssl.org/master/)
- [Källkod](https://github.com/openssl/openssl)
- [Microsoft Learn – Schannel SSP](https://learn.microsoft.com/windows-server/security/tls/tls-ssl-schannel-ssp-overview) – Windows TLS-implementering och SSPI-klassificering.
- [Oracle – JSSE Reference Guide](https://docs.oracle.com/en/java/javase/24/security/java-secure-socket-extension-jsse-reference-guide.html) – Java TLS-API:er, KeyManager och TrustManager.
- [GnuTLS Documentation](https://www.gnutls.org/documentation.html) – alternativt native TLS-bibliotek och verktygsfamilj.
- [BCP 195 – Recommendations for Secure Use of TLS and DTLS](https://www.rfc-editor.org/info/bcp195) – fortlöpande IETF-säkerhetsriktlinje för TLS-konfiguration.
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) – TCP-nåbarhet i Windows.
- [OpenBSD Manual – nc](https://man.openbsd.org/nc) – TCP-anslutningstest i Unix.
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/) – generell TLS- och STARTTLS-diagnosklient.
- [Microsoft Learn – certutil](https://learn.microsoft.com/windows-server/administration/windows-commands/certutil) – Windows certifikatslager och Chain Engine.
- [OpenSSL – x509](https://docs.openssl.org/master/man1/openssl-x509/) – certifikatfält och utmatningsalternativ.
- [OpenSSL – verify](https://docs.openssl.org/master/man1/openssl-verify/) – explicit sökvägsbyggande och sökvägsvalidering.
- [Oracle – keytool](https://docs.oracle.com/en/java/javase/25/docs/specs/man/keytool.html) – hantering av Java-Key- och Truststores.
- [Wireshark User’s Guide](https://www.wireshark.org/docs/wsug_html/index.html) – analys av handshake, records och sessionsnycklar.
- [RFC 6176 – Prohibiting SSL Version 2.0](https://datatracker.ietf.org/doc/html/rfc6176) – formellt avskaffande av SSL 2.0.
