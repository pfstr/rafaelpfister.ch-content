---
title: "Apache James: modulär e-postserver och Mailet-plattform"
blatt: "apache-james"
description: "Apache James ur ett tekniskt perspektiv: protokoll och e-postroller, komponentbaserad arkitektur, kö och Mailet-pipeline, postlåde- och lagringsmodell, driftsvarianter från PostgreSQL till Cassandra samt utvecklingen från Java Apache-projekt till JVM-e-postplattform."
fakten:
  - label: Fullständigt namn
    wert: Java Apache Mail Enterprise Server
    href: https://james.apache.org/
  - label: Kategori
    wert: MTA, MDA, postlådeserver och e-postapplikationsplattform
    href: https://james.apache.org/documentation.html
  - label: Projekt
    wert: Apache Software Foundation
    href: https://projects.apache.org/committee.html?james
  - label: Körmiljö
    wert: JVM · Java 21 från version 3.9
    href: https://james.apache.org/james/update/2025/09/25/james-3.9.0.html
  - label: Språk
    wert: huvudsakligen Java, enskilda moduler i Scala
    href: https://github.com/apache/james-project
  - label: Protokoll
    wert: SMTP, LMTP, IMAP, POP3, ManageSieve, JMAP
    href: https://james.apache.org/server/feature-protocols.html
  - label: Arkitekturstil
    wert: modulär, komponentbaserad, Inversion of Control, händelsedriven
    href: https://james.apache.org/
  - label: Backends
    wert: PostgreSQL/JPA eller Cassandra · OpenSearch · RabbitMQ · S3
    href: https://james.apache.org/download.cgi
  - label: Konfiguration
    wert: conf/*.xml och *.properties · miljövariabler
    href: https://james.apache.org/server/config.html
  - label: Paketering
    wert: ZIP-distributioner och officiella Docker-images
    href: https://james.apache.org/download.cgi
  - label: Administration
    wert: WebAdmin REST API, CLI och mätvärden
    href: https://james.apache.org/server/manage-webadmin.html
  - label: Övervakning
    wert: Health Checks, Prometheus, JMX, loggar och Grafana
    href: https://james.apache.org/server/metrics.html
  - label: Licens
    wert: Apache License 2.0
    href: https://www.apache.org/licenses/LICENSE-2.0
werbung:
  - newsletter
ctaThemen:
  - smtp-mailflow
translationSourceHash: adbfb005b83b16086ba55e53dd469f3aff1e5642364da5ab8b2da5d265a1ce51
translationModel: gpt-5.6-terra
translatedAt: 2026-10-03T09:25:46.668Z
translationReview: required
---

# Apache James: modulär e-postserver och Mailet-plattform

Apache James är en e-postserver med öppen källkod och samtidigt en byggsats för applikationer vars affärslogik bygger på e-post. Namnet står för **Java Apache Mail Enterprise Server**. James kan ta emot och vidarebefordra meddelanden via SMTP, hantera lokala postlådor, tillgängliggöra dem via IMAP, POP3 eller JMAP och styra hela meddelandeflödet genom fritt kombinerbara bearbetningskomponenter. Projektet beskriver sig därför inte bara som en server utan som en modulärt sammansättbar **Inversion-of-Control-plattform på JVM** ([Apache James – projektöversikt](https://james.apache.org/)).

Denna dubbla roll skiljer James från klassiska Mail Transfer Agents som Postfix och från färdiga säkerhetsappliance-lösningar. En administratör kan driva James som ett rent SMTP-relä, som en komplett postlådeserver eller som en inbäddad e-postmotor i en produkt. Spamkontroll, kryptering, arkivering eller domänspecifik routning uppstår då inte ur ett fast funktionsblock utan ur en pipeline av **Matchers** och **Mailets**. Det gör James exceptionellt anpassningsbar, men flyttar en del av produktansvaret från tillverkaren till den organisation som driver systemet.

Förklaringen följer ett meddelande genom James: från protokollservrarna via kön och Mailet-pipelinen till postlådearkivet. Därpå följer driftsvarianter, diagnostik och slutligen projektets tekniska utveckling.

## Klassificering: MTA, MDA och applikationsplattform

I e-postsystemet har inte varje komponent samma roll. En **Mail User Agent** (MUA) är användarens klient, till exempel Thunderbird. En **Mail Transfer Agent** (MTA) transporterar meddelanden mellan system. En **Mail Delivery Agent** (MDA) lägger ett meddelande i målpostlådan. James kan vara både MTA och MDA; genom sina protokoll- och postlådemoduler tillhandahåller den dessutom servertjänster för MUA:er. Den officiella komponentöversikten listar separata projekt för server, protokoll, Mailets, postlådor och tester ([Apache James – Software Components](https://james.apache.org/documentation.html)).

| Roll | Implementering i James | Överlämningspunkt |
|---|---|---|
| Meddelandetransport | SMTP- och LMTP-server, kö, Remote-Delivery-Mailet | andra MTA:er, reläer och gateways |
| Lokal leverans | Mailet-pipeline och Mailbox API | användare, domäner och kvoter |
| Postlådeåtkomst | IMAP, POP3 och JMAP | e-postklienter och webbapplikationer |
| Filterlogik | Matchers, Mailets, Processors och Sieve | interna regler och externa kontrolltjänster |
| Administration | WebAdmin REST API, CLI, Health Checks och mätvärden | automatisering och övervakning |

James är därmed **ingen e-postklient** och inte heller en förkonfigurerad Secure-Mail-Gateway. Den tillhandahåller byggblock för transport, leverans, lagring och bearbetning. Om resultatet blir ett enkelt relä, en multitenant-e-posttjänst eller en produktspecifik gateway avgörs av vald distribution och konfiguration.

## Protokoll, TLS och portar

James tillhandahåller SMTP, LMTP, IMAP, POP3 och ManageSieve som TCP-baserade tjänster; JMAP och WebAdmin använder HTTP ([Apache James – Protocol Servers](https://james.apache.org/server/feature-protocols.html)). TLS skyddar, beroende på listener, en anslutning som är krypterad från början eller läggs in i en befintlig session via StartTLS. DNS ingår inte i James-processen, men är oumbärligt för en publik MTA: MX-poster bestämmer målet, A- och AAAA-poster dess adresser och PTR-poster påverkar ryktet för utgående anslutningar.

Portnumret ensamt beskriver ännu inte säkerhetssemantiken. Port 25 är avsedd för server-till-server-transport; autentiserad inlämning från klienter hör enligt [RFC 6409](https://datatracker.ietf.org/doc/html/rfc6409) till port 587. Port 465 är sedan [RFC 8314](https://datatracker.ietf.org/doc/html/rfc8314) åter registrerad för implicit krypterad Message Submission. För IMAP och POP3 gäller samma två mönster: klartextanslutning med möjlig StartTLS eller omedelbar TLS-etablering.

| Tjänst | Typiska portar | Standard | Betydelse i James |
|---|---:|---|---|
| SMTP | 25, 587, 465 | [RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321) | mottagning, relä och submission |
| LMTP | konfigurerbar, registrerad 24 | [RFC 2033](https://datatracker.ietf.org/doc/html/rfc2033) | lokal överlämning med status per mottagare |
| IMAP4rev2 | 143, 993 | [RFC 9051](https://datatracker.ietf.org/doc/html/rfc9051) | synkron postlådeåtkomst |
| POP3 | 110, 995 | [RFC 1939](https://datatracker.ietf.org/doc/html/rfc1939) | enkel meddelandehämtning |
| ManageSieve | 4190 | [RFC 5804](https://datatracker.ietf.org/doc/html/rfc5804) | hantering av användarspecifika Sieve-regler |
| JMAP Mail | vanligtvis 443 | [RFC 8621](https://datatracker.ietf.org/doc/html/rfc8621) | HTTP-baserad postlådeåtkomst för moderna klienter |

Portarna är konfigurerbara; avgörande är kombinationen av listener, protokoll, TLS-läge och autentisering. [IANA Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) är fortsatt referensen för registrerade tilldelningar.

## Arkitekturprincip

James följer en **komponentbaserad arkitektur**. Protokollservrar, kö, bearbetningslogik, postlåda, användarhantering, sökindex och administration är åtskilda genom API:er och sätts samman via dependency injection. De distributioner som dokumenterats för James 3.9 använder Google Guice för detta; Spring-upplägget tillhör en äldre generation. Frikopplingen är inte bara en kodorganisation: den gör det möjligt att använda samma [Mailbox API](https://james.apache.org/mailbox/mailbox-api.html) med olika persistenslager och samma Mailet-logik i mycket olika serverprofiler.

Den centrala datavägen är asynkron. En SMTP-listener behöver inte leverera ett mottaget meddelande fullständigt innan den svarar på anslutningen. Den lägger ett Mail-objekt i en kö; en **Spooler** hämtar det senare och kör det genom Mailet-containern. Kön skiljer därmed mottagningsbelastning, bearbetningstid och tillgängligheten hos efterföljande system åt. Dokumentationen för distribuerad drift beskriver den följdriktigt som en obligatorisk del av en SMTP-server ([Apache James – Distributed Server Operations](https://james.apache.org/server/manage-guice-distributed-james.html)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 976" src="/images/apache-james-architektur.svg?v=20260813" title="Interaktive Infografik: technische Architektur und Nachrichtenfluss von Apache James" loading="lazy">
  <a href="/images/apache-james-architektur.svg?v=20260813">Öppna infografik om den tekniska arkitekturen</a>
</iframe>

### Ett meddelandes bearbetningsväg

1. **Protokollmottagning:** SMTP eller LMTP kontrollerar session, autentisering, kuvertavsändare och mottagare. Efter slutet av `DATA` skapas ett internt `Mail`-objekt med kuvert, MIME-innehåll och attribut.
2. **Kö:** Objektet köas permanent eller flyktigt. Först från denna punkt är mottagning frikopplad från bearbetning.
3. **Spooler:** Workers hämtar köposter och överlämnar dem till Mailet-containern.
4. **Processor:** En namngiven Processor innehåller en ordnad lista av Matcher/Mailet-par. Den obligatoriska `root`-Processorn utgör ingången.
5. **Matcher:** En Matcher förändrar inte meddelandet utan returnerar den delmängd av mottagare för vilka ett villkor gäller.
6. **Mailet:** Tillhörande Mailet förändrar meddelande eller kuvert, utlöser en sidoeffekt, levererar lokalt eller på distans eller förgrenar till en annan Processor.
7. **Resultat:** Meddelandet hamnar i en användarpostlåda, i utgående leverans, i ett Mail Repository för senare hantering eller avslutas efter en lyckad åtgärd.

En viktig detalj är den **mottagarrelaterade uppdelningen**. Om en Matcher bara matchar en del av adresserna delar containern upp bearbetningen i matchande och icke-matchande mottagaruppsättningar. Regler gäller därför inte nödvändigtvis för ett helt MIME-meddelande. Ett Mailet kan dessutom hoppa direkt till en annan Processor via `ToProcessor`; pipelinen är därmed snarare en riktad bearbetningsgraf än en enda linjär lista. Den officiella [Mailet Container-dokumentationen](https://james.apache.org/server/feature-mailetcontainer.html) beskriver just denna modell.

Ett minimalt, förenklat mönster ser ut så här:

```xml
<processor state="root" enableJmx="true">
  <mailet match="RelayLimit=30" class="ToRepository">
    <repositoryPath>cassandra://var/mail/relay-denied/</repositoryPath>
  </mailet>
  <mailet match="RecipientIsLocal" class="LocalDelivery" />
  <mailet match="All" class="RemoteDelivery" />
</processor>
```

Ordningen är en del av semantiken. En regel med bred matchning i början kan göra efterföljande regler ouppnåeliga; en oändlig slinga mellan Processors kan binda Spoolern. James erbjuder därför definierbart felbeteende per Matcher och Mailet samt egna fel-Processors ([Mailet Container Configuration](https://james.apache.org/server/config-mailetcontainer.html)).

Komponentarkitekturen blir konkret så snart ett meddelande når kön. Då avgör Processor, Matcher och Mailet vilka bearbetningssteg som följer och vart resultatet hamnar.

## Teknisk uppbyggnad

Arkitekturen beskriver meddelandevägen; för installation och drift måste den nu omsättas i en konkret komponentbild. Avgörande är vilken körtid, lagring och vilka tilläggstjänster den valda James-profilen faktiskt kräver.

### Teknikstack och administrationsöversikt

För en första produktklassificering är driftsgränser viktigare än klassnamn. Följande översikt koncentrerar stacken till de frågor som bör klarläggas före installation, integration eller övertagande av en befintlig miljö:

| Område | Teknik eller artefakt | Vad administratören måste veta |
|---|---|---|
| Körtid | Java 21, JVM; källkod huvudsakligen Java, enskilda Scala-moduler | Heap, garbage collection, trådhantering och JVM-patchar hör till serverdriften |
| Build och paket | Maven-multimodulprojekt; ZIP-filer och Docker-images | egna Mailets måste passa James-, Java- och Jakarta-generationen |
| Wiring | Guice i 3.9-generationen; Spring i äldre installationer | den valda distributionen avgör tillgängliga moduler och konfigurationsfiler |
| Konfiguration | `conf/*.xml`, `conf/*.properties`, miljövariabler | särskilt viktigt: `smtpserver.xml`, `mailetcontainer.xml`, `webadmin.properties`, JMAP- och backend-filer |
| Bearbetning | MailQueue, Spooler, Processor, Matcher, Mailet | mottagning, bearbetning och slutleverans är separata tillstånd |
| Data | PostgreSQL/JPA eller Cassandra; valfritt S3, OpenSearch, RabbitMQ | källa, projektion, kö och Blob-innehåll behöver separata återställningsplaner |
| Administration | WebAdmin REST API och `james-cli` | REST är kraftfullare; CLI ingår i varje Wiring-variant |
| Observerbarhet | Health Checks, Dropwizard Metrics, Prometheus, JMX, loggar, Grafana | kö, Mailets, Matchers, protokoll och backends har egna mätvärden |
| Säkerhet | TLS-keystores, SMTP AUTH, JWT för WebAdmin, nätverkssegmentering | WebAdmin utan aktiverad JWT är inte skyddad som standard |

Alla konfigurationsfiler finns enligt projektet i `conf` respektive `conf/META-INF`; vilka som faktiskt gäller beror på Wiring och backend. Värden kan hämtas från miljön med `${env:VARIABLE}` ([Apache James – Configuration](https://james.apache.org/server/config.html)). Det är praktiskt för containrar, men ersätter inte secret management: certifikat, privata nycklar, JWT-nycklar och databaslösenord bör tillhandahållas som monterade secrets eller via orkestreringsplattformen.

### Protokollager

Protocols-projektet erbjuder utbyggbara serverimplementationer för SMTP, LMTP, IMAP, POP3, ManageSieve och JMAP ([James Protocols](https://james.apache.org/server/feature-protocols.html)). Listerners är inte hårdkopplade till en viss lagring. IMAP och JMAP använder Mailbox API; SMTP överlämnar mottagna meddelanden till kön och Mailet-containern. Därmed kan protokoll skalas eller inaktiveras oberoende av backend-topologin.

### Postlåda, Mail Repository och Blob-lagring

James skiljer mellan tre lagringsbegrepp som inte bör blandas ihop i drift:

| Lagring | Innehåll | Synlighet | Typisk återställning |
|---|---|---|---|
| **Mailbox** | mappar, meddelanden, flaggor, UID:er, ACL:er och användarkvoter | IMAP/JMAP/POP3 | återställning eller replikering av Mailbox-backend |
| **Mail Repository** | meddelanden från bearbetningsvägar som `error`, `relay-denied` eller karantän | endast administration | åtgärda orsaken och bearbeta meddelandet igen |
| **Blob Store** | binärt MIME-innehåll respektive stora objekt | refereras indirekt via metadata | konsekvent säkerhetskopia med metadata och referenser |

[Persistensdokumentationen](https://james.apache.org/server/feature-persistence.html) betonar att ett Mail Repository uttryckligen **inte** är användarens postlåda. Denna separation är värdefull för incidenthantering: ett felaktigt meddelande kan isoleras, undersökas och efter en korrigering återföras till pipelinen utan att kringgå postlådemodellen.

### Händelsebuss, sökning och projektioner

Mailbox-operationer skapar händelser, till exempel `MailboxAdded`, `MessageMoveEvent`, `FlagsUpdated` eller kvotändringar. Lyssnare uppdaterar utifrån detta kvoter, sökindex och ytterligare projektioner. I den distribuerade profilen hanterar RabbitMQ kommunikationen, OpenSearch sökningen och Cassandra metadata; binärt innehåll ligger i en S3-kompatibel Object Store. Denna uppdelning möjliggör horisontell skalning, men medför **eventuell konsistens** mellan källa och projektioner. Misslyckade lyssnarhändelser hamnar i en Event Dead Letter och måste övervakas och vid behov levereras på nytt ([Distributed James – Mailbox Event Bus](https://james.apache.org/server/manage-guice-distributed-james.html#Mailbox_Event_Bus)).

### MIME, Sieve och autentisering av avsändare

James-projektet omfattar mer än servern. **Apache Mime4J** parsar MIME-strukturer strömorienterat eller som objektmodell; **jSieve** implementerar filterspråket Sieve; **jSPF** och **jDKIM** tillhandahåller Java-bibliotek för avsändarkontroll respektive DKIM-signering och -verifiering. Dessa moduler är självständiga projekt och kan användas även utanför en komplett James-server ([Apache James – komponenter](https://james.apache.org/documentation.html)).

Vilka av dessa komponenter som körs på en nod eller distribuerat är inte en ren prestandafråga. Valet bestämmer även konsistens, omstart och antalet backends som måste övervakas.

## Driftsvarianter och skalning

För James 3.9.0 dokumenterar Apache flera profiler. Det handlar inte bara om olika installationsprogram, utan om olika modeller för konsistens, skalning och drift. I detta läge betecknas JPA-varianten uttryckligen som **legacy**; utöver den finns en PostgreSQL-distribution och en distribuerad distribution ([Apache James – Downloads](https://james.apache.org/download.cgi)). Punkterna som i grafiken betecknas som **driftshärledning** är rekommendationer härledda från detta och inte ordagranna tillverkaruppgifter.

<iframe class="kb-infographic" style="aspect-ratio: 1280 / 956" src="/images/apache-james-betriebsmodelle.svg?v=20260813" title="Interaktive Infografik: Apache-James-Betriebsmodelle und Technologiestacks" loading="lazy">
  <a href="/images/apache-james-betriebsmodelle.svg?v=20260813">Öppna infografik om jämförelse av driftsmodeller</a>
</iframe>

| Profil | Persistens och tjänster | Lämplig för | Driftsmässig konsekvens |
|---|---|---|---|
| JPA/Guice (legacy) | inbäddad H2-databas eller extern SQL-databas; klassisk enservermodell | labb, migrering av äldre installationer, små speciallösningar | få komponenter, men begränsad strategisk väg och vertikal skalning |
| PostgreSQL | PostgreSQL som kärna; valfritt OpenSearch, RabbitMQ och S3-kompatibel lagring | nya en- eller flernodsinstallationer med relationell grund | backup och HA är välkända; inför tilläggstjänster endast vid behov av skalning |
| Distributed/Guice | Cassandra, RabbitMQ, OpenSearch och S3-kompatibel Object Store | stora, horisontellt skalbara tjänster | flera felområden, projektioner, Dead Letters och mer komplexa konsistenskontroller |
| Memory | flyktiga in-memory-komponenter | test och utveckling | ingen beständig data i produktion |

3.9-versionen lyfter fram den kraftfulla PostgreSQL-implementationen som en viktig nyhet och beskriver den som både standalone-kapabel och skalbar med RabbitMQ, OpenSearch och S3 ([Apache James 3.9.0](https://james.apache.org/james/update/2025/09/25/james-3.9.0.html)). För nya installationer är detta oftast den mest lättbegripliga utgångspunkten: först relationell konsistens och kända säkerhetskopieringsmetoder, därefter ytterligare tjänster endast för konkret uppmätta krav.

## Säkerhetsmodell

James tillhandahåller TLS, SMTP-autentisering, protokollkontroller och kryptografiska Mailets. Detta innebär dock inte automatiskt säker produktionsdrift. Transportkryptering skyddar ett hopp; den ersätter varken end-to-end-kryptering eller bindande mottagarkontroll. [TLS-konfigurationen](https://james.apache.org/server/config-ssl-tls.html) skiljer mellan keystore, aktiverade Cipher Suites, StartTLS och implicit TLS per listener. Ett certifikatbyte måste därför följas upp separat för SMTP, IMAP, POP3 och HTTP.

**WebAdmin** förtjänar särskild uppmärksamhet. REST API:t kan förändra domäner, användare, postlådor, köer, repositories, kvoter och underhållsuppgifter. Enligt [WebAdmin-dokumentationen](https://james.apache.org/server/manage-webadmin.html) är JWT-autentisering inaktiverad som standard; utan ytterligare skydd får API:t därför aldrig vara nåbart från ett okontrollerat nätverk. Hälsoändpunkter och API-dokumentation kan dessutom medvetet ligga utanför autentiseringen.

En minimal härdning för produktion omfattar:

- bind WebAdmin till ett hanteringsnät, aktivera JWT och begränsa åtkomst ytterligare med brandvägg eller reverse proxy;
- förhindra öppna reläer genom explicita regler för relä, autentisering och mottagare;
- kör submission och server-till-server-SMTP på separata listeners med olika policyer;
- ta bort demodomäner, exempelanvändare och standardlösenord från container-images före första externa start;
- hantera privata nycklar utanför containerlagret och övervaka utgångsdatum;
- behandla anpassade Mailets som applikationskod: granska beroenden, kör tester och begränsa körningsrättigheter;
- utforma spam- och skadlig kod-kontroll medvetet. James är en plattform; externa skannrar och ryktestjänster integreras via Mailets eller protokollöverlämningar.

Vid felsökning granskas meddelandevägen åter i samma ordning: listener, kö, Mailet-pipeline, repository, postlåda och utgående leverans.

## Drift och felsökning

För en modulär e-postserver är ”tjänsten körs” ingen tillräcklig statusuppgift. WebAdmin Health Checks skiljer mellan `healthy`, `degraded` och `unhealthy`; i strikt läge leder redan en degraderad komponent till HTTP 503. Beroende på profil kontrolleras bland annat JPA eller Cassandra, OpenSearch, RabbitMQ, Guice-livscykeln, Event Dead Letters och en komplett testleverans ([WebAdmin Health Checks](https://james.apache.org/server/manage-webadmin.html#HealthCheck)).

För diagnostik är en skiktvis metod effektivare än en global logsökning:

1. **Anslutning:** Når klienten rätt listener, och lyckas TLS med förväntat certifikat och värdnamn?
2. **SMTP-transaktion:** Vilken svarskod gavs för `MAIL FROM`, `RCPT TO` och `DATA`? Ett `250` efter `DATA` betyder mottagning, inte nödvändigtvis slutleverans.
3. **Kö:** Växer antalet väntande poster, ökar deras ålder eller upprepas samma fjärrfel?
4. **Mailet-pipeline:** Vilken Processor och vilket Matcher/Mailet-par bearbetade meddelandet? Mail-ID:t fungerar som korrelationsnyckel.
5. **Repository:** Finns meddelandet i `error`, `address-error`, `relay-denied` eller ett eget repository? Åtgärda först orsaken, bearbeta sedan på nytt.
6. **Postlåda och händelser:** Finns meddelandet i den ledande Mailbox-lagringen men saknas i sökindex eller JMAP? Då är lyssnare, Dead Letters och omindexering mer relevanta än SMTP.
7. **Remote Delivery:** Vid utgående leverans ska DNS, rutt, TLS, motpartens kod, återförsöksplan och Bounce-skapande kontrolleras separat.

En kompakt syntetisk kontroll kan koppla samman administrations- och dataplanet:

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für den Health Check">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$headers = @{ Authorization = "Bearer $env:JAMES_ADMIN_JWT" }
Invoke-RestMethod `
  -Uri "https://james-admin.example.net/healthcheck?strict" `
  -Headers $headers</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">curl --fail --silent \
  -H "Authorization: Bearer $JAMES_ADMIN_JWT" \
  "https://james-admin.example.net/healthcheck?strict"</code></pre>
  </div>
</div>

I Windows anropar [`Invoke-RestMethod`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod) REST-slutpunkten; i Linux och Unix utför [`curl`](https://curl.se/docs/manpage.html) samma HTTP-kontroll. Båda kommandona testar här uteslutande den dokumenterade WebAdmin Health Check och ersätter inte en syntetisk SMTP- eller postlådetransaktion.

Dessutom bör minst ködjup och köålder, fel-repositories, Event Dead Letters, OpenSearch-indexeringsfördröjning, backend-latenser, SMTP-svarsklasser, JVM-minne och certifikatens återstående giltighetstid larmövervakas. I den distribuerade varianten är en grön James-process vid störd RabbitMQ eller OpenSearch bara en delvis framgång.

### Verktyg för administratörens arbetsplats

James innehåller en kommandoradsklient för domäner, användare, postlådor, mappningar, kvoter och omindexering; i Guice-containrar finns den som `james-cli` ([James CLI](https://james.apache.org/server/manage-cli.html)). För robust diagnostik bör även några protokollneutrala verktyg finnas på administratörens arbetsplats:

| Verktyg | Användning med James |
|---|---|
| [`swaks`](https://www.jetmore.org/john/code/swaks/) | fullständig SMTP- och submission-transaktion med AUTH, TLS, kuvert och fritt satta headers |
| [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) | kontrollera certifikatkedja, SNI, Cipher och StartTLS på SMTP, IMAP eller POP3 |
| [`curl`](https://curl.se/docs/manpage.html) och [`jq`](https://jqlang.org/manual/) | automatiskt fråga WebAdmin, Health Checks, uppgifter och mätvärden |
| [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) eller [`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) | kontrollera MX, A/AAAA, PTR, SPF, DKIM och DMARC |
| [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) eller [Wireshark](https://www.wireshark.org/docs/wsug_html_chunked/) | skilja mellan handshake, retransmits, anslutningsavbrott och protokolldialoger |
| [Prometheus](https://prometheus.io/docs/introduction/overview/) och [Grafana](https://grafana.com/docs/grafana/latest/) | övervaka kö- och protokollmätvärden, latenspercentiler, Mailet-/Matcher-körtider och backendstatus |
| [JMX](https://docs.oracle.com/en/java/javase/21/management/java-management-extensions-jmx-user-guide.html), [VisualVM](https://visualvm.github.io/documentation.html) och [`jcmd`](https://docs.oracle.com/en/java/javase/21/docs/specs/man/jcmd.html) | undersöka heap, trådar, garbage collection och JVM-interna mätvärden |

Den inbyggda [mätvärdesdokumentationen](https://james.apache.org/server/metrics.html) listar bland annat aktiva SMTP-, IMAP- och LMTP-anslutningar, köposter, skickade och levererade meddelanden, svarstider per protokoll samt körtider för enskilda Mailets och Matchers. Dessa mätvärden är mer utsagekraftiga än en enda process-uptime eftersom de återspeglar ett meddelandes väg genom arkitekturen.

## Teknisk historia

James uppstod inte som en portning av en befintlig Unix-MTA. De äldsta bevarade projektsidorna från **1997/1998** beskriver först en planerad Java-server som ännu inte kunde användas, baserad på gemensamma paket i Java Apache Project. Planen omfattade ett gemensamt protokollgränssnitt, JDBC-lagring och ett **MailServlet**-gränssnitt inspirerat av Servlets; som tekniskt förarbete användes infrastrukturen från Apache-JServ-miljön ([James-1.0-arkiv](https://svn.apache.org/repos/asf/james/server/tags/james_1_0/docs/index.html?p=1400000)). Det senare Mailet API bevarade grundidén med små, deploybara bearbetningskomponenter utan att bli en del av Java Servlet-specifikationen.

| Tidsperiod | Tekniskt utvecklingssteg |
|---|---|
| 1997–1998 | utformning i Java Apache Project: ren Java-server, gemensamma protokoll- och resursgränssnitt, MailServlet-idé |
| februari 2001 | migrering från Java Apache Project till Jakarta-projektet ([Jakarta News 2001](https://jakarta.apache.org/site/news/news-2001.html#20010311.1)) |
| James 1.x/2.x | stabil SMTP-/POP3-server, tidvis NNTP; Mailet-motor, fil- och RDBMS-lagring; komponentcontainer Avalon/Phoenix ([dokumentarkiv](https://james.apache.org/server/archive/document_archive.html)) |
| tidiga 2000-talet | utveckling från Jakarta-underprojekt till självständigt Top-Level-projekt hos Apache Software Foundation ([James 2.1.3 – arkiverad projektsida](https://svn.apache.org/repos/asf/james/server/tags/deprecated/build_2_2_0_RC1/www/index.html?p=1400000)) |
| 2010 | James 3.0 M1 med komplett IMAP-stöd, SMTP/LMTP, omarbetat Mailet API samt Maildir-, JPA- och JCR-lagring ([release-meddelande](https://james.apache.org/james/update/2010/11/05/james-3.0-M1.html)) |
| James 3.x | ersättning av Avalon/Phoenix med Spring och senare strategisk inriktning mot Guice; utbyggnad av IMAP, JMAP, REST-administration och distribuerade backends |
| september 2025 | James 3.9.0: övergång från `javax` till `jakarta`, Java 21 och ny PostgreSQL-implementation ([release-meddelande](https://james.apache.org/james/update/2025/09/25/james-3.9.0.html)) |

Källkoden finns i det officiella repositoryt [apache/james-project](https://github.com/apache/james-project). Den här behandlade 3.9-generationen består huvudsakligen av Java; enskilda moduler använder Scala. Den byggs som ett stort Maven-multimodulprojekt. Den långa utvecklingshistorien förklarar varför flera generationer syns parallellt i dokumentation och installationer: Phoenix- och Spring-begrepp i äldre texter, Guice i 3.x-dokumentationen, JPA som legacy-väg och PostgreSQL- respektive Cassandra-profiler för distribuerade deploymenter.

## Lämplighet och begränsningar

James är särskilt lämplig när e-post är **en del av en applikation** snarare än bara infrastruktur: regelbaserad bearbetning, egna Mailets, öppna protokoll, JMAP, kontrollerbar datahantering eller horisontell skalning utan proprietär serverkärna. De offentliga API:erna gör det möjligt att vidareutveckla transport, postlåda och affärslogik var för sig.

James är mindre lämplig för organisationer som förväntar sig en nyckelfärdig appliance med komplett GUI, förkonfigurerat skydd mot spam och skadlig kod, tillverkar-SLA:er och ett enda backupobjekt. Den modulära friheten skapar integrationsarbete. Särskilt den distribuerade profilen kräver driftserfarenhet av flera datasystem och en tydlig definition av källa, projektion, återuppbyggnad och Recovery Point.

Den avgörande arkitekturfrågan är därför: **Ska e-post drivas som ett konfigurerbart protokollsystem eller som en färdig produkt?** För det första fallet erbjuder James en ovanligt djup och öppen byggsats. För det andra fallet är en mer förkonfigurerad produkt ofta mer ekonomisk.

## Källor

- [Apache Projects – James Committee](https://projects.apache.org/committee.html?james)
- [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0)
- [Microsoft Learn – Invoke-RestMethod](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod)
- [curl – Manpage](https://curl.se/docs/manpage.html)
- [SWAKS – Swiss Army Knife for SMTP](https://www.jetmore.org/john/code/swaks/)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [jq – Manual](https://jqlang.org/manual/)
- [BIND 9 – dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)
- [tcpdump – Manpage](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [Wireshark – User’s Guide](https://www.wireshark.org/docs/wsug_html_chunked/)
- [Prometheus – Overview](https://prometheus.io/docs/introduction/overview/)
- [Grafana – Documentation](https://grafana.com/docs/grafana/latest/)
- [Oracle – JMX User Guide](https://docs.oracle.com/en/java/javase/21/management/java-management-extensions-jmx-user-guide.html)
- [VisualVM – Documentation](https://visualvm.github.io/documentation.html)
- [Oracle – jcmd](https://docs.oracle.com/en/java/javase/21/docs/specs/man/jcmd.html)
- [Apache James – projektöversikt](https://james.apache.org/) – självbeskrivning, JVM, protokoll, moduler och arkitekturmål.
- [Apache James – Software Components](https://james.apache.org/documentation.html) – server-, Mailet-, Mailbox-, Protocols- och delprojekt.
- [Apache James – Protocol Servers](https://james.apache.org/server/feature-protocols.html) – protokolltjänster som stöds.
- [RFC 6409](https://datatracker.ietf.org/doc/html/rfc6409)
- [RFC 8314](https://datatracker.ietf.org/doc/html/rfc8314)
- [IETF: SMTP](https://datatracker.ietf.org/doc/html/rfc5321), [LMTP](https://datatracker.ietf.org/doc/html/rfc2033), [Message Submission](https://datatracker.ietf.org/doc/html/rfc6409), [IMAP4rev2](https://datatracker.ietf.org/doc/html/rfc9051), [POP3](https://datatracker.ietf.org/doc/html/rfc1939), [ManageSieve](https://datatracker.ietf.org/doc/html/rfc5804) och [JMAP Mail](https://datatracker.ietf.org/doc/html/rfc8621) – normativa protokollstandarder.
- [RFC 2033](https://datatracker.ietf.org/doc/html/rfc2033)
- [RFC 9051](https://datatracker.ietf.org/doc/html/rfc9051)
- [RFC 1939](https://datatracker.ietf.org/doc/html/rfc1939)
- [RFC 5804](https://datatracker.ietf.org/doc/html/rfc5804)
- [RFC 8621](https://datatracker.ietf.org/doc/html/rfc8621)
- [IANA Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) – registrerade portar.
- [Mailbox API](https://james.apache.org/mailbox/mailbox-api.html)
- [Apache James – Managing Distributed James](https://james.apache.org/server/manage-guice-distributed-james.html) – Cassandra, S3, OpenSearch, RabbitMQ, händelsebuss och drift.
- [Apache James – Mailet Container](https://james.apache.org/server/feature-mailetcontainer.html) – Matchers, Mailets, Processors, Spooler och mottagaruppdelning.
- [Apache James – Mailet Container Configuration](https://james.apache.org/server/config-mailetcontainer.html) – konfiguration och felhantering för pipelinen.
- [Apache James – Configuration](https://james.apache.org/server/config.html) – konfigurationskatalog, filer och miljövariabler.
- [Apache James – Persistence](https://james.apache.org/server/feature-persistence.html) – avgränsning mellan Mailbox och Mail Repository.
- [Apache James – Downloads](https://james.apache.org/download.cgi) – officiella serverprofiler och nedladdningar.
- [Apache James Server 3.9.0](https://james.apache.org/james/update/2025/09/25/james-3.9.0.html) – Java 21, Jakarta-övergång och PostgreSQL-implementation.
- [Apache James – SSL/TLS Configuration](https://james.apache.org/server/config-ssl-tls.html) – TLS-lägen och listener-konfiguration.
- [Apache James – WebAdmin](https://james.apache.org/server/manage-webadmin.html) – REST-administration, JWT-information och Health Checks.
- [Apache James – Command Line](https://james.apache.org/server/manage-cli.html) – CLI för domäner, användare, postlådor, mappningar, kvoter och omindexering.
- [Apache James – Metrics](https://james.apache.org/server/metrics.html) – Prometheus, JMX och tillgängliga driftmätvärden.
- [James-1.0-arkiv från Java Apache Project](https://svn.apache.org/repos/asf/james/server/tags/james_1_0/docs/index.html?p=1400000) – tidig arkitektur- och MailServlet-planering.
- [Jakarta Project News 2001](https://jakarta.apache.org/site/news/news-2001.html) – migrering av James-projektet till Jakarta.
- [Apache James Document Archive](https://james.apache.org/server/archive/document_archive.html) – dokumentation för versionerna 1.x och 2.x.
- [James 2.1.3 – arkiverad projektsida](https://svn.apache.org/repos/asf/james/server/tags/deprecated/build_2_2_0_RC1/www/index.html?p=1400000)
- [Apache James 3.0 M1](https://james.apache.org/james/update/2010/11/05/james-3.0-M1.html) – IMAP, lagringsprofiler och Mailet API för 3.x-generationen.
- [Apache James – GitHub-repository](https://github.com/apache/james-project) – källkod, build och modulstruktur.
