---
title: "¿Cuánto tiempo permanece abierta una sesión SMTP? ConnectionTimeout 00:10:00 en Exchange y los sistemas para los que es demasiado corto"
navTitle: "Duración de la sesión SMTP"
description: "Exchange finaliza cada sesión SMTP entrante después de diez minutos, incluso si está transmitiendo datos en ese momento. Qué remitentes permanecen tanto tiempo en una conexión, cómo consultar la duración real de la sesión en el registro de protocolo y cuándo conviene ajustar ConnectionTimeout y ConnectionInactivityTimeout en un conector de retransmisión."
date: "2026-09-03"
kategorie: "SMTP y flujo de correo"
timeToRead: "10 min de lectura"
themen:
  - smtp-mailflow
  - exchange-onprem-hybrid
produkte:
  - "exchange-on-premises"
  - "exchange"
protokolle:
  - "smtp"
  - "troubleshooting"
slug: "cuanto-tiempo-permanece-abierta-una-sesion-smtp-connectiontimeout-00-10-00-en-exchange-y-los"
translationId: "article-b40497933bbe0a88"
aiPrompt: |
  Du bist mein Exchange- und Mailflow-Assistent. Hilf mir, die SMTP-Session-Dauer auf einem Exchange-Receive-Connector zu beurteilen: 1. Frage mich, welche Systeme (Relays, Gateways, Applikationen, Scanner) über den Connector einliefern und ob sie Verbindungen über mehrere Nachrichten hinweg offen halten. 2. Lass dir die Ausgabe der Session-Auswertung aus dem Protokoll-Log geben (IP, Mails, Dauer, Timeout-Kennzeichen) und erkläre mir, welche Sessions am ConnectionTimeout abgebrochen wurden. 3. Empfiehl pro Connector konkrete Werte für ConnectionTimeout und ConnectionInactivityTimeout und begründe, warum der internetseitige Connector unverändert bleibt. 4. Nenne mir, was ich stattdessen auf der Client-Seite ändern kann, damit die Verbindung nach einer festen Anzahl Nachrichten neu aufgebaut wird.
translationOf: smtp-session-dauer-exchange-connectiontimeout
url: https://rafaelpfister.ch/es/blog/cuanto-tiempo-permanece-abierta-una-sesion-smtp-connectiontimeout-00-10-00-en-exchange-y-los
translationSourceHash: a107c4edd960dabb30ba1b6f263a693808a5edf6815747d81f5d446c103a7e79
translationModel: gpt-5.6-terra
translatedAt: 2026-09-04T08:21:52.711Z
translationReview: automatic
---

# ¿Cuánto tiempo permanece abierta una sesión SMTP? ConnectionTimeout 00:10:00 en Exchange y los sistemas para los que es demasiado corto

Conclusión breve: una sesión SMTP no tiene un final natural. RFC 5321 solo limita el tiempo de espera para cada paso siguiente, y un cliente puede seguir entregando mensajes a través de una conexión abierta mientras el servidor la mantenga abierta. Exchange las mantiene abiertas durante diez minutos de forma predeterminada en los conectores de recepción; después, el servidor cierra la conexión, independientemente de si se están transmitiendo datos. Para el tráfico de Exchange a Exchange y para la mayoría de los MTA esto es irrelevante, porque esos remitentes vuelven a conectarse por iniciativa propia al cabo de unos segundos. En cambio, para aplicaciones, puertas de enlace y generadores de carga que utilizan una sola conexión para toda una ejecución de envío, este valor provoca interrupciones que el cliente muestra como errores de conexión y el registro de protocolo de Exchange como `421 4.4.1 Connection timed out`.

## Dos tiempos de espera con distinto significado

Un conector de recepción tiene dos límites de tiempo que a menudo se confunden:

| Parámetro | Significado | Valor predeterminado del servidor Mailbox | Valor predeterminado de Edge Transport |
|---|---|---|---|
| `ConnectionInactivityTimeout` | tiempo máximo de inactividad sin actividad del cliente; después se cierra la conexión | 00:05:00 | 00:01:00 |
| `ConnectionTimeout` | duración total máxima de la conexión, incluso si transmite datos activamente | 00:10:00 | 00:05:00 |

Ambos valores aceptan desde 1 segundo hasta 1 día (`1.00:00:00`), y `ConnectionTimeout` debe ser mayor que `ConnectionInactivityTimeout`. Los valores se aplican por conector, es decir, por separado para el `Default Frontend <Server>` orientado a Internet, para el conector del servicio de transporte `Default <Server>` en el puerto 2525 y para cada conector de retransmisión creado manualmente.

El tiempo de espera por inactividad no es crítico: cinco minutos corresponden exactamente al mínimo que RFC 5321 prescribe a un servidor como tiempo de espera para el siguiente comando, y un cliente que no envía nada durante cinco minutos normalmente ya ha olvidado la conexión. El tiempo de espera total es la particularidad de Exchange: cuenta desde que se establece la conexión y sigue transcurriendo mientras el cliente entrega mensaje tras mensaje. Tras diez minutos, Exchange cierra la conexión en el punto en el que se encuentre el diálogo, incluso en mitad de un bloque `DATA` si es necesario.

En el lado de envío no existe equivalente: un conector de envío solo tiene `ConnectionInactivityTimeOut` (diez minutos de forma predeterminada) y limita las sesiones mediante `SmtpMaxMessagesPerConnection`, 20 mensajes de forma predeterminada. Por tanto, Exchange como cliente cierra cada conexión por sí mismo después de como máximo 20 mensajes y establece una nueva. Este es el motivo por el que el tiempo de espera total nunca se nota entre servidores Exchange: las sesiones duran segundos.

## Lo que prescribe RFC 5321

El estándar define en la sección 4.5.3.2 los tiempos de espera mínimos que un cliente debe respetar para cada paso del protocolo antes de abandonar la conexión:

| Paso | Tiempo de espera mínimo en el cliente |
|---|---|
| Esperar el banner `220` | 5 minutos |
| Respuesta a `MAIL` | 5 minutos |
| Respuesta a `RCPT` | 5 minutos |
| Respuesta a `DATA` (el `354`) | 2 minutos |
| Envío de un bloque de datos | 3 minutos |
| Respuesta al punto final | 10 minutos |
| Servidor: espera del siguiente comando | al menos 5 minutos |

El RFC no establece un límite máximo para la duración total de una sesión. Un cliente que entrega mensajes durante treinta minutos por la misma conexión y nunca permanece en silencio más de unos segundos se comporta conforme al estándar. Destaca el último valor del cliente: diez minutos de espera para la respuesta posterior al punto final, porque en esta fase el servidor acepta y asume el mensaje. Si el cliente abandona demasiado pronto, el mensaje ya se ha entregado y se enviará por segunda vez en el siguiente intento. La misma situación se produce de forma inversa cuando el servidor cierra la conexión en ese momento debido al tiempo de espera total.

Si un servidor cierra la conexión con `421`, el cliente debe tratar la transacción en curso según la sección 3.8 como si hubiera recibido un `451`, es decir, como un error temporal con reintento. Un MTA con cola hace exactamente eso. Una aplicación sin cola, en cambio, informa de una excepción y deja el resto al llamador.

## Cuánto tiempo mantienen realmente abiertas sus sesiones los remitentes

La duración de la sesión la determina el cliente, y las diferencias entre tipos de remitentes son grandes:

| Remitente | Duración típica de la sesión | Limitada por |
|---|---|---|
| Conector de envío de Exchange | Segundos | `SmtpMaxMessagesPerConnection` = 20 |
| Postfix con caché de conexiones | máximo 5 minutos | `smtp_connection_reuse_time_limit` = 300s |
| Postfix sin caché de conexiones | un mensaje por conexión | comportamiento predeterminado del cliente `smtp` |
| Aplicación con `.NET SmtpClient`, `JavaMail Transport`, Python `smtplib` | mientras viva el objeto: durante toda la ejecución en un proceso por lotes | solo por el código del programa |
| Notificaciones de cuarentena de puertas de enlace de correo | una sesión por ejecución de notificaciones | comportamiento del producto, en parte con keepalive `NOOP` |
| Dispositivos multifunción, escaneo a correo | un mensaje por conexión; varios minutos para escaneos grandes a través de líneas lentas | tamaño de archivo y ancho de banda |
| Generadores de carga como `smtp-source -d` | hasta el final de la ejecución | parámetros de invocación |

Las dos primeras filas explican por qué nadie nota este valor durante años en entornos clásicos: los MTA mantienen las conexiones cortas por iniciativa propia. Postfix, por ejemplo, utiliza una conexión almacenada en caché durante un máximo de cinco minutos y después abre una nueva, mientras que Exchange se desconecta después de 20 mensajes. Ambos permanecen así por debajo de cualquier valor predeterminado de Exchange.

La fila de aplicaciones es el caso problemático más habitual. Un trabajo por lotes que envía facturas, nóminas o mensajes del sistema suele crear un objeto cliente, llamar al método de envío sobre él en un bucle y cerrarlo al final. `System.Net.Mail.SmtpClient` utiliza la misma conexión desde .NET Framework 4 para llamadas consecutivas a `Send` y solo envía `QUIT` al ejecutar `Dispose`; JavaMail se comporta igual con un `Transport` abierto una vez. Si el trabajo dura más de diez minutos, en algún punto intermedio se produce el `421`, y el trabajo termina con una excepción; en .NET, por ejemplo, con el texto `Service not available, closing transmission channel. The server response was: 4.4.1 Connection timed out`. El mensaje afectado depende del tiempo de ejecución, por lo que el error parece aleatorio: unas veces hay 800 mensajes hasta la interrupción y otras 1200, según el tamaño de los mensajes y la carga del servidor.

La fila de puertas de enlace describe un caso documentado: Symantec (hoy Broadcom) Messaging Gateway envía notificaciones de cuarentena de spam mediante una única conexión y envía `NOOP` como keepalive entre mensajes. Exchange responde a `NOOP` con el retraso de tarpit de cinco segundos, de modo que en diez minutos solo pueden pasar unas 120 notificaciones antes de que la sesión termine con `421 4.4.1` y la puerta de enlace tenga que volver a conectarse.

La fila de escáneres es un problema de tamaño en vez de cantidad: un escaneo de 60 MB a través de una conexión de 2 Mbit/s requiere unos cuatro minutos solo de tiempo de transmisión; con 100 MB son casi siete minutos. En un servidor Edge Transport con un tiempo de espera total de cinco minutos, esto ya basta para provocar una interrupción; en un servidor Mailbox queda margen, pero no mucho.

## Qué ocurre durante la interrupción

Cuando vence el tiempo de espera total, Exchange escribe la respuesta `421 4.4.1 Connection timed out` en el registro de protocolo, la envía al cliente y cierra la conexión. Para la transacción en curso se aplica lo siguiente: si aún no se ha enviado el punto final, el mensaje no ha sido aceptado y debe repetirse por completo. Si se ha enviado el punto y la conexión se cerró antes de la respuesta `250`, el cliente no sabe si Exchange asumió el mensaje; un cliente correctamente implementado lo repite y el destinatario podría recibirlo por duplicado. La probabilidad es pequeña, pero no es nula con miles de mensajes por ejecución.

También hay que tener en cuenta la ruta del proxy: el servicio de transporte front-end acepta la conexión en el puerto 25 y la reenvía como una sesión SMTP propia al servicio de transporte en el puerto 2525, donde se aplica el conector `Default <Server>` con los mismos valores predeterminados. Por ello, una sesión larga aparece en ambos registros y un ajuste debe abarcar ambos conectores.

## Consultar la duración real de la sesión en el registro de protocolo

