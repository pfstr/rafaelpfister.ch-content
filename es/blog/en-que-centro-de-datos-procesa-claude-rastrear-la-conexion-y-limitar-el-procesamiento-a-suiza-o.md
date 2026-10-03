---
title: "¿En qué centro de datos procesa Claude? Rastrear la conexión y limitar el procesamiento a Suiza o la UE"
navTitle: "Centro de datos de Claude"
description: "Qué etapas recorre una solicitud a Claude, por qué el rastro termina en el edge de Cloudflare y qué revela Anthropic sobre sus centros de datos. También se comparan las vías con ubicación de procesamiento fija (Anthropic API, AWS Bedrock, Google Vertex AI, Microsoft Foundry), los límites para operar exclusivamente en Suiza y el coste de la vinculación regional."
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
aiPrompt: |
  Du bist mein Berater für Datenstandorte bei KI-Diensten. Hilf mir Schritt für Schritt zu entscheiden, über welchen Weg (Anthropic API, AWS Bedrock, Google Vertex AI oder Microsoft Foundry) wir Claude nutzen sollen, wenn die Verarbeitung in der Schweiz oder in der EU bleiben muss. Frage mich zuerst nach Anwendungsfall, Datenklassifizierung, benötigtem Modell, erwartetem Token-Volumen pro Monat und bestehendem Cloud-Anbieter. Berechne danach die Monatskosten für den globalen und den regional gebundenen Endpunkt und nenne die konkreten Konfigurationsschritte (Region, Inference Profile, Endpoint, Kontrollmöglichkeiten im Log).
translationOf: claude-rechenzentrum-schweiz-eu
url: https://rafaelpfister.ch/es/blog/en-que-centro-de-datos-procesa-claude-rastrear-la-conexion-y-limitar-el-procesamiento-a-suiza-o
translationSourceHash: dd2606a3781871ddf851af8ceaa22336d00316661e82ca7089325e078f709dc4
translationModel: gpt-5.6-terra
translatedAt: 2026-10-03T09:19:22.899Z
translationReview: required
---

Quien utiliza Claude a través de claude.ai, Claude Code o la API no sabe en qué centro de datos el modelo procesa la solicitud. Solo es visible el primer tramo de la conexión hasta el nodo edge más cercano. Para las empresas que entregan datos personales o contenido confidencial a un modelo de lenguaje, esto no es suficiente: deben poder demostrar ante clientes, asesores de protección de datos o auditoría dónde tiene lugar el procesamiento.

Este artículo muestra hasta dónde puede rastrearse la conexión por cuenta propia, qué se conoce públicamente sobre los centros de datos de Anthropic y mediante qué vías puede limitarse de forma vinculante el procesamiento a la UE o al espacio UE más Suiza. Los precios y las regiones corresponden al estado del 2 de octubre de 2026.

**Enfoque en Suiza:** Es determinante la Ley Federal revisada sobre Protección de Datos (revDSG). La comunicación de datos personales al extranjero está permitida conforme al art. 16 revDSG si el Consejo Federal certifica que el Estado de destino ofrece una protección adecuada (anexo 1 de la Ordenanza sobre Protección de Datos, DSV) o si existen garantías adecuadas, como cláusulas contractuales tipo. Los Estados de la UE y del EEE figuran en esta lista; Estados Unidos, solo para empresas certificadas conforme al Swiss-U.S. Data Privacy Framework. Por ello, una ubicación de procesamiento en la UE simplifica considerablemente la justificación, pero no sustituye las demás obligaciones (contrato de encargado del tratamiento, información a las personas afectadas, seguridad de los datos).

**Nota sobre la UE:** Para las empresas establecidas en la UE se aplican en su lugar los arts. 44 y siguientes del RGPD. La Comisión Europea ha certificado que Suiza cuenta con un nivel adecuado de protección de datos (Decisión de adecuación 2000/518/CE, confirmada en enero de 2024); por tanto, el procesamiento en Zúrich no constituye para las empresas de la UE una transferencia a un tercer país en el sentido problemático.

