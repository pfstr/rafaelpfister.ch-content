---
title: "Claude y Claude Code: arquitectura, API y operación segura de agentes"
blatt: "claude"
description: "Claude y Claude Code para administradores de plataformas, seguridad y automatización: límites de modelos y API, Messages sin estado, bloques de contenido, streaming, Tool Use y MCP, tiempo de ejecución local de agentes, workspaces y claves, límites de tasa, caché, lotes, observabilidad, protección de datos, prompt injection y recuperación."
fakten:
  - label: Función del sistema
    wert: Claude es una familia de modelos generativos de lenguaje; las aplicaciones proporcionan contexto y procesan bloques de contenido generados probabilísticamente
    href: https://platform.claude.com/docs/en/about-claude/models/overview
  - label: Proveedor
    wert: Anthropic desarrolla los modelos, la API directa de Claude, las aplicaciones Claude y Claude Code
    href: https://www.anthropic.com/news/introducing-claude
  - label: API principal
    wert: POST /v1/messages procesa mensajes estructurados; el cliente vuelve a enviar el estado de la conversación
    href: https://platform.claude.com/docs/en/api/messages/create
  - label: Transporte
    wert: HTTPS/JSON; el streaming usa Server-Sent Events sin cambiar el contrato semántico de Messages
    href: https://platform.claude.com/docs/en/build-with-claude/streaming
  - label: Salida
    wert: un mensaje contiene bloques de contenido tipados, métricas de uso y stop_reason en lugar de texto libre garantizado
    href: https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons
  - label: Vinculación de modelo
    wert: Los ID de modelo son contratos fijados; las capacidades y los límites se determinan mediante la Models API y la documentación
    href: https://platform.claude.com/docs/en/about-claude/models/model-ids-and-versions
  - label: Contrato de herramientas
    wert: Claude genera tool_use con argumentos JSON; el código cliente ejecuta y devuelve tool_result
    href: https://platform.claude.com/docs/en/agents-and-tools/tool-use/how-tool-use-works
  - label: MCP
    wert: protocolo abierto cliente-servidor para herramientas, recursos y contexto; cada servidor amplía los derechos sobre datos y acciones
    href: https://docs.anthropic.com/en/docs/mcp
  - label: Claude Code
    wert: agente local para repositorios, archivos, shell y herramientas; la inferencia de API sigue siendo un servicio externo
    href: https://docs.anthropic.com/en/docs/claude-code/getting-started
  - label: Modelo de inquilinos
    wert: Organización → Workspace → miembros, claves, límites y recursos vinculados al workspace
    href: https://platform.claude.com/docs/en/manage-claude/workspaces
  - label: Capacidad
    wert: Spend Limits y límites de solicitudes, tokens de entrada y salida; 429 y retry-after controlan el backoff
    href: https://platform.claude.com/docs/en/api/rate-limits
  - label: Retención de datos
    wert: La retención depende del producto, workspace, función, contrato y clasificación de seguridad, y se verifica antes del uso
    href: https://privacy.claude.com/en/articles/7996866-how-long-do-you-store-my-organization-s-data
werbung:
  - newsletter
ctaThemen:
  - claude
translationSourceHash: a5189670b409d16f452c042199e3237347d4a9a1483be7c38ebbc07a8d640c5f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-04T10:05:48.852Z
translationReview: automatic
---

# Claude y Claude Code: arquitectura, API y operación segura de agentes

Claude designa una familia de grandes modelos generativos de lenguaje de Anthropic, así como varios productos que utilizan estos modelos. La distinción administrativa más importante es: **el modelo, la API, la aplicación y el tiempo de ejecución del agente no son el mismo sistema**. El modelo genera a partir de tokens de entrada una secuencia probabilística de tokens de salida. La Messages API encapsula este proceso como un contrato HTTPS. Las aplicaciones Claude añaden cuentas, memoria de conversaciones e interfaces de usuario. Claude Code incorpora en un endpoint un tiempo de ejecución local con acceso a archivos, shell y herramientas.