Antes de cambiar un valor, conviene revisar las sesiones reales. El requisito es disponer de registro de protocolo detallado en el conector afectado; ya está activo en `Default Frontend <Server>`, pero no en los demás conectores:

```powershell
Set-ReceiveConnector -Identity 'EX01\Relay Applikationen' -ProtocolLoggingLevel Verbose
```

Los registros se encuentran en `%ExchangeInstallPath%TransportRoles\Logs\FrontEnd\ProtocolLog\SmtpReceive` (front-end) y `%ExchangeInstallPath%TransportRoles\Logs\Hub\ProtocolLog\SmtpReceive` (servicio de transporte), y se nombran por hora UTC como `RECVyyyyMMddhh-nnnn.log`. Cada línea es un evento de protocolo con los campos `date-time`, `connector-id`, `session-id`, `sequence-number`, `local-endpoint`, `remote-endpoint`, `event`, `data` y `context`. Todas las líneas de una sesión tienen el mismo `session-id`, por lo que la duración de la sesión es la diferencia entre la primera y la última marca de tiempo de ese ID.

El siguiente script analiza el archivo de registro más reciente del día para un conector, agrupa las líneas por sesión y muestra las 15 sesiones más largas con el número de mensajes, la duración y la información de si Exchange las finalizó con `421 4.4.1`:

```powershell
$logPfad = Join-Path $env:ExchangeInstallPath 'TransportRoles\Logs\FrontEnd\ProtocolLog\SmtpReceive'
$connector = 'Relay Applikationen'
$tag = (Get-Date).ToUniversalTime().ToString('yyyyMMdd')
$datei = Get-ChildItem $logPfad -Filter "RECV$tag*.log" |
    Sort-Object Name -Descending |
    Select-Object -First 1

$sessions = @{}
Get-Content $datei.FullName |
    Where-Object { $_ -notlike '#*' -and $_ -like "*$connector*" } |
    ForEach-Object {
        $c = $_ -split ','
        $s = $c[2]
        if (-not $sessions[$s]) {
            $sessions[$s] = [pscustomobject]@{
                IP = ($c[5] -split ':')[0]; Start = $c[0]; Ende = $c[0]
                Zeilen = 0; Mails = 0; Timeout = $false
            }
        }
        $sessions[$s].Ende = $c[0]
        $sessions[$s].Zeilen++
        if ($c[7] -like 'MAIL FROM*') { $sessions[$s].Mails++ }
        if ($c[7] -like '421 4.4.1*') { $sessions[$s].Timeout = $true }
    }

$sessions.Values |
    Sort-Object Zeilen -Descending |
    Select-Object -First 15 IP, Mails, Zeilen, Timeout,
        @{ n = 'Dauer_s'
           e = { [math]::Round(([datetime]$_.Ende - [datetime]$_.Start).TotalSeconds, 1) } } |
    Format-Table -AutoSize
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Elemento | Efecto |
|---|---|
| `$logPfad` | Directorio de registros del servicio de transporte front-end; para el servicio de transporte use `Hub` en lugar de `FrontEnd` |
| `$connector` | Parte del nombre del conector; filtra por el campo `connector-id`, registrado como `Server\Name` |
| `$tag` | Fecha UTC, porque los archivos de registro se nombran por hora UTC |
| `-Filter "RECV$tag*.log"` | Solo registros de recepción del día actual |
| `Sort-Object Name -Descending`, `Select-Object -First 1` | El archivo más reciente (hora más alta, número de instancia más alto) |
| `$_ -notlike '#*'` | Omite las líneas de cabecera `#Software`, `#Version`, `#Log-Type`, `#Date`, `#Fields` |
| `$_ -split ','` | Descompone la línea CSV; los campos utilizados 0, 2, 5 y 7 se encuentran antes del primer texto libre y, por tanto, son estables |
| `$c[2]` | `session-id`, la clave de agrupación |
| `($c[5] -split ':')[0]` | Dirección IPv4 del `remote-endpoint` (para extremos IPv6 debe adaptarse la descomposición) |
| `$c[0]` como `Start` y `Ende` | Primera y última marca de tiempo de la sesión; `Ende` se sobrescribe con cada línea |
| `$c[7] -like 'MAIL FROM*'` | Cuenta mensajes mediante el comando `MAIL FROM` recibido |
| `$c[7] -like '421 4.4.1*'` | Marca las sesiones que Exchange finalizó debido al tiempo de espera total |
| `Sort-Object Zeilen -Descending` | Las sesiones más activas primero; como alternativa, ordenar por `Dauer_s` |
| `Dauer_s` | Diferencia de las marcas de tiempo ISO 8601 en segundos, redondeada a una decimal |