## Qué etapas recorre una solicitud a Claude

Una solicitud a `claude.ai` o `api.anthropic.com` pasa por tres etapas:

| Etapa | Quién la opera | ¿Es visible para usted? |
|---|---|---|
| Nodo edge (terminación TLS, protección contra abusos) | Cloudflare, en nombre de Anthropic | Sí, la ubicación puede leerse mediante el encabezado |
| Red interna hasta el backend | Cloudflare y Anthropic | No |
| Inferencia (el modelo realiza el cálculo) | Anthropic, con capacidad de cómputo de AWS, Google Cloud y sus propios centros de datos | No, solo indicación general `us` o `global` |

Los nombres de host `claude.ai` y `api.anthropic.com` apuntan ambos a la dirección `160.79.104.10`. Esta pertenece al prefijo `160.79.104.0/23`, registrado a nombre de Anthropic (AS399358) y cuyo origen RPKI válido es Anthropic. Sin embargo, en el enrutamiento global el prefijo se anuncia exclusivamente a través de Cloudflare (AS13335): Anthropic incorpora sus propias direcciones IP a la red anycast de Cloudflare. Así, cada ubicación de Cloudflare en el mundo responde a la misma dirección y la conexión llega al nodo edge más cercano.

## Determinar el propio nodo edge

Cloudflare revela la ubicación que atiende la solicitud en una página de diagnóstico y en el encabezado `cf-ray`. La abreviatura tras la ID de Ray es un código aeroportuario IATA: `ZRH` corresponde a Zúrich, `GVA` a Ginebra, `FRA` a Fráncfort y `MRS` a Marsella.

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
| `-I` | Envía una solicitud HEAD y muestra únicamente los encabezados de respuesta |
| `grep -i` | Busca sin distinguir entre mayúsculas y minúsculas |
| `-E '^(server\|cf-ray):'` | Expresión regular extendida: solo líneas que comienzan por `server:` o `cf-ray:` |

</details>

Desde una red suiza, normalmente cabe esperar `ZRH` o `GVA`. Si la salida a Internet está en otro lugar, por ejemplo en una red corporativa con salida centralizada en el extranjero, el nodo edge responde allí, por ejemplo con `colo=FRA` o `colo=MRS`. Por tanto, la ubicación edge sigue a la salida de su red, no al lugar de procesamiento.

## Por qué el traceroute termina en el edge

Un traceroute muestra el trayecto hasta el nodo edge y no más allá:

```powershell
Test-NetConnection -ComputerName api.anthropic.com -TraceRoute -Hops 20
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-ComputerName api.anthropic.com` | Host de destino de la medición |
| `-TraceRoute` | Determina los saltos de router en el trayecto hacia el destino |
| `-Hops 20` | Número máximo de saltos que se comprueban |

</details>

En Linux y macOS, el equivalente es `traceroute -n api.anthropic.com` (`-n` suprime la resolución DNS de los saltos). En la medición, los últimos saltos antes de `160.79.104.10` se encontraban en el rango de direcciones `162.158.0.0/15`, que pertenece a Cloudflare. Después, el rastro termina: la conexión se termina en el nodo edge y el reenvío al backend se realiza mediante una conexión interna nueva que no es visible desde el exterior.

RIPEstat muestra a quién pertenece el prefijo y a través de qué red se anuncia:

```bash
curl -s "https://stat.ripe.net/data/prefix-overview/data.json?resource=160.79.104.10"
curl -s "https://stat.ripe.net/data/bgp-state/data.json?resource=160.79.104.0/23"
```

En el segundo resultado, todas las rutas AS terminan en `13335 399358`: Cloudflare es el único upstream del prefijo de Anthropic.

