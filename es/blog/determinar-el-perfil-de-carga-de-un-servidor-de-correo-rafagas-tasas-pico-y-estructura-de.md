---
title: "Determinar el perfil de carga de un servidor de correo: ráfagas, tasas máximas y estructura de destinatarios a partir del Message Tracking"
navTitle: "Determinar el perfil de carga"
description: "¿Cuántos correos por minuto procesa realmente su servidor de correo y cuáles son los picos? Cómo determinar el perfil de carga real con PowerShell a partir del Exchange Message Tracking: tasas por minuto y hora, duración de las ráfagas, estructura de destinatarios, tamaños de mensajes y errores de análisis habituales."
date: "2026-08-25"
kategorie: "SMTP y flujo de correo"
timeToRead: "9 min de lectura"
themen:
  - smtp-mailflow
  - exchange-onprem-hybrid
produkte:
  - "exchange-on-premises"
  - "uebergreifend"
protokolle:
  - "smtp"
  - "powershell"
  - "troubleshooting"
slug: "determinar-el-perfil-de-carga-de-un-servidor-de-correo-rafagas-tasas-pico-y-estructura-de"
translationId: "article-1ff17a188d73e289"
aiPrompt: |
  Du bist mein Mailflow-Assistent. Hilf mir Schritt für Schritt, das Lastprofil meines Mailservers zu ermitteln: 1. Die richtige Datenquelle wählen (Message Tracking, Gateway-Logs) und das passende Event pro Nachricht bestimmen. 2. Raten pro Minute, Stunde und Tag berechnen und Bursts mit Dauer und Peak charakterisieren. 3. Empfängerstruktur, Domain-Verteilung und Nachrichtengrössen auswerten. Weise mich auf Doppelzählungen, Export-Limits und Zeitzonen-Fallen hin.
translationOf: mailserver-lastprofil-ermitteln
translationSourceHash: 298fabdf5f8f248539ea8a119681be130cd76f5c8ebc35db5d0c61e1126251b0
translationModel: gpt-5.6-terra
translatedAt: 2026-08-30T09:20:40.848Z
translationReview: required
url: https://rafaelpfister.ch/es/blog/determinar-el-perfil-de-carga-de-un-servidor-de-correo-rafagas-tasas-pico-y-estructura-de
---

# Determinar el perfil de carga de un servidor de correo: ráfagas, tasas máximas y estructura de destinatarios a partir del Message Tracking

Ya sea para sustituir un gateway, dimensionar un servidor o planificar una ventana de mantenimiento: tarde o temprano, todo administrador de correo necesita responder a la pregunta de cuánto procesa realmente su sistema. La intuición suele fallar, ya que el tráfico de correo rara vez es uniforme. Un sistema que registra una media diaria de 20 correos por minuto puede tener que procesar 400 por minuto durante una hora al ejecutar la facturación. Quien solo conoce la media dimensiona para un problema distinto del real.

Un perfil de carga útil consta de cuatro métricas: la tasa media (por minuto, hora y día), las ráfagas (cuál es el pico, cuánto dura y cuándo se produce), la estructura de destinatarios (cuántos destinatarios distintos hay y cuáles son los dominios de destino) y los tamaños de los mensajes. Las cuatro figuran en Message Tracking y, en Exchange, pueden calcularse con unas pocas líneas de PowerShell.

## La fuente de datos: Message Tracking

Exchange registra cada mensaje en el Message Tracking Log. Antes de analizarlo, compruebe hasta qué fecha se remontan los datos; el valor predeterminado es de 30 días, pero un límite de tamaño reducido puede acortar considerablemente la retención real:

```powershell
Get-TransportService |
    Select-Object Name, MessageTrackingLogMaxAge,
        MessageTrackingLogMaxDirectorySize, MessageTrackingLogPath
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Get-TransportService` | Enumera todos los servidores de transporte de la organización; sin parámetros, todos los servidores |
| `Select-Object Name, MessageTrackingLog…` | Reduce la salida a las propiedades indicadas: período de retención, límite de tamaño del directorio de registros y ruta de los registros |

</details>

Para un perfil de carga, el período debe abarcar al menos un ciclo completo de lotes de la empresa: ejecuciones mensuales de facturación, nóminas y boletines. Una semana es el mínimo; un mes es mejor.

## Recopilar datos brutos: un evento por mensaje

La decisión previa más importante es: ¿qué evento cuenta como «un correo»? Message Tracking escribe varias entradas por mensaje (RECEIVE al aceptarlo, SEND al transferirlo al siguiente salto, DELIVER al entregarlo en el buzón, además de AGENTINFO, HAREDIRECT y otros). Si simplemente cuenta todas las líneas, sobreestima el volumen varias veces. Para la carga de entrada, cuente RECEIVE; para la carga de salida hacia el smarthost o Internet, SEND.

