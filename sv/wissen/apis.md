---
title: "API:er: avtal, identiteter och distribuerade felgränser"
blatt: "apis"
description: "API:er för infrastruktur- och meddelandeadministratörer: REST, HTTP och JSON, RPC, GraphQL, gRPC och händelse-API:er, OpenAPI och scheman, OAuth och arbetsbelastningsidentiteter, gateways, timeouter, återförsök, idempotens, paginering, hastighetsbegränsningar, webhooks, observerbarhet, versionshantering och teknisk historia."
fakten:
  - label: Systemroll
    wert: maskinläsbart administrations- eller datagränssnitt mellan separata komponenter
    href: https://ics.uci.edu/~fielding/pubs/dissertation/net_app_arch.htm
  - label: REST
    wert: arkitekturstil med constraints; inte synonymt med HTTP plus JSON
    href: https://ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm
  - label: HTTP-modell
    wert: resurs/URI · metod · fält · representation · status
    href: https://www.rfc-editor.org/rfc/rfc9110.html
  - label: Avtal
    wert: beskriv operationer, scheman, fel, säkerhet och livscykel maskinläsbart
    href: https://spec.openapis.org/oas/
  - label: Dataformat
    wert: JSON · XML · Protocol Buffers · binär- och streamingformat
    href: https://www.rfc-editor.org/rfc/rfc8259.html
  - label: Interaktionsstilar
    wert: request/response · RPC · query · stream · event/webhook
    href: https://ics.uci.edu/~fielding/pubs/dissertation/net_arch_styles.htm
  - label: Identitet
    wert: API-nyckel, klientcertifikat eller token; autentiseringsuppgift och behörighet separerade
    href: https://www.rfc-editor.org/rfc/rfc9700.html
  - label: Återförsök
    wert: endast vid känd semantik; timeout innebär inte att inget hände på serversidan
    href: https://www.rfc-editor.org/rfc/rfc9110.html
  - label: Samtidighet
    wert: ETag/If-Match eller en verksamhetsspecifik versionsnummer förhindrar Lost Updates
    href: https://www.rfc-editor.org/rfc/rfc9110.html
  - label: Felformat
    wert: HTTP-status plus stabil maskinläsbar problemtyp och request-ID
    href: https://www.rfc-editor.org/rfc/rfc9457.html
  - label: Driftstatus
    wert: latens · felfrekvens · mättnad · kvot · tokenutgång · kö-/konsumentfördröjning
    href: https://opentelemetry.io/docs/specs/otel/trace/
  - label: Administratörsbevis
    wert: klient · identitet · scope · endpoint · avtalsversion · request-ID · resultat
    href: https://www.rfc-editor.org/rfc/rfc9110.html
werbung:
  - newsletter
ctaThemen:
  - cloudflare-workers
  - powershell
  - automatisierung
translationSourceHash: 4b29095b24ea1576608e147b1904a65a72c2b09b2e72f991e3e27a64dbcb3332
translationModel: gpt-5.6-terra
translatedAt: 2026-10-03T09:33:20.429Z
translationReview: automatic
---

# API:er: avtal, identiteter och distribuerade felgränser

Ett Application Programming Interface är en **avtals- och förtroendegräns** mellan oberoende drivna komponenter. Avtalet fastställer vilka operationer och data som finns; körtiden avgör transport, identitet, auktorisering, tidsbeteende och fel. För administratörer är denna åtskillnad central: en syntaktiskt giltig JSON-request kan hamna hos fel tenant, misslyckas med en giltig token för fel audience eller trots en klienttimeout ha genomförts framgångsrikt på serversidan.

API:er finns överallt i meddelandemiljöer: mellan administrationsklient och e-postplattform, gateway och katalog, övervakning och telemetriprogramvara, molnapplikation och webhook-mottagare. Ett GUI kan använda samma gränssnitt, men avbildar vanligtvis endast en del av dess tillstånd och fel. Robust drift börjar därför inte med det enskilda curl-anropet, utan med **gränssnittsstil, avtal, resursmodell, identitet, tillståndsövergångar och återställningssemantik**.

Förklaringen följer ett API-anrop från klienten till det verksamhetsmässiga svaret. Först behandlas transport och avtal, därefter identitet, felhantering och händelser; först sedan följer gatewaydrift, säkerhet och diagnostik.

## Klassificering som distribuerat system

Ett nätverks-API är inte bara applikationskod. Ett typiskt anrop passerar:

1. klientbibliotek, CLI eller automatiseringsprocess;
2. namnupplösning, routning och anslutningsuppbyggnad;
3. [TLS](/kb/tls), proxy eller service mesh;
4. lastbalanserare, API-gateway eller reverse proxy;
5. autentisering, tokenvalidering och auktorisering;
6. applikationstjänst, cache, kö och databas;
7. svarsväg, serialisering och klientutvärdering.

