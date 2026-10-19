---
title: "Cloudflare Workers: Isolates, binding e funzionamento edge"
blatt: "cloudflare-workers"
description: "Cloudflare Workers per amministratori di infrastruttura, messaggistica e piattaforme: percorso delle richieste, workerd e V8 Isolates, handler e API Web, binding di capability, KV, D1, R2 e Durable Objects, routing, Compatibility Dates, limiti, osservabilità, deployment, ripristino e storia tecnica."
fakten:
  - label: Ruolo del sistema
    wert: piattaforma di calcolo guidata dagli eventi nella rete Cloudflare per HTTP, pianificazioni, code, e-mail e servizi interni
    href: https://developers.cloudflare.com/workers/runtime-apis/handlers/
  - label: Runtime
    wert: workerd con V8 Isolates; nessun processo server persistente proprio e nessuna istanza di container per applicazione
    href: https://developers.cloudflare.com/workers/reference/how-workers-works/
  - label: Modello di programmazione
    wert: entry point dei moduli ES con API standard Web quali Request, Response, Fetch, Streams e Web Crypto
    href: https://developers.cloudflare.com/workers/runtime-apis/
  - label: Linguaggi
    wert: JavaScript, TypeScript, Python e Rust; altri linguaggi tramite WebAssembly
    href: https://developers.cloudflare.com/workers/languages/
  - label: Percorsi di ingresso
    wert: workers.dev, Route, Custom Domain nonché eventi Fetch, Scheduled, Queue, E-Mail, Alarm e Tail
    href: https://developers.cloudflare.com/workers/wrangler/configuration/
  - label: Binding
    wert: capability e API per risorse della piattaforma senza chiavi di accesso esposte nel codice
    href: https://developers.cloudflare.com/workers/runtime-apis/bindings/
  - label: Modelli di stato
    wert: cache locale · KV eventual-consistent · D1 SQL · Durable Object fortemente consistente · oggetto R2 · coda
    href: https://developers.cloudflare.com/workers/platform/storage-options/
  - label: Compatibilità
    wert: Compatibility Date più flag opzionali fissano le modifiche di runtime rilevanti per il comportamento per ogni deployment
    href: https://developers.cloudflare.com/workers/configuration/compatibility-flags/
  - label: Control Plane
    wert: configurazione Wrangler, Workers API, oggetti versione, deployment, segreti e binding delle risorse
    href: https://developers.cloudflare.com/workers/configuration/
  - label: Rollout
    wert: caricare la versione, creare il deployment, distribuire gradualmente il traffico e ripristinare una versione stabile
    href: https://developers.cloudflare.com/workers/versions-and-deployments/gradual-deployments/
  - label: Segnali operativi
    wert: esito dell'invocazione, stato, eccezione, CPU e Wall Time, subrequest, log, trace e ID versione
    href: https://developers.cloudflare.com/workers/observability/logs/workers-logs/
  - label: Oggetti di ripristino
    wert: artefatto del codice, configurazione, Compatibility Date, binding, segreti, esportazioni di dati, ciclo di vita DO e rollback testato
    href: https://developers.cloudflare.com/workers/versions-and-deployments/
werbung:
  - newsletter
ctaThemen:
  - cloudflare-workers
translationSourceHash: 4c5445ed6d4ead36638b1de03ed9d67a4f82304b58f28d83a9c72e29e760d065
translationModel: gpt-5.6-terra
translatedAt: 2026-10-19T09:15:25.046Z
translationReview: automatic
---

# Cloudflare Workers: Isolates, binding e funzionamento edge

Cloudflare Workers è una piattaforma di calcolo guidata dagli eventi all'interno della rete Cloudflare. Un Worker può rispondere o inoltrare richieste HTTP, eseguire attività pianificate, consumare messaggi di coda, elaborare e-mail in arrivo e fungere da servizio interno per altri Worker. L'operatore non gestisce né un processo listener né una singola VM. Gestisce codice, entry point, route, compatibilità del runtime, autorizzazioni alle risorse della piattaforma, versioni e dati operativi.

La parola «edge» descrive solo una possibile sede di esecuzione, non l'architettura completa. Un Worker HTTP può essere eseguito vicino all'utente; Smart Placement può spostare il calcolo più vicino a un backend; i Durable Objects hanno una sede di stato univoca; D1, KV, R2 e Queues hanno ciascuno modelli propri di replica e consistenza. Per gli amministratori, quindi, la proprietà decisiva non è «serverless», bensì la separazione tra percorso dati, Control Plane, servizi di stato e possibili punti di guasto.

La spiegazione segue una richiesta dall'endpoint pubblico al runtime Workers e quindi ai binding, allo storage e agli upstream. Successivamente inquadra deployment, sicurezza, osservabilità e ripristino dalla prospettiva della gestione della piattaforma.

## Architettura: percorso dati, runtime e Control Plane

Un sistema Workers è composto da più livelli:

1. **Ingress e routing:** Cloudflare riceve una richiesta tramite DNS, TLS e HTTP e associa un hostname o un pattern di percorso a un Worker o a un asset statico.
2. **Dispatch degli eventi:** La piattaforma genera un evento Fetch, Scheduled, Queue, E-Mail, Alarm o Tail e richiama l'entry point appropriato.
3. **Runtime:** `workerd` fornisce V8 Isolates, API Web, compatibilità Node.js e limiti di runtime.
4. **Binding:** L'oggetto `env` conferisce al codice capability per KV, D1, R2, Durable Objects, Queues, Secrets, Assets, altri Worker e ulteriori servizi della piattaforma.
5. **Stato e upstream:** I dati risiedono al di fuori dell'Isolate effimero o in storage Durable Object controllati; le chiamate in uscita usano Fetch, Service Bindings o API socket supportate.
6. **Control Plane:** Wrangler, Dashboard e API gestiscono configurazione, relazioni tra risorse, segreti, versioni, deployment e osservabilità.

