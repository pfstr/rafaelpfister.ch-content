---
title: "Determinar el perfil de carga de un servidor de correo: ráfagas, tasas pico y estructura de destinatarios a partir del seguimiento de mensajes"
navTitle: "Determinar el perfil de carga"
description: "¿Cuántos correos por minuto procesa realmente su servidor de correo y cuáles son los picos? Cómo determinar el perfil de carga real con PowerShell a partir del seguimiento de mensajes de Exchange: tasas por minuto y hora, duración de las ráfagas, estructura de destinatarios y tamaños de mensaje. Con los errores habituales de análisis."
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
url: https://rafaelpfister.ch/es/blog/determinar-el-perfil-de-carga-de-un-servidor-de-correo-rafagas-tasas-pico-y-estructura-de
translationSourceHash: 16095cf53ce6f67abe31387ce2f02958eacc3898d3a42b61ad8c7b885ab7ce5d
translationModel: gpt-5.6-terra
translatedAt: 2026-08-26T04:10:23.307Z
translationReview: required
---

# Determinar el perfil de carga de un servidor de correo: ráfagas, tasas pico y estructura de destinatarios a partir del seguimiento de mensajes

Ya sea para sustituir un gateway, dimensionar un servidor o planificar una ventana de mantenimiento: tarde o temprano, todo administrador de correo necesita responder a la pregunta de cuánto procesa realmente su sistema. La intuición suele equivocarse, porque el tráfico de correo rara vez es uniforme. Un sistema que registra una media diaria de 20 correos por minuto puede tener que procesar 400 por minuto durante una hora al ejecutar una facturación. Quien solo conoce la media dimensiona sin atender al problema real.

Un perfil de carga útil consta de cuatro indicadores: la tasa media (por minuto, hora y día), las ráfagas (cuál es el pico, cuánto dura y cuándo se produce), la estructura de destinatarios (cuántos destinatarios distintos hay y qué dominios de destino) y los tamaños de los mensajes. Los cuatro figuran en el seguimiento de mensajes y, en Exchange, pueden calcularse con unas pocas líneas de PowerShell.

## La fuente de datos: seguimiento de mensajes

Exchange registra cada mensaje en el registro de seguimiento de mensajes. Antes de analizar los datos, compruebe hasta qué fecha se remontan; el valor predeterminado es de 30 días, pero un límite de tamaño ajustado puede reducir considerablemente la retención real:

```powershell
Get-TransportService |
    Select-Object Name, MessageTrackingLogMaxAge,
        MessageTrackingLogMaxDirectorySize, MessageTrackingLogPath
```

Para un perfil de carga, el período debe abarcar al menos un ciclo completo de procesos por lotes de la empresa: ejecuciones de facturación mensual, nóminas y boletines. Una semana es el mínimo; un mes es mejor.

## Recopilar los datos brutos: un evento por mensaje

La decisión previa más importante es esta: ¿qué evento cuenta como «un correo»? El seguimiento de mensajes escribe varias entradas por mensaje (RECEIVE al aceptarlo, SEND al reenviarlo al siguiente salto, DELIVER al entregarlo en el buzón, además de AGENTINFO, HAREDIRECT y otros). Quien simplemente cuenta todas las líneas sobreestima el volumen varias veces. Para la carga de entrada, cuente RECEIVE; para la carga de salida hacia el smarthost o Internet, SEND.

```powershell
$start = (Get-Date).AddDays(-7)
$events = Get-TransportService | ForEach-Object {
    Get-MessageTrackingLog -Server $_.Name -ResultSize Unlimited `
        -Start $start -EventId RECEIVE
}
"{0} Nachrichten seit {1:yyyy-MM-dd}" -f $events.Count, $start
```

La consulta se ejecuta deliberadamente en todos los servidores de transporte, ya que cada servidor registra únicamente su propia parte. Quien consulta un solo servidor solo verá una fracción de la carga en un clúster.

## Tasas por minuto y hora: aquí se revelan las ráfagas

La agregación consiste en un Group-Object sobre la marca temporal redondeada. Los minutos con mayor volumen son directamente sus candidatos a ráfagas:

```powershell
$proMinute = $events |
    Group-Object { $_.Timestamp.ToString("yyyy-MM-dd HH:mm") } |
    Sort-Object Count -Descending

