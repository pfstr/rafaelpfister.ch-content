---
title: "Cloudflare Workers: Isolates, Bindings y operación en el Edge"
blatt: "cloudflare-workers"
description: "Cloudflare Workers para administradores de infraestructura, mensajería y plataformas: ruta de solicitudes, workerd y V8 Isolates, handlers y API web, capability bindings, KV, D1, R2 y Durable Objects, enrutamiento, Compatibility Dates, límites, observabilidad, despliegues, recuperación e historia técnica."
fakten:
  - label: Rol del sistema
    wert: plataforma de cómputo orientada a eventos en la red de Cloudflare para HTTP, programaciones, colas, correo electrónico y servicios internos
    href: https://developers.cloudflare.com/workers/runtime-apis/handlers/
  - label: Runtime
    wert: workerd con V8 Isolates; sin proceso de servidor persistente propio ni instancia de contenedor por aplicación
    href: https://developers.cloudflare.com/workers/reference/how-workers-works/
  - label: Modelo de programación
    wert: entrypoints de módulos ES con API estándar web como Request, Response, Fetch, Streams y Web Crypto
    href: https://developers.cloudflare.com/workers/runtime-apis/
  - label: Lenguajes
    wert: JavaScript, TypeScript, Python y Rust; otros lenguajes mediante WebAssembly
    href: https://developers.cloudflare.com/workers/languages/
  - label: Vías de entrada
    wert: workers.dev, Route, Custom Domain, así como eventos Fetch, Scheduled, Queue, E-Mail, Alarm y Tail
    href: https://developers.cloudflare.com/workers/wrangler/configuration/
  - label: Bindings
    wert: capability y API para recursos de plataforma sin claves de acceso expuestas en el código
    href: https://developers.cloudflare.com/workers/runtime-apis/bindings/
  - label: Modelos de estado
    wert: caché local · KV eventual-consistent · D1 SQL · Durable Object fuertemente consistente · objeto R2 · Queue
    href: https://developers.cloudflare.com/workers/platform/storage-options/
  - label: Compatibilidad
    wert: Compatibility Date más flags opcionales congelan los cambios de runtime relevantes para el comportamiento por despliegue
    href: https://developers.cloudflare.com/workers/configuration/compatibility-flags/
  - label: Control Plane
    wert: configuración de Wrangler, Workers API, objetos de versión, despliegues, secretos y Resource Bindings
    href: https://developers.cloudflare.com/workers/configuration/
  - label: Despliegue gradual
    wert: cargar la versión, crear el despliegue, distribuir gradualmente el tráfico y volver a una versión estable
    href: https://developers.cloudflare.com/workers/versions-and-deployments/gradual-deployments/
  - label: Señales operativas
    wert: resultado de invocación, estado, excepción, CPU y Wall Time, subrequests, logs, traces e ID de versión
    href: https://developers.cloudflare.com/workers/observability/logs/workers-logs/
  - label: Objetos de recuperación
    wert: artefacto de código, configuración, Compatibility Date, bindings, secretos, exportaciones de datos, ciclo de vida de DO y rollback probado
    href: https://developers.cloudflare.com/workers/versions-and-deployments/
werbung:
  - newsletter
ctaThemen:
  - cloudflare-workers
translationSourceHash: 4c5445ed6d4ead36638b1de03ed9d67a4f82304b58f28d83a9c72e29e760d065
translationModel: gpt-5.6-terra
translatedAt: 2026-10-12T11:08:37.683Z
translationReview: automatic
---

# Cloudflare Workers: Isolates, Bindings y operación en el Edge

Cloudflare Workers es una plataforma de cómputo orientada a eventos dentro de la red de Cloudflare. Un Worker puede responder o reenviar solicitudes HTTP, ejecutar tareas programadas, consumir mensajes de colas, procesar correos electrónicos entrantes y servir como servicio interno para otros Workers. El operador no administra ningún proceso listener ni una única VM. Administra código, entrypoints, rutas, compatibilidad del runtime, permisos para recursos de plataforma, versiones y datos operativos.

La palabra «Edge» describe solo una posible ubicación de ejecución, no la arquitectura completa. Un Worker HTTP puede ejecutarse cerca del usuario; Smart Placement puede acercar el cómputo a un backend; los Durable Objects tienen una ubicación de estado única; D1, KV, R2 y Queues tienen cada uno sus propios modelos de replicación y consistencia. Por ello, para los administradores la característica decisiva no es «serverless», sino la separación entre ruta de datos, Control Plane, servicios de estado y posibles puntos de fallo.

La explicación sigue una solicitud desde el endpoint público hasta el runtime de Workers y, después, hacia los bindings, el almacenamiento y los upstreams. A continuación, sitúa el despliegue, la seguridad, la observabilidad y la recuperación desde la perspectiva de la operación de plataforma.

## Arquitectura: ruta de datos, runtime y Control Plane

Un sistema de Workers consta de varios niveles:

