---
title: "HIN: betrodd miljö, e-postgateway och Stargate"
blatt: "hin"
description: "HIN för meddelande- och plattformsadministratörer: förtroende- och identitetsmodell, HIN Mail, klassisk e-post och Access Gateway, HIN Client, PKI, SMTP- och portalflöden, mailstore, Stargate-arkitektur, migrering, övervakning, återställning och diagnostik."
fakten:
  - label: Plattform
    wert: Betrodd miljö för schweizisk hälso- och sjukvård
    href: https://www.hin.ch/de/services/hin-mail/hin-mail.cfm
  - label: Operatör
    wert: Health Info Net AG · grundat 1996
    href: https://www.hin.ch/de/ueber-hin/unternehmen/geschichte.cfm
  - label: E-postmodell
    wert: HIN till HIN automatiskt · externt vid konfidentialitetsmarkering
    href: https://support.hin.ch/de/service/hin-mail-und-mobile.cfm
  - label: Klassisk edge
    wert: Mail och Access som virtuella apparater
    href: https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm
  - label: Målarkitektur
    wert: Postfix → MXEngine → Postfix
    href: https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf
  - label: Kärnstack
    wert: OPA/Rego · PostgreSQL · Vault · MinIO
    href: https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf
  - label: Mesh-transport
    wert: WireGuard · IDAgent · Port 19818
    href: https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf
  - label: Förtroendeankare
    wert: HIN Identitet · S/MIME · nycklar och CSR
    href: https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf
  - label: Webbåtkomst
    wert: HIN Client · Access Gateway · SAML · OAuth 2.0
    href: https://download.hin.ch/oauth2/doku/de/
  - label: Mailstore
    wert: separat från gatewaytransport · IMAP/POP/Webmail
    href: https://support.hin.ch/de/service/hin-gateway.cfm
  - label: Driftsättning
    wert: Linux · Docker Compose · VM-avbildningar
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
translatedAt: 2026-10-06T11:22:06.199Z
translationReview: automatic
---

# HIN: betrodd miljö, e-postgateway och Stargate

HIN är inte ett enskilt krypteringsprogram, utan en branschrelaterad betrodd miljö med verifierade identiteter, centrala plattformstjänster och åtkomst- respektive gatewaykomponenter. **HIN Mail** skyddar e-postkommunikation, **HIN Access** förmedlar åtkomst till skyddade webbapplikationer och en **HIN Identitet** kopplar den behöriga personen eller organisationen till kryptografiskt nyckelmaterial. Hos institutioner ligger Mail- och Access-komponenterna vid gränsen mellan den egna infrastrukturen och HIN-plattformen ([HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm), [HIN kollektivmedlemskap med gateway](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm)).

För meddelandeadministratörer måste fyra tillståndsområden skiljas åt. Den lokala e-postservern har postlådor, anslutningar och köer. Gatewayen hanterar beslut om transport, förtroende och skydd. HIN-plattformen tillhandahåller katalog-, identitets-, nyckel-, e-post- och åtkomsttjänster. För mottagare utanför HIN Community tillkommer ett webb- och autentiseringsflöde. En störning i ett område är inte automatiskt en störning i alla andra; en nåbar SMTP-port bevisar exempelvis varken en giltig HIN Identitet eller en lyckad portalleverans ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm), [HIN Mail till icke-medlemmar](https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm)).

Även produktnamnet behöver placeras i sitt tidssammanhang. Den klassiska gatewaygenerationen omfattar en Mail Gateway, MGW, och beroende på avtal en Access Gateway, AGW. Som efterföljare beskriver HIN den e-postkapabla mesh-nod som utvecklats i projektet **Stargate**. Den förblir SMTP-kompatibel gentemot lokala e-postsystem, men förändrar arkitektur, nyckeldistribution, driftsättning och transport mellan organisationer. Uttalanden om målplattformen får därför inte utan granskning överföras till en befintlig MGW och vice versa ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm)).

Förklaringen följer ett meddelande från avsändaren via HIN-identitet och gateway till mottagaren. Därefter behandlas plattformsbyten, beroenden, diagnostik, nyckelmaterial och återställning.

## Arkitekturansats: betrodd miljö med edge-komponenter

Den klassiska kollektivmodellen tillhandahåller HIN Mail och HIN Access som virtuella apparater. HIN dokumenterar S/MIME-kryptering och signatur på e-postdomännivå samt en granskningsspårning; Access Gateway fungerar som lokal identitetsleverantör, kopplar verifierade förfrågningar till HIN-eID:n och kan integrera befintliga katalog- och autentiseringstjänster ([HIN kollektivmedlemskap med gateway](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm)).