```powershell
$start = (Get-Date).AddDays(-7)
$events = Get-TransportService | ForEach-Object {
    Get-MessageTrackingLog -Server $_.Name -ResultSize Unlimited `
        -Start $start -EventId RECEIVE
}
"{0} Nachrichten seit {1:yyyy-MM-dd}" -f $events.Count, $start
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Server $_.Name` | Consulta el registro de seguimiento del respectivo servidor de transporte desde la canalización |
| `-ResultSize Unlimited` | Elimina el límite predeterminado de 1'000 entradas devueltas |
| `-Start $start` | Límite temporal inferior de la consulta; aquí, los últimos siete días |
| `-EventId RECEIVE` | Filtra exactamente un evento por mensaje, aquí la aceptación por parte del servicio de transporte |
| `-f` | Operador de formato: inserta los valores de la derecha en los marcadores `{0}` y `{1}` de la cadena |

</details>

La consulta se ejecuta deliberadamente en todos los servidores de transporte, ya que cada servidor registra únicamente su propia parte. Si solo consulta un servidor, en un clúster verá apenas una fracción de la carga.

## Tasas por minuto y hora: aquí se ven las ráfagas

La agregación es un Group-Object sobre la marca de tiempo redondeada. Los minutos con mayor actividad son directamente sus candidatos a ráfaga:

```powershell
$proMinute = $events |
    Group-Object { $_.Timestamp.ToString("yyyy-MM-dd HH:mm") } |
    Sort-Object Count -Descending

$proMinute | Select-Object -First 10 Name, Count
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Group-Object { … }` | Agrupa según el valor devuelto por el bloque de script, aquí la marca de tiempo truncada al minuto |
| `Sort-Object Count -Descending` | Ordena los grupos de forma descendente por cantidad; los minutos más intensos aparecen arriba |
| `Select-Object -First 10 Name, Count` | Devuelve solo los diez grupos mayores, reducidos a minuto y cantidad |

</details>

Lo mismo por hora y como patrón diario (qué hora del día suele tener qué nivel de carga):

```powershell
$events |
    Group-Object { $_.Timestamp.ToString("yyyy-MM-dd HH") } |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count

$events |
    Group-Object { $_.Timestamp.ToString("HH") } |
    Sort-Object Name |
    Format-Table Name, Count
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Group-Object { … ToString("yyyy-MM-dd HH") }` | Agrupa por horas completas de un día concreto |
| `Group-Object { … ToString("HH") }` | Agrupa solo por la hora y, por tanto, agrega todos los días: el patrón diario |
| `Sort-Object Count -Descending` | Las horas más intensas aparecen arriba |
| `Sort-Object Name` | Ordena cronológicamente el patrón diario por hora en lugar de por cantidad |
| `Format-Table Name, Count` | Salida tabular de las dos columnas |

</details>

Una ráfaga solo queda caracterizada cuando, además del pico, conoce su duración. Un pico de 400/min que dura dos minutos plantea un requisito distinto que el mismo pico durante una hora. Cuente los minutos por encima de un umbral:

```powershell
$schwelle = 100
$burstMinuten = $proMinute | Where-Object Count -ge $schwelle
"{0} Minuten mit >= {1}/min, Peak: {2}/min" -f $burstMinuten.Count,
    $schwelle, ($proMinute | Select-Object -First 1).Count
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Where-Object Count -ge $schwelle` | Filtra los minutos con al menos tantas mensajes como el umbral (sintaxis simplificada sin bloque de script) |
| `Select-Object -First 1` | Primer grupo de la lista ordenada de forma descendente, es decir, el minuto más intenso |
| `-f` | Operador de formato: inserta la cantidad, el umbral y el pico en los marcadores `{0}` hasta `{2}` |

</details>

Si los minutos de ráfaga son consecutivos (visibles directamente en la salida de `$burstMinuten | Sort-Object Name`), se trata de una ejecución por lotes. Anote la hora de inicio, la duración y el patrón de repetición, pues precisamente esa ventana debe poder soportarla la infraestructura.

## Estructura de destinatarios: cuántos destinos, qué dominios

Para los gateways, la diversidad de destinatarios suele ser más importante que la mera tasa, ya que se realizan búsquedas por cada destinatario (enrutamiento, políticas, reglas de cifrado). Un correo a una lista de distribución con 5'000 miembros genera una carga distinta de 5'000 correos individuales. El campo `RecipientCount` y la lista de destinatarios proporcionan ambas perspectivas:

```powershell
"Nachrichten: {0}, Empfänger-Zustellungen: {1}" -f $events.Count,
    ($events | Measure-Object RecipientCount -Sum).Sum

