---
title: "Hasta 10 millones de tokens gratis al día: aprovecha el programa de intercambio de datos de OpenAI con guardarraíles de costes"
navTitle: "Tokens gratis de OpenAI"
description: "OpenAI concede a las organizaciones que comparten su tráfico de API para entrenamiento una cuota diaria gratuita: hasta 10 millones de tokens según el nivel. Con saldo prepago, límites de proyecto y un presupuesto de tokens en el código, el uso sigue siendo gratuito de forma permanente."
date: "2026-08-27"
kategorie: "API de OpenAI"
timeToRead: "9 min de lectura"
themen:
  - openai-api
produkte:
  - "openai"
protokolle:
  - "apis"
  - "lizenzierung"
slug: "hasta-10-millones-de-tokens-gratis-al-dia-aprovecha-el-programa-de-intercambio-de-datos-de"
translationId: "article-dde41cbe2dd858e6"
aiPrompt: |
  Du bist mein Assistent für die OpenAI-Plattform. Prüfe mit mir Schritt für Schritt, ob mein OpenAI-Konto für das Data-Sharing-Programm mit Gratis-Tokens sauber abgesichert ist: 1) Billing: Prepaid-Guthaben statt Rechnung, Auto-Reload aus. 2) Data controls → Sharing: "Share inputs and outputs" nur für ein dediziertes Projekt aktiviert, Enrollment-Hinweis sichtbar. 3) Projekt: eigenes Spend-Limit gesetzt, nur ein restricted API-Key. 4) Limits: Spend-Alerts konfiguriert. 5) Code: tägliches Token-Budget deutlich unter Gratis-Kontingent und Tages-Rate-Limit. Frage mich nach meinem Usage-Tier und Modell und rechne mir mein Gratis-Kontingent aus.
translationOf: openai-gratis-tokens-data-sharing
url: https://rafaelpfister.ch/es/blog/hasta-10-millones-de-tokens-gratis-al-dia-aprovecha-el-programa-de-intercambio-de-datos-de
translationSourceHash: 0f0fef78a8ab264b755061045a34cc765916b1f1b433473f99a5eb6e0538a6b0
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:44:02.709Z
translationReview: automatic
---

# Hasta 10 millones de tokens gratis al día: aprovecha el programa de intercambio de datos de OpenAI con guardarraíles de costes

OpenAI paga los datos de entrenamiento con capacidad de cómputo en lugar de dinero: desde diciembre de 2024, las organizaciones que comparten sus entradas y salidas de API para entrenamiento reciben una cuota diaria de tokens gratis. Según el nivel de uso y el grupo de modelos, esta oscila entre 250'000 y 10 millones de tokens al día. Para muchas automatizaciones pequeñas, esto basta por completo: una traducción nocturna por lotes, una tarea cron de clasificación o el etiquetado automático de un archivo público permanecen así gratis de forma permanente.

Para que siga siendo gratis, hacen falta límites, y en el lugar correcto. Un contador de tokens en el propio código es una función de comodidad; solo son vinculantes los límites que OpenAI aplica por sí mismo.

## El programa: tokens a cambio de datos de entrenamiento

La participación se gestiona mediante la opción **Share inputs and outputs with OpenAI** en *Settings → Data controls → Sharing*. Solo el propietario de la organización puede modificarla, ya sea para toda la organización o para proyectos individuales. Quienes cumplan los requisitos del programa verán en esta página el aviso "You're eligible for free daily usage on traffic shared with OpenAI"; tras activarlo, cambia a "You're enrolled for complimentary daily tokens". Si no aparece el aviso, la organización no cumple actualmente los requisitos para participar. Las cuentas con Zero Data Retention y los contratos Enterprise están excluidos del intercambio de entradas y salidas.

La cuota depende del nivel de uso de la organización y se calcula por grupo de modelos:

| Grupo de modelos | Nivel 1–2 | Nivel 3–5 |
|---|---|---|
| Modelos grandes (incluidos gpt-5.6-sol, gpt-5.x, serie o, gpt-4.1, gpt-4o) | 250'000 tokens/día | 1 millón de tokens/día |
| Modelos pequeños (incluidos gpt-5.6-terra, gpt-5.6-luna y variantes mini y nano) | 2,5 millones de tokens/día | 10 millones de tokens/día |

