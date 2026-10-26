---
title: "¿En qué centro de datos procesa Claude? Rastrear la conexión y limitar el procesamiento a Suiza o la UE"
navTitle: "Centro de datos de Claude"
description: "Qué etapas recorre una solicitud a Claude, por qué el rastro termina en el edge de Cloudflare y qué revela Anthropic sobre sus centros de datos. También compara las vías con una ubicación de procesamiento fija (API de Anthropic, AWS Bedrock, Google Vertex AI, Microsoft Foundry), los límites de una operación exclusivamente suiza y el coste de la vinculación regional."
date: "2026-10-02"
kategorie: "Claude"
timeToRead: "12 min de lectura"
themen:
  - claude
produkte:
  - "claude"
protokolle:
  - "apis"
  - "tcp"
slug: "en-que-centro-de-datos-procesa-claude-rastrear-la-conexion-y-limitar-el-procesamiento-a-suiza-o"
translationId: "article-d8b7299589ea96f4"
translationOf: claude-rechenzentrum-schweiz-eu
translationSourceHash: 2d2a3c4b5820e6ec99b710813e209ed3b2e6f6fa750028d120a7d5e4bf4e0c17
translationModel: gpt-5.6-terra
translatedAt: 2026-10-26T09:36:35.024Z
translationReview: required
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/en-que-centro-de-datos-procesa-claude-rastrear-la-conexion-y-limitar-el-procesamiento-a-suiza-o
---

Quien utiliza Claude a través de claude.ai, Claude Code o la API no sabe en qué centro de datos el modelo procesa la solicitud. Solo es visible el primer tramo de la conexión hasta el nodo edge más cercano. Para las empresas que transmiten datos personales o contenido confidencial a un modelo de lenguaje, esto no es suficiente: deben poder demostrar ante clientes, asesores de protección de datos o auditoría dónde tiene lugar el procesamiento.

Este artículo muestra hasta dónde puede rastrearse la conexión, qué se conoce públicamente sobre los centros de datos de Anthropic y mediante qué vías puede limitarse de forma vinculante el procesamiento a la UE o al espacio de la UE más Suiza. Los precios y las regiones corresponden al estado del 25 de octubre de 2026.

**Enfoque Suiza:** Es determinante la Ley Federal revisada de Protección de Datos (revDSG). La comunicación de datos personales al extranjero está permitida conforme al art. 16 revDSG si el Consejo Federal certifica un nivel de protección adecuado en el Estado de destino (anexo 1 de la Ordenanza de Protección de Datos, DSV) o si existen garantías adecuadas, como cláusulas contractuales tipo. Los Estados de la UE y del EEE figuran en esta lista; Estados Unidos solo para empresas certificadas conforme al Swiss-U.S. Data Privacy Framework. Por ello, un lugar de procesamiento en la UE simplifica considerablemente la justificación, pero no sustituye las demás obligaciones (contrato de encargo de tratamiento, información a las personas afectadas, seguridad de los datos).

**Nota sobre la UE:** Para las empresas con sede en la UE se aplican en cambio los arts. 44 y siguientes del RGPD. La Comisión Europea ha reconocido a Suiza un nivel adecuado de protección de datos (decisión de adecuación 2000/518/CE, confirmada en enero de 2024); por tanto, el procesamiento en Zúrich no constituye para las empresas de la UE una transferencia a un tercer país en sentido problemático.

