---
title: "Cisco Secure Email: Gateway, AsyncOS och SMA"
blatt: "cisco"
description: "Cisco Secure Email för meddelandeadministratörer: SEG-/ESA-e-postpipeline, lyssnare, HAT och RAT, arbetskö och leverans, e-postpolicyer, AsyncOS, SMA-tjänster, klustergränser, teknikstack, övervakning, återställning och diagnostik."
fakten:
  - label: Produktroller
    wert: Secure Email Gateway (SEG/ESA) · Secure Email and Web Manager (SMA)
    href: https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/datasheet-c78-742868.html
  - label: Systemroll
    wert: SMTP-e-postgateway före eller mellan e-postsystem
    href: https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html
  - label: Operativsystem
    wert: Cisco AsyncOS
    href: https://www.cisco.com/c/en/us/support/security/email-security-appliance/series.html
  - label: Pipeline
    wert: Receipt → Work Queue → Delivery
    href: https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html
  - label: Mottagning
    wert: Listener · HAT · Sender Groups · RAT · LDAP
    href: https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html
  - label: Bearbetning
    wert: Message Filters · Mail Policies · Content Filters · Scan-Engines
    href: https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html
  - label: Centrala tjänster
    wert: Tracking · Reporting · Spam- och policykarantäner på SMA
    href: https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0100.html
  - label: Konfigurationskluster
    wert: peer-to-peer; ingen kö- eller trafik-HA
    href: https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0101000.html
  - label: Formfaktorer
    wert: virtuell appliance · Public Cloud · Cisco Cloud Gateway
    href: https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/datasheet-c78-742868.html
  - label: Administration
    wert: webbgränssnitt · CLI via SSH · REST-API
    href: https://docs.ces.cisco.com/docs/api
  - label: Primära tillstånd
    wert: konfiguration · kö · karantäner · tracking/reporting · nycklar och certifikat
    href: https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0100010.html
  - label: Ursprung
    wert: IronPort-teknik; förvärvad av Cisco 2007
    href: https://newsroom.cisco.com/c/r/newsroom/en/us/a/y2007/m01/cisco-announces-agreement-to-acquire-ironport.html
werbung:
  - tools
  - newsletter
ctaThemen:
  - cisco-esa-sma
translationSourceHash: c796a2c50226bbdcf5a0d7a7562913a5126b28cdbd05a5b72065afcf531bfd42
translationModel: gpt-5.6-terra
translatedAt: 2026-10-03T09:44:38.530Z
translationReview: automatic
---

# Cisco Secure Email: Gateway, AsyncOS och SMA

**Cisco Secure Email Gateway**, förkortat SEG och historiskt **Email Security Appliance** eller ESA, är en tillståndsfull [SMTP-e-postgateway](/kb/smtp). Den avslutar inkommande SMTP-sessioner, beslutar om mottagning, bearbetar meddelanden i en intern Work Queue och öppnar en ny SMTP-session för leverans. Den tekniska ansvarsgränsen ligger därmed inte vid en lyckad TCP- eller TLS-handshake, utan vid det positiva SMTP-svaret efter meddelandeinnehållet: Från denna tidpunkt måste gatewayen leverera eller generera ett standardenligt fel ([Cisco: Email Pipeline](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html), [RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321)).

Den andra klassiska rollen är **Cisco Secure Email and Web Manager**, SMA. Den står normalt inte som en reguljär MTA i den produktiva e-postvägen. Den hanterar centrala tracking- och rapporteringsdata samt – beroende på design – spam-, policy-, virus- och utbrottskarantäner från flera gateways. Ett SMA-avbrott kan därför lämna leveransen på SEG-noderna opåverkad, samtidigt som sökning, slutanvändarkarantän eller hantering av kvarhållna meddelanden avbryts ([Cisco: SMA Message Tracking](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0100.html), [Cisco: Centralized Quarantines](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0110.html)).