Denna konstruktion är en **edge-trust-modell**: Organisationen kontrollerar e-postserver, DNS, brandvägg, intern leverans och den lokala gatewayresursen; HIN driver de överordnade förtroende- och plattformstjänsterna. Ett positivt SMTP-avslut överför ansvaret för ett konkret meddelande till nästa hopp; det säger inget om huruvida det efterföljande HIN-, portal- eller mottagarflödet redan har slutförts. För den nya gatewaystacken tilldelar HIN uttryckligen kunden ansvar för DNS och e-postreputation ([HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf), [RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321)).

Stargate flyttar denna gräns till en organisationsrelaterad mesh-nod. HIN beskriver en molnbaserad mikrotjänstarkitektur med REST-API:er, decentraliserade identiteter, distribuerad nyckelhantering och programmerbara policyer. En egen instans är avsedd för varje organisation; HIN nämner virtuella avbildningar och containrar som driftsättningsformer samt även OpenShift- och Kubernetes-miljöer i produktbeskrivningen. Detta är en publicerad målarkitektur, inte ett bevis på att varje befintlig HIN-organisation redan drivs på detta sätt ([HIN Gateway: arkitektur och driftsättning](https://support.hin.ch/de/service/hin-gateway.cfm), [HIN Gateway produktbeskrivning](https://www.hin.ch/de/services/hin-mail/hin-gateway.cfm)).

## Teknikstack och ansvar

Den offentligt dokumenterade stacken består av flera generationer och får inte blandas till en enda monolit:

| Nivå | Implementering av den nya gatewayen | Tillstånds- och felområde | Administratörsbevis |
|---|---|---|---|
| SMTP-kant | **Postfix Relay** för mottagning, återförsök, DNS-routning och leverans | Transport är separerad från innehållsbeslut | SMTP-slutsvar, kö-ID, nästa hopp, MX och PTR |
| Bearbetning | **MXEngine** för HTTP-/SMTP-inmatning, transformering och leveransstrategi | Bearbetningsfel kan skiljas från Postfix-återförsök | Message-ID, MXEngine-händelse, transformering, återlämning till Postfix |
| Policy | **Open Policy Agent med Rego**, valfritt synkroniserad via Git | Regeltillstånd är data- och versionstillstånd | Policyrevision, indata, resultat, godkännande och återställning |
| Identitet och krypto | **S/MIME Keys Client**, **IDAgent**, Issuer och Verifier | CSR, certifikat, privat nyckel och peeridentitet är separata objekt | Fingerprint, innehavare, utgång, Issuer, peer och rotation |
| Persistens | **PostgreSQL** per tjänst, **Vault** för hemligheter, **MinIO** för meddelanden och bilagor | Databas, secret store och objektlagring har egna återställningsgränser | Volym, backuptid, återställningstest och konsistenskontroll per tjänst |
| Mesh-transport | **WireGuard** via IDAgent | Kanaltillstånd är inte samma sak som SMTP-leverans | Peer-nyckel, endpoint, handshake, port 19818 och efterföljande SMTP-händelse |
| Observability | **Promtail → Loki**, **Node Exporter**, Prometheus-kompatibla mätvärden och Version Collector | Loggtransport, värdmätvärden och tjänstehälsa kan fallera separat | Liveness, scrape-ålder, logginkomst, värdresurser och tidsbas |
| Driftsättning | Linux, **Docker Compose**, persistenta Docker-volymer; VM-avbildningar som installationsväg | Värd, containrar, avbildningar och volymer har olika livscykler | godkänd avbildning, Compose-konfiguration, volyminventering och omstartstest |

HIN dokumenterar uttryckligen meddelandeflödet som `External SMTP → Postfix → MXEngine → Postfix → External SMTP`. Den tvingade överlämningen till MXEngine förhindrar policy-bypass; Postfix fortsätter att ansvara för leverans och återförsök. OPA/Rego håller verksamhetsregler utanför applikationskoden. PostgreSQL, Vault och MinIO lagrar olika tillståndsklasser och får inte behandlas som ett enda filsystem vid backup ([HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf), [HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf)).

Den tekniska installationsdokumentationen nämner Linux-distributioner från RHEL-kompatibla familjer samt Ubuntu och Debian, Docker Compose-drift på en enskild värd och VM-avbildningar för flera plattformar. Detta supportuttalande är versions- och utrullningsrelaterat; avgörande är den HIN-dokumentation som är godkänd vid installationstillfället, inte en statiskt övertagen distributionslista ([HIN Stargate Deployment](https://health-info-net-ag.github.io/Stargate-deployment/de/)).

Mailstore förblir en annan gräns: Enligt HIN fungerar Stargate som Mail Transport Agent, medan IMAP fortsatt tillhandahålls på den befintliga Zimbra-plattformen. HIN Access utgör dessutom ett självständigt autentiserings- och auktoriseringsflöde via Client, AGW respektive Access Control Service ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm), [HIN Client 3-handbok](https://download.hin.ch/documentation/HIN_Client3_Handbuch_de_v1.1.pdf)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-hin.svg?v=20260813" title="Interaktive Infografik: HIN Vertrauensraum mit lokalem Mailserver, klassischem Mail und Access Gateway, HIN Identität, Mailplattform, Nichtmitglieder-Portal sowie Stargate-Zielarchitektur und Betriebssignalen" loading="lazy">
  <a href="/images/kb-interaktiv-hin.svg?v=20260813">Öppna den interaktiva grafiken direkt</a>.
</iframe>

## Identitet och PKI med auktorisering

En HIN Identitet är mer än en e-postadress. Vid aktivering skapar HIN Client ett nyckelpar; lösenordet låser upp det lokala nyckelmaterialet. I den nya gatewayen genererar S/MIME Keys Client kryptografiska nycklar och Certificate Signing Requests, medan Vault hanterar privata nycklar, autentiseringsuppgifter och känslig konfiguration. Ett certifikat binder en offentlig nyckel till en namngiven identitet, men ersätter inte applikationsspecifik auktorisering ([HIN Client 3-handbok](https://download.hin.ch/documentation/HIN_Client3_Handbuch_de_v1.1.pdf), [HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf), [RFC 5280](https://datatracker.ietf.org/doc/html/rfc5280)).

Livscykeln hör hemma i IAM-runbooken: registrering, aktivering, enhetsbyte, roll- eller namnändring, spärr, omregistrering vid misstanke om nyckelkompromettering och utträde. HIN kräver en identitetskontroll efter beställningen och skiljer mellan personliga eID:n och organisations- respektive Device-ID i kollektivmodellen. En fungerande e-posttransport får inte ses som bevis på att en tidigare eller felaktigt tilldelad identitet inte längre har åtkomst ([HIN Identitet](https://support.hin.ch/de/service/hin-identitaet.cfm), [HIN kollektivmedlemskap med gateway](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm)).

HIN Access och HIN Mail delar den betrodda miljön, men inte samma protokollflöde. Access Control Service kan begära en autentiseringsnivå via SAML `RequestedAuthnContext`; HIN dokumenterar profiler för lösenord och MFA. För integrationer tillhandahåller HIN dessutom OAuth 2.0-flöden för Authorization Code och Client Credentials. Autentisering, tokenutfärdande och auktorisering av målapplikationen ska loggas separat ([HIN Authentication Context](https://support.hin.ch/de/thema/hin-access/pwd.cfm), [HIN OAuth2-integration](https://download.hin.ch/oauth2/doku/de/), [SAML 2.0 Core](https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf), [RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749)).

Först när identitet och PKI har klarlagts går det att bedöma e-postflödet. Gateway och policy avgör beroende på avsändare, mottagare och mål vilket skyddsförfarande som används.

## E-postflöden och skyddsbeslut

Med de berörda HIN-komponenterna i åtanke kan nu den konkreta meddelandevägen förklaras. Följande fall skiljer sig åt efter var avsändare och mottagare befinner sig och vilken tjänst som fattar skyddsbeslutet.

### Mellan HIN-deltagare

HIN beskriver meddelanden mellan HIN-adresser som automatiskt överförda i enlighet med dataskyddet. Plattformen markerar integritetsstatusen i ämnesraden med `[HIN secured]` respektive `[Not secured by HIN]`. Denna markering är en användarsignal, men inte tillräcklig teknisk korrelation: För en incident krävs dessutom kuvertavsändare och -mottagare, Internet `Message-ID`, `Received`-kedja, gatewayhändelse och tidsfönster ([HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm), [HIN Mail och Mobile](https://support.hin.ch/de/service/hin-mail-und-mobile.cfm), [RFC 5322](https://datatracker.ietf.org/doc/html/rfc5322)).

Ett klassiskt organisationsflöde kan beskrivas som `Mailserver → SMTP → MGW → HIN → MGW → SMTP → Mailserver`. Varje station avslutar en session och kan ha en egen kö. HIN beskriver kollektivmodellen som S/MIME-kryptering och signatur på e-postdomännivå; den lokala leveransen före och efter denna gatewaygräns förblir ett eget skydds- och driftområde ([HIN kollektivmedlemskap med gateway](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm), [RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321)).

### Till personer utan HIN-medlemskap

För mottagare utan HIN-adress måste avsändaren enligt HIN-dokumentationen uttryckligen markera meddelandet som konfidentiellt, exempelvis med `(Vertraulich)` i ämnesraden. Mottagaren öppnar det skyddade innehållet via ett webbflöde och autentiserar sig med mobilnummer och SMS-kod; ett säkert svar är möjligt. Detta skapar ytterligare tillstånd: aviseringse-post, portalobjekt, mottagarregistrering, andra faktor, lagringstid och svars-kanal ([HIN Mail till icke-medlemmar](https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm), [HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm)).

En levererad avisering är ännu inte ett läst meddelande. Syntetiska tester bör därför täcka hela flödet fram till inloggning, öppning, bilaga och svar. Vidarebefordringar från portalen kan lämna skyddsflödet; HIN påpekar uttryckligen att vissa typer av vidarebefordran skickas okrypterat. Sådana användaråtgärder hör hemma i utbildning, DLP-modell och incidentanalys ([HIN Mail till icke-medlemmar](https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm)).

### Enheter, applikationer och massutskick

Den klassiska Mail Gateway kan fungera som SMTP-gränssnitt för interna enheter; HIN dokumenterar även denna funktion för Stargate. HIN Mail-tjänstesidan anger att systemutskick via gatewayen kräver separat licensiering. Skannrar, KIS, laboratorieapplikationer och batchprocesser behöver därför ett eget, dokumenterat flöde för avsändare, relay, volym och fel i stället för tyst samutnyttjande av användarnas e-postflöde ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm), [HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm)).

## SMTP-routning och mottagningsgränser

Gatewayen måste för varje riktning exakt veta vilka domäner som är lokala, auktoritativa, ska vidarebefordras eller avvisas. Otydligt ansvar mellan Exchange, molnanslutning, Secure Mail Gateway och HIN-edge leder antingen till bypasser eller slingor. [SMTP](/kb/smtp) föreskriver trace-rader och beskriver slingdetektering; den faktiska ansvarövergången sker först med ett positivt svar efter det fullständiga meddelandeinnehållet ([RFC 5321, Trace Information](https://datatracker.ietf.org/doc/html/rfc5321#section-4.4), [RFC 5321, DATA](https://datatracker.ietf.org/doc/html/rfc5321#section-4.1.1.4)).

För varje rutt ska minst dessa värden ingå i driftens dokumentation:

- lokal listener, port och förväntade källnät respektive peeridentiteter;
- EHLO-namn, envelope-domäner och auktoriserade relayområden;
- ordningsföljd för spam-/malwarefilter, HIN-gateway och internt e-postsystem;
- nästa hopp, DNS- eller smarthostupplösning och [TLS](/kb/tls)-krav;
- beteende när HIN-plattformen, identitets- eller nyckelkomponenten inte är nåbar;
- köålder, återförsöksplan, maximalt antal hopp och ansvar för studsmeddelanden;
- undantagsflöden för enheter, systemutskick och migrationssamexistens.

För Exchange Online är detta en fråga om anslutningar och inte bara DNS. Microsoft dokumenterar routning till tredjepartsgatewayer samt certifikat- eller IP-baserade anslutningsvillkor. HIN-FAQ bekräftar det grundläggande stödet för hybrida och Microsoft 365-nära arkitekturer, men hänvisar för den konkreta konfigurationen till kundspecifik migrationsdokumentation ([Microsoft: Mail flow using connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow), [Microsoft: Third-party cloud mail flow](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud), [HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm)).

## Mailstore och klientåtkomst med token

Postlådeåtkomst och gatewaytransport är olika driftmodeller. HIN publicerar IMAP på port 993 med TLS och Message Submission på port 587 med STARTTLS för personliga HIN Mail-konton; ett skapat mail-token används som lösenord. POP på port 995 är också dokumenterat, men laddar vanligtvis ned meddelanden lokalt och kan ta bort dem på servern. Protokollrollerna motsvarar [IMAP](/kb/apache-james#protokolle-tls-und-ports), POP3 och Message Submission, inte gateway-till-gateway-transport ([HIN Mail Token Service](https://support.hin.ch/de/service/hin-mail-und-mobile/token-service-mail-client.cfm), [HIN POP-konfiguration](https://support.hin.ch/de/service/hin-mail-und-mobile/mail-clients-einrichten-pop.cfm), [RFC 9051](https://datatracker.ietf.org/doc/html/rfc9051), [RFC 1939](https://datatracker.ietf.org/doc/html/rfc1939), [RFC 6409](https://datatracker.ietf.org/doc/html/rfc6409)).

HIN Client-handboken dokumenterar ersättningen av den historiska lokala e-postproxyn med tokenbaserad åtkomst. För terminalservrar påpekar HIN uttryckligen att sändning och mottagning sker med mail-token när proxyn är inaktiverad. Detta är relevant för den tekniska historien: Klienten var ursprungligen kommunikationsförmedlare för HTTP, SMTP, POP och IMAP; senare driftmodeller frikopplar e-postklientåtkomst och HIN-identitetsklient i högre grad ([HIN Client 3-handbok](https://download.hin.ch/documentation/HIN_Client3_Handbuch_de_v1.1.pdf), [HIN Client på terminalservrar](https://support.hin.ch/de/thema/hin-client/hin-client-auf-terminalserver.cfm)).

HIN beskriver den befintliga Mail Storage Agent i Stargate-FAQ:n som Zimbra-baserad och initialt opåverkad av den nya gatewaytransporten. Ett lyckat Stargate-e-postflöde bevisar därför varken IMAP-tillgänglighet eller postlådekonsistens. Omvänt kan webmail fungera medan organisationsanslutningen eller mesh-transporten är störd ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm)).

Det klassiska mailstore-flödet är inte den enda driftformen. Stargate flyttar funktioner och ansvarsområden och måste därför förstås som en egen meddelande- och administrationsväg.

## Stargate som målarkitektur

Stargate ska behålla e-postkompatibiliteten vid kanten och samtidigt möjliggöra mer allmänt decentraliserat datautbyte. HIN nämner Self-Sovereign Identity, Data Mesh, mikrotjänster, RESTful API:er, Open Source-komponenter och en e-postkapabel mesh-nod. För kanalen mellan Stargate-instanser aviseras en [WireGuard](https://www.wireguard.com/protocol/)-baserad applikation-till-applikation-transport; HIN skiljer den uttryckligen från en allmän VPN-tunnel ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm)).

För administratörer följer av detta en tredelad trace:

1. Lokalt förblir SMTP med mottagningssvar, kö och anslutning den verifierbara gränsen.
2. Mellan mesh-noderna tillkommer tillstånd för identitet, discovery, nycklar och WireGuard-kanal.
3. Vid målet uppstår återigen ett lokalt SMTP-flöde till det mottagande e-postsystemet.

De publicerade systemöversikterna dokumenterar Postfix, MXEngine, OPA/Rego, PostgreSQL, Vault, MinIO, IDAgent, Promtail/Loki och Prometheus-kompatibla mätvärden. Något programmeringsspråk eller fullständigt källträd för produkttjänsterna anges inte där; därför gissas inte denna information utifrån containeravbildningar eller externa projekt med samma namn ([HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf), [HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf)).

Nätverksgränserna för den nya gatewayen är ovanligt konkret offentligt dokumenterade:

| Port och riktning | Roll | Driftsbetydelse |
|---|---|---|
| TCP 25 inkommande och utgående | SMTP-mottagning och MX-baserad leverans | Internetexponering, reputation, kö och next-hop-routning |
| TCP 8084 inkommande | HTTP-callback från den fjärranslutna Sealern | enligt HIN avsiktligt utan ytterligare TLS-lager, eftersom nyttolasten själv är krypterad |
| TCP och UDP 19818 i båda riktningarna | WireGuard mellan IDAgents | kontrollera peer-nyckel, endpoint, NAT och brandvägg tillsammans |
| TCP 443 och 4433 utgående | Registry, S/MIME-CA, Sealer, Issuer, loggning och Verifier | plattformsberoende trots lokal gatewaydrift |
| TCP och UDP 53 utgående | MX-, SPF-, A/AAAA- och PTR-upplösning | Routning och säkerhetsbeslut är beroende av [DNS](/kb/dns) |

De lokalt exponerade diagnostik- och tjänsteportarna, inklusive PostgreSQL, Vault, MinIO och mätvärdesändpunkter, ska enligt HIN inte vara nåbara från det offentliga internet. Administrationsåtkomst till 22, 443, 8180 eller 8190 hör hemma i ett definierat hanteringsnät och inte i en generell internetöppning ([HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf), [HIN Stargate Deployment](https://health-info-net-ag.github.io/Stargate-deployment/de/)).

HIN beskriver migreringen som en parallell uppbyggnad av den nya gatewayen. Först efter konfiguration, tester och bekräftad driftberedskap beslutar kunden om växlingen. Befintliga IP-adresser kan i princip återanvändas, men konfigurationen måste översättas. En säker cutover-runbook innehåller därför minst inventering, export, parallellrutt, testmatris, växlingskriterium, returväg, köhantering och tydlig rollback ([HIN Gateway: migrering](https://support.hin.ch/de/service/hin-gateway.cfm)).

## Hög tillgänglighet med backup och recovery

HIN dokumenterar ett redundant OpenShift-kluster för sin egen Stargate-sida, men planerar inte automatiskt två redundanta virtuella maskiner på kundsidan. Dessa uttalanden får inte slås ihop till generell end-to-end-högtillgänglighet. En redundant plattformstjänst skyddar inte mot en enskild lokal hypervisor, en felaktig connectorregel, en utgången nyckel eller en blockerad brandvägg ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm)).

I den nya gatewayen persisterar tjänsterna via Docker-volymer. Vault förseglas automatiskt efter en containeromstart och kräver en kontrollerad unseal-process. Den tekniska översikten anger som standard dagliga backuper av databaser, Vault-nycklar och -hemligheter, konfigurationsfiler samt certifikat; ansvar och lagringstid ska avtalas med kunden ([HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf)).

Ett recoveryinventarium bör per generation minst innehålla:

| Objekt | Konsekvens vid förlust | Verifierbart återställningssteg |
|---|---|---|
| Värd, Compose- och containerdefinition | Edge-noden startar inte | tillhandahåll nytt mål från godkänd avbildning och skapa deterministisk runtime |
| Kundkonfiguration och OPA/Rego-policyer | fel rutt, policy eller domän | återställ versionshanterat tillstånd, läs in policy och kör testmatris |
| PostgreSQL-databaser | policy-, metadata- eller agenttillstånd saknas | genomför databasåterställning per tjänst och referentiell kontroll |
| Vault-nycklar, hemligheter och certifikat | dekryptering, peer- eller organisationsidentitet saknas | genomför godkänd restore, unseal och kryptografiskt funktionstest |
| MinIO-meddelanden och bilagor | meddelande eller arkivobjekt saknas | klargör omfattning och lagringstid separat med HIN och testa objektåterställning |
| Anslutningar och DNS | bypass, slinga eller olevererbarhet | kontrollera rutt i båda riktningar med entydigt Message-ID |
| Kö respektive överlämningsbevis | dubbletter eller meddelandeförlust | klargör öppet ansvar per meddelande före växling |
| Postlåda och mail-token | klientåtkomst störd | validera separat via webmail och IMAP/Submission |
| Audit- och driftloggar | incident går inte att rekonstruera | testa tidsbas, export, lagringstid och SIEM-ingång |

Den offentliga backuplistan nämner databas, Vault, konfiguration och certifikat, men inte uttryckligen MinIO-meddelanden och -bilagor. Därav får varken deras säkerhetskopiering eller deras avsiktliga undantag slutsatsas; just denna punkt ska före produktionssättning skriftligt ingå i överenskommelsen om retention, backup och restore. En generell VM-snapshot bevisar dessutom varken ett konsistent PostgreSQL-tillstånd eller ett återställningsbart Vault ([HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf), [HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf)).

Vid en störning följs meddelandet från mottagning via identitets- och policybeslut till vald leveransväg; först därefter startas eller kringgås enskilda komponenter.

## Övervakning och incidenttriage

En användbar driftbild kombinerar lokala och centrala signaler:

- tillgänglighet och köålder per SMTP-next-hop;
- kvoter för mottagning, vidarebefordran och studs med korrelerbart Message-ID;
- HIN-, portal-, identitets-, nyckel- och policyfel separat;
- certifikatutgång, registrerings- och tokenstatus;
- postlådeåtkomst via webmail och IMAP oberoende av gatewayen;
- DNS- och brandväggsupplösning via namn i stället för hårdkodade HIN-IP-adresser;
- plattformsmeddelanden på [HIN Status](https://status.hin.ch/) plus lokal telemetri.

Den nya stacken levererar Prometheus-kompatibla tjänstemätvärden, värdmätvärden via Node Exporter, central loggöverföring via Promtail till Loki och en Version Collector som frågar Liveness-endpoints. För larmhantering ska minst utebliven scrape, utebliven logginkomst, Postfix-köålder, MXEngine-fel, Vault-seal-status, PostgreSQL- och MinIO-kapacitet, WireGuard-peerstatus samt certifikatutgång hanteras separat ([HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf), [HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf)).

HIN:s brandväggsdokumentation rekommenderar DNS-namn eftersom IP-adresser kan ändras, och nämner bland annat HTTPS- och SMTP-flöden för klienttjänster. En global statuswebbplats kan inte upptäcka en lokal DNS-, NAT-, MTU-, connector- eller nyckelstörning. Triage börjar därför med omfattningen: en användare, en identitet, en domän, en riktning, en gateway eller plattformen ([HIN brandväggsanpassningar](https://support.hin.ch/de/thema/hin-client/anpassung-firewall-hin-client.cfm), [HIN Status](https://status.hin.ch/)).

Det klassiska kollektivutbudet omfattar ett audit trail för e-postflödet; den nya gatewayen kompletterar detta med strukturerade centrala loggar. För en sammanhängande e-postanalys måste dessa belägg korreleras med lokala SMTP- och e-postsystemloggar. Tidssynkronisering och enhetliga tidszoner är driftkrav, inte kosmetiska inställningar ([HIN kollektivmedlemskap med gateway](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm), [HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf)).

## Diagnostikverktyg

Diagnostiken börjar vid det publika respektive interna namnet och följer därefter det faktiska e-postflödet. Först när DNS, anslutning och certifikat stämmer utvärderas gatewaytillstånd, kö och HIN-specifika händelser.

### DNS och HIN-endpoints

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

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) och [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) visar om de dokumenterade namnen kan lösas upp ur det resolverperspektiv som faktiskt används. Detta är särskilt viktigt vid split-DNS och proxier, viktigare än ett kopierat IP-värde; HIN rekommenderar uttryckligen DNS-namn i stället för långsiktigt fastställda adresser ([HIN brandväggsanpassningar](https://support.hin.ch/de/thema/hin-client/anpassung-firewall-hin-client.cfm)).

### TCP- och TLS-nåbarhet

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

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) och [`nc`](https://man.openbsd.org/nc) bevisar TCP-flödet. UDP-anropet från `nc` kan högst indikera nåbarhet; eftersom WireGuard förkastar obehöriga paket utan svar är först den autentiserade peer-handshaken ett tillförlitligt bevis för port 19818. Windows-[`curl.exe`](https://curl.se/docs/manpage.html) och [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) kontrollerar SMTP-/STARTTLS-kanten. En lyckad handshake bevisar ännu ingen policybearbetning eller meddelandeleverans ([HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf), [RFC 8446](https://datatracker.ietf.org/doc/html/rfc8446)).

### Kontrollerad SMTP-transaktion

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

[`curl`](https://curl.se/docs/manpage.html) och [`swaks`](https://jetmore.org/john/code/swaks/) får endast användas mot en uttryckligen auktoriserad listener och testmottagare. Följande registreras: slutsvar efter `DATA`, lokalt kö-ID, gatewayhändelse, valt skyddsflöde, nästa hopp och faktisk ankomst. Ett `250` på `RCPT TO` är ännu ingen mottagning av meddelandeinnehållet ([RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321)).

### Lokala socket-tillstånd

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

[`Get-NetTCPConnection`](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection) och [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) visar lokala listeners och etablerade TCP-sessioner. De är endast meningsfulla där administratören har åtkomst till den berörda värden; en hanterad appliance- eller containerprodukt får inte ändras genom odokumenterade shellåtkomster.

### Paketflöde vid rätt mätpunkt

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

[`pktmon`](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon) och [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) ser endast trafiken vid den valda mätpunkten. För Stargate kan en lokal SMTP-trace inte helt förklara mesh-kanalen; ytterligare gateway- och plattformshändelser behövs. Inspelningar kan innehålla adresser, ämnesrader eller okrypterade protokolldelar och måste behandlas som känsliga driftdata.

## Teknisk historia

FMH och Ärztekasse grundade Health Info Net AG 1996, när e-post inom hälso- och sjukvården växte fram och överföring av känsliga data via vanlig internetpost ansågs vara otillräckligt skyddad. HIN började därmed som leverantör av skyddad e-postkommunikation för läkarkåren och utvecklades till en bredare förtroende- och åtkomstmiljö ([HIN företagshistoria](https://www.hin.ch/de/ueber-hin/unternehmen/geschichte.cfm)).

Klientarkitekturen visar den tekniska förändringen. HIN Client 1 och 2 ersattes av HIN Client 3. Av kompatibilitetsskäl fortsatte klienten att fungera som lokal proxy för webbläsare och e-postprogram; för webbåtkomst dokumenterar HIN senare Challenge/Response, för e-postkonton övergången till oberoende tokens via standardportar. Historiken förklarar varför äldre installationsanvisningar nämner lokala proxyportar, medan nyare underlag använder direkta IMAP-, POP- och Submission-endpoints ([HIN Client 3-handbok](https://download.hin.ch/documentation/HIN_Client3_Handbuch_de_v1.1.pdf), [HIN Mail Token Service](https://support.hin.ch/de/service/hin-mail-und-mobile/token-service-mail-client.cfm)).

Den klassiska HIN-kollektivmodellen samlade Mail- och Access-appliances i kundnätet. Den nuvarande tjänstesidan dokumenterar för denna generation S/MIME på e-postdomännivå, audit trail, en lokal identitetsleverantör samt anslutning av befintliga autentiserings- och katalogtjänster. Dessa funktioner förklarar den framväxta separationen mellan e-posttransport och webbåtkomst ([HIN kollektivmedlemskap med gateway](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm)).

Från 2025 införde HIN en ny leverans till icke-medlemmar; samtidigt förnyades plattforms- och Access-infrastrukturen. Gateway-dokumentationen som publicerades 2026 beskriver Stargate som nästa generationsskifte: från enbart en e-postkrypteringsgateway till en decentraliserad, molnbaserad nod för e-post och strukturerat utbyte av hälsodata. Den tekniska dokumentationen konkretiserar detta skifte med Postfix, MXEngine, OPA/Rego, PostgreSQL, Vault, MinIO och containeriserad drift. För en migrationsplan handlar det därför inte bara om att ersätta en VM, utan om att på nytt mäta identitet, nycklar, transport, observability och recovery ([HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm), [HIN Access](https://support.hin.ch/de/thema/hin-access.cfm), [HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm), [HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf)).

## Källor

- [HIN – HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm)
- [HIN – Kollektivmedlemskap med gateway](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm)
- [HIN Support – HIN Gateway och Stargate](https://support.hin.ch/de/service/hin-gateway.cfm)
- [HIN Support – HIN Mail till icke-medlemmar](https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm)
- [HIN – Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf)
- [RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321)
- [HIN – HIN Gateway produktbeskrivning](https://www.hin.ch/de/services/hin-mail/hin-gateway.cfm)
- [HIN – Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf)
- [HIN – Stargate Deployment](https://health-info-net-ag.github.io/Stargate-deployment/de/)
- [HIN – HIN Client 3-handbok](https://download.hin.ch/documentation/HIN_Client3_Handbuch_de_v1.1.pdf)
- [RFC 5280 – Internet X.509 PKI](https://datatracker.ietf.org/doc/html/rfc5280)
- [HIN Support – HIN Identitet](https://support.hin.ch/de/service/hin-identitaet.cfm)
- [HIN Support – SAML Authentication Context](https://support.hin.ch/de/thema/hin-access/pwd.cfm)
- [HIN – OAuth2-integration](https://download.hin.ch/oauth2/doku/de/)
- [OASIS – SAML 2.0 Core](https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf)
- [RFC 6749 – OAuth 2.0](https://datatracker.ietf.org/doc/html/rfc6749)
- [HIN Support – HIN Mail och Mobile](https://support.hin.ch/de/service/hin-mail-und-mobile.cfm)
- [RFC 5322 – Internet Message Format](https://datatracker.ietf.org/doc/html/rfc5322)
- [Microsoft – Mail flow using connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow)
- [Microsoft – Mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud)
- [HIN Support – HIN Mail Token Service](https://support.hin.ch/de/service/hin-mail-und-mobile/token-service-mail-client.cfm)
- [HIN Support – POP-konfiguration](https://support.hin.ch/de/service/hin-mail-und-mobile/mail-clients-einrichten-pop.cfm)
- [RFC 9051 – IMAP4rev2](https://datatracker.ietf.org/doc/html/rfc9051)
- [RFC 1939 – POP3](https://datatracker.ietf.org/doc/html/rfc1939)
- [RFC 6409 – Message Submission](https://datatracker.ietf.org/doc/html/rfc6409)
- [HIN Support – HIN Client på terminalservrar](https://support.hin.ch/de/thema/hin-client/hin-client-auf-terminalserver.cfm)
- [WireGuard – Protocol and Cryptography](https://www.wireguard.com/protocol/)
- [HIN Status](https://status.hin.ch/)
- [HIN Support – Brandväggsanpassningar för HIN Client](https://support.hin.ch/de/thema/hin-client/anpassung-firewall-hin-client.cfm)
- [Microsoft – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [ISC BIND – dig manual](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc manual](https://man.openbsd.org/nc)
- [curl – command-line manual](https://curl.se/docs/manpage.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [RFC 8446 – TLS 1.3](https://datatracker.ietf.org/doc/html/rfc8446)
- [swaks – Swiss Army Knife for SMTP](https://jetmore.org/john/code/swaks/)
- [Microsoft – Get-NetTCPConnection](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection)
- [Linux man-pages – ss](https://man7.org/linux/man-pages/man8/ss.8.html)
- [Microsoft – Packet Monitor](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon)
- [tcpdump – tcpdump(1)](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [HIN – Företagshistoria](https://www.hin.ch/de/ueber-hin/unternehmen/geschichte.cfm)
- [HIN Support – HIN Access](https://support.hin.ch/de/thema/hin-access.cfm)
