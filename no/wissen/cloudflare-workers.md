---
title: "Cloudflare Workers: Isolates, bindings og Edge-drift"
blatt: "cloudflare-workers"
description: "Cloudflare Workers for infrastruktur-, meldings- og plattformadministratorer: forespørselsbane, workerd og V8-isolater, handlere og web-API-er, capability-bindings, KV, D1, R2 og Durable Objects, ruting, Compatibility Dates, grenser, observerbarhet, utrullinger, gjenoppretting og teknisk historie."
fakten:
  - label: Systemrolle
    wert: hendelsesdrevet databehandlingsplattform i Cloudflare-nettverket for HTTP, tidsplaner, køer, e-post og interne tjenester
    href: https://developers.cloudflare.com/workers/runtime-apis/handlers/
  - label: Kjøretid
    wert: workerd med V8-isolater; ingen egen varig serverprosess og ingen containerinstans per applikasjon
    href: https://developers.cloudflare.com/workers/reference/how-workers-works/
  - label: Programmeringsmodell
    wert: ES-modul-entrypoints med webstandard-API-er som Request, Response, Fetch, Streams og Web Crypto
    href: https://developers.cloudflare.com/workers/runtime-apis/
  - label: Språk
    wert: JavaScript, TypeScript, Python og Rust; andre språk via WebAssembly
    href: https://developers.cloudflare.com/workers/languages/
  - label: Inngangsveier
    wert: workers.dev, Route, Custom Domain samt Fetch-, Scheduled-, Queue-, e-post-, Alarm- og Tail-hendelser
    href: https://developers.cloudflare.com/workers/wrangler/configuration/
  - label: Bindings
    wert: Capability og API for plattformressurser uten at tilgangsnøkler eksponeres i koden
    href: https://developers.cloudflare.com/workers/runtime-apis/bindings/
  - label: Tilstandsmodeller
    wert: lokal cache · eventually consistent KV · D1 SQL · sterkt konsistent Durable Object · R2-objekt · kø
    href: https://developers.cloudflare.com/workers/platform/storage-options/
  - label: Kompatibilitet
    wert: Compatibility Date pluss valgfrie flagg fryser kjøretidsendringer som påvirker atferd, per utrulling
    href: https://developers.cloudflare.com/workers/configuration/compatibility-flags/
  - label: Control plane
    wert: Wrangler-konfigurasjon, Workers API, versjonsobjekter, utrullinger, secrets og ressursbindings
    href: https://developers.cloudflare.com/workers/configuration/
  - label: Utrulling
    wert: last opp versjon, opprett utrulling, fordel trafikk gradvis og gå tilbake til stabil versjon
    href: https://developers.cloudflare.com/workers/versions-and-deployments/gradual-deployments/
  - label: Driftssignaler
    wert: Invocation Outcome, status, exception, CPU- og wall time, subrequests, logger, spor og versjons-ID
    href: https://developers.cloudflare.com/workers/observability/logs/workers-logs/
  - label: Gjenopprettingsobjekter
    wert: kodeartefakt, konfigurasjon, Compatibility Date, bindings, secrets, dataeksporter, DO-livssyklus og testet tilbakeføring
    href: https://developers.cloudflare.com/workers/versions-and-deployments/
werbung:
  - newsletter
ctaThemen:
  - cloudflare-workers
translationSourceHash: 4c5445ed6d4ead36638b1de03ed9d67a4f82304b58f28d83a9c72e29e760d065
translationModel: gpt-5.6-terra
translatedAt: 2026-10-05T10:59:32.790Z
translationReview: automatic
---

# Cloudflare Workers: Isolates, bindings og Edge-drift

Cloudflare Workers er en hendelsesdrevet databehandlingsplattform i Cloudflare-nettverket. En Worker kan besvare eller videresende HTTP-forespørsler, kjøre planlagte oppgaver, konsumere kømeldinger, behandle innkommende e-post og fungere som en intern tjeneste for andre Workers. Operatøren administrerer verken en lytteprosess eller en enkelt VM. Operatøren administrerer kode, entrypoints, ruter, kjøretidskompatibilitet, tillatelser til plattformressurser, versjoner og driftsdata.

Ordet «Edge» beskriver bare ett mulig kjøringssted, ikke hele arkitekturen. En HTTP-Worker kan kjøre nær brukeren; Smart Placement kan flytte databehandling nærmere et backend-system; Durable Objects har et entydig tilstandshjemsted; D1, KV, R2 og Queues har hver sine replikerings- og konsistensmodeller. For administratorer er derfor ikke «serverløs» den avgjørende egenskapen, men oppdelingen i dataplan, control plane, tilstandstjenester og mulige feilpunkter.

Forklaringen følger en forespørsel fra det offentlige endepunktet inn i Workers-kjøretiden og videre til bindings, lagring og upstreams. Deretter settes utrulling, sikkerhet, observerbarhet og gjenoppretting inn i plattformdriftens perspektiv.

## Arkitektur: dataplan, kjøretid og control plane

Et Workers-system består av flere lag:

1. **Ingress og ruting:** Cloudflare mottar en forespørsel via DNS, TLS og HTTP og tilordner vertsnavn- eller banemønsteret til en Worker eller en statisk ressurs.
2. **Hendelsesdistribusjon:** Plattformen oppretter en Fetch-, Scheduled-, Queue-, e-post-, Alarm- eller Tail-hendelse og kaller det aktuelle entrypointet.
3. **Kjøretid:** `workerd` leverer V8-isolater, web-API-er, Node.js-kompatibilitet og kjøretidsgrenser.
4. **Bindings:** `env`-objektet gir koden capabilities for KV, D1, R2, Durable Objects, Queues, secrets, assets, andre Workers og ytterligere plattformtjenester.
5. **Tilstand og upstreams:** Data ligger utenfor det flyktige isolatet eller i kontrollerte Durable Object-lagre; utgående kall bruker Fetch, Service Bindings eller støttede socket-API-er.
6. **Control plane:** Wrangler, Dashboard og API administrerer konfigurasjon, ressursrelasjoner, secrets, versjoner, utrullinger og observerbarhet.