Båda rollerna körs på **AsyncOS**, en programvaruplattform som Cisco underhåller som en appliance-enhet. Administratörer hanterar inte de underliggande paketen som på en vanlig Linux-server; den tillförlitliga tekniska ytan består av AsyncOS-konfiguration, CLI, webbgränssnitt, REST-API, loggprenumerationer, MIB:er, uppdateringskanaler och de dokumenterade integrationerna. Ciscos översikter över öppen källkod visar många inbäddade komponenter, men ingen offentligt underhållbar stycklista över den proprietära e-postpipelinen. Enskilda bibliotek får därför inte likställas med den övergripande arkitekturen ([Cisco: Open Source Used in AsyncOS](https://www.cisco.com/c/dam/en_us/about/doing_business/open_source/docs/CiscoEmailSecurityAppliance-AsyncOS1253-1688323737.pdf), [Cisco: AsyncOS API](https://docs.ces.cisco.com/docs/api)).

Förklaringen följer ett meddelande genom Cisco Secure Email: från lyssnaren via HAT, RAT och Work Queue till leveransen. Därefter behandlas SMA, klusterdrift, beroenden, diagnostik och återställning.

## Produktroller och förtroendegränser

En typisk lokal design placerar minst två SEG-noder i DMZ och en SMA i ett internt hanteringsnät. DNS-MX eller en framförliggande tjänst distribuerar inkommande anslutningar till gatewayarna; utgående bestämmer e-postsystemets Smarthost-anslutningar gatewayvägen. Flera SEG-noder är högtilgängliga först när DNS, lastbalanserare eller den sändande MTA:n kan använda alternativa mål. AsyncOS-konfigurationsklustret tar inte ensamt över denna trafikstyrning ([Cisco: Centralized Management Using Clusters](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0101000.html), [RFC 5321, Address Resolution](https://datatracker.ietf.org/doc/html/rfc5321)).

Cisco dokumenterar virtuella appliances, Public Cloud-distributioner och en driftad Secure Email Cloud Gateway. Dessa varianter delar produktbegrepp, men flyttar ansvaret: För den virtuella appliance-enheten ansvarar kunden för hypervisor, nätverk, kapacitet och återställning; för Cloud Gateway tillhandahåller Cisco gatewayinfrastrukturen. **Secure Email Threat Defense** är i sin tur en molnbaserad analys- och skyddsplattform som kan integreras via gateway, journaling eller Microsoft-API. Den är varken en synonym för den lokala Work Queue eller en ersättningsterm för SMA ([Cisco: Secure Email Data Sheet](https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/datasheet-c78-742868.html), [Cisco: Email Threat Defense Data Sheet](https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/secure-email-threat-defense-ds.html)).

| Roll | I SMTP-sökvägen | Beständigt tillstånd | Effekt vid avbrott |
|---|---:|---|---|
| Secure Email Gateway | ja | Kö, lokala karantäner, konfiguration, certifikat, loggar | Mottagning eller leverans på denna nod störd |
| Secure Email and Web Manager | normalt nej | Tracking, rapportering, centrala karantäner, Safe-/Blocklists, egen konfiguration | Synlighet och centrala karantäntjänster påverkas |
| Email Threat Defense | beroende på integration | Molnbaserad telemetri, undersökning och policyer | Ytterligare analys eller åtgärd påverkas |
| E-postsystem | före eller efter gatewayen | Postlådor, transportköer, anslutningar | Användaråtkomst eller end-to-end-leverans påverkas |

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-cisco.svg?v=20260813" title="Interaktive Infografik: Cisco Secure Email mit SEG-Mailpipeline, Listener, HAT und RAT, Work Queue, Delivery, SMA-Diensten, Konfigurationscluster und Admin-Kontrollpunkten" loading="lazy">
  <a href="/images/kb-interaktiv-cisco.svg?v=20260813">Öppna interaktiv grafik direkt</a>.
</iframe>

## Receipt: Listener, HAT och RAT

En **Listener** binder SMTP till ett IP-gränssnitt och utgör den första policygränsen. Public Listeners tar normalt emot internettrafik för lokala domäner; Private Listeners tar emot utgående meddelanden från kontrollerade nät. Dessa roller är konfiguration, inte en inneboende förtroendeegenskap hos porten. En Private Listener med alltför bred relaybehörighet är en Open Relay, även om den har ett internt namn.

**Host Access Table**, HAT, tilldelar anslutande värdar till Sender Groups. Deras Mail Flow Policies bestämmer bland annat om en anslutning accepteras, avvisas, begränsas eller behandlas utan enskilda skanningar. **Recipient Access Table**, RAT, definierar lokala mottagardomäner för inkommande meddelanden. Valfritt kontrollerar [LDAP](/kb/ldap) konkreta mottagare under SMTP-sessionen eller senare i Work Queue; alternativt kan SMTP Call-Ahead fråga den efterföljande servern. Cisco skiljer därmed mellan fyra identiteter som inte får blandas ihop vid en störning: käll-IP, envelope-avsändare, envelope-mottagare och headeridentiteter ([Cisco: Email Pipeline, Incoming](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html)).

En tydlig Listener-dokumentation innehåller minst bindnings-IP och port per riktning, tillåtna källnät, förväntade EHLO-namn, TLS-läge, krav på klientcertifikat, HAT-ordning, RAT-domäner, mottagarkontroll, maximal meddelandestorlek, rate limits och bounce-profil. Ordningen är särskilt kritisk: Ett tidigt HAT-avslag skapar ingen Message-ID-post som ett senare accepterat meddelande; en helpdesk kan därför inte hitta det med samma sökning.

Efter SMTP-mottagning börjar den egentliga innehålls- och policybearbetningen. Dess ordning är viktig eftersom ett tidigt resultat kan påverka senare kontroller, mottagargrupper eller leveransvägar.

## Work Queue: ordning, splintering och policy

Efter mottagningen hamnar meddelandet i **Work Queue**. Cisco dokumenterar där routing och masquerading, Message Filters, Safe-/Blocklists, Anti-Spam, Anti-Virus, Graymail, filreputation och -analys, Content Filters, Outbreak Filters och karantäner. Ordningen är en del av säkerhetsmodellen. En policyändring gäller i allmänhet inte retroaktivt för redan lagrade meddelanden; en efterföljande aktivering av en skanner reparerar därför inte automatiskt en tidigare bypass ([Cisco: Email Pipeline, Work Queue](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html)).

**Message Filters** arbetar före mottagarrelaterad Mail Policy och kan ändra, arkivera, sätta i karantän, studsa eller kasta meddelanden baserat på envelope, headers, innehåll, bilagor eller anslutningsdata. Därefter kan AsyncOS **splintra** ett meddelande med flera mottagare: Separata Message IDs och därmed olika sluttillstånd skapas för olika mottagarpolicyer. Ett enda ursprungligt Inject- eller ICID-värde kan följaktligen förgrena sig till flera MID:er och leveransresultat. Tracking måste visa detta träd, inte bara söka efter ämnesrad.

**Mail Policies** styr mottagar- eller avsändarrelaterade skanningar och Content Filters. Enligt Cisco är DLP begränsat till utgående bearbetning. Licenser, motorernas uppdateringsstatus och molnanslutning avgör dessutom vilka kontroller som faktiskt sker. För varje policy behöver ett tillförlitligt test ett ofarligt positivt fall, ett riktat negativt fall och det förväntade sluttillståndet – leverans, ändring, karantän, drop eller bounce.

## Delivery: SMTP Routes, Destination Controls och kö

I Delivery-fasen väljer AsyncOS rutt, källgränssnitt och mål. **SMTP Routes** åsidosätter normal MX-upplösning för konfigurerade domäner; **Destination Controls** begränsar parallella anslutningar och mottagare per mål. Virtual Gateways kan tillhandahålla olika käll-IP-adresser, värdnamn och leveransköer. Dessa inställningar påverkar rykte, SPF, tillåtelselistor hos motparter och platsen där ett meddelande väntar ([Cisco: Email Pipeline, Delivery](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html), [RFC 7208](https://datatracker.ietf.org/doc/html/rfc7208)).

Utgående TLS är hop-by-hop. AsyncOS kan använda [STARTTLS](/kb/tls) med motparter; framgång skyddar denna transportsträcka men säger inget om tidigare eller efterföljande hopp. För tvingande policyer måste målpattern, certifikatkontroll, namnknytning och felbeteende dokumenteras. Opportunistisk TLS får falla tillbaka till klartext vid ett handshakefel; en obligatorisk policy måste i stället köa eller misslyckas ([Cisco: Verify and Troubleshoot TLS Certificates](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118844-technote-esa-00.html), [RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207)).

Köålder är viktigare än enbart kölängd. En hög volym kan vara frisk vid hög genomströmning; få mycket gamla meddelanden tyder på en ihärdig mål-, DNS-, TLS- eller policyblockering. För varje rutt ska den äldsta meddelandet, retryorsak, nästa försök, målsvar och ansvarig motpart ingå i incidentbilden.

Så snart ESA har vidarebefordrat eller satt ett meddelande i karantän flyttas en del av administratörsvyn till SMA. Den ersätter dock inte de lokala kö- och systemdata på ESA.

## SMA: tracking, rapportering och karantäner

SMA samlar tracking- och rapporteringsdata från flera SEG-noder. Message Tracking kan visa sluttillstånd som `Delivered`, `Dropped`, `Bounced`, `Quarantined`, `Queued`, `Processing` och `Splintered`. Det är dock ett härlett index: Om exportdata saknas, en tjänst är fördröjd eller meddelandet ligger utanför lagringsperioden, bevisar ett tomt träffresultat inte att meddelandet aldrig behandlats. Primärt bevis är de matchande Mail Logs och MID-kedjan på SEG ([Cisco: Tracking Messages](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0100.html)).

Spamkarantän och policy-/virus-/utbrottskarantäner är separata tjänster med olika användare, frisläppningsvägar och lagringstider. Centraliserade karantäner lagrar meddelanden på SMA bakom brandväggen och kan inkluderas i dess standardsäkerhetskopiering. Vid 75, 85 och 95 procents beläggning genererar AsyncOS dokumenterade tröskelvarningar. Om en central karantäntjänst blir otillgänglig behöver driften ett i förväg testat beslut: tillfälligt köa, växla lokal bearbetning eller kontrollerat inaktivera den tillhörande policyn ([Cisco: Centralized Quarantines](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0110.html), [Cisco: Centralizing Services](https://www.cisco.com/c/en/us/td/docs/security/esa/esa13-7-0/user_guide/b_ESA_Admin_Guide_13-7/b_ESA_Admin_Guide_12_1_chapter_0101011.html)).

## Konfigurationskluster är inte e-post-HA

AsyncOS kan ansluta flera gateways till ett peer-to-peer-**konfigurationskluster**. Inställningar kan hållas på kluster-, grupp- eller maskinnivå; det finns ingen primär klusternod. Medlemmarna måste använda en kompatibel AsyncOS-version och kommunicerar via SSH eller Cluster Communication Service. Klustret replikerar konfiguration, inte aktiva SMTP-sessioner, köinnehåll, lokala karantäner eller leveransförlopp ([Cisco: Centralized Management Using Clusters](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0101000.html)).

Därför finns tre separata mekanismer:

- **Trafikdistribution:** flera MX-mål, lastbalanserare eller Smarthost-failover;
- **Konfigurationskonsistens:** AsyncOS-kluster med tydliga överstyrningar för kluster, grupper och maskiner;
- **Datatillgänglighet:** köstatus per SEG samt tracking- och karantändata på SMA.

En nodförlust efter positiv SMTP-mottagning kan påverka meddelanden som endast finns i dess lokala kö. Avsändaren får inte bara skicka dem igen så länge den ursprungliga leveransstatusen är oklar; annars uppstår dubbletter. Ett återställningstest måste därför inte bara ladda konfigurationen, utan följa accepterade testmeddelanden genom ett kontrollerat nodfel.

## Teknikstack och administrationsytor

AsyncOS är en sluten appliance-plattform. Cisco publicerar information om öppen källkod för medföljande komponenter, men ingen fullständig källkods- eller språkplan för proprietära tjänster. Påståenden som ”skriven i Python” eller ”baserad på FreeBSD” är utan versionsspecifikt tillverkarunderlag ingen tillförlitlig driftinformation. För administratörer är följande verifierbara stack mer relevant ([Cisco: Open Source Used in AsyncOS](https://www.cisco.com/c/dam/en_us/about/doing_business/open_source/docs/CiscoEmailSecurityAppliance-AsyncOS1253-1688323737.pdf)):

| Nivå | Verifierbar teknik | Driftsrelevans |
|---|---|---|
| E-posttransport | SMTP-Listener, Receipt, Work Queue, Delivery Queue | Mottagningsgräns, policyordning, retry och bounce |
| Policy och analys | HAT/RAT, Message Filters, Mail Policies, Content Filters, Scan-Engines | Ordning, licenser, motoruppdateringar, splintering |
| Data och sökning | lokala köer/karantäner; SMA-tracking, rapportering och centrala karantäner | Kapacitet, lagring, backup och dataskydd |
| Administration | HTTPS-GUI, interaktiv CLI via SSH, XML-konfiguration | Ändring, commit, export, restore och revision |
| Automation | RESTful AsyncOS API med Swagger | Rapportering, tracking och karantänåtkomst; ingen otestad fullständig konfiguration |
| Telemetri | Mail Logs, ytterligare Log Subscriptions, Syslog, Alerts, SNMP/MIB, API | Korrelation via ICID/MID/DCID och resursstatus |
| Plattform | Hårdvaru-, virtuell och moln-appliance | Ansvar för compute, lagring, nätverk och livscykel |

CLI-ändringar följer ett transaktionsmönster: Kommandon ändrar först en körande konfiguration, `commit` aktiverar den, `clearchanges` förkastar den. En runbook måste ange hela dialogen och konfigurationsläget; rena copy-and-paste-fragment är farliga på grund av skillnader mellan versioner och kluster. REST-API:t ger säkert autentiserad åtkomst till rapporter, räknare, tracking- och karantändata; dess lokala Swagger-gränssnitt dokumenterar det faktiskt installerade API-omfånget ([Cisco: AsyncOS API](https://docs.ces.cisco.com/docs/api), [Cisco: SEG Support Documentation](https://www.cisco.com/c/en/us/support/security/email-security-appliance/series.html)).

## Nätverks-, identitets- och tidsberoenden

När e-postpipelinen är fastställd kan dess externa anslutningar kontrolleras. Varje rad visar vilket system som initierar en anslutning, vad den behövs för och hur ett fel märks.

| Anslutning | Vanlig port | Initiator | Syfte och felbild |
|---|---:|---|---|
| SMTP | TCP 25 | extern MTA, internt e-postsystem eller SEG | Mottagning och vidarebefordran; timeout, 4xx/5xx, köökning |
| HTTPS | TCP 443 respektive konfigurerad | Admin, slutanvändare eller API-klient | GUI, API, karantän; kontrollera certifikat, SSO och roller separat |
| SSH | TCP 22 respektive konfigurerad | Admin eller SEG-medlem | CLI och valfri klusterkommunikation |
| CCS | TCP 2222 som standard, konfigurerbar | SEG-medlem | Konfigurationskluster; inget e-postflöde |
| DNS | UDP/TCP 53 | SEG/SMA | MX, A/AAAA, PTR, reputation och uppdateringar |
| LDAP/LDAPS | TCP 389/636 | SEG/SMA | Mottagare, routing, grupper och adminautentisering |
| Syslog | UDP/TCP 514 eller TLS 6514 enligt design | SEG/SMA | Extern loggtransport; definiera modell för förlust och backpressure |
| SNMP | UDP 161/162 | Övervakning respektive appliance | Statusfrågor och traps; föredra SNMPv3 |

Portnummer ensamma bevisar ingen aktiv funktion. Tilldelningarna kommer från [IANA Service Name and Port Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml); Cisco dokumenterar CCS och dess konfigurerbarhet i klusterkapitlet. Brandväggar bör omfatta källa, mål, riktning, protokoll, TLS- eller autentiseringskrav och affärssyfte.

[LDAP](/kb/ldap) kan tillhandahålla mottagarmottagning, routing, gruppmedlemskap och adminautentisering. Dessa frågor har olika scheman, timeouter och felkonsekvenser. Om Recipient Acceptance fallerar kan systemet beroende på konfiguration fördröjt studsa eller kasta; ett autentiseringsfel i GUI:t är därför inget bevis på fel i SMTP-mottagarkontrollen. Tjänstkonton, Base DNs, filter, referralbeteende, certifikatkedja och failoverordning ska dokumenteras per fråga ([Cisco: Email Pipeline, LDAP Recipient Acceptance](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html), [RFC 4511](https://datatracker.ietf.org/doc/html/rfc4511)).

DNS och korrekt tid är systemberoenden. MX- och värdupplösning styr leverans och klustrets nåbarhet; PTR och reputation påverkar klassificering. NTP håller logg-, Received- och trackingtider korrelerbara. Cisco kräver för kluster upplösbara värdnamn eller konsekvent använda IP-adresser och beskriver systemtid samt NTP som del av grundkonfigurationen ([Cisco: Setup and Installation](https://www.cisco.com/c/en/us/td/docs/security/esa/esa13-7-0/user_guide/b_ESA_Admin_Guide_13-7/b_ESA_Admin_Guide_12_1_chapter_010.html)).

Vid störningar följs samma väg baklänges: Deliverystatus, Work Queue-beslut, Receipt-policy, Listener och nätverksberoenden.

## Övervakning och incidenttriage

Den centrala frågan är: **Har gatewayen tagit emot meddelandet, bearbetat det och till vilket hopp överlämnat det?** För detta kedjas anslutnings-, Message- och Delivery-ID:n från Mail Logs samman. Message Tracking på SMA snabbar upp sökningen, men ersätter inte råloggarna. Meningsfulla tekniska signaler är:

- mottagningshastighet, 4xx-/5xx-svar och avvisade anslutningar per Listener och Sender Group;
- Work- och Delivery Queue, ålder på det äldsta meddelandet samt återkommande målsvar;
- processing- och splinteringtid, Scan-Engine-fel och uppdateringsålder;
- beläggning i lokala och centrala karantäner, frisläppnings- och raderingshändelser;
- Resource-Conservation-värde, CPU, minne, diskbeläggning och kritiska Alerts;
- nåbarhet och latens för DNS, LDAP, SMA, uppdaterings- och molntjänster;
- klusterkonsistens och oavsiktliga Machine-Overrides;
- utgång och användning för varje TLS-certifikat samt ändringar i Truststore.

I **Resource Conservation Mode** begränsar AsyncOS mottagningen stegvis så att leveransen kan minska eftersläpningen; vid extrem resursbrist tas inga nya meddelanden emot. Symptomet är därför ofta minskad inkommande genomströmning, medan den faktiska utlösaren är en långsam målroute eller full resurs. Cisco tillhandahåller status och Alerts i GUI/CLI; SNMPv3 och `ASYNCOS-MAIL-MIB` möjliggör extern övervakning ([Cisco: Resource Conservation](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/117834-qanda-esa-00.html), [Cisco: SNMP Monitoring](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/117831-qanda-esa-00.html)).

## Backup, återställning och uppgradering

En exporterad XML-konfigurationsfil är nödvändig, men inte en fullständig systembackup. Cisco dokumenterar `saveconfig`, `mailconfig` och `loadconfig`; maskerade lösenfraser kan inte laddas igen. Certifikat och nycklar, klustertillstånd, Feature Keys, lokala köer, lokala karantäner, SMA-data samt externa beroenden behöver egna bevis ([Cisco: System Administration](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0100010.html), [Cisco: Automated Configuration Backup](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118403-technote-esa-00.html)).

| Återställningsobjekt | Säkerhetskopiering eller rekonstruktion | Acceptanstest |
|---|---|---|
| SEG-konfiguration | omaskerad, skyddat lagrad export plus dokumenterade lösenfraser | ladda på ersättningsinstans, diff och Listener-/policytest |
| Certifikat och privata nycklar | krypterad nyckelbackup, CA-kedja och rollmatris | HTTPS- och SMTP-TLS-handshake med namnkontroll |
| Lokal kö | kan normalt inte rekonstrueras från konfigurationsbackup | nodfel med accepterat testmeddelande och dubblettkontroll |
| SMA-data | SMA-backup för tracking, rapportering, karantäner och listor | kontrollera sökning, frisläppning av testmeddelande och lagring |
| Kluster | export per nivå plus dokumenterade Machine-Overrides | återanslut medlem och kontrollera konsistens |
| Externa tjänster | DNS-, LDAP-, Syslog-, NTP-, uppdaterings- och molnkonfiguration | syntetisk end-to-end-kontroll |

Uppgraderingar är appliance-migreringar. Innan dess ska målsökväg, kompatibla mellansteg, hypervisor- eller molnkrav, funktionsändringar, klusterordning, ledigt utrymme, driftstopp och rollbackgräns kontrolleras. Ciscos releasekategorier GD och MD är ingen automatisk rekommendation för varje miljö; avgörande är Security Advisories, supportmatrisen och det testade egna policyomfånget. Supportsidan och livscykelförklaringarna ska ingå i patchförfarandet, inte som ett statiskt versionsnummer i artikeln ([Cisco: SEG Release Notes](https://www.cisco.com/c/en/us/support/security/email-security-appliance/products-release-notes-list.html), [Cisco: Software Lifecycle Support Statement](https://www.cisco.com/c/dam/en/us/td/docs/security/esa/lifecycle_support_statement/Secure_Email_Gateway_Software_Lifecycle_Support_Statement.pdf)).

## Diagnostikverktyg

Felsökningen följer meddelandevägen utifrån och in. Först kontrolleras namn och nåbarhet, därefter SMTP-mottagning, pipelinehändelser, kö och vid behov SMA-utvärderingen.

### DNS, MX och målupplösning

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

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) och [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) visar MX-, forward- och reverseupplösning. Frågan ska upprepas ur interna och externa resolverperspektiv; AsyncOS SMTP Routes kan åsidosätta det synliga MX-resultatet.

### TCP och SMTP-TLS

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

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) och [`nc`](https://man.openbsd.org/nc) visar endast TCP-sökvägen. [`curl`](https://curl.se/docs/manpage.html) och [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) begär STARTTLS och visar handshake samt certifikatkedja; först den förväntade namn- och trustkontrollen bevisar den konfigurerade TLS-policyn.

### Auktoriserat SMTP-testmeddelande

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

[`curl`](https://curl.se/docs/manpage.html) och [`swaks`](https://jetmore.org/john/code/swaks/) skickar ett kontrollerat testmeddelande. Avsändare, mottagare och mål måste vara auktoriserade. Dokumentera SMTP-slutsvar, ICID/MID, splinter-MID:er, policy, karantän- eller Deliverystatus och faktisk ankomst.

### Webbgränssnitt och REST-API

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

[`Invoke-WebRequest`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest) och [`curl`](https://curl.se/docs/manpage.html) kontrollerar HTTP- och TLS-nåbarhet. En statuskod bevisar varken inloggning eller rollbehörighet, trackingimport eller karantänfunktion. Swagger-sidan beskriver endast API:t för den tilltalade instansen; produktiva API-tester använder ett minimalt skrivskyddat konto och lagrar inga tokens i shellhistoriken.

### Paketväg vid en auktoriserad mätpunkt

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

[`pktmon`](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon) och [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) ser endast trafiken vid den valda mätpunkten. En adminklient observerar inte automatiskt vägen mellan lastbalanserare, SEG, SMA och mål-MTA. Paketdata kan innehålla SMTP-innehåll före STARTTLS och personrelaterade metadata och ska skyddas därefter.

## Teknisk historia

IronPort Systems utvecklade specialiserade meddelandegatewayer och produktlinjen AsyncOS. Cisco tillkännagav förvärvet av företaget i januari 2007 och placerade dess e-post- och webbsäkerhetsteknik i sin egen säkerhetsportfölj ([Cisco: Agreement to Acquire IronPort](https://newsroom.cisco.com/c/r/newsroom/en/us/a/y2007/m01/cisco-announces-agreement-to-acquire-ironport.html)). Produktnamnen ändrades därefter från Cisco IronPort Email Security Appliance via Cisco Email Security Appliance till **Cisco Secure Email Gateway**; historiska termer som ESA, C-Series och M-Series förblir synliga i runbooks, loggmeddelanden, licenser och dokumentationssökvägar.

Arkitekturidén förblev igenkännbar genom dessa namnbyten: en specialiserad gateway med den trestegade pipelinen Receipt, Work Queue och Delivery samt ett separat hanteringssystem för aggregerade data och karantäner. Senare tillkom virtuella och Public Cloud-appliances, Cloud Gateway, REST-API:er och molnbaserade analystjänster. Email Threat Defense utökar portföljen med API-, journaling- och gatewaymodeller; det förändrar inte i efterhand tillståndsgränserna för en befintlig ESA-/SMA-installation ([Cisco: Secure Email Data Sheet](https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/datasheet-c78-742868.html), [Cisco: Email Threat Defense](https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/secure-email-threat-defense-ds.html)).

Namnet på en installerad appliance räcker därför inte som livscykelinformation. Hårdvarumodell, virtuell plattform, AsyncOS-gren, aktiverade licenser, motor- och regeluppdateringar samt beroende molntjänster har egna livscykler. Ciscos support-, release- och End-of-Life-sidor är dynamiska driftkällor; en statisk artikel bör länka till dem men inte fastslå en förment permanent aktuell versionsstatus ([Cisco: SEG End-of-Life Notices](https://www.cisco.com/c/en/us/products/security/email-security-appliance/eos-eol-notice-listing.html), [Cisco: SEG Support](https://www.cisco.com/c/en/us/support/security/email-security-appliance/series.html)).

## Källor

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