1. **Ingreso y enrutamiento:** Cloudflare recibe una solicitud mediante DNS, TLS y HTTP, y asigna un patrón de host o ruta a un Worker o activo estático.
2. **Distribución de eventos:** La plataforma genera un evento Fetch, Scheduled, Queue, E-Mail, Alarm o Tail e invoca el entrypoint correspondiente.
3. **Runtime:** `workerd` proporciona V8 Isolates, API web, compatibilidad con Node.js y límites de ejecución.
4. **Bindings:** El objeto `env` concede al código capacidades para KV, D1, R2, Durable Objects, Queues, secretos, activos, otros Workers y otros servicios de plataforma.
5. **Estado y upstreams:** Los datos residen fuera del Isolate efímero o en almacenes controlados de Durable Objects; las llamadas salientes usan Fetch, Service Bindings o API de sockets compatibles.
6. **Control Plane:** Wrangler, Dashboard y API administran configuración, relaciones entre recursos, secretos, versiones, despliegues y observabilidad.

Cloudflare describe los Isolates, el tiempo de cómputo por solicitud y la ejecución distribuida como las tres diferencias más importantes respecto a los runtimes de servidor clásicos ([How Workers works](https://developers.cloudflare.com/workers/reference/how-workers-works/)). La referencia de la plataforma separa el propio Worker de los productos de almacenamiento y Developer Platform conectados ([Cloudflare Workers documentation](https://developers.cloudflare.com/workers/)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-cloudflare-workers.svg?v=20260813" title="Interaktive Infografik: Cloudflare-Workers-Aufruf von DNS, TLS und Routing über Event Dispatch, workerd und V8-Isolate bis zu Bindings, Storage, Upstreams, Observability, Versionen und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-cloudflare-workers.svg?v=20260813">Abrir el gráfico interactivo directamente</a>.
</iframe>

## Ruta de solicitud y ubicación de ejecución

En una solicitud HTTP, la configuración de Cloudflare decide primero si un hostname de `workers.dev`, una ruta o un Custom Domain activa el Worker. Una ruta se sitúa delante de un origen existente y puede modificar, responder o reenviar solicitudes. Un Custom Domain conecta un Worker directamente con un hostname. Los activos estáticos pueden evaluarse antes o después del Worker; `run_worker_first` modifica este orden ([Wrangler configuration – routes](https://developers.cloudflare.com/workers/wrangler/configuration/), [Static Assets – configuration and bindings](https://developers.cloudflare.com/workers/static-assets/binding/)).

La ruta externa puede modelarse como `Client → DNS → Cloudflare Edge → TLS/HTTP → Route → Worker/Asset → Binding oder Origin`. La conexión con el origen es un nuevo contexto de transporte. Por tanto, el certificado de cliente, el encabezado `Authorization`, la clave de caché, el encabezado Host y la IP de origen deben tratarse conscientemente en cada límite. En cambio, un Service Binding evita la resolución pública de nombres y proporciona una ruta explícitamente autorizada de Worker a Worker ([Service bindings](https://developers.cloudflare.com/workers/runtime-apis/bindings/service-bindings/)).

Smart Placement puede ejecutar un Worker más cerca de sus backends, en vez de priorizar únicamente la proximidad al usuario. Esto resulta útil para llamadas intensivas a bases de datos, pero puede alargar innecesariamente la ruta para activos o transformaciones puras de Edge. Por ello, Cloudflare documenta recomendaciones distintas para Assets-first y Worker-first ([Placement](https://developers.cloudflare.com/workers/configuration/placement/)).

## workerd y V8 Isolates

`workerd` es el runtime de JavaScript/WebAssembly orientado a servidores detrás de Workers. V8 proporciona Isolates como contextos de ejecución separados. Muchos Isolates pueden compartir un proceso sin que cada aplicación requiera su propio proceso JavaScript o contenedor. Las API web las proporciona en gran medida el runtime de forma nativa ([Introducing workerd](https://blog.cloudflare.com/workerd-open-source-workers-runtime/), [Workers security model](https://developers.cloudflare.com/workers/reference/security-model/)).

Un Isolate no es una máquina persistente. Puede reutilizarse, ocuparse en paralelo con varias solicitudes o eliminarse. Dos solicitudes consecutivas no necesariamente alcanzan la misma instancia ni la misma ubicación. Los objetos globales son adecuados para estructuras auxiliares inmutables y costosas de crear o cachés oportunistas, pero no como fuente de verdad. Toda corrección que dependa de una variable global entre solicitudes es un error de arquitectura.

El aislamiento de V8 no sustituye todos los controles de seguridad. Cloudflare añade protección del runtime y de procesos, límites de recursos y defensas específicas contra Spectre. El operador sigue siendo responsable de la validación de entradas, autorización, alcance de secretos, listas permitidas de destinos, límites de salida y clasificación de datos. El límite del Isolate no protege frente a una aplicación que lee o escribe datos incorrectos con sus bindings legítimos.

## Modelo de eventos y handlers

Workers implementa entrypoints para diferentes tipos de eventos. El catálogo general de handlers incluye Fetch, Scheduled, Queue, E-Mail, Alarm y Tail ([Handlers](https://developers.cloudflare.com/workers/runtime-apis/handlers/)).

| Handler | Activador | Límite de retorno y error | Pregunta administrativa típica |
|---|---|---|---|
| `fetch()` | Solicitud HTTP o llamada de servicio | `Response`, stream o excepción | ¿Qué ruta y versión procesaron la solicitud? |
| `scheduled()` | Cron Trigger | Promise/Invocation Outcome | ¿Se activó y terminó la ejecución programada? |
| `queue()` | Lote de mensajes | Ack, Retry o comportamiento Dead Letter | ¿Qué mensaje es idempotente, reintentable o poison? |
| `email()` | Email Routing | Entregar, rechazar, reenviar | ¿Qué límite de envelope y política se aplica? |
| `alarm()` | Alarma de Durable Object | ejecución local al objeto | ¿Qué objeto posee la alarma y el estado? |
| `tail()` | Evento de trace de un Worker productor | exportación asíncrona | ¿Puede fallar la observabilidad o volverse recursiva? |

El handler Fetch recibe `Request`, `env` y `ctx`, y devuelve una `Response` de API web ([Fetch handler](https://developers.cloudflare.com/workers/runtime-apis/handlers/fetch/)). `ctx.waitUntil()` registra trabajo que puede continuar después de enviar la respuesta; sigue sujeto a los límites documentados de ejecución y errores, y no es un servidor de trabajos de larga duración ([Context – waitUntil](https://developers.cloudflare.com/workers/runtime-apis/context/)). Para procesos persistentes y repetibles, Queues o Workflows son más adecuados que una cadena de Background Promises sin supervisión.

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

El ejemplo utiliza un Service Binding en lugar de una URL configurable libremente. Con ello, la accesibilidad y autorización pasan a la configuración de despliegue, mientras el código solo ve una capability de `Fetcher`. Las API del runtime están orientadas a la web e incluyen, entre otras, Fetch, Streams, Web Crypto, WebSockets, HTMLRewriter y TCP Sockets ([Runtime APIs](https://developers.cloudflare.com/workers/runtime-apis/)).

El código no se ejecuta en un entorno local de Node elegido libremente, sino frente a un contrato de runtime versionado. La Compatibility Date determina qué cambios de comportamiento se aplican a un despliegue.

## Compatibility Date y contrato de runtime

Una Compatibility Date no es una nota de versión ni una afirmación sobre el «estado actual» del artículo. Forma parte del contrato de runtime de un despliegue concreto de Worker. Los cambios que podrían romper el comportamiento existente se activan por fecha o flag. Una fecha anterior mantiene el comportamiento compatible; una fecha más reciente debe probarse y desplegarse como una actualización de dependencia ([Compatibility flags](https://developers.cloudflare.com/workers/configuration/compatibility-flags/)).

Por ello, los administradores tratan conjuntamente estos campos:

- commit de código y artefacto empaquetado;
- Compatibility Date y flags explícitos;
- cadena de herramientas de Wrangler y compilación;
- definiciones de bindings, rutas y placement;
- ID de versión y peso de despliegue;
- migraciones de datos o de Durable Objects.

La compatibilidad con Node.js es opt-in e implementa solo una parte documentada de las API de Node. Algunos módulos están completos, otros son parciales o solo están disponibles como stub de importación. Por tanto, `nodejs_compat` no debe equipararse con «runtime completo de Node» ([Node.js compatibility](https://developers.cloudflare.com/workers/runtime-apis/nodejs/)).

## Lenguajes y stack tecnológico

Cloudflare documenta JavaScript, TypeScript, Python y Rust como rutas de lenguaje de primera clase; WebAssembly abre otras lenguas fuente ([Languages](https://developers.cloudflare.com/workers/languages/)). El stack tecnológico resultante distingue entre desarrollo y ejecución:

| Nivel | Tecnología típica | Relevancia operativa |
|---|---|---|
| Código fuente | TypeScript/JavaScript, Python, Rust | toolchain de lenguaje, dependencias, pruebas |
| Build | Wrangler/esbuild o adaptador de framework | bundle, Source Maps, externals, builds deterministas |
| Runtime | workerd, V8, API web y Node opcionales | Compatibility Date, CPU/memoria, Event Loop |
| Capability | bindings de `env` | mínimo privilegio, ID de recurso, separación de entornos |
| Estado | Cache, KV, D1, R2, Durable Objects, Queues | consistencia, ubicación de datos, backup, recuperación |
| Ingreso | Route, Custom Domain, workers.dev, trigger | DNS, TLS, orden, fail-open/closed |
| Control Plane | Wrangler, API, Dashboard, CI/CD | autenticación, versionado, rollout, auditoría |

Rust se integra mediante `workers-rs` y WebAssembly; el módulo Wasm importado sigue siendo parte del artefacto del Worker ([Rust language support](https://developers.cloudflare.com/workers/languages/rust/)). Los Python Workers utilizan sus propias clases de entrypoint y una integración de runtime, no un proceso CPython administrable libremente.

## Configuración y Wrangler

La configuración de Wrangler es la conexión declarativa entre el código y la plataforma. Define como mínimo nombre, entrypoint, Compatibility Date, rutas y bindings. Los Named Environments pueden tener valores diferentes; los bindings deben revisarse conscientemente por entorno. Cloudflare documenta los campos y la herencia en el esquema de Wrangler ([Workers configuration](https://developers.cloudflare.com/workers/configuration/), [Wrangler configuration reference](https://developers.cloudflare.com/workers/wrangler/configuration/)).

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

[`npx`](https://docs.npmjs.com/cli/commands/npx) inicia la CLI fijada localmente en el proyecto; los [`Wrangler Commands`](https://developers.cloudflare.com/workers/wrangler/commands/) documentan los comandos de autenticación, generación de tipos, despliegue, versiones y logs. `wrangler types` genera tipos a partir de la configuración real de bindings. `deploy --dry-run` comprueba el bundle y la configuración, pero no demuestra que los recursos remotos funcionen ni que la autorización sea correcta.

Una vez aclarados el runtime y el build, se plantea la cuestión de acceso. Un Worker alcanza bases de datos, colas, secretos y otros Workers mediante bindings declarados en vez de credenciales distribuidas libremente.

## Bindings como modelo de capabilities

Un binding es a la vez permiso y API de runtime. Por ejemplo, el Worker recibe `env.ARCHIVE` como bucket R2, `env.DB` como base de datos D1 o `env.AUTH` como servicio interno. La credencial de plataforma subyacente no se expone al código de aplicación como una API key reutilizable ([Bindings](https://developers.cloudflare.com/workers/runtime-apis/bindings/)).

El modelo de capabilities desplaza la pregunta administrativa central de «¿Qué secreto hay en la variable de entorno?» a «¿Qué recurso está vinculado, bajo qué nombre y en qué entorno, a qué versión?». Un ID de bucket o una versión de Service Binding erróneos pueden causar pérdida de datos o acceso entre entornos incluso con código idéntico.

Los bindings deben separarse por tarea y entorno. Los Workers de producción y prueba no comparten buckets, colas ni bases de datos con escritura, salvo que ello forme explícitamente parte de una prueba de integración controlada. Nombres genéricos como `DB` son suficientes en el código si el despliegue vincula el recurso real de forma inequívoca y verificable.

### Secretos y variables

Las variables normales son configuración, no secretos. Los secretos son bindings de texto cifrados y se administran fuera del código fuente. Los nombres de secretos requeridos pueden declararse sin escribir sus valores en el archivo de configuración ([Environment variables](https://developers.cloudflare.com/workers/configuration/environment-variables/), [Secrets](https://developers.cloudflare.com/workers/configuration/secrets/)).

Un cambio de secreto es un cambio de estado de la aplicación. Los clientes derivados globalmente de `env` pueden conservar un valor antiguo en Isolates reutilizados; Cloudflare recomienda crear tales derivados por solicitud. Por tanto, la rotación incluye establecer el secreto, comprobar el despliegue/binding, validez paralela si el sistema de destino lo requiere, telemetría y eliminación controlada.

## Service Bindings y arquitectura de Worker a Worker

Los Service Bindings conectan Workers explícitamente, sin necesitar un endpoint HTTP públicamente direccionable. El servicio invocado puede exponerse mediante Fetch o RPC. Esto reduce la configuración de red y credenciales, pero no elimina los problemas de versiones y contratos. Con rollouts separados, caller y callee pueden usar estados de API diferentes ([Service bindings](https://developers.cloudflare.com/workers/runtime-apis/bindings/service-bindings/), [Gradual deployments](https://developers.cloudflare.com/workers/versions-and-deployments/gradual-deployments/)).

Los administradores documentan para cada binding:

- caller y servicio de destino;
- entorno y versión de destino o estrategia de rollout;
- accesibilidad pública del Worker de destino;
- semántica de timeout, errores y reintentos;
- propagación de Request ID y correlación de traces;
- compatibilidad contractual en despliegues independientes.

Una cadena de muchos Workers distribuye una solicitud entre más posibles puntos de fallo. Menos superficie de ataque pública es valioso; sin embargo, una división demasiado fina puede aumentar la profundidad de subrequests, el esfuerzo de depuración y el version skew.

## Servicios de estado: modelo antes que nombre de producto

El heap del Isolate es efímero. El estado persistente reside en servicios vinculados cuya semántica difiere notablemente. La vista general de almacenamiento de Cloudflare clasifica los productos según patrón de acceso y consistencia ([Choosing a data or storage product](https://developers.cloudflare.com/workers/platform/storage-options/)).

| Servicio | Modelo | Límite de consistencia/ubicación | Adecuado para | Pregunta administrativa crítica |
|---|---|---|---|---|
| Cache API | caché de respuestas HTTP | local al centro de datos, efímero | respuestas repetibles | ¿La clave de caché es completa y segura? |
| Workers KV | almacén global de clave-valor legible | eventual consistent, Read Cache | configuración, datos con muchas lecturas | ¿El flujo tolera lecturas antiguas o negativas? |
| D1 | SQL gestionado basado en SQLite | semántica relacionada con la base de datos y sesión | tablas de aplicación relacionales | ¿Qué garantía de réplica/sesión requiere la lectura? |
| Durable Objects | instancia de objeto única más almacenamiento | fuertemente consistente y serializable por objeto | coordinación, locks, sesiones, WebSockets | ¿Se eligió correctamente la clave de partición? |
| R2 | almacenamiento de objetos compatible con S3 | fuertemente consistente por objeto | blobs, archivos, payloads grandes | ¿Cómo se versionan conjuntamente objeto, metadatos e índice? |
| Queues | mensajes asíncronos | procesamiento orientado a al menos una vez y política de reintento | desacoplamiento, nivelación de carga | ¿El consumidor es idempotente y existe una DLQ? |

### Cache API

La Cache API opera localmente en el centro de datos donde el Worker procesa la solicitud. Por tanto, `cache.put()` no es replicación global ni escritura persistente en base de datos. `caches.default` comparte el contexto de caché estándar; las cachés nombradas crean namespaces, pero tampoco consistencia global ([How the Cache works](https://developers.cloudflare.com/workers/reference/how-the-cache-works/)).

Las respuestas personalizadas solo deben almacenarse en caché con una clave que refleje todas las características relevantes de identidad y variante. `Authorization`, cookies, idioma, codificación y pertenencia a tenant son límites típicos de filtración. La purga de caché y la validación del origen son parte de la recuperación, no solo los TTL.

### Workers KV

KV replica datos mediante stores centrales y cachés Edge. Los cambios pueden hacerse visibles con retraso en otras ubicaciones; incluso «no encontrado» puede estar en caché. Cloudflare indica retrasos de hasta 60 segundos o más para ubicaciones remotas y no recomienda KV para transacciones atómicas de lectura-modificación-escritura ([How KV works](https://developers.cloudflare.com/kv/concepts/how-kv-works/)).

KV es adecuado para configuración de lectura predominante, listas Allow/Deny o cachés cuando la obsolescencia se tolera explícitamente. Los límites de tasa globales, secuencias únicas o revocación inmediata de credenciales no deben colocarse en KV sin coordinación adicional.

### Durable Objects

Un Durable Object combina un ID de objeto único a nivel mundial con coordinación serial y almacenamiento privado. Las solicitudes para el mismo ID se enrutan a la misma instancia lógica. Los namespaces nuevos utilizan la ruta de almacenamiento SQLite; esta ofrece almacenamiento transaccional y fuertemente consistente, así como funciones de recuperación Point-in-Time ([Durable Objects overview](https://developers.cloudflare.com/durable-objects/), [SQLite-backed Durable Object Storage](https://developers.cloudflare.com/durable-objects/api/sqlite-storage-api/)).

Un Durable Object tampoco es un servidor en ejecución permanente. El estado en memoria puede desaparecer durante la hibernación o eviction y debe reconstruirse desde Storage cuando sea necesario. No hay un Shutdown Hook fiable ([Durable Object lifecycle](https://developers.cloudflare.com/durable-objects/concepts/durable-object-lifecycle/)). Los cambios en clases y asignación de almacenamiento son operaciones de ciclo de vida; se tratan como migraciones de datos ([Durable Object class lifecycle](https://developers.cloudflare.com/durable-objects/reference/durable-objects-migrations/)).

### D1, R2 y Queues

D1 proporciona SQL para datos relacionales. R2 almacena objetos grandes no estructurados y expone una API compatible con S3 externamente, o una API de binding dentro de Workers. Queues desacopla productores y consumidores. Estos servicios no se sustituyen entre sí: un objeto R2 puede transportar el payload, D1 el índice, una Queue iniciar el procesamiento y un Durable Object coordinar cambios concurrentes.

El commit conjunto de estos cuatro estados no es automáticamente atómico. Los administradores planifican Idempotency Keys, patrones Outbox/Inbox, reconciliación y procesamiento de Dead Letter. La documentación y límites de cada servicio para [D1](https://developers.cloudflare.com/d1/), [R2](https://developers.cloudflare.com/r2/) y [Queues](https://developers.cloudflare.com/queues/) forman parte del runbook.

## Activos estáticos y Workers full-stack

Un Worker puede entregar activos estáticos junto con código. El Assets Binding permite al código recuperar explícitamente un activo. La opción de enrutamiento determina si los activos existentes omiten el Worker o si el Worker se ejecuta primero ([Static Assets](https://developers.cloudflare.com/workers/static-assets/)).

El orden es una decisión de seguridad. Con Assets-first, una ruta de archivo existente por casualidad no debe omitir la autenticación. Con Worker-first, cada solicitud de activo incrementa la carga de cómputo y dependencias. Los adaptadores de frameworks ocultan parcialmente este orden; el artefacto de despliegue construido y la configuración de Wrangler siguen siendo la verdad determinante.

## Perspectiva de red y protocolos

Workers se sitúa en la ruta de aplicación por encima de DNS, TLS y HTTP. La plataforma termina el transporte externo; el Worker procesa solicitudes de API web. Un `fetch()` saliente genera una nueva llamada HTTP. TCP Sockets permite protocolos de salida seleccionados, pero no convierte el Worker en un servidor TCP accesible de forma general ([Workers protocols](https://developers.cloudflare.com/workers/reference/protocols/), [TCP sockets](https://developers.cloudflare.com/workers/runtime-apis/tcp-sockets/)).

Para administradores de mensajería son importantes tres límites:

- Un handler de correo electrónico se activa mediante Cloudflare Email Routing; no escucha por sí mismo en [SMTP](/kb/smtp).
- Un Worker que llama a una API de correo debe tratar OAuth Audience, ciclo de vida del token y reintentos como cualquier otro cliente de API.
- Los errores de DNS, TLS y HTTP se producen antes del entrypoint; los errores de binding, autenticación y aplicación, después. Un timeout de `fetch()` no indica sin correlación qué fase falló.

## Límites de seguridad y confianza

El modelo de seguridad tiene al menos cinco límites de confianza separados:

1. **Solicitud pública:** encabezados, tamaño de body, método e identidad arbitrarios.
2. **Configuración de Cloudflare:** Zone, Route, WAF, Access, certificados y fail-open/closed.
3. **Artefacto del Worker:** código, dependencias, Source Maps, Compatibility Date y cadena de suministro.
4. **Bindings:** permisos concretos sobre recursos, secretos y rutas internas de servicio.
5. **Upstreams y datos almacenados:** autorización, consistencia y recuperación propias.

El mínimo privilegio se logra principalmente mediante superficies de bindings pequeñas y entornos separados. Un Worker con acceso de escritura a todos los buckets y bases de datos sigue siendo un principal altamente privilegiado, aunque no posea claves de acceso visibles. La documentación de seguridad de Workers explica el aislamiento del runtime; no sustituye una arquitectura de seguridad de aplicaciones ([Security model](https://developers.cloudflare.com/workers/reference/security-model/)).

Los controles de cadena de suministro incluyen versiones de paquetes fijadas, lockfile, build reproducible, escaneo de dependencias, escaneo de secretos, permisos mínimos de carga y entornos de despliegue protegidos. El bundle generado se revisa antes de cargarlo; revisar solo el código fuente no basta si el bundler o framework integra módulos adicionales.

La proximidad al Edge no elimina las restricciones de recursos. El tiempo de CPU, subrequests, memoria y límites de plataforma deben considerarse ya en el diseño y el modelo de carga.

## Límites como parámetros de arquitectura

Workers limita, entre otros, el tiempo de CPU, memoria, tiempo de inicio, subrequests, tamaño del bundle, volumen de logs, número de rutas y activos estáticos. Los valores difieren según el plan y el tipo de invocación, y pueden cambiar. Por ello, un artículo estático no debe incluir una tabla de límites copiada como supuesta verdad atemporal; la página oficial [Workers Limits](https://developers.cloudflare.com/workers/platform/limits/) es la fuente operativa.

Los administradores distinguen:

- **CPU Time:** tiempo de cómputo activo; esperar I/O no cuenta igual que CPU.
- **Wall Time:** duración transcurrida del evento; las reglas difieren entre Fetch, Cron, Queue y Durable Object.
- **Startup Time:** evaluación de módulos e inicialización antes del handler.
- **Subrequests:** operaciones Fetch y de plataforma por invocación.
- **Memory:** heap, streams, buffers y estado de bibliotecas del Isolate.
- **Log Budget:** payloads demasiado grandes o secretos son un problema de costes y protección de datos.

Un error `1102` indica que se superaron límites de recursos; `10021` puede aparecer cuando la inicialización de Startup es demasiado costosa. El número de error es un punto de partida, no una causa raíz. Deben analizarse conjuntamente perfil de CPU, tipo de invocación, versión, clase de entrada y grafo de subrequests.

## Desarrollo local y pruebas

El desarrollo local ejecuta código con `workerd` mediante Miniflare. Los bindings se simulan localmente de forma predeterminada; bindings remotos individuales pueden conectarse deliberadamente con recursos reales de plataforma. `wrangler dev --remote` completamente ejecutado de forma remota sigue disponible para casos específicos de red, pero ya no es la ruta estándar ([Local development](https://developers.cloudflare.com/workers/local-development/), [Bindings per development mode](https://developers.cloudflare.com/workers/local-development/bindings-per-env/)).

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

[`Invoke-WebRequest`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest) y [`curl`](https://curl.se/docs/manpage.html) comprueban el estado y los encabezados de la ruta HTTP local. La integración de Workers con Vitest ejecuta pruebas dentro de `workerd` y proporciona ayudantes para bindings, solicitudes y Durable Objects ([Vitest integration](https://developers.cloudflare.com/workers/testing/vitest-integration/), [Workers test APIs](https://developers.cloudflare.com/workers/testing/vitest-integration/test-apis/)).

Una prueba unitaria correcta no prueba una ruta, permiso remoto o migración de datos correctos. La pirámide de pruebas comprende lógica pura, integración de runtime, semántica de binding, ruta de staging, prueba smoke controlada de producción y ejercicio de recuperación.

## Despliegue, versión y rollout

Una **versión** es un objeto inmutable de código/configuración. Un **despliegue** distribuye tráfico entre una o más versiones. Los Gradual Deployments desplazan los pesos de forma gradual y permiten observar y volver a una versión estable ([Versions and deployments](https://developers.cloudflare.com/workers/versions-and-deployments/), [Gradual deployments](https://developers.cloudflare.com/workers/versions-and-deployments/gradual-deployments/)).

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

Antes de un rollout se guarda la asignación `Git commit → Bundlehash → Worker-Version → Deploymentgewicht`. Cuando existen varios Service Bindings, el version skew forma parte de las pruebas. Los Durable Objects requieren reglas especiales de migración y rollout, ya que para un ID de objeto no pueden estar activos simultáneamente estados de clase arbitrarios.

El rollback revierte código y configuración, pero no automáticamente los datos externos. Un Worker nuevo puede haber escrito datos en un formato que el antiguo no entiende. Los cambios de esquema utilizan Expand/Contract, compatibilidad hacia adelante o una reversión de datos probada por separado. «Rollback posible» solo se demuestra cuando se han comprobado conjuntamente código, bindings y datos.

## Observabilidad y evidencia operativa

Workers Logs recopila logs de invocación, logs propios, errores y excepciones no controladas. Tail Workers o Logpush pueden exportar eventos; los destinos OpenTelemetry ofrecen una ruta de exportación más directa para logs y traces ([Workers Logs](https://developers.cloudflare.com/workers/observability/logs/workers-logs/), [Tail Workers](https://developers.cloudflare.com/workers/observability/logs/tail-workers/), [Traces](https://developers.cloudflare.com/workers/observability/traces/)).

Cada invocación debería permitir correlacionar como mínimo:

- nombre del Worker, ID de versión y entorno;
- tipo de evento y ruta;
- ID de solicitud/mensaje/correlación;
- resultado, estado HTTP o Ack/Retry;
- CPU y Wall Time;
- destino y latencia de subrequest sin secretos;
- clase de binding/recurso, no contenido completo sensible;
- Idempotency Key funcional en procesamiento asíncrono.

`console.log()` no es un almacén de evidencia ilimitado. Los límites de logs, sampling y redaction afectan a la visibilidad. No se registran credenciales, Authorization Header, cookies, contenido completo de correos ni payloads personales. El monitoring también supervisa la ruta de exportación: un Tail Worker defectuoso no debe imposibilitar el diagnóstico del error del productor.

El diagnóstico sigue la ruta de publicación: Route y DNS, versión activa, runtime y bindings, dependencias salientes y, finalmente, logs y traces.

## Diagnóstico administrativo por fases

Los errores de Workers se separan eficazmente a lo largo de la ruta de ejecución:

1. **DNS:** ¿El host esperado resuelve a Cloudflare y la Zone está activa?
2. **TLS/HTTP:** comprobar certificado, SNI, protocolo y estado.
3. **Route/Asset:** ¿Host/ruta alcanza el Worker, un activo o el origen?
4. **Versión:** ¿Qué versión y qué peso de despliegue procesaron la solicitud?
5. **Runtime:** comprobar Startup, CPU, Memory, excepción y salida del handler.
6. **Binding:** ¿Existe el recurso en este Environment y posee la capability esperada?
7. **Upstream/Storage:** investigar timeout, autenticación, consistencia, estado de datos y reintento.
8. **Recuperación:** confirmar versión estable, compatibilidad de datos y reconciliación.

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

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname), [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection), [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility), [`nc`](https://man.openbsd.org/nc), [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) y [`curl`](https://curl.se/docs/manpage.html) separan DNS, TCP, TLS y HTTP antes del runtime. [DNS](/kb/dns), [TCP](/kb/tcp), [TLS](/kb/tls), [APIs](/kb/apis) y [Troubleshooting](/kb/troubleshooting) tratan estos niveles en profundidad.

### Patrones de fallo típicos

| Síntoma | Fase probable | Evidencia | Error de razonamiento frecuente |
|---|---|---|---|
| Respuesta de origen en lugar de respuesta de Worker | Route/Asset/Fail-open | Route, host, ruta, marcador de respuesta | «Despliegue correcto» significa «ruta activa» |
| 1101 o excepción | Aplicación/Binding | Workers Logs, versión, stack | volver a desplegar sin comprobar la clase de entrada |
| 1102 | Límite de recursos | CPU/Wall Time, perfil, tipo de invocación | equiparar espera de I/O y CPU |
| valor KV antiguo esporádico | Consistencia KV | clave, ubicación, TTL de caché, hora de escritura | tratar KV como transacción global |
| local correcto, error de binding remoto | Environment/Capability | ID de recurso concreto y nombre de binding | la simulación prueba el estado IAM/recurso |
| solo falla parte del tráfico | Gradual Deployment | ID de versión y peso | agregar métricas sin dimensión de versión |
| el rollback corrige código, no datos | Límite de esquema/estado | formato escrito, migración, reconciliación | el despliegue es una recuperación completa |

## Backup y Disaster Recovery

Serverless no elimina la necesidad de backup. Desplaza los objetos que deben protegerse:

- repositorio Git, lockfile y definición de build reproducible;
- configuración de Wrangler, rutas, Compatibility Date y flags;
- inventario de todos los bindings y recursos de destino;
- valores de secretos o su fuente externa de verdad y proceso de rotación;
- exportaciones de datos y procedimientos de restauración para D1, R2, KV y sistemas externos;
- clases de Durable Object, estado de ciclo de vida/migración y, en su caso, PITR;
- estado de Queue/DLQ, Idempotency Keys y reconciliación;
- historial de versión/despliegue y una ruta de rollback probada.

No todos los productos tienen la misma semántica de exportación y restauración. Un backup de objetos R2 no protege el índice D1; una restauración D1 no recupera un mensaje de Queue ya confirmado; un Worker revertido puede interpretar erróneamente nuevos formatos de objetos. [Backup y Disaster Recovery](/kb/backup-dr) debe definir por ello un punto consistente de aplicación, no solo copias individuales de productos.

La recuperación se prueba en escenarios: secreto incorrecto, ruta eliminada, versión defectuosa, esquema D1 incompatible, objeto R2 perdido, mensaje poison en Queue, clase Durable Object defectuosa y upstream inaccesible. RTO y RPO se establecen para cada servicio de estado y para la aplicación compuesta.

## Historia técnica

Cloudflare presentó Workers en 2017 como ejecución programable en la red global. El punto de partida era el límite de latencia de los centros de datos centrales y la idea de ejecutar código cerca de la ruta de datos ([Code Everywhere: Why We Built Cloudflare Workers](https://blog.cloudflare.com/code-everywhere-cloudflare-workers/)).

La arquitectura apostó pronto por V8 Isolates en lugar de un contenedor o proceso por función. En 2018, Cloudflare describió las diferencias económicas y técnicas de este modelo, pero también su restricción a lenguajes compatibles con JavaScript o WebAssembly ([Cloud Computing without Containers](https://blog.cloudflare.com/cloud-computing-without-containers/)).

Workers KV añadió estado global legible y eventual-consistent. Durable Objects se anunciaron en 2020 como modelo opuesto para estados de objeto coordinados y fuertemente consistentes ([Introducing Workers Durable Objects](https://blog.cloudflare.com/introducing-workers-durable-objects/)). Con ello, Workers evolucionó de una mera transformación de solicitudes a una plataforma de aplicaciones con varios modelos de consistencia explícitos.

En 2022, Cloudflare publicó `workerd` bajo Apache 2.0. El runtime abierto comparte código con el sistema de producción y mejoró la precisión del desarrollo local; aun así, no es toda la plataforma Cloudflare con su routing, orquestación y operación de seguridad ([Introducing workerd](https://blog.cloudflare.com/workerd-open-source-workers-runtime/), [workerd repository](https://github.com/cloudflare/workerd)).

Las extensiones posteriores aportaron entrypoints de módulos ES, Compatibility Dates, Service/RPC Bindings, D1, R2, Queues, Workflows, Python, compatibilidad Node ampliada, objetos de versión y despliegues graduales. Esta historia explica el núcleo actual: Workers no es JavaScript de navegador sobre un CDN ni un servidor Linux arbitrario, sino un runtime orientado a eventos y basado en capabilities, con productos separados de datos y Control Plane.

## Lista de verificación para administradores

Antes de la operación en producción, se acreditan como mínimo los siguientes puntos:

- Route, Custom Domain, orden de activos y fail-open/closed están documentados.
- Compatibility Date, flags, versión de Wrangler, bundle y Source Maps son reproducibles.
- Cada binding tiene owner, Environment, Resource ID, permisos y ruta de recuperación.
- La semántica de Cache, KV, D1, R2, Durable Object y Queue no se mezcla.
- Los secretos están fuera del repositorio, son rotables y no visibles en logs.
- Timeouts, reintentos, idempotencia y comportamiento Dead Letter se definen por tipo de evento.
- Los logs y traces contienen versión y Correlation ID, pero no payloads sensibles.
- Gradual Deployment y rollback consideran las versiones de Service Binding y los formatos de datos.
- Los límites se supervisan desde la página oficial de la plataforma, no desde una tabla copiada.
- Se han probado backup, restauración y reconciliación para la aplicación compuesta.

## Fuentes

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
