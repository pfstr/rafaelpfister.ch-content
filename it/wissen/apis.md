---
title: "API: contratti, identità e confini di errore distribuiti"
blatt: "apis"
description: "API per amministratori di infrastrutture e messaggistica: REST, HTTP e JSON, RPC, GraphQL, gRPC e API evento, OpenAPI e schemi, OAuth e identità di workload, gateway, timeout, retry, idempotenza, paginazione, limiti di velocità, webhook, osservabilità, versionamento e storia tecnica."
fakten:
  - label: Ruolo di sistema
    wert: interfaccia di gestione o dati leggibile dalla macchina tra componenti separati
    href: https://ics.uci.edu/~fielding/pubs/dissertation/net_app_arch.htm
  - label: REST
    wert: stile architetturale con vincoli; non equivale a HTTP più JSON
    href: https://ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm
  - label: Modello HTTP
    wert: risorsa/URI · metodo · campi · rappresentazione · stato
    href: https://www.rfc-editor.org/rfc/rfc9110.html
  - label: Contratto
    wert: descrivere operazioni, schemi, errori, sicurezza e ciclo di vita in modo leggibile dalla macchina
    href: https://spec.openapis.org/oas/
  - label: Formati dati
    wert: JSON · XML · Protocol Buffers · formati binari e di streaming
    href: https://www.rfc-editor.org/rfc/rfc8259.html
  - label: Stili di interazione
    wert: richiesta/risposta · RPC · query · stream · evento/webhook
    href: https://ics.uci.edu/~fielding/pubs/dissertation/net_arch_styles.htm
  - label: Identità
    wert: API key, certificato client o token; credenziale e autorizzazione separati
    href: https://www.rfc-editor.org/rfc/rfc9700.html
  - label: Ripetizione
    wert: solo con semantica nota; un timeout non significa che lato server non sia accaduto nulla
    href: https://www.rfc-editor.org/rfc/rfc9110.html
  - label: Concorrenza
    wert: ETag/If-Match o numero di versione di dominio evita Lost Updates
    href: https://www.rfc-editor.org/rfc/rfc9110.html
  - label: Formato di errore
    wert: stato HTTP più tipo di problema stabile e leggibile dalla macchina e Request-ID
    href: https://www.rfc-editor.org/rfc/rfc9457.html
  - label: Stato operativo
    wert: latenza · tasso di errore · saturazione · quota · scadenza del token · Queue/Consumer Lag
    href: https://opentelemetry.io/docs/specs/otel/trace/
  - label: Evidenza amministrativa
    wert: client · identità · scope · endpoint · versione del contratto · Request-ID · risultato
    href: https://www.rfc-editor.org/rfc/rfc9110.html
werbung:
  - newsletter
ctaThemen:
  - cloudflare-workers
  - powershell
  - automatisierung
translationSourceHash: 4b29095b24ea1576608e147b1904a65a72c2b09b2e72f991e3e27a64dbcb3332
translationModel: gpt-5.6-terra
translatedAt: 2026-10-03T09:30:00.055Z
translationReview: automatic
---

# API: contratti, identità e confini di errore distribuiti

Una Application Programming Interface è un **confine contrattuale e di fiducia** tra componenti gestiti in modo indipendente. Il contratto stabilisce quali operazioni e dati esistono; il runtime decide trasporto, identità, autorizzazione, comportamento temporale ed errori. Per gli amministratori questa separazione è centrale: una richiesta JSON sintatticamente valida può arrivare al tenant sbagliato, fallire con un token valido per l'audience sbagliata o essere stata comunque eseguita con successo lato server dopo un timeout del client.

Le API sono ovunque negli ambienti di messaggistica: tra client di gestione e piattaforma di posta, gateway e directory, monitoraggio e backend di telemetria, applicazione cloud e ricevitore webhook. Una GUI può usare la stessa interfaccia, ma di norma rappresenta solo una parte dei suoi stati ed errori. Un esercizio affidabile non inizia quindi dalla singola chiamata curl, ma da **stile di interfaccia, contratto, modello di risorse, identità, transizioni di stato e semantica di ripristino**.

La spiegazione segue una chiamata API dal client alla risposta di dominio. Prima tratta trasporto e contratto, poi identità, gestione degli errori ed eventi; solo in seguito tratta esercizio del gateway, sicurezza e diagnosi.

## Inquadramento come sistema distribuito

Un'API di rete non è solo codice applicativo. Una chiamata tipica attraversa:

1. libreria client, CLI o processo di automazione;
2. risoluzione dei nomi, routing e instaurazione della connessione;
3. [TLS](/kb/tls), proxy o service mesh;
4. load balancer, API gateway o reverse proxy;
5. autenticazione, verifica del token e autorizzazione;
6. servizio applicativo, cache, coda e database;
7. percorso di risposta, serializzazione e valutazione del client.

Il lavoro architetturale di Roy Fielding distingue esplicitamente i sistemi basati sulla rete dall'esecuzione locale trasparente: la comunicazione di rete presenta proprie latenze, costi e modalità di errore. Uno stile architetturale è un insieme coordinato di vincoli che produce determinate proprietà e compromessi ([Fielding – Network-based Application Architectures](https://ics.uci.edu/~fielding/pubs/dissertation/net_app_arch.htm), [Fielding – Network-based Architectural Styles](https://ics.uci.edu/~fielding/pubs/dissertation/net_arch_styles.htm)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-apis.svg?v=20260813" title="Interaktive Infografik: API-Aufruf über DNS, TLS, Gateway, Identität und Dienst sowie Vertrag, Fehler- und Wiederholungssemantik, Events und Betriebsnachweis" loading="lazy">
  <a href="/images/kb-interaktiv-apis.svg?v=20260813">Apri direttamente il grafico interattivo</a>.
</iframe>

Lo stack rilevante viene documentato per ogni integrazione:

| Livello | Esempi | Tipico ambito di guasto |
|---|---|---|
| Identificazione | URI, Service Discovery, DNS | host, regione, tenant o percorso API errato |
| Trasporto | TCP/TLS, HTTP/1.1, HTTP/2, HTTP/3 | timeout, proxy, certificato, ALPN, connection pool |
| Interazione | REST, RPC, GraphQL, gRPC, webhook/evento | semantica errata, retry inappropriato, interruzione dello streaming |
| Rappresentazione | JSON, XML, Protobuf, Multipart, dati binari | errore di schema, encoding, dimensione o compatibilità |
| Contratto | OpenAPI, JSON Schema, Protobuf IDL, GraphQL SDL, AsyncAPI | breaking change, deriva tra documentazione e runtime |
| Identità | API key, token OAuth, mTLS, richiesta firmata | scadenza, scope, audience, rotazione delle chiavi, clock skew |
| Policy | gateway, WAF, RBAC/ABAC, quota | 401/403/429, normalizzazione degli header, principal errato |
| Stato | servizio, cache, coda, database | esecuzione parziale, ritardo di replica, eventual consistency |
| Evidenza | Request-ID, trace, audit log, metriche | correlazione mancante, sampling, protezione dei dati |