$alleEmpfaenger = $events | ForEach-Object { $_.Recipients } |
    ForEach-Object { $_.ToLower() }
"Eindeutige Empfänger: {0}" -f ($alleEmpfaenger | Sort-Object -Unique).Count
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Measure-Object RecipientCount -Sum` | Suma el campo `RecipientCount` en todos los eventos: el número de entregas a destinatarios |
| `ForEach-Object { $_.Recipients }` | Despliega la lista de destinatarios de cada evento en direcciones individuales |
| `ForEach-Object { $_.ToLower() }` | Normaliza las direcciones a minúsculas para que se reconozcan los duplicados como tales |
| `Sort-Object -Unique` | Ordena y elimina duplicados; `Count` devuelve después las direcciones únicas |

</details>

La distribución por dominios muestra hacia dónde fluye el tráfico. Si predominan Gmail y Microsoft, sus límites de tasa y la reputación de su propia IP determinan el rendimiento alcanzable, no su propio hardware:

```powershell
$alleEmpfaenger |
    ForEach-Object { ($_ -split "@")[1] } |
    Group-Object |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `($_ -split "@")[1]` | Divide la dirección en `@` y conserva la parte del dominio |
| `Group-Object` | Agrupa sin argumento por el propio valor, aquí el dominio |
| `Sort-Object Count -Descending` | Los dominios más frecuentes aparecen arriba |
| `Select-Object -First 10 Name, Count` | Limita la salida al top 10 |

</details>

Y en sentido contrario: ¿qué remitentes (aplicaciones, buzones funcionales) generan realmente la carga? Esto también responde a la pregunta de qué sistemas deben tenerse en cuenta en una migración:

```powershell
$events |
    Group-Object Sender |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Group-Object Sender` | Agrupa por el campo `Sender` (parámetro posicional `-Property`) |
| `Sort-Object Count -Descending` | Los remitentes con más mensajes aparecen arriba |
| `Select-Object -First 10 Name, Count` | Limita la salida al top 10 |

</details>

## Tamaños de mensajes: bytes por segundo en lugar de correos por segundo

Las especificaciones de rendimiento de los gateways suelen referirse al volumen de datos, no al número de mensajes. Dos sistemas con la misma tasa de correos difieren por un factor de 100 si uno envía notificaciones de 50 KB y el otro PDF de facturas de 5 MB. El campo `TotalBytes` proporciona la distribución:

```powershell
$events | Measure-Object TotalBytes -Average -Maximum -Sum |
    Select-Object @{n = "MittelKB"; e = { [math]::Round($_.Average / 1KB) } },
        @{n = "MaxMB"; e = { [math]::Round($_.Maximum / 1MB, 1) } },
        @{n = "TotalGB"; e = { [math]::Round($_.Sum / 1GB, 1) } }
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Measure-Object TotalBytes -Average -Maximum -Sum` | Calcula la media, el máximo y la suma del campo `TotalBytes` en una sola pasada |
| `@{n = "…"; e = { … }}` | Propiedad calculada: `n` nombra la columna, `e` proporciona el valor mediante un bloque de script, aquí la conversión a KB, MB y GB |

</details>

Multiplique la tasa de ráfaga por el tamaño medio en la ventana de ráfaga y tendrá el requisito de ancho de banda que debe soportar un nuevo gateway o enlace WAN.

## Tasas en tiempo real sin seguimiento: vistazo a las colas

Para una visión puntual (¿el servidor está procesando mucho ahora, se está acumulando algo?), no necesita seguimiento: las colas lo muestran directamente. `IncomingRate` y `OutgoingRate` son correos por minuto, suavizados durante los últimos minutos:

```powershell
Get-TransportService |
    ForEach-Object { Get-Queue -Server $_.Name } |
    Sort-Object MessageCount -Descending |
    Select-Object Identity, Status, MessageCount, IncomingRate,
        OutgoingRate, NextHopDomain, LastError |
    Format-Table -AutoSize
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Get-Queue -Server $_.Name` | Enumera las colas de transporte del respectivo servidor desde la canalización |
| `Sort-Object MessageCount -Descending` | Las colas más llenas aparecen arriba |
| `Select-Object Identity, Status, …` | Limita la salida a los campos relevantes para evaluar la carga |
| `Format-Table -AutoSize` | Ajusta el ancho de las columnas al contenido en lugar de truncarlas |

</details>

