---
title: "Analizar el flujo de correo de Exchange: Message Tracking, registros SMTP y conectores de recepción"
navTitle: "Analizar el flujo de correo"
description: "Cómo determinar sistemáticamente en Exchange OnPrem, Hybrid y Exchange Online dónde ha quedado un mensaje: consultas con ejemplos de salida, cómo leer correctamente el protocolo SMTP y los obstáculos que llevan periódicamente por pistas falsas."
date: "2026-08-11"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "22 min de lectura"
themen:
  - exchange-onprem-hybrid
  - smtp-mailflow
  - microsoft-365-exchange
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange"
  - "exchange-on-premises"
  - "exchange-hybrid"
  - "exchange-online"
protokolle:
  - "smtp"
  - "powershell"
  - "troubleshooting"
related:
  - typische-ursachen-fuer-mail-loops-und-deren-behebung
  - einliefernde-ip-adressen-aggregieren
  - mailflow-fehlersuche-kontrollierte-experimente
slug: "analizar-el-flujo-de-correo-de-exchange-message-tracking-registros-smtp-y-conectores-de"
translationId: "article-ad93c41ab6cd20e6"
draft: false
translationOf: exchange-message-tracking-und-receive-connectoren-analysieren
url: https://rafaelpfister.ch/es/blog/analizar-el-flujo-de-correo-de-exchange-message-tracking-registros-smtp-y-conectores-de
translationSourceHash: 646cb713e4dd97300a2cd068ee8f04953f2e80a99aec63ed11eddc46e1981f13
translationModel: gpt-5.6-terra
translatedAt: 2026-08-12T05:17:04.091Z
translationReview: automatic
---

# Analizar el flujo de correo de Exchange: Message Tracking, registros SMTP y conectores de recepción

La pregunta más frecuente en la operación de correo es: un mensaje no ha llegado, ¿dónde se ha quedado? El Message Tracking responde a ello de forma fiable, pero solo si sabe lo que **no** le indica. Este artículo describe el procedimiento en el orden que ha demostrado ser eficaz, muestra la salida típica de cada consulta e identifica los obstáculos que periódicamente cuestan horas porque sugieren conclusiones plausibles, pero erróneas.