</details>

En la salida, puede identificar los sistemas afectados porque `Timeout` está establecido en `True` y `Dauer_s` se sitúa cerca de 600: la sesión ha vivido exactamente el tiempo que permite el conector. Las sesiones con muchos mensajes y una duración claramente inferior a 600 segundos no son críticas, aunque en ese momento sean las más largas. Para obtener una vista general de las fuentes afectadas, basta agrupar las sesiones marcadas:

```powershell
$sessions.Values |
    Where-Object { $_.Timeout } |
    Group-Object IP |
    Select-Object Name, Count |
    Sort-Object Count -Descending
```

Dos limitaciones del enfoque: una sesión que atraviesa un límite de hora se distribuye entre dos archivos de registro y aparece acortada en el archivo individual; para una evaluación diaria, lea todos los archivos del día. Además, el valor `Mails` cuenta comandos `MAIL FROM`, es decir, intentos, no mensajes aceptados.

## Ajustar valores: en qué conector y hasta qué punto

Los valores predeterminados protegen el conector orientado a Internet, donde cualquier contraparte puede ocupar conexiones. Allí no se modifican; de todos modos, un MTA externo legítimo vuelve a conectarse. Se ajusta el conector dedicado mediante el cual entregan los sistemas internos identificados. Si no existe uno, se puede crear restringido a las IP de los remitentes mediante `RemoteIPRanges`; es mejor que aumentar el valor en `Default Frontend`. El estado actual de todos los conectores se obtiene con:

```powershell
Get-ReceiveConnector |
    Format-Table Name, TransportRole, ConnectionTimeout, ConnectionInactivityTimeout, TarpitInterval -AutoSize
```

El ajuste propiamente dicho, aquí con una hora de duración total y el tiempo de espera por inactividad sin cambios:

```powershell
$werte = @{
    ConnectionTimeout           = '01:00:00'
    ConnectionInactivityTimeout = '00:05:00'
}
Set-ReceiveConnector -Identity 'EX01\Relay Applikationen' @werte
Set-ReceiveConnector -Identity 'EX01\Default EX01' @werte
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Parámetro | Efecto |
|---|---|
| `ConnectionTimeout` | Duración total de una conexión; se permite de 00:00:01 a 1.00:00:00 y debe ser superior a `ConnectionInactivityTimeout` |
| `ConnectionInactivityTimeout` | Tiempo de inactividad hasta el cierre; cinco minutos corresponden al mínimo del RFC y pueden mantenerse |
| `-Identity 'EX01\Relay Applikationen'` | El conector front-end de los remitentes internos |
| `-Identity 'EX01\Default EX01'` | El conector del servicio de transporte en el puerto 2525, al que el front-end reenvía la sesión |
| `@werte` | Splatting: pasa ambos parámetros de la tabla hash al cmdlet |

</details>

Para el valor se aplica lo siguiente: debe superar la sesión legítima más larga mostrada por el análisis, con margen para picos de carga. Una hora cubre la mayoría de los trabajos por lotes; para una ejecución nocturna de dos horas se necesita más en consecuencia, hasta el máximo de un día. Sin embargo, el valor tampoco debería ser arbitrariamente alto en un conector interno, porque `MaxInboundConnectionPerSource` (20 de forma predeterminada) y `MaxInboundConnection` (5000 de forma predeterminada) también cuentan: un cliente que, además de una conexión bloqueada, abre repetidamente otras nuevas alcanza el límite por fuente tanto antes cuanto más tiempo permanezcan abiertas las conexiones antiguas.

Para remitentes que envían `NOOP` entre mensajes, `TarpitInterval` debe establecerse en `00:00:00` en el mismo conector. El retraso de tarpit no aporta utilidad para remitentes internos autenticados o restringidos por IP y alarga artificialmente cada sesión.

El cambio en Exchange corrige el síntoma. La solución más estable está en el cliente: vuelve a establecer la conexión después de un número fijo de mensajes, como hacen Exchange con 20 y Postfix con cinco minutos. En `.NET SmtpClient` esto significa crear y descartar el objeto por bloques de, por ejemplo, 100 mensajes; en JavaMail, el `Transport` se cierra y vuelve a abrir según corresponda. Así, el envío también funciona frente a destinos cuyos tiempos de espera no se pueden ajustar, especialmente Exchange Online, cuyos conectores de entrada no conocen parámetros de tiempo de espera.

## Otros límites de tiempo en la ruta

El valor de Exchange no es el único límite. Los firewalls y los balanceadores de carga mantienen sus propios temporizadores de inactividad para conexiones TCP: un perfil FastL4 de una F5 BIG-IP está configurado de forma predeterminada en 300 segundos y un Azure Load Balancer en cuatro minutos. Estos temporizadores miden inactividad, no duración total, y por ello se activan durante pausas de envío, por ejemplo cuando un trabajo por lotes lee datos de la base de datos entre dos bloques. Siempre prevalece el valor más bajo de toda la ruta. El artículo [F5 BIG-IP como proxy de salida para envíos masivos de correo](https://rafaelpfister.ch/blog/f5-big-ip-outbound-smtp-massenversand) describe cómo dimensionar los tiempos de espera en un balanceador de carga para conexiones SMTP persistentes.

## Fuentes

1.  [Microsoft Learn: Set-ReceiveConnector](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-receiveconnector): referencia con los valores predeterminados y los rangos de valores de `ConnectionTimeout`, `ConnectionInactivityTimeout`, `TarpitInterval`, `MaxInboundConnection` y `MaxInboundConnectionPerSource` para servidores Mailbox y Edge Transport.

2.  [Microsoft Learn: Set-SendConnector](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-sendconnector): `ConnectionInactivityTimeOut` y `SmtpMaxMessagesPerConnection` en el lado de envío.

3.  [Microsoft Learn: Protocol logging](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/protocol-logging): ubicaciones de almacenamiento, nombres de archivo y estructura de campos de los registros de protocolo SMTP para front-end y servicio de transporte.

4.  [Microsoft Learn: Mail routing in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing): el servicio de transporte front-end como proxy sin estado delante del servicio de transporte.

5.  [RFC 5321, sección 4.5.3.2 Timeouts](https://www.rfc-editor.org/rfc/rfc5321.html#section-4.5.3.2): tiempos de espera mínimos por paso del protocolo, la justificación de los diez minutos después del punto final y el comportamiento con `421` en la sección 3.8.

6.  [Postfix: postconf(5)](https://www.postfix.org/postconf.5.html): `smtp_connection_reuse_time_limit` (300s) y `smtpd_timeout` como ejemplo de un MTA que mantiene las sesiones cortas por iniciativa propia.

7.  [Broadcom Knowledge Base: Quarantine notification process appears to be failing, logs may show 421 4.4.1 Connection timed out](https://knowledge.broadcom.com/external/article/154389/quarantine-notification-process-appears.html): caso documentado de una puerta de enlace que alcanza el tiempo de espera total de Exchange con keepalive `NOOP` y tarpit.

8.  [Microsoft Learn: SmtpClient Class](https://learn.microsoft.com/en-us/dotnet/api/system.net.mail.smtpclient): reutilización de la conexión mediante varias llamadas a `Send` y `QUIT` solo al ejecutar `Dispose`.