La primera versión de Claude se presentó en 2023 como servicio de chat y API ([Presentación de Claude](https://www.anthropic.com/news/introducing-claude)). Anthropic describe Claude como un asistente orientado a un comportamiento útil, honesto e inocuo. Sin embargo, esta orientación, una gran ventana de contexto o un texto redactado de forma convincente no prueban la exactitud factual, la autorización ni la ejecución segura. Un sistema de producción debe tratar las respuestas del modelo como entrada no confiable, validada contra esquemas y políticas.

La explicación comienza con una solicitud al modelo y sigue la respuesta a través de la Messages API, los bloques de contenido y las llamadas a herramientas. Solo cuando este flujo está claro se profundiza en Claude Code, MCP, permisos, prompt injection, operación y recuperación.

Claude no es un sistema operativo que actúe de forma autónoma, sino un servicio de inferencia. La aplicación circundante proporciona contexto, comprueba respuestas, ejecuta herramientas aprobadas y asume la responsabilidad de identidades, acceso a datos y auditoría.

## Enfoque de arquitectura: servicio de inferencia más aplicación de control

Claude se ejecuta normalmente como un servicio de inferencia externo. Un cliente envía instrucciones del sistema, mensajes de conversación, bloques de contenido, definiciones de herramientas y parámetros de generación. El servicio autentica y limita la solicitud, tokeniza el contexto, realiza la inferencia y entrega un mensaje o un flujo de eventos. A continuación, la aplicación decide si muestra texto, valida JSON, ejecuta una herramienta, devuelve un resultado o cancela la ejecución.

La API directa no es la única vía de aprovisionamiento. Los modelos Claude también se ofrecen a través de plataformas cloud; la identidad, el endpoint, las regiones, las cuotas, el logging y las condiciones contractuales difieren en ellas. La arquitectura de la aplicación mantiene separados los adaptadores de proveedores y el flujo de trabajo funcional. Un cambio de modelo no es una simple conmutación DNS si difieren los tipos de herramientas, los límites de contexto, las razones de parada o las funciones del proveedor.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1116" src="/images/kb-interaktiv-claude.svg?v=20260813" title="Interaktive Infografik: Claude von Benutzer, Anwendung und Workspace über Messages API, Tokenisierung und Modellinferenz bis zu Content Blocks, Tool Use, MCP, Claude Code, Rate Limits, Logging, Datenschutz und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-claude.svg?v=20260813">Abrir directamente el gráfico interactivo</a>.
</iframe>

## Capa de modelo y semántica probabilística

Los modelos Claude procesan texto, código y, según el modelo, otras modalidades. La [visión general de modelos](https://platform.claude.com/docs/en/about-claude/models/overview) documenta familias de modelos, ventanas de contexto y capacidades. Estos valores no deben incluirse como un hecho permanentemente congelado en un artículo estático. Antes del despliegue o durante la compilación, un cliente consulta mediante la Models API las capacidades documentadas y guarda el identificador del modelo realmente utilizado con cada resultado.

Un modelo de lenguaje no es una base de datos ni un motor de reglas. Las mismas entradas pueden producir salidas distintas según el muestreo, el contexto del sistema, la revisión del modelo y los resultados de herramientas. Incluso con una temperatura baja, se mantienen las siguientes propiedades:

- Los hechos pueden faltar, estar desactualizados o ser inventados.
- Las instrucciones pueden ponderarse de forma ambigua.
- Los contextos largos pueden ocultar detalles relevantes.
- El texto con apariencia estructurada no es un objeto válido sin cierre de esquema.
- Una explicación plausible no demuestra que una herramienta se haya ejecutado realmente.
- Un modelo no conoce permisos salvo la información y los límites de herramientas que aplique la aplicación.

Por ello, la aceptación técnica utiliza casos de evaluación, esquemas esperados, verificaciones deterministas y fuentes especializadas. Un benchmark de modelo exitoso no sustituye una medición específica de la aplicación sobre precisión, latencia, costes y daños ante errores.

## Entrenamiento, Constitution y responsabilidades

Anthropic desarrolló **Constitutional AI** como complemento del aprendizaje supervisado y el aprendizaje por refuerzo. El modelo critica y revisa respuestas conforme a principios explícitos; un segundo paso de entrenamiento utiliza feedback generado por AI. Anthropic explica este enfoque y sus límites en [Claude’s Constitution](https://www.anthropic.com/research/claudes-constitution). La [tarjeta de modelo de Claude 3](https://assets.anthropic.com/m/61e7d27f8c8f5919/original/Claude-3-Model-Card.pdf) documenta para una generación concreta de modelos el entrenamiento, la evaluación y las medidas de seguridad.

Para los administradores, esta historia es relevante porque explica el comportamiento, no porque sustituya el control en tiempo de ejecución. El entrenamiento de seguridad puede reducir respuestas dañinas, pero no puede imponer ni la separación de inquilinos ni la autorización de herramientas. Las negativas son posibles salidas normales del modelo. La aplicación debe tratarlas como `stop_reason` o como un tipo de contenido y no debe deducir, de la ausencia de una negativa, que una acción es segura.

El modelo probabilístico solo se convierte en un servicio administrable mediante la Messages API. Cada solicitud transmite de nuevo el contexto de conversación necesario y recibe bloques de contenido estructurados como respuesta.

## Messages API: contrato de conversación sin estado

`POST /v1/messages` recibe una lista de mensajes `user` y `assistant` y genera el siguiente turno del asistente. Un prompt de sistema se encuentra en el campo de nivel superior independiente `system`; no existe un rol `system` en `messages`. El contenido se puede enviar como cadena o como lista de bloques tipados ([Create a Message](https://platform.claude.com/docs/en/api/messages/create)).

La API no tiene estado **sin almacenamiento adicional de hilos**. Para un diálogo de varios turnos, el cliente vuelve a enviar el historial necesario. Esto tiene consecuencias directas:

1. La aplicación es propietaria del ID de conversación, el orden y la retención.
2. Recortar, resumir u omitir modifica el contexto del modelo.
3. El prompt de sistema, las definiciones de herramientas y el historial cuentan para el presupuesto de entrada.
4. Un ID de solicitud del proveedor no sustituye un ID funcional de trabajo.
5. Reintentar la misma solicitud puede generar una salida nueva y costes adicionales.

Por tanto, la operación funcional recibe su propio ID de correlación e idempotencia. Antes de reintentar, el orquestador comprueba si ya existe un resultado o un efecto irreversible de una herramienta.

## Bloques de contenido y Stop Reasons

Una respuesta correcta no es necesariamente un único texto. `content` es una lista ordenada de bloques tipados, como texto, Tool Use, Thinking o bloques de herramientas de servidor. `usage` informa de las clases de tokens. `stop_reason` describe por qué terminó la generación. La [documentación oficial de Stop Reasons](https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons) distingue, entre otros, `end_turn`, `max_tokens`, `stop_sequence`, `tool_use`, `pause_turn`, `refusal` y `model_context_window_exceeded`.

Solo `end_turn` significa un turno finalizado de forma natural; no prueba que esté completo desde el punto de vista funcional. `max_tokens` y `model_context_window_exceeded` indican datos potencialmente truncados. `tool_use` es una solicitud al orquestador, no un efecto ejecutado. `pause_turn` exige una continuación conforme al protocolo. Pueden añadirse nuevos valores enum, por lo que los analizadores deben rechazar visiblemente los tipos desconocidos o ignorarlos de forma segura, en vez de caer en un éxito predeterminado.

## Versión de API y versión de modelo

Cada solicitud directa a la API incluye una cabecera `anthropic-version`. Esta versión de API estabiliza campos y la semántica de streaming, pero no puede impedir nuevas entradas opcionales, campos de salida o variantes enum ([API versioning](https://platform.claude.com/docs/en/api/versioning)). Por ello, el analizador del cliente se construye de forma compatible hacia adelante y registra los bloques desconocidos.

De ello se diferencia el **ID de modelo**. Anthropic garantiza para un ID fijado una versión de modelo constante durante su vida útil; los alias de conveniencia pueden tener otras reglas ([Model IDs and versioning](https://platform.claude.com/docs/en/about-claude/models/model-ids-and-versions)). Para auditoría y reproducción, un sistema guarda:

- versión de API y cabeceras beta,
- ID de modelo en lugar de solo un nombre mostrado,
- versión del prompt de sistema o de la política,
- versión del conjunto de herramientas y del esquema JSON,
- versiones de retrieval y documentos,
- parámetros de generación,
- correlación de solicitud, trabajo y usuario,
- Stop Reason, Usage y resultado de validación.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Claude-Modellinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$headers = @{
  'x-api-key' = $env:ANTHROPIC_API_KEY
  'anthropic-version' = '2023-06-01'
}
Invoke-RestMethod -Headers $headers -Uri 'https://api.anthropic.com/v1/models' |
  ConvertTo-Json -Depth 12</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">curl --fail --silent --show-error https://api.anthropic.com/v1/models \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H 'anthropic-version: 2023-06-01' | jq .</code></pre>
  </div>
</div>

[`Invoke-RestMethod`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod) y [`ConvertTo-Json`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertto-json) procesan Windows; [`curl`](https://curl.se/docs/manpage.html) y [`jq`](https://jqlang.org/manual/) procesan Unix. La clave procede de un almacén de secretos y no se escribe ni se muestra en la línea de comandos.

## Tokenización, contexto y costes

Las ventanas de contexto se miden en tokens, no en caracteres ni archivos. Los esquemas de herramientas, el prompt de sistema, los mensajes, las imágenes, los documentos y los resultados de herramientas contribuyen a la entrada; el texto generado y, en su caso, Thinking contribuyen a la salida. La [Token Counting API](https://platform.claude.com/docs/en/build-with-claude/token-counting) recibe la misma estructura de entrada que Messages y proporciona una estimación previa.

Un contexto grande no es un archivo. Cuantos más datos irrelevantes envíe el cliente, mayores serán los costes, la latencia y el riesgo de instrucciones contradictorias. Por ello, los sistemas de producción construyen una **Context Assembly Pipeline**:

1. Comprobar derechos de usuario e inquilino antes del retrieval.
2. Recuperar documentos mediante ID y versiones estables.
3. Marcar el contenido no confiable como datos, no como instrucciones del sistema.
4. Limitar el volumen de datos, los tipos de archivo y el presupuesto de tokens.
5. Incluir metadatos de fuentes y hashes.
6. Validar la respuesta contra las mismas fuentes y esquemas.

Los bloques de contenido de resultados de búsqueda pueden transmitir fuentes RAG con título y procedencia para que Claude genere citas ([Search results](https://platform.claude.com/docs/en/build-with-claude/search-results)). Estas citas solo son tan fiables como el retrieval, la identidad del documento y los metadatos suministrados.

## Prompt Caching

Prompt Caching almacena prefijos reutilizables y reduce costes de procesamiento y latencia. La jerarquía de caché sigue `tools` → `system` → `messages`; los cambios en una parte anterior invalidan esa parte y los niveles posteriores. La [documentación oficial de Prompt Caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) describe breakpoints automáticos y explícitos, así como TTL cortos.

Un acierto de caché solo demuestra que se reutilizó un prefijo idéntico. No garantiza ni datos de fuente actuales ni una salida idéntica. Las métricas de caché se registran por separado para Creation y Read. Los workspaces aíslan las cachés de prompts en la API directa de Claude ([Workspaces](https://platform.claude.com/docs/en/manage-claude/workspaces)). Los secretos no deben incluirse en prompts pese al TTL corto; caché y retención son mecanismos distintos.

## Streaming con Server-Sent Events

Con `stream: true` la API entrega Server-Sent Events incrementales. Los bloques de contenido se inician, se amplían mediante deltas y se completan; la información final de Message y Usage sigue siendo necesaria para un procesamiento correcto. La [documentación de streaming](https://platform.claude.com/docs/en/build-with-claude/streaming) describe tipos de eventos y acumuladores de SDK.

Un delta de texto visible no es un commit. Si se interrumpe la conexión, el usuario puede haber visto ya texto parcial mientras que el cliente no dispone de un Message completo. Los argumentos JSON o de herramientas solo se pueden utilizar tras completar el bloque y validar el esquema. Gateways y proxies deben gestionar correctamente conexiones HTTP largas, buffering, timeouts y backpressure.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Claude-Messages und Streaming">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$body = @{
  model = $env:CLAUDE_MODEL
  max_tokens = 256
  messages = @(@{ role = 'user'; content = 'Antworte mit einem Satz.' })
} | ConvertTo-Json -Depth 8
curl.exe --no-buffer --fail-with-body https://api.anthropic.com/v1/messages `
  -H "x-api-key: $env:ANTHROPIC_API_KEY" `
  -H 'anthropic-version: 2023-06-01' -H 'content-type: application/json' `
  --data-raw $body</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">jq -n --arg model "$CLAUDE_MODEL" '{
  model: $model, max_tokens: 256,
  messages: [{role: "user", content: "Antworte mit einem Satz."}]
}' | curl --no-buffer --fail-with-body https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H 'anthropic-version: 2023-06-01' -H 'content-type: application/json' \
  --data-binary @-</code></pre>
  </div>
</div>

La solicitud de ejemplo utiliza deliberadamente un ID de modelo leído de la configuración. Para SSE, también se establece `stream: true` y se analiza el flujo de eventos nombrado conforme al protocolo; leer solo líneas no basta para código de producción.

La salida de texto aún no modifica ningún sistema. Solo Tool Use vincula una propuesta del modelo con una acción; y precisamente en este punto deben intervenir la aplicación, los permisos y la aprobación humana.

## Tool Use: el modelo solicita, la aplicación actúa

Tool Use es un contrato entre el modelo y el orquestador. La aplicación describe una herramienta con nombre, propósito y esquema JSON. Claude puede generar entonces un bloque `tool_use` con argumentos. El cliente valida el nombre y la entrada, autoriza la llamada concreta, la ejecuta y devuelve un bloque `tool_result` con el mismo ID de Tool Use. Solo otro turno de modelo puede formar una respuesta a partir de ello ([How tool use works](https://platform.claude.com/docs/en/agents-and-tools/tool-use/how-tool-use-works)).

La [visión general de herramientas](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview) distingue:

- **Client Tools:** La propia aplicación ejecuta.
- **Anthropic-schema tools:** Anthropic define el esquema, pero el cliente sigue ejecutando.
- **Server Tools:** Anthropic ejecuta en su infraestructura y entrega bloques de resultados.
- **MCP Connector:** La API se conecta a un servidor MCP remoto.

Esta distinción determina la ruta de red, los secretos, el logging, la protección de datos y el ámbito de fallo. `strict: true` impone la conformidad con el esquema, no la corrección funcional ni la autorización. Una llamada sintácticamente válida `delete_user(id)` aún puede eliminar al usuario equivocado.

### Bucle de herramientas apto para producción

Un orquestador seguro ejecuta los siguientes pasos por cada Tool Use:

1. Comprobar el nombre de la herramienta contra la lista de permitidas y la fase del flujo de trabajo.
2. Validar estrictamente el esquema JSON y rechazar campos desconocidos.
3. Volver a comprobar en el servidor los permisos de usuario, inquilino y objeto.
4. Normalizar las entradas; limitar rutas, URL, ID y tamaños.
5. Clasificar Read, Write, External Message y Destructive Action.
6. Exigir aprobación funcional o principio de cuatro ojos para alto riesgo.
7. Ejecutar con una identidad efímera y mínima en un tiempo de ejecución aislado.
8. Establecer timeout, límite de salida y clave de idempotencia.
9. Limpiar el resultado de secretos e instrucciones no confiables.
10. Auditar de forma inmutable Tool Use, decisión, efecto y resultado.

El bucle limita turnos, herramientas paralelas, costes acumulados y errores repetidos. El resultado de una herramienta vuelve a ser contenido no confiable. Una fila de base de datos o una página web puede contener prompt injection y no debe sobrescribir la política del orquestador.

## MCP como límite de protocolo

El Model Context Protocol estandariza la conexión de aplicaciones de AI con herramientas, recursos y prompts. Anthropic describe MCP como un protocolo cliente-servidor abierto ([visión general de MCP](https://docs.anthropic.com/en/docs/mcp)); la [especificación normativa de MCP](https://modelcontextprotocol.io/specification/2025-06-18) define mensajes y capacidades.

MCP hace que las integraciones sean intercambiables, pero no automáticamente confiables. Un servidor puede leer datos, desencadenar acciones, entregar resultados muy grandes o devolver contenido de un tercero. Por eso, incluirlo en un archivo de configuración es una decisión de software y permisos. Se inventarían operador, transporte, endpoint, autenticación, herramientas, recursos, scopes, clases de datos, versión, timeout, límite de salida y revocación.

El [MCP Connector de Messages API](https://platform.claude.com/docs/en/agents-and-tools/mcp-connector) es una herramienta de servidor en infraestructura de Anthropic. En cambio, un servidor stdio iniciado localmente en Claude Code se ejecuta en el endpoint. Mismo protocolo, distintas rutas de datos y ámbitos de fallo.

## Claude Code: tiempo de ejecución local de agentes

Claude Code es un agente para terminal y repositorio. Recopila contexto del proyecto, lo envía a un endpoint de modelo, interpreta propuestas del modelo y utiliza herramientas locales para leer, modificar y ejecutar. La [instalación oficial](https://docs.anthropic.com/en/docs/claude-code/getting-started) documenta Windows mediante WSL o Git Bash, así como macOS y Linux. La [referencia de CLI](https://docs.anthropic.com/en/docs/claude-code/cli-usage) describe el funcionamiento interactivo, Print Mode, salida JSON, reanudación de sesión, selección de modelo y reglas de herramientas.

El límite de confianza central es el proceso local. Claude Code solo puede actuar con los permisos del sistema operativo y las credenciales accesibles del usuario que lo inició, pero estos permisos pueden ser muy amplios: repositorio, agente SSH, CLI cloud, registro de paquetes, Kubernetes, cookies del navegador, variables de entorno o túneles de producción. «El agente preguntó» no es aislamiento.

La [documentación de seguridad de Claude Code](https://docs.anthropic.com/en/docs/claude-code/security) describe valores predeterminados de solo lectura, prompts de permisos, límites de proyecto y protección frente a prompt injection. Al mismo tiempo, establece que ningún sistema es completamente inmune. La operación no interactiva necesita políticas más estrictas, pues el contenido del repositorio, el texto de incidencias, los logs de compilación y las salidas de herramientas pueden contener instrucciones de atacantes.

### Perfil de ejecución y permisos

Un endpoint gestionado o un runner de CI utiliza:

- una cuenta de usuario o carga de trabajo dedicada,
- un directorio de trabajo específico del proyecto,
- permisos mínimos de sistema de archivos y red,
- credenciales efímeras limitadas por destino y acción,
- herramientas permitidas y prohibidas mediante política central,
- contenedor, VM o sandbox para compilaciones no confiables,
- escaneo de secretos antes de incorporar contexto,
- turnos, tiempo, salida y costes limitados,
- Git diff, pruebas y aprobación antes de commit o despliegue,
- correlación de sesión, herramientas y proveedor en el registro de auditoría.

El interruptor `--dangerously-skip-permissions` no es una estrategia de automatización. Elimina una capa de protección y solo es aceptable en una sandbox previa con su propia política.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Claude-Code-Inventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-Command claude | Format-List Source,Version
claude doctor
Get-FileHash (Get-Command claude).Source -Algorithm SHA256
Get-ChildItem .\.claude -Force -Recurse | Select-Object FullName,Length</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">command -v claude
claude doctor
file "$(command -v claude)"
sha256sum "$(command -v claude)"
find ./.claude -maxdepth 3 -type f -printf '%p %s\n'</code></pre>
  </div>
</div>

[`Get-Command`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/get-command), [`Get-FileHash`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash) y [`Get-ChildItem`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-childitem) inventarían Windows. POSIX [`command`](https://pubs.opengroup.org/onlinepubs/9799919799/utilities/command.html), [`file`](https://man7.org/linux/man-pages/man1/file.1.html), [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) y [`find`](https://man7.org/linux/man-pages/man1/find.1.html) se encargan de Unix. Un hash solo se evalúa contra una fuente de versión o paquete confiable.

## Operación no interactiva y CI

Print Mode hace que Claude Code sea apto para scripts. `--output-format json` o `stream-json` proporcionan resultados legibles por máquina; `--max-turns` limita la ejecución del agente. Los datos de entrada y el código de salida siguen formando parte del registro de trabajo. La salida de texto libre no se evalúa como script de shell.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für begrenzten Claude-Code-Print-Mode">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-Content .\review-request.txt -Raw |
  claude -p --max-turns 3 --output-format json `
    --allowedTools 'Read' --disallowedTools 'Bash' 'Edit' |
  Tee-Object -FilePath .\claude-review.json |
  ConvertFrom-Json</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">cat ./review-request.txt |
  claude -p --max-turns 3 --output-format json \
    --allowedTools Read --disallowedTools Bash Edit |
  tee ./claude-review.json | jq .</code></pre>
  </div>
</div>

[`Get-Content`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-content), [`Tee-Object`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/tee-object) y [`ConvertFrom-Json`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json) procesan Windows. [`cat`](https://www.gnu.org/software/coreutils/manual/html_node/cat-invocation.html) y [`tee`](https://www.gnu.org/software/coreutils/manual/html_node/tee-invocation.html) se encargan de Unix. Un trabajo de CI recibe además reglas fijas de directorio de trabajo, red, secretos y ramas.

## Settings, Hooks y Policies

Claude Code combina configuraciones de usuario, proyecto y gestión centralizada. Los archivos de proyecto pueden estar en el repositorio y, por tanto, forman parte de la revisión de código. Los hooks ejecutan comandos en puntos definidos del ciclo de vida; son código ejecutable, no configuración de prompts inocua. Los servidores MCP amplían los sistemas accesibles. Por ello, una política empresarial debe controlar conjuntamente Settings, Hooks, plugins, Skills, MCP y patrones de shell permitidos.

La CLI puede permitir permisos una vez o de forma permanente. Los comodines amplios aceleran el trabajo, pero amplían el radio de impacto. Una buena regla no permite «Bash», sino una función de lectura limitada o una herramienta tipada previa. Para operaciones de escritura, se comprueban ruta de destino, rama y diff. Para despliegues o mensajes externos se mantiene una aprobación independiente fuera del modelo.

## Identidad, organización y workspaces

Claude Platform asigna el uso a una organización y workspaces. Las claves de workspace están limitadas a los recursos y el uso de ese workspace; los miembros tienen roles de workspace. Los workspaces separados para desarrollo, pruebas y producción separan claves, límites, lotes, archivos y cachés de prompts ([Workspaces](https://platform.claude.com/docs/en/manage-claude/workspaces)).

Admin API e Inference API utilizan tipos de clave distintos. La [documentación de Admin API](https://platform.claude.com/docs/en/manage-claude/admin-api) describe miembros, invitaciones, workspaces y API Keys. Una clave de administrador nunca debe estar en una aplicación que solo necesita enviar Messages. El offboarding revoca el acceso de usuario, las claves personales de Claude Code y, cuando corresponda, las claves de servicio creadas por separado.

Claude Code puede autenticarse mediante Console/OAuth, planes Claude o proveedores empresariales. Lo decisivo es la **ruta efectiva de facturación y datos** de la sesión. De otro modo, las cuentas personales en un repositorio empresarial eluden controles de workspace, retención, centros de coste o auditoría.

## Rate Limits, gasto y backpressure

Anthropic distingue entre Spend Limits y Rate Limits. Messages se limita por solicitudes por minuto, tokens de entrada por minuto y tokens de salida por minuto. Los límites utilizan token buckets; por ello, ráfagas cortas pueden desencadenar 429 a pesar de una media por minuto aparentemente adecuada. `retry-after` y las cabeceras de respuesta proporcionan el marco de backoff ([Rate limits](https://platform.claude.com/docs/en/api/rate-limits)).

Un gateway implementa:

- cola y prioridad por inquilino o flujo de trabajo,
- conteo de tokens antes de aceptar trabajos grandes,
- backoff exponencial con jitter y `retry-after`,
- límites globales y por workspace de paralelismo,
- circuit breaker para errores 5xx y de red,
- alerta de presupuesto y límite máximo estricto de costes,
- métricas separadas para Cache Read, Cache Write, Input y Output,
- fallback controlado con cambio de calidad documentado.

La [documentación de errores de API](https://platform.claude.com/docs/en/api/errors) distingue 400, 401, 402, 403, 404, 413, 429, 500 y 529, y proporciona `request_id`. Los SDK reintentan automáticamente determinados errores transitorios. Estos reintentos se incluyen en la planificación de capacidad y costes.

## Lotes y procesamiento asíncrono

La Message Batches API procesa muchas solicitudes independientes de forma asíncrona. Los lotes están vinculados al workspace; los resultados individuales pueden tener éxito o fallar. La disponibilidad de resultados y la retención en el servidor difieren de la vía síncrona. La [documentación de lotes](https://platform.claude.com/docs/en/build-with-claude/batch-processing) indica límites de tamaño, tiempo de ejecución y recuperación.

Un trabajo por lotes posee un `custom_id` por objeto funcional, un manifiesto de entrada, un cursor de recuperación y una comprobación de resultados. «Batch completed» solo significa que todas las entradas tienen un estado final. La importación procesa cada resultado de forma idempotente, detecta ID faltantes y archiva las versiones de modelo, prompt y esquema. Para datos personales o regulados, se comprueba si la vía de lotes está permitida por contrato y modelo de retención de datos.

Después de la función y la escala sigue la cuestión de datos: qué contenidos salen del sistema propio, cuánto tiempo se conservan y qué operadores adicionales participan en las plataformas cloud.

## Retención de datos, privacidad y plataformas de terceros

La retención de datos depende de la interfaz y el contrato. Para el uso comercial de API, Anthropic describe una eliminación estándar de inputs y outputs en un plazo de 30 días, pero menciona excepciones para funciones con almacenamiento más prolongado, acuerdos diferentes, Safety Enforcement y obligaciones legales ([retención de datos comerciales](https://privacy.claude.com/en/articles/7996866-how-long-do-you-store-my-organization-s-data)). Las aplicaciones Claude almacenan conversaciones para funciones de producto según sus propias reglas.

Zero Data Retention no es un interruptor global para todos los productos. La [documentación de ZDR](https://privacy.claude.com/en/articles/8956058-i-have-a-zero-data-retention-agreement-with-anthropic-what-products-does-it-apply-to) describe API, organizaciones y excepciones elegibles. Para el acceso mediante proveedores cloud se aplican adicionalmente su ruta de datos, regiones, gestión de claves y contrato. MCP, Web Search o herramientas externas pueden transferir datos a otros responsables.

Antes de aprobar producción, se crea una matriz de flujo de datos:

| Ruta | Datos | Control responsable |
|---|---|---|
| Cliente → endpoint del modelo | Prompt, archivos, imágenes, esquema de herramientas | Clasificación, minimización, contrato, región |
| Endpoint del modelo → cliente | Bloques de contenido, Usage, Request ID | Validación, redacción, logging |
| Cliente → herramienta/MCP | Argumentos de herramientas, contexto de usuario | Autorización, scope, DPA, auditoría |
| Herramienta → modelo | Resultado y contenido no confiable | Filtro, límite de salida, protección contra injection |
| Claude Code local | Repositorio, shell, credenciales | Política del endpoint, sandbox, secretos |
| Logs/tracing | Prompts, resultados, metadatos | Redacción, acceso, retención |

## Prompt injection y contenido no confiable

Prompt injection surge cuando los datos intentan convertirse en instrucciones. Las superficies de ataque son páginas web, correos electrónicos, documentos, incidencias, comentarios de código fuente, recursos MCP, resultados de herramientas y salida de terminal. El ataque no tiene que «convencer» al modelo si la aplicación ya otorga permisos amplios a un Tool Use no comprobado.

La defensa es multicapa:

1. Aplicar la política del sistema y los permisos fuera del texto del modelo.
2. Filtrar el retrieval conforme a derechos de usuario e inquilino.
3. Marcar bloques de datos con procedencia, nivel de confianza y propósito.
4. Autorizar herramientas de forma mínima, tipada y orientada a objetos.
5. Aprobar por separado las acciones de escritura, envío, pago y eliminación.
6. Aislar en sandbox la red y el sistema de archivos de la ejecución.
7. No incluir secretos en contexto, resultados de herramientas ni logs visibles.
8. Validar salida y argumentos de herramientas contra esquema y reglas funcionales.
9. Limitar los bucles de agentes por tiempo, turnos, costes y efectos.
10. Realizar pruebas adversariales con injection indirecta en rutas de datos reales.

Un Human-in-the-Loop solo es eficaz si la persona ve el objetivo, el efecto y los datos relevantes. Un mensaje genérico «¿Permitir?» provoca fatiga de aprobación. Las acciones de alto riesgo muestran parámetros normalizados y son verificadas por un Policy Enforcement Point independiente.

## Ruta de red y operación con proxy

Claude API y Claude Code necesitan acceso HTTPS. Un proxy empresarial puede encargarse de autenticación, inspección TLS, control de egreso y logging, pero con ello pasa a formar parte de la cadena de confidencialidad y disponibilidad. La [documentación de proxy para Claude Code](https://docs.anthropic.com/en/docs/claude-code/corporate-proxy) indica variables de proxy compatibles, paquetes CA y destinos necesarios.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Claude-DNS-, TCP- und TLS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName api.anthropic.com
Test-NetConnection api.anthropic.com -Port 443
curl.exe -sS -D - -o NUL https://api.anthropic.com/v1/models
Get-NetTCPConnection -State Established | Where-Object RemotePort -eq 443</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig +short api.anthropic.com A api.anthropic.com AAAA
curl -sS -D - -o /dev/null https://api.anthropic.com/v1/models
ss -ntp state established '( dport = :443 )'
openssl s_client -connect api.anthropic.com:443 -servername api.anthropic.com -brief</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname), [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) y [`Get-NetTCPConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection) comprueban Windows. [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility), [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) y [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) comprueban Unix. Un 401 sin clave confirma que DNS, TCP, TLS y HTTP alcanzan la ruta de API esperada; no es una prueba de inferencia satisfactoria.

La operación segura de agentes no solo debe contar llamadas al modelo, sino hacer trazable todo el camino desde la entrada, pasando por la decisión de herramienta, hasta el efecto externo.

## Observabilidad y auditoría

Una solicitud productiva genera métricas funcionales, técnicas y de costes. Como mínimo se registran:

- hora, workspace, aplicación, flujo de trabajo y correlación de usuario seudonimizada,
- ID de modelo, versión de API, versión de prompt, conjunto de herramientas y esquema,
- latencia hasta cabeceras, primer token y Message completo,
- tokens de Input, Cache Creation, Cache Read y Output,
- Stop Reason, tipos de bloques de contenido y estado de validación,
- clase de error, número de reintentos, `request_id` y `retry-after`,
- nombre de herramienta, decisión de autorización, duración, efecto y clase de resultado,
- costes estimados y facturados,
- clase de redacción y retención.

Los prompts y las respuestas de modelos no se registran automáticamente en su totalidad. Pueden contener datos personales, secretos, código fuente o payloads de atacantes. Los metadatos de auditoría y el contenido de depuración utilizan almacenamientos, roles y plazos de eliminación separados. La [Usage Report API](https://platform.claude.com/docs/en/api/admin/usage_report) proporciona datos agregables de uso de API y Claude Code; no obstante, los efectos de herramientas locales deben auditarse en el propio sistema.

## Gestión de errores y reanudación

Una solicitud al modelo no es automáticamente idempotente. En caso de timeout, el proveedor puede haber completado la generación aunque el cliente no recibiera respuesta. Con Tool Use, el efecto externo puede haberse producido ya. Por ello, el orquestador almacena transiciones de estado:

`accepted → context_built → request_sent → response_received → validated → tool_authorized → tool_executed → result_returned → completed`

Para cada transición existen ID de correlación, hora, versiones y checkpoint. Un reintento antes de `tool_executed` puede repetir el turno del modelo; un reintento posterior consulta primero el efecto funcional. Las herramientas externas admiten Idempotency Keys o una prueba Read-after-Write. Ante un estado desconocido, se escala, no se ejecuta ciegamente de nuevo.

Los modelos de fallback cambian calidad, costes, Context Limit y comportamiento de herramientas. Son una ruta propia probada. La aplicación no oculta un fallback si modifica la validez de la afirmación o el cumplimiento.

## Copia de seguridad y recuperación

El modelo en sí, como servicio externo, no lo respalda el cliente. Deben ser recuperables los datos y las configuraciones de control y almacenamiento propios:

- prompts de sistema, políticas y conjuntos de evaluación,
- esquemas de herramientas, implementaciones y reglas de permisos,
- inventario MCP y versiones de servidores aprobadas,
- aprovisionamiento de workspaces y claves como código o runbook,
- fuentes de retrieval, embeddings o índices y versiones de documentos,
- estado de conversación o trabajo y datos de idempotencia,
- metadatos de redacción, auditoría y costes,
- configuraciones de proyecto de Claude Code, hooks y Skills comprobados,
- adaptadores de proveedores y migraciones de modelos probadas.

Las API Keys no se «restauran» desde copias de seguridad, sino que se revocan y se aprovisionan de nuevo. Una prueba de recuperación ante desastres crea un nuevo tiempo de ejecución, asigna identidades mínimas, envía un caso de prueba conocido, valida esquema y fuentes, ejecuta una herramienta de prueba inocua y correlaciona logs de proveedor, gateway y herramientas. Los fundamentos se encuentran en [Copia de seguridad y recuperación ante desastres](/kb/backup-dr), [APIs](/kb/apis), [Hardening](/kb/haertung) y [Troubleshooting](/kb/troubleshooting).

## Historia técnica

Anthropic se fundó en 2021 como empresa de investigación y producto para sistemas de AI seguros. Claude se presentó en marzo de 2023 como asistente de chat y API tras una fase cerrada de socios ([Presentación de Claude](https://www.anthropic.com/news/introducing-claude)). Claude 2 amplió el contexto y la disponibilidad pública en julio de 2023 ([Claude 2](https://www.anthropic.com/news/claude-2)).

En 2024, la familia Claude 3 estableció las denominaciones Haiku, Sonnet y Opus como niveles de velocidad, costes y rendimiento ([Claude 3](https://www.anthropic.com/news/claude-3-family)). Estos nombres de productos no son una comprobación estable de capacidades; las aplicaciones vinculan ID fijados y consultan los límites por separado.

En febrero de 2025 apareció Claude Code como Research Preview junto con un modelo de razonamiento híbrido ([Claude 3.7 y Claude Code](https://www.anthropic.com/news/claude-3-7-sonnet)). En mayo de 2025, Claude Code estuvo disponible de forma general; al mismo tiempo, Anthropic añadió a la API componentes orientados a agentes como Code Execution, MCP Connector y Prompt Caching más prolongado ([Claude 4](https://www.anthropic.com/news/claude-4)). Con ello, la cuestión operativa pasó de «¿Cómo consumo texto?» a «¿A qué datos y efectos puede acceder una capa probabilística de planificación?»

Posteriormente, la plataforma desarrolló una versionación de modelos más sólida, workspaces, API de uso y costes, esquemas de herramientas, caché, lotes y SDK de agentes. El punto de arquitectura permanente sigue siendo: el modelo propone; las API y los marcos de agentes transportan; la aplicación de control autoriza, ejecuta, valida y audita.

## Lista de comprobación para administradores

Las numerosas interfaces de Claude se vuelven gestionables si se registran por separado el modelo, la API, la herramienta y el proceso local. La lista resume los controles que deberían demostrarse antes de un uso productivo.

- **Capas:** Inventariar por separado el modelo, API, aplicación Claude, gateway, herramienta y endpoint de Claude Code.
- **Contratos:** Tratar versión de API, ID de modelo fijado, bloques de contenido, Stop Reasons y tipos desconocidos.
- **Contexto:** Minimizar datos, autorizar retrieval, contar tokens y versionar fuentes.
- **Herramientas:** Imponer esquema, permiso funcional, idempotencia, timeout, límite de salida y auditoría.
- **MCP:** Comprobar procedencia del servidor, transporte, scopes, clases de datos y riesgo de prompt injection.
- **Claude Code:** Limitar permisos de usuario, secretos, red, sandbox, Settings, Hooks y política de herramientas.
- **Identidad:** Separar claramente organización, workspace, claves de usuario, API y administrador.
- **Capacidad:** Supervisar RPM, ITPM, OTPM, gasto, caché y costes de reintentos.
- **Privacidad:** Evaluar por separado interfaz de producto, proveedor, retención, ZDR, herramientas de terceros y logs.
- **Calidad:** Aplicar evaluaciones, comprobación de esquema y fuentes, y aprobación funcional antes de los efectos.
- **Recuperación:** Mantener reproducibles prompts, políticas, herramientas, retrieval, trabajos y aprovisionamiento.
- **Evidencia:** Ejecutar una prueba completa desde la autenticación, pasando por Message y herramienta, hasta la auditoría y el informe de costes.

## Fuentes

- [Anthropic – Presentación de Claude](https://www.anthropic.com/news/introducing-claude)
- [Claude Platform – Visión general de modelos](https://platform.claude.com/docs/en/about-claude/models/overview)
- [Anthropic – Claude’s Constitution](https://www.anthropic.com/research/claudes-constitution)
- [Anthropic – Tarjeta de modelo Claude 3](https://assets.anthropic.com/m/61e7d27f8c8f5919/original/Claude-3-Model-Card.pdf)
- [Claude API – Create a Message](https://platform.claude.com/docs/en/api/messages/create)
- [Claude Platform – Stop reasons](https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons)
- [Claude Platform – API versioning](https://platform.claude.com/docs/en/api/versioning)
- [Claude Platform – Model IDs and versions](https://platform.claude.com/docs/en/about-claude/models/model-ids-and-versions)
- [Microsoft – Invoke-RestMethod](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod)
- [Microsoft – ConvertTo-Json](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertto-json)
- [curl – Manual](https://curl.se/docs/manpage.html)
- [jq – Manual](https://jqlang.org/manual/)
- [Claude Platform – Token counting](https://platform.claude.com/docs/en/build-with-claude/token-counting)
- [Claude Platform – Search results and citations](https://platform.claude.com/docs/en/build-with-claude/search-results)
- [Claude Platform – Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [Claude Platform – Workspaces](https://platform.claude.com/docs/en/manage-claude/workspaces)
- [Claude Platform – Streaming messages](https://platform.claude.com/docs/en/build-with-claude/streaming)
- [Claude Platform – How tool use works](https://platform.claude.com/docs/en/agents-and-tools/tool-use/how-tool-use-works)
- [Claude Platform – Tool use overview](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview)
- [Anthropic – Model Context Protocol](https://docs.anthropic.com/en/docs/mcp)
- [Model Context Protocol – Specification](https://modelcontextprotocol.io/specification/2025-06-18)
- [Claude Platform – MCP connector](https://platform.claude.com/docs/en/agents-and-tools/mcp-connector)
- [Anthropic – Configurar Claude Code](https://docs.anthropic.com/en/docs/claude-code/getting-started)
- [Anthropic – Referencia de CLI de Claude Code](https://docs.anthropic.com/en/docs/claude-code/cli-usage)
- [Anthropic – Seguridad de Claude Code](https://docs.anthropic.com/en/docs/claude-code/security)
- [Microsoft – Get-Command](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/get-command)
- [Microsoft – Get-FileHash](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash)
- [Microsoft – Get-ChildItem](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-childitem)
- [POSIX – command](https://pubs.opengroup.org/onlinepubs/9799919799/utilities/command.html)
- [Linux man-pages – file(1)](https://man7.org/linux/man-pages/man1/file.1.html)
- [GNU Coreutils – utilidades sha2](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Linux man-pages – find(1)](https://man7.org/linux/man-pages/man1/find.1.html)
- [Microsoft – Get-Content](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-content)
- [Microsoft – Tee-Object](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/tee-object)
- [Microsoft – ConvertFrom-Json](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json)
- [GNU Coreutils – cat](https://www.gnu.org/software/coreutils/manual/html_node/cat-invocation.html)
- [GNU Coreutils – tee](https://www.gnu.org/software/coreutils/manual/html_node/tee-invocation.html)
- [Claude Platform – Admin API](https://platform.claude.com/docs/en/manage-claude/admin-api)
- [Claude Platform – Rate limits](https://platform.claude.com/docs/en/api/rate-limits)
- [Claude Platform – API errors](https://platform.claude.com/docs/en/api/errors)
- [Claude Platform – Batch processing](https://platform.claude.com/docs/en/build-with-claude/batch-processing)
- [Anthropic Privacy Center – Retención de datos comerciales](https://privacy.claude.com/en/articles/7996866-how-long-do-you-store-my-organization-s-data)
- [Anthropic Privacy Center – Zero Data Retention](https://privacy.claude.com/en/articles/8956058-i-have-a-zero-data-retention-agreement-with-anthropic-what-products-does-it-apply-to)
- [Anthropic – Proxy corporativo de Claude Code](https://docs.anthropic.com/en/docs/claude-code/corporate-proxy)
- [Microsoft – Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)
- [Microsoft – Test-NetConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection)
- [Microsoft – Get-NetTCPConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection)
- [BIND 9 – dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Linux man-pages – ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Claude API – Usage Report](https://platform.claude.com/docs/en/api/admin/usage_report)
- [Anthropic – Claude 2](https://www.anthropic.com/news/claude-2)
- [Anthropic – Familia Claude 3](https://www.anthropic.com/news/claude-3-family)
- [Anthropic – Claude 3.7 y Claude Code](https://www.anthropic.com/news/claude-3-7-sonnet)
- [Anthropic – Claude 4 y disponibilidad general de Claude Code](https://www.anthropic.com/news/claude-4)