Cloudflare beskriver isolater, forespørselsrelatert databehandlingstid og distribuert kjøring som de tre viktigste forskjellene fra klassiske serverkjøretider ([How Workers works](https://developers.cloudflare.com/workers/reference/how-workers-works/)). Plattformreferansen skiller selve Workeren fra tilknyttede storage- og Developer Platform-produkter ([Cloudflare Workers documentation](https://developers.cloudflare.com/workers/)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-cloudflare-workers.svg?v=20260813" title="Interaktive Infografik: Cloudflare-Workers-Aufruf von DNS, TLS und Routing über Event Dispatch, workerd und V8-Isolate bis zu Bindings, Storage, Upstreams, Observability, Versionen und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-cloudflare-workers.svg?v=20260813">Åpne interaktiv grafikk direkte</a>.
</iframe>

## Forespørselsbane og kjøringssted

Ved en HTTP-forespørsel avgjør Cloudflare-konfigurasjonen først om et `workers.dev`-vertsnavn, en rute eller et Custom Domain utløser Workeren. En rute ligger foran en eksisterende origin og kan endre, besvare eller videresende forespørsler. Et Custom Domain kobler en Worker direkte til et vertsnavn. Statiske assets kan evalueres før eller etter Workeren; `run_worker_first` endrer denne rekkefølgen ([Wrangler configuration – routes](https://developers.cloudflare.com/workers/wrangler/configuration/), [Static Assets – configuration and bindings](https://developers.cloudflare.com/workers/static-assets/binding/)).

Den eksterne banen kan modelleres som `Client → DNS → Cloudflare Edge → TLS/HTTP → Route → Worker/Asset → Binding oder Origin`. Forbindelsen til origin er en ny transportkontekst. Klientsertifikat, `Authorization`-header, cache-nøkkel, Host-header og kilde-IP må derfor håndteres bevisst ved hver grense. En Service Binding omgår derimot offentlig navneoppløsning og gir en eksplisitt autorisert Worker-til-Worker-bane ([Service bindings](https://developers.cloudflare.com/workers/runtime-apis/bindings/service-bindings/)).

Smart Placement kan kjøre en Worker nærmere backend-systemene i stedet for å prioritere nærhet til brukeren alene. Dette er nyttig ved databaseintensive kall, men kan forlenge feil vei for assets eller rene Edge-transformasjoner. Cloudflare dokumenterer derfor ulike anbefalinger for Assets-first og Worker-first ([Placement](https://developers.cloudflare.com/workers/configuration/placement/)).

## workerd og V8-isolater

`workerd` er den serverorienterte JavaScript-/WebAssembly-kjøretiden bak Workers. V8 tilbyr isolater som separate kjøringskontekster. Mange isolater kan dele én prosess uten at hver applikasjon trenger sin egen JavaScript-prosess eller container. Web-API-er leveres i stor grad direkte av kjøretiden ([Introducing workerd](https://blog.cloudflare.com/workerd-open-source-workers-runtime/), [Workers security model](https://developers.cloudflare.com/workers/reference/security-model/)).

Et isolat er ikke en persistent maskin. Det kan gjenbrukes, håndtere flere forespørsler parallelt eller fjernes. To påfølgende forespørsler trenger ikke treffe samme instans eller sted. Globale objekter egner seg for uforanderlige, kostbare hjelpestrukturer eller opportunistiske cacher, men ikke som sannhetskilde. All korrekthet som avhenger av en global variabel mellom forespørsler, er en arkitekturfeil.

V8-isolasjon erstatter ikke alle sikkerhetskontroller. Cloudflare supplerer med kjøretids- og prosessbeskyttelse, ressursgrenser og spesifikt Spectre-forsvar. Operatøren er fortsatt ansvarlig for inputvalidering, autorisering, secret-scope, måltillatlister, utgangsgrenser og dataklassifisering. Isolatgrensen beskytter ikke mot en applikasjon som leser eller skriver feil data med sine legitime bindings.

## Hendelsesmodell og handlere

Workers implementerer entrypoints for ulike hendelsestyper. Den generelle handlerkatalogen omfatter Fetch, Scheduled, Queue, e-post, Alarm og Tail ([Handlers](https://developers.cloudflare.com/workers/runtime-apis/handlers/)).

| Handler | Utløser | Retur- og feilgrense | Typisk administratorspørsmål |
|---|---|---|---|
| `fetch()` | HTTP-forespørsel eller tjenestekall | `Response`, stream eller exception | Hvilken rute og versjon behandlet forespørselen? |
| `scheduled()` | Cron Trigger | Promise/Invocation Outcome | Ble den planlagte kjøringen utløst og fullført? |
| `queue()` | Meldingsbatch | Ack, retry eller dead-letter-atferd | Hvilken melding er idempotent, retrybar eller poison? |
| `email()` | Email Routing | Lever, avvis, videresend | Hvilken envelope- og policygrense gjelder? |
| `alarm()` | Durable Object-alarm | objektlokal kjøring | Hvilket objekt eier alarmen og tilstanden? |
| `tail()` | Trace-hendelse fra en producer-Worker | asynkron eksport | Kan observerbarheten selv feile eller bli rekursiv? |

Fetch-handleren mottar `Request`, `env` og `ctx` og leverer en web-API-`Response` ([Fetch handler](https://developers.cloudflare.com/workers/runtime-apis/handlers/fetch/)). `ctx.waitUntil()` registrerer arbeid som kan fortsette etter at responsen er sendt; det er fortsatt bundet til dokumenterte kjøretids- og feilgrenser og er ikke en langvarig jobbserver ([Context – waitUntil](https://developers.cloudflare.com/workers/runtime-apis/context/)). For varige, repeterbare prosesser er Queues eller Workflows bedre egnet enn en kjede av uovervåkede bakgrunns-Promises.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für ein minimales Workers-Modul">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-typescript">export interface Env {
  UPSTREAM: Fetcher;
  RELEASE: string;
}

export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise&lt;Response&gt; {
    const started = Date.now();
    const response = await env.UPSTREAM.fetch(request);
    ctx.waitUntil(Promise.resolve().then(() =&gt;
      console.log(JSON.stringify({ release: env.RELEASE, status: response.status, ms: Date.now() - started }))
    ));
    return response;
  },
} satisfies ExportedHandler&lt;Env&gt;;</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-typescript">export interface Env {
  UPSTREAM: Fetcher;
  RELEASE: string;
}

export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise&lt;Response&gt; {
    const started = Date.now();
    const response = await env.UPSTREAM.fetch(request);
    ctx.waitUntil(Promise.resolve().then(() =&gt;
      console.log(JSON.stringify({ release: env.RELEASE, status: response.status, ms: Date.now() - started }))
    ));
    return response;
  },
} satisfies ExportedHandler&lt;Env&gt;;</code></pre>
  </div>
</div>

Eksemplet bruker en Service Binding i stedet for en fritt konfigurerbar URL. Dermed flyttes tilgjengelighet og autorisering til utrullingskonfigurasjonen, mens koden bare ser en `Fetcher`-capability. Kjøretids-API-ene er weborienterte og omfatter blant annet Fetch, Streams, Web Crypto, WebSockets, HTMLRewriter og TCP Sockets ([Runtime APIs](https://developers.cloudflare.com/workers/runtime-apis/)).

Koden kjører ikke i et fritt valgt lokalt Node-miljø, men mot en versjonert kjøretidskontrakt. Compatibility Date bestemmer hvilke atferdsendringer som gjelder for en utrulling.

## Compatibility Date og kjøretidskontrakt

En Compatibility Date er ikke en release-merknad og ikke en uttalelse om artikkelens «nåværende status». Den er en del av kjøretidskontrakten for en konkret Worker-utplassering. Endringer som kan bryte eksisterende atferd, aktiveres etter dato eller flagg. En eldre dato fastholder kompatibel atferd; en nyere dato må testes og rulles ut som en avhengighetsoppdatering ([Compatibility flags](https://developers.cloudflare.com/workers/configuration/compatibility-flags/)).

Administratorer behandler derfor disse feltene samlet:

- Kodecommit og buntet artefakt;
- Compatibility Date og eksplisitte flagg;
- Wrangler- og byggeverktøykjede;
- Bindingdefinisjoner, routes og placement;
- Versjons-ID og utrullingsvekt;
- Data- eller Durable Object-migreringer.

Node.js-kompatibilitet er opt-in og implementerer bare en dokumentert del av Node-API-ene. Enkelte moduler er fullstendige, andre delvise eller bare tilgjengelige som import-stubber. `nodejs_compat` må derfor ikke likestilles med «fullstendig Node-kjøretid» ([Node.js compatibility](https://developers.cloudflare.com/workers/runtime-apis/nodejs/)).

## Språk og teknologistack

Cloudflare dokumenterer JavaScript, TypeScript, Python og Rust som førsteklasses språkbaner; WebAssembly åpner for flere kildespråk ([Languages](https://developers.cloudflare.com/workers/languages/)). Den resulterende teknologistacken skiller utvikling og kjøring:

| Lag | Typisk teknologi | Driftsrelevans |
|---|---|---|
| Kildekode | TypeScript/JavaScript, Python, Rust | Språktoolchain, avhengigheter, tester |
| Bygg | Wrangler/esbuild eller rammeverksadapter | Bundle, Source Maps, externals, deterministiske bygg |
| Kjøretid | workerd, V8, web- og valgfrie Node-API-er | Compatibility Date, CPU/minne, event loop |
| Capability | `env`-bindings | Least Privilege, ressurs-ID, miljøseparasjon |
| Tilstand | Cache, KV, D1, R2, Durable Objects, Queues | Konsistens, datahjemsted, backup, recovery |
| Ingress | Route, Custom Domain, workers.dev, trigger | DNS, TLS, rekkefølge, fail-open/closed |
| Control plane | Wrangler, API, Dashboard, CI/CD | Autentisering, versjonering, rollout, audit |

Rust integreres via `workers-rs` og WebAssembly; den importerte Wasm-modulen forblir del av Worker-artefaktet ([Rust language support](https://developers.cloudflare.com/workers/languages/rust/)). Python Workers bruker egne entrypoint-klasser og en kjøretidsintegrasjon, ikke en fritt administrerbar CPython-prosess.

## Konfigurasjon og Wrangler

Wrangler-konfigurasjonen er den deklarative koblingen mellom kode og plattform. Den definerer minst navn, entrypoint, Compatibility Date, routes og bindings. Navngitte miljøer kan ha avvikende verdier; bindings må kontrolleres bevisst per miljø. Cloudflare dokumenterer feltene og arv i Wrangler-skjemaet ([Workers configuration](https://developers.cloudflare.com/workers/configuration/), [Wrangler configuration reference](https://developers.cloudflare.com/workers/wrangler/configuration/)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Workers-Konfigurations- und Typprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">npx wrangler whoami
npx wrangler types
npx wrangler deploy --dry-run
npx wrangler deployments status</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">npx wrangler whoami
npx wrangler types
npx wrangler deploy --dry-run
npx wrangler deployments status</code></pre>
  </div>
</div>

[`npx`](https://docs.npmjs.com/cli/commands/npx) starter prosjektets lokalt låste CLI; [`Wrangler Commands`](https://developers.cloudflare.com/workers/wrangler/commands/) dokumenterer autentisering, typegenerering, utrulling, versjons- og loggkommandoer. `wrangler types` genererer typer fra den faktiske bindingkonfigurasjonen. `deploy --dry-run` kontrollerer bundle og konfigurasjon, men gir ikke bevis for fungerende fjernressurser eller korrekt autorisering.

Etter at kjøretid og bygg er avklart, oppstår tilgangsspørsmålet. En Worker når databaser, køer, secrets og andre Workers via deklarerte bindings i stedet for fritt distribuerte tilgangsdata.

## Bindings som capability-modell

En binding er samtidig tillatelse og kjøretids-API. Workeren mottar for eksempel `env.ARCHIVE` som R2-bucket, `env.DB` som D1-database eller `env.AUTH` som intern tjeneste. Den underliggende plattformlegitimasjonen eksponeres ikke for applikasjonskoden som en gjenbrukbar API-nøkkel ([Bindings](https://developers.cloudflare.com/workers/runtime-apis/bindings/)).

Capability-modellen flytter det sentrale administratorspørsmålet fra «Hvilket secret ligger i miljøvariabelen?» til «Hvilken ressurs er bundet under hvilket navn i hvilket miljø til hvilken versjon?». En feil bucket-ID eller Service Binding-versjon kan føre til datatap eller tilgang på tvers av miljøer til tross for identisk kode.

Bindings bør være adskilt etter oppgave og miljø. Produksjons- og test-Workers deler ikke skrivbare buckets, køer eller databaser, med mindre dette uttrykkelig inngår i en kontrollert integrasjonstest. Generiske navn som `DB` er tilstrekkelige i koden når utrullingen binder den faktiske ressursen entydig og verifiserbart.

### Secrets og variabler

Vanlige variabler er konfigurasjon, ikke hemmeligheter. Secrets er krypterte tekstbindings og administreres utenfor kildekoden. Nødvendige secret-navn kan deklareres uten at verdiene skrives i konfigurasjonsfilen ([Environment variables](https://developers.cloudflare.com/workers/configuration/environment-variables/), [Secrets](https://developers.cloudflare.com/workers/configuration/secrets/)).

Et secret-bytte er en tilstandsendring i applikasjonen. Klienter som globalt avledes fra `env` kan beholde en gammel verdi i gjenbrukte isolater; Cloudflare anbefaler å opprette slike avledninger per forespørsel. Rotasjon omfatter derfor setting, kontroll av utrulling/binding, parallell gyldighet dersom målsystemet krever det, telemetri og kontrollert fjerning.

## Service Bindings og Worker-til-Worker-arkitektur

Service Bindings kobler Workers eksplisitt uten behov for et offentlig adresserbart HTTP-endepunkt. Den kalte tjenesten kan eksponeres via Fetch eller RPC. Dette reduserer nettverks- og credentialkonfigurasjon, men eliminerer ikke versjons- og kontraktsproblemer. Ved separate utrullinger kan caller og callee bruke ulike API-versjoner ([Service bindings](https://developers.cloudflare.com/workers/runtime-apis/bindings/service-bindings/), [Gradual deployments](https://developers.cloudflare.com/workers/versions-and-deployments/gradual-deployments/)).

Administratorer dokumenterer per binding:

- Caller og måltjeneste;
- Miljø og målversjon eller utrullingsstrategi;
- Offentlig tilgjengelighet for mål-Workeren;
- Semantikk for timeout, feil og retry;
- Videreføring av request-ID og trace-korrelasjon;
- Kontraktskompatibilitet ved uavhengige utrullinger.

En kjede av mange Workers fordeler en forespørsel over flere mulige feilpunkter. Mindre offentlig angrepsflate er verdifullt; for fin oppdeling kan til gjengjeld øke subrequest-dybde, feilsøkingsarbeid og versionsskew.

## Tilstandstjenester: modell før produktnavn

Isolat-heapen er flyktig. Persistent tilstand ligger i bundne tjenester med tydelig forskjellig semantikk. Cloudflares storage-oversikt kategoriserer produktene etter tilgangsmønster og konsistens ([Choosing a data or storage product](https://developers.cloudflare.com/workers/platform/storage-options/)).

| Tjeneste | Modell | Konsistens-/stedsgrense | Egnet for | Kritisk administratorspørsmål |
|---|---|---|---|---|
| Cache API | HTTP-respons-cache | lokalt i datasenteret, flyktig | repeterbare responser | Er cache-nøkkelen fullstendig og sikker? |
| Workers KV | globalt lesbar key-value-store | eventually consistent, read cache | konfigurasjon, read-heavy-data | Tåler arbeidsflyten gamle eller negative reads? |
| D1 | administrert SQL basert på SQLite | database- og sesjonsrelatert semantikk | relasjonelle applikasjonstabeller | Hvilken replika-/sesjonsgaranti trenger lesingen? |
| Durable Objects | entydig objektinstans pluss storage | sterkt konsistent og serialiserbar per objekt | koordinering, låser, sesjoner, WebSockets | Er partition key valgt riktig? |
| R2 | S3-kompatibel objektlagring | sterkt konsistent per objekt | blobs, arkiver, store payloads | Hvordan versjoneres objekt, metadata og indeks sammen? |
| Queues | asynkrone meldinger | minst-én-gang-orientert behandling og retry-policy | frakobling, utjevning av last | Er consumeren idempotent, og finnes det en DLQ? |

### Cache API

Cache API arbeider lokalt i datasenteret der Workeren behandler forespørselen. `cache.put()` er derfor ikke global replikering og ikke en persistent databaseskriving. `caches.default` deler standard cache-kontekst; navngitte cacher oppretter namespaces, men heller ingen global konsistens ([How the Cache works](https://developers.cloudflare.com/workers/reference/how-the-cache-works/)).

Personaliserte svar må bare caches med en nøkkel som avbilder alle relevante identitets- og variantkjennetegn. `Authorization`, cookies, språk, encoding og tenanttilhørighet er typiske lekkasjegrenser. Cache-purge og originvalidering hører til recovery, ikke bare TTL-er.

### Workers KV

KV replikerer data via sentrale stores og Edge-cacher. Endringer kan bli synlige forsinket på andre steder; også «ikke funnet» kan caches. Cloudflare oppgir forsinkelser på opptil 60 sekunder eller mer for fjerne steder og anbefaler ikke KV for atomiske read-modify-write-transaksjoner ([How KV works](https://developers.cloudflare.com/kv/concepts/how-kv-works/)).

KV egner seg for hovedsakelig lest konfigurasjon, allow-/deny-lister eller cacher når staleness tolereres eksplisitt. Globale rate limits, entydige sekvenser eller umiddelbar tilbakekalling av credentials hører ikke hjemme i KV uten ekstra koordinering.

### Durable Objects

Et Durable Object kombinerer en globalt entydig objekt-ID med seriell koordinering og privat lagring. Forespørsler for samme ID rutes til samme logiske instans. Nye namespaces bruker SQLite-storagebanen; denne tilbyr transaksjonell, sterkt konsistent lagring og Point-in-Time-Recovery-funksjoner ([Durable Objects overview](https://developers.cloudflare.com/durable-objects/), [SQLite-backed Durable Object Storage](https://developers.cloudflare.com/durable-objects/api/sqlite-storage-api/)).

Et Durable Object er heller ikke en kontinuerlig kjørende server. In-memory-tilstand kan forsvinne ved hibernation eller eviction og må om nødvendig rekonstrueres fra storage. Det finnes ingen pålitelig shutdown hook ([Durable Object lifecycle](https://developers.cloudflare.com/durable-objects/concepts/durable-object-lifecycle/)). Endringer i klasser og storage-tilordning er livssyklusoperasjoner; de behandles som datamigreringer ([Durable Object class lifecycle](https://developers.cloudflare.com/durable-objects/reference/durable-objects-migrations/)).

### D1, R2 og Queues

D1 tilbyr SQL for relasjonelle data. R2 lagrer store ustrukturerte objekter og eksponerer et S3-kompatibelt API utenfor, og et Binding-API innenfor Workers. Queues frakobler producer og consumer. Disse tjenestene erstatter ikke hverandre: Et R2-objekt kan bære payloaden, D1 indeksen, en kø kan utløse behandlingen og et Durable Object kan koordinere konkurrerende endringer.

Den felles committen av disse fire tilstandene er ikke automatisk atomisk. Administratorer planlegger idempotency keys, Outbox-/Inbox-mønstre, reconciliation og dead-letter-behandling. Produktdokumentasjon og grenser for de enkelte tjenestene for [D1](https://developers.cloudflare.com/d1/), [R2](https://developers.cloudflare.com/r2/) og [Queues](https://developers.cloudflare.com/queues/) er del av runbooken.

## Statiske assets og full-stack-Workers

En Worker kan levere statiske assets sammen med kode. Assets Binding lar koden hente en asset eksplisitt. Rutingalternativet bestemmer om eksisterende assets omgår Workeren, eller om Workeren kjører først ([Static Assets](https://developers.cloudflare.com/workers/static-assets/)).

Rekkefølgen er en sikkerhetsbeslutning. Ved assets-first må en tilfeldig eksisterende filbane ikke kunne omgå autentisering. Ved Worker-first øker hver asset-forespørsel databehandlings- og avhengighetsbelastningen. Rammeverksadaptere skjuler delvis denne rekkefølgen; det bygde utrullingsartefaktet og Wrangler-konfigurasjonen er fortsatt den autoritative sannheten.

## Nettverks- og protokollperspektiv

Workers ligger i applikasjonsbanen over DNS, TLS og HTTP. Plattformen terminerer den eksterne transporten; Workeren behandler web-API-forespørsler. Utgående `fetch()` oppretter et nytt HTTP-kall. TCP Sockets tillater utvalgte utgående protokoller, men gjør ikke Workeren om til en generelt tilgjengelig TCP-server ([Workers protocols](https://developers.cloudflare.com/workers/reference/protocols/), [TCP sockets](https://developers.cloudflare.com/workers/runtime-apis/tcp-sockets/)).

For meldingsadministratorer er tre grenser viktige:

- En e-posthandler utløses av Cloudflare Email Routing; den lytter ikke selv på [SMTP](/kb/smtp).
- En Worker som kaller et e-post-API, må behandle OAuth-audience, tokenlivssyklus og retries som enhver annen API-klient.
- DNS-, TLS- og HTTP-feil oppstår før entrypointet; binding-, auth- og applikasjonsfeil etterpå. En `fetch()`-timeout sier uten korrelasjon ikke hvilken fase som feilet.

## Sikkerhets- og tillitsgrenser

Sikkerhetsmodellen har minst fem separate tillitsgrenser:

1. **Offentlig forespørsel:** vilkårlige headere, body-størrelse, metode og identitet.
2. **Cloudflare-konfigurasjon:** zone, route, WAF, Access, sertifikater og fail-open/closed.
3. **Worker-artefakt:** kode, avhengigheter, Source Maps, Compatibility Date og forsyningskjede.
4. **Bindings:** konkrete ressurstillatelser, secrets og interne tjenestebaner.
5. **Upstreams og lagrede data:** egen autorisering, konsistens og recovery.

Least Privilege oppnås primært gjennom små bindingflater og separate miljøer. En Worker med skrivetilgang til alle buckets og databaser forblir en høyt privilegert principal, selv om den ikke har synlige tilgangsnøkler. Workers-sikkerhetsdokumentasjonen forklarer kjøretidsisolasjon; den erstatter ikke en applikasjonssikkerhetsarkitektur ([Security model](https://developers.cloudflare.com/workers/reference/security-model/)).

Kontroller for forsyningskjeden omfatter låste pakkeversjoner, lockfile, reproduserbart bygg, dependency-scanning, secret-scanning, minimale opplastingsrettigheter og beskyttede utrullingsmiljøer. Det genererte bundle-et kontrolleres før opplasting; kun gjennomgang av kildekode er ikke tilstrekkelig når bundlere eller rammeverk bygger inn ekstra moduler.

Nærhet til Edge opphever ikke ressursbegrensninger. CPU-tid, subrequests, minne og plattformgrenser må allerede tas med i design og lastmodell.

## Grenser som arkitekturparametere

Workers begrenser blant annet CPU-tid, minne, oppstartstid, subrequests, bundle-størrelse, loggvolum, antall ruter og statiske assets. Verdier varierer etter plan og invocationtype og kan endres. Derfor hører ingen kopiert grenseoversikt hjemme i en statisk artikkel som påstått tidløs sannhet; den offisielle siden [Workers Limits](https://developers.cloudflare.com/workers/platform/limits/) er driftskilden.

Administratorer skiller mellom:

- **CPU Time:** aktiv databehandlingstid; venting på I/O teller ikke på samme måte som CPU.
- **Wall Time:** hendelsens forløpte varighet; regler skiller mellom Fetch, Cron, Queue og Durable Object.
- **Startup Time:** modulevaluering og initialisering før handleren.
- **Subrequests:** Fetch- og plattformoperasjoner per invocation.
- **Memory:** heap, streams, buffere og bibliotekstilstand i isolatet.
- **Log Budget:** for store eller hemmelige payloads er både et kostnads- og personvernproblem.

En feil `1102` peker på overskredne ressursgrenser; `10021` kan oppstå ved for kostbar oppstartsinitialisering. Feilnummeret er et startpunkt, ikke root cause. CPU-profil, invocationtype, versjon, inputklasse og subrequest-graf må vurderes samlet.

## Lokal utvikling og testing

Lokal utvikling kjører kode med `workerd` via Miniflare. Bindings simuleres lokalt som standard; enkelte remote bindings kan målrettet kobles til ekte plattformressurser. Fullstendig eksternt kjørt `wrangler dev --remote` er fortsatt tilgjengelig for nettverksspesifikke tilfeller, men er ikke lenger standardbanen ([Local development](https://developers.cloudflare.com/workers/local-development/), [Bindings per development mode](https://developers.cloudflare.com/workers/local-development/bindings-per-env/)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für lokale Workers-Tests">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">npx wrangler dev --local
$response = Invoke-WebRequest 'http://127.0.0.1:8787/health'
$response.StatusCode
$response.Headers['content-type']
npx vitest run</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">npx wrangler dev --local
curl --silent --show-error --fail --dump-header - http://127.0.0.1:8787/health
npx vitest run</code></pre>
  </div>
</div>

[`Invoke-WebRequest`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest) og [`curl`](https://curl.se/docs/manpage.html) kontrollerer status og headere for den lokale HTTP-banen. Workers-Vitest-integrasjonen kjører tester innenfor `workerd` og leverer hjelpere for bindings, forespørsler og Durable Objects ([Vitest integration](https://developers.cloudflare.com/workers/testing/vitest-integration/), [Workers test APIs](https://developers.cloudflare.com/workers/testing/vitest-integration/test-apis/)).

En grønn unit test beviser ikke korrekt route, remote-tillatelse eller datamigrering. Testpyramiden omfatter ren logikk, kjøretidsintegrasjon, bindingsemantikk, stagingrute, kontrollert Production-smoke-test og recoveryøvelse.

## Utrulling, versjon og rollout

En **versjon** er et uforanderlig kode-/konfigurasjonsobjekt. En **utrulling** fordeler trafikk på én eller flere versjoner. Gradual Deployments forskyver vekter trinnvis og gjør det mulig å observere og gå tilbake til en stabil versjon ([Versions and deployments](https://developers.cloudflare.com/workers/versions-and-deployments/), [Gradual deployments](https://developers.cloudflare.com/workers/versions-and-deployments/gradual-deployments/)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Workers-Versionen und Logs">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">npx wrangler versions list
npx wrangler deployments status
npx wrangler tail --format json
npx wrangler check startup</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">npx wrangler versions list
npx wrangler deployments status
npx wrangler tail --format json
npx wrangler check startup</code></pre>
  </div>
</div>

Før en rollout lagres tilordningen `Git commit → Bundlehash → Worker-Version → Deploymentgewicht`. Ved flere Service Bindings er versionsskew del av testen. Durable Objects krever spesielle migrerings- og rolloutregler fordi vilkårlige klassestadier ikke kan være aktive samtidig for en objekt-ID.

Rollback tilbakefører kode og konfigurasjon, men ikke automatisk eksterne data. En ny Worker kan ha skrevet data i et format som den gamle ikke forstår. Skjemaendringer bruker Expand/Contract, fremoverkompatibilitet eller separat testet datatilbakeføring. «Rollback mulig» er først dokumentert når kode-, binding- og datasiden er kontrollert sammen.

## Observerbarhet og driftsbevis

Workers Logs samler invocation-logger, egne logger, feil og ubehandlede exceptions. Tail Workers eller Logpush kan eksportere hendelser; OpenTelemetry-mål tilbyr en mer direkte eksportbane for logger og spor ([Workers Logs](https://developers.cloudflare.com/workers/observability/logs/workers-logs/), [Tail Workers](https://developers.cloudflare.com/workers/observability/logs/tail-workers/), [Traces](https://developers.cloudflare.com/workers/observability/traces/)).

Hver invocation bør minst gjøre følgende korrelerbart:

- Worker-navn, versjons-ID og miljø;
- Hendelsestype og route;
- Request-/Message-/Correlation-ID;
- Resultat, HTTP-status eller Ack/Retry;
- CPU- og Wall Time;
- Subrequest-mål og latens uten secrets;
- Binding-/ressursklasse, ikke sensitivt fullinnhold;
- Faglig idempotency key ved asynkron behandling.

`console.log()` er ikke et ubegrenset bevisarkiv. Logggrenser, sampling og redaction påvirker synligheten. Credentials, Authorization-headere, cookies, fullstendige e-postinnhold og personrelaterte payloads logges ikke. Overvåking overvåker også eksportbanen: En feilende Tail Worker må ikke gjøre diagnose av producer-feilen umulig.

Diagnosen følger publiseringsveien: route og DNS, aktiv versjon, kjøretid og bindings, utgående avhengigheter og til slutt logger og spor.

## Admin-diagnose etter faser

Workers-feil skilles effektivt langs utførelsesbanen:

1. **DNS:** Løser forventet host til Cloudflare, og er sonen aktiv?
2. **TLS/HTTP:** Kontroller sertifikat, SNI, protokoll og status.
3. **Route/Asset:** Treffer host/path Workeren, en asset eller origin?
4. **Versjon:** Hvilken versjon og hvilken utrullingsvekt behandlet forespørselen?
5. **Kjøretid:** Kontroller startup, CPU, memory, exception og handlerutfall.
6. **Binding:** Finnes ressursen i dette miljøet, og har den forventet capability?
7. **Upstream/Storage:** Undersøk timeout, auth, konsistens, datatilstand og retry.
8. **Recovery:** Bekreft stabil versjon, datakompatibilitet og reconciliation.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für DNS-, TLS- und Workers-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName $env:WORKER_HOST
Test-NetConnection $env:WORKER_HOST -Port 443
$r = Invoke-WebRequest "https://$env:WORKER_HOST/health" -Headers @{ 'x-correlation-id' = [guid]::NewGuid() }
$r.StatusCode
$r.Headers
npx wrangler deployments status</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig +short "$WORKER_HOST"
nc -vz "$WORKER_HOST" 443
openssl s_client -connect "$WORKER_HOST:443" -servername "$WORKER_HOST" -brief &lt;/dev/null
curl --silent --show-error --fail --dump-header - "https://$WORKER_HOST/health"
npx wrangler deployments status</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname), [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection), [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility), [`nc`](https://man.openbsd.org/nc), [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) og [`curl`](https://curl.se/docs/manpage.html) skiller DNS, TCP, TLS og HTTP før kjøretiden. [DNS](/kb/dns), [TCP](/kb/tcp), [TLS](/kb/tls), [APIs](/kb/apis) og [Troubleshooting](/kb/troubleshooting) behandler disse lagene i dybden.

### Typiske feilbilder

| Symptom | Sannsynlig fase | Bevis | Vanlig tankefeil |
|---|---|---|---|
| Originrespons i stedet for Worker-respons | Route/Asset/Fail-open | Route, host, path, responsemarkør | «Utrulling vellykket» betyr «route aktiv» |
| 1101 eller exception | Applikasjon/binding | Workers Logs, versjon, stack | Bare distribuere på nytt i stedet for å kontrollere inputklasse |
| 1102 | Ressursgrense | CPU/Wall Time, profil, invocationtype | Likestille I/O-ventetid og CPU |
| Sporadisk gammel KV-verdi | KV-konsistens | Key, sted, cache-TTL, skrivetid | Behandle KV som en global transaksjon |
| Lokalt grønt, remote binding-feil | Miljø/capability | Konkret ressurs-ID og bindingnavn | Simulering beviser IAM-/ressurstilstand |
| Bare deler av trafikken feiler | Gradual Deployment | Versjons-ID og vekt | Aggregere metrikker uten versjonsdimensjon |
| Rollback fikser kode, men ikke data | Skjema-/tilstandsgrense | Skrevet format, migrering, reconciliation | Utrulling er fullstendig recovery |

## Backup og disaster recovery

Serverløshet eliminerer ikke backup. Den flytter objektene som må beskyttes:

- Git-repositorium, lockfile og reproduserbar byggdefinisjon;
- Wrangler-konfigurasjon, routes, Compatibility Date og flagg;
- Inventar over alle bindings og målressurser;
- Secret-verdier eller deres eksterne source of truth og rotasjonsprosess;
- Dataeksporter og restore-prosedyrer for D1, R2, KV og eksterne systemer;
- Durable Object-klasser, lifecycle-/migreringsstatus og eventuelt PITR;
- Queue-/DLQ-tilstand, idempotency keys og reconciliation;
- Versjons-/utrullingshistorikk samt en testet rollback-bane.

Ikke alle produkter har samme eksport- og restore-semantikk. En R2-objektbackup beskytter ikke D1-indeksen; en D1-restore gjenoppretter ikke en allerede bekreftet kømelding; en tilbakeført Worker kan tolke nye objektformater feil. [Backup og disaster recovery](/kb/backup-dr) må derfor definere et konsistent applikasjonspunkt i stedet for bare enkeltstående produktkopier.

Recovery testes i scenarier: feil secret, slettet route, feil versjon, inkompatibelt D1-skjema, tapt R2-objekt, poison Queue Message, feil Durable Object-klasse og utilgjengelig upstream. RTO og RPO fastsettes per tilstandstjeneste og for den sammensatte applikasjonen.

## Teknisk historie

Cloudflare presenterte Workers i 2017 som programmerbar kjøring i det globale nettet. Utgangspunktet var latenstidsgrensen til sentrale datasentre og ideen om å kjøre kode nær databanen ([Code Everywhere: Why We Built Cloudflare Workers](https://blog.cloudflare.com/code-everywhere-cloudflare-workers/)).

Arkitekturen satset tidlig på V8-isolater i stedet for en container eller prosess per funksjon. Cloudflare beskrev i 2018 de økonomiske og tekniske forskjellene i denne modellen, men også begrensningen til JavaScript- eller WebAssembly-egnede språk ([Cloud Computing without Containers](https://blog.cloudflare.com/cloud-computing-without-containers/)).

Workers KV la til globalt lesbar, eventually consistent tilstand. Durable Objects ble annonsert i 2020 som en motmodell for koordinert, sterkt konsistent objekttilstand ([Introducing Workers Durable Objects](https://blog.cloudflare.com/introducing-workers-durable-objects/)). Dermed utviklet Workers seg fra ren forespørselstransformasjon til en applikasjonsplattform med flere eksplisitte konsistensmodeller.

I 2022 publiserte Cloudflare `workerd` under Apache 2.0. Den åpne kjøretiden deler kode med produksjonssystemet og forbedret nøyaktigheten i lokal utvikling; den er likevel ikke hele Cloudflare-plattformen med dens ruting, orkestrering og sikkerhetsdrift ([Introducing workerd](https://blog.cloudflare.com/workerd-open-source-workers-runtime/), [workerd repository](https://github.com/cloudflare/workerd)).

Senere utvidelser førte til ES-modul-entrypoints, Compatibility Dates, Service/RPC-bindings, D1, R2, Queues, Workflows, Python, utvidet Node-kompatibilitet, versjonsobjekter og trinnvise utrullinger. Denne historien forklarer dagens kjerne: Workers er verken nettleser-JavaScript på et CDN eller en vilkårlig Linux-server, men en hendelsesdrevet, capability-basert kjøretid med separate dataplan- og control plane-produkter.

## Admin-sjekkliste

Før produksjonsdrift er minst følgende punkter dokumentert:

- Route, Custom Domain, asset-rekkefølge og fail-open/closed er dokumentert.
- Compatibility Date, flagg, Wrangler-versjon, bundle og Source Maps er reproduserbare.
- Hver binding har owner, miljø, ressurs-ID, rettigheter og recovery-bane.
- Cache-, KV-, D1-, R2-, Durable Object- og Queue-semantikk blandes ikke.
- Secrets er utenfor repositoriet, kan roteres og er ikke synlige i logger.
- Timeouts, retries, idempotens og dead-letter-atferd er definert per hendelsestype.
- Logger og spor inneholder versjon og Correlation-ID, men ingen sensitive payloads.
- Gradual Deployment og rollback tar hensyn til Service Binding-versjoner og dataformater.
- Grenser overvåkes fra den offisielle plattformsiden, ikke fra en kopiert tabell.
- Backup, restore og reconciliation er testet for den sammensatte applikasjonen.

## Kilder

- [How Workers works](https://developers.cloudflare.com/workers/reference/how-workers-works/)
- [Cloudflare Workers documentation](https://developers.cloudflare.com/workers/)
- [Wrangler configuration reference](https://developers.cloudflare.com/workers/wrangler/configuration/)
- [Static Assets – configuration and bindings](https://developers.cloudflare.com/workers/static-assets/binding/)
- [Service bindings](https://developers.cloudflare.com/workers/runtime-apis/bindings/service-bindings/)
- [Placement](https://developers.cloudflare.com/workers/configuration/placement/)
- [Introducing workerd](https://blog.cloudflare.com/workerd-open-source-workers-runtime/)
- [Workers security model](https://developers.cloudflare.com/workers/reference/security-model/)
- [Handlers](https://developers.cloudflare.com/workers/runtime-apis/handlers/)
- [Fetch handler](https://developers.cloudflare.com/workers/runtime-apis/handlers/fetch/)
- [Context – waitUntil](https://developers.cloudflare.com/workers/runtime-apis/context/)
- [Runtime APIs](https://developers.cloudflare.com/workers/runtime-apis/)
- [Compatibility flags](https://developers.cloudflare.com/workers/configuration/compatibility-flags/)
- [Node.js compatibility](https://developers.cloudflare.com/workers/runtime-apis/nodejs/)
- [Languages](https://developers.cloudflare.com/workers/languages/)
- [Rust language support](https://developers.cloudflare.com/workers/languages/rust/)
- [Workers configuration](https://developers.cloudflare.com/workers/configuration/)
- [npx documentation](https://docs.npmjs.com/cli/commands/npx)
- [Wrangler Commands](https://developers.cloudflare.com/workers/wrangler/commands/)
- [Bindings](https://developers.cloudflare.com/workers/runtime-apis/bindings/)
- [Environment variables](https://developers.cloudflare.com/workers/configuration/environment-variables/)
- [Secrets](https://developers.cloudflare.com/workers/configuration/secrets/)
- [Gradual deployments](https://developers.cloudflare.com/workers/versions-and-deployments/gradual-deployments/)
- [Choosing a data or storage product](https://developers.cloudflare.com/workers/platform/storage-options/)
- [How the Cache works](https://developers.cloudflare.com/workers/reference/how-the-cache-works/)
- [How KV works](https://developers.cloudflare.com/kv/concepts/how-kv-works/)
- [Durable Objects overview](https://developers.cloudflare.com/durable-objects/)
- [SQLite-backed Durable Object Storage](https://developers.cloudflare.com/durable-objects/api/sqlite-storage-api/)
- [Durable Object lifecycle](https://developers.cloudflare.com/durable-objects/concepts/durable-object-lifecycle/)
- [Durable Object class lifecycle](https://developers.cloudflare.com/durable-objects/reference/durable-objects-migrations/)
- [D1](https://developers.cloudflare.com/d1/)
- [R2](https://developers.cloudflare.com/r2/)
- [Queues](https://developers.cloudflare.com/queues/)
- [Static Assets](https://developers.cloudflare.com/workers/static-assets/)
- [Workers protocols](https://developers.cloudflare.com/workers/reference/protocols/)
- [TCP sockets](https://developers.cloudflare.com/workers/runtime-apis/tcp-sockets/)
- [Workers Limits](https://developers.cloudflare.com/workers/platform/limits/)
- [Local development](https://developers.cloudflare.com/workers/local-development/)
- [Bindings per development mode](https://developers.cloudflare.com/workers/local-development/bindings-per-env/)
- [Invoke-WebRequest](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest)
- [curl manual](https://curl.se/docs/manpage.html)
- [Vitest integration](https://developers.cloudflare.com/workers/testing/vitest-integration/)
- [Workers test APIs](https://developers.cloudflare.com/workers/testing/vitest-integration/test-apis/)
- [Versions and deployments](https://developers.cloudflare.com/workers/versions-and-deployments/)
- [Workers Logs](https://developers.cloudflare.com/workers/observability/logs/workers-logs/)
- [Tail Workers](https://developers.cloudflare.com/workers/observability/logs/tail-workers/)
- [Traces](https://developers.cloudflare.com/workers/observability/traces/)
- [Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)
- [Test-NetConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection)
- [dig manual](https://bind9.readthedocs.io/en/latest/manpages.html)
- [nc(1)](https://man.openbsd.org/nc)
- [openssl-s_client(1)](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Code Everywhere: Why We Built Cloudflare Workers](https://blog.cloudflare.com/code-everywhere-cloudflare-workers/)
- [Cloud Computing without Containers](https://blog.cloudflare.com/cloud-computing-without-containers/)
- [Introducing Workers Durable Objects](https://blog.cloudflare.com/introducing-workers-durable-objects/)
- [workerd repository](https://github.com/cloudflare/workerd)