$proMinute | Select-Object -First 10 Name, Count
```

Lo mismo por hora y como patrón diario (qué hora suele estar sometida a qué nivel de carga):

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

Una ráfaga solo queda caracterizada cuando, además del pico, conoce su duración. Un pico de 400/min que dura dos minutos supone un requisito distinto del mismo pico durante una hora. Cuente los minutos por encima de un umbral:

```powershell
$schwelle = 100
$burstMinuten = $proMinute | Where-Object Count -ge $schwelle
"{0} Minuten mit >= {1}/min, Peak: {2}/min" -f $burstMinuten.Count,
    $schwelle, ($proMinute | Select-Object -First 1).Count
```

Si los minutos de ráfaga son consecutivos (visibles directamente en la salida de `$burstMinuten | Sort-Object Name`), se trata de una ejecución por lotes. Anote la hora de inicio, la duración y el patrón de repetición, pues es precisamente esa ventana la que debe soportar la infraestructura.

## Estructura de destinatarios: cuántos destinos, qué dominios

Para los gateways, la diversidad de destinatarios suele ser más importante que la tasa pura, porque cada destinatario implica consultas (enrutamiento, políticas y reglas de cifrado). Un correo a una lista de distribución con 5'000 miembros carga el sistema de forma distinta a 5'000 correos individuales. El campo `RecipientCount` y la lista de destinatarios proporcionan ambas perspectivas:

```powershell
"Nachrichten: {0}, Empfänger-Zustellungen: {1}" -f $events.Count,
    ($events | Measure-Object RecipientCount -Sum).Sum

$alleEmpfaenger = $events | ForEach-Object { $_.Recipients } |
    ForEach-Object { $_.ToLower() }
"Eindeutige Empfänger: {0}" -f ($alleEmpfaenger | Sort-Object -Unique).Count
```

La distribución por dominios muestra hacia dónde fluye el tráfico. Si predominan Gmail y Microsoft, sus límites de tasa y la reputación de su propia IP determinan el rendimiento alcanzable, no su propio hardware:

```powershell
$alleEmpfaenger |
    ForEach-Object { ($_ -split "@")[1] } |
    Group-Object |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count
```

Y en sentido contrario: ¿qué remitentes (aplicaciones, buzones funcionales) generan la carga? Esto también responde a la pregunta de qué sistemas deben tenerse en cuenta en una migración:

```powershell
$events |
    Group-Object Sender |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count
```

## Tamaños de mensaje: bytes por segundo en lugar de correos por segundo

Las especificaciones de rendimiento de los gateways suelen referirse al volumen de datos, no al número de mensajes. Dos sistemas con la misma tasa de correo difieren por un factor de 100 si uno envía notificaciones de 50 KB y el otro facturas en PDF de 5 MB. El campo `TotalBytes` proporciona la distribución:

```powershell
$events | Measure-Object TotalBytes -Average -Maximum -Sum |
    Select-Object @{n = "MittelKB"; e = { [math]::Round($_.Average / 1KB) } },
        @{n = "MaxMB"; e = { [math]::Round($_.Maximum / 1MB, 1) } },
        @{n = "TotalGB"; e = { [math]::Round($_.Sum / 1GB, 1) } }
```

Multiplique la tasa de ráfaga por el tamaño medio dentro de la ventana de ráfaga y tendrá el requisito de ancho de banda que debe soportar un nuevo gateway o un enlace WAN.

## Tasas en directo sin seguimiento: vistazo a las colas

Para una visión instantánea (¿está procesando mucho el servidor ahora mismo?, ¿se está acumulando algo?) no necesita seguimiento: las colas lo muestran directamente. `IncomingRate` y `OutgoingRate` son correos por minuto, suavizados durante los últimos minutos:

```powershell
Get-TransportService |
    ForEach-Object { Get-Queue -Server $_.Name } |
    Sort-Object MessageCount -Descending |
    Select-Object Identity, Status, MessageCount, IncomingRate,
        OutgoingRate, NextHopDomain, LastError |
    Format-Table -AutoSize
