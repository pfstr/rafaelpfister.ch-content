---
title: "APIs: contratos, identidades y límites de fallo distribuidos"
blatt: "apis"
description: "APIs para administradores de infraestructura y mensajería: REST, HTTP y JSON, RPC, GraphQL, gRPC y APIs de eventos, OpenAPI y esquemas, OAuth e identidades de carga de trabajo, gateways, tiempos de espera, reintentos, idempotencia, paginación, límites de tasa, webhooks, observabilidad, versionado e historia técnica."
fakten:
  - label: Función del sistema
    wert: interfaz de administración o datos legible por máquina entre componentes separados
    href: https://ics.uci.edu/~fielding/pubs/dissertation/net_app_arch.htm
  - label: REST
    wert: estilo arquitectónico con restricciones; no equivale a HTTP más JSON
    href: https://ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm
  - label: Modelo HTTP
    wert: recurso/URI · método · campos · representación · estado
    href: https://www.rfc-editor.org/rfc/rfc9110.html
  - label: Contrato
    wert: describir de forma legible por máquina operaciones, esquemas, errores, seguridad y ciclo de vida
    href: https://spec.openapis.org/oas/
  - label: Formatos de datos
    wert: JSON · XML · Protocol Buffers · formatos binarios y de streaming
    href: https://www.rfc-editor.org/rfc/rfc8259.html
  - label: Estilos de interacción
    wert: solicitud/respuesta · RPC · consulta · flujo · evento/webhook
    href: https://ics.uci.edu/~fielding/pubs/dissertation/net_arch_styles.htm
  - label: Identidad
    wert: API key, certificado de cliente o token; credencial y autorización separadas
    href: https://www.rfc-editor.org/rfc/rfc9700.html
  - label: Repetición
    wert: solo con semántica conocida; un tiempo de espera no significa que no ocurriera nada en el servidor
    href: https://www.rfc-editor.org/rfc/rfc9110.html
  - label: Concurrencia
    wert: ETag/If-Match o un número de versión de negocio evita actualizaciones perdidas
    href: https://www.rfc-editor.org/rfc/rfc9110.html
  - label: Formato de error
    wert: estado HTTP más tipo de problema estable legible por máquina e ID de solicitud
    href: https://www.rfc-editor.org/rfc/rfc9457.html
  - label: Estado operativo
    wert: latencia · tasa de errores · saturación · cuota · caducidad del token · retraso de cola/consumidor
    href: https://opentelemetry.io/docs/specs/otel/trace/
  - label: Evidencia administrativa
    wert: cliente · identidad · alcance · endpoint · versión del contrato · ID de solicitud · resultado
    href: https://www.rfc-editor.org/rfc/rfc9110.html
werbung:
  - newsletter
ctaThemen:
  - cloudflare-workers
  - powershell
  - automatisierung
translationSourceHash: 4b29095b24ea1576608e147b1904a65a72c2b09b2e72f991e3e27a64dbcb3332
translationModel: gpt-5.6-terra
translatedAt: 2026-10-03T09:31:33.714Z
translationReview: automatic
---

# APIs: contratos, identidades y límites de fallo distribuidos

Una interfaz de programación de aplicaciones es una **frontera de contrato y confianza** entre componentes operados de forma independiente. El contrato define qué operaciones y datos existen; el tiempo de ejecución decide sobre transporte, identidad, autorización, comportamiento temporal y errores. Para los administradores, esta separación es fundamental: una solicitud JSON sintácticamente válida puede llegar al tenant equivocado, fallar con un token válido para la audiencia equivocada o haberse ejecutado correctamente en el servidor a pesar de un tiempo de espera del cliente.

Las APIs están presentes en todas partes de los entornos de mensajería: entre el cliente de administración y la plataforma de correo, el gateway y el directorio, el monitoreo y el backend de telemetría, la aplicación en la nube y el receptor de webhooks. Una GUI puede usar la misma interfaz, pero normalmente solo representa parte de sus estados y errores. Por tanto, una operación fiable no comienza con una llamada curl aislada, sino con el **estilo de interfaz, el contrato, el modelo de recursos, la identidad, las transiciones de estado y la semántica de recuperación**.

La explicación sigue una llamada API desde el cliente hasta la respuesta de negocio. Primero trata el transporte y el contrato, después la identidad, el tratamiento de errores y los eventos; solo entonces aborda la operación del gateway, la seguridad y el diagnóstico.

## Clasificación como sistema distribuido

Una API de red no es solo código de aplicación. Una llamada típica atraviesa:

1. Biblioteca de cliente, CLI o proceso de automatización;
2. Resolución de nombres, enrutamiento y establecimiento de conexión;
3. [TLS](/kb/tls), proxy o malla de servicios;
4. Balanceador de carga, API Gateway o proxy inverso;
5. Autenticación, validación de tokens y autorización;
6. Servicio de aplicación, caché, cola y base de datos;
7. Ruta de respuesta, serialización y evaluación del cliente.