<aside class="pilot-card">
  <span class="pilot-card__tag">Nuevo de adeptio · Se buscan empresas piloto</span>
  <p class="pilot-card__titel">Utilice Claude y OpenAI sin que los datos personales salgan de Suiza</p>
  <p class="pilot-card__lead">Nuestra capa intermedia se sitúa entre sus aplicaciones en Suiza y los modelos de Anthropic y OpenAI, y anonimiza cada solicitud antes de que salga del país. Para la operación en producción buscamos empresas que utilicen el producto en su día a día.</p>
  <div class="pilot-card__spalten">
    <div class="pilot-card__spalte">
      <p class="pilot-card__spalte-titel">Su empresa</p>
      <ul>
        <li>Los datos personales solo salen de Suiza anonimizados</li>
        <li>Justificación más sencilla ante protección de datos y auditoría</li>
        <li>Acceso antes de la disponibilidad general y participación en las funcionalidades</li>
        <li>Contacto directo con los desarrolladores</li>
      </ul>
    </div>
    <div class="pilot-card__spalte">
      <p class="pilot-card__spalte-titel">adeptio</p>
      <ul>
        <li>Conclusiones a partir de prompts reales en lugar de datos de prueba</li>
        <li>Métricas de latencia y calidad de respuesta bajo carga real</li>
        <li>Referencias del mercado suizo</li>
      </ul>
    </div>
  </div>
  <div class="pilot-card__aktionen">
    <a class="pilot-card__cta" href="/kontakt">Inscribirse como empresa piloto</a>
    <a class="pilot-card__mehr" href="#anonymisierung-vor-dem-modell-pilotunternehmen-gesucht">Detalles y clasificación jurídica</a>
  </div>
</aside>

## Qué etapas recorre una solicitud a Claude

Una solicitud a `claude.ai` o `api.anthropic.com` pasa por tres etapas:

| Etapa | Quién la opera | ¿Visible para usted? |
|---|---|---|
| Nodo edge (terminación TLS, protección contra abusos) | Cloudflare, en nombre de Anthropic | Sí, la ubicación puede leerse mediante el encabezado |
| Red interna hasta el backend | Cloudflare y Anthropic | No |
| Inferencia (el modelo procesa) | Anthropic, con capacidad de cómputo de AWS, Google Cloud y centros de datos propios | No, solo indicación general `us` o `global` |

Los nombres de host `claude.ai` y `api.anthropic.com` apuntan ambos a la dirección `160.79.104.10`. Pertenece al prefijo `160.79.104.0/23`, registrado a nombre de Anthropic (AS399358) y que, según RPKI, procede válidamente de Anthropic. Sin embargo, en el enrutamiento global el prefijo se anuncia exclusivamente a través de Cloudflare (AS13335): Anthropic incorpora sus propias direcciones IP a la red Anycast de Cloudflare. De este modo, todos los centros de Cloudflare en el mundo responden a la misma dirección, y la conexión llega al nodo edge más cercano.

## Determinar el propio nodo edge

Cloudflare revela la ubicación que atiende la solicitud en una página de diagnóstico y en el encabezado `cf-ray`. La abreviatura detrás del ID Ray es un código aeroportuario IATA: `ZRH` corresponde a Zúrich, `GVA` a Ginebra, `FRA` a Fráncfort y `MRS` a Marsella.

```bash
curl -s https://api.anthropic.com/cdn-cgi/trace
```

En la salida son relevantes las líneas `colo=` (ubicación edge) y `loc=` (país al que Cloudflare asigna su dirección de origen). En Windows, PowerShell ofrece lo mismo:

```powershell
$trace = Invoke-WebRequest -Uri "https://api.anthropic.com/cdn-cgi/trace" -UseBasicParsing
$trace.Content
```

Puede leer el encabezado mediante una solicitud HEAD:

```bash
curl -sI https://api.anthropic.com/ \
  | grep -i -E '^(server|cf-ray):'
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-s` | Suprime el indicador de progreso y los mensajes de error de curl |
| `-I` | Envía una solicitud HEAD y muestra solo los encabezados de respuesta |
| `grep -i` | Busca sin distinguir mayúsculas y minúsculas |
| `-E '^(server\|cf-ray):'` | Expresión regular extendida: solo líneas que empiezan por `server:` o `cf-ray:` |

</details>

Desde una red suiza, normalmente cabe esperar `ZRH` o `GVA`. Si la salida a Internet está en otro lugar, por ejemplo, en una red empresarial con salida centralizada en el extranjero, el nodo edge responderá allí, por ejemplo con `colo=FRA` o `colo=MRS`. Por tanto, la ubicación edge sigue la salida de su red, no el lugar de procesamiento.

## Por qué el traceroute termina en el edge

Un traceroute muestra el camino hasta el nodo edge y no más allá:

```powershell
Test-NetConnection -ComputerName api.anthropic.com -TraceRoute -Hops 20
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-ComputerName api.anthropic.com` | Host de destino de la medición |
| `-TraceRoute` | Determina los saltos de enrutadores en el camino al destino |
| `-Hops 20` | Número máximo de saltos que se comprueban |