Las reglas más importantes en detalle:

- Se cuentan conjuntamente los tokens de entrada y salida, compartidos entre todos los modelos de un grupo. El contador se reinicia cada día a las 00:00 UTC.
- Quedan excluidos los modelos ajustados, el entrenamiento de ajuste fino, las evaluaciones y el uso de herramientas.
- La cuenta necesita un saldo positivo; de lo contrario, tampoco funcionan los tokens gratis.
- OpenAI se reserva el derecho de finalizar el programa con 30 días de preaviso.

La regla de facturación más importante: la solicitud que supera la cuota diaria se factura **íntegramente** a la tarifa normal, no solo la parte excedente. Quien esté en 975'000 de 1 millón de tokens y envíe una solicitud de 30'000 tokens paga los 30'000 completos. Para planificar el presupuesto propio, esto significa: prever un margen de seguridad, no optimizar hasta agotar la cuota.

## Lo que se entrega a cambio

La contraprestación es inequívoca: todas las entradas y salidas de los proyectos compartidos pasan a OpenAI y pueden utilizarse para entrenar modelos futuros. Esto excluye categorías enteras de casos de uso. Los datos de clientes, tickets de soporte, documentos internos, código con detalles de configuración y todo aquello que contenga datos personales no deben llegar a un proyecto compartido; para las empresas suizas, la LPD revisada ya establece aquí el límite, antes incluso de hablar de la confidencialidad frente a los clientes.

Son adecuados los flujos de trabajo con datos que ya son públicos. Un ejemplo es la traducción nocturna de un blog público a varios idiomas: los artículos están en internet, cualquier rastreador ya puede leerlos hoy y las traducciones también se publican. En tal caso, compartirlos no revela nada que no se hubiera revelado ya. Otros candidatos son textos alternativos para un archivo público de imágenes, el etiquetado de documentación de código abierto o resúmenes de notas de versión públicas para un registro de cambios.

## Configurar guardarraíles de costes en la cuenta de OpenAI

El orden se ha elegido deliberadamente: primero van los límites que OpenAI aplica en el servidor. Funcionan incluso si el código propio tiene un error, una tarea cron se ejecuta dos veces o una clave cae en manos equivocadas.

**Saldo prepago, recarga automática desactivada.** Configura la facturación como "Pay as you go" con saldo prepagado y desactiva la recarga automática. De este modo, el daño máximo queda limitado al saldo restante: si se agota, la API rechaza nuevas solicitudes. Como el programa requiere un saldo positivo, hace falta una pequeña base; entre 5 y 10 dólares bastan y, con un funcionamiento correcto, permanecen intactos. Este paso es el único que realmente lo detiene todo en el peor de los casos, por eso va en primer lugar.

**Un proyecto dedicado para el tráfico compartido.** Configura el intercambio como "Enabled for selected projects" y comparte solo un proyecto creado expresamente para ello. Todos los demás proyectos de la organización quedan excluidos del entrenamiento, y el tráfico accidental de otras aplicaciones no termina ni en el conjunto de datos de entrenamiento ni en el presupuesto equivocado.

**Establecer un límite de gasto de proyecto bajo.** Los proyectos tienen su propio límite mensual de gasto, y este es estricto: las solicitudes fallan en cuanto se alcanza. Para un proyecto que previsiblemente cuesta 0 dólares, puede ser muy bajo; 5 dólares bastan como reserva por si una ejecución individual supera la cuota gratuita. El límite a nivel de organización, en cambio, está pensado como techo máximo con alertas; los umbrales de advertencia, por ejemplo al 90 y al 100 por ciento, envían correos electrónicos.

**Una clave restringida por proyecto, solo como secreto de CI.** La clave de API se crea en el proyecto, no a nivel de organización, y recibe únicamente los permisos que necesita el flujo de trabajo. Para un flujo de CI, esto significa: exactamente una clave con permisos restringidos, guardada como secreto en el entorno de CI. No aparece en ningún repositorio, shell local ni segundo servicio.

**Elegir un modelo del grupo económico.** La diferencia entre los grupos es un factor de 10. Quien trabaja en el nivel 1 dispone de 2,5 millones de tokens al día con un modelo del grupo pequeño, en lugar de 250'000. Para tareas estructuradas como traducción, clasificación o extracción, el grupo pequeño suele ser suficiente.