Según mi investigación, no existe una medición pública que permita rastrear el trayecto hasta un centro de datos de inferencia, y debido a la terminación en el edge tampoco es posible hacerlo con herramientas de red. Los servicios de medición como llmlatency.dev solo registran el tiempo de respuesta hasta el primer byte en el edge (estado de septiembre de 2026: mediana de 96 ms desde US-Central y de 199 ms desde Alemania). Aunque el tiempo hasta el primer token (Time to First Token) podría compararse desde varios lugares, depende más del modelo, de la longitud del prompt y de la carga que de la distancia. Por tanto, no permite inferir de forma fiable una ubicación.

## Qué se sabe sobre los centros de datos de Anthropic

Hasta ahora, Anthropic no opera ningún centro de datos de inferencia propio asociado públicamente a una dirección. La capacidad de cómputo procede de asociaciones recopiladas por Tim Cadenbach en el blog TCDEV:

| Ubicación / socio | Datos clave | Función conocida |
|---|---|---|
| Project Rainier, New Carlisle (Indiana, EE. UU.), Amazon Web Services | Inaugurado en octubre de 2025, unos 11 000 millones de USD, aproximadamente 500 000 chips Trainium2, más de 2,2 GW previstos | Anthropic es el principal inquilino; principalmente entrenamiento |
| Google Cloud | Acuerdo de octubre de 2025 para hasta 1 millón de TPU, más de 1 GW a partir de 2026 | Regiones no publicadas |
| Centros de datos propios con Fluidstack | Anunciados en noviembre de 2025, 50 000 millones de USD, ubicaciones en Texas y Nueva York, puesta en marcha a partir de 2026 | Aún en construcción |

El entrenamiento y la inferencia no tienen por qué realizarse en el mismo lugar. La inferencia puede ejecutarse en cualquier región de AWS o Google Cloud, y Anthropic no revela qué ubicación procesa la respuesta para un usuario en Suiza. Lo único confirmado es lo indicado en la política de privacidad: la contraparte contractual y responsable para clientes del EEE, Reino Unido y Suiza es Anthropic Ireland, Limited, en Dublín; los datos se transfieren a servidores en Estados Unidos u otros países fuera del EEE sobre la base de cláusulas contractuales tipo.

## Control directamente con Anthropic: solo EE. UU. o global

La API de Claude admite desde los modelos de generación 4.6 el parámetro `inference_geo`. Tiene exactamente dos valores:

| Valor | Efecto | Precio |
|---|---|---|
| `global` (predeterminado) | Inferencia en cualquier región disponible | Precio de lista |
| `us` | Inferencia exclusivamente en EE. UU. | Precio de lista × 1,1 |

No existe un valor para la UE ni para Suiza. Actualmente, la ubicación de almacenamiento del workspace (`workspace geo`) también solo puede establecerse en `us`. La respuesta indica en el campo `usage.inference_geo` qué configuración se aplicó, pero no una región concreta. Para Haiku 4.5 y modelos anteriores, la API responde al parámetro con un error 400.

Para claude.ai y Claude Code con una suscripción (Pro, Max, Team, Enterprise), no existe ninguna configuración de la ubicación de procesamiento. Por tanto, las ofertas directas de Anthropic no permiten limitar el procesamiento ni a la UE ni a Suiza. Esto solo es posible mediante los proveedores cloud que operan Claude en sus propias regiones.

## AWS Bedrock: espacio UE más Suiza desde Zúrich

AWS Bedrock opera Claude sobre infraestructura de AWS. Según AWS, Anthropic no tiene acceso a los prompts, las respuestas ni los registros de los clientes. La región se elige mediante la región de origen de la llamada API y el Inference Profile:

| Tipo de endpoint | Prefijo del ID del modelo | Ubicación de procesamiento |
|---|---|---|
| Global Cross-Region Inference | `global.` | Cualquier región comercial de AWS en el mundo |
| Geographic Cross-Region Inference | `eu.` | Solo regiones dentro de la geografía |
| In-Region | sin prefijo | Solo la región llamada |