## REST è uno stile architetturale, non un formato di dati

REST indica i vincoli descritti da Fielding per sistemi hypermedia distribuiti: client/server, statelessness, cache, interfaccia uniforme, layering e Code-on-Demand opzionale. L'interfaccia uniforme comprende identificazione delle risorse, manipolazione tramite rappresentazioni, messaggi auto-descrittivi e hypermedia come macchina a stati. La standardizzazione dell'interfaccia migliora visibilità e sviluppo indipendente, ma può essere meno efficiente rispetto a protocolli specializzati ([Fielding – Representational State Transfer](https://ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)).

Un'API HTTP con JSON e percorsi come `/v1/getUser` non è quindi automaticamente REST. Può semplicemente essere RPC su HTTP. Questo non è intrinsecamente negativo; diventa problematico se gli operatori si aspettano proprietà che lo stile effettivo non offre. Ad esempio, un client non può ripetere in sicurezza una POST solo perché l'endpoint è chiamato «REST».

### URI, risorsa e rappresentazione

Un URI identifica una risorsa; non garantisce né raggiungibilità né una determinata operazione. RFC 3986 separa esplicitamente identificazione e interazione. Scheme, authority, path, query e fragment hanno una sintassi definita, mentre l'API concreta stabilisce la semantica della risorsa ([RFC 3986 – URI Generic Syntax](https://www.rfc-editor.org/rfc/rfc3986.html)).

Una risorsa non coincide con il suo file JSON. La stessa risorsa può essere rappresentata come JSON, XML o altro formato a seconda dell'header `Accept`. `Content-Type` descrive il body inviato, `Accept` la risposta preferita. Stato, campi e body formano insieme il messaggio; registrare solo il body JSON fa scomparire importanti informazioni diagnostiche.

## Semantica HTTP: il metodo prima del nome del percorso

RFC 9110 separa l'identificazione della risorsa dalla semantica della richiesta. Il metodo definisce l'operazione prevista; il solo percorso URI no. «Safe» significa che il client non intende modificare lo stato. «Idempotent» significa che più richieste identiche hanno lo stesso effetto previsto di una richiesta; effetti collaterali come il logging possono comunque verificarsi più volte ([RFC 9110 – HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)).

| Metodo | Safe | Idempotente | Semantica API tipica | Retry senza informazioni aggiuntive |
|---|---:|---:|---|---|
| GET | sì | sì | lettura della rappresentazione | in linea di principio possibile, ma considerare carico/quota |
| HEAD | sì | sì | metadati senza body | in linea di principio possibile |
| OPTIONS | sì | sì | capacità/opzioni di comunicazione | in linea di principio possibile |
| PUT | no | sì | sostituzione dello stato a URI noto | possibile se il contratto rispetta realmente la semantica PUT |
| DELETE | no | sì | rimozione dell'associazione | effetto ripetibile; lo stato della risposta può cambiare |
| POST | no | no | elaborazione, azione o nuova risorsa | non ripetere alla cieca |
| PATCH | no | non in generale | modifica parziale | solo con semantica Patch e di idempotenza documentata |

L'idempotenza descrive l'**effetto server previsto**, non la risposta di trasporto. Un PUT può essere stato completato sul server mentre la risposta va persa. Ripetere il PUT è allora semanticamente giustificabile; ripetere una POST può creare un secondo oggetto o un secondo messaggio. Per operazioni non idempotenti sono necessari un identificatore dell'operazione generato dal client, una Idempotency-Key specifica del produttore o una successiva ricerca dello stato.

### I codici di stato sono categorie, non una diagnosi completa

- `2xx`: la richiesta è stata elaborata nel modo definito dallo stato; non ogni `202 Accepted` è già concluso dal punto di vista di dominio.
- `3xx`: ulteriore azione o diversa rappresentazione; i redirect possono modificare metodo e flusso delle credenziali.
- `400`: la richiesta è errata dal punto di vista del server.
- `401`: credenziali di autenticazione mancanti o non valide; la risposta usa in linea di principio `WWW-Authenticate`.
- `403`: il server comprende la richiesta, ma la rifiuta.
- `404`: risorsa non trovata o nascosta intenzionalmente; nessuna prova sicura della sua inesistenza.
- `409`: conflitto con lo stato corrente.
- `412`: precondizione come `If-Match` non soddisfatta.
- `429`: troppe richieste in una finestra temporale; `Retry-After` può indicare un tempo di attesa.
- `5xx`: il server non ha potuto soddisfare una richiesta fondamentalmente valida; non automaticamente ripetibile.

RFC 6585 definisce `429 Too Many Requests`, ma né l'ambito della quota né il contatore. Questi possono valere per credenziale, utente, tenant, risorsa, regione o cluster ([RFC 6585 – Additional HTTP Status Codes](https://www.rfc-editor.org/rfc/rfc6585.html)). Il client salva quindi stato, campi di risposta rilevanti, Request-ID e body di errore troncato.

## Varianti di trasporto e costi di connessione

La semantica HTTP è separata dalla versione wire concreta. HTTP/1.1 usa regole testuali di framing dei messaggi, HTTP/2 stream multiplexati e framing binario, HTTP/3 realizza HTTP su QUIC. Un API gateway può accettare HTTP/2 lato client e parlare HTTP/1.1 verso il backend; il protocollo sul client non prova l'intero percorso backend ([RFC 9112 – HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112.html), [RFC 9113 – HTTP/2](https://www.rfc-editor.org/rfc/rfc9113.html), [RFC 9114 – HTTP/3](https://www.rfc-editor.org/rfc/rfc9114.html)).

Gli amministratori osservano non solo la latenza della richiesta, ma anche tempi DNS e di connessione, handshake TLS, riuso della connessione, versione HTTP/ALPN, tempo di proxy/gateway, Time to First Byte, trasferimento del body, numero di retry e durata wall-clock complessiva.

### Verificare percorso dei nomi, TCP e TLS

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) e [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) verificano [DNS](/kb/dns). [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) e [`nc`](https://man.openbsd.org/nc) verificano TCP. [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) mostra handshake TLS, catena di certificati e ALPN; nessuno di questi test dimostra un'autorizzazione API riuscita.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für API-Netz- und TLS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName api.example.ch -Type A
Resolve-DnsName api.example.ch -Type AAAA
Test-NetConnection api.example.ch -Port 443 -InformationLevel Detailed
curl.exe -sSvk --http2 -o NUL $env:API_HEALTH_ENDPOINT</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig api.example.ch A
dig api.example.ch AAAA
nc -vz api.example.ch 443
openssl s_client -connect api.example.ch:443 -servername api.example.ch -alpn h2,http/1.1 &lt;/dev/null</code></pre>
  </div>
</div>
Una volta compresi metodo, trasporto e rappresentazione, segue la prima particolarità distribuita: una modifica riuscita non deve essere immediatamente visibile in ogni percorso di lettura.

## Caching e Read-after-write

Il caching HTTP memorizza rappresentazioni in base a chiavi cache e direttive. `Cache-Control`, `Vary`, validator e regole di autenticazione determinano se e come vengono riutilizzate. Un `200` può provenire da una cache; un GET immediatamente successivo a PUT può ancora visualizzare stato precedente a seconda dell'architettura ([RFC 9111 – HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111.html)).

Domande per l'amministratore:

- Nel percorso è presente una cache del browser, proxy, CDN, gateway o applicazione?
- Quali header formano la cache key, soprattutto Authorization e tenant?
- La rappresentazione è privata, pubblica o non cacheabile?
- Per quanto tempo le risposte negative sono cacheabili?
- Esiste Read-your-writes o eventual consistency?
- Quale regione/replica legge il GET successivo?
- ETag è una versione del contenuto o solo un cache validator?

La pulizia della cache non è una riparazione universale. Può generare picchi di carico e nascondere l'incoerenza reale.

## Rate limit, quote e saturazione

Rate limit, quota e concurrency limit sono controlli diversi:

- **Rate:** richieste o punti di costo per finestra temporale.
- **Quota:** consumo complessivo per giorno, mese o subscription.
- **Concorrenza:** richieste/stream in esecuzione simultanea.
- **Limite di payload:** dimensione di body, oggetto, batch o risposta.
- **Limite di complessità:** profondità della query, costo GraphQL o relazioni espanse.

`429` può fornire `Retry-After`, ma gli header rate limit specifici del produttore non sono uniformi. Il client tratta i campi documentati come parte del contratto concreto, non come standard universale. Limita localmente la velocità di interrogazione, distribuisce il budget tra workload e salva ambito, budget residuo e tempo di reset.

Il throttling è un segnale di protezione, non una normale modalità di throughput. Ondate persistenti di 429 indicano paginazione inadeguata, caching mancante, parallelismo eccessivo o capacità insufficiente.

## Body di errore e correlazione

Uno stato HTTP è troppo generico per l'automazione di dominio. RFC 9457 definisce Problem Details con URI `type` stabile, `title`, `status`, `detail` e `instance`, nonché campi di estensione. Il tipo di problema è l'identità leggibile dalla macchina; il testo formulato liberamente non è destinato ai parser ([RFC 9457 – Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457.html)).

Un buon contratto di errore fornisce tipo di errore/problema stabile, stato HTTP/RPC, dettaglio sicuro, percorsi dei campi interessati, Request-/Correlation-ID, ripetibilità e un link alla documentazione. Il client non registra token completi, header Authorization o payload riservati.

### Acquisire risposta di errore con header

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für API-Fehler- und Korrelationsdiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">try {
  Invoke-WebRequest -Uri $env:API_ENDPOINT -Headers @{
    Authorization = "Bearer $env:API_TOKEN"
    Accept = 'application/problem+json, application/json'
  } -ErrorAction Stop
}
catch {
  $http = $_.Exception.Response
  [pscustomobject]@{
    Status = [int]$http.StatusCode
    RequestId = $http.Headers['x-request-id']
    RetryAfter = $http.Headers['Retry-After']
    Body = $_.ErrorDetails.Message
  }
}</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">curl --silent --show-error --fail-with-body \
  --dump-header error.headers --output error.json \
  --header "Authorization: Bearer $API_TOKEN" \
  --header 'Accept: application/problem+json, application/json' \
  "$API_ENDPOINT"
grep -Ei '^(HTTP/|x-request-id:|retry-after:)' error.headers
jq '{type,title,status,detail,instance}' error.json</code></pre>
  </div>
</div>

## RPC, GraphQL e gRPC

Non tutte le API si adattano allo stile delle risorse.

| Stile | Centro del contratto | Punto di forza | Limite operativo |
|---|---|---|---|
| REST/HTTP | risorsa, rappresentazione, semantica HTTP | intermediari web, caching, ampio supporto di strumenti | convenzioni di dettaglio non uniformi |
| RPC | servizio e operazione | mappatura diretta delle azioni di dominio | retry/idempotenza espliciti per metodo |
| GraphQL | schema tipizzato e query del client | selezione flessibile di dati correlati | costo delle query, N+1, spesso HTTP 200 nonostante errori di campo |
| gRPC | servizio/messaggio Protobuf | codegen, HTTP/2, unary e streaming | framing binario, proxy, stato/trailer gRPC |
| API evento | canale, messaggio, tipo di evento | disaccoppiamento ed elaborazione asincrona | ordine, deduplicazione, replay, consumer lag |

La specifica GraphQL definisce linguaggio, sistema di tipi, validazione ed esecuzione; trasporto, autenticazione, rate limit e costi operativi delle query sono contratti aggiuntivi ([GraphQL Specification](https://spec.graphql.org/September2025/)). Gli errori di campo possono comparire insieme a dati parziali; il solo stato HTTP non descrive il risultato.

gRPC mappa channel, RPC e messaggi con prefisso di lunghezza su stream HTTP/2. Lo stato gRPC viene trasmesso nei trailer e va distinto dallo stato HTTP. Le chiamate non sono automaticamente idempotenti; deadline, cancellazione e policy di retry vengono comprese per servizio ([gRPC – What is gRPC?](https://grpc.io/docs/what-is-grpc/), [gRPC over HTTP/2 protocol](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md)). Protocol Buffers fornisce un modello di interfaccia e serializzazione; i numeri di campo sono ancore di compatibilità e, dopo la rimozione, non devono essere riutilizzati con un altro significato ([Protocol Buffers – Language Guide](https://protobuf.dev/programming-guides/proto3/)).

[`grpcurl`](https://github.com/fullstorydev/grpcurl) può usare Server Reflection o descrittori locali per esaminare servizi gRPC. La reflection è essa stessa una superficie esposta e non viene abilitata pubblicamente senza verifica.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für gRPC-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">grpcurl.exe -cacert $env:GRPC_CA -H "authorization: Bearer $env:API_TOKEN" $env:GRPC_TARGET list
grpcurl.exe -cacert $env:GRPC_CA -H "authorization: Bearer $env:API_TOKEN" $env:GRPC_TARGET describe $env:GRPC_SERVICE</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">grpcurl -cacert "$GRPC_CA" \
  -H "authorization: Bearer $API_TOKEN" "$GRPC_TARGET" list
grpcurl -cacert "$GRPC_CA" \
  -H "authorization: Bearer $API_TOKEN" "$GRPC_TARGET" describe "$GRPC_SERVICE"</code></pre>
  </div>
</div>

## Rappresentazioni: JSON è sintassi, lo schema è contratto

RFC 8259 definisce JSON come formato di scambio con oggetti, array, numeri, stringhe, booleani e null. JSON non definisce quale campo sia un ID stabile, se un campo mancante e `null` abbiano lo stesso significato, quale fuso orario abbia un timestamp o se proprietà sconosciute siano tollerate ([RFC 8259 – JSON](https://www.rfc-editor.org/rfc/rfc8259.html)).

Questa semantica appartiene a uno schema e alla documentazione del contratto:

- nome del campo, tipo, formato e unità;
- required, optional, nullable e default;
- read-only/write-only e generato dal server;
- valori enum e comportamento per valori sconosciuti;
- formato dell'ora, fuso orario e precisione;
- ID stabile rispetto al nome visualizzato;
- semantica di riferimento, incorporamento ed eliminazione;
- regola di compatibilità per campi nuovi o rimossi.

JSON Schema definisce vocabolari per la validazione delle istanze JSON. Uno schema può verificare la struttura, ma non sostituisce invarianti di dominio o autorizzazione ([JSON Schema – Specification](https://json-schema.org/specification)). La OpenAPI Specification può descrivere operazioni HTTP, parametri, schemi request/response e security scheme in modo leggibile dalla macchina; non prova che l'implementazione in esecuzione corrisponda al documento ([OpenAPI Specification](https://spec.openapis.org/oas/)).

### Ispezionare risposta e campi senza GUI

[`Invoke-RestMethod`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod) deserializza risposte strutturate; [`Invoke-WebRequest`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest) restituisce più dettagli HTTP. [`curl`](https://curl.se/docs/manpage.html) mostra richiesta/risposta e tempi, [`jq`](https://jqlang.org/manual/) filtra JSON.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für API-Response-Inspektion">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$response = Invoke-WebRequest -Uri $env:API_ENDPOINT -Headers @{
  Accept = 'application/json'
  Authorization = "Bearer $env:API_TOKEN"
}
[pscustomobject]@{
  Status = [int]$response.StatusCode
  ContentType = $response.Headers['Content-Type']
  ETag = $response.Headers['ETag']
  RequestId = $response.Headers['x-request-id']
}
$response.Content | ConvertFrom-Json | ConvertTo-Json -Depth 20</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">curl --silent --show-error --dump-header response.headers \
  --header 'Accept: application/json' \
  --header "Authorization: Bearer $API_TOKEN" \
  "$API_ENDPOINT" |
  jq .
grep -Ei '^(HTTP/|content-type:|etag:|x-request-id:)' response.headers</code></pre>
  </div>
</div>

## Contratti e Contract Drift

Un contratto API completo comprende più degli schemi happy path:

| Ambito contrattuale | Deve essere definito |
|---|---|
| Discovery | URL di base, regione, tenant, endpoint di servizio/metadati |
| Operazione | metodo/RPC/evento, binding dei parametri, effetto collaterale |
| Dati | schema, ID, ordine, semantica null/default, limiti di dimensione |
| Sicurezza | flusso di autenticazione, tipo di credenziale, audience, scope/ruolo, durata del token |
| Errori | stato/codice/tipo di problema, ripetibilità, Request-ID |
| Coerenza | Read-after-write, ritardo di replica, cache ed ETag |
| Quantità | paginazione, filtro, ordinamento, semantica snapshot/cursore |
| Tempo | deadline client/gateway/server, Retry-After, clock skew |
| Ciclo di vita | versione del contratto, deprecazione, sunset e percorso di migrazione |
| Esercizio | quota, SLO, manutenzione, pagina di stato, correlazione con il supporto |

La Contract Drift si verifica quando documento, SDK e implementazione in produzione divergono. Perciò la specifica pubblicata viene salvata come artefatto versionato, validata in CI e verificata su un ambiente di test reale. I client generati riducono il lavoro di digitazione, ma propagano anche errori e breaking change dello schema a molti consumer.

I test consumer-driven possono rendere visibili le assunzioni di un client. Non sostituiscono la semantica del provider: un mock può restituire un `200`, mentre la produzione restituisce un altro header, un'altra paginazione o un nuovo enum dopo un aggiornamento del gateway.

Fin qui la chiamata era tecnicamente valida. Se possa essere eseguita dalla giusta identità per l'oggetto giusto è deciso dalla verifica di sicurezza.

## L'autenticazione non è autorizzazione

Una API key o un token risponde anzitutto a **quale client o principal** sta parlando. L'autorizzazione decide poi quale azione è consentita su quale risorsa e in quale scope. Un token valido può quindi essere correttamente respinto con `403`.

| Procedura | Punto di forza e utilizzo | Rischio operativo |
|---|---|---|
| API key | semplice identificazione del client o ancoraggio della quota | spesso valida a lungo, poco scope, copiabile |
| Basic Auth | nome utente/password via TLS | ciclo di vita della password, limiti MFA/delega |
| mTLS | TLS reciproco, certificato client | PKI, rotazione, terminazione proxy, mapping al principal |
| OAuth Access Token | autorizzazione delegata o workload con scope/audience | acquisizione del token, scadenza, consenso, replay |
| richiesta firmata | integrità di parti selezionate del messaggio | canonicalization, clock skew, nonce/replay store |
| identità di rete | reti private, service mesh, certificati workload | non deve sostituire silenziosamente RBAC di dominio |

OAuth 2.0 definisce ruoli e meccanismi di grant per l'emissione di Access Token; i Bearer Token possono essere usati da chiunque li possieda ([RFC 6749 – OAuth 2.0](https://www.rfc-editor.org/rfc/rfc6749.html), [RFC 6750 – Bearer Token Usage](https://www.rfc-editor.org/rfc/rfc6750.html)). Il BCP di sicurezza RFC 9700 richiede privilegi minimi, Audience Restriction e protezione dei flussi di redirect; vieta il Resource Owner Password Credentials Grant. Per la protezione dal replay cita token legati al mittente tramite mTLS o DPoP ([RFC 9700 – OAuth 2.0 Security BCP](https://www.rfc-editor.org/rfc/rfc9700.html), [RFC 8705 – OAuth mTLS](https://www.rfc-editor.org/rfc/rfc8705.html), [RFC 9449 – DPoP](https://www.rfc-editor.org/rfc/rfc9449.html)).

OpenID Connect aggiunge un livello di identità a OAuth; un ID Token è destinato al client e non è automaticamente un Access Token per un'API ([OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html)). Un JWT è solo un formato compatto di claim. La sola verifica della firma non è sufficiente: devono essere validati algoritmo, issuer, audience, claim temporali, selezione della chiave e claim specifici dell'applicazione ([RFC 7519 – JSON Web Token](https://www.rfc-editor.org/rfc/rfc7519.html), [RFC 8725 – JWT Best Current Practices](https://www.rfc-editor.org/rfc/rfc8725.html)).

### Ottenere un token workload e chiamare l'API

Il flusso Client Credentials è adatto solo se l'applicazione agisce a proprio nome e può conservare la propria credenziale in modo sicuro. Secret, certificato o identità workload federata, endpoint token, audience/resource e scope appartengono alla documentazione della piattaforma concreta.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für OAuth-Client-Credentials-Test">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$token = Invoke-RestMethod -Method Post -Uri $env:TOKEN_ENDPOINT -Body @{
  grant_type = 'client_credentials'
  client_id = $env:CLIENT_ID
  client_secret = $env:CLIENT_SECRET
  scope = $env:API_SCOPE
}
Invoke-RestMethod -Uri $env:API_ENDPOINT -Headers @{
  Authorization = "Bearer $($token.access_token)"
  Accept = 'application/json'
}</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">API_TOKEN="$(curl --silent --show-error --fail \
  --request POST "$TOKEN_ENDPOINT" \
  --data-urlencode grant_type=client_credentials \
  --data-urlencode client_id="$CLIENT_ID" \
  --data-urlencode client_secret="$CLIENT_SECRET" \
  --data-urlencode scope="$API_SCOPE" |
  jq --raw-output .access_token)"
curl --silent --show-error --fail \
  --header "Authorization: Bearer $API_TOKEN" \
  --header 'Accept: application/json' "$API_ENDPOINT" | jq .</code></pre>
  </div>
</div>

I segreti non compaiono né nella riga di comando né nel transcript o nel debug log. L'esempio mostra il flusso del protocollo, non un trasporto di secret appropriato per processi di produzione.

## API Gateway e confini di fiducia

Un gateway può terminare TLS, validare token, riunire routing, quote, filtri di schema, regole WAF e osservabilità. È quindi punto di controllo e ambito di guasto. Il servizio backend non deve presumere silenziosamente che ogni richiesta sia arrivata proprio attraverso questo percorso gateway.

- **Client → Gateway:** identità host pubblica, TLS, DDoS/quota, credenziale client.
- **Gateway → Servizio:** identità mTLS o workload propria; nessuna cieca assunzione di fiducia dall'IP sorgente.
- **Identity Provider → Validator:** metadati dell'issuer, JWKS, rotazione delle chiavi, cache e orologio.
- **Servizio → Archiviazione dati:** autorizzazione di dominio e confine del tenant.
- **Provider webhook → Receiver:** firma, finestra temporale, ID evento e controllo del replay.

RFC 9700 avverte esplicitamente, nel caso di reverse proxy, degli header di forwarding in ingresso non verificati. Il proxy deve ripulire i campi rilevanti per la sicurezza; il collegamento interno deve essere protetto da intercettazione, injection e replay ([RFC 9700 – OAuth 2.0 Security BCP](https://www.rfc-editor.org/rfc/rfc9700.html)).

Lo stato del gateway `200` non prova che una coda o replica a valle sia sana. Al contrario, il backend può essere sano mentre DNS, certificato, validatore del token o quota bloccano ogni client.

Dopo gateway e verifica dei permessi resta la domanda operativa più difficile: cosa è successo quando il client non riceve una risposta in tempo? Un timeout non prova che il server non abbia modificato nulla.

## Timeout, deadline ed esecuzione parziale

«Timeout» non è un risultato del server. Il client sa soltanto che entro il proprio termine non è giunta alcuna risposta utilizzabile. La richiesta può essere fallita prima dell'instaurazione della connessione, scartata al gateway, ancora attiva nel servizio o già committed, mentre è andata persa solo la risposta.

Ogni livello può avere il proprio termine: DNS, connessione, TLS, tempo complessivo del client, proxy, gateway, upstream, database e coda. La deadline esterna deve essere coordinata con i termini interni; altrimenti il client interrompe dopo 30 secondi mentre il server continua a lavorare per 60 secondi e un retry avvia la stessa azione in parallelo.

Quando possibile, un servizio propaga una deadline residua invece di ricominciare il tempo pieno per ogni hop. La cancellazione è best effort: non prova che un effetto collaterale già committed sia stato annullato.

## Retry, backoff e idempotenza

La ripetizione automatica è consentita solo se **classe di errore e operazione** lo consentono. Un client robusto chiarisce:

1. È stata affatto stabilita una connessione?
2. È presente uno stato o un errore di protocollo?
3. L'operazione è safe/idempotente o protetta con deduplicazione?
4. Il server restituisce `Retry-After` o un'indicazione di backoff specifica del prodotto?
5. Resta sufficiente deadline end-to-end?
6. Un retry aumenta il sovraccarico?

L'exponential backoff con jitter evita ondate di retry sincrone. Il numero di tentativi è limitato e parte della latenza complessiva. `401` o `403` non vengono risolti con ripetizioni più frequenti; `429` richiede rispetto della quota; un `500` dopo una POST può lasciare un effetto collaterale parziale nonostante il body di errore.

Per un'operazione di dominio il client memorizza un ID operazione stabile. Il server conserva il risultato o lo stato di deduplicazione almeno quanto la finestra massima di retry. In assenza di tale garanzia, il client legge prima del retry in base a un ID oggetto stabile o a una condizione di ricerca.

## Concorrenza ottimistica

Read-modify-write senza condizione di versione genera Lost Updates:

```text
Client A liest Version 7     Client B liest Version 7
Client A schreibt Änderung  → Version 8
Client B schreibt alten Stand plus Änderung → A geht verloren
```

HTTP supporta richieste condizionali con validator come `ETag`. Il client legge l'ETag e invia `If-Match` durante la modifica; se la rappresentazione è cambiata, il server risponde con `412 Precondition Failed` invece di sovrascrivere uno stato altrui. L'API concreta deve documentare se l'ETag è sufficientemente forte per questa semantica ([RFC 9110 – HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für bedingte API-Änderung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$read = Invoke-WebRequest -Uri $env:OBJECT_ENDPOINT -Headers @{
  Authorization = "Bearer $env:API_TOKEN"
}
$body = $read.Content | ConvertFrom-Json
$body.enabled = $false
Invoke-RestMethod -Method Put -Uri $env:OBJECT_ENDPOINT -Headers @{
  Authorization = "Bearer $env:API_TOKEN"
  'If-Match' = $read.Headers['ETag']
} -ContentType 'application/json' -Body ($body | ConvertTo-Json -Depth 20)</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">ETAG="$(curl --silent --show-error --dump-header headers.txt \
  --header "Authorization: Bearer $API_TOKEN" \
  --output object.json "$OBJECT_ENDPOINT" &amp;&amp;
  grep -i '^etag:' headers.txt | cut -d' ' -f2- | tr -d '\\r')"
jq '.enabled = false' object.json &gt; object.updated.json
curl --silent --show-error --fail-with-body \
  --request PUT --header "Authorization: Bearer $API_TOKEN" \
  --header "If-Match: $ETAG" --header 'Content-Type: application/json' \
  --data-binary @object.updated.json "$OBJECT_ENDPOINT"</code></pre>
  </div>
</div>

## Paginazione, filtri e insiemi coerenti

Un endpoint che oggi restituisce 50 oggetti può restituirne 50'000 domani. La paginazione è parte del contratto:

- **Offset/Page:** semplice, ma inserimenti ed eliminazioni possono generare duplicati o lacune.
- **Cursor/Continuation Token:** codifica il progresso lato server; il token è opaco e non viene interpretato.
- **Keyset:** ordinato per ID di continuazione stabile e univoco.
- **Snapshot:** mantiene una vista coerente su più pagine, ma richiede stato server o un'ancora di versione.

Il client segue il Next-Link o cursore documentato e non lo costruisce su supposizioni. RFC 8288 definisce link tipizzati, ma non una paginazione universale; relazione concreta e forma del body restano parte del contratto API ([RFC 8288 – Web Linking](https://www.rfc-editor.org/rfc/rfc8288.html)).

Filtro e ordinamento devono rimanere stabili tra le pagine. Un ordinamento solo per timestamp non univoco è insufficiente; è necessario un tie-breaker come un ID immutabile. Per API delta/change vengono documentati cursore, scadenza e percorso di risincronizzazione.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für API-Pagination">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$next = $env:COLLECTION_ENDPOINT
$items = while ($next) {
  $page = Invoke-RestMethod -Uri $next -Headers @{
    Authorization = "Bearer $env:API_TOKEN"
  }
  $page.value
  $next = $page.nextLink
}
$items | Sort-Object id -Unique | Export-Csv .\\api-items.csv -NoTypeInformation</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">next="$COLLECTION_ENDPOINT"
while [ -n "$next" ]; do
  page="$(curl --silent --show-error --fail \
    --header "Authorization: Bearer $API_TOKEN" "$next")" || exit 1
  printf '%s\\n' "$page" | jq -c '.value[]'
  next="$(printf '%s\\n' "$page" | jq -r '.nextLink // empty')"
done</code></pre>
  </div>
</div>


## Webhook, eventi e API asincrone

Un successo HTTP sincrono e un'elaborazione di dominio completata sono stati diversi. Un `202 Accepted` conferma secondo RFC 9110 soltanto che il server ha accettato l'elaborazione; l'incarico può fallire successivamente. Un ricevitore webhook, al contrario, conferma spesso soltanto l'accettazione persistita di un evento. Chi equipara `2xx` a «processo aziendale completato» perde proprio quegli stati intermedi rilevanti per code, retry e guasti parziali ([RFC 9110 – HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)).

Un evento utilizzabile operativamente contiene almeno un ID evento stabile, tipo di evento e versione dello schema, momento di creazione, produttore, ID risorsa e, se l'ordine conta dal punto di vista di dominio, una versione della risorsa o della sequenza. CloudEvents standardizza a tale scopo un involucro evento indipendente dal produttore; AsyncAPI descrive canali e operazioni di messaggistica in modo leggibile dalla macchina, in modo analogo al ruolo di OpenAPI per API request/response ([CloudEvents Specification](https://github.com/cloudevents/spec), [AsyncAPI Specification](https://www.asyncapi.com/docs/reference/specification/latest)).

I webhook vengono gestiti come un client esterno che ripete le chiamate:

- Il mittente firma il **body della richiesta non modificato** insieme a metadati temporali o nonce; il ricevitore convalida firma, finestra temporale accettata e contesto di destinazione prima del parsing. HTTP Message Signatures standardizzate possono vincolare crittograficamente componenti e campi derivati ([RFC 9421 – HTTP Message Signatures](https://www.rfc-editor.org/rfc/rfc9421.html)).
- Il ricevitore deduplica tramite ID evento e memorizza l'accettazione prima della conferma positiva. L'elaborazione viene progettata in modo idempotente.
- I retry hanno durata limitata, backoff e un percorso Dead-Letter o di quarantena. Un replay è registrato e non genera una nuova identità di dominio.
- Un **processo di reconciliation** periodico confronta il sistema sorgente con lo stato locale. I webhook sono acceleratori, non necessariamente l'unica fonte di verità.

## Sicurezza API: oggetto, funzione e flusso dei dati

Una verifica del token superata risponde soltanto a chi, o quale workload, stia parlando e per quale audience sia destinata la credenziale. Per ogni oggetto e operazione, l'applicazione deve inoltre decidere se questa identità possa leggere o modificare proprio quel tenant, utente, chiave o insieme di messaggi. OWASP API Security Top 10 evidenzia quindi, tra l'altro, Broken Object Level Authorization, Broken Authentication, consumo illimitato di risorse, SSRF e inventario API difettoso come classi di rischio autonome ([OWASP API Security Top 10](https://owasp.org/API-Security/editions/2023/en/0x11-t10/)).

Per le API di infrastruttura ne derivano controlli concreti:

- **Riferimento all'oggetto:** ancore di tenant e oggetto provengono da contesto validato lato server, non solo da un campo di percorso o body liberamente selezionabile.
- **Limiti in ingresso:** Content-Type, schema, lunghezze dei campi, annidamento, dimensione complessiva, rapporto di compressione e tempo di elaborazione sono limitati.
- **Connessioni in uscita:** URL da richieste o webhook passano attraverso allowlist, verifica DNS/IP ed egress policy; i redirect vengono riesaminati.
- **Credenziali:** i token non compaiono né nell'URI né nei log; i secret vengono ruotati, limitati all'audience di destinazione e agli scope minimi e non incorporati in artefatti client.
- **Trust hop:** se un gateway termina [TLS](/kb/tls), l'hop backend deve essere autenticato e autorizzato separatamente. Un header Forwarded affidabile nasce solo su un confine proxy controllato.
- **Audit:** le modifiche privilegiate registrano client, principal, oggetto di destinazione, azione, riferimento prima/dopo, Request-ID e risultato, senza secret o payload sensibile completo.

OAuth 2.0 Security Best Current Practice sconsiglia tra l'altro il Resource Owner Password Credentials Grant, richiede confronti esatti degli URI di redirect e privilegia token legati al mittente o di breve durata quando il modello di minaccia lo richiede ([RFC 9700 – Best Current Practice for OAuth 2.0 Security](https://www.rfc-editor.org/rfc/rfc9700.html)). Mutual TLS e DPoP sono due procedure differenti per il legame al mittente; entrambe modificano l'esercizio delle chiavi e la diagnosi degli errori e non sono semplici interruttori sul gateway ([RFC 8705 – OAuth 2.0 Mutual-TLS Client Authentication](https://www.rfc-editor.org/rfc/rfc8705.html), [RFC 9449 – OAuth 2.0 Demonstrating Proof of Possession](https://www.rfc-editor.org/rfc/rfc9449.html)).

Retry, paginazione ed eventi generano più operazioni tecniche per un'azione di dominio. Correlazione e audit devono ricomporle in un processo tracciabile.

## Osservabilità e chiamate dimostrabili

Le metriche mostrano il volume, i log le singole decisioni e i trace il percorso di una richiesta attraverso i confini di processo. OpenTelemetry modella un trace come insieme causale di span e definisce ID trace e span per la correlazione ([OpenTelemetry – Traces](https://opentelemetry.io/docs/specs/otel/trace/)). Per scopi amministrativi, una chiamata API dovrebbe consentire di ricostruire almeno i seguenti fatti:

| Dimensione | Evidenza operativa |
|---|---|
| Chiamante | ID client, workload o utente; metodo di autenticazione; ruoli/scope effettivi |
| Destinazione | host, tenant, versione API/contratto, metodo o operazione, identificatore stabile della risorsa |
| Runtime | ora di inizio, durata totale, tempo DNS/connessione/TLS se disponibile, deadline, numero di retry |
| Risultato | stato di trasporto, codice di errore di dominio, dimensione della risposta, stato rate-limit/quota |
| Correlazione | Request-ID del server, Trace-ID, Job-/Event-ID e, per messaggistica, Message-ID |

Gli ID vengono propagati oltre i confini di processo, ma non adottati ciecamente come autorità interna da client esterni arbitrari. Le etichette delle metriche evitano ID utente, percorsi completi e altri valori ad alta cardinalità. Payload, header Authorization, cookie e firme webhook non appartengono normalmente alla telemetria. Un trace può dimostrare il percorso, ma non sostituisce una prova di audit protetta da manomissioni su una modifica privilegiata.

## Versionamento, deprecazione e sunset

Un numero di versione non è un ciclo di vita. Innanzitutto si distingue tra **estensioni compatibili** e **breaking change**. Nuovi campi opzionali, valori enum aggiuntivi o un ordine modificato possono rompere i client nonostante una presunta compatibilità all'indietro, se questi implementano il contratto in modo troppo restrittivo. I test consumer e lo schema diffing verificano quindi non solo percorsi, ma anche semantica, autorizzazioni, errori e valori limite.

Le versioni possono trovarsi nel percorso, host, header o media type; è decisivo che routing, documentazione, telemetria e supporto denominino chiaramente la stessa variante. Per la dismissione, RFC 9745 standardizza il campo HTTP `Deprecation`; RFC 8594 definisce `Sunset` come momento dal quale una risorsa presumibilmente non risponderà più. Nessuno dei due sostituisce istruzioni di migrazione, link alternativi o un inventario dimostrato dei client ([RFC 9745 – The Deprecation HTTP Response Header Field](https://www.rfc-editor.org/rfc/rfc9745.html), [RFC 8594 – The Sunset HTTP Header Field](https://www.rfc-editor.org/rfc/rfc8594.html)).

Un processo di deprecazione affidabile comprende inventario dei consumer, metriche di utilizzo per versione e client, date annunciate, esercizio parallelo, ambiente di test, percorso di fallback e una decisione di spegnimento esplicita. «Annunciato nel wiki» non è prova che le automazioni non supervisionate siano state migrate.

## Modelli operativi: locale, cloud e control plane

Il luogo di un'API non determina da solo sicurezza o governabilità. Un'interfaccia locale può essere direttamente legata a account di sistema operativo privilegiati, chiavi di lunga durata e reti poco segmentate. Un cloud control plane può invece offrire solide identità workload e audit log, ma rimane dipendente dal percorso Internet, IAM del provider, configurazione del tenant, quote e disponibilità del servizio. Decisivo è lo spazio concreto di errori e fiducia.

| Modello | Confine tipico | Domande dell'amministratore |
|---|---|---|
| API locale di processo/host | Unix Socket, Named Pipe, loopback o LAN di gestione | Quale identità OS si applica? Chi possiede socket/ACL? L'accesso remoto è davvero escluso? |
| API di servizio interna | segmento, service mesh, gateway o load balancer | Dove terminano TLS e autorizzazione? Come vengono gestite identità di servizio e DNS? |
| SaaS control plane | endpoint provider e tenant IAM | Quali percorsi di regione, quota, audit e token valgono? Come funziona il Break Glass? |
| piano dati più control plane | configurazione controlla worker o appliance separati | Quando una modifica è distribuita? Come vengono rilevati drift, rollback e stati parziali? |
| integrazione evento/webhook | produttore, broker o callback pubblico | Chi gestisce consegna, retry, firma, DLQ e reconciliation? |

I backup non proteggono automaticamente un'API esterna. Per il ripristino vengono invece inventariati contratti, configurazione client, riferimenti ai secret, certificati, regole gateway, stato di idempotenza, job aperti e capacità di reconciliation. I test di recovery devono coprire anche token scaduti, destinazioni DNS modificate e Continuation Token reimpostati.

## Storia tecnica

Le prime interfacce distribuite erano spesso strettamente legate a Remote Procedure Call e stub specifici del linguaggio. SOAP 1.2 ha successivamente definito un framework di messaggistica basato su XML con modello di elaborazione estensibile e si è diffuso, insieme a WSDL e WS-*, nelle piattaforme aziendali ([W3C – SOAP Version 1.2 Part 1](https://www.w3.org/TR/soap12-part1/)). La tesi di Roy Fielding ha descritto nel 2000 REST come stile architetturale per sistemi hypermedia distribuiti e ha derivato i vincoli dai requisiti del Web, non come ricetta «HTTP più JSON» ([Fielding – Architectural Styles and the Design of Network-based Software Architectures](https://ics.uci.edu/~fielding/pubs/dissertation/top.htm)).

HTTP si è evoluto parallelamente dalle connessioni TCP persistenti in HTTP/1.1, agli stream multiplexati in HTTP/2, fino a HTTP/3 su QUIC. La metodologia e la semantica degli stati sono descritte in modo indipendente dal trasporto in RFC 9110; i formati wire si trovano in RFC 9112, RFC 9113 e RFC 9114 ([RFC 9112 – HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112.html), [RFC 9113 – HTTP/2](https://www.rfc-editor.org/rfc/rfc9113.html), [RFC 9114 – HTTP/3](https://www.rfc-editor.org/rfc/rfc9114.html)).

JSON è stato standardizzato come formato di scambio leggero; JSON Schema e OpenAPI hanno aggiunto contratti di struttura e operazione leggibili dalla macchina. GraphQL descrive un modello di query ed esecuzione tipizzato, in cui i client scelgono i campi; gRPC collega definizioni RPC orientate ai servizi con Protocol Buffers e framing basato su HTTP/2 ([JSON Schema Specification](https://json-schema.org/specification), [OpenAPI Specification](https://spec.openapis.org/oas/), [GraphQL Specification](https://spec.graphql.org/September2025/), [gRPC – What is gRPC?](https://grpc.io/docs/what-is-grpc/), [Protocol Buffers – Language Guide](https://protobuf.dev/programming-guides/proto3/)). I modelli di evento e streaming completano request/response, ma non eliminano né i contratti né le questioni di consegna e coerenza.

## Checklist amministrativa in sintesi

Dopo contratto, runtime ed esercizio, la checklist seguente concentra le domande alle quali occorre rispondere prima di approvare un'API. È concepita come supporto al collaudo, non come sostituto delle spiegazioni precedenti.

| Domanda | Evidenza o artefatto |
|---|---|
| Quale stile di interazione gestisco? | OpenAPI/schema GraphQL/Proto/AsyncAPI, operazione concreta e profilo di trasporto |
| Quale endpoint è valido? | Scheme, FQDN, porta, Base Path, regione/tenant, evidenza DNS e certificato |
| Chi chiama? | ID client/workload, tipo di credenziale, issuer del token, audience, scope/ruoli, possesso della chiave |
| Qual è il contratto? | metodi, schemi, catalogo di stati ed errori, limiti, paginazione, idempotenza e ciclo di vita |
| Quando è possibile ripetere? | deadline, semantica o chiave idempotente, backoff, budget retry e operazione di lookup |
| Come evito Lost Updates? | ETag/`If-Match`, numero di versione di dominio o operazione transazionale |
| Come riconosco stati parziali? | stato job/evento, Request-ID, trace, Queue-/Consumer-Lag, reconciliation |
| Come viene modificato? | staging/canary, test di contratto e consumer, rollback, deprecazione/sunset |
| Come viene ripristinato? | configurazione, contratti, riferimenti a secret/certificati, cursori/job, test di replay e reconciliation |
| Cosa va nel runbook? | codici di errore noti, percorsi 401/403/404/409/412/429/5xx, contatti e dati di escalation |

## Fonti

- [Fielding – Network-based Application Architectures](https://ics.uci.edu/~fielding/pubs/dissertation/net_app_arch.htm)
- [Fielding – Network-based Architectural Styles](https://ics.uci.edu/~fielding/pubs/dissertation/net_arch_styles.htm)
- [Fielding – Representational State Transfer](https://ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)
- [RFC 3986 – Uniform Resource Identifier](https://www.rfc-editor.org/rfc/rfc3986.html)
- [RFC 9110 – HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)
- [RFC 6585 – Additional HTTP Status Codes](https://www.rfc-editor.org/rfc/rfc6585.html)
- [RFC 9112 – HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112.html)
- [RFC 9113 – HTTP/2](https://www.rfc-editor.org/rfc/rfc9113.html)
- [RFC 9114 – HTTP/3](https://www.rfc-editor.org/rfc/rfc9114.html)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)
- [BIND 9 – dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc](https://man.openbsd.org/nc)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [RFC 9111 – HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111.html)
- [RFC 9457 – Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457.html)
- [GraphQL Specification](https://spec.graphql.org/September2025/)
- [gRPC – What is gRPC?](https://grpc.io/docs/what-is-grpc/)
- [gRPC over HTTP/2 Protocol](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md)
- [Protocol Buffers – Language Guide](https://protobuf.dev/programming-guides/proto3/)
- [grpcurl](https://github.com/fullstorydev/grpcurl)
- [RFC 8259 – JSON](https://www.rfc-editor.org/rfc/rfc8259.html)
- [JSON Schema Specification](https://json-schema.org/specification)
- [OpenAPI Specification](https://spec.openapis.org/oas/)
- [Microsoft Learn – Invoke-RestMethod](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod)
- [Microsoft Learn – Invoke-WebRequest](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest)
- [curl manual](https://curl.se/docs/manpage.html)
- [jq manual](https://jqlang.org/manual/)
- [RFC 6749 – OAuth 2.0 Authorization Framework](https://www.rfc-editor.org/rfc/rfc6749.html)
- [RFC 6750 – OAuth 2.0 Bearer Token Usage](https://www.rfc-editor.org/rfc/rfc6750.html)
- [RFC 9700 – Best Current Practice for OAuth 2.0 Security](https://www.rfc-editor.org/rfc/rfc9700.html)
- [RFC 8705 – OAuth 2.0 Mutual-TLS Client Authentication](https://www.rfc-editor.org/rfc/rfc8705.html)
- [RFC 9449 – OAuth 2.0 Demonstrating Proof of Possession](https://www.rfc-editor.org/rfc/rfc9449.html)
- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- [RFC 7519 – JSON Web Token](https://www.rfc-editor.org/rfc/rfc7519.html)
- [RFC 8725 – JSON Web Token Best Current Practices](https://www.rfc-editor.org/rfc/rfc8725.html)
- [RFC 8288 – Web Linking](https://www.rfc-editor.org/rfc/rfc8288.html)
- [CloudEvents Specification](https://github.com/cloudevents/spec)
- [AsyncAPI Specification](https://www.asyncapi.com/docs/reference/specification/latest)
- [RFC 9421 – HTTP Message Signatures](https://www.rfc-editor.org/rfc/rfc9421.html)
- [OWASP API Security Top 10](https://owasp.org/API-Security/editions/2023/en/0x11-t10/)
- [OpenTelemetry – Traces](https://opentelemetry.io/docs/specs/otel/trace/)
- [RFC 9745 – Deprecation](https://www.rfc-editor.org/rfc/rfc9745.html)
- [RFC 8594 – Sunset](https://www.rfc-editor.org/rfc/rfc8594.html)
- [W3C – SOAP Version 1.2 Part 1](https://www.w3.org/TR/soap12-part1/)
- [Fielding – Architectural Styles and the Design of Network-based Software Architectures](https://ics.uci.edu/~fielding/pubs/dissertation/top.htm)