Roy Fieldings arkitekturarbete skiljer uttryckligen nätverksbaserade system från transparent lokal exekvering: nätverkskommunikation har egen latens, egna kostnader och felmoder. En arkitekturstil är ett koordinerat antal constraints som ger upphov till vissa egenskaper och avvägningar ([Fielding – Network-based Application Architectures](https://ics.uci.edu/~fielding/pubs/dissertation/net_app_arch.htm), [Fielding – Network-based Architectural Styles](https://ics.uci.edu/~fielding/pubs/dissertation/net_arch_styles.htm)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-apis.svg?v=20260813" title="Interaktive Infografik: API-Aufruf über DNS, TLS, Gateway, Identität und Dienst sowie Vertrag, Fehler- und Wiederholungssemantik, Events und Betriebsnachweis" loading="lazy">
  <a href="/images/kb-interaktiv-apis.svg?v=20260813">Öppna den interaktiva grafiken direkt</a>.
</iframe>

Den relevanta stacken dokumenteras för varje integration:

| Nivå | Exempel | Typiskt felområde |
|---|---|---|
| Identifiering | URI, Service Discovery, DNS | fel host, region, tenant eller API-sökväg |
| Transport | TCP/TLS, HTTP/1.1, HTTP/2, HTTP/3 | timeout, proxy, certifikat, ALPN, connection pool |
| Interaktion | REST, RPC, GraphQL, gRPC, webhook/event | fel semantik, olämpligt återförsök, avbruten streaming |
| Representation | JSON, XML, Protobuf, Multipart, binärdata | schema-, kodnings-, storleks- eller kompatibilitetsfel |
| Avtal | OpenAPI, JSON Schema, Protobuf IDL, GraphQL SDL, AsyncAPI | breaking change, drift mellan dokumentation och körtid |
| Identitet | API-nyckel, OAuth-token, mTLS, signerad request | utgång, scope, audience, nyckelrotation, clock skew |
| Policy | Gateway, WAF, RBAC/ABAC, kvot | 401/403/429, headernormalisering, fel principal |
| Tillstånd | Tjänst, cache, kö, databas | partiell exekvering, replikeringsfördröjning, eventual consistency |
| Bevisunderlag | Request-ID, trace, auditlogg, mätvärden | saknad korrelation, sampling, dataskydd |

## REST är en arkitekturstil, inte ett dataformat

REST betecknar de constraints för distribuerade hypermediasystem som Fielding beskrev: klient/server, statelessness, cache, enhetligt gränssnitt, lagerindelning och valfritt Code-on-Demand. Det enhetliga gränssnittet omfattar resursidentifiering, manipulation genom representationer, självbeskrivande meddelanden och hypermedia som tillståndsmaskin. Standardiseringen av gränssnittet förbättrar synlighet och oberoende utveckling, men kan vara mindre effektiv än specialiserade protokoll ([Fielding – Representational State Transfer](https://ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)).

Ett HTTP-API med JSON och sökvägar som `/v1/getUser` är därför inte automatiskt REST. Det kan helt enkelt vara RPC över HTTP. Det är inte i sig dåligt; problem uppstår när operatörer förväntar sig egenskaper som den faktiska stilen inte erbjuder. En klient kan exempelvis inte säkert upprepa en POST bara för att endpointen kallas ”REST”.

### URI, resurs och representation

En URI identifierar en resurs; den garanterar varken nåbarhet eller en viss operation. RFC 3986 skiljer uttryckligen identifiering från interaktion. Scheme, authority, path, query och fragment har definierad syntax, medan det konkreta API:et fastställer resurssemantiken ([RFC 3986 – URI Generic Syntax](https://www.rfc-editor.org/rfc/rfc3986.html)).

En resurs är inte dess JSON-fil. Samma resurs kan representeras som JSON, XML eller ett annat format beroende på `Accept`-headern. `Content-Type` beskriver den skickade body, `Accept` det föredragna svaret. Status, fält och body utgör tillsammans meddelandet; att endast logga JSON-body gör att viktig diagnostisk information försvinner.

## HTTP-semantik: metod före sökvägsnamn

RFC 9110 skiljer resursidentifiering från requestsemantik. Metoden definierar den avsedda operationen; URI-sökvägen ensam gör det inte. ”Safe” innebär att klienten inte avser någon tillståndsändring. ”Idempotent” innebär att flera identiska requests har samma avsedda effekt som en request; bieffekter som loggning får ändå uppstå flera gånger ([RFC 9110 – HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)).

| Metod | Safe | Idempotent | Typisk API-semantik | Återförsök utan ytterligare kunskap |
|---|---:|---:|---|---|
| GET | ja | ja | Läs representation | i princip möjligt, men beakta belastning/kvot |
| HEAD | ja | ja | Metadata utan body | i princip möjligt |
| OPTIONS | ja | ja | Funktioner/kommunikationsalternativ | i princip möjligt |
| PUT | nej | ja | Ersätt tillstånd under känd URI | möjligt om avtalet verkligen följer PUT-semantik |
| DELETE | nej | ja | Ta bort koppling | effekten kan upprepas; svarsstatus kan ändras |
| POST | nej | nej | Bearbeta, åtgärd eller ny resurs | upprepa inte blint |
| PATCH | nej | inte generellt | Partiell ändring | endast med dokumenterad patch- och idempotenssemantik |

Idempotens beskriver den **avsedda servereffekten**, inte transportsvaret. En PUT kan vara slutförd på servern medan svaret går förlorat. En ny PUT är då semantiskt försvarbar; en ny POST kan skapa ett andra objekt eller ett andra meddelande. För icke-idempotenta operationer behövs en operationsidentifierare som klienten skapar, en leverantörsspecifik Idempotency-Key eller en efterföljande statusuppslagning.

### Statuskoder är kategorier, inte fullständig diagnostik

- `2xx`: requesten har bearbetats på det sätt som statusen definierar; inte varje `202 Accepted` är redan verksamhetsmässigt avslutad.
- `3xx`: ytterligare åtgärd eller annan representation; redirects kan ändra metod och credentialflöde.
- `400`: requesten är felaktig ur serverns perspektiv.
- `401`: autentiseringsuppgifter saknas eller är ogiltiga; svaret använder i princip `WWW-Authenticate`.
- `403`: servern förstår requesten men nekar den.
- `404`: resursen hittades inte eller döljs avsiktligt; inget säkert bevis för att den inte finns.
- `409`: konflikt med aktuellt tillstånd.
- `412`: precondition som `If-Match` är inte uppfyllt.
- `429`: för många requests under ett tidsfönster; `Retry-After` kan ange en väntetid.
- `5xx`: servern kunde inte uppfylla en i grunden giltig request; inte automatiskt möjlig att försöka igen.

RFC 6585 definierar `429 Too Many Requests`, men varken kvotens scope eller räknaren. Dessa kan gälla per credential, användare, tenant, resurs, region eller kluster ([RFC 6585 – Additional HTTP Status Codes](https://www.rfc-editor.org/rfc/rfc6585.html)). Klienten sparar därför status, relevanta svarsfält, request-ID och förkortad felbody.

## Transportvarianter och anslutningskostnader

HTTP-semantik är skild från den konkreta wire-versionen. HTTP/1.1 använder textregler för meddelandeinramning, HTTP/2 multiplexade strömmar och binär inramning, HTTP/3 bygger HTTP på QUIC. En API-gateway kan ta emot HTTP/2 på klientsidan och tala HTTP/1.1 med backend; protokollet hos klienten bevisar inte hela backendvägen ([RFC 9112 – HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112.html), [RFC 9113 – HTTP/2](https://www.rfc-editor.org/rfc/rfc9113.html), [RFC 9114 – HTTP/3](https://www.rfc-editor.org/rfc/rfc9114.html)).

Administratörer observerar inte bara requestlatens, utan även DNS- och anslutningstid, TLS-handshake, connection reuse, HTTP-version/ALPN, proxy-/gatewaytid, Time to First Byte, bodyöverföring, antal återförsök och total wall-clock-tid.

### Kontrollera namn-, TCP- och TLS-väg

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) och [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) kontrollerar [DNS](/kb/dns). [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) och [`nc`](https://man.openbsd.org/nc) kontrollerar TCP. [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) visar TLS-handshake, certifikatskedja och ALPN; inget av dessa test bevisar en lyckad API-auktorisering.

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
När metod, transport och representation är förstådda följer den första distribuerade egenheten: En framgångsrik ändring behöver inte genast vara synlig i varje läsväg.

## Cachning och read-after-write

HTTP-cachning lagrar representationer baserat på cachenycklar och direktiv. `Cache-Control`, `Vary`, validators och autentiseringsregler bestämmer om och hur återanvändning sker. En `200` kan komma från en cache; en GET direkt efter PUT kan beroende på arkitekturen fortfarande se gammalt tillstånd ([RFC 9111 – HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111.html)).

Administratörsfrågor:

- Finns en webbläsar-, proxy-, CDN-, gateway- eller applikationscache i vägen?
- Vilka headers bildar cachenyckeln, särskilt Authorization och tenant?
- Är representationen privat, offentlig eller inte cachebar alls?
- Hur länge kan negativa svar cachelagras?
- Finns read-your-writes eller eventual consistency?
- Vilken region/replika läser efterföljande GET från?
- Är ETag en innehållsversion eller bara en cachevalidator?

Att rensa cache är ingen universell reparation. Det kan skapa lasttoppar och dölja den egentliga inkonsekvensen.

## Rate limits, kvoter och mättnad

Rate limit, kvot och samtidighetsgräns är olika kontroller:

- **Rate:** requests eller kostnadspoäng per tidsfönster.
- **Kvot:** total förbrukning per dag, månad eller subscription.
- **Samtidighet:** samtidigt pågående requests/strömmar.
- **Payloadgräns:** storlek på body, objekt, batch eller svar.
- **Komplexitetsgräns:** querydjup, GraphQL Cost eller expanderade relationer.

`429` kan leverera `Retry-After`, men leverantörsspecifika rate-limit-headers är inte enhetliga. Klienten behandlar dokumenterade fält som en del av det konkreta avtalet, inte som en universell standard. Den begränsar frågehastigheten lokalt, fördelar budget mellan workloads och sparar scope, återstående budget och återställningstid.

Throttling är en skyddssignal, inte ett normalt genomströmningsläge. Varaktiga 429-vågor indikerar olämplig paginering, avsaknad av cachning, för hög parallellitet eller otillräcklig kapacitet.

## Felbody och korrelation

En HTTP-status är för grov för verksamhetsmässig automatisering. RFC 9457 definierar Problem Details med stabil `type`-URI, `title`, `status`, `detail` och `instance` samt utökningsfält. Problemtypen är den maskinläsbara identiteten; fritt formulerad text är inte avsedd för parsers ([RFC 9457 – Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457.html)).

Ett bra felavtal ger en stabil fel-/problemtyp, HTTP-/RPC-status, säker detaljuppgift, berörda fältsökvägar, request-/correlation-ID, möjlighet till återförsök och en dokumentationslänk. Klienten loggar inga kompletta tokens, Authorization-headers eller konfidentiella payloads.

### Registrera felsvar med headers

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

## RPC, GraphQL och gRPC

Inte alla API:er passar resursstilen.

| Stil | Avtalets centrum | Styrka | Driftsgräns |
|---|---|---|---|
| REST/HTTP | Resurs, representation, HTTP-semantik | Webbmellanled, cachning, brett verktygsstöd | inkonsekventa detaljkonventioner |
| RPC | Tjänst och operation | direkt avbildning av verksamhetsåtgärder | retry-/idempotens per metod explicit |
| GraphQL | Typat schema och klientquery | flexibelt urval av relaterade data | querykostnader, N+1, ofta HTTP 200 trots fältfel |
| gRPC | Protobuf Service/Message | Codegen, HTTP/2, unary och streaming | binär inramning, proxys, gRPC-status/trailers |
| Event API | Channel, Message, händelsetyp | frikoppling och asynkron bearbetning | ordning, deduplicering, replay, consumer lag |

GraphQL-specifikationen definierar språk, typsystem, validering och exekvering; transport, autentisering, rate limit och driftmässiga querykostnader är ytterligare avtal ([GraphQL Specification](https://spec.graphql.org/September2025/)). Fältfel kan förekomma tillsammans med partiella data; enbart HTTP-status beskriver inte resultatet.

gRPC avbildar channels, RPC:er och length-prefixed messages på HTTP/2-strömmar. gRPC-status överförs i trailers och måste särskiljas från HTTP-status. Calls är inte automatiskt idempotenta; deadline, cancellation och retry policy förstås per tjänst ([gRPC – What is gRPC?](https://grpc.io/docs/what-is-grpc/), [gRPC over HTTP/2 protocol](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md)). Protocol Buffers ger en modell för gränssnitt och serialisering; fältnummer är kompatibilitetsankare och får inte återanvändas för en annan betydelse efter att de har tagits bort ([Protocol Buffers – Language Guide](https://protobuf.dev/programming-guides/proto3/)).

[`grpcurl`](https://github.com/fullstorydev/grpcurl) kan använda server reflection eller lokala deskriptorer för att undersöka gRPC-tjänster. Reflection är själv en exponerad yta och aktiveras inte offentligt utan kontroll.

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

## Representationer: JSON är syntax, schema är avtal

RFC 8259 definierar JSON som ett utbytesformat med objekt, arrayer, tal, strängar, booleaner och null. JSON definierar inte vilket fält som är ett stabilt ID, om ett saknat fält och `null` betyder samma sak, vilken tidszon en timestamp har eller om okända egenskaper tolereras ([RFC 8259 – JSON](https://www.rfc-editor.org/rfc/rfc8259.html)).

Denna semantik hör hemma i ett schema och avtalsdokumentationen:

- fältnamn, typ, format och enhet;
- required, optional, nullable och default;
- read-only/write-only och servergenererat;
- enumvärden och beteende för okända värden;
- tidsformat, tidszon och precision;
- stabilt ID jämfört med visningsnamn;
- referens-, inbäddnings- och raderingssemantik;
- kompatibilitetsregel för nya eller borttagna fält.

JSON Schema definierar vokabulär för validering av JSON-instanser. Ett schema kan kontrollera struktur, men ersätter inte verksamhetsmässiga invariants eller auktorisering ([JSON Schema – Specification](https://json-schema.org/specification)). OpenAPI Specification kan maskinläsbart beskriva HTTP-operationer, parametrar, request/response-scheman och Security Schemes; den bevisar inte att den körande implementationen motsvarar dokumentet ([OpenAPI Specification](https://spec.openapis.org/oas/)).

### Inspektera svar och fält utan GUI

[`Invoke-RestMethod`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod) deserialiserar strukturerade svar; [`Invoke-WebRequest`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest) returnerar fler HTTP-detaljer. [`curl`](https://curl.se/docs/manpage.html) visar request/response och tidsinformation, [`jq`](https://jqlang.org/manual/) filtrerar JSON.

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

## Avtal och contract drift

Ett fullständigt API-avtal omfattar mer än happy-path-scheman:

| Avtalsområde | Måste fastställas |
|---|---|
| Discovery | Base URL, region, tenant, service-/metadata-endpoint |
| Operation | Metod/RPC/event, parameterbinding, bieffekt |
| Data | Schema, ID:n, ordning, null-/defaultsemantik, storleksgränser |
| Säkerhet | Authflow, credentialtyp, audience, scope/roll, tokenlivslängd |
| Fel | Status/kod/problemtyp, möjlighet till återförsök, request-ID |
| Konsistens | Read-after-write, replikeringsfördröjning, cache och ETag |
| Mängd | Paginering, filter, sortering, snapshot-/cursorsemantik |
| Tid | Klient-/gateway-/serverdeadline, Retry-After, clock skew |
| Livscykel | Avtalsversion, deprecation, sunset och migreringsväg |
| Drift | Kvot, SLO, underhåll, statussida, supportkorrelation |

Contract drift uppstår när dokument, SDK och produktiv implementation glider isär. Därför lagras den publicerade specifikationen som en versionshanterad artefakt, valideras i CI och kontrolleras mot en verklig testmiljö. Genererade klienter minskar skrivarbete, men sprider också schemafel och breaking changes till många konsumenter.

Consumer-driven tester kan synliggöra en klients antaganden. De ersätter inte providersemantik: en mock kan leverera en `200`, trots att produktionen efter en gatewayuppdatering returnerar en annan header, annan paginering eller ett nytt enum.

Hittills var anropet tekniskt giltigt. Om det också får utföras av rätt identitet på rätt objekt avgörs av säkerhetskontrollen.

## Autentisering är inte auktorisering

En API-nyckel eller token besvarar först **vilken klient eller principal** som talar. Auktorisering avgör därefter vilken åtgärd som är tillåten på vilken resurs i vilket scope. En giltig token kan därför korrekt avvisas med `403`.

| Metod | Styrka och användning | Driftrisk |
|---|---|---|
| API-nyckel | enkel klientidentifiering eller kvotankare | ofta lång giltighet, begränsat scope, lätt att kopiera |
| Basic Auth | användarnamn/lösenord över TLS | lösenordslivscykel, MFA-/delegeringsgränser |
| mTLS | ömsesidig TLS, klientcertifikat | PKI, rotation, proxyterminering, mappning till principal |
| OAuth Access Token | delegerad eller workload-behörighet med scope/audience | tokenhämtning, utgång, consent, replay |
| signerad request | integritet för utvalda meddelandedelar | canonicalization, clock skew, nonce/replay store |
| nätverksidentitet | privata nät, service mesh, workloadcertifikat | får inte tyst ersätta verksamhetsmässig RBAC |

OAuth 2.0 definierar roller och grantmekanismer för utfärdande av Access Tokens; Bearer Tokens kan användas av vem som helst som innehar dem ([RFC 6749 – OAuth 2.0](https://www.rfc-editor.org/rfc/rfc6749.html), [RFC 6750 – Bearer Token Usage](https://www.rfc-editor.org/rfc/rfc6750.html)). Security BCP RFC 9700 kräver minsta privilegium, Audience Restriction och skydd av redirectflöden; den förbjuder Resource Owner Password Credentials Grant. För replay-skydd nämner den sändarbundna tokens genom mTLS eller DPoP ([RFC 9700 – OAuth 2.0 Security BCP](https://www.rfc-editor.org/rfc/rfc9700.html), [RFC 8705 – OAuth mTLS](https://www.rfc-editor.org/rfc/rfc8705.html), [RFC 9449 – DPoP](https://www.rfc-editor.org/rfc/rfc9449.html)).

OpenID Connect lägger ett identitetslager ovanpå OAuth; en ID Token är avsedd för klienten och inte automatiskt en Access Token för ett API ([OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html)). En JWT är bara ett kompakt claimformat. Enbart signaturkontroll räcker inte: algoritm, issuer, audience, tidsclaims, nyckelval och applikationsspecifika claims måste valideras ([RFC 7519 – JSON Web Token](https://www.rfc-editor.org/rfc/rfc7519.html), [RFC 8725 – JWT Best Current Practices](https://www.rfc-editor.org/rfc/rfc8725.html)).

### Hämta workload-token och anropa API

Client Credentials-flödet är endast lämpligt när applikationen agerar i eget namn och kan hålla sitt credential säkert. Secret, certifikat eller federerad workloadidentitet, tokenendpoint, audience/resource och scope hör hemma i den konkreta plattformsdokumentationen.

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

Secrets visas varken på kommandoraden eller i transcript eller debuglogg. Exemplet visar protokollflödet, inte en lämplig secrettransport för produktionsprocesser.

## API-gateway och förtroendegränser

En gateway kan terminera TLS, validera tokens, samla routning, kvoter, schemafilter, WAF-regler och observerbarhet. Den är därmed en kontrollpunkt och ett felområde. Backendtjänsten får inte tyst förutsätta att varje request kom via just denna gatewayväg.

- **Klient → gateway:** offentlig hostidentitet, TLS, DDoS/kvot, klientcredential.
- **Gateway → tjänst:** egen mTLS- eller workloadidentitet; inget blint förtroende baserat på käll-IP.
- **Identity Provider → validator:** issuer metadata, JWKS, nyckelrotation, cache och klocka.
- **Tjänst → datalagring:** verksamhetsmässig auktorisering och tenantgräns.
- **Webhookprovider → receiver:** signatur, tidsfönster, event-ID och replaykontroll.

RFC 9700 varnar uttryckligen för okontrollerade inkommande forwarding-headers vid reverse proxys. Proxyn måste rensa säkerhetsrelevanta fält; den interna länken måste skyddas mot avlyssning, injection och replay ([RFC 9700 – OAuth 2.0 Security BCP](https://www.rfc-editor.org/rfc/rfc9700.html)).

Gatewaystatus `200` bevisar inte att en efterföljande kö eller replikering är frisk. Omvänt kan en backend vara frisk medan DNS, certifikat, tokenvalidator eller kvot blockerar varje klient.

Efter gateway och behörighetskontroll återstår den svåraste driftfrågan: Vad hände när klienten inte får ett svar i tid? En timeout bevisar inte att servern inte har ändrat något.

## Timeouter, deadlines och partiell exekvering

”Timeout” är inget serverresultat. Klienten vet endast att inget användbart svar har kommit inom dess tidsfrist. Requesten kan ha misslyckats före anslutningsuppbyggnaden, ha kastats av gatewayen, fortfarande vara aktiv i tjänsten eller redan ha committats medan endast svaret försvann.

Varje lager kan ha en egen tidsfrist: DNS, connect, TLS, klientens totaltid, proxy, gateway, upstream, databas och kö. Den yttre deadlinen måste samordnas med de inre tidsfristerna; annars avbryter klienten efter 30 sekunder medan servern arbetar vidare i 60 sekunder och ett återförsök parallellt startar samma åtgärd.

En tjänst propagerar om möjligt en återstående deadline i stället för att starta om hela tiden för varje hopp. Cancellation är best effort: den bevisar inte att en redan committad bieffekt har återställts.

## Återförsök, backoff och idempotens

Automatiskt återförsök är endast tillåtet om **felklassen och operationen** tillåter det. En robust klient klargör:

1. Upprättades över huvud taget en anslutning?
2. Finns en status eller ett protokollfel?
3. Är operationen safe/idempotent eller skyddad med deduplicering?
4. Ger servern `Retry-After` eller en produktspecifik backoffuppgift?
5. Återstår tillräckligt med end-to-end-deadline?
6. Förvärrar ett återförsök överbelastningen?

Exponentiell backoff med jitter förhindrar synkrona återförsöksvågor. Antalet försök är begränsat och en del av den totala latensen. `401` eller `403` repareras inte genom tätare upprepning; `429` kräver respekt för kvoter; en `500` efter en POST kan trots felbody lämna en partiell bieffekt.

För en verksamhetsoperation sparar klienten ett stabilt operations-ID. Servern behåller resultatet eller dedupliceringsstatusen minst lika länge som maximalt återförsöksfönster. Om en sådan garanti saknas läser klienten före återförsöket utifrån ett stabilt objekt-ID eller sökvillkor.

## Optimistisk samtidighet

Read-modify-write utan versionsvillkor skapar Lost Updates:

```text
Client A liest Version 7     Client B liest Version 7
Client A schreibt Änderung  → Version 8
Client B schreibt alten Stand plus Änderung → A geht verloren
```

HTTP stöder villkorliga requests med validators som `ETag`. Klienten läser ETag och skickar vid ändringen `If-Match`; om representationen har ändrats svarar servern med `412 Precondition Failed` i stället för att skriva över någon annans tillstånd. Det konkreta API:et måste dokumentera om ETag är tillräckligt stark för denna semantik ([RFC 9110 – HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)).

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

## Paginering, filter och konsekventa mängder

En endpoint som i dag levererar 50 objekt kan i morgon leverera 50 000. Paginering är en del av avtalet:

- **Offset/Page:** enkelt, men infogningar och borttagningar kan skapa dubbletter eller luckor.
- **Cursor/Continuation Token:** kodar framsteg på serversidan; token är opak och tolkas inte.
- **Keyset:** sorterar efter ett stabilt, unikt fortsättnings-ID.
- **Snapshot:** behåller en konsekvent vy över flera sidor, men kräver servertillstånd eller versionsankare.

Klienten följer dokumenterad Next-Link eller cursor och konstruerar den inte utifrån antaganden. RFC 8288 definierar typade länkar, men ingen universell paginering; konkret relation och bodyform är fortfarande API-avtal ([RFC 8288 – Web Linking](https://www.rfc-editor.org/rfc/rfc8288.html)).

Filter och sortering måste vara stabila över sidor. En sortering enbart efter en icke-unik tidsstämpel är otillräcklig; en tie-breaker som oföränderligt ID måste ingå. För delta-/change-API:er dokumenteras cursor, utgång och återsynkroniseringsväg.

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


## Webhooks, händelser och asynkrona API:er

En synkron HTTP-framgång och en verksamhetsmässigt avslutad bearbetning är olika tillstånd. En `202 Accepted` bekräftar enligt RFC 9110 endast att servern har accepterat bearbetningen; uppdraget kan fortfarande misslyckas senare. En webhook-mottagare bekräftar omvänt ofta bara att en händelse har accepterats och sparats. Den som likställer `2xx` med ”affärsprocess avslutad” förlorar just de mellanliggande tillstånd som är relevanta vid köer, återförsök och delstörningar ([RFC 9110 – HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)).

En driftmässigt användbar händelse innehåller minst ett stabilt event-ID, händelsetyp och schemaversion, skapandetid, producent, resurs-ID samt – om ordningen är verksamhetsmässigt viktig – en resurs- eller sekvensversion. CloudEvents standardiserar ett leverantörsneutralt händelsekuvert för detta; AsyncAPI beskriver meddelandekanaler och operationer maskinläsbart, liknande OpenAPI:s roll för request/response-API:er ([CloudEvents Specification](https://github.com/cloudevents/spec), [AsyncAPI Specification](https://www.asyncapi.com/docs/reference/specification/latest)).

Webhooks drivs som en extern klient som upprepar sig:

- Avsändaren signerar den **oförändrade request-body** tillsammans med tids- eller nonce-metadata; mottagaren validerar signatur, accepterat tidsfönster och målkontext före parsning. Standardiserade HTTP Message Signatures kan kryptografiskt binda komponenter och härledda fält ([RFC 9421 – HTTP Message Signatures](https://www.rfc-editor.org/rfc/rfc9421.html)).
- Mottagaren deduplicerar med event-ID och sparar acceptansen före den positiva bekräftelsen. Bearbetning utformas idempotent.
- Retries har begränsad löptid, backoff och en Dead-Letter- eller karantänväg. En replay protokollförs och skapar ingen ny verksamhetsmässig identitet.
- En periodisk **reconciliation-körning** jämför källsystem och lokalt tillstånd. Webhooks är acceleratorer, inte nödvändigtvis den enda sanningskällan.

## API-säkerhet: objekt, funktion och dataflöde

En godkänd tokenkontroll besvarar bara vem eller vilken workload som talar och för vilken audience credentialet är avsett. Applikationen måste dessutom för varje objekt och varje operation avgöra om denna identitet får läsa eller ändra just denna tenant, användare, nyckel eller meddelandebestånd. OWASP API Security Top 10 lyfter därför bland annat Broken Object Level Authorization, Broken Authentication, obegränsad resursförbrukning, SSRF och bristfälligt API-inventarium som egna riskklasser ([OWASP API Security Top 10](https://owasp.org/API-Security/editions/2023/en/0x11-t10/)).

För infrastruktur-API:er följer konkreta kontroller av detta:

- **Objektkoppling:** tenant- och objektankare kommer från servervaliderad kontext, inte enbart från ett fritt valbart sökvägs- eller bodyfält.
- **Inmatningsgränser:** Content-Type, schema, fältlängder, nästling, total storlek, komprimeringsförhållande och bearbetningstid är begränsade.
- **Utgående anslutningar:** URL:er från requests eller webhooks passerar allowlist, DNS-/IP-kontroll och egress-policy; redirects kontrolleras på nytt.
- **Credentials:** tokens förekommer varken i URI eller logg; secrets roteras, begränsas till målaudience och minsta scopes och bäddas inte in i klientartefakter.
- **Trust hops:** Om en gateway terminerar [TLS](/kb/tls), måste backendhoppet autentiseras och auktoriseras separat. En betrodd Forwarded-header uppstår endast vid en kontrollerad proxygräns.
- **Audit:** privilegierade ändringar loggar klient, principal, målobjekt, åtgärd, före-/efterreferens, request-ID och resultat – utan secret eller komplett känslig payload.

OAuth 2.0 Security Best Current Practice avråder bland annat från Resource Owner Password Credentials Grant, kräver exakta jämförelser av redirect-URI:er och föredrar sändarbundna respektive kortlivade tokens där hotmodellen kräver det ([RFC 9700 – Best Current Practice for OAuth 2.0 Security](https://www.rfc-editor.org/rfc/rfc9700.html)). Mutual TLS och DPoP är två olika metoder för sändarbindning; båda ändrar nyckeldrift och feldiagnostik och är inte bara omkopplare på gatewayen ([RFC 8705 – OAuth 2.0 Mutual-TLS Client Authentication](https://www.rfc-editor.org/rfc/rfc8705.html), [RFC 9449 – OAuth 2.0 Demonstrating Proof of Possession](https://www.rfc-editor.org/rfc/rfc9449.html)).

Retries, paginering och händelser skapar flera tekniska processer för en verksamhetsåtgärd. Korrelation och audit måste åter knyta samman dem till ett spårbart förlopp.

## Observerbarhet och bevisbara anrop

Mätvärden visar volymen, loggar enskilda beslut och traces vägen för en request över processgränser. OpenTelemetry modellerar en trace som en kausal mängd spans och definierar trace- och span-ID:n för korrelation ([OpenTelemetry – Traces](https://opentelemetry.io/docs/specs/otel/trace/)). För administratörsändamål bör ett API-anrop åtminstone göra följande fakta rekonstruerbara:

| Dimension | Driftbevis |
|---|---|
| Anropare | Klient-ID, workload eller användare; autentiseringsmetod; effektiva roller/scopes |
| Mål | Host, tenant, API-/avtalsversion, metod respektive operation, stabil resursidentifierare |
| Körtid | Starttid, total varaktighet, DNS-/connect-/TLS-tid där tillgänglig, deadline, retry-nummer |
| Resultat | Transportstatus, verksamhetsmässig felkod, svarsstorlek, rate-limit-/kvotstatus |
| Korrelation | Serverns request-ID, trace-ID, job-/event-ID och Message-ID vid meddelandekoppling |

ID:n förs vidare över processgränser, men tas inte blint över från godtyckliga externa klienter som intern auktoritet. Mätvärdesetiketter undviker användar-ID:n, fullständiga sökvägar och andra högkardinalitetsvärden. Payloads, Authorization-headers, cookies och webhook-signaturer hör som standard inte hemma i telemetri. En trace kan bevisa vägen, men ersätter inte ett manipulationsskyddat auditbevis för en privilegierad ändring.

## Versionshantering, deprecation och sunset

Ett versionsnummer är ingen livscykel. Först skiljs mellan **kompatibla utökningar** och **breaking changes**. Nya valfria fält, ytterligare enumvärden eller ändrad ordning kan bryta klienter trots påstådd bakåtkompatibilitet om dessa implementerar avtalet för snävt. Consumer-tester och schema-diffing granskar därför inte bara sökvägar utan även semantik, behörigheter, fel och gränsvärden.

Versioner kan stå i sökvägen, hosten, headern eller medietypen; avgörande är att routning, dokumentation, telemetri och support entydigt benämner samma variant. För utfasning standardiserar RFC 9745 HTTP-fältet `Deprecation`; RFC 8594 definierar `Sunset` som tidpunkten från vilken en resurs sannolikt inte längre svarar. Båda ersätter varken en migreringsanvisning, en alternativ länk eller ett verifierat klientbestånd ([RFC 9745 – The Deprecation HTTP Response Header Field](https://www.rfc-editor.org/rfc/rfc9745.html), [RFC 8594 – The Sunset HTTP Header Field](https://www.rfc-editor.org/rfc/rfc8594.html)).

En robust avvecklingsprocess omfattar inventarium över konsumenter, användningsmätvärden per version och klient, aviserade datum, parallell drift, testmiljö, återfallsväg och ett uttryckligt beslut om avstängning. ”Annonserat i wikin” är inget bevis på att oövervakade automatiseringar har migrerats.

## Driftmodeller: lokalt, moln och control plane

Var ett API finns avgör inte ensamt säkerheten eller hanterbarheten. Ett lokalt gränssnitt kan vara direkt bundet till privilegierade operativsystemkonton, långlivade nycklar och svagt segmenterade nät. En cloud control plane kan däremot erbjuda starka workloadidentiteter och auditloggar, men förblir beroende av internetväg, provider-IAM, tenantkonfiguration, kvoter och tjänstetillgänglighet. Avgörande är det konkreta fel- och förtroendeutrymmet.

| Modell | Typisk gräns | Administratörsfrågor |
|---|---|---|
| lokal process-/host-API | Unix Socket, Named Pipe, loopback eller management-LAN | Vilken OS-identitet gäller? Vem äger socket/ACL? Är fjärråtkomst verkligen utesluten? |
| intern service-API | Segment, service mesh, gateway eller load balancer | Var slutar TLS och auktorisering? Hur drivs serviceidentiteter och DNS? |
| SaaS-control plane | Provider-endpoint och tenant-IAM | Vilken region, kvot, audit- och tokenvägar gäller? Hur fungerar Break Glass? |
| dataplan plus control plane | Konfiguration styr separata workers eller appliances | När har en ändring distribuerats? Hur identifieras drift, rollback och partiella tillstånd? |
| event-/webhook-integration | Producent, broker eller offentlig callback | Vem äger leverans, retry, signatur, DLQ och reconciliation? |

Backuper skyddar inte automatiskt ett externt API. För återstart inventeras i stället avtal, klientkonfiguration, secretreferenser, certifikat, gatewayregler, idempotenstillstånd, öppna jobb och förmågan till reconciliation. Recoverytester måste även omfatta utgångna tokens, ändrade DNS-mål och återställda Continuation Tokens.

## Teknisk historia

Tidiga distribuerade gränssnitt var ofta tätt bundna till Remote Procedure Call och språkspecifika stubs. SOAP 1.2 definierade senare ett XML-baserat meddelanderamverk med utökningsbar bearbetningsmodell och spreds tillsammans med WSDL och WS-* i företagsplattformar ([W3C – SOAP Version 1.2 Part 1](https://www.w3.org/TR/soap12-part1/)). Roy Fieldings avhandling beskrev år 2000 REST som en arkitekturstil för distribuerade hypermediasystem och härledde constraints från kraven på webben – inte som receptet ”HTTP plus JSON” ([Fielding – Architectural Styles and the Design of Network-based Software Architectures](https://ics.uci.edu/~fielding/pubs/dissertation/top.htm)).

HTTP utvecklades parallellt från persistenta TCP-anslutningar i HTTP/1.1 via multiplexade strömmar i HTTP/2 till HTTP/3 över QUIC. Metodik och statussemantik beskrivs transportoberoende i RFC 9110; wireformaten finns i RFC 9112, RFC 9113 och RFC 9114 ([RFC 9112 – HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112.html), [RFC 9113 – HTTP/2](https://www.rfc-editor.org/rfc/rfc9113.html), [RFC 9114 – HTTP/3](https://www.rfc-editor.org/rfc/rfc9114.html)).

JSON standardiserades som ett lättviktigt utbytesformat; JSON Schema och OpenAPI kompletterade maskinläsbara struktur- och operationsavtal. GraphQL beskriver en typad query- och exekveringsmodell där klienter väljer fält; gRPC kombinerar tjänsteorienterade RPC-definitioner med Protocol Buffers och HTTP/2-baserad inramning ([JSON Schema Specification](https://json-schema.org/specification), [OpenAPI Specification](https://spec.openapis.org/oas/), [GraphQL Specification](https://spec.graphql.org/September2025/), [gRPC – What is gRPC?](https://grpc.io/docs/what-is-grpc/), [Protocol Buffers – Language Guide](https://protobuf.dev/programming-guides/proto3/)). Event- och streamingmodeller kompletterar request/response, men eliminerar varken avtal eller frågor om leverans och konsistens.

## Administratörschecklista i korthet

Efter avtal, körtid och drift sammanfattar följande checklista de frågor som bör besvaras innan ett API godkänns. Den är avsedd som stöd vid acceptans, inte som ersättning för föregående förklaringar.

| Fråga | Bevis eller artefakt |
|---|---|
| Vilken interaktionsstil driver jag? | OpenAPI/GraphQL-schema/Proto/AsyncAPI, konkret operation och transportprofil |
| Vilken endpoint gäller? | Scheme, FQDN, port, base path, region/tenant, DNS- och certifikatbevis |
| Vem anropar? | Klient-/workload-ID, credentialtyp, token-issuer, audience, scopes/roller, nyckelinnehav |
| Vad är avtalet? | Metoder, scheman, status- och felkatalog, gränser, paginering, idempotens och livscykel |
| När får återförsök göras? | Deadline, idempotent semantik eller key, backoff, retrybudget och uppslagsoperation |
| Hur förhindrar jag Lost Updates? | ETag/`If-Match`, verksamhetsspecifikt versionsnummer eller transaktionell operation |
| Hur känner jag igen partiella tillstånd? | Jobb-/eventstatus, request-ID, trace, kö-/consumer lag, reconciliation |
| Hur ändras systemet? | Staging/Canary, contract- och consumer-tester, rollback, deprecation/sunset |
| Hur återställs systemet? | Konfiguration, avtal, secret-/certifikatreferenser, cursor/job, replay- och reconciliation-test |
| Vad ska ingå i runbooken? | kända felkoder, 401/403/404/409/412/429/5xx-vägar, kontaktpersoner och eskaleringsuppgifter |

## Källor

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