</details>

En Linux y macOS, el equivalente es `traceroute -n api.anthropic.com` (`-n` suprime la resolución DNS de los saltos). En la medición, los últimos saltos antes de `160.79.104.10` estaban en el rango de direcciones `162.158.0.0/15`, que pertenece a Cloudflare. Después, el rastro termina: la conexión se termina en el nodo edge y el reenvío al backend se realiza en una nueva conexión interna, no visible desde el exterior.

RIPEstat muestra a quién pertenece el prefijo y a través de qué red se anuncia:

```bash
curl -s "https://stat.ripe.net/data/prefix-overview/data.json?resource=160.79.104.10"
curl -s "https://stat.ripe.net/data/bgp-state/data.json?resource=160.79.104.0/23"
```

En el segundo resultado, todas las rutas AS terminan en `13335 399358`: Cloudflare es el único upstream del prefijo de Anthropic.

Según mi investigación, no existe una medición pública que permita rastrear el camino hasta un centro de datos de inferencia y, debido a la terminación en el edge, tampoco es posible hacerlo con herramientas de red. Servicios de medición como llmlatency.dev solo registran el tiempo de respuesta hasta el primer byte en el edge (a septiembre de 2026: mediana de 96 ms desde US-Central y 199 ms desde Alemania). Aunque podría compararse el tiempo hasta el primer token (Time to First Token) desde varios lugares, este depende más del modelo, de la longitud del prompt y de la carga que de la distancia. Por tanto, no permite extraer una conclusión fiable sobre la ubicación.

## Lo que se sabe sobre los centros de datos de Anthropic

Hasta ahora, Anthropic no opera ningún centro de datos de inferencia propio con una ubicación pública asignada. La capacidad de cómputo procede de asociaciones recopiladas por Tim Cadenbach en el blog TCDEV:

| Ubicación / socio | Datos clave | Función conocida |
|---|---|---|
| Project Rainier, New Carlisle (Indiana, EE. UU.), Amazon Web Services | Inaugurado en octubre de 2025, unos 11 000 millones de USD, aproximadamente 500 000 chips Trainium2, más de 2,2 GW previstos | Anthropic es el principal inquilino; principalmente entrenamiento |
| Google Cloud | Acuerdo de octubre de 2025 para hasta 1 millón de TPU, más de 1 GW a partir de 2026 | Regiones no publicadas |
| Centros de datos propios con Fluidstack | Anunciados en noviembre de 2025, 50 000 millones de USD, ubicaciones en Texas y Nueva York, puesta en servicio a partir de 2026 | Aún en construcción |

El entrenamiento y la inferencia no tienen necesariamente lugar en el mismo sitio. La inferencia puede ejecutarse en cualquier región de AWS o Google Cloud, y Anthropic no revela qué ubicación procesa la respuesta para un usuario en Suiza. Solo está garantizado lo que establece la política de privacidad: la parte contratante y responsable para clientes del EEE, Reino Unido y Suiza es Anthropic Ireland, Limited, en Dublín; los datos se transfieren a servidores en Estados Unidos u otros países fuera del EEE, sobre la base de cláusulas contractuales tipo.

## Control directamente en Anthropic: solo EE. UU. o global

La API de Claude conoce, desde los modelos de la generación 4.6, el parámetro `inference_geo`. Tiene exactamente dos valores:

| Valor | Efecto | Precio |
|---|---|---|
| `global` (predeterminado) | Inferencia en cualquier región disponible | Precio de lista |
| `us` | Inferencia exclusivamente en Estados Unidos | Precio de lista × 1,1 |

No existe un valor para la UE ni para Suiza. Actualmente, la ubicación de almacenamiento del workspace (`workspace geo`) solo puede establecerse en `us`. La respuesta indica en el campo `usage.inference_geo` qué configuración se ha aplicado, pero no una región concreta. Para Haiku 4.5 y modelos anteriores, la API responde al parámetro con un error 400.

Para claude.ai y Claude Code con una suscripción (Pro, Max, Team, Enterprise) no existe ninguna configuración de la ubicación de procesamiento. Por tanto, a través de las ofertas directas de Anthropic no puede limitarse el procesamiento ni a la UE ni a Suiza. Esto solo es posible mediante los proveedores cloud que operan Claude en sus propias regiones.

