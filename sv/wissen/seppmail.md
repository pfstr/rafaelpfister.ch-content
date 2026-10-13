---
title: "SEPPmail: e-postgateway, nycklar och GINA-leverans"
blatt: "seppmail"
description: "SEPPmail för meddelandeadministratörer: SMTP-inline-arkitektur, Postfix- och ruleset-pipeline, S/MIME, OpenPGP och GINA, nyckel- och certifikatstatus, kataloganslutning, kluster, övervakning, backup och återställning."
fakten:
  - label: Produkt
    wert: SEPPmail Secure E-Mail Gateway
    href: https://www.seppmail.com/products/
  - label: Systemroll
    wert: SMTP-gateway i meddelandevägen
    href: https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html
  - label: Transportkärna
    wert: Postfix MTA
    href: https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html
  - label: Systembas
    wert: Appliance-stack baserad på OpenBSD
    href: https://docs.seppmail.com/ch/20_revision-history.html
  - label: Meddelandekryptografi
    wert: S/MIME · OpenPGP · domännycklar
    href: https://docs.seppmail.com/ch/07_mi_07_mp_01_ep_01_addedit-encryption-policy.html
  - label: Utan mottagarnyckel
    wert: GINA-webbleverans via HTTPS
    href: https://docs.seppmail.com/en/07_mi_17_gd__gina-domains.html
  - label: Regelmodell
    wert: genererat ruleset · policyer · Custom Commands
    href: https://docs.seppmail.com/ch/08_cr_01_gi_04_macros.html
  - label: Katalog
    wert: LDAP/LDAPS för frågor om nycklar, identiteter och regler
    href: https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html
  - label: Webbgränssnitt
    wert: GINA 443 · Admin 8443 · REST 8445
    href: https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html
  - label: Övervakning
    wert: Postmaster · SNMP · NRPE · Syslog
    href: https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html
  - label: Klustertransport
    wert: SSH mellan appliances · NTP krävs
    href: https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html
  - label: Ursprung
    wert: Schweizisk produktutveckling sedan 2001
    href: https://www.seppmail.com/de/die-seppmail-story/
werbung:
  - tools
  - newsletter
ctaThemen:
  - seppmail
translationSourceHash: f01b161a8f003ca453398dd3d393ca671b7346ad63164cd311dabb11055526fe
translationModel: gpt-5.6-terra
translatedAt: 2026-10-12T11:50:44.633Z
translationReview: automatic
---

# SEPPmail: e-postgateway, nycklar och GINA-leverans