Para Suiza, es determinante la región `eu-central-2` (Zúrich). Allí están disponibles Opus 5.5, Sonnet 5.5 y Haiku 4.5, aunque solo como perfil Global o EU y no In-Region. No existe un perfil que procese exclusivamente en Zúrich. Si se llama al perfil EU desde Zúrich, AWS distribuye las solicitudes según la tarjeta de modelo (documentado para Haiku 4.5 y Sonnet 4.6) entre estas regiones:

| Región | Ubicación |
|---|---|
| `eu-central-2` | Zúrich |
| `eu-central-1` | Fráncfort |
| `eu-north-1` | Estocolmo |
| `eu-south-1` | Milán |
| `eu-south-2` | España |
| `eu-west-1` | Irlanda |
| `eu-west-3` | París |

Zúrich solo es un destino posible si la solicitud procede de Zúrich. El procesamiento se mantiene así en el espacio UE más Suiza. Para Opus 5.5 y Sonnet 5.5, la tarjeta de modelo no indica las regiones de destino; la lista real la devuelve la API:

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
| `bedrock get-inference-profile` | Lee la definición de un Inference Profile |
| `--region eu-central-2` | Región de origen Zúrich; la lista de destinos depende de la región de origen |
| `--inference-profile-identifier` | ID del perfil EU para Sonnet 5.5 |
| `--query "models[].modelArn"` | Muestra únicamente los ARN de modelos; la región aparece en cada ARN |

</details>

Por defecto, los datos se almacenan solo en la región de origen. Una excepción son los contenidos retenidos para la detección de abusos: se almacenan en la región de destino. El transporte entre las regiones se realiza cifrado a través de la red de AWS.

### Acreditar la región de procesamiento por solicitud

Bedrock es la única de las vías descritas que registra la región de procesamiento real para cada solicitud. CloudTrail la escribe en la región de origen, en el campo `additionalEventData.inferenceRegion`:

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
| `--region eu-central-2` | Región de origen en la que se registran las llamadas |
| `--lookup-attributes AttributeKey=EventSource,AttributeValue=bedrock.amazonaws.com` | Filtra eventos de Bedrock |
| `--max-results 20` | Limita la salida a 20 eventos |
| `--query "Events[].CloudTrailEvent"` | Muestra únicamente el evento completo como texto JSON |
| `--output text` | Salida sin envoltorio JSON, un evento por línea |
| `jq -r '… \| @tsv'` | Extrae hora, acción y región de procesamiento como línea separada por tabuladores |

</details>

Además, una Service Control Policy (SCP) en AWS Organizations permite bloquear todas las regiones fuera de la UE y Suiza. Si una región de destino de un perfil está bloqueada, la solicitud falla en lugar de recurrir a otra región. Con esta combinación, la ubicación de procesamiento se impone técnicamente y puede demostrarse para cada solicitud.

### Claude Code mediante Bedrock en la UE

Claude Code también puede utilizarse mediante Bedrock. En una región `eu-*`, Claude Code selecciona automáticamente el prefijo `eu.`:

```bash
export CLAUDE_CODE_USE_BEDROCK=1
export AWS_REGION=eu-central-2
claude
```

| Variable | Efecto |
|---|---|
| `CLAUDE_CODE_USE_BEDROCK=1` | Cambia Claude Code de la API de Anthropic a Bedrock |
| `AWS_REGION=eu-central-2` | Región de origen Zúrich; de ello se deriva el perfil EU |

Con `ANTHROPIC_BEDROCK_REGION_PREFIX` puede anularse el prefijo. La autenticación de AWS se realiza mediante los mecanismos habituales (perfil, SSO, variables de entorno). La facturación se realiza a través de la cuenta de AWS, no de una suscripción de Claude.

## Google Vertex AI: solo UE, expresamente sin Suiza

En Google Cloud (Vertex AI, ahora bajo el nombre Gemini Enterprise Agent Platform), la situación es menos favorable para Suiza:

| Endpoint | Modelos (selección) | Ubicación de procesamiento |
|---|---|---|
| `global` | todos | Cualquier región de Google Cloud, sin garantía |
| Multirregión `eu` (`aiplatform.eu.rep.googleapis.com`) | Opus 5.5, Sonnet 5.5, Opus 5, Sonnet 5, Opus 4.8, Opus 4.7 | Solo Estados miembros de la UE |
| Región `europe-west1` (Bélgica) | Haiku 4.5, Sonnet 4.6, Opus 4.6 y anteriores | Según la página del modelo, multirregión Europa |
| Región `europe-west6` (Zúrich) | ninguno | Claude no disponible |

Google excluye expresamente a Suiza del endpoint multirregión de la UE: solo cubre Estados miembros de la UE; el Reino Unido y Suiza no están incluidos. Varios sitios suizos de asesoramiento mencionan `europe-west6` como vía para acceder a Claude en Zúrich; según la tabla oficial de ubicaciones de Google, allí no está disponible ningún modelo Claude. Google no documenta un campo de registro con la región de procesamiento real por solicitud. Para el endpoint global, Google indica que la región de procesamiento no puede controlarse ni determinarse.

Por tanto, Vertex AI es adecuado para el procesamiento exclusivamente en la UE, pero no para el procesamiento en Suiza.

## Microsoft Foundry: actualmente sin opción para la UE

Microsoft ofrece Claude en Foundry en dos variantes. «Hosted on Azure» procesa en infraestructura de Azure, como Global Standard o Data Zone Standard; para Claude, la Data Zone solo existe para Estados Unidos. «Hosted on Anthropic» procesa en infraestructura de Anthropic, y Microsoft advierte de que los datos pueden procesarse fuera de Azure y fuera de la región seleccionada. Anthropic lista Foundry para Europa como «Coming soon». En Microsoft 365 Copilot y Copilot Studio, los modelos de Anthropic están excluidos de la EU Data Boundary y están desactivados por defecto en la UE, la EFTA (incluida Suiza) y el Reino Unido.

## Resumen: qué vía garantiza qué ubicación de procesamiento

| Vía | Solo Suiza | Solo UE | Espacio UE más Suiza | Región demostrable por solicitud |
|---|---|---|---|---|
| claude.ai, Claude Code (suscripción) | no | no | no | no |
| API de Claude con `inference_geo` | no | no | no (solo `us`) | solo `us`/`global` |
| AWS Bedrock, origen `eu-central-2`, perfil `eu.` | no | no | sí | sí (CloudTrail) |
| AWS Bedrock, origen en la UE, perfil `eu.` | no | sí | sí | sí (CloudTrail) |
| Google Vertex AI, endpoint `eu` | no | sí | (solo la parte de la UE) | no |
| Microsoft Foundry | no | no | no | no |

Actualmente, ningún proveedor ofrece una vía que garantice que el procesamiento de Claude se limita a Suiza. Lo más cercano es AWS Bedrock con región de origen Zúrich y perfil EU. Si además debe excluirse que los datos salgan de la UE (por ejemplo, debido a compromisos contractuales con clientes de la UE), puede llamarse al perfil EU desde una región de la UE como Fráncfort; entonces Zúrich deja de ser un destino.

## Cuánto cuesta la vinculación regional

Los tres proveedores cloud y Anthropic cobran un recargo del 10 % frente al endpoint global por vincular a una región o geografía. En Bedrock, los precios se basan en la región de origen; Zúrich, Fráncfort y Norte de Virginia cuestan lo mismo. El enrutamiento entre regiones no genera costes adicionales.

Precios de lista en USD por 1 millón de tokens (entrada / salida):