## AWS Bedrock: espacio UE más Suiza desde Zúrich

AWS Bedrock opera Claude sobre infraestructura de AWS. Según AWS, Anthropic no tiene acceso a los prompts, respuestas ni registros de los clientes. La región se elige mediante la región de origen de su llamada API y el perfil de inferencia:

| Tipo de endpoint | Prefijo del ID de modelo | Lugar de procesamiento |
|---|---|---|
| Global Cross-Region Inference | `global.` | Cualquier región comercial de AWS en todo el mundo |
| Geographic Cross-Region Inference | `eu.` | Solo regiones dentro de la geografía |
| In-Region | sin prefijo | Solo la región llamada |

Para Suiza, es determinante la región `eu-central-2` (Zúrich). Allí están disponibles Opus 5.5, Sonnet 5.5, Haiku 4.5 y, desde el 7 de octubre de 2026, Haiku 5.5, aunque solo como perfil Global o EU y no In-Region. No existe un perfil que procese exclusivamente en Zúrich. Si se llama al perfil EU desde Zúrich, AWS distribuye las solicitudes, según la tarjeta de modelo (documentado para Opus 5.5, Sonnet 5.5, Haiku 5.5, Haiku 4.5 y Sonnet 4.6), entre estas regiones:

| Región | Ubicación |
|---|---|
| `eu-central-2` | Zúrich |
| `eu-central-1` | Fráncfort |
| `eu-north-1` | Estocolmo |
| `eu-south-1` | Milán |
| `eu-south-2` | España |
| `eu-west-1` | Irlanda |
| `eu-west-3` | París |

Zúrich solo es un posible destino si la solicitud procede de Zúrich. Por tanto, el procesamiento se mantiene en el espacio de la UE más Suiza. La lista vinculante es la que la API devuelve para la propia región de origen:

```bash
aws bedrock get-inference-profile \
  --region eu-central-2 \
  --inference-profile-identifier eu.anthropic.claude-sonnet-5-5 \
  --query "models[].modelArn"
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `bedrock get-inference-profile` | Lee la definición de un perfil de inferencia |
| `--region eu-central-2` | Región de origen Zúrich; la lista de destinos depende de la región de origen |
| `--inference-profile-identifier` | ID del perfil EU para Sonnet 5.5 |
| `--query "models[].modelArn"` | Muestra solo los ARN de modelo; la región aparece en cada ARN |

</details>

De forma predeterminada, los datos solo se almacenan en la región de origen. Una excepción son los contenidos retenidos para la detección de abusos: se guardan en la región de destino. El transporte entre regiones se realiza cifrado a través de la red de AWS.

### Documentar la región de procesamiento por solicitud

Bedrock es la única de las vías descritas que registra la región de procesamiento efectiva para cada solicitud. CloudTrail la escribe en la región de origen, en el campo `additionalEventData.inferenceRegion`:

```bash
aws cloudtrail lookup-events \
  --region eu-central-2 \
  --lookup-attributes AttributeKey=EventSource,AttributeValue=bedrock.amazonaws.com \
  --max-results 20 \
  --query "Events[].CloudTrailEvent" \
  --output text \
  | jq -r '[.eventTime, .eventName, .additionalEventData.inferenceRegion] | @tsv'
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `cloudtrail lookup-events` | Busca eventos de CloudTrail de los últimos 90 días |
| `--region eu-central-2` | Región de origen donde se registran las llamadas |
| `--lookup-attributes AttributeKey=EventSource,AttributeValue=bedrock.amazonaws.com` | Filtra por eventos de Bedrock |
| `--max-results 20` | Limita la salida a 20 eventos |
| `--query "Events[].CloudTrailEvent"` | Muestra solo el evento completo como texto JSON |
| `--output text` | Salida sin envoltura JSON, un evento por línea |
| `jq -r '… \| @tsv'` | Extrae fecha y hora, acción y región de procesamiento como línea separada por tabuladores |

</details>

