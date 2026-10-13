---
title: "Cloudflare Workers: Isolates, bindningar och edge-drift"
blatt: "cloudflare-workers"
description: "Cloudflare Workers för infrastruktur-, meddelande- och plattformsadministratörer: requestväg, workerd och V8-isolat, hanterare och webb-API:er, capability-bindningar, KV, D1, R2 och Durable Objects, routing, Compatibility Dates, gränser, observability, driftsättningar, återställning och teknisk historia."
fakten:
  - label: Systemroll
    wert: händelsedriven compute-plattform i Cloudflare-nätverket för HTTP, scheman, köer, e-post och interna tjänster
    href: https://developers.cloudflare.com/workers/runtime-apis/handlers/
  - label: Körtid
    wert: workerd med V8-isolat; ingen egen långvarig serverprocess och ingen containerinstans per applikation
    href: https://developers.cloudflare.com/workers/reference/how-workers-works/
  - label: Programmeringsmodell
    wert: ES-modulentrypoints med webbstandard-API:er som Request, Response, Fetch, Streams och Web Crypto
    href: https://developers.cloudflare.com/workers/runtime-apis/
  - label: Språk
    wert: JavaScript, TypeScript, Python och Rust; ytterligare språk via WebAssembly
    href: https://developers.cloudflare.com/workers/languages/
  - label: Ingångsvägar
    wert: workers.dev, Route, Custom Domain samt Fetch-, Scheduled-, Queue-, E-Mail-, Alarm- och Tail-händelser
    href: https://developers.cloudflare.com/workers/wrangler/configuration/
  - label: Bindningar
    wert: Capability och API för plattformsresurser utan åtkomstnycklar som exponeras i koden
    href: https://developers.cloudflare.com/workers/runtime-apis/bindings/
  - label: Tillståndsmodeller
    wert: lokal cache · eventual-consistent KV · D1 SQL · starkt konsistent Durable Object · R2-objekt · kö
    href: https://developers.cloudflare.com/workers/platform/storage-options/
  - label: Kompatibilitet
    wert: Compatibility Date plus valfria flaggor fryser körtidsändringar som påverkar beteendet per driftsättning
    href: https://developers.cloudflare.com/workers/configuration/compatibility-flags/
  - label: Control Plane
    wert: Wrangler-konfiguration, Workers API, versionsobjekt, driftsättningar, hemligheter och resursbindningar
    href: https://developers.cloudflare.com/workers/configuration/
  - label: Utrullning
    wert: ladda upp version, skapa driftsättning, fördela trafik stegvis och återgå till stabil version
    href: https://developers.cloudflare.com/workers/versions-and-deployments/gradual-deployments/
  - label: Driftssignaler
    wert: Invocation Outcome, status, exception, CPU- och Wall Time, subrequests, loggar, traces och versions-ID
    href: https://developers.cloudflare.com/workers/observability/logs/workers-logs/
  - label: Återställningsobjekt
    wert: kodartefakt, konfiguration, Compatibility Date, bindningar, hemligheter, dataexporter, DO-livscykel och testad rollback
    href: https://developers.cloudflare.com/workers/versions-and-deployments/
werbung:
  - newsletter
ctaThemen:
  - cloudflare-workers
translationSourceHash: 4c5445ed6d4ead36638b1de03ed9d67a4f82304b58f28d83a9c72e29e760d065
translationModel: gpt-5.6-terra
translatedAt: 2026-10-11T10:17:32.968Z
translationReview: automatic
---

# Cloudflare Workers: Isolates, bindningar och edge-drift

Cloudflare Workers är en händelsedriven compute-plattform inom Cloudflare-nätverket. En Worker kan besvara eller vidarebefordra HTTP-förfrågningar, köra schemalagda uppgifter, konsumera kömeddelanden, behandla inkommande e-post och fungera som intern tjänst för andra Workers. Operatören hanterar då ingen lyssnarprocess och ingen enskild VM. I stället hanterar denne kod, entrypoints, rutter, runtime-kompatibilitet, behörigheter till plattformsresurser, versioner och driftdata.

Ordet «Edge» beskriver bara en möjlig körplats, inte hela arkitekturen. En HTTP-Worker kan köras nära användaren; Smart Placement kan flytta compute närmare en backend; Durable Objects har en entydig hemvist för tillstånd; D1, KV, R2 och Queues har egna replikerings- och konsistensmodeller. För administratörer är därför inte «serverless» den avgörande egenskapen, utan uppdelningen mellan datapath, Control Plane, tillståndstjänster och möjliga felpunkter.

Förklaringen följer en request från den offentliga ändpunkten till Workers-körtiden och vidare till bindningar, lagring och upstreams. Därefter sätts deployment, säkerhet, observability och återställning in i plattformsdriftens perspektiv.

## Arkitektur: datapath, runtime och Control Plane

Ett Workers-system består av flera lager:

1. **Ingress och routing:** Cloudflare tar emot en förfrågan via DNS, TLS och HTTP och tilldelar ett host- eller path-mönster till en Worker eller statisk asset.
2. **Händelsedispatch:** Plattformen skapar en Fetch-, Scheduled-, Queue-, E-Mail-, Alarm- eller Tail-händelse och anropar lämplig entrypoint.
3. **Runtime:** `workerd` tillhandahåller V8-isolat, webb-API:er, Node.js-kompatibilitet och runtimegränser.
4. **Bindningar:** Objektet `env` ger koden capabilities för KV, D1, R2, Durable Objects, Queues, Secrets, Assets, andra Workers och ytterligare plattformstjänster.
5. **Tillstånd och upstreams:** Data ligger utanför det flyktiga isolatet eller i kontrollerade Durable Object-lagringar; utgående anrop använder Fetch, Service Bindings eller stödda Socket-API:er.
6. **Control Plane:** Wrangler, Dashboard och API hanterar konfiguration, resursrelationer, hemligheter, versioner, driftsättningar och observability.

Cloudflare beskriver isolat, requestrelaterad beräkningstid och distribuerad körning som de tre viktigaste skillnaderna jämfört med klassiska serverruntimer ([How Workers works](https://developers.cloudflare.com/workers/reference/how-workers-works/)). Plattformreferensen skiljer själva Workern från de anslutna Storage- och Developer Platform-produkterna ([Cloudflare Workers documentation](https://developers.cloudflare.com/workers/)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-cloudflare-workers.svg?v=20260813" title="Interaktive Infografik: Cloudflare-Workers-Aufruf von DNS, TLS und Routing über Event Dispatch, workerd und V8-Isolate bis zu Bindings, Storage, Upstreams, Observability, Versionen und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-cloudflare-workers.svg?v=20260813">Öppna interaktiv grafik direkt</a>.
</iframe>

## Requestväg och körplats

Vid en HTTP-förfrågan avgör först Cloudflare-konfigurationen om ett `workers.dev`-hostnamn, en route eller en Custom Domain utlöser Workern. En route ligger framför en befintlig origin och kan ändra, besvara eller vidarebefordra förfrågningar. En Custom Domain ansluter en Worker direkt till ett hostnamn. Statiska assets kan utvärderas före eller efter Workern; `run_worker_first` ändrar denna ordning ([Wrangler configuration – routes](https://developers.cloudflare.com/workers/wrangler/configuration/), [Static Assets – configuration and bindings](https://developers.cloudflare.com/workers/static-assets/binding/)).

Den externa vägen kan modelleras som `Client → DNS → Cloudflare Edge → TLS/HTTP → Route → Worker/Asset → Binding oder Origin`. Anslutningen till origin är en ny transportkontext. Klientcertifikat, headern `Authorization`, cachekey, hostheader och käll-IP måste därför hanteras medvetet vid varje gräns. En Service Binding kringgår däremot offentlig namnupplösning och tillhandahåller en uttryckligen behörig Worker-till-Worker-väg ([Service bindings](https://developers.cloudflare.com/workers/runtime-apis/bindings/service-bindings/)).

Smart Placement kan köra en Worker närmare dess backends i stället för att enbart prioritera närhet till användaren. Det är användbart vid databasintensiva anrop, men kan för assets eller rena edge-transformeringar förlänga fel väg. Cloudflare dokumenterar därför olika rekommendationer för Assets-first och Worker-first ([Placement](https://developers.cloudflare.com/workers/configuration/placement/)).

## workerd och V8-isolat

`workerd` är den serverorienterade JavaScript-/WebAssembly-runtime som ligger bakom Workers. V8 tillhandahåller isolat som separata exekveringskontexter. Många isolat kan dela en process utan att varje applikation behöver en egen JavaScript-process eller container. Webb-API:er tillhandahålls i stor utsträckning direkt av runtime ([Introducing workerd](https://blog.cloudflare.com/workerd-open-source-workers-runtime/), [Workers security model](https://developers.cloudflare.com/workers/reference/security-model/)).

Ett isolat är ingen beständig maskin. Det kan återanvändas, parallellt belastas med flera requests eller tas bort. Två efterföljande requests behöver inte träffa samma instans eller plats. Globala objekt lämpar sig för oföränderliga, kostsamma hjälpkonstruktioner eller opportunistiska cacher, men inte som sanningskälla. All korrekthet som beror på en global variabel mellan requests är ett arkitekturfel.

V8-isolering ersätter inte alla säkerhetskontroller. Cloudflare kompletterar med runtime- och processkydd, resursgränser och särskilt Spectre-skydd. Operatören ansvarar fortfarande för indatavalidering, auktorisering, Secret-scope, måltillåtelselistor, utdata-gränser och dataklassificering. Isolatgränsen skyddar inte mot en applikation som läser eller skriver fel data med sina legitima bindningar.

## Händelsemodell och handlers

Workers implementerar entrypoints för olika händelsetyper. Den allmänna handlerkatalogen omfattar Fetch, Scheduled, Queue, E-Mail, Alarm och Tail ([Handlers](https://developers.cloudflare.com/workers/runtime-apis/handlers/)).

| Handler | Utlösare | Retur- och felgräns | Typisk adminfråga |
|---|---|---|---|
| `fetch()` | HTTP-förfrågan eller serviceanrop | `Response`, stream eller exception | Vilken route och version behandlade requesten? |
| `scheduled()` | Cron Trigger | Promise/Invocation Outcome | Utlöstes och avslutades den schemalagda körningen? |
| `queue()` | Meddelandebatch | Ack, Retry eller Dead Letter-beteende | Vilket meddelande är idempotent, retriable eller poison? |
| `email()` | Email Routing | Leverera, avvisa, vidarebefordra | Vilken envelope- och policygräns gäller? |
| `alarm()` | Durable Object-alarm | objektlokal exekvering | Vilket objekt äger alarm och tillstånd? |
| `tail()` | Trace-händelse från en Producer Worker | asynkron export | Kan observability själv fallera eller bli rekursiv? |

Fetch-handlern får `Request`, `env` och `ctx` och returnerar ett Web API-`Response` ([Fetch handler](https://developers.cloudflare.com/workers/runtime-apis/handlers/fetch/)). `ctx.waitUntil()` registrerar arbete som får fortsätta efter att svaret har skickats; det är bundet till dokumenterade runtime- och felgränser och är ingen långlivad jobbserver ([Context – waitUntil](https://developers.cloudflare.com/workers/runtime-apis/context/)). För varaktiga, repeterbara flöden är Queues eller Workflows lämpligare än en kedja av oövervakade Background Promises.

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

Exemplet använder en Service Binding i stället för en fritt konfigurerad URL. Därmed flyttas nåbarhet och behörighet till deploymentkonfigurationen, medan koden bara ser en `Fetcher`-capability. Runtime-API:erna är webborienterade och omfattar bland annat Fetch, Streams, Web Crypto, WebSockets, HTMLRewriter och TCP Sockets ([Runtime APIs](https://developers.cloudflare.com/workers/runtime-apis/)).

Koden körs inte i en fritt vald lokal Node-miljö, utan mot ett versionshanterat runtimekontrakt. Compatibility Date avgör vilka beteendeförändringar som gäller för en driftsättning.

## Compatibility Date och runtimekontrakt

Ett Compatibility Date är varken en releaseinformation eller ett påstående om artikelns «aktuella status». Det är en del av runtimekontraktet för en konkret Worker-driftsättning. Ändringar som skulle kunna bryta befintligt beteende aktiveras efter datum eller flagga. Ett äldre datum bevarar kompatibelt beteende; ett nyare datum måste testas och rullas ut som en beroendeuppdatering ([Compatibility flags](https://developers.cloudflare.com/workers/configuration/compatibility-flags/)).

Administratörer behandlar därför dessa fält tillsammans:

- Kodcommit och bundlat artefakt;
- Compatibility Date och explicita flaggor;
- Wrangler- och buildverktygskedja;
- bindningsdefinitioner, routes och placement;
- versions-ID och deploymentvikt;
- data- eller Durable Object-migreringar.

Node.js-kompatibilitet är opt-in och implementerar bara en dokumenterad del av Node-API:erna. Vissa moduler är kompletta, andra delvis tillgängliga eller enbart tillgängliga som import-stub. `nodejs_compat` får därför inte likställas med «fullständig Node-runtime» ([Node.js compatibility](https://developers.cloudflare.com/workers/runtime-apis/nodejs/)).

## Språk och teknikstack

Cloudflare dokumenterar JavaScript, TypeScript, Python och Rust som förstklassiga språkvägar; WebAssembly öppnar för ytterligare källspråk ([Languages](https://developers.cloudflare.com/workers/languages/)). Den resulterande teknikstacken skiljer mellan utveckling och exekvering:

| Lager | Typisk teknik | Driftsrelevans |
|---|---|---|
| Källkod | TypeScript/JavaScript, Python, Rust | Språktoolchain, beroenden, tester |
| Build | Wrangler/esbuild eller frameworkadapter | Bundle, Source Maps, externals, deterministiska builds |
| Runtime | workerd, V8, webb- och valfria Node-API:er | Compatibility Date, CPU/Memory, Event Loop |
| Capability | `env`-bindningar | Least Privilege, Resource-ID, miljöseparering |
| Tillstånd | Cache, KV, D1, R2, Durable Objects, Queues | Konsistens, datahemvist, backup, recovery |
| Ingress | Route, Custom Domain, workers.dev, triggers | DNS, TLS, ordning, fail-open/closed |
| Control Plane | Wrangler, API, Dashboard, CI/CD | Autentisering, versionering, rollout, audit |

Rust integreras via `workers-rs` och WebAssembly; den importerade Wasm-modulen förblir en del av Workerartefaktet ([Rust language support](https://developers.cloudflare.com/workers/languages/rust/)). Python Workers använder egna entrypointklasser och en runtimeintegration, inte en fritt administrerbar CPython-process.

## Konfiguration och Wrangler

Wrangler-konfigurationen är den deklarativa kopplingen mellan kod och plattform. Den definierar minst namn, entrypoint, Compatibility Date, routes och bindningar. Named Environments kan ha avvikande värden; bindningar måste granskas medvetet per miljö. Cloudflare dokumenterar fälten och arv i Wrangler-schemat ([Workers configuration](https://developers.cloudflare.com/workers/configuration/), [Wrangler configuration reference](https://developers.cloudflare.com/workers/wrangler/configuration/)).

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

[`npx`](https://docs.npmjs.com/cli/commands/npx) startar den projektlokalt fixerade CLI:n; [`Wrangler Commands`](https://developers.cloudflare.com/workers/wrangler/commands/) dokumenterar autentisering, typgenerering, deployment samt versions- och loggkommandon. `wrangler types` skapar typer från den faktiska bindningskonfigurationen. `deploy --dry-run` granskar bundle och konfiguration, men utgör inget bevis för fungerande fjärrresurser eller korrekt auktorisering.

När runtime och build är klargjorda uppstår åtkomstfrågan. En Worker når databaser, köer, hemligheter och andra Workers via deklarerade bindningar i stället för fritt distribuerade åtkomstuppgifter.

## Bindningar som capability-modell

En bindning är både behörighet och runtime-API. Workern får exempelvis `env.ARCHIVE` som R2-bucket, `env.DB` som D1-databas eller `env.AUTH` som intern tjänst. Den underliggande plattformscredentialen exponeras inte för applikationskoden som en återanvändbar API-nyckel ([Bindings](https://developers.cloudflare.com/workers/runtime-apis/bindings/)).

Capability-modellen flyttar den centrala adminfrågan från «Vilken hemlighet finns i miljövariabeln?» till «Vilken resurs är, under vilket namn och i vilken miljö, bunden till vilken version?». Fel bucket-ID eller Service Binding-version kan trots identisk kod orsaka dataförlust eller åtkomst mellan miljöer.

Bindningar bör separeras efter uppgift och miljö. Produktions- och testworkers delar inga skrivbara buckets, köer eller databaser, om detta inte uttryckligen är en del av ett kontrollerat integrationstest. Generiska namn som `DB` räcker i koden om deploymenten binder den faktiska resursen entydigt och verifierbart.

### Hemligheter och variabler

Vanliga variabler är konfiguration, inte hemligheter. Secrets är krypterade textbindningar och hanteras utanför källkoden. Obligatoriska Secret-namn kan deklareras utan att deras värden skrivs till konfigurationsfilen ([Environment variables](https://developers.cloudflare.com/workers/configuration/environment-variables/), [Secrets](https://developers.cloudflare.com/workers/configuration/secrets/)).

Ett Secret-byte är ett tillståndsbyte för applikationen. Klienter som globalt härleds från `env` kan behålla ett gammalt värde i återanvända isolat; Cloudflare rekommenderar att sådana härledningar skapas requestbaserat. Rotation omfattar därför att sätta värdet, kontrollera deployment och bindning, parallell giltighet om målsystemet kräver det, telemetri och kontrollerad borttagning.

## Service Bindings och Worker-till-Worker-arkitektur

Service Bindings kopplar samman Workers uttryckligen utan att behöva en offentligt adresserbar HTTP-slutpunkt. Den anropade tjänsten kan exponeras via Fetch eller RPC. Det minskar nätverks- och credentialkonfiguration, men eliminerar inte versions- och kontraktsproblem. Vid separata rollouts kan caller och callee använda olika API-versioner ([Service bindings](https://developers.cloudflare.com/workers/runtime-apis/bindings/service-bindings/), [Gradual deployments](https://developers.cloudflare.com/workers/versions-and-deployments/gradual-deployments/)).

Administratörer dokumenterar per bindning:

- Caller och måltjänst;
- miljö och målversion respektive rolloutstrategi;
- offentlig nåbarhet för målworkern;
- timeout-, fel- och retrysemantik;
- vidarebefordran av request-ID och tracekorrelation;
- kontraktskompatibilitet vid oberoende deployments.

En kedja av många Workers fördelar en request över fler möjliga felpunkter. Mindre offentlig angreppsyta är värdefullt; en alltför fin uppdelning kan däremot öka subrequestdjup, felsökningsarbete och versionsskew.

## Tillståndstjänster: modell före produktnamn

Isolatets heap är flyktigt. Beständigt tillstånd ligger i bundna tjänster vars semantik skiljer sig avsevärt. Cloudflares Storage-översikt placerar produkterna utifrån åtkomstmönster och konsistens ([Choosing a data or storage product](https://developers.cloudflare.com/workers/platform/storage-options/)).

| Tjänst | Modell | Konsistens-/platsgräns | Lämplig för | Kritisk adminfråga |
|---|---|---|---|---|
| Cache API | HTTP-response-cache | lokal för datacentret, efemär | återkommande responses | Är cachekeyn komplett och säker? |
| Workers KV | globalt läsbar key-value-store | eventual consistent, Read Cache | konfiguration, read-heavy-data | Tål arbetsflödet gamla eller negativa reads? |
| D1 | hanterad SQL baserad på SQLite | databas- och sessionsrelaterad semantik | relationella applikationstabeller | Vilken replikat-/sessionsgaranti kräver readen? |
| Durable Objects | entydig objektinstans plus storage | starkt konsistent och serialiserbar per objekt | koordinering, locks, sessioner, WebSockets | Är partition key korrekt vald? |
| R2 | S3-kompatibel objektlagring | starkt konsistent per objekt | blobs, arkiv, stora payloads | Hur versionshanteras objekt, metadata och index tillsammans? |
| Queues | asynkrona meddelanden | minst-en-gång-orienterad behandling och retrypolicy | frikoppling, lastutjämning | Är konsumenten idempotent och finns en DLQ? |

### Cache API

Cache API arbetar lokalt i det datacenter där Workern behandlar förfrågan. `cache.put()` är därför ingen global replikering och ingen beständig databasskrivning. `caches.default` delar standardcachekontexten; namngivna cacher skapar namespaces, men inte heller global konsistens ([How the Cache works](https://developers.cloudflare.com/workers/reference/how-the-cache-works/)).

Personliga svar får endast cachas med en key som avbildar alla relevanta identitets- och variantattribut. `Authorization`, cookies, språk, encoding och klienttillhörighet är typiska läckagegränser. Cache-purge och originvalidering hör till recovery, inte bara TTL:er.

### Workers KV

KV replikerar data via centrala stores och edge-cacher. Ändringar kan bli synliga fördröjt på andra platser; även «hittades inte» kan vara cachat. Cloudflare nämner för avlägsna platser fördröjningar på upp till 60 sekunder eller mer och rekommenderar inte KV för atomära read-modify-write-transaktioner ([How KV works](https://developers.cloudflare.com/kv/concepts/how-kv-works/)).

KV lämpar sig för främst läst konfiguration, allow-/deny-listor eller cacher när staleness uttryckligen tolereras. Globala rate limits, entydiga sekvenser eller omedelbar credential-revocation hör inte hemma i KV utan ytterligare koordinering.

### Durable Objects

Ett Durable Object kopplar ett globalt entydigt objekt-ID till seriell koordinering och privat lagring. Requests för samma ID routas till samma logiska instans. Nya namespaces använder SQLite-storagevägen; den ger transaktionell, starkt konsistent lagring och Point-in-Time-Recovery-funktioner ([Durable Objects overview](https://developers.cloudflare.com/durable-objects/), [SQLite-backed Durable Object Storage](https://developers.cloudflare.com/durable-objects/api/sqlite-storage-api/)).

Inte heller ett Durable Object är en ständigt körande server. In-memory-tillstånd kan försvinna vid hibernation eller eviction och måste vid behov rekonstrueras från storage. Det finns ingen tillförlitlig shutdown hook ([Durable Object lifecycle](https://developers.cloudflare.com/durable-objects/concepts/durable-object-lifecycle/)). Ändringar av klasser och storageallokering är livscykeloperationer; de behandlas som datamigreringar ([Durable Object class lifecycle](https://developers.cloudflare.com/durable-objects/reference/durable-objects-migrations/)).

### D1, R2 och Queues

D1 tillhandahåller SQL för relationella data. R2 lagrar stora ostrukturerade objekt och exponerar ett S3-kompatibelt API utanför respektive ett binding-API inom Workers. Queues frikopplar producer och consumer. Dessa tjänster ersätter inte varandra: Ett R2-objekt kan bära payloaden, D1 indexet, en queue utlösa bearbetningen och ett Durable Object koordinera konkurrerande ändringar.

Gemensam commit av dessa fyra tillstånd är inte automatiskt atomär. Administratörer planerar Idempotency Keys, Outbox-/Inbox-mönster, reconciliation och Dead Letter-behandling. Produktdokumentation och gränser för de enskilda tjänsterna för [D1](https://developers.cloudflare.com/d1/), [R2](https://developers.cloudflare.com/r2/) och [Queues](https://developers.cloudflare.com/queues/) ingår i runbooken.

## Statiska assets och full-stack-Workers

En Worker kan leverera statiska assets tillsammans med kod. Assets Binding gör det möjligt för koden att explicit hämta en asset. Routingalternativet avgör om befintliga assets kringgår Workern eller om Workern körs först ([Static Assets](https://developers.cloudflare.com/workers/static-assets/)).

Ordningen är ett säkerhetsbeslut. Med Assets-first får en slumpmässigt befintlig filsökväg inte kringgå autentisering. Med Worker-first ökar varje assetrequest compute- och beroendebelastningen. Frameworkadaptrar döljer delvis denna ordning; det byggda deploymentartefaktet och Wrangler-konfigurationen förblir den avgörande sanningen.

## Nätverks- och protokollperspektiv

Workers ligger i applikationsvägen ovanför DNS, TLS och HTTP. Plattformen terminerar den externa transporten; Workern behandlar Web API-requests. Utgående `fetch()` skapar ett nytt HTTP-anrop. TCP Sockets tillåter utvalda outbound-protokoll, men förvandlar inte Workern till en allmänt nåbar TCP-server ([Workers protocols](https://developers.cloudflare.com/workers/reference/protocols/), [TCP sockets](https://developers.cloudflare.com/workers/runtime-apis/tcp-sockets/)).

För meddelandeadministratörer är tre gränser viktiga:

- En e-posthandler utlöses av Cloudflare Email Routing; den lyssnar inte själv på [SMTP](/kb/smtp).
- En Worker som anropar ett mail-API måste hantera OAuth-audience, tokenlivscykel och retries som varje annan API-klient.
- DNS-, TLS- och HTTP-fel ligger före entrypointen; binding-, auth- och applikationsfel efter den. En timeout för `fetch()` säger utan korrelation inte vilken fas som fallerade.

## Säkerhets- och förtroendegränser

Säkerhetsmodellen har minst fem separata förtroendegränser:

1. **Offentlig request:** godtyckliga headers, bodystorlek, metod och identitet.
2. **Cloudflare-konfiguration:** Zone, Route, WAF, Access, certifikat och fail-open/closed.
3. **Workerartefakt:** kod, beroenden, Source Maps, Compatibility Date och supply chain.
4. **Bindningar:** konkreta resursbehörigheter, Secrets och interna servicevägar.
5. **Upstreams och lagrade data:** egen auktorisering, konsistens och recovery.

Least Privilege uppnås främst genom små bindningsytor och separata miljöer. En Worker med skrivåtkomst till alla buckets och databaser förblir en högprivilegierad principal, även om den inte har synliga åtkomstnycklar. Workers säkerhetsdokumentation förklarar runtimeisolering; den ersätter ingen applikationssäkerhetsarkitektur ([Security model](https://developers.cloudflare.com/workers/reference/security-model/)).

Supply-chain-kontroller omfattar fixerade paketversioner, lockfile, reproducerbar build, dependency-scanning, secret-scanning, minimala uppladdningsbehörigheter och skyddade deploymentmiljöer. Den skapade bundlen granskas före uppladdning; enbart källkodsgranskning räcker inte om bundler eller framework bäddar in ytterligare moduler.

Edge-närhet upphäver inte resursgränser. CPU-tid, subrequests, minne och plattformsgränser måste beaktas redan i design och lastmodell.

## Gränser som arkitekturparametrar

Workers begränsar bland annat CPU-tid, minne, startup-tid, subrequests, bundlestorlek, loggvolym, antal routes och statiska assets. Värden skiljer sig mellan planer och invocationtyper och kan ändras. Därför hör ingen kopierad gränstabell hemma i en statisk artikel som påstått tidlös sanning; den officiella sidan [Workers Limits](https://developers.cloudflare.com/workers/platform/limits/) är driftkällan.

Administratörer skiljer mellan:

- **CPU Time:** aktiv beräkningstid; väntan på I/O räknas inte på samma sätt som CPU.
- **Wall Time:** händelsens förflutna varaktighet; regler skiljer mellan Fetch, Cron, Queue och Durable Object.
- **Startup Time:** modulevaluation och initialisering före handlern.
- **Subrequests:** Fetch- och plattformsoperationer per invocation.
- **Memory:** heap, streams, buffertar och bibliotekstillstånd i isolatet.
- **Log Budget:** för stora eller hemliga payloads är både ett kostnads- och dataskyddsproblem.

Ett fel `1102` indikerar överskridna resursgränser; `10021` kan inträffa vid alltför kostsam startup-initialisering. Felnumret är en ingångspunkt, inte en grundorsak. CPU-profil, invocationtyp, version, inputklass och subrequestgraf måste analyseras tillsammans.

## Lokal utveckling och tester

Lokal utveckling kör kod med `workerd` via Miniflare. Bindningar simuleras som standard lokalt; enskilda Remote Bindings kan riktat anslutas till riktiga plattformsresurser. Helt fjärrkört `wrangler dev --remote` finns kvar för nätverksspecifika fall, men är inte längre standardvägen ([Local development](https://developers.cloudflare.com/workers/local-development/), [Bindings per development mode](https://developers.cloudflare.com/workers/local-development/bindings-per-env/)).

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

[`Invoke-WebRequest`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest) och [`curl`](https://curl.se/docs/manpage.html) kontrollerar status och headers för den lokala HTTP-vägen. Workers-Vitest-integrationen kör tester inom `workerd` och tillhandahåller hjälpare för bindningar, requests och Durable Objects ([Vitest integration](https://developers.cloudflare.com/workers/testing/vitest-integration/), [Workers test APIs](https://developers.cloudflare.com/workers/testing/vitest-integration/test-apis/)).

Ett godkänt unit test bevisar inte korrekt route, fjärrbehörighet eller datamigrering. Testpyramiden omfattar ren logik, runtimeintegration, bindingsemantik, stagingroute, kontrollerat production-smoke-test och recoveryövning.

## Deployment, version och rollout

En **version** är ett oföränderligt kod-/konfigurationsobjekt. En **deployment** fördelar trafik på en eller flera versioner. Gradual Deployments flyttar vikter stegvis och möjliggör observation samt återgång till en stabil version ([Versions and deployments](https://developers.cloudflare.com/workers/versions-and-deployments/), [Gradual deployments](https://developers.cloudflare.com/workers/versions-and-deployments/gradual-deployments/)).

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

Före en rollout sparas tilldelningen `Git commit → Bundlehash → Worker-Version → Deploymentgewicht`. Vid flera Service Bindings ingår versionsskew i testningen. Durable Objects kräver särskilda migrations- och rolloutregler, eftersom godtyckliga klassversioner inte kan vara aktiva samtidigt för ett objekt-ID.

Rollback återställer kod och konfiguration, men inte automatiskt externa data. En ny Worker kan ha skrivit data i ett format som den gamla inte förstår. Schemaändringar använder Expand/Contract, framåtkompatibilitet eller en separat testad dataåterföring. «Rollback möjlig» är först bevisat när kod-, binding- och datasidan har kontrollerats tillsammans.

## Observability och driftbevis

Workers Logs samlar Invocation Logs, egna loggar, fel och obehandlade exceptions. Tail Workers eller Logpush kan exportera händelser; OpenTelemetry-mål erbjuder en mer direkt exportväg för loggar och traces ([Workers Logs](https://developers.cloudflare.com/workers/observability/logs/workers-logs/), [Tail Workers](https://developers.cloudflare.com/workers/observability/logs/tail-workers/), [Traces](https://developers.cloudflare.com/workers/observability/traces/)).

Varje invocation bör minst möjliggöra korrelation av:

- Workernamn, versions-ID och miljö;
- händelsetyp och route;
- request-/message-/correlation-ID;
- resultat, HTTP-status eller Ack/Retry;
- CPU- och Wall Time;
- subrequestmål och latens utan hemligheter;
- binding-/resursklass, inte känsligt fullständigt innehåll;
- verksamhetsmässig Idempotency Key vid asynkron bearbetning.

`console.log()` är inget obegränsat bevisarkiv. Logggränser, sampling och redaction påverkar synligheten. Credentials, Authorization Headers, cookies, fullständigt e-postinnehåll och personuppgiftspayloads loggas inte. Övervakningen bevakar även exportvägen: En felaktig Tail Worker får inte omöjliggöra diagnostik av producerfelet.

Diagnostiken följer publiceringsvägen: Route och DNS, aktiv version, runtime och bindningar, utgående beroenden och slutligen loggar samt traces.

## Admin-diagnostik efter faser

Workers-fel separeras effektivt längs exekveringsvägen:

1. **DNS:** Löses det förväntade hostnamnet till Cloudflare och är zonen aktiv?
2. **TLS/HTTP:** Kontrollera certifikat, SNI, protokoll och status.
3. **Route/Asset:** Träffar host/path Workern, en asset eller origin?
4. **Version:** Vilken version och vilken deploymentvikt behandlade requesten?
5. **Runtime:** Kontrollera startup, CPU, memory, exception och handlerutfall.
6. **Binding:** Finns resursen i denna miljö och har den förväntad capability?
7. **Upstream/Storage:** Undersök timeout, auth, konsistens, datatillstånd och retry.
8. **Recovery:** Bekräfta stabil version, datakompatibilitet och reconciliation.

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

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname), [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection), [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility), [`nc`](https://man.openbsd.org/nc), [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) och [`curl`](https://curl.se/docs/manpage.html) separerar DNS, TCP, TLS och HTTP före runtime. [DNS](/kb/dns), [TCP](/kb/tcp), [TLS](/kb/tls), [APIs](/kb/apis) och [Troubleshooting](/kb/troubleshooting) behandlar dessa lager mer ingående.

### Typiska felbilder

| Symtom | Sannolik fas | Bevis | Vanligt tankefel |
|---|---|---|---|
| Origin-svar i stället för Worker-svar | Route/Asset/Fail-open | Route, host, path, responsemarkör | «Deployment lyckades» betyder «route aktiv» |
| 1101 eller exception | Applikation/binding | Workers Logs, version, stack | bara deploya igen i stället för att kontrollera inputklass |
| 1102 | Resursgräns | CPU/Wall Time, profil, invocationtyp | likställa I/O-väntetid med CPU |
| sporadiskt gammalt KV-värde | KV-konsistens | Key, plats, cache-TTL, skrivtid | behandla KV som global transaktion |
| lokalt grönt, fjärr-bindingfel | Environment/Capability | konkret Resource-ID och bindingnamn | simulering bevisar IAM-/resurstillstånd |
| endast deltrafik felaktig | Gradual Deployment | versions-ID och vikt | aggregera metrik utan versionsdimension |
| rollback åtgärdar kod, inte data | Schema-/tillståndsgräns | skrivet format, migrering, reconciliation | deployment är fullständig recovery |

## Backup och Disaster Recovery

Serverless eliminerar inte backup. Det flyttar de skyddsvärda objekten:

- Git-repository, lockfile och reproducerbar builddefinition;
- Wrangler-konfiguration, routes, Compatibility Date och flaggor;
- inventering av alla bindningar och målresurser;
- Secret-värden eller deras externa Source of Truth och rotationsprocess;
- dataexporter och restoreprocedurer för D1, R2, KV och externa system;
- Durable Object-klasser, lifecycle-/migreringsstatus och vid behov PITR;
- Queue-/DLQ-tillstånd, Idempotency Keys och reconciliation;
- version-/deploymenthistorik samt en testad rollbackväg.

Alla produkter har inte samma export- och restoresemantik. En R2-objektbackup skyddar inte D1-indexet; en D1-restore återställer inte ett redan bekräftat kömeddelande; en rollbackad Worker kan feltolka nya objektformat. [Backup och Disaster Recovery](/kb/backup-dr) måste därför definiera en konsekvent applikationspunkt i stället för enbart enskilda produktkopior.

Recovery testas i scenarier: fel Secret, borttagen route, felaktig version, inkompatibelt D1-schema, förlorat R2-objekt, poison Queue Message, felaktig Durable Object-klass och onåbar upstream. RTO och RPO fastställs per tillståndstjänst och för den sammansatta applikationen.

## Teknisk historia

Cloudflare presenterade Workers 2017 som programmerbar exekvering i det globala nätverket. Utgångspunkten var latensgränsen för centrala datacenter och idén att köra kod nära datapathen ([Code Everywhere: Why We Built Cloudflare Workers](https://blog.cloudflare.com/code-everywhere-cloudflare-workers/)).

Arkitekturen använde tidigt V8-isolat i stället för en container eller process per funktion. Cloudflare beskrev 2018 de ekonomiska och tekniska skillnaderna i denna modell, men också begränsningen till JavaScript respektive språk med WebAssembly-stöd ([Cloud Computing without Containers](https://blog.cloudflare.com/cloud-computing-without-containers/)).

Workers KV lade till globalt läsbart, eventual-consistent tillstånd. Durable Objects tillkännagavs 2020 som motmodell för koordinerade, starkt konsistenta objekttillstånd ([Introducing Workers Durable Objects](https://blog.cloudflare.com/introducing-workers-durable-objects/)). Därmed utvecklades Workers från ren requesttransformering till en applikationsplattform med flera explicita konsistensmodeller.

2022 publicerade Cloudflare `workerd` under Apache 2.0. Den öppna runtime delar kod med produktionssystemet och förbättrade precisionen i lokal utveckling; den är dock inte hela Cloudflare-plattformen med dess routing, orkestrering och säkerhetsdrift ([Introducing workerd](https://blog.cloudflare.com/workerd-open-source-workers-runtime/), [workerd repository](https://github.com/cloudflare/workerd)).

Senare utbyggnader gav ES-modulentrypoints, Compatibility Dates, Service/RPC-bindningar, D1, R2, Queues, Workflows, Python, utökad Node-kompatibilitet, versionsobjekt och stegvisa deployments. Denna historia förklarar dagens kärna: Workers är varken browser-JavaScript på ett CDN eller en godtycklig Linuxserver, utan en händelsedriven, capabilitybaserad runtime med separata data- och Control Plane-produkter.

## Admin-checklista

Före produktionsdrift är minst följande punkter belagda:

- Route, Custom Domain, assetordning och fail-open/closed är dokumenterade.
- Compatibility Date, flaggor, Wrangler-version, bundle och Source Maps är reproducerbara.
- Varje binding har owner, environment, Resource-ID, rättigheter och recoveryväg.
- Cache-, KV-, D1-, R2-, Durable Object- och Queue-semantik blandas inte.
- Secrets finns utanför repositoryt, kan roteras och syns inte i loggar.
- Timeouts, retries, idempotens och Dead Letter-beteende är definierade per händelsetyp.
- Loggar och traces innehåller version och Correlation-ID, men inga känsliga payloads.
- Gradual Deployment och rollback tar hänsyn till Service Binding-versioner och dataformat.
- Gränser övervakas från den officiella plattformssidan, inte från en kopierad tabell.
- Backup, restore och reconciliation har testats för den sammansatta applikationen.

## Källor

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