Cloudflare descrive gli Isolates, il tempo di calcolo per richiesta e l'esecuzione distribuita come le tre principali differenze rispetto ai runtime server classici ([Come funzionano i Workers](https://developers.cloudflare.com/workers/reference/how-workers-works/)). Il riferimento della piattaforma separa il Worker stesso dai prodotti di storage e Developer Platform collegati ([Documentazione Cloudflare Workers](https://developers.cloudflare.com/workers/)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-cloudflare-workers.svg?v=20260813" title="Interaktive Infografik: Cloudflare-Workers-Aufruf von DNS, TLS und Routing über Event Dispatch, workerd und V8-Isolate bis zu Bindings, Storage, Upstreams, Observability, Versionen und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-cloudflare-workers.svg?v=20260813">Apri direttamente il grafico interattivo</a>.
</iframe>

## Percorso della richiesta e sede di esecuzione

Per una richiesta HTTP, la configurazione Cloudflare decide innanzitutto se un hostname `workers.dev`, una Route o un Custom Domain attivano il Worker. Una Route si colloca davanti a un Origin esistente e può modificare, rispondere o inoltrare richieste. Un Custom Domain collega direttamente un Worker a un hostname. Gli asset statici possono essere valutati prima o dopo il Worker; `run_worker_first` modifica questo ordine ([Configurazione Wrangler – route](https://developers.cloudflare.com/workers/wrangler/configuration/), [Static Assets – configurazione e binding](https://developers.cloudflare.com/workers/static-assets/binding/)).

Il percorso esterno può essere modellato come `Client → DNS → Cloudflare Edge → TLS/HTTP → Route → Worker/Asset → Binding oder Origin`. La connessione all'Origin costituisce un nuovo contesto di trasporto. Certificato client, header `Authorization`, chiave di cache, header Host e IP di origine devono pertanto essere trattati consapevolmente a ogni confine. Un Service Binding, invece, aggira la risoluzione pubblica dei nomi e fornisce un percorso Worker-to-Worker esplicitamente autorizzato ([Service bindings](https://developers.cloudflare.com/workers/runtime-apis/bindings/service-bindings/)).

Smart Placement può eseguire un Worker più vicino ai suoi backend anziché dare priorità soltanto alla vicinanza all'utente. Ciò è utile per chiamate intensive verso database, ma per asset o trasformazioni edge pure può allungare il percorso sbagliato. Cloudflare documenta quindi raccomandazioni diverse per Assets-first e Worker-first ([Placement](https://developers.cloudflare.com/workers/configuration/placement/)).

## workerd e V8 Isolates

`workerd` è il runtime JavaScript/WebAssembly orientato ai server alla base di Workers. V8 mette a disposizione gli Isolates come contesti di esecuzione separati. Molti Isolates possono condividere un processo senza che ogni applicazione richieda un processo JavaScript o un container dedicato. Le API Web sono in larga misura fornite nativamente dal runtime ([Introduzione a workerd](https://blog.cloudflare.com/workerd-open-source-workers-runtime/), [Modello di sicurezza Workers](https://developers.cloudflare.com/workers/reference/security-model/)).

Un Isolate non è una macchina persistente. Può essere riutilizzato, occupato in parallelo da più richieste o rimosso. Due richieste consecutive non devono necessariamente raggiungere la stessa istanza o località. Gli oggetti globali sono adatti a strutture ausiliarie immutabili e costose da creare oppure a cache opportunistiche, ma non come fonte di verità. Qualsiasi correttezza che dipenda da una variabile globale tra richieste è un errore architetturale.

L'isolamento V8 non sostituisce tutti i controlli di sicurezza. Cloudflare aggiunge protezione del runtime e dei processi, limiti di risorse e difese Spectre specifiche. L'operatore rimane responsabile della convalida degli input, dell'autorizzazione, dell'ambito dei segreti, delle allowlist delle destinazioni, dei limiti di output e della classificazione dei dati. Il confine dell'Isolate non protegge da un'applicazione che legge o scrive dati errati attraverso i propri binding legittimi.

## Modello a eventi e handler

Workers implementa entry point per diversi tipi di evento. Il catalogo generale degli handler comprende Fetch, Scheduled, Queue, E-Mail, Alarm e Tail ([Handler](https://developers.cloudflare.com/workers/runtime-apis/handlers/)).

| Handler | Trigger | Confine di ritorno ed errore | Tipica domanda amministrativa |
|---|---|---|---|
| `fetch()` | Richiesta HTTP o chiamata di servizio | `Response`, stream o eccezione | Quale route e versione ha elaborato la richiesta? |
| `scheduled()` | Trigger Cron | Esito Promise/invocazione | L'esecuzione pianificata è stata attivata e completata? |
| `queue()` | Batch di messaggi | Ack, retry o comportamento Dead Letter | Quale messaggio è idempotente, ripetibile o poison? |
| `email()` | Email Routing | Recapitare, rifiutare, inoltrare | Quale confine di envelope e policy si applica? |
| `alarm()` | Allarme Durable Object | esecuzione locale all'oggetto | Quale oggetto possiede l'allarme e lo stato? |
| `tail()` | Evento di trace di un Worker producer | esportazione asincrona | L'osservabilità stessa può fallire o diventare ricorsiva? |

L'handler Fetch riceve `Request`, `env` e `ctx` e restituisce una `Response` Web API ([Handler Fetch](https://developers.cloudflare.com/workers/runtime-apis/handlers/fetch/)). `ctx.waitUntil()` registra lavoro che può proseguire dopo l'invio della risposta; resta soggetto ai limiti documentati di runtime e di errore e non è un job server di lunga durata ([Context – waitUntil](https://developers.cloudflare.com/workers/runtime-apis/context/)). Per flussi persistenti e ripetibili, Queues o Workflows sono più adatti di una catena di Background Promises non osservate.

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

L'esempio utilizza un Service Binding invece di un URL liberamente configurato. In questo modo raggiungibilità e autorizzazione vengono trasferite alla configurazione di deployment, mentre il codice vede solo una capability `Fetcher`. Le API del runtime sono orientate al Web e comprendono, tra le altre, Fetch, Streams, Web Crypto, WebSockets, HTMLRewriter e TCP Sockets ([API runtime](https://developers.cloudflare.com/workers/runtime-apis/)).

Il codice non viene eseguito in un ambiente Node locale scelto liberamente, ma rispetto a un contratto di runtime versionato. La Compatibility Date determina quali modifiche di comportamento si applicano a un deployment.

## Compatibility Date e contratto di runtime

Una Compatibility Date non è una nota di rilascio né una dichiarazione sullo «stato attuale» dell'articolo. Fa parte del contratto di runtime di uno specifico deployment Worker. Le modifiche che potrebbero interrompere il comportamento esistente vengono attivate per data o flag. Una data meno recente preserva il comportamento compatibile; una data più recente deve essere testata e distribuita come un aggiornamento di dipendenza ([Compatibility flags](https://developers.cloudflare.com/workers/configuration/compatibility-flags/)).

Gli amministratori trattano quindi insieme questi campi:

- commit del codice e artefatto bundle;
- Compatibility Date e flag espliciti;
- catena di strumenti Wrangler e build;
- definizioni dei binding, Routes e Placement;
- ID versione e peso del deployment;
- migrazioni di dati o Durable Object.

La compatibilità Node.js è opt-in e implementa solo una parte documentata delle API Node. Alcuni moduli sono completi, altri parziali oppure disponibili soltanto come stub di importazione. `nodejs_compat` non deve quindi essere equiparato a «runtime Node completo» ([Compatibilità Node.js](https://developers.cloudflare.com/workers/runtime-apis/nodejs/)).

## Linguaggi e stack tecnologico

Cloudflare documenta JavaScript, TypeScript, Python e Rust come percorsi linguistici di prima classe; WebAssembly apre ulteriori linguaggi sorgente ([Linguaggi](https://developers.cloudflare.com/workers/languages/)). Lo stack tecnologico risultante distingue sviluppo ed esecuzione:

| Livello | Tecnologia tipica | Rilevanza operativa |
|---|---|---|
| Codice sorgente | TypeScript/JavaScript, Python, Rust | toolchain linguistica, dipendenze, test |
| Build | Wrangler/esbuild o adattatore di framework | bundle, Source Maps, externals, build deterministiche |
| Runtime | workerd, V8, API Web e Node opzionali | Compatibility Date, CPU/memoria, Event Loop |
| Capability | binding `env` | privilegio minimo, ID risorsa, separazione degli ambienti |
| Stato | Cache, KV, D1, R2, Durable Objects, Queues | consistenza, residenza dei dati, backup, ripristino |
| Ingress | Route, Custom Domain, workers.dev, trigger | DNS, TLS, ordine, Fail-open/closed |
| Control Plane | Wrangler, API, Dashboard, CI/CD | autenticazione, versionamento, rollout, audit |

Rust viene integrato tramite `workers-rs` e WebAssembly; il modulo Wasm importato rimane parte dell'artefatto Worker ([Supporto del linguaggio Rust](https://developers.cloudflare.com/workers/languages/rust/)). I Python Workers utilizzano classi di entry point proprie e un'integrazione runtime, non un processo CPython liberamente amministrabile.

## Configurazione e Wrangler

La configurazione Wrangler è il collegamento dichiarativo tra codice e piattaforma. Definisce almeno nome, entry point, Compatibility Date, Routes e binding. I Named Environments possono avere valori diversi; i binding devono essere verificati consapevolmente per ambiente. Cloudflare documenta i campi e l'ereditarietà nello schema Wrangler ([Configurazione Workers](https://developers.cloudflare.com/workers/configuration/), [Riferimento alla configurazione Wrangler](https://developers.cloudflare.com/workers/wrangler/configuration/)).

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

[`npx`](https://docs.npmjs.com/cli/commands/npx) avvia la CLI fissata localmente al progetto; i [`Wrangler Commands`](https://developers.cloudflare.com/workers/wrangler/commands/) documentano comandi per autenticazione, generazione dei tipi, deployment, versioni e log. `wrangler types` genera tipi dalla configurazione effettiva dei binding. `deploy --dry-run` verifica bundle e configurazione, ma non costituisce prova del funzionamento delle risorse remote o della corretta autorizzazione.

Dopo aver chiarito runtime e build, si pone la questione dell'accesso. Un Worker raggiunge database, code, segreti e altri Worker tramite binding dichiarati anziché tramite credenziali distribuite liberamente.

## Binding come modello di capability

Un binding è al tempo stesso autorizzazione e API di runtime. Il Worker riceve per esempio `env.ARCHIVE` come bucket R2, `env.DB` come database D1 o `env.AUTH` come servizio interno. La credenziale di piattaforma sottostante non viene esposta al codice applicativo come API key riutilizzabile ([Binding](https://developers.cloudflare.com/workers/runtime-apis/bindings/)).

Il modello di capability sposta la domanda amministrativa centrale da «Quale segreto è nella variabile d'ambiente?» a «Quale risorsa è associata, con quale nome, in quale ambiente e a quale versione?». Un ID bucket errato o una versione Service Binding errata possono provocare perdita di dati o accesso tra ambienti anche con codice identico.

I binding dovrebbero essere separati per compito e ambiente. Worker di produzione e test non condividono bucket, code o database scrivibili, salvo che ciò non sia esplicitamente parte di un test di integrazione controllato. Nomi generici come `DB` sono sufficienti nel codice se il deployment associa la risorsa effettiva in modo univoco e verificabile.

### Segreti e variabili

Le variabili normali sono configurazione, non segreti. I Secrets sono binding di testo cifrati e vengono gestiti al di fuori del codice sorgente. I nomi dei segreti richiesti possono essere dichiarati senza scriverne i valori nel file di configurazione ([Variabili d'ambiente](https://developers.cloudflare.com/workers/configuration/environment-variables/), [Secrets](https://developers.cloudflare.com/workers/configuration/secrets/)).

Una modifica del segreto è una modifica dello stato dell'applicazione. I client derivati globalmente da `env` possono mantenere un valore precedente in Isolates riutilizzati; Cloudflare raccomanda di creare tali derivazioni per richiesta. La rotazione comprende quindi impostazione, verifica di deployment/binding, validità parallela se richiesta dal sistema target, telemetria e rimozione controllata.

## Service Bindings e architettura Worker-to-Worker

I Service Bindings collegano esplicitamente i Worker senza richiedere un endpoint HTTP pubblicamente indirizzabile. Il servizio chiamato può essere esposto tramite Fetch o RPC. Ciò riduce la configurazione di rete e credenziali, ma non elimina problemi di versione e contratto. Con rollout separati, caller e callee possono usare stati API diversi ([Service bindings](https://developers.cloudflare.com/workers/runtime-apis/bindings/service-bindings/), [Deployment graduali](https://developers.cloudflare.com/workers/versions-and-deployments/gradual-deployments/)).

Gli amministratori documentano per ogni binding:

- caller e servizio di destinazione;
- ambiente e versione di destinazione oppure strategia di rollout;
- raggiungibilità pubblica del Worker di destinazione;
- semantica di timeout, errore e retry;
- propagazione dell'ID richiesta e correlazione dei trace;
- compatibilità contrattuale in caso di deployment indipendenti.

Una catena di molti Worker distribuisce una richiesta su più possibili punti di guasto. Una minore superficie d'attacco pubblica è preziosa; una suddivisione troppo fine può però aumentare profondità delle subrequest, sforzo di debugging e version skew.

## Servizi di stato: modello prima del nome del prodotto

L'heap dell'Isolate è effimero. Lo stato persistente risiede in servizi associati, la cui semantica differisce notevolmente. La panoramica dello storage Cloudflare classifica i prodotti per pattern di accesso e consistenza ([Scegliere un prodotto dati o storage](https://developers.cloudflare.com/workers/platform/storage-options/)).

| Servizio | Modello | Confine di consistenza/località | Adatto per | Domanda amministrativa critica |
|---|---|---|---|---|
| Cache API | Cache di risposte HTTP | locale al data center, effimera | risposte ripetibili | La cache key è completa e sicura? |
| Workers KV | Store key-value leggibile globalmente | eventual consistent, Read Cache | configurazione, dati a prevalenza di lettura | Il workflow tollera letture obsolete o negative? |
| D1 | SQL gestito basato su SQLite | semantica relativa a database e sessione | tabelle applicative relazionali | Quale garanzia di replica/sessione richiede la lettura? |
| Durable Objects | istanza oggetto univoca più storage | fortemente consistente e serializzabile per oggetto | coordinamento, lock, sessioni, WebSockets | La Partition Key è scelta correttamente? |
| R2 | storage oggetti compatibile S3 | fortemente consistente per oggetto | blob, archivi, payload grandi | Come vengono versionati insieme oggetto, metadati e indice? |
| Queues | messaggi asincroni | elaborazione orientata almeno una volta e retry policy | disaccoppiamento, livellamento del carico | Il consumer è idempotente e c'è una DLQ? |

### Cache API

La Cache API opera localmente nel data center in cui il Worker elabora la richiesta. `cache.put()` non è quindi una replica globale né una scrittura persistente nel database. `caches.default` condivide il contesto cache standard; le cache denominate creano namespace, ma neppure esse forniscono consistenza globale ([Come funziona la Cache](https://developers.cloudflare.com/workers/reference/how-the-cache-works/)).

Le risposte personalizzate possono essere memorizzate nella cache solo con una chiave che rappresenti tutte le caratteristiche rilevanti di identità e variante. `Authorization`, cookie, lingua, encoding e appartenenza al tenant sono confini tipici di leakage. Cache purge e validazione dell'Origin fanno parte del ripristino, non solo i TTL.

### Workers KV

KV replica i dati tramite store centrali e cache edge. Le modifiche possono diventare visibili in ritardo in altre località; anche «non trovato» può essere memorizzato nella cache. Cloudflare indica ritardi fino a 60 secondi o più per sedi remote e raccomanda di non usare KV per transazioni atomiche Read-Modify-Write ([Come funziona KV](https://developers.cloudflare.com/kv/concepts/how-kv-works/)).

KV è adatto a configurazioni prevalentemente lette, allowlist/denylist o cache, quando la staleness viene esplicitamente tollerata. Rate limit globali, sequenze univoche o revoca immediata di credenziali non appartengono a KV senza coordinamento aggiuntivo.

### Durable Objects

Un Durable Object collega un ID oggetto univoco a livello globale con coordinamento seriale e storage privato. Le richieste per lo stesso ID vengono instradate alla stessa istanza logica. I nuovi namespace usano il percorso di storage SQLite; questo offre storage transazionale e fortemente consistente nonché funzionalità di Point-in-Time Recovery ([Panoramica Durable Objects](https://developers.cloudflare.com/durable-objects/), [Storage Durable Object basato su SQLite](https://developers.cloudflare.com/durable-objects/api/sqlite-storage-api/)).

Anche un Durable Object non è un server sempre in esecuzione. Lo stato in memoria può scomparire con hibernation o eviction e, se necessario, deve essere ricostruito dallo storage. Non esiste un Shutdown Hook affidabile ([Ciclo di vita Durable Object](https://developers.cloudflare.com/durable-objects/concepts/durable-object-lifecycle/)). Le modifiche a classi e assegnazione dello storage sono operazioni di ciclo di vita; vengono trattate come migrazioni di dati ([Ciclo di vita delle classi Durable Object](https://developers.cloudflare.com/durable-objects/reference/durable-objects-migrations/)).

### D1, R2 e Queues

D1 offre SQL per dati relazionali. R2 memorizza grandi oggetti non strutturati ed espone un'API compatibile S3 all'esterno o una Binding API all'interno di Workers. Queues disaccoppia producer e consumer. Questi servizi non si sostituiscono: un oggetto R2 può contenere il payload, D1 l'indice, una Queue può avviare l'elaborazione e un Durable Object può coordinare modifiche concorrenti.

Il commit congiunto di questi quattro stati non è automaticamente atomico. Gli amministratori pianificano Idempotency Keys, pattern Outbox/Inbox, reconciliation ed elaborazione Dead Letter. Documentazione e limiti dei singoli servizi per [D1](https://developers.cloudflare.com/d1/), [R2](https://developers.cloudflare.com/r2/) e [Queues](https://developers.cloudflare.com/queues/) fanno parte del runbook.

## Asset statici e Worker full-stack

Un Worker può distribuire asset statici insieme al codice. L'Assets Binding consente al codice di recuperare esplicitamente un asset. L'opzione di routing determina se gli asset esistenti aggirano il Worker o se il Worker viene eseguito per primo ([Static Assets](https://developers.cloudflare.com/workers/static-assets/)).

L'ordine è una decisione di sicurezza. Con Assets-first, un percorso file esistente accidentalmente non deve aggirare l'autenticazione. Con Worker-first, ogni richiesta di asset aumenta il carico di calcolo e dipendenze. Gli adattatori di framework nascondono in parte questo ordine; l'artefatto di deployment generato e la configurazione Wrangler rimangono la fonte di verità determinante.

## Prospettiva di rete e protocolli

Workers è nel percorso applicativo sopra DNS, TLS e HTTP. La piattaforma termina il trasporto esterno; il Worker elabora richieste Web API. Un `fetch()` in uscita genera una nuova chiamata HTTP. I TCP Sockets consentono protocolli in uscita selezionati, ma non trasformano il Worker in un server TCP generalmente raggiungibile ([Protocolli Workers](https://developers.cloudflare.com/workers/reference/protocols/), [TCP sockets](https://developers.cloudflare.com/workers/runtime-apis/tcp-sockets/)).

Per gli amministratori di messaggistica sono importanti tre confini:

- Un handler e-mail è attivato da Cloudflare Email Routing; non rimane esso stesso in ascolto su [SMTP](/kb/smtp).
- Un Worker che chiama un'API di posta deve trattare OAuth audience, ciclo di vita del token e retry come qualsiasi altro client API.
- Gli errori DNS, TLS e HTTP si verificano prima dell'entry point; gli errori di binding, autenticazione e applicazione dopo. Un timeout `fetch()` non indica, senza correlazione, quale fase sia fallita.

## Confini di sicurezza e fiducia

Il modello di sicurezza comprende almeno cinque confini di fiducia separati:

1. **Richiesta pubblica:** header, dimensione del body, metodo e identità arbitrari.
2. **Configurazione Cloudflare:** Zone, Route, WAF, Access, certificati e Fail-open/closed.
3. **Artefatto Worker:** codice, dipendenze, Source Maps, Compatibility Date e supply chain.
4. **Binding:** autorizzazioni concrete alle risorse, segreti e percorsi di servizio interni.
5. **Upstream e dati memorizzati:** autorizzazione, consistenza e ripristino propri.

Il privilegio minimo si ottiene principalmente tramite superfici di binding ridotte e ambienti separati. Un Worker con accesso in scrittura a tutti i bucket e database resta un principal altamente privilegiato, anche se non possiede chiavi di accesso visibili. La documentazione di sicurezza Workers illustra l'isolamento del runtime; non sostituisce un'architettura di sicurezza applicativa ([Modello di sicurezza](https://developers.cloudflare.com/workers/reference/security-model/)).

I controlli della supply chain comprendono versioni dei pacchetti fissate, lockfile, build riproducibili, Dependency Scanning, Secret Scanning, autorizzazioni di upload minime e ambienti di deployment protetti. Il bundle generato viene verificato prima dell'upload; la sola revisione del codice sorgente non basta se bundler o framework incorporano moduli aggiuntivi.

La vicinanza all'edge non elimina i limiti delle risorse. Tempo CPU, subrequest, memoria e limiti di piattaforma devono essere considerati già nella progettazione e nel modello di carico.

## I limiti come parametri architetturali

Workers limita, tra l'altro, tempo CPU, memoria, tempo di startup, subrequest, dimensione del bundle, volume dei log, numero di route e asset statici. I valori differiscono in base al piano e al tipo di invocazione e possono cambiare. Per questo nell'articolo statico non deve apparire una tabella dei limiti copiata come presunta verità senza tempo; la pagina ufficiale [Workers Limits](https://developers.cloudflare.com/workers/platform/limits/) è la fonte operativa.

Gli amministratori distinguono:

- **CPU Time:** tempo di calcolo attivo; l'attesa I/O non conta come CPU nello stesso modo.
- **Wall Time:** durata trascorsa dell'evento; le regole distinguono Fetch, Cron, Queue e Durable Object.
- **Startup Time:** valutazione e inizializzazione del modulo prima dell'handler.
- **Subrequests:** operazioni Fetch e di piattaforma per invocazione.
- **Memory:** heap, stream, buffer e stato delle librerie dell'Isolate.
- **Log Budget:** payload troppo grandi o segreti sono sia un problema di costo sia di protezione dei dati.

Un errore `1102` indica limiti di risorse superati; `10021` può verificarsi in caso di inizializzazione di startup troppo costosa. Il numero di errore è un punto di partenza, non una root cause. Profilo CPU, tipo di invocazione, versione, classe di input e grafo delle subrequest devono essere analizzati insieme.

## Sviluppo locale e test

Lo sviluppo locale esegue codice con `workerd` tramite Miniflare. Per impostazione predefinita, i binding vengono simulati localmente; singoli Remote Bindings possono essere collegati in modo mirato a risorse reali della piattaforma. `wrangler dev --remote` eseguito completamente in remoto rimane disponibile per casi specifici di rete, ma non è più il percorso standard ([Sviluppo locale](https://developers.cloudflare.com/workers/local-development/), [Binding per modalità di sviluppo](https://developers.cloudflare.com/workers/local-development/bindings-per-env/)).

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

[`Invoke-WebRequest`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest) e [`curl`](https://curl.se/docs/manpage.html) verificano stato e header del percorso HTTP locale. L'integrazione Workers-Vitest esegue i test all'interno di `workerd` e fornisce helper per binding, richieste e Durable Objects ([Integrazione Vitest](https://developers.cloudflare.com/workers/testing/vitest-integration/), [API di test Workers](https://developers.cloudflare.com/workers/testing/vitest-integration/test-apis/)).

Un Unit Test verde non dimostra route corretta, autorizzazione remota o migrazione dei dati. La piramide dei test comprende logica pura, integrazione runtime, semantica dei binding, route di staging, Production Smoke Test controllato ed esercitazione di ripristino.

## Deployment, versione e rollout

Una **versione** è un oggetto immutabile di codice/configurazione. Un **deployment** distribuisce il traffico su una o più versioni. I Gradual Deployments spostano i pesi gradualmente e consentono osservazione e ritorno a una versione stabile ([Versioni e deployment](https://developers.cloudflare.com/workers/versions-and-deployments/), [Deployment graduali](https://developers.cloudflare.com/workers/versions-and-deployments/gradual-deployments/)).

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

Prima di un rollout viene salvata l'associazione `Git commit → Bundlehash → Worker-Version → Deploymentgewicht`. Con più Service Bindings, il version skew fa parte del test. I Durable Objects richiedono regole speciali di migrazione e rollout, poiché per un ID oggetto non possono essere attivi contemporaneamente stati di classe arbitrari.

Il rollback ripristina codice e configurazione, ma non automaticamente i dati esterni. Un nuovo Worker può aver scritto dati in un formato che quello precedente non comprende. Le modifiche allo schema usano Expand/Contract, compatibilità in avanti o un ripristino dei dati testato separatamente. «Rollback possibile» è dimostrato solo quando sono state verificate insieme le parti di codice, binding e dati.

## Osservabilità e prova operativa

Workers Logs raccoglie Invocation Logs, log propri, errori ed eccezioni non gestite. Tail Workers o Logpush possono esportare eventi; le destinazioni OpenTelemetry offrono un percorso di esportazione più diretto per log e trace ([Workers Logs](https://developers.cloudflare.com/workers/observability/logs/workers-logs/), [Tail Workers](https://developers.cloudflare.com/workers/observability/logs/tail-workers/), [Trace](https://developers.cloudflare.com/workers/observability/traces/)).

Ogni invocazione dovrebbe rendere correlabili almeno:

- nome del Worker, ID versione e ambiente;
- tipo di evento e route;
- ID richiesta/messaggio/correlazione;
- risultato, stato HTTP o Ack/Retry;
- CPU e Wall Time;
- destinazione e latenza delle subrequest senza segreti;
- classe di binding/risorsa, non il contenuto completo sensibile;
- Idempotency Key funzionale nell'elaborazione asincrona.

`console.log()` non è un archivio di evidenze illimitato. Limiti dei log, sampling e redaction influenzano la visibilità. Credenziali, Authorization Header, cookie, contenuti completi delle e-mail e payload personali non vengono registrati. Il monitoring sorveglia anche il percorso di esportazione: un Tail Worker difettoso non deve rendere impossibile la diagnosi dell'errore del producer.

La diagnosi segue il percorso di pubblicazione: route e DNS, versione attiva, runtime e binding, dipendenze in uscita e infine log e trace.

## Diagnosi amministrativa per fasi

Gli errori Workers vengono separati in modo efficiente lungo il percorso di esecuzione:

1. **DNS:** L'host previsto risolve verso Cloudflare e la Zone è attiva?
2. **TLS/HTTP:** Verificare certificato, SNI, protocollo e stato.
3. **Route/Asset:** Host/percorso raggiunge il Worker, un asset o l'Origin?
4. **Versione:** Quale versione e quale peso di deployment hanno elaborato la richiesta?
5. **Runtime:** Verificare startup, CPU, memoria, eccezione e output dell'handler.
6. **Binding:** La risorsa esiste in questo ambiente e possiede la capability prevista?
7. **Upstream/Storage:** Indagare timeout, autenticazione, consistenza, stato dei dati e retry.
8. **Ripristino:** Confermare versione stabile, compatibilità dei dati e reconciliation.

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

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname), [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection), [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility), [`nc`](https://man.openbsd.org/nc), [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) e [`curl`](https://curl.se/docs/manpage.html) separano DNS, TCP, TLS e HTTP prima del runtime. [DNS](/kb/dns), [TCP](/kb/tcp), [TLS](/kb/tls), [API](/kb/apis) e [Risoluzione dei problemi](/kb/troubleshooting) trattano questi livelli in modo approfondito.

### Problemi tipici

| Sintomo | Fase probabile | Evidenza | Errore di ragionamento frequente |
|---|---|---|---|
| Risposta Origin invece di risposta Worker | Route/Asset/Fail-open | Route, host, percorso, marcatore di risposta | «Deployment riuscito» significhi «route attiva» |
| 1101 o eccezione | Applicazione/binding | Workers Logs, versione, stack | eseguire solo un nuovo deployment invece di verificare la classe di input |
| 1102 | Limite di risorse | CPU/Wall Time, profilo, tipo di invocazione | equiparare attesa I/O e CPU |
| Valore KV occasionalmente vecchio | Consistenza KV | chiave, località, TTL cache, ora di scrittura | trattare KV come transazione globale |
| Locale verde, errore binding remoto | Ambiente/capability | ID risorsa concreto e nome binding | la simulazione dimostri lo stato IAM/risorsa |
| Solo parte del traffico difettosa | Gradual Deployment | ID versione e peso | aggregare metriche senza dimensione versione |
| Il rollback risolve il codice, non i dati | Confine schema/stato | formato scritto, migrazione, reconciliation | il deployment sia un ripristino completo |

## Backup e Disaster Recovery

Il serverless non elimina il backup. Sposta gli oggetti da proteggere:

- repository Git, lockfile e definizione della build riproducibile;
- configurazione Wrangler, Routes, Compatibility Date e flag;
- inventario di tutti i binding e delle risorse di destinazione;
- valori dei segreti o la loro Source of Truth esterna e processo di rotazione;
- esportazioni dei dati e procedure di restore per D1, R2, KV e sistemi esterni;
- classi Durable Object, stato del ciclo di vita/migrazione ed eventualmente PITR;
- stato Queue/DLQ, Idempotency Keys e reconciliation;
- cronologia di versioni/deployment e percorso di rollback testato.

Non tutti i prodotti hanno la stessa semantica di esportazione e restore. Un backup di oggetto R2 non protegge l'indice D1; un restore D1 non ripristina un messaggio Queue già confermato; un Worker riportato indietro può interpretare erroneamente nuovi formati di oggetti. [Backup e Disaster Recovery](/kb/backup-dr) deve quindi definire un punto applicativo coerente anziché soltanto copie di singoli prodotti.

Il ripristino viene testato in scenari: segreto errato, Route eliminata, versione difettosa, schema D1 incompatibile, oggetto R2 perso, poison Queue Message, classe Durable Object difettosa e upstream irraggiungibile. RTO e RPO vengono definiti per ciascun servizio di stato e per l'applicazione composta.

## Storia tecnica

Cloudflare ha presentato Workers nel 2017 come esecuzione programmabile nella rete globale. Il punto di partenza era il limite di latenza dei data center centrali e l'idea di eseguire codice vicino al percorso dati ([Code Everywhere: perché abbiamo creato Cloudflare Workers](https://blog.cloudflare.com/code-everywhere-cloudflare-workers/)).

L'architettura si è basata presto sui V8 Isolates anziché su un container o processo per funzione. Nel 2018 Cloudflare ha descritto le differenze economiche e tecniche di questo modello, ma anche la limitazione a JavaScript o linguaggi compatibili con WebAssembly ([Cloud Computing senza container](https://blog.cloudflare.com/cloud-computing-without-containers/)).

Workers KV ha aggiunto stato globale leggibile ed eventual-consistent. Nel 2020 i Durable Objects sono stati annunciati come modello opposto per stati oggetto coordinati e fortemente consistenti ([Introduzione ai Workers Durable Objects](https://blog.cloudflare.com/introducing-workers-durable-objects/)). Workers si è così evoluto dalla pura trasformazione delle richieste in una piattaforma applicativa con più modelli di consistenza espliciti.

Nel 2022 Cloudflare ha pubblicato `workerd` sotto Apache 2.0. Il runtime aperto condivide codice con il sistema di produzione e ha migliorato l'accuratezza dello sviluppo locale; tuttavia non è l'intera piattaforma Cloudflare con routing, orchestrazione e gestione della sicurezza ([Introduzione a workerd](https://blog.cloudflare.com/workerd-open-source-workers-runtime/), [Repository workerd](https://github.com/cloudflare/workerd)).

Le espansioni successive hanno introdotto entry point ES Module, Compatibility Dates, Service/RPC Bindings, D1, R2, Queues, Workflows, Python, compatibilità Node ampliata, oggetti versione e deployment graduali. Questa storia spiega il nucleo attuale: Workers non è né JavaScript per browser su una CDN né un server Linux arbitrario, bensì un runtime guidato dagli eventi e basato su capability con prodotti separati per dati e Control Plane.

## Checklist amministrativa

Prima della messa in produzione sono comprovati almeno i seguenti punti:

- Route, Custom Domain, ordine degli asset e Fail-open/closed sono documentati.
- Compatibility Date, flag, versione Wrangler, bundle e Source Maps sono riproducibili.
- Ogni binding possiede owner, ambiente, ID risorsa, diritti e percorso di ripristino.
- La semantica di Cache, KV, D1, R2, Durable Object e Queue non viene confusa.
- I segreti sono esterni al repository, ruotabili e non visibili nei log.
- Timeout, retry, idempotenza e comportamento Dead Letter sono definiti per ogni tipo di evento.
- Log e trace contengono versione e Correlation ID, ma nessun payload sensibile.
- Gradual Deployment e rollback considerano versioni Service Binding e formati dati.
- I limiti sono monitorati dalla pagina ufficiale della piattaforma, non da una tabella copiata.
- Backup, restore e reconciliation sono stati testati per l'applicazione composta.

## Fonti

- [Come funzionano i Workers](https://developers.cloudflare.com/workers/reference/how-workers-works/)
- [Documentazione Cloudflare Workers](https://developers.cloudflare.com/workers/)
- [Riferimento alla configurazione Wrangler](https://developers.cloudflare.com/workers/wrangler/configuration/)
- [Static Assets – configurazione e binding](https://developers.cloudflare.com/workers/static-assets/binding/)
- [Service bindings](https://developers.cloudflare.com/workers/runtime-apis/bindings/service-bindings/)
- [Placement](https://developers.cloudflare.com/workers/configuration/placement/)
- [Introduzione a workerd](https://blog.cloudflare.com/workerd-open-source-workers-runtime/)
- [Modello di sicurezza Workers](https://developers.cloudflare.com/workers/reference/security-model/)
- [Handler](https://developers.cloudflare.com/workers/runtime-apis/handlers/)
- [Handler Fetch](https://developers.cloudflare.com/workers/runtime-apis/handlers/fetch/)
- [Context – waitUntil](https://developers.cloudflare.com/workers/runtime-apis/context/)
- [API runtime](https://developers.cloudflare.com/workers/runtime-apis/)
- [Compatibility flags](https://developers.cloudflare.com/workers/configuration/compatibility-flags/)
- [Compatibilità Node.js](https://developers.cloudflare.com/workers/runtime-apis/nodejs/)
- [Linguaggi](https://developers.cloudflare.com/workers/languages/)
- [Supporto del linguaggio Rust](https://developers.cloudflare.com/workers/languages/rust/)
- [Configurazione Workers](https://developers.cloudflare.com/workers/configuration/)
- [Documentazione npx](https://docs.npmjs.com/cli/commands/npx)
- [Comandi Wrangler](https://developers.cloudflare.com/workers/wrangler/commands/)
- [Binding](https://developers.cloudflare.com/workers/runtime-apis/bindings/)
- [Variabili d'ambiente](https://developers.cloudflare.com/workers/configuration/environment-variables/)
- [Secrets](https://developers.cloudflare.com/workers/configuration/secrets/)
- [Deployment graduali](https://developers.cloudflare.com/workers/versions-and-deployments/gradual-deployments/)
- [Scegliere un prodotto dati o storage](https://developers.cloudflare.com/workers/platform/storage-options/)
- [Come funziona la Cache](https://developers.cloudflare.com/workers/reference/how-the-cache-works/)
- [Come funziona KV](https://developers.cloudflare.com/kv/concepts/how-kv-works/)
- [Panoramica Durable Objects](https://developers.cloudflare.com/durable-objects/)
- [Storage Durable Object basato su SQLite](https://developers.cloudflare.com/durable-objects/api/sqlite-storage-api/)
- [Ciclo di vita Durable Object](https://developers.cloudflare.com/durable-objects/concepts/durable-object-lifecycle/)
- [Ciclo di vita delle classi Durable Object](https://developers.cloudflare.com/durable-objects/reference/durable-objects-migrations/)
- [D1](https://developers.cloudflare.com/d1/)
- [R2](https://developers.cloudflare.com/r2/)
- [Queues](https://developers.cloudflare.com/queues/)
- [Static Assets](https://developers.cloudflare.com/workers/static-assets/)
- [Protocolli Workers](https://developers.cloudflare.com/workers/reference/protocols/)
- [TCP sockets](https://developers.cloudflare.com/workers/runtime-apis/tcp-sockets/)
- [Workers Limits](https://developers.cloudflare.com/workers/platform/limits/)
- [Sviluppo locale](https://developers.cloudflare.com/workers/local-development/)
- [Binding per modalità di sviluppo](https://developers.cloudflare.com/workers/local-development/bindings-per-env/)
- [Invoke-WebRequest](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest)
- [Manuale curl](https://curl.se/docs/manpage.html)
- [Integrazione Vitest](https://developers.cloudflare.com/workers/testing/vitest-integration/)
- [API di test Workers](https://developers.cloudflare.com/workers/testing/vitest-integration/test-apis/)
- [Versioni e deployment](https://developers.cloudflare.com/workers/versions-and-deployments/)
- [Workers Logs](https://developers.cloudflare.com/workers/observability/logs/workers-logs/)
- [Tail Workers](https://developers.cloudflare.com/workers/observability/logs/tail-workers/)
- [Trace](https://developers.cloudflare.com/workers/observability/traces/)
- [Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)
- [Test-NetConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection)
- [Manuale dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [nc(1)](https://man.openbsd.org/nc)
- [openssl-s_client(1)](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Code Everywhere: perché abbiamo creato Cloudflare Workers](https://blog.cloudflare.com/code-everywhere-cloudflare-workers/)
- [Cloud Computing senza container](https://blog.cloudflare.com/cloud-computing-without-containers/)
- [Introduzione ai Workers Durable Objects](https://blog.cloudflare.com/introducing-workers-durable-objects/)
- [Repository workerd](https://github.com/cloudflare/workerd)