```

La interpretación: una cola `Submission` con una tasa alta y profundidad 0 significa que el servidor procesa la carga sin acumulación. `MessageCount` alto con `OutgoingRate` cercano a cero significa retención. `Status Retry` con un mensaje 4xx en `LastError` significa que el destino está limitando la tasa. En cambio, las colas `Shadow` con elementos son normales: son copias de redundancia para el servidor asociado, no acumulación.

Para una curva continua durante una ventana de carga, resulta adecuado el contador de rendimiento de las colas de transporte, aquí cada cinco segundos durante un minuto:

```powershell
Get-Counter "\MSExchangeTransport Queues(_total)\Messages Completed Delivery Per Second" `
    -SampleInterval 5 -MaxSamples 12
```

## Otros sistemas: el mismo principio con CSV

Los gateways y appliances suelen proporcionar una exportación CSV del seguimiento en lugar de objetos PowerShell. El procedimiento sigue siendo idéntico (elegir un evento por correo y agrupar por intervalos de tiempo); solo cambia la herramienta, por ejemplo, a Python:

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

## Los cinco errores clásicos de análisis

**Eventos múltiples por correo.** La fuente de error más frecuente: contar líneas en lugar de mensajes. Compruebe con `$events | Group-Object EventId` qué contiene realmente su conjunto de datos y filtre exactamente un evento por mensaje.

**Exportaciones truncadas.** Muchas funciones de exportación devuelven como máximo 10'000 o 50'000 líneas y luego las recortan sin avisar, a menudo justo en medio de la mayor ráfaga. Un número de líneas sospechosamente redondo es una señal de alarma. Compruebe siempre si el período de los datos coincide con el período solicitado.

**Bucles de gateway.** Si el flujo de correo pasa por una estación intermedia (gateway de cifrado, appliance de higiene) y vuelve, el mismo correo aparece varias veces en el seguimiento. Elimine duplicados mediante el ID de mensaje o filtre por un punto inequívoco de la cadena.

**Zonas horarias.** `Get-MessageTrackingLog` proporciona marcas temporales en la hora local del servidor, mientras que las exportaciones CSV de los appliances suelen estar en UTC. Una ráfaga que aparentemente se produce a las 13:00 puede ser en realidad el proceso por lotes de las 15:00. Aclare la base temporal antes de interpretar los datos.

**Ventanas demasiado cortas.** Un perfil de carga basado en dos días tranquilos no sirve de nada si falta la ejecución de facturación mensual. La ventana de análisis debe incluir los ciclos de procesos por lotes conocidos; consulte a los responsables de las aplicaciones sobre sus planes de envío antes de definirla.

## Qué hacer con el perfil

Al final tendrá cuatro cifras en una página: tasa media, ráfaga (pico, duración, momento y patrón de repetición), estructura de destinatarios (destinatarios únicos por ejecución, dominios principales) y distribución de tamaños. Con ellas se pueden dimensionar gateways, programar ventanas de mantenimiento durante las horas nocturnas de carga real cero y formular criterios de aceptación, por ejemplo: el nuevo sistema debe procesar sin errores el doble del pico medido. El artículo [Prueba de carga SMTP con Apache JMeter en la práctica](/blog/jmeter-smtp-lasttest-html-report) muestra cómo convertir ese perfil en una prueba de carga reproducible.

## Fuentes

1.  [Microsoft Learn: Get-MessageTrackingLog](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog): referencia de la consulta de seguimiento, incluidos todos los campos como EventId, RecipientCount y TotalBytes.

2.  [Microsoft Learn: Message tracking](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): estructura de los registros de seguimiento, tipos de eventos y configuración de retención y tamaño de directorio.

3.  [Microsoft Learn: Get-Queue](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue): referencia de la consulta de colas, incluidos los campos IncomingRate, OutgoingRate y Velocity.

4.  [Microsoft Learn: Queues and messages in queues](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues): tipos de cola, redundancia de sombra y significado de los valores de estado.