| Modelo | API de Anthropic, `global` | API de Anthropic, `us` | Bedrock o Vertex, global | Bedrock `eu.` o Vertex `eu` |
|---|---|---|---|---|
| Opus 5.5 | 4.00 / 20.00 | 4.40 / 22.00 | 4.00 / 20.00 | 4.40 / 22.00 |
| Sonnet 5.5 | 2.00 / 10.00 | 2.20 / 11.00 | 2.00 / 10.00 | 2.20 / 11.00 |
| Haiku 4.5 | 1.00 / 5.00 | no disponible | 1.00 / 5.00 | 1.10 / 5.50 |
| Sonnet 4.6 | 3.00 / 15.00 | 3.30 / 16.50 | 3.00 / 15.00 | 3.30 / 16.50 |

En Vertex AI, el precio de la columna UE se aplica a Haiku 4.5 y Sonnet 4.6 en la región `europe-west1`. Un ejemplo de cálculo para una aplicación interna con 50 millones de tokens de entrada y 10 millones de tokens de salida al mes:

| Modelo | Global | Vinculado a la UE | Coste adicional mensual |
|---|---|---|---|
| Sonnet 5.5 | 50 × 2 + 10 × 10 = 200 USD | 220 USD | 20 USD |
| Opus 5.5 | 50 × 4 + 10 × 20 = 400 USD | 440 USD | 40 USD |

Por tanto, la vinculación regional cuesta un 10 % más que el endpoint global. Frente al precio de lista de Anthropic directamente con `inference_geo: "us"`, un perfil EU en Bedrock cuesta lo mismo. Al elegir, pesan más los costes indirectos: hay que implementar y operar una cuenta de AWS o Google Cloud con políticas organizativas, registro y alertas presupuestarias, y las tarifas mensuales fijas de las suscripciones de Claude se sustituyen por una facturación exclusivamente según el consumo. La rentabilidad del cambio depende del volumen; una suscripción tiene un precio fijo por persona, pero no ofrece control sobre la ubicación de procesamiento.

## Conservación y entrenamiento

Además de la ubicación, importa cuánto tiempo se almacenan los datos:

- **API de Anthropic:** Los prompts y las respuestas no se utilizan para entrenamiento mientras no exista consentimiento expreso. Zero Data Retention (ZDR) está disponible previa solicitud por organización y puede combinarse con `inference_geo`. Los contenidos marcados por la detección de abusos se conservan hasta dos años incluso con ZDR. Para los modelos Fable y Mythos, es obligatoria una conservación de 30 días.
- **AWS Bedrock:** Los contenidos no se envían a Anthropic. La conservación puede elegirse por modelo entre `none`, `default` y `aws_review`; Fable requiere `aws_review` con almacenamiento de hasta 30 días para revisión por AWS.
- **Google Vertex AI:** Google utiliza los datos para entrenamiento o ajuste fino solo con consentimiento previo.

## Fuentes