Interpretación: una cola `Submission` con tasa alta y profundidad 0 significa que el servidor procesa la carga sin acumulación. `MessageCount` alto con `OutgoingRate` cercano a cero significa acumulación. `Status Retry` con un mensaje 4xx en `LastError` significa que el extremo remoto está limitando la tasa. En cambio, las colas `Shadow` con elementos son normales: son copias redundantes para el servidor asociado, no una acumulación.

Para una curva continua durante una ventana de carga, resulta adecuado el contador de rendimiento de las colas de transporte, aquí durante un minuto cada cinco segundos:

```powershell
Get-Counter "\MSExchangeTransport Queues(_total)\Messages Completed Delivery Per Second" `
    -SampleInterval 5 -MaxSamples 12
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `"\MSExchangeTransport Queues(_total)\…"` | Ruta del contador de rendimiento (parámetro posicional `-Counter`); la instancia `_total` suma todas las colas |
| `-SampleInterval 5` | Intervalo entre dos mediciones en segundos |
| `-MaxSamples 12` | Número de mediciones; 12 mediciones cada 5 segundos dan un minuto |

</details>

## Otros sistemas: el mismo principio con CSV

Los gateways y appliances suelen proporcionar una exportación CSV del seguimiento en lugar de objetos de PowerShell. El procedimiento sigue siendo idéntico (elegir un evento por correo, agrupar por ventanas temporales); solo cambia la herramienta, por ejemplo a Python:

```python
import csv, collections, datetime

per_min = collections.Counter()
with open("tracking-export.csv", encoding="utf-8") as f:
    reader = csv.reader(f)
    next(reader)
    for row in reader:
        if "response '2" not in row[6]:   # nur finale Zustellungen
            continue
        d = datetime.datetime.strptime(row[0][:16], "%Y-%m-%d %H:%M")
        per_min[d.strftime("%Y-%m-%d %H:%M")] += 1

print(per_min.most_common(10))
```

## Los cinco errores habituales de análisis

**Varios eventos por correo.** La fuente de error más frecuente: contar líneas en lugar de mensajes. Compruebe con `$events | Group-Object EventId` qué contiene realmente su conjunto de datos y filtre exactamente un evento por mensaje.

**Exportaciones truncadas.** Muchas funciones de exportación devuelven como máximo 10'000 o 50'000 líneas y después las cortan sin avisar, a menudo en mitad de la mayor ráfaga. Una cantidad de líneas sospechosamente redonda es una señal de alarma. Compruebe siempre si el período de los datos coincide con el período solicitado.

**Bucles de gateway.** Si el flujo de correo pasa por una estación intermedia (gateway de cifrado, appliance de higiene) y vuelve, el mismo correo aparece varias veces en el seguimiento. Elimine duplicados mediante el ID del mensaje o filtre por un punto inequívoco de la cadena.

**Zonas horarias.** `Get-MessageTrackingLog` proporciona marcas de tiempo en la hora local del servidor, mientras que las exportaciones CSV de appliances suelen estar en UTC. Una ráfaga que aparentemente se produce a las 13:00 puede ser en realidad el lote de las 15:00. Aclare la base temporal antes de interpretar los datos.

**Ventanas demasiado cortas.** Un perfil de carga de dos días tranquilos no sirve de nada si falta la ejecución mensual de facturación. La ventana de análisis debe incluir los ciclos de lotes conocidos; consulte a los responsables de las aplicaciones sobre sus planes de envío antes de definirla.

## Qué hacer con el perfil

Al final tendrá cuatro cifras en una página: tasa media, ráfaga (pico, duración, momento y patrón de repetición), estructura de destinatarios (destinatarios únicos por ejecución, dominios principales) y distribución de tamaños. Con ello se pueden dimensionar gateways, programar ventanas de mantenimiento durante las horas nocturnas de carga cero real y formular criterios de aceptación, por ejemplo: el nuevo sistema debe procesar sin errores el doble del pico medido. El artículo [Prueba de carga SMTP con Apache JMeter en la práctica](/blog/jmeter-smtp-lasttest-html-report) muestra cómo convertir un perfil de este tipo en una prueba de carga reproducible.

## Fuentes

1.  [Microsoft Learn: Get-MessageTrackingLog](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog): Referencia de la consulta de seguimiento, incluidos todos los campos como EventId, RecipientCount y TotalBytes.

2.  [Microsoft Learn: Message tracking](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): Estructura de los registros de seguimiento, tipos de eventos y configuración de retención y tamaño de directorio.

3.  [Microsoft Learn: Get-Queue](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue): Referencia de la consulta de colas, incluidos los campos IncomingRate, OutgoingRate y Velocity.

4.  [Microsoft Learn: Queues and messages in queues](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues): Tipos de cola, Shadow Redundancy y el significado de los valores de estado.
