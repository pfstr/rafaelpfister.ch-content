---
title: "API-er: kontrakter, identiteter og distribuerte feilgrenser"
blatt: "apis"
description: "API-er for infrastruktur- og meldingsadministratorer: REST, HTTP og JSON, RPC, GraphQL, gRPC og event-API-er, OpenAPI og skjemaer, OAuth og arbeidsbelastningsidentiteter, gatewayer, tidsavbrudd, nye forsøk, idempotens, paginering, rategrenser, webhooks, observerbarhet, versjonering og teknisk historie."
fakten:
  - label: Systemrolle
    wert: maskinlesbart administrasjons- eller datagrensesnitt mellom separate komponenter
    href: https://ics.uci.edu/~fielding/pubs/dissertation/net_app_arch.htm
  - label: REST
    wert: arkitekturstil med constraints; ikke synonymt med HTTP pluss JSON
    href: https://ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm
  - label: HTTP-modell
    wert: ressurs/URI · metode · felt · representasjon · status
    href: https://www.rfc-editor.org/rfc/rfc9110.html
  - label: Kontrakt
    wert: beskrive operasjoner, skjemaer, feil, sikkerhet og livssyklus maskinlesbart
    href: https://spec.openapis.org/oas/
  - label: Dataformater
    wert: JSON · XML · Protocol Buffers · binær- og strømmeformater
    href: https://www.rfc-editor.org/rfc/rfc8259.html
  - label: Interaksjonsstiler
    wert: request/response · RPC · spørring · strøm · event/webhook
    href: https://ics.uci.edu/~fielding/pubs/dissertation/net_arch_styles.htm
  - label: Identitet
    wert: API-nøkkel, klientsertifikat eller token; credential og rettighet er atskilt
    href: https://www.rfc-editor.org/rfc/rfc9700.html
  - label: Gjenta
    wert: bare med kjent semantikk; tidsavbrudd betyr ikke at ingenting skjedde på serveren
    href: https://www.rfc-editor.org/rfc/rfc9110.html
  - label: Samtidighet
    wert: ETag/If-Match eller faglig versjonsnummer forhindrer Lost Updates
    href: https://www.rfc-editor.org/rfc/rfc9110.html
  - label: Feilformat
    wert: HTTP-status pluss stabil maskinlesbar problemtype og request-ID
    href: https://www.rfc-editor.org/rfc/rfc9457.html
  - label: Driftstilstand
    wert: latens · feilrate · metning · kvote · tokenutløp · kø/Consumer Lag
    href: https://opentelemetry.io/docs/specs/otel/trace/
  - label: Adminbevis
    wert: klient · identitet · scope · endpoint · kontraktsversjon · request-ID · resultat
    href: https://www.rfc-editor.org/rfc/rfc9110.html
werbung:
  - newsletter
ctaThemen:
  - cloudflare-workers
  - powershell
  - automatisierung
translationSourceHash: 4b29095b24ea1576608e147b1904a65a72c2b09b2e72f991e3e27a64dbcb3332
translationModel: gpt-5.6-terra
translatedAt: 2026-10-03T09:35:00.814Z
translationReview: automatic
---

# API-er: kontrakter, identiteter og distribuerte feilgrenser

Et Application Programming Interface er en **kontrakts- og tillitsgrense** mellom uavhengig driftede komponenter. Kontrakten fastsetter hvilke operasjoner og data som finnes; kjøretiden avgjør transport, identitet, autorisasjon, tidsatferd og feil. For administratorer er dette skillet sentralt: En syntaktisk gyldig JSON-request kan havne hos feil tenant, mislykkes med et gyldig token mot feil audience eller likevel ha blitt utført på serversiden etter en klient-timeout.

API-er finnes overalt i meldingsmiljøer: mellom administrasjonsklient og e-postplattform, gateway og katalog, overvåking og telemetri-backend, skyapplikasjon og webhook-mottaker. En GUI kan bruke det samme grensesnittet, men gjengir vanligvis bare en del av tilstandene og feilene. Robust drift begynner derfor ikke med det enkelte curl-kallet, men med **grensesnittstil, kontrakt, ressursmodell, identitet, tilstandsoverganger og gjenopprettingssemantikk**.

Forklaringen følger et API-kall fra klienten til det faglige svaret. Først handler det om transport og kontrakt, deretter om identitet, feilbehandling og hendelser; først etterpå følger gatewaydrift, sikkerhet og diagnose.

## Plassering som distribuert system

Et nettverks-API er ikke bare applikasjonskode. Et typisk kall går gjennom:

1. klientbibliotek, CLI eller automatiseringsprosess;
2. navneoppløsning, ruting og forbindelsesopprettelse;
3. [TLS](/kb/tls), proxy eller service mesh;
4. lastbalanserer, API-gateway eller reverse proxy;
5. autentisering, tokenkontroll og autorisasjon;
6. applikasjonstjeneste, cache, kø og database;
7. svarbane, serialisering og klientevaluering.