1.  [Claude Platform Docs: Data residency](https://platform.claude.com/docs/en/manage-claude/data-residency): parámetro `inference_geo`, valores `us` y `global`, geografía del workspace, modelos compatibles, campo `usage.inference_geo`.

2.  [Claude Platform Docs: Pricing](https://platform.claude.com/docs/en/about-claude/pricing): precios de lista de la API de Anthropic y factor 1,1 para `inference_geo: "us"`.

3.  [Claude Platform Docs: API and data retention](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention): Zero Data Retention, excepciones para contenidos marcados y para Fable/Mythos.

4.  [Anthropic: Privacy Policy](https://www.anthropic.com/legal/privacy): Anthropic Ireland como responsable para el EEE, Reino Unido y Suiza, transferencia a Estados Unidos basada en cláusulas contractuales tipo.

5.  [Anthropic: Regional compliance](https://claude.com/regional-compliance): resumen de qué plataforma ofrece residencia de datos en qué región; Foundry Europa como «Coming soon».

6.  [Claude Platform Docs: Claude in Amazon Bedrock](https://platform.claude.com/docs/en/build-with-claude/claude-in-amazon-bedrock): tipos de endpoint por región, recargo del 10 % para endpoints regionales.

7.  [AWS: Tarjeta de modelo Claude Sonnet 4.6](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-4-6.html): regiones de destino del perfil EU según la región de origen, incluido Zúrich.

8.  [AWS: Tarjeta de modelo Claude Sonnet 5.5](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-5-5.html): disponibilidad en `eu-central-2` solo como perfil Geo y Global.

9.  [AWS: Geographic cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/geographic-cross-region-inference.html): permanencia de datos en la geografía, ubicación de almacenamiento, bloqueo de regiones de destino mediante SCP.

10.  [AWS: Cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html): campo `additionalEventData.inferenceRegion` en CloudTrail, precio según región de origen.

11.  [AWS: Data protection in Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/data-protection.html): los proveedores de modelos no tienen acceso a prompts, respuestas ni registros.

12.  [AWS: Amazon Bedrock Pricing](https://aws.amazon.com/bedrock/pricing/): precios de lista para perfiles Global y Geo, idénticos para Zúrich, Fráncfort y Norte de Virginia.

13.  [AWS Alps Blog: Cross-region inference for EU data processing in Switzerland](https://aws.amazon.com/blogs/alps/unlocking-ai-flexibility-in-switzerland-a-guide-to-cross-region-inference-for-eu-data-processing-and-model-access/): guía de AWS Suiza para el perfil EU desde Zúrich.

14.  [Claude Code Docs: Amazon Bedrock](https://code.claude.com/docs/en/amazon-bedrock): variables de entorno y prefijo automático `eu.` en regiones de la UE.

15.  [Google Cloud: Generative AI locations](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/locations): tabla de ubicaciones de los modelos Claude, sin disponibilidad en `europe-west6`.

16.  [Google Cloud: Data residency](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/data-residency): multirregión UE solo para Estados miembros de la UE; Suiza y Reino Unido excluidos.

17.  [Google Cloud: Vertex AI Pricing](https://cloud.google.com/vertex-ai/generative-ai/pricing): precios de lista globales, multirregión UE y `europe-west1`.

18.  [Microsoft Learn: Claude models hosting comparison](https://learn.microsoft.com/en-us/azure/foundry/foundry-models/concepts/claude-models-hosting-comparison): Hosted on Azure y Hosted on Anthropic, Data Zone solo en EE. UU.

19.  [Microsoft Learn: Anthropic as AI subprocessor in Microsoft 365 Copilot](https://learn.microsoft.com/en-us/microsoft-365/copilot/connect-to-ai-subprocessor): excepción de la EU Data Boundary, configuración predeterminada en UE/EFTA.

20.  [TCDEV Blog: Where Are Claude's Data Centers?](https://www.tcdev.de/blog/where-are-claudes-data-centers/): resumen de Tim Cadenbach sobre Project Rainier, el acuerdo de TPU de Google y los centros de datos propios con Fluidstack.

21.  [Anthropic: Inversión de 50 000 millones de USD en infraestructura de EE. UU.](https://www.anthropic.com/news/anthropic-invests-50-billion-in-american-ai-infrastructure): anuncio de los centros de datos propios en Texas y Nueva York con Fluidstack.

22.  [Claude Platform Docs: IP addresses](https://platform.claude.com/docs/en/api/ip-addresses): rango de direcciones entrante `160.79.104.0/23`.

23.  [RIPEstat](https://stat.ripe.net/): resumen de prefijo y estado BGP para `160.79.104.0/23` (AS399358, upstream AS13335).

24.  [llmlatency.dev: Anthropic](https://llmlatency.dev/provider/anthropic): tiempos de respuesta en el edge por ubicación de medición.

25.  [Fedlex: Ordenanza sobre Protección de Datos (DSV), anexo 1](https://www.fedlex.admin.ch/eli/cc/2022/568/de): lista de Estados con protección de datos adecuada, incluida UE/EEE y Estados Unidos para empresas certificadas.