SEPPmail är en specialiserad e-postgateway som tar emot meddelanden i SMTP-sökvägen, bearbetar dem enligt centrala regler och vidarebefordrar dem till ett internt e-postsystem, en smarthost eller nästa motpart på internet. De dokumenterade bearbetningsmetoderna omfattar S/MIME- och OpenPGP-signering respektive -kryptering, S/MIME-baserad domänkryptering, anslutningsrelaterade TLS-krav och GINA-webbleverans för mottagare utan användbar offentlig nyckel ([SEPPmail: Products](https://www.seppmail.com/products/), [SEPPmail: Mail System](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html), [SEPPmail: Encryption Policy](https://docs.seppmail.com/ch/07_mi_07_mp_01_ep_01_addedit-encryption-policy.html)).

Den centrala bearbetningen förenklar klienten: användare kan skicka från sitt vanliga e-postsystem medan gatewayen söker nycklar, utvärderar policyer och utför kryptografiska operationer. Samtidigt blir appliance-lösningen därmed en förtroende- och tillgänglighetsgräns. Den ser meddelandeinnehållet före utgående kryptering respektive efter inkommande dekryptering, lagrar privat nyckelmaterial och kan efter lyckad SMTP-mottagning bli den nya ansvarspunkten för meddelandet. En administratör måste därför inte bara kunna påvisa att ”kryptering är aktiv”, utan kunna följa hela sökvägen med routning, regel, nyckelval, leveransmetod, kö, loggning och återställning ([RFC 5321: SMTP responsibility](https://datatracker.ietf.org/doc/html/rfc5321#section-6.1), [RFC 8551: S/MIME](https://datatracker.ietf.org/doc/html/rfc8551), [SEPPmail: Frontend/Backend Cluster](https://docs.seppmail.com/en/04_com_09_cl_05_frontend-backend-cluster.html)).

Förklaringen följer ett meddelande genom SMTP-mottagning, ruleset och vald krypteringsmetod fram till leverans eller GINA-portalen. Nycklar, kluster, diagnostik och återställning förklaras därefter längs denna väg.

SEPPmail fungerar som en tillståndsfull gateway i meddelandevägen. Den tar emot SMTP, fastställer policy och mottagarkapacitet, väljer S/MIME, OpenPGP, TLS eller GINA och levererar sedan vidare. Arkitektur, drift och återställning utgår från detta flöde.

## Arkitekturansats: tillståndsfull inline-gateway

I den enklaste uppbyggnaden skickar det interna e-postsystemet utgående meddelanden via [SMTP](/kb/smtp) till SEPPmail; för inkommande meddelanden pekar den publika [MX-posten](/kb/dns#dns-im-mail--und-identitätsbetrieb) direkt eller via en föregående smarthost på gatewayen. För varje hanterad domän definieras en intern vidarebefordringsserver. Utgående kan appliance-lösningen själv leverera via DNS eller använda en fastställd SMTP-smarthost. Tillverkarens dokumenterade brandväggsmatris redovisar dessa riktningar separat och anger TCP 25 mellan e-postserver, appliance, internet eller smarthost ([SEPPmail: Setting Up The Firewall/Router](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html), [SEPPmail: Mail System](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)).

Gatewayen är ingen transparent paketfiltreringsenhet. Den avslutar en SMTP-session och öppnar en ny för vidarebefordran. Efter `250` i slutet av DATA-fasen ligger leveransansvaret hos det mottagande SMTP-systemet; en lyckad anslutningsetablering eller `STARTTLS` ensamt styrker inte denna överlämning ([RFC 5321, avsnitt 4.1.1.4](https://datatracker.ietf.org/doc/html/rfc5321#section-4.1.1.4), [RFC 3207: SMTP STARTTLS](https://datatracker.ietf.org/doc/html/rfc3207)). Ur driftsperspektiv uppstår minst fem separata kontrollobjekt: inkommande SMTP-session, lokal bearbetning, nästa SMTP-hopp, nyckel- och policystatus samt vid behov GINA-webbtillståndet.

För Exchange Online beskriver SEPPmail en uttrycklig modell med connectors och regler för e-postflöde. I miljöer med flera klientorganisationer rekommenderar tillverkarens instruktioner certifikatbaserade connectors; flera gatewayvärdar måste dessutom beaktas i Microsoft-connectors och vid behov i antispamkonfigurationen. Microsoft dokumenterar också connectors som en definierad förtroende- och routningsrelation, inte som en allmän avstängning av alla övriga leveransvägar ([SEPPmail: Setup with Exchange Online](https://docs.seppmail.com/en/04_com_powershell_setup-exo.html), [Microsoft: Configure mail flow using connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow)).

## Teknikstack och beständiga tillstånd

Den offentligt tillgängliga revisionshistoriken dokumenterar en appliance-stack baserad på OpenBSD med Postfix som MTA samt bland annat Apache HTTP Server, OpenSSL respektive LibreSSL, GnuPG, OpenLDAP, Dovecot, Fetchmail, ClamAV, OpenSSH och Perl. Dessa komponenter fyller olika roller; versionsnivåer som nämns i en viss release note är inte tidlösa produktegenskaper. För driften är det avgörande vilket gränssnitt tillverkaren stödjer: SMTP- och TLS-parametrar hanteras exempelvis via ”Mail System”, medan ”Extended Postfix MTA settings” uttryckligen beskrivs som riskfyllda och inte klustersynkroniserade ([SEPPmail: Revision History](https://docs.seppmail.com/ch/20_revision-history.html), [SEPPmail: Mail System – Extended Postfix settings](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)).

Det synliga e-postflödet är bara en del av tillståndet. För en robust inventering måste minst följande objekt inventeras separat:

| Tillståndsområde | Exempel | Varför det granskas separat |
|---|---|---|
| SMTP och routning | Managed Domains, Forwarding Server, Smarthost, relaynät, TLS-mål | avgör mottagning och nästa hopp |
| Bearbetning | genererat ruleset, Encryption Policies, Custom Commands, header- och ämnestriggers | avgör signering, kryptering, GINA, studs eller klartextväg |
| Kryptografi | privata S/MIME-/OpenPGP-nycklar, offentliga mottagarnycklar, CA-förtroende, OCSP/CRL, domännycklar | avgör om och för vem en operation är möjlig och verifierbar |
| GINA | domäner, layout, konton, lösenord, leveranshistorik, lagrade meddelanden och nycklar | utgör en ytterligare HTTPS-, identitets- och lagringsväg |
| Plattform | appliance-konfiguration, systemnamn, nätverk, certifikat, licenser, klusterroll | avgör tillgänglighet, administration och återstart |
| Observerbarhet | e-post-, system- och auditloggar, Postmaster-meddelanden, SNMP/NRPE/Syslog | tillhandahåller bevis och larm, men ingår inte fullständigt i backupen |

Indelningen följer den dokumenterade meny- och komponentstrukturen, GINA-kontohanteringen, klusterbeskrivningen och Quick Setup Guide. Den är en inventeringsmodell för administratörer, inte ett påstående om en odokumenterad intern databasstruktur ([SEPPmail: Mail System](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html), [SEPPmail: GINA User Details](https://docs.seppmail.com/en/07_mi_18_ga_01_user-x.html), [SEPPmail: Frontend/Backend Cluster](https://docs.seppmail.com/en/04_com_09_cl_05_frontend-backend-cluster.html), [SEPPmail: Quick Setup Guide](https://downloads.seppmail.com/wp-content/uploads/docu/Quick-Setup-Guide-en.pdf)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-seppmail.svg?v=20260813" title="Interaktive Infografik: SEPPmail-SMTP-Pfad, Ruleset, Verschlüsselungsverfahren, GINA, Schlüsselzustand, Cluster, Monitoring und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-seppmail.svg?v=20260813">Öppna interaktiv grafik direkt</a>.
</iframe>

## SMTP-mottagning, routning och relaygräns

En Managed Domain kopplar ett mottaget SMTP-namnrymd till den interna Forwarding Server och ytterligare domänspecifika inställningar. Gatewayen kan valfritt kontrollera mottagaradressen via SMTP-uppslag mot denna server. Den utgående vägen bestäms antingen av den inbyggda MTA:n via DNS eller genom en konfigurerad SMTP-server; domänspecifika TLS- och Sender-Based-Routing-inställningar kan åsidosätta denna standardväg ([SEPPmail: Mail System – Managed Domains and Outgoing Server](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)).

”Relaying allowed” definierar från vilka IP-adresser eller nät appliance-lösningen tar emot meddelanden till externa mottagare. Tillverkarens dokumentation varnar uttryckligen för att ange den utgående servern generellt, eftersom det kan skapa ett open relay. Portöppning, relaybehörighet, hanterad avsändardomän och eventuellt SMTP-autentisering är därför olika kontroller. Ett positivt TCP-test säger inget om relaybeslutet efter `MAIL FROM` och `RCPT TO` ([SEPPmail: Mail System – Relaying](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html), [RFC 5321: SMTP commands](https://datatracker.ietf.org/doc/html/rfc5321#section-4.1)).

Med flera gateways måste den föregående lastbalanseraren eller den sändande tjänsten kunna nå varje produktionsvärd. Samtidigt måste efterföljande system acceptera alla förväntade gatewayidentiteter. I Exchange Online gäller detta connectorcertifikat eller IP-intervall; i en klustermiljö dessutom virtuella IP-adresser och den dokumenterade egenheten att mottagning sker med IP-alias men sändning via den sändande systemets fysiska adress ([SEPPmail: Firewall and IP aliases](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html), [SEPPmail: Exchange Online clustering and multi-host](https://docs.seppmail.com/en/04_com_powershell_setup-exo.html)).

Efter SMTP-mottagningen avgör rulesetet vilken skydds- och leveransväg som gäller. Detta beslut måste senare kunna följas utifrån meddelande, identitet, katalogdata och regelversion.

## Ruleset, policyer och beslutsbevis

SEPPmail skapar körbara bearbetningsregler från inställningarna i Ruleset Generator. Ytterligare Encryption Policies kan tvinga fram metoder som enbart S/MIME, enbart OpenPGP, enbart GINA eller ett hierarkiskt urval för utvalda Managed Domains och valfria flaggor. Efter en policyändring måste rulesetet enligt tillverkaren genereras på nytt innan ändringen är aktiv. ”Sparat i GUI:t” och ”verksamt i den aktiva meddelandevägen” är därmed två olika tillstånd ([SEPPmail: Add/Edit Encryption Policy](https://docs.seppmail.com/ch/07_mi_07_mp_01_ep_01_addedit-encryption-policy.html)).

Custom Commands utökar denna pipeline med villkor och åtgärder. Tillverkaren dokumenterar bland annat LDAP-frågor, flaggor, användar- och nyckelskapande, headeråtgärder, signering, kryptering, leverans och avvisning. Makron expanderas när rulesetet genereras; ett syntaxfel kan förhindra genereringen ([SEPPmail: Macros](https://docs.seppmail.com/ch/08_cr_01_gi_04_macros.html), [SEPPmail: LDAP-based user creation](https://docs.seppmail.com/en/09_ht_user_creation-with-ldap.html)).

Header- eller ämnestriggers är indata till ett säkerhetsbeslut. De får endast utvärderas från en betrodd riktning eller rensas före utvärderingen. SEPPmail-exemplen för signering mellan klientorganisationer varnar uttryckligen för att förutsätta avsändaridentiteten utan kontroll. För varje trigger ska därför källa, tillåten connector, autentisering, normalisering, prioritet och fail-closed-beteende dokumenteras ([SEPPmail: Signing emails between clients](https://docs.seppmail.com/en/09_ht_tenant_signing-e-mails-between-tenants.html), [SEPPmail: Internal X-header processing](https://docs.seppmail.com/en/08_cr_01_gi_06_x-header.html)).

Ett reproducerbart beslutsbevis innehåller minst: envelope-avsändare och -mottagare, synlig `From:`- och vid behov `Sender:`-header, riktning och Managed Domain, utlösande policy respektive regel, hittade nycklar, vald metod, slutlig åtgärd och kö-ID. Detta är särskilt viktigt eftersom S/MIME-signaturidentitet normalt kontrolleras mot den synliga avsändaren, medan SMTP-routning först arbetar med envelopen ([SEPPmail: Sender identity selection](https://docs.seppmail.com/en/09_ht_user_creation-based-on-other-criteria-than-from-header.html), [RFC 5322: Originator fields](https://datatracker.ietf.org/doc/html/rfc5322#section-3.6.2)).

## S/MIME, OpenPGP och TLS som separata skyddslager

[S/MIME](https://datatracker.ietf.org/doc/html/rfc8551) kapslar in signerade eller krypterade MIME-innehåll med Cryptographic Message Syntax. Identiteter och offentliga nycklar binds vanligtvis genom X.509-certifikat; sökvägsvalidering, giltighetstid, nyckelanvändning och spärrinformation ingår därför i beslutet ([RFC 5652: CMS](https://datatracker.ietf.org/doc/html/rfc5652), [RFC 5280: PKIX certificate validation](https://datatracker.ietf.org/doc/html/rfc5280), [RFC 6960: OCSP](https://datatracker.ietf.org/doc/html/rfc6960)). SEPPmail kan hantera användar- och domäncertifikat och hämta dem via MPKI-gränssnitt; brandväggsdokumentationen anger HTTPS för detta samt HTTP eller HTTPS för OCSP/CRL ([SEPPmail: Firewall – MPKI and revocation](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html)).

[OpenPGP](https://datatracker.ietf.org/doc/html/rfc9580) använder en annan paket-, nyckel- och förtroendemodell. En befintlig offentlig nyckel styrker inte automatiskt att den tillhör den avsedda mottagaren eller fortfarande ska användas. Fingeravtryck, ursprung, User ID, utgång, återkallelse, algoritm och privat dekrypterbarhet måste hanteras som en livscykel. Encryption Policy skiljer mellan PGP/MIME och Inline PGP och kan konfigurera signering respektive kryptering separat ([SEPPmail: Encryption Policy – OpenPGP options](https://docs.seppmail.com/ch/07_mi_07_mp_01_ep_01_addedit-encryption-policy.html), [RFC 9580: Key structures](https://datatracker.ietf.org/doc/html/rfc9580#section-5.5)).

[SMTP-TLS](/kb/tls#tls-policy-bei-smtp) skyddar en anslutning mellan två hopp. Utan bindande policy är STARTTLS opportunistiskt: om det inte är tillgängligt kan den sändande MTA:n, beroende på konfiguration, leverera okrypterat. SEPPmail kan konfigurera TLS per mål med nivåer, fast server, certifikatfingeravtryck eller andra krav. Detta är varken S/MIME eller OpenPGP och skyddar inte meddelandet mot de medverkande MTA:erna ([RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207), [RFC 7672: SMTP DANE](https://datatracker.ietf.org/doc/html/rfc7672), [SEPPmail: Mail System – TLS Settings](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)).

Vid gatewaykryptering finns klartext och privata avsändarnycklar inom SEPPmails förtroendegräns. Det möjliggör centrala policyer men innebär också att administratörsroller, backuper, klusteranslutningar, debugexporter, loggar och efterföljande innehållsbearbetning måste skyddas som om de kunde beröra konfidentiella kommunikationsdata eller nycklar. Om en viss export faktiskt innehåller innehåll ska verifieras utifrån tillverkarens dokumentation och ett kontrollerat test.

Om ingen lämplig mottagarnyckel eller direkt skyddsväg finns tillgänglig flyttar GINA åtkomsten till en webbportal. Därmed tillkommer webbidentitet, session, lagring och meddelande-e-post till SMTP-vägen.

## GINA som webbtjänst med identitet och lagring

GINA är den dokumenterade reservvägen när ett meddelande inte levereras med en lämplig mottagarnyckel eller när en policy uttryckligen tvingar GINA. Kommunikation med portalen sker via HTTPS på TCP 443. GINA Domains styr bland annat portalnamn, layout, konto- och lösenordsbeteende, initial lösenordsöverföring samt lagringsparametrar ([SEPPmail: GINA Domains](https://docs.seppmail.com/en/07_mi_17_gd__gina-domains.html), [SEPPmail: Firewall – GINA](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html)).

För administratörer är GINA inte bara ett presentationsformat, utan ett eget tjänstetillstånd. Ett GINA-konto har status, lösenords- och eventuellt externa autentiseringsinställningar, användarloggar, e-posthistorik och en nyckel. Tillverkarens dokumentation varnar för att radering av kontot även oåterkalleligt raderar dess nyckel och att återstående GINA-meddelanden därmed kan bli oläsbara ([SEPPmail: GINA User Details](https://docs.seppmail.com/en/07_mi_18_ga_01_user-x.html)).

GINA-drift omfattar därför minst:

- ett stabilt, certifikatkompatibelt publikt värdnamn och dess [DNS](/kb/dns)-livscykel;
- HTTPS-certifikat, fullständig kedja, förnyelse och extern tillgänglighet;
- separat överföring eller härledning av den initiala hemligheten;
- kontospärr, återställning, extern identitet och supportprocess;
- lagrings- och raderingsregler för meddelande, metadata, loggar och nycklar;
- ett återställningstest som faktiskt öppnar ett GINA-meddelande som skapades före backupen efter återställningen.

Quick Setup Guide påpekar att en ändring av värdnamnet kan innebära att tidigare GINA-meddelanden inte längre går att nå. Den dokumenterar också att en restore till ett nytt system kräver lösenordet som angavs vid backupen ([SEPPmail: Quick Setup Guide](https://downloads.seppmail.com/wp-content/uploads/docu/Quick-Setup-Guide-en.pdf)). Värdnamn, certifikat, GINA-domän, konto och nyckel måste därför betraktas tillsammans.

## Katalog och livscykel för nycklar och certifikat

LDAP/LDAPS kan tillhandahålla offentliga nycklar, utvärdera användar- eller gruppvillkor och begränsa automatisk konto- respektive nyckelskapande. Appliance-lösningen kan dessutom erbjuda egna LDAP-slutpunkter för distribution av insamlade eller interna offentliga nycklar. Nödvändiga riktningar och portar skiljer sig åt beroende på roll; den officiella brandväggsmatrisen anger 389/636 för externa eller interna katalogfrågor samt separata portar för de integrerade keyserver-slutpunkterna ([SEPPmail: Firewall – LDAP and Key server](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html), [SEPPmail: LDAP-based user creation](https://docs.seppmail.com/en/09_ht_user_creation-with-ldap.html)).

En LDAP-fråga är en del av meddelandevägen så snart en regel inte kan fatta beslut utan dess resultat. Search Base, filter, objektklass, bind-identitet, TLS-förtroende, timeout, cachebeteende och beteende vid otillgänglighet måste därför ingå i driftmodellen. Ett tomt sökresultat, en timeout och ett icke betrott certifikat får inte behandlas som samma resultat. De allmänna protokoll- och TLS-kraven beskrivs i [LDAP-artikeln](/kb/ldap#bind-sasl-und-tls-vertrauensgrenzen).

För nycklar och certifikat krävs minst följande, separat mätbara livscykler:

| Objekt | Skapande eller hämtning | Löpande kontroll | Återställningsbevis |
|---|---|---|---|
| S/MIME-användarnyckel | lokalt, import eller MPKI | utgång, CA-sökväg, spärrstatus, adressbindning, algoritm | dekryptera gammalt meddelande med historisk privat nyckel |
| OpenPGP-nyckel | lokalt, import eller keyserver | fingeravtryck, User ID, utgång/återkallelse, kapaciteter | dekryptera gammalt meddelande med matchande Secret Key |
| Domännyckel | Managed Domain Service eller manuellt utbyte | publicerat motpartstillstånd, rotation, fullständig e-postväg | testa gammalt och nytt domänmeddelande över rotationsgränsen |
| SMTP-/HTTPS-certifikat | import eller lokal/offentlig CA | SAN, kedja, utgång, private-key match, peer-policy | starta tjänsten med återställd nyckel och förväntad kedja |
| CA-förtroende | Root-/Intermediate-import | syfte, omfattning, borttagning av gamla CA:er, OCSP/CRL | testa mot en känd giltig och spärrad kedja |

Kontrollegenskaperna följer respektive standarder och de dokumenterade import-, policy- och signaturkontrollerna. Vid en ogiltig S/MIME-signatur nämner SEPPmail särskilt CA-förtroende, fullständig kedja, avsändarbindning och efterföljande innehållsändring som orsaker ([SEPPmail: Reasons for an invalid signature](https://docs.seppmail.com/en/09_ht_signature_reasons-for-an-invalid-signature.html), [RFC 5280](https://datatracker.ietf.org/doc/html/rfc5280), [RFC 9580](https://datatracker.ietf.org/doc/html/rfc9580)).

Eftersom gateway, nycklar, katalog, GINA och kö samverkar måste hög tillgänglighet omfatta mer än en virtuell listener. Frontend-/backend-uppdelningen avgör vilka tillstånd som måste vara gemensamt tillgängliga.

## Kluster, frontend/backend-separation och felområden

Den allmänna klusterkommunikationen använder enligt brandväggsmatrisen SSH mellan appliances; tidssynkronisering via NTP är obligatorisk i klustret. Matrisen anger även virtuella IP-adresser och påpekar att sändning vid IP-alias sker via det aktiva systemets fysiska adress. Brandväggs-, smarthost- och connector-tillåtelselistor måste därför avbilda den faktiska avsändarvägen, inte bara den virtuella mottagaradressen ([SEPPmail: Firewall – Cluster and NTP](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html)).

Ett särskilt frontend/backend-kluster separerar e-postbearbetning och beständiga konfigurationsdata. Enligt tillverkarens beskrivning finns ingen konfigurationsdatabas på frontend; nödvändig konfiguration och nyckelmaterial tillhandahålls av backend för en åtgärd och kastas därefter på frontend. Vid aktiverad Large File Transfer krävs ändå ytterligare filtillstånd på frontend ([SEPPmail: Frontend/Backend Cluster](https://docs.seppmail.com/en/04_com_09_cl_05_frontend-backend-cluster.html)).

Denna arkitektur minskar den beständiga nyckelförvaringen i DMZ, men eliminerar inte alla beroenden. Backend, SSH-väg, namnupplösning, tid, certifikat och vid behov LFT-lagring förblir felområden. ”Frontend tar emot SMTP” bevisar varken nåbar backend eller en körbar kryptoåtgärd. Omvänt bevisar en fungerande backend inte att virtuell IP, extern rutt och nästa SMTP-hopp fungerar.

Alla inställningar är inte klusteromfattande. Mail System-dokumentationen beskriver flera maskinspecifika parametrar och i synnerhet Extended-Postfix-inställningar samt vissa TLS-mål som lokalt underhållna. Före failovertester är därför en plan-faktisk-matris per nod lämplig: systemnamn, IP, certifikat, Outgoing Identity, TLS-rutter, Extended Settings, lokala tjänster, övervakningsmål och förväntad klusterreplikering ([SEPPmail: Mail System – cluster notes](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)).

## Övervakning, loggning, backup och återställning

Enligt tillverkaren får Postmaster-adressen viktiga Watchdog- och Daily Report-meddelanden; utan en konfigurerad och faktiskt nåbar adress saknas denna larmväg. Därutöver dokumenterar SEPPmail SNMP via UDP 161, NRPE via TCP 5666 samt Syslog via UDP 514 eller TLS-skyddat TCP 6514. Vilka värden och händelser som övervakas måste fastställas per installation; ett öppet övervakningsgränssnitt ensamt är ingen hälsokontroll ([SEPPmail: Mail System – Postmaster](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html), [SEPPmail: Firewall – Monitoring and Syslog](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html)).

En meningsfull dashboard skiljer minst mellan SMTP-mottagning och -avvisningar, köålder per Next Hop, ruleset-fel, kryptoframgång och -fel, certifikatutgång, OCSP/CRL- och MPKI-nåbarhet, LDAP-latens, GINA-inloggnings- och leveransfel, diskstatus, klusterkommunikation och backupålder. Tröskelvärden behöver relateras till normal genomströmning och överenskommen RPO/RTO. En hög kö kan indikera en långsam motpart; en tom kö kan lika gärna betyda att inga meddelanden längre tas emot.

Quick Setup Guide beskriver en lösenordsskyddad backup som säkrar appliance-data inklusive registrerade användare och certifikat, men exkluderar loggar. Lösenordet krävs för restore till ett nytt system. Guiden beskriver även en nattlig leverans till ”Backup Operator” ([SEPPmail: Quick Setup Guide](https://downloads.seppmail.com/wp-content/uploads/docu/Quick-Setup-Guide-en.pdf)). Detta leder till tre separata kontroller:

1. Backupfilen måste komma fram utanför appliance-lösningen och dess ålder måste övervakas.
2. Backup-lösenordet måste förvaras separat, åtkomstskyddat och vara tillgängligt i en nödsituation.
3. Loggar behöver en egen export- och lagringsväg om de ska bevaras för forensik eller bevisning.

En restore är först testad när målsystemet startar, nätverk och licenser stämmer, regler är aktiva, SMTP fungerar i båda riktningarna, historiska S/MIME-/OpenPGP- och GINA-innehåll kan dekrypteras och ett dokumenterat testfall utför den förväntade policyn. Klusterreplikering ersätter inte detta test: den hanterar tillgänglighet, inte en oberoende kopia mot felkonfiguration, radering eller komprometterade nycklar.

Diagnostiken följer ett meddelande från SMTP-mottagning via regelmatchning och nyckelbeslut till vald utgång eller GINA-portal.

## Diagnosverktyg

Felsökningen följer meddelandet från det publika namnet via SMTP och TLS till rulesetet och vald krypteringsväg. Därmed styrks först transporten innan nyckel- eller portalfunktioner ändras.

### DNS, MX och publika namn

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

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) och [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) kontrollerar MX, gatewayadress och vid behov DANE-TLSA. Extern och intern resolvervy ska testas separat. GINA-värdnamnet måste dessutom matcha certifikatet; enbart MX säger inget om efterföljande Forwarding Server eller Exchange Online-connector ([RFC 5321: Address resolution and mail routing](https://datatracker.ietf.org/doc/html/rfc5321#section-5), [RFC 7672](https://datatracker.ietf.org/doc/html/rfc7672)).

### TCP-, SMTP- och STARTTLS-nåbarhet

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

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) och [`nc`](https://man.openbsd.org/nc) styrker endast TCP-nåbarhet. Windows-[`curl.exe`](https://curl.se/docs/manpage.html) kan begära STARTTLS; [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) visar SMTP-funktioner, handshake och certifikatkedja på Unix-system. För en kontrollerad SMTP-transaktion med envelope och DATA lämpar sig [`swaks`](https://jetmore.org/john/code/swaks/); den får endast användas mot auktoriserade testmottagare. Ett leveranstest är först komplett med kö-ID, logghändelse och ankomst till nästa hopp.

### GINA-HTTPS och certifikatnamn

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

[`Invoke-WebRequest`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest) och [`curl`](https://curl.se/docs/manpage.html) kontrollerar HTTP-nåbarhet och svarsbeteende; [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) synliggör den förhandlade TLS-sessionen och presenterade kedjan. Ett `200`, `301` eller `302` styrker varken en lyckad GINA-inloggning eller åtkomst till ett historiskt meddelande. För detta krävs ett syntetiskt användarflöde med registrering, inloggning, läsning, svar och återställning enligt den konfigurerade GINA-domänen.

### Paketväg vid en auktoriserad mätpunkt

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

[`pktmon`](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon) och [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) fångar endast trafik som passerar den valda mätpunkten. På en administrationsklient är detta inte automatiskt e-posttrafiken mellan e-postsystemet och gatewayen; en auktoriserad värd, port mirror eller annan lämplig observationspunkt krävs. SMTP före STARTTLS kan innehålla personuppgifter, medan HTTPS och STARTTLS åtminstone lämnar metadata. Insamling, åtkomst, lagring och radering ska därför ingå i incident-runbooken.

## Teknisk historia

SEPPmail beskriver företagets ursprung år 2001: grundaren Stefan Klein utvecklade först en lösning för krypterad kommunikation mellan en advokatbyrå i Zürich och dess klienter. Detta behov gav upphov till det senare produktföretaget ([SEPPmail: Die SEPPmail-Story](https://www.seppmail.com/de/die-seppmail-story/)).

Den offentligt dokumenterade produkthistoriken visar utbyggnaden från en central krypteringsgateway till en modulär appliance: historiska release notes nämner redan 2007 en intern CA/RA, S/MIME-domänkryptering, webbmailfunktioner, antispam/antivirus och en begränsad backupanvändare. Senare generationer lade bland annat till policyer för flera klientorganisationer, REST-gränssnitt, MPKI-connectors, Large File Transfer, DKIM/ARC, moderna S/MIME-metoder och specialiserade klusterroller ([SEPPmail: Revision History](https://docs.seppmail.com/ch/20_revision-history.html)).

Även underbyggnaden utvecklades vidare. Revisionshistoriken nämner genom åren OpenBSD, Postfix, Apache, Perl, GnuPG, OpenLDAP, OpenSSL/LibreSSL, Dovecot, Fetchmail och ClamAV. För administratörer är detta ursprung relevant eftersom loggbegrepp, MTA-parametrar, certifikatsformat och nätverksbeteende härrör från dessa komponenter. Direkta ingrepp under tillverkarens gränssnitt stöds dock inte automatiskt av detta; dokumentationen varnar särskilt för Extended-Postfix-ändringar och deras bristande klustersynkronisering ([SEPPmail: Mail System](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)).

## Källor

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