Además, una Service Control Policy (SCP) en AWS Organizations permite bloquear todas las regiones fuera de la UE y Suiza. Si una región de destino de un perfil está bloqueada, la solicitud falla en lugar de recurrir a otra región. Con esta combinación, el lugar de procesamiento queda impuesto técnicamente y puede demostrarse para cada solicitud.

### Claude Code a través de Bedrock en la UE

Claude Code también puede operarse a través de Bedrock. En una región `eu-*`, Claude Code elige automáticamente el prefijo `eu.`:

```bash
export CLAUDE_CODE_USE_BEDROCK=1
export AWS_REGION=eu-central-2
claude
```

| Variable | Efecto |
|---|---|
| `CLAUDE_CODE_USE_BEDROCK=1` | Cambia Claude Code de la API de Anthropic a Bedrock |
| `AWS_REGION=eu-central-2` | Región de origen Zúrich; de ello resulta el perfil EU |

Con `ANTHROPIC_BEDROCK_REGION_PREFIX` puede sobrescribirse el prefijo. La autenticación de AWS se realiza mediante los mecanismos habituales (perfil, SSO, variables de entorno). La facturación se realiza a través de la cuenta de AWS, no mediante una suscripción de Claude.

## Google Vertex AI: solo UE, expresamente sin Suiza

En Google Cloud (Vertex AI, ahora bajo el nombre Gemini Enterprise Agent Platform), la situación es menos favorable para Suiza:

| Endpoint | Modelos (selección) | Lugar de procesamiento |
|---|---|---|
| `global` | todos | Cualquier región de Google Cloud, sin garantía |
| Multirregión `eu` (`aiplatform.eu.rep.googleapis.com`) | Opus 5.5, Sonnet 5.5, Haiku 5.5, Opus 5, Sonnet 5, Opus 4.8, Opus 4.7 | Solo Estados miembros de la UE |
| Región `europe-west1` (Bélgica) | Haiku 4.5, Sonnet 4.6, Opus 4.6 y anteriores | Según la página del modelo, multirregión Europa |
| Región `europe-west6` (Zúrich) | ninguno | Claude no disponible |

Google excluye expresamente a Suiza del endpoint multirregión de la UE: solo cubre Estados miembros de la UE; el Reino Unido y Suiza no están incluidos. Varios sitios de asesoramiento suizos mencionan `europe-west6` como vía para usar Claude en Zúrich; según la tabla oficial de ubicaciones de Google, allí no hay ningún modelo Claude disponible. Google no documenta ningún campo de registro con la región de procesamiento efectiva por solicitud. Para el endpoint global, Google indica que la región de procesamiento no puede controlarse ni determinarse.

Por tanto, Vertex AI es adecuado para un procesamiento exclusivamente en la UE, pero no para un procesamiento en Suiza.

## Microsoft Foundry: actualmente sin opción para la UE

Microsoft ofrece Claude en Foundry en dos variantes. «Hosted on Azure» procesa en infraestructura de Azure, como Global Standard o Data Zone Standard; Data Zone solo está disponible para Claude en Estados Unidos. «Hosted on Anthropic» procesa en infraestructura de Anthropic, y Microsoft señala que los datos pueden procesarse fuera de Azure y fuera de la región seleccionada. Anthropic incluye Foundry para Europa como «Coming soon». En Microsoft 365 Copilot y Copilot Studio, los modelos de Anthropic están excluidos de la EU Data Boundary y están desactivados de forma predeterminada en la UE, la EFTA (incluida Suiza) y el Reino Unido.

## Resumen: qué vía garantiza qué lugar de procesamiento

| Vía | Solo Suiza | Solo UE | Espacio UE más Suiza | Región demostrable por solicitud |
|---|---|---|---|---|
| claude.ai, Claude Code (suscripción) | no | no | no | no |
| API de Claude con `inference_geo` | no | no | no (solo `us`) | solo `us`/`global` |
| AWS Bedrock, origen `eu-central-2`, perfil `eu.` | no | no | sí | sí (CloudTrail) |
| AWS Bedrock, origen en la UE, perfil `eu.` | no | sí | sí | sí (CloudTrail) |
| Google Vertex AI, endpoint `eu` | no | sí | (solo parte de la UE) | no |
| Microsoft Foundry | no | no | no | no |