Roy Fieldings arkitekturarbeid skiller uttrykkelig nettverksbaserte systemer fra transparent lokal utførelse: Nettverkskommunikasjon har egen latens, egne kostnader og feilmoduser. En arkitekturstil er et koordinert sett med constraints som frembringer bestemte egenskaper og avveininger ([Fielding – Network-based Application Architectures](https://ics.uci.edu/~fielding/pubs/dissertation/net_app_arch.htm), [Fielding – Network-based Architectural Styles](https://ics.uci.edu/~fielding/pubs/dissertation/net_arch_styles.htm)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-apis.svg?v=20260813" title="Interaktive Infografik: API-Aufruf über DNS, TLS, Gateway, Identität und Dienst sowie Vertrag, Fehler- und Wiederholungssemantik, Events und Betriebsnachweis" loading="lazy">
  <a href="/images/kb-interaktiv-apis.svg?v=20260813">Åpne interaktiv grafikk direkte</a>.
</iframe>

Den relevante stakken dokumenteres per integrasjon:

| Lag | Eksempler | Typisk feilområde |
|---|---|---|
| Identifikasjon | URI, Service Discovery, DNS | feil host, region, tenant eller API-sti |
| Transport | TCP/TLS, HTTP/1.1, HTTP/2, HTTP/3 | timeout, proxy, sertifikat, ALPN, connection pool |
| Interaksjon | REST, RPC, GraphQL, gRPC, webhook/event | feil semantikk, uegnet retry, strømbrudd |
| Representasjon | JSON, XML, Protobuf, Multipart, binærdata | skjema-, encoding-, størrelses- eller kompatibilitetsfeil |
| Kontrakt | OpenAPI, JSON Schema, Protobuf IDL, GraphQL SDL, AsyncAPI | breaking change, avvik mellom dokumentasjon og kjøretid |
| Identitet | API-key, OAuth-token, mTLS, signert request | utløp, scope, audience, nøkkelrotasjon, clock skew |
| Policy | Gateway, WAF, RBAC/ABAC, quota | 401/403/429, headernormalisering, feil principal |
| Tilstand | Service, cache, kø, database | delvis utførelse, replikasjonsforsinkelse, eventual consistency |
| Evidens | Request-ID, trace, auditlogg, måledata | manglende korrelasjon, sampling, personvern |

## REST er en arkitekturstil, ikke et dataformat

REST betegner constraintene Fielding beskrev for distribuerte hypermediasystemer: klient/server, statelessness, cache, enhetlig grensesnitt, layering og valgfri code-on-demand. Det enhetlige grensesnittet omfatter ressursidentifikasjon, manipulering gjennom representasjoner, selvbeskrivende meldinger og hypermedia som tilstandsmaskin. Standardiseringen av grensesnittet forbedrer synlighet og uavhengig utvikling, men kan være mindre effektiv enn spesialiserte protokoller ([Fielding – Representational State Transfer](https://ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)).

Et HTTP-API med JSON og stier som `/v1/getUser` er derfor ikke automatisk REST. Det kan ganske enkelt være RPC over HTTP. Det er ikke grunnleggende dårlig; problematisk blir det når operatører forventer egenskaper som den faktiske stilen ikke tilbyr. En klient kan for eksempel ikke trygt gjenta en POST bare fordi endpointet kalles «REST».

### URI, ressurs og representasjon

En URI identifiserer en ressurs; den garanterer verken tilgjengelighet eller en bestemt operasjon. RFC 3986 skiller uttrykkelig identifikasjon fra interaksjon. Scheme, Authority, Path, Query og Fragment har definert syntaks, mens det konkrete API-et fastsetter ressurssemantikken ([RFC 3986 – URI Generic Syntax](https://www.rfc-editor.org/rfc/rfc3986.html)).

En ressurs er ikke JSON-filen sin. Den samme ressursen kan representeres som JSON, XML eller et annet format avhengig av `Accept`-headeren. `Content-Type` beskriver den sendte bodyen, `Accept` det foretrukne svaret. Status, felter og body utgjør sammen meldingen; å logge bare JSON-bodyen fjerner viktig diagnoseinformasjon.

## HTTP-semantikk: metode før stinavn

RFC 9110 skiller ressursidentifikasjon fra request-semantikk. Metoden definerer den tiltenkte operasjonen; URI-stien alene gjør ikke det. «Safe» betyr at klienten ikke har til hensikt å endre tilstand. «Idempotent» betyr at flere identiske requests har samme tiltenkte virkning som én request; bivirkninger som logging kan likevel oppstå flere ganger ([RFC 9110 – HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)).

| Metode | Safe | Idempotent | Typisk API-semantikk | Retry uten tilleggsinformasjon |
|---|---:|---:|---|---|
| GET | ja | ja | lese representasjon | i utgangspunktet mulig, men ta hensyn til last/quota |
| HEAD | ja | ja | metadata uten body | i utgangspunktet mulig |
| OPTIONS | ja | ja | egenskaper/kommunikasjonsalternativer | i utgangspunktet mulig |
| PUT | nei | ja | erstatte tilstand under kjent URI | mulig hvis kontrakten faktisk overholder PUT-semantikk |
| DELETE | nei | ja | fjerne tilordning | virkningen kan gjentas; svarstatusen kan endres |
| POST | nei | nei | behandle, handling eller ny ressurs | ikke gjenta blindt |
| PATCH | nei | ikke generelt | delvis endring | bare med dokumentert patch- og idempotenssemantikk |

Idempotens beskriver den **tiltenkte servervirkningen**, ikke transportsvaret. En PUT kan være fullført på serveren mens svaret går tapt. En ny PUT er da semantisk forsvarlig; en ny POST kan opprette et nytt objekt eller en ny melding. For ikke-idempotente operasjoner er det nødvendig med en operasjonsidentifikator opprettet av klienten, en leverandørspesifikk Idempotency-Key eller et påfølgende statusoppslag.

### Statuskoder er kategorier, ikke en komplett diagnose

- `2xx`: Requesten ble behandlet slik statusen definerer; ikke hver `202 Accepted` er allerede faglig fullført.
- `3xx`: videre handling eller annen representasjon; redirects kan endre metode og credentialflyt.
- `400`: Requesten er feil fra serverens perspektiv.
- `401`: manglende eller ugyldige autentiseringscredentials; svaret bruker som hovedregel `WWW-Authenticate`.
- `403`: Serveren forstår requesten, men avviser den.
- `404`: Ressursen ble ikke funnet eller er bevisst skjult; dette er ikke et sikkert bevis på at den ikke eksisterer.
- `409`: Konflikt med gjeldende tilstand.
- `412`: Precondition som `If-Match` er ikke oppfylt.
- `429`: for mange requests i et tidsvindu; `Retry-After` kan oppgi en ventetid.
- `5xx`: Serveren kunne ikke oppfylle en i utgangspunktet gyldig request; ikke automatisk mulig å retry.

RFC 6585 definerer `429 Too Many Requests`, men verken quotascope eller teller. Disse kan gjelde per credential, bruker, tenant, ressurs, region eller cluster ([RFC 6585 – Additional HTTP Status Codes](https://www.rfc-editor.org/rfc/rfc6585.html)). Klienten lagrer derfor status, relevante responsfelter, request-ID og forkortet feilbody.

## Transportvarianter og forbindelseskostnader

HTTP-semantikk er skilt fra den konkrete wire-versjonen. HTTP/1.1 bruker tekstlige meldingsframingregler, HTTP/2 multipleksede strømmer og binær framing, HTTP/3 bygger HTTP på QUIC. En API-gateway kan godta HTTP/2 på klientsiden og bruke HTTP/1.1 mot backend; protokollen hos klienten beviser ikke hele backendbanen ([RFC 9112 – HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112.html), [RFC 9113 – HTTP/2](https://www.rfc-editor.org/rfc/rfc9113.html), [RFC 9114 – HTTP/3](https://www.rfc-editor.org/rfc/rfc9114.html)).

Administratorer observerer ikke bare requestlatens, men DNS- og tilkoblingstid, TLS-handshake, connection reuse, HTTP-versjon/ALPN, proxy-/gatewaytid, Time to First Byte, bodyoverføring, antall retries og samlet wall-clock-varighet.

### Kontroller navne-, TCP- og TLS-banen

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) og [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) kontrollerer [DNS](/kb/dns). [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) og [`nc`](https://man.openbsd.org/nc) kontrollerer TCP. [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) viser TLS-handshake, sertifikatbane og ALPN; ingen av disse testene beviser vellykket API-autorisasjon.

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
Når metode, transport og representasjon er forstått, følger den første distribuerte særegenheten: En vellykket endring trenger ikke være synlig umiddelbart på alle leseveier.

## Caching og read-after-write

HTTP-caching lagrer representasjoner basert på cache keys og direktiver. `Cache-Control`, `Vary`, validators og autentiseringsregler avgjør om og hvordan de gjenbrukes. En `200` kan komme fra en cache; en GET rett etter PUT kan, avhengig av arkitekturen, fortsatt se gammel tilstand ([RFC 9111 – HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111.html)).

Adminspørsmål:

- Finnes det en browser-, proxy-, CDN-, gateway- eller applikasjonscache i banen?
- Hvilke headere utgjør cache key, særlig Authorization og tenant?
- Er representasjonen privat, offentlig eller ikke cachebar?
- Hvor lenge kan negative svar caches?
- Finnes read-your-writes eller eventual consistency?
- Hvilken region/replika leser den påfølgende GET-en?
- Er ETag en innholdsversjon eller bare en cachevalidator?

Å tømme cachen er ikke en universell reparasjon. Det kan skape lasttopper og skjule den egentlige inkonsistensen.

## Rate limits, quotaer og metning

Rate limit, quota og concurrency limit er ulike kontroller:

- **Rate:** requests eller kostnadspoeng per tidsvindu.
- **Quota:** samlet forbruk per dag, måned eller subscription.
- **Concurrency:** requests/streams som kjører samtidig.
- **Payloadlimit:** body-, objekt-, batch- eller svarstørrelse.
- **Kompleksitetslimit:** spørringsdybde, GraphQL Cost eller utvidede relasjoner.

`429` kan levere `Retry-After`, men leverandørspesifikke rate-limit-headere er ikke ensartede. Klienten behandler dokumenterte felter som del av den konkrete kontrakten, ikke som en universell standard. Den begrenser spørringsraten lokalt, fordeler budsjett mellom workloads og lagrer scope, gjenstående budsjett og reset-tid.

Throttling er et beskyttelsessignal, ikke en normal gjennomstrømningsmodus. Vedvarende 429-bølger peker på uegnet paginering, manglende caching, for høy parallellitet eller utilstrekkelig kapasitet.

## Feilkropper og korrelasjon

En HTTP-status er for grov for faglig automatisering. RFC 9457 definerer Problem Details med stabil `type`-URI, `title`, `status`, `detail` og `instance` samt utvidelsesfelter. Problemtypen er den maskinlesbare identiteten; fritt formulert tekst er ikke ment for parsere ([RFC 9457 – Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457.html)).

En god feilkontrakt gir stabil feil-/problemtype, HTTP-/RPC-status, sikker detaljangivelse, berørte feltstier, request-/correlation-ID, retrybarhet og en dokumentasjonslenke. Klienten logger ikke komplette tokens, Authorization-headere eller konfidensielle payloads.

### Registrer feilsvar med headere

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

## RPC, GraphQL og gRPC

Ikke alle API-er passer til ressursstilen.

| Stil | Kontraktssentrum | Styrke | Driftsgrense |
|---|---|---|---|
| REST/HTTP | Ressurs, representasjon, HTTP-semantikk | webmellomledd, caching, bred verktøystøtte | uensartede detaljkonvensjoner |
| RPC | Tjeneste og operasjon | direkte avbildning av faglige handlinger | retry-/idempotens per metode eksplisitt |
| GraphQL | typet skjema og klientspørring | fleksibelt utvalg av sammenhengende data | spørringskostnader, N+1, ofte HTTP 200 til tross for feltfeil |
| gRPC | Protobuf Service/Message | codegen, HTTP/2, unary og streaming | binær framing, proxier, gRPC-status/trailers |
| Event API | Channel, Message, eventtype | frikobling og asynkron behandling | rekkefølge, deduplisering, replay, consumer lag |

GraphQL-spesifikasjonen definerer språk, typesystem, validering og utførelse; transport, autentisering, rate limit og operative spørringskostnader er tilleggskontrakter ([GraphQL Specification](https://spec.graphql.org/September2025/)). Feltfeil kan forekomme sammen med delvise data; en HTTP-status alene beskriver ikke resultatet.

gRPC avbilder channels, RPC-er og length-prefixed meldinger på HTTP/2-streamer. gRPC-statusen overføres i trailers og må skilles fra HTTP-statusen. Kall er ikke automatisk idempotente; deadline, cancellation og retry policy forstås per tjeneste ([gRPC – What is gRPC?](https://grpc.io/docs/what-is-grpc/), [gRPC over HTTP/2 protocol](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md)). Protocol Buffers gir en grensesnitt- og serialiseringsmodell; feltnumre er kompatibilitetsankere og må ikke gjenbrukes for en annen betydning etter fjerning ([Protocol Buffers – Language Guide](https://protobuf.dev/programming-guides/proto3/)).

[`grpcurl`](https://github.com/fullstorydev/grpcurl) kan bruke server reflection eller lokale descriptors for å undersøke gRPC-tjenester. Reflection er selv en eksponert flate og skal ikke offentlig aktiveres ukontrollert.

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

## Representasjoner: JSON er syntaks, skjema er kontrakt

RFC 8259 definerer JSON som utvekslingsformat med objekter, arrays, tall, strenger, booleans og null. JSON definerer ikke hvilket felt som er en stabil ID, om et manglende felt og `null` betyr det samme, hvilken tidssone en timestamp har eller om ukjente egenskaper tolereres ([RFC 8259 – JSON](https://www.rfc-editor.org/rfc/rfc8259.html)).

Denne semantikken hører hjemme i et skjema og i kontraktsdokumentasjonen:

- feltnavn, type, format og enhet;
- required, optional, nullable og default;
- read-only/write-only og servergenerert;
- enumverdier og atferd ved ukjente verdier;
- tidsformat, tidssone og presisjon;
- stabil ID kontra visningsnavn;
- referanse-, innbyggings- og slettesemantikk;
- kompatibilitetsregel ved nye eller fjernede felter.

JSON Schema definerer vokabularer for validering av JSON-instanser. Et skjema kan kontrollere struktur, men erstatter ikke faglige invarianter eller autorisasjon ([JSON Schema – Specification](https://json-schema.org/specification)). OpenAPI Specification kan beskrive HTTP-operasjoner, parametere, request/response-skjemaer og Security Schemes maskinlesbart; den beviser ikke at den kjørende implementasjonen samsvarer med dokumentet ([OpenAPI Specification](https://spec.openapis.org/oas/)).

### Inspiser svar og felter uten GUI

[`Invoke-RestMethod`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod) deserialiserer strukturerte svar; [`Invoke-WebRequest`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest) returnerer flere HTTP-detaljer. [`curl`](https://curl.se/docs/manpage.html) viser request/response og timing, [`jq`](https://jqlang.org/manual/) filtrerer JSON.

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

## Kontrakter og contract drift

En fullstendig API-kontrakt omfatter mer enn happy-path-skjemaer:

| Kontraktsområde | Må være fastlagt |
|---|---|
| Discovery | Base URL, region, tenant, service-/metadata-endpoint |
| Operasjon | metode/RPC/event, parameterbinding, bivirkning |
| Data | skjema, ID-er, rekkefølge, null-/defaultsemantikk, størrelsesgrenser |
| Sikkerhet | authflow, credentialtype, audience, scope/rolle, tokenlevetid |
| Feil | status/kode/problemtype, retrybarhet, request-ID |
| Konsistens | read-after-write, replikasjonsforsinkelse, cache og ETag |
| Mengde | paginering, filter, sortering, snapshot-/cursorsemantikk |
| Tid | klient-/gateway-/serverdeadline, Retry-After, clock skew |
| Livssyklus | kontraktsversjon, deprecation, sunset og migreringsbane |
| Drift | quota, SLO, vedlikehold, statusside, supportkorrelasjon |

Contract drift oppstår når dokumentasjon, SDK og produksjonsimplementering divergerer. Den publiserte spesifikasjonen lagres derfor som et versjonert artefakt, valideres i CI og kontrolleres mot et reelt testmiljø. Generated clients reduserer skrivearbeidet, men overfører også feil og breaking changes i skjemaet til mange consumers.

Consumer-driven tester kan synliggjøre antakelser hos en klient. De erstatter ikke providersemantikk: En mock kan returnere `200`, selv om produksjon etter en gatewayoppdatering returnerer en annen header, en annen paginering eller en ny enum.

Til nå var kallet teknisk gyldig. Om det også kan utføres av riktig identitet for riktig objekt, avgjøres av sikkerhetskontrollen.

## Autentisering er ikke autorisasjon

En API-key eller et token svarer først på **hvilken klient eller principal** som snakker. Autorisasjon avgjør deretter hvilken handling som er tillatt på hvilken ressurs i hvilket scope. Et gyldig token kan derfor korrekt avvises med `403`.

| Metode | Styrke og bruk | Driftsrisiko |
|---|---|---|
| API-key | enkel klientidentifikasjon eller quotaanker | ofte lang gyldighet, lite scope, lett å kopiere |
| Basic Auth | brukernavn/passord over TLS | passordlivssyklus, MFA-/delegeringsgrenser |
| mTLS | gjensidig TLS, klientsertifikat | PKI, rotasjon, proxyterminering, mapping til principal |
| OAuth Access Token | delegert eller workload-rettighet med scope/audience | tokeninnhenting, utløp, consent, replay |
| signert request | integritet for utvalgte meldingsdeler | canonicalization, clock skew, nonce/replay store |
| nettverksidentitet | private nettverk, service mesh, workload-sertifikater | må ikke stilltiende erstatte faglig RBAC |

OAuth 2.0 definerer roller og grantmekanismer for utstedelse av Access Tokens; Bearer Tokens kan brukes av alle som besitter dem ([RFC 6749 – OAuth 2.0](https://www.rfc-editor.org/rfc/rfc6749.html), [RFC 6750 – Bearer Token Usage](https://www.rfc-editor.org/rfc/rfc6750.html)). Security BCP RFC 9700 krever minimale privilegier, Audience Restriction og beskyttelse av redirect flows; den forbyr Resource Owner Password Credentials Grant. For replaybeskyttelse nevner den senderbundne tokens over mTLS eller DPoP ([RFC 9700 – OAuth 2.0 Security BCP](https://www.rfc-editor.org/rfc/rfc9700.html), [RFC 8705 – OAuth mTLS](https://www.rfc-editor.org/rfc/rfc8705.html), [RFC 9449 – DPoP](https://www.rfc-editor.org/rfc/rfc9449.html)).

OpenID Connect legger et identitetslag over OAuth; et ID Token er for klienten og ikke automatisk et Access Token for et API ([OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html)). En JWT er bare et kompakt claimformat. Signaturkontroll alene er ikke nok: Algoritme, issuer, audience, tidsclaims, nøkkelvalg og applikasjonsspesifikke claims må valideres ([RFC 7519 – JSON Web Token](https://www.rfc-editor.org/rfc/rfc7519.html), [RFC 8725 – JWT Best Current Practices](https://www.rfc-editor.org/rfc/rfc8725.html)).

### Hent workload-token og kall API-et

Client Credentials Flow er bare egnet når applikasjonen handler i eget navn og kan holde credentialet sikkert. Secret, sertifikat eller føderert workloadidentitet, tokenendpoint, audience/resource og scope hører hjemme i den konkrete plattformdokumentasjonen.

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

Secrets vises verken på kommandolinjen eller i transkripsjon eller debuglogg. Eksemplet viser protokollflyten, ikke en egnet secrettransport for produksjonsprosesser.

## API-gateway og tillitsgrenser

En gateway kan terminere TLS, validere tokens og samle ruting, quotaer, skjema-filtre, WAF-regler og observerbarhet. Den er dermed kontrollpunkt og feilområde. Backendtjenesten må ikke stilltiende forutsette at enhver request kom via nøyaktig denne gatewaybanen.

- **Klient → Gateway:** offentlig hostidentitet, TLS, DDoS/quota, klientcredential.
- **Gateway → Tjeneste:** egen mTLS- eller workloadidentitet; ingen blind tillitsantakelse basert på kilde-IP.
- **Identity Provider → Validator:** issuer metadata, JWKS, nøkkelrotasjon, cache og klokke.
- **Tjeneste → Datalagring:** faglig autorisasjon og tenantgrense.
- **Webhookprovider → Mottaker:** signatur, tidsvindu, event-ID og replaykontroll.

RFC 9700 advarer uttrykkelig mot ukontrollerte innkommende forwarding-headere ved reverse proxier. Proxyen må rense sikkerhetsrelevante felter; den interne lenken må beskyttes mot avlytting, injection og replay ([RFC 9700 – OAuth 2.0 Security BCP](https://www.rfc-editor.org/rfc/rfc9700.html)).

Gatewaystatusen `200` beviser ikke at en etterfølgende kø eller replikering er sunn. Omvendt kan backend være sunn mens DNS, sertifikat, tokenvalidator eller quota blokkerer alle klienter.

Etter gateway- og rettighetskontroll gjenstår det vanskeligste driftsspørsmålet: Hva skjedde når klienten ikke mottar et rettidig svar? En timeout beviser ikke at serveren ikke endret noe.

## Timeouts, deadlines og delvis utførelse

«Timeout» er ikke et serverresultat. Klienten vet bare at det ikke kom noe brukbart svar innen fristen. Requesten kan ha mislyktes før forbindelsesopprettelsen, blitt forkastet i gatewayen, fortsatt være aktiv i tjenesten eller allerede være committed, mens bare svaret gikk tapt.

Hvert lag kan ha sin egen frist: DNS, connect, TLS, klientens total tid, proxy, gateway, upstream, database og kø. Den ytre deadline må avstemmes med de indre fristene; ellers avbryter klienten etter 30 sekunder mens serveren arbeider videre i 60 sekunder, og en retry starter samme handling parallelt.

En tjeneste propagerer om mulig gjenstående deadline i stedet for å starte med full tid på nytt for hvert hopp. Cancellation er best effort: Den beviser ikke at en allerede committed bivirkning ble reversert.

## Retries, backoff og idempotens

Automatisk gjentakelse er bare tillatt når **feilklasse og operasjon** tillater det. En robust klient avklarer:

1. Ble det i det hele tatt opprettet en forbindelse?
2. Finnes det en status eller en protokollfeil?
3. Er operasjonen safe/idempotent eller beskyttet med deduplisering?
4. Oppgir serveren `Retry-After` eller en produktspesifikk backoff-angivelse?
5. Gjenstår det nok end-to-end-deadline?
6. Forstørrer en retry overbelastningen?

Exponential backoff med jitter hindrer synkrone retrybølger. Antall forsøk er begrenset og en del av den samlede latensen. `401` eller `403` repareres ikke ved hyppigere gjentakelse; `429` krever quotarespekt; en `500` etter en POST kan etterlate en delvis bivirkning til tross for feilbody.

For en faglig operasjon lagrer klienten en stabil operasjons-ID. Serveren beholder resultat eller dedupliseringsstatus minst like lenge som det maksimale retryvinduet. Mangler en slik garanti, leser klienten før retry basert på en stabil objekt-ID eller søkebetingelse.

## Optimistisk samtidighet

Read-modify-write uten versjonsbetingelse skaper Lost Updates:

```text
Client A liest Version 7     Client B liest Version 7
Client A schreibt Änderung  → Version 8
Client B schreibt alten Stand plus Änderung → A geht verloren
```

HTTP støtter betingede requests med validators som `ETag`. Klienten leser ETag-en og sender ved endringen `If-Match`; har representasjonen endret seg, svarer serveren med `412 Precondition Failed` i stedet for å overskrive en fremmed tilstand. Det konkrete API-et må dokumentere om ETag-en er sterk nok for denne semantikken ([RFC 9110 – HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)).

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

## Paginering, filtrering og konsistente mengder

Et endpoint som leverer 50 objekter i dag, kan levere 50 000 i morgen. Paginering er en del av kontrakten:

- **Offset/Page:** enkelt, men innsettinger og slettinger kan skape duplikater eller hull.
- **Cursor/Continuation Token:** koder fremdrift på serversiden; tokenet er opakt og skal ikke tolkes.
- **Keyset:** sortert etter stabil, entydig fortsettelses-ID.
- **Snapshot:** holder en konsistent visning over flere sider, men krever servertilstand eller versjonsanker.

Klienten følger den dokumenterte Next-Link eller cursoren og konstruerer den ikke ut fra antakelser. RFC 8288 definerer typede lenker, men ingen universell paginering; konkret relasjon og bodyform forblir API-kontrakt ([RFC 8288 – Web Linking](https://www.rfc-editor.org/rfc/rfc8288.html)).

Filter og sortering må være stabile på tvers av sider. En sortering bare etter et ikke-entydig tidsstempel er utilstrekkelig; en tie-breaker som en uforanderlig ID hører med. For delta-/change-API-er dokumenteres cursor, utløp og resynkroniseringsbane.

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


## Webhooks, events og asynkrone API-er

En synkron HTTP-suksess og faglig fullført behandling er ulike tilstander. En `202 Accepted` bekrefter ifølge RFC 9110 bare at serveren har akseptert behandlingen; oppdraget kan fortsatt mislykkes senere. En webhook-mottaker bekrefter omvendt ofte bare den persistente aksepten av et event. Den som likestiller `2xx` med «forretningsprosessen er fullført», mister nettopp de mellomtilstandene som er relevante ved køer, retries og delvise forstyrrelser ([RFC 9110 – HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)).

Et operativt brukbart event inneholder minst en stabil event-ID, eventtype og skjemaversjon, opprettelsestid, produsent, ressurs-ID samt – hvis rekkefølgen er faglig relevant – en ressurs- eller sekvensversjon. CloudEvents standardiserer en leverandørnøytral eventkonvolutt for dette; AsyncAPI beskriver meldingskanaler og operasjoner maskinlesbart, på samme måte som OpenAPI for request/response-API-er ([CloudEvents Specification](https://github.com/cloudevents/spec), [AsyncAPI Specification](https://www.asyncapi.com/docs/reference/specification/latest)).

Webhooks driftes som en fremmed, repeterende klient:

- Avsenderen signerer den **uendrede request-bodyen** sammen med tids- eller nonce-metadata; mottakeren validerer signatur, akseptert tidsvindu og målkontekst før parsing. Standardiserte HTTP Message Signatures kan binde komponenter og avledede felter kryptografisk ([RFC 9421 – HTTP Message Signatures](https://www.rfc-editor.org/rfc/rfc9421.html)).
- Mottakeren dedupliserer via event-ID og lagrer aksept før positiv bekreftelse. Behandlingen utformes idempotent.
- Retries har begrenset varighet, backoff og en Dead-Letter- eller karantenebane. En replay loggføres og oppretter ingen ny faglig identitet.
- En periodisk **reconciliation-kjøring** sammenligner kildesystem og lokal tilstand. Webhooks er akseleratorer, ikke nødvendigvis den eneste kilden til sannhet.

## API-sikkerhet: objekt, funksjon og dataflyt

En bestått tokenkontroll svarer bare på hvem eller hvilken workload som snakker, og for hvilken audience credentialet er ment. Applikasjonen må for hvert objekt og hver operasjon i tillegg avgjøre om denne identiteten kan lese eller endre nøyaktig denne tenantens, brukerens, nøkkelens eller meldingsbeholdningens data. OWASP API Security Top 10 fremhever derfor blant annet Broken Object Level Authorization, Broken Authentication, ubegrenset ressursforbruk, SSRF og mangelfullt API-inventar som egne risikoklasser ([OWASP API Security Top 10](https://owasp.org/API-Security/editions/2023/en/0x11-t10/)).

For infrastruktur-API-er følger konkrete kontroller av dette:

- **Objekttilknytning:** Tenant- og objektankere kommer fra servervalidert kontekst, ikke bare fra et fritt valgbart sti- eller bodyfelt.
- **Inngangsgrenser:** Content-Type, skjema, feltlengder, nestingsnivå, samlet størrelse, kompresjonsforhold og behandlingstid er begrenset.
- **Utgående forbindelser:** URL-er fra requests eller webhooks passerer allowlist, DNS-/IP-kontroll og egress-policy; redirects kontrolleres på nytt.
- **Credentials:** Tokens vises verken i URI eller logg; secrets roteres, begrenses til mål-audience og minimale scopes og bygges ikke inn i klientartefakter.
- **Trust Hops:** Terminerer en gateway [TLS](/kb/tls), må backend-hoppet autentiseres og autoriseres separat. En pålitelig Forwarded-header oppstår bare ved en kontrollert proxygrense.
- **Audit:** privilegerte endringer logger klient, principal, målobjekt, handling, før/etter-referanse, request-ID og resultat – uten secret eller komplett sensitiv payload.

OAuth 2.0 Security Best Current Practice fraråder blant annet Resource Owner Password Credentials Grant, krever eksakte redirect-URI-sammenligninger og foretrekker senderbundne eller kortlivede tokens der trusselmodellen krever det ([RFC 9700 – Best Current Practice for OAuth 2.0 Security](https://www.rfc-editor.org/rfc/rfc9700.html)). Mutual TLS og DPoP er to ulike metoder for senderbinding; begge endrer nøkkeldrift og feildiagnose og er ikke bare brytere i gatewayen ([RFC 8705 – OAuth 2.0 Mutual-TLS Client Authentication](https://www.rfc-editor.org/rfc/rfc8705.html), [RFC 9449 – OAuth 2.0 Demonstrating Proof of Possession](https://www.rfc-editor.org/rfc/rfc9449.html)).

Retries, paginering og events skaper flere tekniske prosesser for én faglig handling. Korrelasjon og audit må koble dem sammen igjen til et etterprøvbart forløp.

## Observerbarhet og bevisbare kall

Måledata viser volumet, logger viser enkeltbeslutninger og traces viser banen til en request over prosessgrenser. OpenTelemetry modellerer en trace som en kausal mengde av spans og definerer trace- og span-ID-er for korrelasjon ([OpenTelemetry – Traces](https://opentelemetry.io/docs/specs/otel/trace/)). For administrative formål bør et API-kall minst gjøre følgende fakta rekonstruerbare:

| Dimensjon | Driftsbevis |
|---|---|
| Kaller | klient-ID, workload eller bruker; autentiseringsmetode; effektive roller/scopes |
| Mål | host, tenant, API-/kontraktsversjon, metode eller operasjon, stabil ressursidentifikator |
| Kjøretid | starttid, total varighet, DNS-/connect-/TLS-tid der tilgjengelig, deadline, retrynummer |
| Resultat | transportstatus, faglig feilkode, svarstørrelse, rate-limit-/quota-tilstand |
| Korrelasjon | request-ID fra serveren, trace-ID, jobb-/event-ID og ved meldingsreferanse Message-ID |

ID-er videresendes over prosessgrenser, men overtas ikke blindt fra vilkårlige eksterne klienter som intern autoritet. Metrikketiketter unngår bruker-ID-er, komplette stier og andre verdier med høy kardinalitet. Payloads, Authorization-headere, cookies og webhook-signaturer hører som standard ikke hjemme i telemetri. En trace kan bevise banen, men erstatter ikke et manipulasjonsbeskyttet auditbevis for en privilegert endring.

## Versjonering, deprecation og sunset

Et versjonsnummer er ikke en livssyklus. Først skilles det mellom **kompatible utvidelser** og **breaking changes**. Nye valgfrie felter, flere enumverdier eller endret rekkefølge kan bryte klienter til tross for påstått bakoverkompatibilitet dersom de implementerer kontrakten for snevert. Consumer-tester og schema-diffing kontrollerer derfor ikke bare stier, men semantikk, rettigheter, feil og grenseverdier.

Versjoner kan stå i stien, hosten, headeren eller medietypen; det avgjørende er at ruting, dokumentasjon, telemetri og support entydig navngir samme variant. For utfasing standardiserer RFC 9745 HTTP-feltet `Deprecation`; RFC 8594 definerer `Sunset` som tidspunktet da en ressurs forventes å slutte å svare. Ingen av dem erstatter en migreringsveiledning, en alternativ lenke eller en påvist klientbestand ([RFC 9745 – The Deprecation HTTP Response Header Field](https://www.rfc-editor.org/rfc/rfc9745.html), [RFC 8594 – The Sunset HTTP Header Field](https://www.rfc-editor.org/rfc/rfc8594.html)).

En robust avviklingsprosess omfatter inventar over consumers, bruksmåledata per versjon og klient, annonserte datoer, parallell drift, testmiljø, tilbakefallsvei og en eksplisitt beslutning om avstenging. «Annonsert i wikien» er ikke bevis på at automatiseringer uten tilsyn er migrert.

## Driftsmodeller: lokalt, sky og control plane

Hvor et API befinner seg, avgjør ikke alene sikkerhet eller håndterbarhet. Et lokalt grensesnitt kan være direkte knyttet til privilegerte operativsystemkontoer, langlivede nøkler og lite segmenterte nettverk. En cloud control plane kan derimot tilby sterke workloadidentiteter og auditlogger, men avhenger fortsatt av internettbane, provider-IAM, tenantkonfigurasjon, quotaer og tjenestetilgjengelighet. Det avgjørende er det konkrete feil- og tillitsrommet.

| Modell | Typisk grense | Adminspørsmål |
|---|---|---|
| lokalt prosess-/host-API | Unix Socket, Named Pipe, Loopback eller administrasjons-LAN | Hvilken OS-identitet gjelder? Hvem eier socket/ACL? Er ekstern tilgang faktisk utelukket? |
| intern service-API | segment, service mesh, gateway eller lastbalanserer | Hvor termineres TLS og autorisasjon? Hvordan driftes serviceidentiteter og DNS? |
| SaaS-control plane | provider-endpoint og tenant-IAM | Hvilken region, quota, audit- og tokenbaner gjelder? Hvordan fungerer Break Glass? |
| dataplan pluss control plane | konfigurasjon styrer separate workere eller appliances | Når er en endring distribuert? Hvordan oppdages drift, rollback og deltilstander? |
| event-/webhook-integrasjon | produsent, broker eller offentlig callback | Hvem eier levering, retry, signatur, DLQ og reconciliation? |

Sikkerhetskopier sikrer ikke automatisk et eksternt API. For gjenoppstart inventariseres i stedet kontrakter, klientkonfigurasjon, secret-referanser, sertifikater, gatewayregler, idempotensstatus, åpne jobber og evnen til reconciliation. Recoverytester må også dekke utløpte tokens, endrede DNS-mål og tilbakestilte continuation tokens.

## Teknisk historie

Tidlige distribuerte grensesnitt var ofte tett bundet til Remote Procedure Call og språkspesifikke stubs. SOAP 1.2 definerte senere et XML-basert meldingsrammeverk med utvidbar behandlingsmodell og ble utbredt i bedriftsplattformer sammen med WSDL og WS-* ([W3C – SOAP Version 1.2 Part 1](https://www.w3.org/TR/soap12-part1/)). Roy Fieldings avhandling beskrev i 2000 REST som en arkitekturstil for distribuerte hypermediasystemer og avledet constraintene fra krav til weben – ikke som oppskriften «HTTP pluss JSON» ([Fielding – Architectural Styles and the Design of Network-based Software Architectures](https://ics.uci.edu/~fielding/pubs/dissertation/top.htm)).

HTTP utviklet seg parallelt fra persistente TCP-forbindelser i HTTP/1.1 via multipleksede strømmer i HTTP/2 til HTTP/3 over QUIC. Metodikken og statussemantikken er beskrevet transportuavhengig i RFC 9110; wireformatene ligger i RFC 9112, RFC 9113 og RFC 9114 ([RFC 9112 – HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112.html), [RFC 9113 – HTTP/2](https://www.rfc-editor.org/rfc/rfc9113.html), [RFC 9114 – HTTP/3](https://www.rfc-editor.org/rfc/rfc9114.html)).

JSON ble standardisert som et lettvekts utvekslingsformat; JSON Schema og OpenAPI kompletterte maskinlesbare struktur- og operasjonskontrakter. GraphQL beskriver en typet spørrings- og utførelsesmodell der klienter velger felter; gRPC kombinerer tjenesteorienterte RPC-definisjoner med Protocol Buffers og HTTP/2-basert framing ([JSON Schema Specification](https://json-schema.org/specification), [OpenAPI Specification](https://spec.openapis.org/oas/), [GraphQL Specification](https://spec.graphql.org/September2025/), [gRPC – What is gRPC?](https://grpc.io/docs/what-is-grpc/), [Protocol Buffers – Language Guide](https://protobuf.dev/programming-guides/proto3/)). Event- og streamingmodeller kompletterer request/response, men fjerner verken kontrakter eller spørsmål om levering og konsistens.

## Admin-sjekkliste på ett blikk

Etter kontrakt, kjøretid og drift samler følgende sjekkliste spørsmålene som bør være besvart før et API frigis. Den er ment som akseptansehjelp, ikke som erstatning for forklaringene ovenfor.

| Spørsmål | Bevis eller artefakt |
|---|---|
| Hvilken interaksjonsstil drifter jeg? | OpenAPI/GraphQL-skjema/Proto/AsyncAPI, konkret operasjon og transportprofil |
| Hvilket endpoint gjelder? | Scheme, FQDN, port, base path, region/tenant, DNS- og sertifikatbevis |
| Hvem kaller? | klient-/workload-ID, credentialtype, token-issuer, audience, scopes/roller, nøkkelbesittelse |
| Hva er kontrakten? | metoder, skjemaer, status- og feilkatalog, grenser, paginering, idempotens og livssyklus |
| Når kan det gjentas? | deadline, idempotent semantikk eller key, backoff, retrybudsjett og oppslagsoperasjon |
| Hvordan forhindrer jeg Lost Updates? | ETag/`If-Match`, faglig versjonsnummer eller transaksjonell operasjon |
| Hvordan oppdager jeg deltilstander? | jobb-/eventstatus, request-ID, trace, kø-/consumer-lag, reconciliation |
| Hvordan endres det? | staging/canary, contract- og consumer-tester, rollback, deprecation/sunset |
| Hvordan gjenopprettes det? | konfigurasjon, kontrakter, secret-/sertifikatreferanser, cursorer/jobber, replay- og reconciliation-test |
| Hva hører hjemme i runbooken? | kjente feilkoder, 401/403/404/409/412/429/5xx-baner, kontaktpersoner og eskaleringsdata |

## Kilder

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