Todos los ejemplos utilizan nombres genéricos: `SRV-MAIL01` y `SRV-MAIL02` como servidores de transporte, `example.com` como dominio. Si quiere crear los comandos para su entorno en lugar de escribirlos: el [generador de comandos](https://rafaelpfister.ch/tools/command-builder) contiene los comandos habituales de Message Tracking y captura para PowerShell y shell Unix uno junto a otro, íntegramente en local en el navegador.

## El principio: primero localizar, después explicar

El reflejo es buscar inmediatamente la causa. Es más eficiente determinar primero hasta dónde ha llegado el mensaje. Esto reduce drásticamente el espacio de búsqueda en un solo paso, porque después sabrá si debe buscar en su propio sistema, en el gateway anterior o en el destino.

Por tanto, el orden es: encontrar el mensaje, leer el último evento, leer el motivo del error, determinar si es un caso aislado o un patrón, y solo entonces reconstruir la ruta de entrega.

## Paso 1: Encontrar el mensaje

Empiece por el destinatario, porque casi siempre lo conoce. Es importante ejecutar la consulta en **todos** los servidores de transporte, no solo en uno.

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object {
    Get-MessageTrackingLog -Server $_ `
        -Start (Get-Date).AddHours(-6) `
        -ResultSize Unlimited `
        -Recipients "empfaenger@example.com"
} | Sort-Object Timestamp |
    Format-Table Timestamp, ServerHostname, EventId, Source, ConnectorId, MessageId `
        -AutoSize -Wrap
```

Una salida típica para un mensaje que se ha procesado correctamente:

```text
Timestamp           ServerHostname EventId      Source  ConnectorId
---------           -------------- -------      ------  -----------
11.08.2026 08:27:15 SRV-MAIL02     HARECEIVE    SMTP
11.08.2026 08:27:15 SRV-MAIL01     RECEIVE      SMTP    SRV-MAIL01\Default SRV-MAIL01
11.08.2026 08:27:15 SRV-MAIL01     HAREDIRECT   SMTP
11.08.2026 08:27:15 SRV-MAIL01     RESOLVE      ROUTING
11.08.2026 08:27:15 SRV-MAIL01     AGENTINFO    AGENT
11.08.2026 08:27:16 SRV-MAIL01     SENDEXTERNAL SMTP    Outbound-to-O365
11.08.2026 08:27:53 SRV-MAIL02     HADISCARD    SMTP
```

Si la consulta no encuentra nada, compruebe si el destinatario se expandió a través de una lista de distribución. En ese caso, es mejor buscar por remitente:

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object {
    Get-MessageTrackingLog -Server $_ `
        -Start (Get-Date).AddHours(-6) `
        -ResultSize Unlimited
} | Where-Object { $_.Sender -like "*@example.com" } |
    Sort-Object Timestamp |
    Format-Table Timestamp, EventId, Sender,
        @{n='To'; e={$_.Recipients -join ','}}, MessageSubject -AutoSize -Wrap
```

## Paso 2: Leer el último evento

Todo el diagnóstico depende del **último** `EventId` del mensaje. Le indica cuál es el siguiente espacio de búsqueda.

| Último EventId | Significado | Siguiente paso |
|---|---|---|
| `RECEIVE`, y después nada | El mensaje está retenido | Comprobar las colas |
| `SEND` o `SENDEXTERNAL` | Transferido correctamente | Seguir buscando en el siguiente salto |
| `FAIL` | Error definitivo | Leer el motivo en `RecipientStatus` |
| `DEFER` | Hay un reintento en curso | Comprobar la cola y el sistema de destino |
| `DROP` o `POISONMESSAGE` | Descartado | Regla de transporte o agente |
| `DELIVER` | Entregado a un buzón local | Comprobar las reglas del buzón |
| `RESOLVE` | Se reescribió el destinatario | Leer la dirección de destino en la entrada |

`RESOLVE` es el paso intermedio más revelador en entornos Hybrid, ya que allí se hace visible la reescritura hacia la dirección de enrutamiento de la nube:

```text
EventId : RESOLVE
Source  : ROUTING
Sender  : absender@example.com
To      : BENUTZER@example.mail.onmicrosoft.com
```

Si aparece la dirección `onmicrosoft.com` esperada, el objeto del destinatario está configurado correctamente y puede descartar el asunto. Si sigue apareciendo la dirección original, falta la dirección de destino en el objeto local y Exchange intenta entregar localmente.

Si el mensaje está retenido, la cola suele mostrar el motivo en texto claro:

```powershell
Get-Queue -Server SRV-MAIL01 |
    Where-Object { $_.MessageCount -gt 0 } |
    Format-Table Identity, DeliveryType, Status, MessageCount, NextHopDomain, LastError `
        -AutoSize -Wrap
```

## Obstáculo 1: el Tracking depende del servidor y muchas entradas son copias de sombra

Si en la salida ve pares de `HARECEIVE` y `HADISCARD`, a menudo con el añadido `ExplicitlyDiscarded`, este servidor **no procesó** el mensaje. Solo mantenía una copia de sombra en el marco de Shadow Redundancy, mientras otro servidor se encargaba de la entrega real. En cuanto el servidor primario comunica que tuvo éxito, el servidor asociado descarta su copia.

Así se ve cuando solo ha consultado el servidor equivocado:

```text
Timestamp           EventId    SourceContext
---------           -------    -------------
11.08.2026 09:47:13 HARECEIVE  1a2aae6b-f3a3-4e04-9233-7de460b92223
11.08.2026 09:48:07 HADISCARD  ExplicitlyDiscarded;1a2aae6b-f3a3-4e04-9233-7de460b92223
```

Dos líneas, sin error, sin entrega. Quien concluya de ello que el mensaje ha desaparecido está buscando en el lugar equivocado. El procesamiento real se encuentra en el Tracking del servidor asociado.

En la práctica, esto significa dos cosas. Primero, estas líneas no indican un problema, sino un funcionamiento normal. Segundo, debe consultar obligatoriamente todos los servidores de transporte.

## Obstáculo 2: `Format-Table` elimina precisamente las columnas decisivas

El motivo del error está en `RecipientStatus`, y este campo es largo. En una tabla o bien desaparece por completo o bien se trunca. Precisamente esto hace que vea el `FAIL`, pero no el motivo, y empiece a adivinar.

Por ello, en cuanto encuentre un caso de error, cambie a `Format-List` y expanda los campos de colección:

```powershell
Get-MessageTrackingLog -Server SRV-MAIL01 `
    -Start (Get-Date).AddHours(-6) `
    -ResultSize Unlimited `
    -Recipients "empfaenger@example.com" `
    -EventId FAIL |
  Format-List Timestamp, Sender,
    @{n='To';     e={$_.Recipients -join ','}},
    @{n='Status'; e={$_.RecipientStatus -join ' | '}},
    MessageSubject, MessageId, SourceContext
```

Y así se ve la diferencia. Primero la vista de tabla, que no explica nada:

```text
Timestamp           EventId ConnectorId
---------           ------- -----------
11.08.2026 09:47:13 FAIL    Outbound-to-O365
```

Después el mismo mensaje como lista:

```text
Timestamp      : 11.08.2026 09:47:13
Sender         : dienst@example-test.com
To             : BENUTZER@example.mail.onmicrosoft.com
Status         : [{LED=550 5.1.8 Access denied, bad outbound sender AS(42000001)
                 [XX1PEPF00000000.eurprd02.prod.outlook.com]};{MSG=};
                 {FQDN=10.0.0.40};{IP=10.0.0.40};{LRT=11.08.2026 09:47:13}]
MessageSubject : Statusmeldung Nachtlauf
MessageId      : <1897281176.1319@app01.intern.example.com>
```

Con ello el diagnóstico queda establecido sin haber necesitado una sola suposición: el sistema remoto objeta al remitente. `LED` contiene la respuesta SMTP completa, `FQDN` y `IP` identifican el sistema que respondió, y `LRT` el momento del último intento.

## Paso 3: ¿Caso aislado o patrón?

Antes de profundizar en un caso individual, aclare el alcance. Esta única consulta decide si se trata de una nota al margen o de un incidente:

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object {
    Get-MessageTrackingLog -Server $_ `
        -Start (Get-Date).AddHours(-8) `
        -EventId FAIL -ResultSize Unlimited
} | Where-Object { ($_.RecipientStatus -join '') -like "*5.1.8*" } |
    Group-Object { ($_.Sender -split '@')[-1] } |
    Sort-Object Count -Descending |
    Format-Table Count, Name -AutoSize
```

Sustituya `5.1.8` por el código de estado que está investigando. La salida responde a la pregunta en una línea:

```text
Count Name
----- ----
    9 example-test.com
```

Un único dominio de remitente significa: problema muy acotado, no es un incidente, puede seguir investigando tranquilamente. Si aparecieran veinte dominios distintos, tendría una interrupción en curso y todo lo demás debería esperar. Hacer esta distinción tan pronto es, por experiencia, lo que más tiempo ahorra.

## Obstáculo 3: la `ConnectorId` no revela el conector de recepción real

Este es el obstáculo más costoso porque la salida parece fiable. El correo que un cliente o un sistema externo entrega en el puerto 25 llega primero al **Front End Transport**. Este reenvía el mensaje al **Transport Service** en el puerto 2525. El Message Tracking solo se escribe allí; el Front End Transport no escribe su propio Tracking.

La consecuencia se aprecia en esta línea:

```text
EventId        : RECEIVE
ConnectorId    : SRV-MAIL01\Default SRV-MAIL01
ClientIp       : 10.0.1.11
ClientHostname : srv-mail01.intern.example.com
```

La `ConnectorId` indica el conector interno en el puerto 2525, y la `ClientIp` es la dirección del **servidor que actúa como proxy**, no la del remitente original. El Tracking simplemente no indica cuál de los conectores configurados en el puerto 25 alcanzó realmente un sistema. Quien confíe en ese dato buscará el error en un conector que ni siquiera participó.

Hay dos formas de obtener la respuesta. La primera es reconstruirlo a partir de la configuración:

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object { Get-ReceiveConnector -Server $_ } |
    Format-List Identity, Enabled,
        @{n='Bindings';       e={$_.Bindings -join ','}},
        @{n='RemoteIPRanges'; e={$_.RemoteIPRanges -join ','}},
        PermissionGroups, AuthMechanism
```

```text
Identity         : SRV-MAIL01\Default Frontend SRV-MAIL01
Bindings         : 10.0.1.11:25
RemoteIPRanges   : 0.0.0.0-255.255.255.255
PermissionGroups : AnonymousUsers, ExchangeServers, ExchangeLegacyServers
AuthMechanism    : Tls, Integrated, BasicAuth, BasicAuthRequireTLS, ExchangeServer

Identity         : SRV-MAIL01\smtp-noauth SRV-MAIL01
Bindings         : 10.0.1.13:25
RemoteIPRanges   : 10.0.20.22,10.0.21.11,10.0.21.12
PermissionGroups : AnonymousUsers, Custom
AuthMechanism    : Tls
```

Determine la IP de origen del sistema que entrega el mensaje y busque el conector cuyo `RemoteIPRanges` la contenga. Si no pertenece a ninguno de los conectores restringidos, queda el conector frontend predeterminado, que normalmente acepta todo el espacio de direcciones. También aquí use `Format-List`, pues `RemoteIPRanges` y `PermissionGroups` se truncan periódicamente en las tablas.

La segunda forma es el protocolo SMTP, que merece su propia sección.

## El protocolo SMTP: el único lugar con toda la verdad

El protocolo del Front End Transport registra la sesión SMTP completa: a qué conector se dirigió, qué IP se conectó y qué se dijeron el cliente y el servidor. Es la única fuente que resuelve el obstáculo anterior.

### Activar el registro

De forma predeterminada, el registro está **desactivado** en la mayoría de los conectores. Se activa por conector:

```powershell
Set-ReceiveConnector -Identity "SRV-MAIL01\Default Frontend SRV-MAIL01" `
    -ProtocolLoggingLevel Verbose
```

Para conexiones salientes, de forma correspondiente mediante `Set-SendConnector`. Recuerde volver a establecer el valor en `None` después del análisis, ya que el registro detallado consume espacio en disco y genera cantidades considerables de datos con alto volumen.

### Dónde se encuentran los archivos

Exchange separa los protocolos por servicio y dirección. No es necesario fijar las rutas de forma rígida: consúltelas:

```powershell
Get-FrontendTransportService SRV-MAIL01 |
    Format-List ReceiveProtocolLogPath, SendProtocolLogPath,
        ReceiveProtocolLogMaxAge, ReceiveProtocolLogMaxDirectorySize

Get-TransportService SRV-MAIL01 |
    Format-List ReceiveProtocolLogPath, SendProtocolLogPath
```

Normalmente se encuentran bajo la ruta de instalación en `TransportRoles\Logs\FrontEnd\ProtocolLog\SmtpReceive` para Front End Transport y en `TransportRoles\Logs\Hub\ProtocolLog\SmtpReceive` para Transport Service. **Este es el punto clave:** las conexiones de clientes en el puerto 25 se encuentran exclusivamente en la ruta `FrontEnd`; la ruta `Hub` solo contiene el tráfico interno de reenvío en 2525.

Tenga en cuenta la retención. `ReceiveProtocolLogMaxAge` suele estar configurado en 30 días, y `ReceiveProtocolLogMaxDirectorySize` limita adicionalmente el uso de espacio. Con mucho volumen, la limitación por tamaño entra en vigor mucho antes que el límite de antigüedad, y entonces sus protocolos solo tendrán unos pocos días.

### Entender el formato

Los archivos son CSV con líneas de encabezado que comienzan por `#`. Las columnas principales son `date-time`, `connector-id`, `session-id`, `sequence-number`, `local-endpoint`, `remote-endpoint`, `event` y `data`.

La columna decisiva es `event`, un único carácter:

| Carácter | Significado |
|---|---|
| `+` | Conexión establecida |
| `-` | Conexión finalizada |
| `>` | El servidor envía al cliente |
| `<` | El cliente envía al servidor |
| `*` | Información del servidor, sin tráfico SMTP |

Puede identificar una sesión por el mismo `session-id`; `sequence-number` indica el orden dentro de la sesión. Un extracto típico tiene este aspecto:

```text
2026-08-11T09:47:10.4Z,SRV-MAIL01\smtp-noauth SRV-MAIL01,08DEF44EC454A414,0,
  10.0.1.13:25,10.0.20.22:51244,+,,
2026-08-11T09:47:10.4Z,SRV-MAIL01\smtp-noauth SRV-MAIL01,08DEF44EC454A414,1,
  10.0.1.13:25,10.0.20.22:51244,>,"220 srv-mail01.intern.example.com Microsoft ESMTP",
2026-08-11T09:47:10.5Z,SRV-MAIL01\smtp-noauth SRV-MAIL01,08DEF44EC454A414,2,
  10.0.1.13:25,10.0.20.22:51244,<,EHLO app01.intern.example.com,
2026-08-11T09:47:10.6Z,SRV-MAIL01\smtp-noauth SRV-MAIL01,08DEF44EC454A414,6,
  10.0.1.13:25,10.0.20.22:51244,<,MAIL FROM:<dienst@example-test.com>,
2026-08-11T09:47:10.7Z,SRV-MAIL01\smtp-noauth SRV-MAIL01,08DEF44EC454A414,8,
  10.0.1.13:25,10.0.20.22:51244,>,"250 2.1.5 Recipient OK",
```

Aquí está todo lo que faltaba en el Message Tracking: el conector **real** (`smtp-noauth`), la IP de origen **real** (`10.0.20.22`) y el nombre con el que el sistema se identifica en `EHLO`.

### Buscar específicamente

Para casos individuales, un filtro de texto es considerablemente más rápido que evaluar objetos. Busque la dirección del remitente o el nombre `EHLO` y obtenga el identificador de sesión:

```powershell
$pfad = (Get-FrontendTransportService SRV-MAIL01).ReceiveProtocolLogPath
Select-String -Path "$pfad\*.log" -Pattern "dienst@example-test.com" -SimpleMatch |
    Select-Object -First 5 Line
```

Con el `session-id` encontrado, obtendrá la sesión completa:

```powershell
Select-String -Path "$pfad\*.log" -Pattern "08DEF44EC454A414" -SimpleMatch |
    ForEach-Object { $_.Line } | Select-Object -First 40
```

Si solo quiere saber qué conectores ven tráfico, cuente los establecimientos de conexión. En archivos grandes esto es órdenes de magnitud más rápido que analizar cada línea:

```powershell
Select-String -Path "$pfad\*.log" -Pattern ',\+,' |
    ForEach-Object { ($_.Line -split ',')[1] } |
    Group-Object | Sort-Object Count -Descending |
    Format-Table Count, Name -AutoSize
```

```text
Count Name
----- ----
51479 SRV-MAIL01\Default Frontend SRV-MAIL01
50756 SRV-MAIL01\smtp-auth SRV-MAIL01
19405 SRV-MAIL01\smtp-intern SRV-MAIL01
15789 SRV-MAIL01\smtp-noauth SRV-MAIL01
```

Esta distribución responde a una pregunta que no puede responderse con Message Tracking: ¿qué rutas utilizan realmente sus aplicaciones? Antes de cambiar un conector, es la cifra más importante.

### Si no se registró nada

Si no hay ninguna línea en el momento en cuestión, existen tres motivos habituales: el registro estaba desactivado en el conector correspondiente, el límite de retención ya desplazó el archivo o está mirando en la ruta equivocada, es decir, en el directorio `Hub` en lugar del `FrontEnd`. Compruébelos en este orden.

## Paso 4: Comprobar permisos

Si se rechaza una entrega o, por el contrario, se permite más de lo esperado, el camino pasa por los permisos del conector. Aquí acecha un obstáculo técnico: `Get-ADPermission` requiere el **DistinguishedName**. Si pasa la identidad habitual con la forma `Server\Connectorname`, la llamada falla en una sesión remota con el engañoso mensaje de que no se encuentra el objeto.

```powershell
$dn = (Get-ReceiveConnector "SRV-MAIL01\Default Frontend SRV-MAIL01").DistinguishedName
Get-ADPermission -Identity $dn -User "NT AUTHORITY\ANONYMOUS LOGON" |
    Where-Object { $_.ExtendedRights -like "*ms-Exch-SMTP*" } |
    Format-Table User, @{n='Rights'; e={$_.ExtendedRights}} -AutoSize
```

```text
User                         Rights
----                         ------
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Submit
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Accept-Any-Sender
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Accept-Authoritative-Domain-Sender
```

La evaluación es más sencilla de lo que parece si distingue cuatro permisos:

| Permiso | Significado |
|---|---|
| `ms-Exch-SMTP-Submit` | Puede entregar mensajes en primer lugar |
| `ms-Exch-SMTP-Accept-Any-Sender` | Puede utilizar direcciones de remitente arbitrarias |
| `ms-Exch-SMTP-Accept-Authoritative-Domain-Sender` | Puede presentarse como su propio dominio |
| `ms-Exch-SMTP-Accept-Any-Recipient` | **Puede reenviar a dominios externos** |

Los tres primeros son el conjunto estándar necesario para la entrega anónima y la recepción de correo de Internet. Solo el cuarto permiso convierte un conector de entrada en un relay. En un conector que acepta desde todo el espacio de direcciones, es un relay abierto. En un conector con restricción de IP estricta, en cambio, es la forma habitual y prevista para que los servidores de aplicaciones puedan enviar al exterior.

No confunda `Accept-Any-Sender` con `Accept-Any-Recipient`. El primero es inocuo y necesario; el segundo es la configuración relevante para la seguridad.

## Paso 5: Verificación mediante entrega propia

Si el análisis sigue siendo ambiguo, entregue usted mismo un mensaje. Así controla por completo el remitente, el destinatario y el punto de entrega:

```powershell
Send-MailMessage -SmtpServer 10.0.1.11 -Port 25 `
    -From 'test@example.com' `
    -To 'empfaenger@example.net' `
    -Subject 'Diagnose, bitte ignorieren' `
    -Body 'Testeinlieferung' `
    -Encoding UTF8
```

`Send-MailMessage` está oficialmente descontinuado, pero para fines de diagnóstico sigue siendo la herramienta más rápida y está disponible en todos los servidores Windows. En caso de éxito no muestra salida, lo cual requiere acostumbrarse.

Si prueba una ruta TLS en el puerto 587 y el sistema remoto presenta un certificado que no coincide con el nombre utilizado, por ejemplo porque se dirige a la dirección IP, la llamada falla con un error de certificado. Para la prueba puede suspender la validación en la sesión:

```powershell
[Net.ServicePointManager]::ServerCertificateValidationCallback = { $true }
```

Esto solo se aplica a la sesión actual de PowerShell. Establézcalo conscientemente y nunca en scripts que se ejecuten en producción.

Si llega el mensaje de prueba y quiere saber qué le ocurrió por el camino, el [analizador de encabezados de correo](https://rafaelpfister.ch/tools/header-analyzer) ayuda: descompone los encabezados, traza el recorrido por los saltos y muestra los resultados de las comprobaciones de autenticación, íntegramente en local en el navegador, sin que el mensaje abandone su dispositivo.

## Exchange Online: la misma pregunta, otra herramienta

En el tenant se aplican otras reglas, y aquí es donde fallan los procedimientos habituales. Cuente con estas diferencias:

| | Exchange OnPrem | Exchange Online |
|---|---|---|
| Consulta | `Get-MessageTrackingLog` | `Get-MessageTraceV2` |
| Granularidad | Cada evento de transporte | Una línea por mensaje y destinatario |
| Conector visible | Sí (con limitación, véase arriba) | **No** |
| Dependencia de servidor | Sí, consultar por servidor | No aplica |
| Protocolo SMTP | Disponible | **No disponible** |
| Retención | Su configuración | Unos 10 días mediante el cmdlet |
| Retraso | Casi inmediato | Algunos minutos |

Las tres consecuencias más importantes en la práctica: **no hay asignación de conector**, debe recurrir a `FromIP` y `ToIP`. **No hay protocolo SMTP**, la conversación SMTP no se puede reconstruir. Y los datos aparecen **con retraso**, un mensaje recién enviado no aparece inmediatamente.

### La consulta básica

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-4) `
    -EndDate (Get-Date) `
    -RecipientAddress "empfaenger@example.com" `
    -ResultSize 1000 |
  Sort-Object Received |
  Format-Table Received, SenderAddress, RecipientAddress, Status, FromIP, Size -AutoSize
```

```text
Received            SenderAddress          RecipientAddress          Status    FromIP
--------            -------------          ----------------          ------    ------
11.08.2026 08:27:16 emma@partner.example   empfaenger@example.com    Delivered 10.0.20.23
11.08.2026 09:05:24 dienst@example-test.com empfaenger@example.com   Failed    10.0.20.23
```

Los valores principales de `Status` son: `Delivered`, `Failed`, `Pending`, `Quarantined`, `FilteredAsSpam` y `Expanded` para listas de distribución expandidas. `Pending` significa que todavía hay intentos de entrega en curso, no que algo esté roto.

### Los detalles de un mensaje

El estado por sí solo no dice nada sobre el motivo. Para ello necesita la vista detallada, que requiere el identificador del mensaje de la consulta básica:

```powershell
$n = Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-4) -EndDate (Get-Date) `
        -RecipientAddress "empfaenger@example.com" -ResultSize 100 |
     Where-Object { $_.Status -eq 'Failed' } | Select-Object -First 1

Get-MessageTraceDetailV2 -MessageTraceId $n.MessageTraceId `
    -RecipientAddress $n.RecipientAddress |
  Format-List Date, Event, Action, Detail
```

Allí figuran los pasos de procesamiento en el servicio, como aplicaciones de reglas, decisiones de filtrado y el motivo de un rechazo.

### Más allá de diez días

El cmdlet cubre unos diez días. Para períodos más antiguos existe la búsqueda histórica, que se ejecuta de forma asíncrona y proporciona el resultado como CSV, con un intervalo de hasta 90 días:

```powershell
Start-HistoricalSearch -ReportTitle "Analyse Nachtlauf" `
    -StartDate (Get-Date).AddDays(-45) `
    -EndDate (Get-Date).AddDays(-30) `
    -ReportType MessageTrace `
    -SenderAddress "dienst@example-test.com" `
    -NotifyAddress "admin@example.com"

Get-HistoricalSearch | Format-Table JobId, ReportTitle, Status, SubmitDate -AutoSize
```

Reserve tiempo: según el alcance, estas tareas pueden ejecutarse durante horas.

### Obstáculo 4: la ausencia de resultados no demuestra la ausencia de tráfico

Este es el obstáculo más sutil en el tenant. `Get-MessageTraceV2` devuelve páginas de un máximo de 5000 líneas por llamada. Con mucho volumen, una página puede cubrir solo unos minutos aunque haya consultado siete días. Si después filtra localmente, por ejemplo por una IP de origen, estará filtrando sobre un extracto muy reducido.

Esto se reconoce por la advertencia que indica que hay más resultados:

```text
WARNING: There are more results, use the following command to get more.
Get-MessageTraceV2 -StartDate "2026-08-11T07:25:19Z" -EndDate "2026-08-11T09:05:46Z"
  -StartingRecipientAddress "naechster@example.com" -ResultSize 5000
```

Si aparece, su análisis está **incompleto**. Si no devuelve ningún resultado, la conclusión correcta es: no se encontró en el extracto. No es: no existe.

Hay dos salidas correctas. O bien reduce la ventana temporal hasta que una página la cubra por completo, lo que se reconoce por la ausencia de la advertencia. O bien recorre todas las páginas mediante la información de continuación de la advertencia. Para responder si algo **nunca** ocurre, una comprobación de configuración es de todos modos superior: si un sistema no dispone de una ruta hacia un destino, no puede entregar allí, independientemente de cualquier ventana de observación.

La evaluación completa de todas las direcciones que entregan correo es un tema aparte, con sus propios obstáculos de interpretación. Se explica en [¿Quién entrega realmente en su tenant? Agregar direcciones IP de entrega](https://rafaelpfister.ch/blog/einliefernde-ip-adressen-aggregieren).

## Un procedimiento que ha demostrado ser eficaz

En resumen, este orden ha demostrado ser el más rápido. Busque el mensaje en todos los servidores y determine el último evento. En caso de fallo, cambie inmediatamente a `Format-List` y lea la respuesta SMTP completa, en vez de deducir a partir del tipo de evento. Después aclare el alcance, es decir, agrupe y cuente. Solo cuando el caso esté bien delimitado, reconstruya la ruta de entrega mediante la configuración de conectores y el protocolo SMTP. Por último, si es necesario, verifique con una entrega propia.

Los mayores consumidores de tiempo suelen ser siempre los mismos: leer una tabla truncada en lugar del mensaje de error completo, considerar las copias de sombra como pasos de procesamiento, creer en la `ConnectorId` del Tracking y tomar una muestra vacía como prueba. Quien conoce estos cuatro puntos normalmente llega al nivel correcto en pocos minutos.

## Fuentes

1.  [Message tracking in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): descripción de campos y lista completa de tipos de eventos en Message Tracking.

2.  [Protocol logging in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/protocol-logging): ubicaciones, formato y retención de los protocolos SMTP, incluido Front End Transport.

3.  [Shadow redundancy in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/shadow-redundancy): explica los eventos relacionados con las copias de sombra y su descarte.

4.  [Mail routing in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing): interacción entre Front End Transport y Transport Service, base del comportamiento de proxy.

5.  [Receive connectors in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/receive-connectors): enlaces, grupos de permisos y mecanismos de autenticación.

6.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2): sucesor de Get-MessageTrace, incluida la lógica de paginación y la lista de campos.

7.  [Start-HistoricalSearch](https://learn.microsoft.com/en-us/powershell/module/exchange/start-historicalsearch): seguimiento asíncrono de mensajes durante hasta 90 días.