Actualmente, ningún proveedor ofrece una vía que garantice que el procesamiento de Claude se limita a Suiza. AWS Bedrock con región de origen Zúrich y perfil EU es la opción más cercana. Si además debe excluirse que los datos salgan de la UE —por ejemplo, debido a compromisos contractuales con clientes de la UE—, el perfil EU puede invocarse desde una región de la UE como Fráncfort; entonces Zúrich deja de ser destino.

## Anonimización antes del modelo: se buscan empresas piloto

En la mayoría de los casos, el lugar de procesamiento solo es relevante porque el prompt contiene datos personales o información confidencial. Aquí es donde entra un nuevo producto de mi empresa, adeptio gmbh: actúa como capa intermedia entre las aplicaciones en Suiza y los modelos de Anthropic y OpenAI, y anonimiza las solicitudes antes de que salgan de Suiza. El modelo solo recibe contenido sin referencia personal, independientemente del centro de datos donde procese.

**Enfoque Suiza:** La información que ya no puede atribuirse a una persona determinada o determinable no constituye datos personales en el sentido del art. 5, letra a, revDSG. Las normas sobre comunicación al extranjero (arts. 16 y siguientes revDSG) no se aplican a esos contenidos. Que una anonimización alcance ese umbral depende de la implementación y de los datos concretos; precisamente eso queremos demostrar en la operación en producción.

Para la introducción, buscamos empresas que utilicen Claude o modelos de OpenAI en producción, o que planifiquen su implantación, y que empleen el producto como empresa piloto en su día a día. Ambas partes se benefician de ello:

| Su empresa | adeptio |
|---|---|
| Los datos personales y la información confidencial solo salen de Suiza anonimizados. La justificación ante asesores de protección de datos, clientes y auditoría se simplifica. | Conclusiones a partir de prompts y flujos de trabajo reales en lugar de datos de prueba: qué información debe detectarse y dónde la anonimización elimina demasiado o demasiado poco. |
| La elección entre Anthropic, OpenAI, Bedrock o Vertex AI vuelve a basarse en la oferta de modelos y el precio, en lugar de únicamente en el lugar de procesamiento. | Métricas de latencia y calidad de respuesta bajo carga real. |
| Acceso al producto antes de la disponibilidad general e influencia sobre qué tipos de datos y casos de uso cubre primero. | Comentarios de protección de datos y auditoría que orientan las evidencias del producto. |
| Contacto directo con los desarrolladores en lugar de soporte mediante tickets. | Referencias del mercado suizo para la disponibilidad general. |

<aside class="offer-box">
  <span class="offer-box__tag">Se buscan empresas piloto</span>
  <p><strong>¿Utiliza Claude u OpenAI con datos personales?</strong> Utilice la capa de anonimización de adeptio en producción y contribuya a definir su evolución.</p>
  <a class="offer-box__cta" href="/kontakt">Contactar</a>
</aside>

## El coste de la vinculación regional

Los tres proveedores cloud y la propia Anthropic cobran un recargo del 10 % frente al endpoint global por la vinculación a una región o geografía. En Bedrock, los precios dependen de la región de origen; Zúrich, Fráncfort y North Virginia tienen el mismo coste. El enrutamiento entre regiones no conlleva costes adicionales.

Precios de lista en USD por 1 millón de tokens (entrada / salida):

| Modelo | API de Anthropic, `global` | API de Anthropic, `us` | Bedrock o Vertex, global | Bedrock `eu.` o Vertex `eu` |
|---|---|---|---|---|
| Opus 5.5 | 4.00 / 20.00 | 4.40 / 22.00 | 4.00 / 20.00 | 4.40 / 22.00 |
| Sonnet 5.5 | 2.00 / 10.00 | 2.20 / 11.00 | 2.00 / 10.00 | 2.20 / 11.00 |
| Haiku 5.5 (prompts de hasta 100 000 tokens) | 0.10 / 0.50 | 0.11 / 0.55 | 0.10 / 0.50 | 0.11 / 0.55 |
| Haiku 4.5 | 1.00 / 5.00 | no disponible | 1.00 / 5.00 | 1.10 / 5.50 |
| Sonnet 4.6 | 3.00 / 15.00 | 3.30 / 16.50 | 3.00 / 15.00 | 3.30 / 16.50 |