El trabajo arquitectónico de Roy Fielding distingue explícitamente los sistemas basados en red de la ejecución local transparente: la comunicación de red tiene su propia latencia, costes y modos de fallo. Un estilo arquitectónico es un conjunto coordinado de restricciones que da lugar a determinadas propiedades y compromisos ([Fielding – Network-based Application Architectures](https://ics.uci.edu/~fielding/pubs/dissertation/net_app_arch.htm), [Fielding – Network-based Architectural Styles](https://ics.uci.edu/~fielding/pubs/dissertation/net_arch_styles.htm)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-apis.svg?v=20260813" title="Interaktive Infografik: API-Aufruf über DNS, TLS, Gateway, Identität und Dienst sowie Vertrag, Fehler- und Wiederholungssemantik, Events und Betriebsnachweis" loading="lazy">
  <a href="/images/kb-interaktiv-apis.svg?v=20260813">Abrir directamente el gráfico interactivo</a>.
</iframe>

La pila relevante se documenta para cada integración:

| Capa | Ejemplos | Área de fallo típica |
|---|---|---|
| Identificación | URI, descubrimiento de servicios, DNS | host, región, tenant o ruta API incorrectos |
| Transporte | TCP/TLS, HTTP/1.1, HTTP/2, HTTP/3 | tiempo de espera, proxy, certificado, ALPN, pool de conexiones |
| Interacción | REST, RPC, GraphQL, gRPC, webhook/evento | semántica incorrecta, reintento inadecuado, interrupción del streaming |
| Representación | JSON, XML, Protobuf, Multipart, datos binarios | error de esquema, codificación, tamaño o compatibilidad |
| Contrato | OpenAPI, JSON Schema, Protobuf IDL, GraphQL SDL, AsyncAPI | cambio incompatible, divergencia entre documentación y ejecución |
| Identidad | API key, token OAuth, mTLS, solicitud firmada | caducidad, scope, audiencia, rotación de claves, desfase de reloj |
| Política | Gateway, WAF, RBAC/ABAC, cuota | 401/403/429, normalización de cabeceras, principal incorrecto |
| Estado | servicio, caché, cola, base de datos | ejecución parcial, retraso de replicación, consistencia eventual |
| Evidencia | ID de solicitud, traza, registro de auditoría, métricas | falta de correlación, muestreo, protección de datos |

## REST es un estilo arquitectónico, no un formato de datos

REST designa las restricciones descritas por Fielding para sistemas hipermedia distribuidos: cliente/servidor, ausencia de estado, caché, interfaz uniforme, capas y código bajo demanda opcional. La interfaz uniforme incluye identificación de recursos, manipulación mediante representaciones, mensajes autodescriptivos e hipermedia como máquina de estados. La estandarización de la interfaz mejora la visibilidad y el desarrollo independiente, pero puede ser menos eficiente que protocolos especializados ([Fielding – Representational State Transfer](https://ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)).

Por tanto, una API HTTP con JSON y rutas como `/v1/getUser` no es automáticamente REST. Puede ser simplemente RPC sobre HTTP. Esto no es intrínsecamente malo; se vuelve problemático cuando los operadores esperan propiedades que el estilo real no ofrece. Por ejemplo, un cliente no puede repetir de forma segura un POST solo porque el endpoint se llame «REST».

### URI, recurso y representación

Una URI identifica un recurso; no garantiza ni accesibilidad ni una operación determinada. RFC 3986 separa explícitamente la identificación de la interacción. El esquema, la autoridad, la ruta, la consulta y el fragmento tienen sintaxis definida, mientras que la API concreta determina la semántica de sus recursos ([RFC 3986 – URI Generic Syntax](https://www.rfc-editor.org/rfc/rfc3986.html)).

Un recurso no es su archivo JSON. El mismo recurso puede representarse como JSON, XML u otro formato según la cabecera `Accept`. `Content-Type` describe el cuerpo enviado, `Accept` la respuesta preferida. El estado, los campos y el cuerpo forman conjuntamente el mensaje; registrar solo el cuerpo JSON hace desaparecer información de diagnóstico importante.

## Semántica HTTP: el método antes que el nombre de la ruta

RFC 9110 separa la identificación del recurso de la semántica de la solicitud. El método define la operación prevista; la ruta URI por sí sola no. «Safe» significa que el cliente no pretende cambiar el estado. «Idempotent» significa que varias solicitudes idénticas tienen el mismo efecto previsto que una sola solicitud; los efectos secundarios, como el registro, pueden producirse varias veces ([RFC 9110 – HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)).

| Método | Safe | Idempotent | Semántica API típica | Reintento sin información adicional |
|---|---:|---:|---|---|
| GET | sí | sí | leer representación | posible en principio, pero tener en cuenta carga/cuota |
| HEAD | sí | sí | metadatos sin cuerpo | posible en principio |
| OPTIONS | sí | sí | capacidades/opciones de comunicación | posible en principio |
| PUT | no | sí | sustituir estado bajo una URI conocida | posible si el contrato cumple realmente la semántica PUT |
| DELETE | no | sí | eliminar asignación | efecto repetible; el estado de respuesta puede cambiar |
| POST | no | no | procesar, acción o recurso nuevo | no repetir a ciegas |
| PATCH | no | no en general | modificación parcial | solo con semántica documentada de parche e idempotencia |

La idempotencia describe el **efecto previsto en el servidor**, no la respuesta de transporte. Un PUT puede haberse completado en el servidor mientras se pierde la respuesta. Entonces, repetir el PUT es semánticamente admisible; repetir un POST puede crear un segundo objeto o mensaje. Para operaciones no idempotentes, se requiere un identificador de operación generado por el cliente, una Idempotency-Key específica del fabricante o una consulta posterior de estado.

### Los códigos de estado son categorías, no un diagnóstico completo

- `2xx`: la solicitud se procesó del modo definido por el estado; no todo `202 Accepted` está ya finalizado desde el punto de vista de negocio.
- `3xx`: se requiere otra acción u otra representación; las redirecciones pueden cambiar el método y el flujo de credenciales.
- `400`: la solicitud es incorrecta desde la perspectiva del servidor.
- `401`: faltan credenciales de autenticación o no son válidas; la respuesta usa normalmente `WWW-Authenticate`.
- `403`: el servidor entiende la solicitud, pero la rechaza.
- `404`: recurso no encontrado u ocultado deliberadamente; no es una prueba segura de su inexistencia.
- `409`: conflicto con el estado actual.
- `412`: no se cumple una precondición como `If-Match`.
- `429`: demasiadas solicitudes en una ventana de tiempo; `Retry-After` puede indicar un tiempo de espera.
- `5xx`: el servidor no pudo satisfacer una solicitud básicamente válida; no es automáticamente reintentable.

RFC 6585 define `429 Too Many Requests`, pero ni el alcance de la cuota ni el contador. Estos pueden aplicarse por credencial, usuario, tenant, recurso, región o clúster ([RFC 6585 – Additional HTTP Status Codes](https://www.rfc-editor.org/rfc/rfc6585.html)). Por ello, el cliente guarda el estado, los campos de respuesta relevantes, el ID de solicitud y el cuerpo de error truncado.

## Variantes de transporte y costes de conexión

La semántica HTTP está separada de la versión wire concreta. HTTP/1.1 usa reglas de framing de mensajes textuales, HTTP/2 flujos multiplexados y framing binario, y HTTP/3 implementa HTTP sobre QUIC. Un API Gateway puede aceptar HTTP/2 del lado del cliente y hablar HTTP/1.1 con el backend; el protocolo en el cliente no demuestra toda la ruta al backend ([RFC 9112 – HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112.html), [RFC 9113 – HTTP/2](https://www.rfc-editor.org/rfc/rfc9113.html), [RFC 9114 – HTTP/3](https://www.rfc-editor.org/rfc/rfc9114.html)).

Los administradores no observan solo la latencia de la solicitud, sino también el tiempo DNS y de conexión, el handshake TLS, la reutilización de conexiones, la versión HTTP/ALPN, el tiempo de proxy/gateway, el tiempo hasta el primer byte, la transferencia del cuerpo, el número de reintentos y la duración total de reloj.

### Comprobar la ruta de nombres, TCP y TLS

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) y [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) comprueban [DNS](/kb/dns). [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) y [`nc`](https://man.openbsd.org/nc) comprueban TCP. [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) muestra el handshake TLS, la cadena de certificados y ALPN; ninguna de estas pruebas demuestra una autorización API correcta.

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

Una vez comprendidos el método, el transporte y la representación, viene la primera particularidad distribuida: un cambio correcto no tiene por qué ser visible inmediatamente en todas las rutas de lectura.

## Caché y lectura tras escritura

La caché HTTP almacena representaciones según claves de caché y directivas. `Cache-Control`, `Vary`, los validadores y las reglas de autenticación determinan si se reutilizan y cómo. Un `200` puede proceder de una caché; según la arquitectura, un GET inmediatamente posterior a un PUT aún puede ver un estado antiguo ([RFC 9111 – HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111.html)).

Preguntas administrativas:

- ¿Hay una caché de navegador, proxy, CDN, gateway o aplicación en la ruta?
- ¿Qué cabeceras forman la clave de caché, especialmente Authorization y tenant?
- ¿La representación es privada, pública o no almacenable en caché?
- ¿Durante cuánto tiempo se pueden almacenar en caché las respuestas negativas?
- ¿Existe read-your-writes o consistencia eventual?
- ¿Qué región/réplica lee el GET posterior?
- ¿ETag es una versión de contenido o solo un validador de caché?

Limpiar la caché no es una reparación universal. Puede generar picos de carga y ocultar la inconsistencia real.

## Límites de tasa, cuotas y saturación

El límite de tasa, la cuota y el límite de concurrencia son controles distintos:

- **Tasa:** solicitudes o puntos de coste por ventana de tiempo.
- **Cuota:** consumo total por día, mes o suscripción.
- **Concurrencia:** solicitudes/flujos que se ejecutan simultáneamente.
- **Límite de payload:** tamaño de cuerpo, objeto, lote o respuesta.
- **Límite de complejidad:** profundidad de consulta, coste GraphQL o relaciones expandidas.

`429` puede entregar `Retry-After`, pero las cabeceras de límite de tasa específicas de cada fabricante no son uniformes. El cliente trata los campos documentados como parte del contrato concreto, no como un estándar universal. Limita localmente la tasa de consultas, distribuye el presupuesto entre cargas de trabajo y guarda el alcance, el presupuesto restante y la hora de restablecimiento.

El throttling es una señal de protección, no un modo de rendimiento normal. Las oleadas persistentes de 429 indican paginación inadecuada, falta de caché, paralelismo excesivo o capacidad insuficiente.

## Cuerpos de error y correlación

Un estado HTTP es demasiado impreciso para la automatización de negocio. RFC 9457 define Problem Details con una URI `type` estable, `title`, `status`, `detail` y `instance`, además de campos de extensión. El tipo de problema es la identidad legible por máquina; el texto de redacción libre no está destinado a analizadores ([RFC 9457 – Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457.html)).

Un buen contrato de errores proporciona un tipo de error/problema estable, estado HTTP/RPC, detalle seguro, rutas de campos afectados, ID de solicitud/correlación, posibilidad de reintento y un enlace de documentación. El cliente no registra tokens completos, cabeceras Authorization ni payloads confidenciales.

### Capturar una respuesta de error con cabeceras

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

## RPC, GraphQL y gRPC

No todas las APIs encajan en el estilo de recursos.

| Estilo | Centro del contrato | Ventaja | Límite operativo |
|---|---|---|---|
| REST/HTTP | recurso, representación, semántica HTTP | intermediarios web, caché, amplio soporte de herramientas | convenciones de detalle no uniformes |
| RPC | servicio y operación | asignación directa de acciones de negocio | reintento/idempotencia explícitos por método |
| GraphQL | esquema tipado y consulta del cliente | selección flexible de datos relacionados | coste de consulta, N+1, a menudo HTTP 200 pese a errores de campo |
| gRPC | servicio/mensaje Protobuf | generación de código, HTTP/2, unario y streaming | framing binario, proxies, estado/trailers gRPC |
| API de eventos | canal, mensaje, tipo de evento | desacoplamiento y procesamiento asíncrono | orden, deduplicación, repetición, retraso del consumidor |

La especificación GraphQL define el lenguaje, el sistema de tipos, la validación y la ejecución; el transporte, la autenticación, el límite de tasa y los costes operativos de consulta son contratos adicionales ([GraphQL Specification](https://spec.graphql.org/September2025/)). Los errores de campo pueden ocurrir junto con datos parciales; un estado HTTP por sí solo no describe el resultado.

gRPC asigna canales, RPC y mensajes con prefijo de longitud a flujos HTTP/2. El estado gRPC se transmite en trailers y debe distinguirse del estado HTTP. Las llamadas no son automáticamente idempotentes; deadline, cancelación y política de reintentos se entienden por servicio ([gRPC – What is gRPC?](https://grpc.io/docs/what-is-grpc/), [gRPC over HTTP/2 protocol](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md)). Protocol Buffers proporciona un modelo de interfaz y serialización; los números de campo son anclas de compatibilidad y no deben reutilizarse con otro significado tras su eliminación ([Protocol Buffers – Language Guide](https://protobuf.dev/programming-guides/proto3/)).

[`grpcurl`](https://github.com/fullstorydev/grpcurl) puede utilizar Server Reflection o descriptores locales para examinar servicios gRPC. Reflection es en sí misma una superficie expuesta y no se habilita públicamente sin revisión.

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

## Representaciones: JSON es sintaxis, el esquema es el contrato

RFC 8259 define JSON como formato de intercambio con objetos, arrays, números, cadenas, booleanos y null. JSON no define qué campo es un ID estable, si un campo ausente y `null` significan lo mismo, qué zona horaria tiene una marca de tiempo o si se toleran propiedades desconocidas ([RFC 8259 – JSON](https://www.rfc-editor.org/rfc/rfc8259.html)).

Esta semántica pertenece a un esquema y a la documentación del contrato:

- nombre de campo, tipo, formato y unidad;
- required, optional, nullable y valor predeterminado;
- solo lectura/solo escritura y generado por el servidor;
- valores enum y comportamiento ante valores desconocidos;
- formato de hora, zona horaria y precisión;
- ID estable frente a nombre visible;
- semántica de referencia, incrustación y eliminación;
- regla de compatibilidad para campos nuevos o eliminados.

JSON Schema define vocabularios para validar instancias JSON. Un esquema puede comprobar la estructura, pero no sustituye invariantes de negocio ni autorización ([JSON Schema – Specification](https://json-schema.org/specification)). La OpenAPI Specification puede describir de forma legible por máquina operaciones HTTP, parámetros, esquemas de solicitud/respuesta y esquemas de seguridad; no demuestra que la implementación en ejecución corresponda al documento ([OpenAPI Specification](https://spec.openapis.org/oas/)).

### Inspeccionar respuestas y campos sin GUI

[`Invoke-RestMethod`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod) deserializa respuestas estructuradas; [`Invoke-WebRequest`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest) devuelve más detalles HTTP. [`curl`](https://curl.se/docs/manpage.html) muestra solicitud/respuesta y tiempos, [`jq`](https://jqlang.org/manual/) filtra JSON.

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

## Contratos y divergencia de contrato

Un contrato API completo comprende más que esquemas para la ruta feliz:

| Área contractual | Debe definirse |
|---|---|
| Descubrimiento | URL base, región, tenant, endpoint de servicio/metadatos |
| Operación | método/RPC/evento, vinculación de parámetros, efecto secundario |
| Datos | esquema, ID, orden, semántica de null/valor predeterminado, límites de tamaño |
| Seguridad | flujo de autenticación, tipo de credencial, audiencia, scope/rol, duración del token |
| Errores | estado/código/tipo de problema, posibilidad de reintento, ID de solicitud |
| Consistencia | lectura tras escritura, retraso de replicación, caché y ETag |
| Cantidad | paginación, filtro, ordenación, semántica de snapshot/cursor |
| Tiempo | deadline de cliente/gateway/servidor, Retry-After, desfase de reloj |
| Ciclo de vida | versión del contrato, deprecación, Sunset y ruta de migración |
| Operación | cuota, SLO, mantenimiento, página de estado, correlación de soporte |

La divergencia de contrato surge cuando el documento, el SDK y la implementación en producción se separan. Por eso, la especificación publicada se guarda como artefacto versionado, se valida en CI y se prueba contra un entorno de pruebas real. Los clientes generados reducen el trabajo de escritura, pero también propagan errores y cambios incompatibles del esquema a muchos consumidores.

Las pruebas orientadas al consumidor pueden hacer visibles las suposiciones de un cliente. No sustituyen la semántica del proveedor: un mock puede entregar un `200` aunque la producción devuelva otra cabecera, otra paginación o un nuevo enum tras una actualización del gateway.

Hasta aquí, la llamada era técnicamente válida. La comprobación de seguridad decide si también puede ser ejecutada por la identidad correcta para el objeto correcto.

## La autenticación no es autorización

Una API key o un token responde primero **qué cliente o principal** habla. La autorización decide después qué acción está permitida sobre qué recurso y en qué scope. Por ello, un token válido puede ser rechazado correctamente con `403`.

| Procedimiento | Ventaja y uso | Riesgo operativo |
|---|---|---|
| API key | identificación simple de cliente o ancla de cuota | a menudo larga vigencia, poco scope, copiable |
| Basic Auth | nombre de usuario/contraseña sobre TLS | ciclo de vida de contraseñas, límites de MFA/delegación |
| mTLS | TLS mutuo, certificado de cliente | PKI, rotación, terminación de proxy, asignación a principal |
| token de acceso OAuth | autorización delegada o de carga de trabajo con scope/audiencia | obtención, caducidad, consentimiento, repetición del token |
| solicitud firmada | integridad de partes seleccionadas del mensaje | canonicalización, desfase de reloj, almacén de nonce/repetición |
| identidad de red | redes privadas, malla de servicios, certificados de carga de trabajo | no debe sustituir silenciosamente el RBAC de negocio |

OAuth 2.0 define roles y mecanismos de grant para emitir tokens de acceso; los Bearer Tokens pueden ser usados por cualquiera que los posea ([RFC 6749 – OAuth 2.0](https://www.rfc-editor.org/rfc/rfc6749.html), [RFC 6750 – Bearer Token Usage](https://www.rfc-editor.org/rfc/rfc6750.html)). La BCP de seguridad RFC 9700 exige privilegios mínimos, restricción de audiencia y protección de flujos de redirección; prohíbe el Resource Owner Password Credentials Grant. Para proteger contra repeticiones, menciona tokens vinculados al emisor mediante mTLS o DPoP ([RFC 9700 – OAuth 2.0 Security BCP](https://www.rfc-editor.org/rfc/rfc9700.html), [RFC 8705 – OAuth mTLS](https://www.rfc-editor.org/rfc/rfc8705.html), [RFC 9449 – DPoP](https://www.rfc-editor.org/rfc/rfc9449.html)).

OpenID Connect añade una capa de identidad sobre OAuth; un ID Token es para el cliente y no automáticamente un token de acceso para una API ([OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html)). Un JWT es solo un formato compacto de claims. La validación de firma por sí sola no basta: deben validarse algoritmo, emisor, audiencia, claims temporales, selección de claves y claims específicos de la aplicación ([RFC 7519 – JSON Web Token](https://www.rfc-editor.org/rfc/rfc7519.html), [RFC 8725 – JWT Best Current Practices](https://www.rfc-editor.org/rfc/rfc8725.html)).

### Obtener un token de carga de trabajo y llamar a la API

El flujo Client Credentials solo es adecuado cuando la aplicación actúa en su propio nombre y puede proteger su credencial de forma segura. El secreto, certificado o identidad de carga de trabajo federada, endpoint de token, audiencia/recurso y scope pertenecen a la documentación concreta de la plataforma.

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

Los secretos no aparecen ni en la línea de comandos ni en la transcripción ni en el registro de depuración. El ejemplo muestra el flujo de protocolo, no un transporte de secretos adecuado para procesos de producción.

## API Gateway y fronteras de confianza

Un gateway puede terminar TLS, validar tokens y agrupar enrutamiento, cuotas, filtros de esquema, reglas WAF y observabilidad. Por tanto, es un punto de control y un área de fallo. El servicio backend no debe asumir silenciosamente que cada solicitud llegó exactamente por esta ruta de gateway.

- **Cliente → Gateway:** identidad pública del host, TLS, DDoS/cuota, credencial de cliente.
- **Gateway → Servicio:** identidad mTLS o de carga de trabajo propia; no confiar ciegamente en la IP de origen.
- **Proveedor de identidad → Validador:** metadatos del emisor, JWKS, rotación de claves, caché y reloj.
- **Servicio → Almacenamiento de datos:** autorización de negocio y límite de tenant.
- **Proveedor de webhook → Receptor:** firma, ventana temporal, ID de evento y verificación de repetición.

RFC 9700 advierte expresamente sobre cabeceras de reenvío entrantes no verificadas en proxies inversos. El proxy debe depurar los campos relevantes para la seguridad; el enlace interno debe protegerse contra escucha, inyección y repetición ([RFC 9700 – OAuth 2.0 Security BCP](https://www.rfc-editor.org/rfc/rfc9700.html)).

El estado del gateway `200` no demuestra que una cola o replicación posterior esté sana. A la inversa, un backend puede estar sano mientras DNS, certificado, validador de token o cuota bloquean a todos los clientes.

Tras el gateway y la comprobación de autorización, queda la pregunta operativa más difícil: ¿qué ocurrió si el cliente no recibe una respuesta a tiempo? Un tiempo de espera no demuestra que el servidor no haya cambiado nada.

## Tiempos de espera, deadlines y ejecución parcial

«Timeout» no es un resultado del servidor. El cliente solo sabe que no llegó una respuesta utilizable dentro de su plazo. La solicitud puede haber fallado antes de establecer la conexión, haber sido descartada en el gateway, seguir activa en el servicio o haberse confirmado ya mientras solo se perdió la respuesta.

Cada capa puede tener su propio plazo: DNS, conexión, TLS, tiempo total del cliente, proxy, gateway, upstream, base de datos y cola. El deadline externo debe coordinarse con los plazos internos; de lo contrario, el cliente abandona tras 30 segundos mientras el servidor sigue trabajando 60 segundos y un reintento inicia la misma acción en paralelo.

Siempre que sea posible, un servicio propaga un deadline restante en lugar de reiniciar el tiempo completo para cada salto. La cancelación es best effort: no demuestra que se haya revertido un efecto secundario ya confirmado.

## Reintentos, backoff e idempotencia

La repetición automática solo está permitida si **la clase de error y la operación** lo permiten. Un cliente robusto aclara:

1. ¿Se estableció alguna conexión?
2. ¿Hay un estado o un error de protocolo?
3. ¿La operación es safe/idempotent o está protegida con deduplicación?
4. ¿El servidor proporciona `Retry-After` o una indicación de backoff específica del producto?
5. ¿Queda suficiente deadline de extremo a extremo?
6. ¿Un reintento aumenta una sobrecarga?

El backoff exponencial con jitter evita oleadas sincronizadas de reintentos. El número de intentos es limitado y forma parte de la latencia total. `401` o `403` no se solucionan repitiendo con mayor frecuencia; `429` exige respetar la cuota; un `500` después de un POST puede haber dejado un efecto secundario parcial pese al cuerpo de error.

Para una operación de negocio, el cliente guarda un ID de operación estable. El servidor conserva el resultado o el estado de deduplicación al menos durante la ventana máxima de reintento. Si falta ese compromiso, el cliente consulta antes del reintento utilizando un ID de objeto estable o una condición de búsqueda.

## Concurrencia optimista

Read-modify-write sin condición de versión provoca actualizaciones perdidas:

```text
Client A liest Version 7     Client B liest Version 7
Client A schreibt Änderung  → Version 8
Client B schreibt alten Stand plus Änderung → A geht verloren
```

HTTP admite solicitudes condicionales con validadores como `ETag`. El cliente lee el ETag y envía `If-Match` al modificar; si la representación ha cambiado, el servidor responde con `412 Precondition Failed` en lugar de sobrescribir un estado ajeno. La API concreta debe documentar si el ETag es suficientemente fuerte para esta semántica ([RFC 9110 – HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)).

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

## Paginación, filtros y conjuntos consistentes

Un endpoint que hoy entrega 50 objetos puede entregar 50'000 mañana. La paginación forma parte del contrato:

- **Offset/Page:** sencillo, pero las inserciones y eliminaciones pueden generar duplicados o huecos.
- **Cursor/Continuation Token:** codifica el progreso en el servidor; el token es opaco y no se interpreta.
- **Keyset:** ordenado por ID de continuación estable y único.
- **Snapshot:** mantiene una vista consistente a través de varias páginas, pero requiere estado del servidor o un ancla de versión.

El cliente sigue el Next-Link o cursor documentado y no lo construye a partir de suposiciones. RFC 8288 define enlaces tipados, pero no una paginación universal; la relación y el formato del cuerpo concretos siguen siendo parte del contrato API ([RFC 8288 – Web Linking](https://www.rfc-editor.org/rfc/rfc8288.html)).

El filtro y la ordenación deben ser estables entre páginas. Una ordenación solo por una marca de tiempo no única es insuficiente; debe incluir un desempate como un ID inmutable. Para APIs delta/de cambios se documentan cursor, caducidad y ruta de resincronización.

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


## Webhooks, eventos y APIs asíncronas

Un éxito HTTP síncrono y un procesamiento de negocio finalizado son estados diferentes. Un `202 Accepted` solo confirma según RFC 9110 que el servidor aceptó el procesamiento; la tarea aún puede fallar más tarde. A la inversa, un receptor de webhook a menudo solo confirma la aceptación persistida de un evento. Quien equipara `2xx` con «proceso de negocio finalizado» pierde precisamente los estados intermedios relevantes para colas, reintentos y fallos parciales ([RFC 9110 – HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)).

Un evento útil operativamente contiene al menos un ID de evento estable, tipo de evento y versión de esquema, hora de creación, productor, ID de recurso y, si el orden es relevante para el negocio, una versión de recurso o secuencia. CloudEvents estandariza para ello una envoltura de eventos independiente del fabricante; AsyncAPI describe de forma legible por máquina canales y operaciones de mensajes, de manera similar al papel de OpenAPI para APIs de solicitud/respuesta ([CloudEvents Specification](https://github.com/cloudevents/spec), [AsyncAPI Specification](https://www.asyncapi.com/docs/reference/specification/latest)).

Los webhooks se operan como un cliente externo que repite:

- El emisor firma el **cuerpo de la solicitud sin modificar** junto con metadatos de tiempo o nonce; el receptor valida firma, ventana temporal aceptada y contexto de destino antes de analizarlo. Las HTTP Message Signatures estandarizadas pueden vincular criptográficamente componentes y campos derivados ([RFC 9421 – HTTP Message Signatures](https://www.rfc-editor.org/rfc/rfc9421.html)).
- El receptor deduplica mediante ID de evento y guarda la aceptación antes de la confirmación positiva. El procesamiento se diseña de forma idempotente.
- Los reintentos tienen duración limitada, backoff y una ruta de dead letter o cuarentena. Una repetición queda registrada y no crea una nueva identidad de negocio.
- Una **ejecución de reconciliación** periódica compara el sistema fuente con el estado local. Los webhooks son aceleradores, no necesariamente la única fuente de verdad.

## Seguridad de API: objeto, función y flujo de datos

Una comprobación de token superada solo responde quién o qué carga de trabajo habla y para qué audiencia está destinada la credencial. Para cada objeto y operación, la aplicación debe decidir adicionalmente si esta identidad puede leer o modificar exactamente este tenant, usuario, clave o conjunto de mensajes. Por ello, OWASP API Security Top 10 destaca, entre otras, Broken Object Level Authorization, Broken Authentication, consumo ilimitado de recursos, SSRF e inventario API defectuoso como clases de riesgo propias ([OWASP API Security Top 10](https://owasp.org/API-Security/editions/2023/en/0x11-t10/)).

Para APIs de infraestructura se derivan los siguientes controles concretos:

- **Relación con el objeto:** los anclajes de tenant y objeto provienen de un contexto validado en el servidor, no solo de un campo de ruta o cuerpo elegible libremente.
- **Límites de entrada:** Content-Type, esquema, longitudes de campo, anidamiento, tamaño total, relación de compresión y tiempo de procesamiento están limitados.
- **Conexiones salientes:** las URL de solicitudes o webhooks pasan por allowlist, comprobación DNS/IP y política de salida; las redirecciones se vuelven a comprobar.
- **Credenciales:** los tokens no aparecen ni en URI ni en registros; los secretos se rotan, se restringen a la audiencia de destino y scopes mínimos, y no se incrustan en artefactos de cliente.
- **Saltos de confianza:** si un gateway termina [TLS](/kb/tls), el salto backend debe autenticarse y autorizarse por separado. Una cabecera Forwarded de confianza solo surge en una frontera de proxy controlada.
- **Auditoría:** los cambios privilegiados registran cliente, principal, objeto de destino, acción, referencia antes/después, ID de solicitud y resultado, sin secreto ni payload sensible completo.

OAuth 2.0 Security Best Current Practice desaconseja, entre otras cosas, el Resource Owner Password Credentials Grant, exige comparaciones exactas de URI de redirección y prefiere tokens vinculados al emisor o de corta duración cuando el modelo de amenazas lo requiere ([RFC 9700 – Best Current Practice for OAuth 2.0 Security](https://www.rfc-editor.org/rfc/rfc9700.html)). Mutual TLS y DPoP son dos procedimientos distintos para la vinculación al emisor; ambos modifican la operación de claves y el diagnóstico de errores, y no son meros interruptores del gateway ([RFC 8705 – OAuth 2.0 Mutual-TLS Client Authentication](https://www.rfc-editor.org/rfc/rfc8705.html), [RFC 9449 – OAuth 2.0 Demonstrating Proof of Possession](https://www.rfc-editor.org/rfc/rfc9449.html)).

Los reintentos, la paginación y los eventos generan varios procesos técnicos para una acción de negocio. La correlación y la auditoría deben volver a unirlos en un flujo comprensible.

## Observabilidad y llamadas demostrables

Las métricas muestran el volumen, los registros decisiones individuales y las trazas la ruta de una solicitud a través de fronteras de proceso. OpenTelemetry modela una traza como un conjunto causal de spans y define ID de traza y de span para la correlación ([OpenTelemetry – Traces](https://opentelemetry.io/docs/specs/otel/trace/)). Para fines administrativos, una llamada API debería permitir reconstruir al menos los siguientes hechos:

| Dimensión | Evidencia operativa |
|---|---|
| Llamador | ID de cliente, carga de trabajo o usuario; método de autenticación; roles/scopes efectivos |
| Destino | host, tenant, versión de API/contrato, método u operación, identificador de recurso estable |
| Ejecución | hora de inicio, duración total, tiempo DNS/conexión/TLS si está disponible, deadline, número de reintento |
| Resultado | estado de transporte, código de error de negocio, tamaño de respuesta, estado de límite de tasa/cuota |
| Correlación | ID de solicitud del servidor, ID de traza, ID de trabajo/evento e ID de mensaje en caso de mensajería |

Los ID se transmiten a través de fronteras de proceso, pero no se adoptan ciegamente de clientes externos arbitrarios como autoridad interna. Las etiquetas de métricas evitan ID de usuario, rutas completas y otros valores de alta cardinalidad. Los payloads, cabeceras Authorization, cookies y firmas de webhook no forman parte de la telemetría de forma predeterminada. Una traza puede demostrar la ruta, pero no sustituye una evidencia de auditoría protegida contra manipulación sobre un cambio privilegiado.

## Versionado, deprecación y Sunset

Un número de versión no es un ciclo de vida. Primero se distingue entre **extensiones compatibles** y **cambios incompatibles**. Nuevos campos opcionales, valores enum adicionales o un orden cambiado pueden romper clientes pese a una supuesta compatibilidad hacia atrás si estos implementan el contrato de forma demasiado estricta. Por ello, las pruebas de consumidores y la comparación de esquemas no comprueban solo rutas, sino semántica, permisos, errores y valores límite.

Las versiones pueden estar en la ruta, host, cabecera o tipo de medio; lo decisivo es que el enrutamiento, la documentación, la telemetría y el soporte nombren inequívocamente la misma variante. Para la retirada gradual, RFC 9745 estandariza el campo HTTP `Deprecation`; RFC 8594 define `Sunset` como el momento a partir del cual previsiblemente un recurso ya no responderá. Ninguno de los dos sustituye una guía de migración, un enlace alternativo ni un inventario de clientes demostrado ([RFC 9745 – The Deprecation HTTP Response Header Field](https://www.rfc-editor.org/rfc/rfc9745.html), [RFC 8594 – The Sunset HTTP Header Field](https://www.rfc-editor.org/rfc/rfc8594.html)).

Un proceso de retirada fiable incluye inventario de consumidores, métricas de uso por versión y cliente, fechas anunciadas, funcionamiento en paralelo, entorno de prueba, ruta de reversión y una decisión explícita de apagado. «Anunciado en el wiki» no demuestra que se hayan migrado automatizaciones desatendidas.

## Modelos operativos: local, nube y plano de control

La ubicación de una API no determina por sí sola la seguridad ni la controlabilidad. Una interfaz local puede estar ligada directamente a cuentas privilegiadas del sistema operativo, claves de larga duración y redes poco segmentadas. En cambio, un plano de control en la nube puede ofrecer identidades de carga de trabajo robustas y registros de auditoría, pero sigue dependiendo de la ruta de Internet, IAM del proveedor, configuración del tenant, cuotas y disponibilidad del servicio. Lo decisivo es el espacio concreto de fallos y confianza.

| Modelo | Límite típico | Preguntas administrativas |
|---|---|---|
| API local de proceso/host | Unix Socket, Named Pipe, loopback o LAN de administración | ¿Qué identidad de SO se aplica? ¿Quién posee socket/ACL? ¿El acceso remoto está realmente excluido? |
| API interna de servicio | segmento, malla de servicios, gateway o balanceador de carga | ¿Dónde terminan TLS y la autorización? ¿Cómo se operan las identidades de servicio y DNS? |
| plano de control SaaS | endpoint del proveedor e IAM del tenant | ¿Qué región, cuota, rutas de auditoría y token se aplican? ¿Cómo funciona Break Glass? |
| plano de datos más plano de control | la configuración controla workers o appliances separados | ¿Cuándo se distribuye un cambio? ¿Cómo se detectan divergencia, rollback y estados parciales? |
| integración de evento/webhook | productor, broker o callback público | ¿Quién posee la entrega, reintento, firma, DLQ y reconciliación? |

Las copias de seguridad no protegen automáticamente una API externa. Para la recuperación se inventarían más bien contratos, configuración de cliente, referencias de secretos, certificados, reglas de gateway, estado de idempotencia, trabajos abiertos y capacidad de reconciliación. Las pruebas de recuperación también deben cubrir tokens caducados, destinos DNS modificados y Continuation Tokens restablecidos.

## Historia técnica

Las primeras interfaces distribuidas solían estar estrechamente ligadas a Remote Procedure Call y stubs específicos de lenguaje. SOAP 1.2 definió más tarde un marco de mensajes basado en XML con un modelo de procesamiento extensible y se difundió junto con WSDL y WS-* en plataformas empresariales ([W3C – SOAP Version 1.2 Part 1](https://www.w3.org/TR/soap12-part1/)). La tesis de Roy Fielding describió REST en 2000 como estilo arquitectónico para sistemas hipermedia distribuidos y derivó las restricciones de los requisitos de la web, no como receta de «HTTP más JSON» ([Fielding – Architectural Styles and the Design of Network-based Software Architectures](https://ics.uci.edu/~fielding/pubs/dissertation/top.htm)).

HTTP evolucionó en paralelo desde conexiones TCP persistentes en HTTP/1.1, pasando por flujos multiplexados en HTTP/2, hasta HTTP/3 sobre QUIC. La semántica de métodos y estados se describe independientemente del transporte en RFC 9110; los formatos wire se encuentran en RFC 9112, RFC 9113 y RFC 9114 ([RFC 9112 – HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112.html), [RFC 9113 – HTTP/2](https://www.rfc-editor.org/rfc/rfc9113.html), [RFC 9114 – HTTP/3](https://www.rfc-editor.org/rfc/rfc9114.html)).

JSON se estandarizó como formato ligero de intercambio; JSON Schema y OpenAPI añadieron contratos de estructura y operaciones legibles por máquina. GraphQL describe un modelo tipado de consulta y ejecución en el que los clientes seleccionan campos; gRPC combina definiciones RPC orientadas a servicios con Protocol Buffers y framing basado en HTTP/2 ([JSON Schema Specification](https://json-schema.org/specification), [OpenAPI Specification](https://spec.openapis.org/oas/), [GraphQL Specification](https://spec.graphql.org/September2025/), [gRPC – What is gRPC?](https://grpc.io/docs/what-is-grpc/), [Protocol Buffers – Language Guide](https://protobuf.dev/programming-guides/proto3/)). Los modelos de eventos y streaming complementan solicitud/respuesta, pero no eliminan las cuestiones de contratos, entrega y consistencia.

## Lista de comprobación administrativa de un vistazo

Tras el contrato, el tiempo de ejecución y la operación, la siguiente lista resume las preguntas que deberían responderse antes de aprobar una API. Está pensada como ayuda de aceptación, no como sustituto de las explicaciones anteriores.

| Pregunta | Evidencia o artefacto |
|---|---|
| ¿Qué estilo de interacción opero? | OpenAPI/esquema GraphQL/Proto/AsyncAPI, operación concreta y perfil de transporte |
| ¿Qué endpoint se aplica? | esquema, FQDN, puerto, ruta base, región/tenant, evidencia DNS y de certificado |
| ¿Quién llama? | ID de cliente/carga de trabajo, tipo de credencial, emisor del token, audiencia, scopes/roles, posesión de clave |
| ¿Cuál es el contrato? | métodos, esquemas, catálogo de estados y errores, límites, paginación, idempotencia y ciclo de vida |
| ¿Cuándo se puede repetir? | deadline, semántica o clave idempotente, backoff, presupuesto de reintentos y operación de consulta |
| ¿Cómo evito actualizaciones perdidas? | ETag/`If-Match`, número de versión de negocio u operación transaccional |
| ¿Cómo detecto estados parciales? | estado de trabajo/evento, ID de solicitud, traza, retraso de cola/consumidor, reconciliación |
| ¿Cómo se modifica? | staging/canary, pruebas de contrato y consumidores, rollback, Deprecation/Sunset |
| ¿Cómo se recupera? | configuración, contratos, referencias de secretos/certificados, cursor/trabajos, prueba de repetición y reconciliación |
| ¿Qué debe incluir el runbook? | códigos de error conocidos, rutas 401/403/404/409/412/429/5xx, contactos y datos de escalación |

## Fuentes

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