## La segunda línea de defensa en el código

Los límites de la cuenta evitan daños financieros, pero provocan errores duros: al alcanzar un límite de gasto, la ejecución se interrumpe a mitad del lote. Por ello, quien quiera mantenerse limpiamente dentro de la cuota gratuita puede llevar un recuento adicional. Un sencillo contador diario ha demostrado ser eficaz, configurado por ejemplo así:

```json
{
  "openai": {
    "model": "gpt-5.6-terra",
    "reasoningEffort": "none",
    "maxOutputTokens": 32000,
    "dailyTokenBudget": 1000000
  }
}
```

El mecanismo se basa en cuatro reglas:

- Tras cada respuesta, la tarea suma los `input_tokens` y `output_tokens` comunicados por la API a un contador en un archivo de estado. No hay estimación ni segunda consulta, solo los datos de uso de la propia respuesta.
- Antes de cada solicitud, comprueba el presupuesto restante. Si ya no alcanza con seguridad para una respuesta completa, la ejecución termina normalmente con el motivo de parada `token-budget` en lugar de con un error.
- El contador trabaja con días naturales UTC y, por tanto, está sincronizado con el reinicio de la cuota gratuita a las 00:00 UTC.
- Independientemente del presupuesto, el número de llamadas a la API por ejecución está limitado, para que ni siquiera una serie de intentos fallidos pueda agotar la cuota. Los errores de transporte y de cuota interrumpen la ejecución sin reintentos automáticos.

El presupuesto de este ejemplo, de 1 millón, está deliberadamente muy por debajo de la cuota de 2,5 millones. El margen se debe a dos particularidades de la facturación. En primer lugar, el contador no conoce de antemano el tamaño de la siguiente solicitud; por ello, un presupuesto calculado al límite puede superarse por el tamaño de una solicitud, y precisamente esa solicitud se facturaría íntegramente según la regla descrita anteriormente. En segundo lugar, los límites de tasa diarios (TPD) están, según el nivel y el modelo, por debajo de la cuota gratuita; un presupuesto por encima del límite TPD nunca se alcanzaría de forma normal, porque antes la API rechazaría con HTTP 429.

## Control: el panel debe mostrar 0.00

El panel de uso de la plataforma muestra si el cálculo cuadra. Bastan dos vistas:

- La vista **Usage** cuenta todos los tokens, incluidos los facturados gratis. Aquí figura el consumo completo del flujo de trabajo.
- La vista **Costs** (y el campo "Monthly spend" en la lista de proyectos) muestra solo los tokens de pago. Aquí debe aparecer permanentemente 0.00.

Quien quiera saberlo con más precisión puede agrupar la vista Usage por *Service tier*: los tokens facturados gratis aparecen como una partida propia, "data sharing incentive tier". Una entrada mensual en el calendario para revisar el panel completa la cadena de guardarraíles, pues OpenAI puede finalizar el programa con un preaviso de 30 días y, a partir de ese día, el mismo flujo de trabajo seguiría funcionando a la tarifa normal.

## Fuentes

1.  [OpenAI Help Center: Sharing feedback, evaluation and fine-tuning data, and API inputs and outputs](https://help.openai.com/en/articles/10306912-sharing-feedback-evaluation-and-fine-tuning-data-and-api-inputs-and-outputs-with-openai): descripción determinante del programa con grupos de modelos, cuotas por nivel, reinicio UTC y la regla de facturación para solicitudes que superan la cuota.

2.  [OpenAI Developer Community: Extended: Free tokens on traffic shared with OpenAI](https://community.openai.com/t/good-news-extended-free-tokens-on-traffic-shared-with-openai/1241322): anuncio de la ampliación del programa en abril de 2025 con la garantía del plazo de 30 días.

3.  [OpenAI Platform: Data sharing settings](https://platform.openai.com/settings/organization/data-controls/sharing): interruptor de adhesión y estado de inscripción de la propia organización (requiere inicio de sesión).

4.  [OpenAI Platform: Rate limits guide](https://platform.openai.com/docs/guides/rate-limits): explicación de los límites TPM, RPM y TPD que se aplican además de la cuota gratuita.

5.  [OpenAI Platform: Pricing](https://platform.openai.com/docs/pricing): tarifas normales a las que se facturan los excesos de cuota.