En Vertex AI, el precio de la columna UE se aplica a Haiku 4.5 y Sonnet 4.6 en la región `europe-west1`. Un ejemplo de cálculo para una aplicación interna con 50 millones de tokens de entrada y 10 millones de tokens de salida al mes:

| Modelo | Global | Vinculado a la UE | Coste adicional mensual |
|---|---|---|---|
| Sonnet 5.5 | 50 × 2 + 10 × 10 = 200 USD | 220 USD | 20 USD |
| Opus 5.5 | 50 × 4 + 10 × 20 = 400 USD | 440 USD | 40 USD |

Por tanto, la vinculación regional cuesta un 10 % más que el endpoint global. Frente al precio de lista de Anthropic directamente con `inference_geo: "us"`, un perfil EU en Bedrock cuesta lo mismo. En la elección pesan más bien los costes indirectos: debe configurarse y operarse una cuenta de AWS o Google Cloud con políticas organizativas, registro y alertas presupuestarias, y los precios mensuales fijos de las suscripciones de Claude se sustituyen por una facturación puramente por consumo. Que el cambio resulte rentable depende del volumen; una suscripción tiene un precio fijo por persona, pero no ofrece control sobre el lugar de procesamiento.

## Retención y entrenamiento

Además del lugar, importa cuánto tiempo se almacenan los datos:

- **API de Anthropic:** Los prompts y las respuestas no se utilizan para entrenamiento mientras no exista consentimiento explícito. Zero Data Retention (ZDR) está disponible bajo solicitud por organización y puede combinarse con `inference_geo`. Los contenidos marcados por la detección de abusos se conservan hasta dos años incluso con ZDR. Para los modelos Fable y Mythos, es obligatoria una retención de 30 días.
- **AWS Bedrock:** El contenido no se envía a Anthropic. La retención puede elegirse por modelo entre `none`, `default` y `aws_review`; Fable exige `aws_review` con almacenamiento de hasta 30 días para revisión por AWS.
- **Google Vertex AI:** Google utiliza los datos para entrenamiento o ajuste fino solo con consentimiento previo.

## Historial de cambios

- **25 de octubre de 2026:** Añadido: Haiku 5.5 (desde el 7 de octubre de 2026), con disponibilidad en Zúrich y en la multirregión UE de Google Cloud, así como precios; las regiones de destino del perfil EU ahora también están documentadas para Opus 5.5 y Sonnet 5.5; estado de precios y regiones a 25 de octubre de 2026. Enlaces de fuentes actualizados a las nuevas direcciones de la documentación de Google Cloud.

## Fuentes

