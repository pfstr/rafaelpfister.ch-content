---
title: "Apache James: modulær e-postserver og Mailet-plattform"
blatt: "apache-james"
description: "Apache James i teknisk kontekst: protokoller og e-postroller, komponentbasert arkitektur, kø og Mailet-pipeline, postboks- og lagringsmodell, driftsvarianter fra PostgreSQL til Cassandra samt utviklingen fra Java Apache-prosjektet til JVM-e-postplattformen."
fakten:
  - label: Fullt navn
    wert: Java Apache Mail Enterprise Server
    href: https://james.apache.org/
  - label: Kategori
    wert: MTA, MDA, postboksserver og e-postapplikasjonsplattform
    href: https://james.apache.org/documentation.html
  - label: Prosjekt
    wert: Apache Software Foundation
    href: https://projects.apache.org/committee.html?james
  - label: Kjøretid
    wert: JVM · Java 21 fra versjon 3.9
    href: https://james.apache.org/james/update/2025/09/25/james-3.9.0.html
  - label: Språk
    wert: hovedsakelig Java, enkelte moduler i Scala
    href: https://github.com/apache/james-project
  - label: Protokoller
    wert: SMTP, LMTP, IMAP, POP3, ManageSieve, JMAP
    href: https://james.apache.org/server/feature-protocols.html
  - label: Arkitekturstil
    wert: modulær, komponentbasert, Inversion of Control, hendelsesdrevet
    href: https://james.apache.org/
  - label: Bakender
    wert: PostgreSQL/JPA eller Cassandra · OpenSearch · RabbitMQ · S3
    href: https://james.apache.org/download.cgi
  - label: Konfigurasjon
    wert: conf/*.xml og *.properties · miljøvariabler
    href: https://james.apache.org/server/config.html
  - label: Pakking
    wert: ZIP-distribusjoner og offisielle Docker-images
    href: https://james.apache.org/download.cgi
  - label: Administrasjon
    wert: WebAdmin REST API, CLI og måledata
    href: https://james.apache.org/server/manage-webadmin.html
  - label: Overvåking
    wert: Health Checks, Prometheus, JMX, logger og Grafana
    href: https://james.apache.org/server/metrics.html
  - label: Lisens
    wert: Apache License 2.0
    href: https://www.apache.org/licenses/LICENSE-2.0
werbung:
  - newsletter
ctaThemen:
  - smtp-mailflow
translationSourceHash: adbfb005b83b16086ba55e53dd469f3aff1e5642364da5ab8b2da5d265a1ce51
translationModel: gpt-5.6-terra
translatedAt: 2026-10-03T09:26:54.697Z
translationReview: required
---

# Apache James: modulær e-postserver og Mailet-plattform

Apache James er en åpen kildekode-e-postserver og samtidig et byggesett for applikasjoner der forretningslogikken bygger på e-post. Navnet står for **Java Apache Mail Enterprise Server**. James kan ta imot og videresende meldinger via SMTP, administrere lokale postbokser, gjøre dem tilgjengelige via IMAP, POP3 eller JMAP og styre hele meldingsflyten gjennom fritt kombinerbare behandlingskomponenter. Prosjektet beskriver seg derfor ikke bare som en server, men som en modulært sammensettbar **Inversion-of-Control-plattform på JVM** ([Apache James – prosjektoversikt](https://james.apache.org/)).

Denne dobbeltrollen skiller James fra klassiske Mail Transfer Agents som Postfix og fra ferdige sikkerhetsappliances. En administrator kan bruke James som et rent SMTP-relé, som en komplett postboksserver eller som en innebygd e-postmotor i et produkt. Spamkontroll, kryptering, arkivering eller domenespesifikk ruting oppstår ikke fra en fast funksjonsblokk, men fra en pipeline av **Matchers** og **Mailets**. Dette gjør James svært tilpasningsdyktig, men flytter deler av produktansvaret fra produsenten til organisasjonen som drifter løsningen.

Forklaringen følger en melding gjennom James: fra protokollserverne via køen og Mailet-pipelinen til postbokslagringen. Deretter behandles driftsvariantene, diagnostikk og til slutt prosjektets tekniske utvikling.

## Innplassering: MTA, MDA og applikasjonsplattform

I e-postsystemet har ikke alle komponenter samme rolle. En **Mail User Agent** (MUA) er brukerens klient, for eksempel Thunderbird. En **Mail Transfer Agent** (MTA) transporterer meldinger mellom systemer. En **Mail Delivery Agent** (MDA) legger en melding i målpostboksen. James kan være MTA og MDA samtidig; gjennom protokoll- og postboksmodulene tilbyr den i tillegg tjenester på serversiden for MUA-er. Den offisielle komponentoversikten angir separate prosjekter for server, protokoller, Mailets, postbokser og tester ([Apache James – Software Components](https://james.apache.org/documentation.html)).

| Rolle | Implementasjon i James | Overleveringspunkt |
|---|---|---|
| Meldingstransport | SMTP- og LMTP-server, kø, Remote-Delivery-Mailet | andre MTA-er, reléer og gatewayer |
| Lokal levering | Mailet-pipeline og Mailbox API | brukere, domener og kvoter |
| Postbokstilgang | IMAP, POP3 og JMAP | e-postklienter og webapplikasjoner |
| Filterlogikk | Matchers, Mailets, Processors og Sieve | interne regler og eksterne kontrolltjenester |
| Administrasjon | WebAdmin REST API, CLI, Health Checks og måledata | automatisering og overvåking |

James er dermed **ingen e-postklient** og heller ingen forhåndskonfigurert sikker e-postgateway. Den leverer byggeklosser for transport, levering, lagring og behandling. Om resultatet blir et enkelt relé, en flerleietakertjeneste for e-post eller en produktspesifikk gateway, avgjøres av valgt distribusjon og konfigurasjon.

## Protokoller, TLS og porter

James tilbyr SMTP, LMTP, IMAP, POP3 og ManageSieve som TCP-baserte tjenester; JMAP og WebAdmin bruker HTTP ([Apache James – Protocol Servers](https://james.apache.org/server/feature-protocols.html)). TLS beskytter, avhengig av lytter, enten en forbindelse som er kryptert fra starten, eller legges inn i en eksisterende økt via StartTLS. DNS er ikke en del av James-prosessen, men er uunnværlig for en offentlig MTA: MX-poster bestemmer målet, A- og AAAA-poster adressene og PTR-poster påvirker omdømmet til utgående forbindelser.

Portnummeret alene beskriver ikke sikkerhetssemantikken. Port 25 er beregnet på server-til-server-transport; autentisert innsending fra klienter hører ifølge [RFC 6409](https://datatracker.ietf.org/doc/html/rfc6409) hjemme på port 587. Port 465 er siden [RFC 8314](https://datatracker.ietf.org/doc/html/rfc8314) igjen registrert for implisitt kryptert Message Submission. For IMAP og POP3 gjelder de samme to mønstrene: klartekstforbindelse med mulig StartTLS eller umiddelbar TLS-etablering.

| Tjeneste | Typiske porter | Standard | Betydning i James |
|---|---:|---|---|
| SMTP | 25, 587, 465 | [RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321) | Mottak, relé og innsending |
| LMTP | konfigurerbar, registrert 24 | [RFC 2033](https://datatracker.ietf.org/doc/html/rfc2033) | lokal overlevering med status per mottaker |
| IMAP4rev2 | 143, 993 | [RFC 9051](https://datatracker.ietf.org/doc/html/rfc9051) | synkron postbokstilgang |
| POP3 | 110, 995 | [RFC 1939](https://datatracker.ietf.org/doc/html/rfc1939) | enkel meldingshenting |
| ManageSieve | 4190 | [RFC 5804](https://datatracker.ietf.org/doc/html/rfc5804) | administrasjon av brukerspesifikke Sieve-regler |
| JMAP Mail | vanligvis 443 | [RFC 8621](https://datatracker.ietf.org/doc/html/rfc8621) | HTTP-basert postbokstilgang for moderne klienter |

Portene kan konfigureres; det bindende er kombinasjonen av lytter, protokoll, TLS-modus og autentisering. [IANA Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) er fortsatt referansen for registrerte tilordninger.

## Arkitekturtilnærming

James følger en **komponentbasert arkitektur**. Protokollservere, kø, behandlingslogikk, postboks, brukeradministrasjon, søkeindeks og administrasjon er adskilt gjennom API-er og settes sammen med dependency injection. Distribusjonene dokumentert for James 3.9 bruker Google Guice til dette; Spring-oppsettet tilhører en eldre generasjon. Frikoblingen er ikke bare kodeorganisering: Den gjør det mulig å bruke den samme [Mailbox API](https://james.apache.org/mailbox/mailbox-api.html) med ulike persistenslag og den samme Mailet-logikken i svært ulike serverprofiler.

Den sentrale databanen er asynkron. En SMTP-lytter trenger ikke å levere en mottatt melding fullstendig før den svarer på forbindelsen. Den legger et e-postobjekt i en kø; en **Spooler** henter det senere og fører det gjennom Mailet-containeren. Køen skiller dermed mottaksbelastning, behandlingstid og tilgjengeligheten til etterfølgende systemer. Den distribuerte driftsdokumentasjonen beskriver den derfor som en obligatorisk del av en SMTP-server ([Apache James – Distributed Server Operations](https://james.apache.org/server/manage-guice-distributed-james.html)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 976" src="/images/apache-james-architektur.svg?v=20260813" title="Interaktive Infografik: technische Architektur und Nachrichtenfluss von Apache James" loading="lazy">
  <a href="/images/apache-james-architektur.svg?v=20260813">Åpne infografikk om den tekniske arkitekturen</a>
</iframe>

### Behandlingsveien til en melding

1. **Protokollmottak:** SMTP eller LMTP kontrollerer økt, autentisering, konvoluttavsender og mottaker. Etter slutten på `DATA` opprettes et internt `Mail`-objekt med konvolutt, MIME-innhold og attributter.
2. **Kø:** Objektet settes i en permanent eller flyktig kø. Først fra dette punktet er mottak frikoblet fra behandling.
3. **Spooler:** Arbeidere henter køoppføringer og overleverer dem til Mailet-containeren.
4. **Processor:** En navngitt Processor inneholder en ordnet liste med Matcher/Mailet-par. Den obligatoriske `root`-Processoren utgjør inngangen.
5. **Matcher:** En Matcher endrer ikke meldingen, men returnerer delmengden av mottakere som oppfyller en betingelse.
6. **Mailet:** Den tilhørende Maileten endrer melding eller konvolutt, utløser en bieffekt, leverer lokalt eller eksternt, eller forgrener til en annen Processor.
7. **Resultat:** Meldingen havner i en brukerpostboks, i utgående levering, i et Mail Repository for senere behandling, eller er fullført etter vellykket handling.

En viktig detalj er den **mottakerrelaterte splittingen**. Dersom en Matcher bare passer for noen av adressatene, deler containeren behandlingen i passende og ikke-passende mottakergrupper. Regler gjelder derfor ikke nødvendigvis for en komplett MIME-melding. En Mailet kan dessuten hoppe direkte til en annen Processor via `ToProcessor`; pipelinen er dermed mer en rettet behandlingsgraf enn én enkelt lineær liste. Den offisielle [dokumentasjonen for Mailet-containeren](https://james.apache.org/server/feature-mailetcontainer.html) beskriver nettopp denne modellen.

Et minimalt, forenklet mønster ser slik ut:

```xml
<processor state="root" enableJmx="true">
  <mailet match="RelayLimit=30" class="ToRepository">
    <repositoryPath>cassandra://var/mail/relay-denied/</repositoryPath>
  </mailet>
  <mailet match="RecipientIsLocal" class="LocalDelivery" />
  <mailet match="All" class="RemoteDelivery" />
</processor>
```

Rekkefølgen er en del av semantikken. En regel som passer bredt i starten, kan gjøre etterfølgende regler uoppnåelige; en endeløs løkke mellom Processors kan binde Spooleren. James tilbyr derfor definerbar feilhåndtering per Matcher og Mailet samt egne feil-Processors ([Mailet Container Configuration](https://james.apache.org/server/config-mailetcontainer.html)).

Komponentarkitekturen blir konkret så snart en melding når køen. Da avgjør Processor, Matcher og Mailet hvilke behandlingstrinn som følger, og hvor resultatet havner.

## Teknisk oppbygning

Arkitekturen beskriver meldingsveien; for installasjon og drift må den nå omsettes til et konkret komponentbilde. Avgjørende er hvilken kjøretid, lagring og tilleggstjenester den valgte James-profilen faktisk krever.

### Teknologistack og administrasjonsoversikt

For den første produktvurderingen er driftsgrensene viktigere enn klassenavn. Følgende oversikt kondenserer stakken til spørsmålene som bør avklares før installasjon, integrasjon eller overtakelse av et eksisterende miljø:

| Område | Teknologi eller artefakt | Det administratoren må vite |
|---|---|---|
| Kjøretid | Java 21, JVM; kildekode hovedsakelig Java, enkelte Scala-moduler | Heap, Garbage Collection, trådhåndtering og JVM-patcher er del av serverdriften |
| Bygg og pakke | Maven-multimodulprosjekt; ZIP-filer og Docker-images | egne Mailets må passe til James-, Java- og Jakarta-generasjonen |
| Kobling | Guice i 3.9-generasjonen; Spring i eldre installasjoner | valgt distribusjon avgjør tilgjengelige moduler og konfigurasjonsfiler |
| Konfigurasjon | `conf/*.xml`, `conf/*.properties`, miljøvariabler | særlig viktig: `smtpserver.xml`, `mailetcontainer.xml`, `webadmin.properties`, JMAP- og backend-filer |
| Behandling | MailQueue, Spooler, Processor, Matcher, Mailet | mottak, behandling og endelig levering er separate tilstander |
| Data | PostgreSQL/JPA eller Cassandra; valgfritt S3, OpenSearch, RabbitMQ | kilde, projeksjon, kø og Blob-innhold krever separate gjenopprettingsplaner |
| Administrasjon | WebAdmin REST API og `james-cli` | REST er kraftigere; CLI følger med i alle koblingsvarianter |
| Observerbarhet | Health Checks, Dropwizard Metrics, Prometheus, JMX, logger, Grafana | kø, Mailets, Matchers, protokoller og bakender har egne måledata |
| Sikkerhet | TLS-keystores, SMTP AUTH, JWT for WebAdmin, nettverkssegmentering | WebAdmin uten aktivert JWT er ikke beskyttet som standard |

Alle konfigurasjonsfiler ligger ifølge prosjektet i `conf` eller `conf/META-INF`; hvilke som faktisk gjelder, avhenger av kobling og backend. Verdier kan hentes fra miljøet med `${env:VARIABLE}` ([Apache James – Configuration](https://james.apache.org/server/config.html)). Dette er praktisk for containere, men erstatter ikke secrets-håndtering: Sertifikater, private nøkler, JWT-nøkler og databasepassord bør leveres som monterte secrets eller gjennom orkestreringsplattformen.

### Protokollag

Protocols-prosjektet tilbyr utvidbare serverimplementasjoner for SMTP, LMTP, IMAP, POP3, ManageSieve og JMAP ([James Protocols](https://james.apache.org/server/feature-protocols.html)). Lytterne er ikke fast koblet til en bestemt lagring. IMAP og JMAP bruker Mailbox API; SMTP overleverer mottatte meldinger til kø og Mailet-container. Dermed kan protokoller skaleres eller deaktiveres uavhengig av backend-topologien.

### Postboks, Mail Repository og Blob-lagring

James skiller mellom tre lagringsbegreper som ikke bør blandes i drift:

| Lagring | Innhold | Synlighet | Typisk gjenoppretting |
|---|---|---|---|
| **Mailbox** | mapper, meldinger, flagg, UID-er, ACL-er og kvoter for en bruker | IMAP/JMAP/POP3 | gjenoppretting eller replikering av postboks-backenden |
| **Mail Repository** | meldinger fra behandlingsveier som `error`, `relay-denied` eller karantene | kun administrasjon | rett årsaken og behandle meldingen på nytt |
| **Blob Store** | binært MIME-innhold eller store objekter | indirekte referert via metadata | konsistent sikkerhetskopi med metadata og referanser |

[PERSISTENSDOKUMENTASJONEN](https://james.apache.org/server/feature-persistence.html) understreker at et Mail Repository nettopp **ikke** er brukerens postboks. Dette skillet er verdifullt for Incident Response: En feilaktig melding kan isoleres, undersøkes og etter en korrigering føres tilbake i pipelinen uten å omgå postboksmodellen.

### Hendelsesbuss, søk og projeksjoner

Postboksoperasjoner genererer hendelser, for eksempel `MailboxAdded`, `MessageMoveEvent`, `FlagsUpdated` eller kvoteendringer. Lyttere oppdaterer kvoter, søkeindekser og andre projeksjoner fra disse. I den distribuerte profilen håndterer RabbitMQ kommunikasjonen, OpenSearch søket og Cassandra metadataene; binært innhold ligger i en S3-kompatibel Object Store. Denne oppdelingen muliggjør horisontal skalering, men medfører **eventual consistency** mellom kilde og projeksjoner. Mislykkede lytterhendelser havner i en Event Dead Letter og må overvåkes og eventuelt leveres på nytt ([Distributed James – Mailbox Event Bus](https://james.apache.org/server/manage-guice-distributed-james.html#Mailbox_Event_Bus)).

### MIME, Sieve og avsenderautentisering

James-prosjektet omfatter mer enn serveren. **Apache Mime4J** analyserer MIME-strukturer strømorientert eller som objektmodell; **jSieve** implementerer Sieve-filterspråket; **jSPF** og **jDKIM** tilbyr Java-biblioteker for henholdsvis avsenderkontroll og DKIM-signering og -verifisering. Disse modulene er selvstendige prosjekter og kan også brukes utenfor en komplett James-server ([Apache James – komponenter](https://james.apache.org/documentation.html)).

Hvilke av disse komponentene som kjører på én node eller distribuert, er ikke bare et ytelsesspørsmål. Valget fastsetter også konsistens, gjenoppstart og antall bakender som må overvåkes.

## Driftsvarianter og skalering

For James 3.9.0 dokumenterer Apache flere profiler. Dette er ikke bare ulike installasjonsprogrammer, men forskjellige konsistens-, skalerings- og driftsmodeller. I denne versjonen er JPA-varianten uttrykkelig betegnet som **legacy**; ved siden av den finnes en PostgreSQL-distribusjon og en distribuert distribusjon ([Apache James – Downloads](https://james.apache.org/download.cgi)). Punktene som i grafikken betegnes som **driftsavledning**, er anbefalinger utledet av dette og ikke ordrette produsentutsagn.

<iframe class="kb-infographic" style="aspect-ratio: 1280 / 956" src="/images/apache-james-betriebsmodelle.svg?v=20260813" title="Interaktive Infografik: Apache-James-Betriebsmodelle und Technologiestacks" loading="lazy">
  <a href="/images/apache-james-betriebsmodelle.svg?v=20260813">Åpne infografikk som sammenligner driftsmodellene</a>
</iframe>

| Profil | Persistens og tjenester | Egnet for | Driftsmessig konsekvens |
|---|---|---|---|
| JPA/Guice (legacy) | innebygd H2-database eller ekstern SQL-database; klassisk enkeltservermodell | laboratorium, migrering av eldre installasjoner, små spesialløsninger | få komponenter, men begrenset strategisk vei og vertikal skalering |
| PostgreSQL | PostgreSQL som kjerne; valgfritt OpenSearch, RabbitMQ og S3-kompatibel lagring | nye enkelt- eller flernodeinstallasjoner med relasjonell base | backup og HA er velkjent; innfør tilleggstjenester bare ved nødvendig skalering |
| Distributed/Guice | Cassandra, RabbitMQ, OpenSearch og S3-kompatibel Object Store | store, horisontalt skalerbare tjenester | flere feilområder, projeksjoner, Dead Letters og mer krevende konsistenskontroller |
| Memory | flyktige in-memory-komponenter | tester og utvikling | ingen bevaring av produksjonsdata |

3.9-utgivelsen fremhever den kraftige PostgreSQL-implementasjonen som en vesentlig nyhet og beskriver den som både standalone-kapabel og skalerbar med RabbitMQ, OpenSearch og S3 ([Apache James 3.9.0](https://james.apache.org/james/update/2025/09/25/james-3.9.0.html)). For nye installasjoner er dette vanligvis det mest forståelige utgangspunktet: først relasjonell konsistens og kjente sikkerhetskopieringsmetoder, deretter tilleggstjenester kun for konkrete, målte behov.

## Sikkerhetsmodell

James tilbyr TLS, SMTP-autentisering, protokollkontroller og kryptografiske Mailets. Dette innebærer likevel ikke automatisk sikker produksjonsdrift. Transportkryptering beskytter ett hopp; den erstatter verken ende-til-ende-kryptering eller bindende mottakerkontroll. [TLS-konfigurasjonen](https://james.apache.org/server/config-ssl-tls.html) skiller mellom keystore, aktiverte Cipher Suites, StartTLS og implisitt TLS per lytter. Et sertifikatbytte må derfor spores separat for SMTP, IMAP, POP3 og HTTP.

**WebAdmin** fortjener særlig oppmerksomhet. REST API-et kan endre domener, brukere, postbokser, køer, repositories, kvoter og vedlikeholdsoppgaver. Ifølge [WebAdmin-dokumentasjonen](https://james.apache.org/server/manage-webadmin.html) er JWT-autentisering deaktivert som standard; uten ytterligere sikring må API-et derfor aldri være tilgjengelig fra et ukontrollert nettverk. Helseendepunkter og API-dokumentasjon kan dessuten bevisst ligge utenfor autentiseringen.

En minimal produksjonsherding omfatter:

- bind WebAdmin til et administrasjonsnettverk, aktiver JWT og begrens tilgangen ytterligere med brannmur eller reverse proxy;
- forhindre åpne reléer med eksplisitte regler for relé, autentisering og mottakere;
- drift innsending og server-til-server-SMTP på separate lyttere med ulike policyer;
- fjern demodomener, eksempelbrukere og standardpassord fra container-images før første eksterne oppstart;
- administrer private nøkler utenfor containerlaget og overvåk utløpsdatoer;
- behandle egendefinerte Mailets som applikasjonskode: kontroller avhengigheter, kjør tester og begrens kjøretidsrettigheter;
- utform spam- og malwarekontroll bevisst. James er en plattform; eksterne skannere og omdømmetjenester integreres via Mailets eller protokolloverleveringer.

Ved feilsøking kontrolleres meldingsveien igjen i samme rekkefølge: lytter, kø, Mailet-pipeline, repository, postboks og utgående levering.

## Drift og feilsøking

For en modulær e-postserver er «tjenesten kjører» ikke en tilstrekkelig statusbeskrivelse. WebAdmin-Health-Checks skiller mellom `healthy`, `degraded` og `unhealthy`; i streng modus fører allerede en degradert komponent til HTTP 503. Avhengig av profil kontrolleres blant annet JPA eller Cassandra, OpenSearch, RabbitMQ, Guice-livssyklusen, Event Dead Letters og en fullstendig testlevering ([WebAdmin Health Checks](https://james.apache.org/server/manage-webadmin.html#HealthCheck)).

For diagnostikk er en lagvis fremgangsmåte mer effektiv enn et globalt logsøk:

1. **Forbindelse:** Når klienten riktig lytter, og lykkes TLS med forventet sertifikat og vertsnavn?
2. **SMTP-transaksjon:** Hvilken svarkode ble levert for `MAIL FROM`, `RCPT TO` og `DATA`? En `250` etter `DATA` betyr mottak, ikke nødvendigvis endelig levering.
3. **Kø:** Vokser antallet ventende oppføringer, øker alderen deres, eller gjentas den samme eksterne feilen?
4. **Mailet-pipeline:** Hvilken Processor og hvilket Matcher/Mailet-par behandlet meldingen? Mail-ID-en fungerer som korrelasjonsnøkkel.
5. **Repository:** Ligger meldingen i `error`, `address-error`, `relay-denied` eller i et eget repository? Rett først årsaken, og behandle deretter på nytt.
6. **Postboks og hendelser:** Finnes meldingen i den ledende postboksbutikken, men mangler i søkeindeksen eller JMAP? Da er lyttere, Dead Letters og reindeksering mer relevante enn SMTP.
7. **Remote Delivery:** Ved utgående levering må DNS, rute, TLS, motpartskode, retry-plan og bounce-generering kontrolleres separat.

En kompakt syntetisk kontroll kan koble sammen administrasjons- og dataplanet:

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

I Windows kaller [`Invoke-RestMethod`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod) REST-endepunktet; i Linux og Unix utfører [`curl`](https://curl.se/docs/manpage.html) den samme HTTP-kontrollen. Begge kommandoene tester her utelukkende den dokumenterte WebAdmin-Health-Check og erstatter ingen syntetisk SMTP- eller postbokstransaksjon.

I tillegg bør minst kødybde og -alder, feil-repositories, Event Dead Letters, forsinkelse i OpenSearch-indeksering, backend-latenser, SMTP-svarklasser, JVM-minne og sertifikatgyldighet alarmovervåkes. I den distribuerte varianten er en grønn James-prosess ved forstyrret RabbitMQ eller OpenSearch bare en delvis suksess.

### Verktøy for administratorarbeidsplassen

James har en kommandolinjeklient for domener, brukere, postbokser, mappings, kvoter og reindeksering; i Guice-containere er den tilgjengelig som `james-cli` ([James CLI](https://james.apache.org/server/manage-cli.html)). For robust diagnostikk bør administratorarbeidsplassen dessuten ha noen protokollnøytrale verktøy:

| Verktøy | Bruk med James |
|---|---|
| [`swaks`](https://www.jetmore.org/john/code/swaks/) | fullstendig SMTP- og innsendingstransaksjon med AUTH, TLS, konvolutt og fritt angitte headere |
| [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) | kontroller sertifikatkjede, SNI, Cipher og StartTLS for SMTP, IMAP eller POP3 |
| [`curl`](https://curl.se/docs/manpage.html) og [`jq`](https://jqlang.org/manual/) | spør WebAdmin, Health Checks, oppgaver og måledata automatisert |
| [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) eller [`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) | kontroller MX, A/AAAA, PTR, SPF, DKIM og DMARC |
| [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) eller [Wireshark](https://www.wireshark.org/docs/wsug_html_chunked/) | skill mellom handshake, retransmits, forbindelsesbrudd og protokolldialoger |
| [Prometheus](https://prometheus.io/docs/introduction/overview/) og [Grafana](https://grafana.com/docs/grafana/latest/) | observer kø- og protokollmåledata, latenspersentiler, Mailet-/Matcher-kjøretider og backendtilstander |
| [JMX](https://docs.oracle.com/en/java/javase/21/management/java-management-extensions-jmx-user-guide.html), [VisualVM](https://visualvm.github.io/documentation.html) og [`jcmd`](https://docs.oracle.com/en/java/javase/21/docs/specs/man/jcmd.html) | undersøk heap, tråder, Garbage Collection og JVM-interne måledata |

Den innebygde [måledokumentasjonen](https://james.apache.org/server/metrics.html) lister blant annet aktive SMTP-, IMAP- og LMTP-forbindelser, køoppføringer, sendte og leverte meldinger, svartider per protokoll samt kjøretider for enkeltstående Mailets og Matchers. Disse måledataene er mer utsagnskraftige enn en enkelt prosessoppetid fordi de avbilder meldingsveien gjennom arkitekturen.

## Teknisk historie

James oppstod ikke som en port av en eksisterende Unix-MTA. De eldste bevarte prosjektsidene fra **1997/1998** beskriver først en planlagt Java-server som ennå ikke kunne brukes, basert på felles pakker fra Java Apache Project. Planen omfattet et felles protokollgrensesnitt, JDBC-lagring og et **MailServlet**-grensesnitt inspirert av Servlets; infrastrukturen fra Apache-JServ-miljøet fungerte som teknisk forarbeid ([James-1.0-arkiv](https://svn.apache.org/repos/asf/james/server/tags/james_1_0/docs/index.html?p=1400000)). Den senere Mailet API-en beholdt grunntanken om små, distribuerbare behandlingskomponenter uten å bli en del av Java Servlet-spesifikasjonen.

| Tidsrom | Teknisk utviklingstrinn |
|---|---|
| 1997–1998 | utforming i Java Apache Project: ren Java-server, felles protokoll- og ressursgrensesnitt, MailServlet-idé |
| Februar 2001 | migrering fra Java Apache Project til Jakarta-prosjektet ([Jakarta News 2001](https://jakarta.apache.org/site/news/news-2001.html#20010311.1)) |
| James 1.x/2.x | stabil SMTP-/POP3-server, periodevis NNTP; Mailet-motor, fil- og RDBMS-lagring; komponentcontainere Avalon/Phoenix ([dokumentarkiv](https://james.apache.org/server/archive/document_archive.html)) |
| tidlig 2000-tall | opprykk fra Jakarta-underprosjekt til selvstendig Top-Level Project i Apache Software Foundation ([James 2.1.3 – arkivert prosjektside](https://svn.apache.org/repos/asf/james/server/tags/deprecated/build_2_2_0_RC1/www/index.html?p=1400000)) |
| 2010 | James 3.0 M1 med full IMAP-støtte, SMTP/LMTP, revidert Mailet API samt Maildir-, JPA- og JCR-lagring ([utgivelsesmelding](https://james.apache.org/james/update/2010/11/05/james-3.0-M1.html)) |
| James 3.x | erstatning av Avalon/Phoenix med Spring og senere strategisk orientering mot Guice; utbygging av IMAP, JMAP, REST-administrasjon og distribuerte bakender |
| September 2025 | James 3.9.0: overgang fra `javax` til `jakarta`, Java 21 og ny PostgreSQL-implementasjon ([utgivelsesmelding](https://james.apache.org/james/update/2025/09/25/james-3.9.0.html)) |

Kildekoden ligger i det offisielle repositoriet [apache/james-project](https://github.com/apache/james-project). Den her omtalte 3.9-generasjonen består hovedsakelig av Java; enkelte moduler bruker Scala. Den bygges som et stort Maven-multimodulprosjekt. Den lange utviklingshistorien forklarer hvorfor flere generasjoner er synlige side om side i dokumentasjon og installasjoner: Phoenix- og Spring-begreper i eldre tekster, Guice i 3.x-dokumentasjonen, JPA som legacy-vei og PostgreSQL- eller Cassandra-profiler for distribuerte utrullinger.

## Egnethet og begrensninger

James er særlig egnet når e-post er **en del av en applikasjon** og ikke bare infrastruktur: regelbasert behandling, egne Mailets, åpne protokoller, JMAP, kontrollerbar datahåndtering eller horisontal skalering uten en proprietær serverkjerne. De offentlige API-ene gjør det mulig å videreutvikle transport, postboks og forretningslogikk separat.

James er mindre egnet for organisasjoner som forventer en nøkkelferdig appliance med komplett GUI, forhåndskonfigurert spam- og malwarebeskyttelse, produsent-SLA-er og ett enkelt backup-objekt. Den modulære friheten skaper integrasjonsarbeid. Særlig den distribuerte profilen krever driftserfaring med flere datasystemer og en klar definisjon av kilde, projeksjon, gjenoppbygging og Recovery Point.

Det avgjørende arkitekturspørsmålet er derfor: **Skal e-post driftes som et konfigurerbart protokollsystem eller som et ferdig produkt?** I det første tilfellet tilbyr James et uvanlig dypt, åpent byggesett. I det andre tilfellet er et mer forhåndskonfigurert produkt ofte mer økonomisk.

## Kilder

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
- [Apache James – prosjektoversikt](https://james.apache.org/) – selvbeskrivelse, JVM, protokoller, moduler og arkitekturmål.
- [Apache James – Software Components](https://james.apache.org/documentation.html) – server-, Mailet-, Mailbox-, Protocols- og delprosjekter.
- [Apache James – Protocol Servers](https://james.apache.org/server/feature-protocols.html) – støttede protokolltjenester.
- [RFC 6409](https://datatracker.ietf.org/doc/html/rfc6409)
- [RFC 8314](https://datatracker.ietf.org/doc/html/rfc8314)
- [IETF: SMTP](https://datatracker.ietf.org/doc/html/rfc5321), [LMTP](https://datatracker.ietf.org/doc/html/rfc2033), [Message Submission](https://datatracker.ietf.org/doc/html/rfc6409), [IMAP4rev2](https://datatracker.ietf.org/doc/html/rfc9051), [POP3](https://datatracker.ietf.org/doc/html/rfc1939), [ManageSieve](https://datatracker.ietf.org/doc/html/rfc5804) og [JMAP Mail](https://datatracker.ietf.org/doc/html/rfc8621) – normative protokollstandarder.
- [RFC 2033](https://datatracker.ietf.org/doc/html/rfc2033)
- [RFC 9051](https://datatracker.ietf.org/doc/html/rfc9051)
- [RFC 1939](https://datatracker.ietf.org/doc/html/rfc1939)
- [RFC 5804](https://datatracker.ietf.org/doc/html/rfc5804)
- [RFC 8621](https://datatracker.ietf.org/doc/html/rfc8621)
- [IANA Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) – registrerte porter.
- [Mailbox API](https://james.apache.org/mailbox/mailbox-api.html)
- [Apache James – Managing Distributed James](https://james.apache.org/server/manage-guice-distributed-james.html) – Cassandra, S3, OpenSearch, RabbitMQ, hendelsesbuss og drift.
- [Apache James – Mailet Container](https://james.apache.org/server/feature-mailetcontainer.html) – Matchers, Mailets, Processors, Spooler og mottakersplitting.
- [Apache James – Mailet Container Configuration](https://james.apache.org/server/config-mailetcontainer.html) – konfigurasjon og feilhåndtering i pipelinen.
- [Apache James – Configuration](https://james.apache.org/server/config.html) – konfigurasjonskatalog, filer og miljøvariabler.
- [Apache James – Persistence](https://james.apache.org/server/feature-persistence.html) – avgrensning mellom Mailbox og Mail Repository.
- [Apache James – Downloads](https://james.apache.org/download.cgi) – offisielle serverprofiler og nedlastinger.
- [Apache James Server 3.9.0](https://james.apache.org/james/update/2025/09/25/james-3.9.0.html) – Java 21, Jakarta-overgang og PostgreSQL-implementasjon.
- [Apache James – SSL/TLS Configuration](https://james.apache.org/server/config-ssl-tls.html) – TLS-moduser og lytterkonfigurasjon.
- [Apache James – WebAdmin](https://james.apache.org/server/manage-webadmin.html) – REST-administrasjon, JWT-merknad og Health Checks.
- [Apache James – Command Line](https://james.apache.org/server/manage-cli.html) – CLI for domener, brukere, postbokser, mappings, kvoter og reindeksering.
- [Apache James – Metrics](https://james.apache.org/server/metrics.html) – Prometheus, JMX og tilgjengelige driftsmåledata.
- [James-1.0-arkiv fra Java Apache Project](https://svn.apache.org/repos/asf/james/server/tags/james_1_0/docs/index.html?p=1400000) – tidlig arkitektur- og MailServlet-planlegging.
- [Jakarta Project News 2001](https://jakarta.apache.org/site/news/news-2001.html) – migrering av James-prosjektet til Jakarta.
- [Apache James Document Archive](https://james.apache.org/server/archive/document_archive.html) – dokumentasjon for versjonene 1.x og 2.x.
- [James 2.1.3 – arkivert prosjektside](https://svn.apache.org/repos/asf/james/server/tags/deprecated/build_2_2_0_RC1/www/index.html?p=1400000)
- [Apache James 3.0 M1](https://james.apache.org/james/update/2010/11/05/james-3.0-M1.html) – IMAP, lagringsprofiler og Mailet-API i 3.x-generasjonen.
- [Apache James – GitHub-repositorium](https://github.com/apache/james-project) – kildekode, bygg og modulstruktur.