1.  [Claude Platform Docs: Data residency](https://platform.claude.com/docs/en/manage-claude/data-residency): parámetro `inference_geo`, valores `us` y `global`, geo del workspace, modelos compatibles, campo `usage.inference_geo`.

2.  [Claude Platform Docs: Pricing](https://platform.claude.com/docs/en/about-claude/pricing): precios de lista de la API de Anthropic y factor 1,1 para `inference_geo: "us"`.

3.  [Claude Platform Docs: API and data retention](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention): Zero Data Retention, excepciones para contenido marcado y para Fable/Mythos.

4.  [Anthropic: Privacy Policy](https://www.anthropic.com/legal/privacy): Anthropic Ireland como responsable para el EEE, Reino Unido y Suiza, transferencia a Estados Unidos sobre la base de cláusulas contractuales tipo.

5.  [Anthropic: Regional compliance](https://claude.com/regional-compliance): resumen de qué plataforma ofrece residencia de datos en qué región; Foundry Europa como «Coming soon».

6.  [Claude Platform Docs: Claude in Amazon Bedrock](https://platform.claude.com/docs/en/build-with-claude/claude-in-amazon-bedrock): tipos de endpoint por región, recargo del 10 % para endpoints regionales.

7.  [AWS: Tarjeta de modelo Claude Sonnet 4.6](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-4-6.html): regiones de destino del perfil EU por región de origen, incluido Zúrich.

8.  [AWS: Tarjeta de modelo Claude Sonnet 5.5](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-5-5.html): disponibilidad en `eu-central-2` solo como perfil Geo y Global, regiones de destino del perfil EU desde Zúrich.

9.  [AWS: Geographic cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/geographic-cross-region-inference.html): permanencia de datos en la geografía, ubicación de almacenamiento, bloqueo de regiones de destino mediante SCP.

10.  [AWS: Cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html): campo `additionalEventData.inferenceRegion` en CloudTrail, precio según región de origen.

11.  [AWS: Data protection in Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/data-protection.html): los proveedores de modelos no tienen acceso a prompts, respuestas ni registros.

12.  [AWS: Amazon Bedrock Pricing](https://aws.amazon.com/bedrock/pricing/): precios de lista para perfiles Global y Geo, idénticos para Zúrich, Fráncfort y North Virginia.

13.  [AWS Alps Blog: Cross-region inference for EU data processing in Switzerland](https://aws.amazon.com/blogs/alps/unlocking-ai-flexibility-in-switzerland-a-guide-to-cross-region-inference-for-eu-data-processing-and-model-access/): guía de AWS Suiza sobre el perfil EU desde Zúrich.

14.  [Claude Code Docs: Amazon Bedrock](https://code.claude.com/docs/en/amazon-bedrock): variables de entorno y prefijo automático `eu.` en regiones de la UE.

15.  [Google Cloud: Generative AI locations](https://docs.cloud.google.com/gemini-enterprise-agent-platform/resources/locations): tabla de ubicaciones de modelos Claude, sin disponibilidad en `europe-west6`.

16.  [Google Cloud: Data residency](https://docs.cloud.google.com/gemini-enterprise-agent-platform/resources/data-residency): multirregión UE solo para Estados miembros de la UE; Suiza y Reino Unido excluidos.

17.  [Google Cloud: Vertex AI Pricing](https://cloud.google.com/gemini-enterprise-agent-platform/generative-ai/pricing): precios de lista globales, multirregión UE y `europe-west1`.

18.  [Microsoft Learn: Claude models hosting comparison](https://learn.microsoft.com/en-us/azure/foundry/foundry-models/concepts/claude-models-hosting-comparison): Hosted on Azure y Hosted on Anthropic, Data Zone solo EE. UU.

19.  [Microsoft Learn: Anthropic as AI subprocessor in Microsoft 365 Copilot](https://learn.microsoft.com/en-us/microsoft-365/copilot/connect-to-ai-subprocessor): excepción de la EU Data Boundary, configuración predeterminada en UE/EFTA.

20.  [TCDEV Blog: Where Are Claude's Data Centers?](https://www.tcdev.de/blog/where-are-claudes-data-centers/): resumen de Tim Cadenbach sobre Project Rainier, el acuerdo de TPU con Google y los centros de datos propios con Fluidstack.

21.  [Anthropic: Inversión de 50 000 millones de USD en infraestructura estadounidense](https://www.anthropic.com/news/anthropic-invests-50-billion-in-american-ai-infrastructure): anuncio de los centros de datos propios en Texas y Nueva York con Fluidstack.

22.  [Claude Platform Docs: IP addresses](https://platform.claude.com/docs/en/api/ip-addresses): rango de direcciones entrantes `160.79.104.0/23`.

23.  [RIPEstat](https://stat.ripe.net/): resumen de prefijo y estado BGP para `160.79.104.0/23` (AS399358, upstream AS13335).

24.  [llmlatency.dev: Anthropic](https://llmlatency.dev/provider/anthropic): tiempos de respuesta en el edge por ubicación de medición.

25.  [Fedlex: Ordenanza de Protección de Datos (DSV), anexo 1](https://www.fedlex.admin.ch/eli/cc/2022/568/de): lista de Estados con protección de datos adecuada, incluida UE/EEE y Estados Unidos para empresas certificadas.

26.  [Anthropic: Claude Haiku 5.5](https://www.anthropic.com/news/claude-haiku-5-5): lanzamiento el 7 de octubre de 2026, disponibilidad en AWS, Google Cloud y Microsoft, precios con tramo a partir de 100 000 tokens.

27.  [AWS: Tarjeta de modelo Claude Haiku 5.5](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-haiku-5-5.html): disponibilidad en `eu-central-2` como perfil Geo y Global, y regiones de destino del perfil EU desde Zúrich.
