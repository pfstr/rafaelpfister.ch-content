---
title: "Analizar el flujo de correo de Exchange: Message Tracking, protocolos SMTP y conectores de recepción"
navTitle: "Analizar el flujo de correo"
description: "Cómo determinar sistemáticamente en Exchange OnPrem, Hybrid y Exchange Online dónde ha quedado un mensaje: las consultas con ejemplos de salida, cómo leer correctamente el protocolo SMTP y los puntos que llevan regularmente a conclusiones erróneas."
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
translationSourceHash: 867f3a9a27b8db05d2721cce6282149f4616c4170576d0ab3f39a3cbde3701e3
translationModel: gpt-5.6-terra
translatedAt: 2026-10-30T03:26:29.475Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/analizar-el-flujo-de-correo-de-exchange-message-tracking-registros-smtp-y-conectores-de
---

# Analizar el flujo de correo de Exchange: Message Tracking, protocolos SMTP y conectores de recepción

La pregunta más frecuente en la operación de correo es: un mensaje no ha llegado, ¿dónde se ha quedado? Message Tracking responde a esto de forma fiable, pero solo si sabe lo que **no** le indica. Este artículo describe el procedimiento en el orden que ha demostrado ser eficaz, muestra la salida típica de cada consulta e identifica las fuentes de error que regularmente cuestan horas porque sugieren conclusiones plausibles, pero falsas.

Todos los ejemplos utilizan nombres genéricos: `SRV-MAIL01` y `SRV-MAIL02` como servidores de transporte, `example.com` como dominio. Si desea generar los comandos para su entorno en lugar de escribirlos: el [generador de comandos](https://rafaelpfister.ch/tools/command-builder) incluye los comandos habituales de Message Tracking y captura para PowerShell y shell Unix en paralelo, completamente local en el navegador.

## El principio: primero localizar, después explicar

El reflejo es buscar inmediatamente la causa. Es más eficiente determinar primero hasta dónde ha llegado realmente el mensaje. Esto reduce drásticamente el espacio de búsqueda en un solo paso, porque después sabrá si debe buscar en su propio sistema, en el gateway anterior o en el destino.

Por tanto, el orden es: encontrar el mensaje, leer el último evento, leer la causa del error, determinar si es un caso aislado o un patrón y solo entonces reconstruir la vía de entrega.

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

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Server` | Servidor de transporte cuyo registro de seguimiento se consulta; aquí ambos servidores se recorren secuencialmente mediante la canalización |
| `-Start` | Límite temporal inferior de la búsqueda, aquí las últimas seis horas |
| `-ResultSize Unlimited` | Elimina el límite predeterminado de 1000 entradas |
| `-Recipients` | Filtra mensajes enviados a esta dirección de destinatario |
| `Sort-Object Timestamp` | Ordena cronológicamente los resultados combinados de ambos servidores |
| `-AutoSize -Wrap` | Ajusta el ancho de columna al contenido y ajusta valores largos en lugar de truncarlos |

</details>

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

Si la consulta no encuentra nada, compruebe si el destinatario se expandió mediante una lista de distribución. En ese caso, es mejor buscar por el remitente:

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

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Server` | Servidor de transporte cuyo registro de seguimiento se consulta |
| `-Start` | Límite temporal inferior de la búsqueda |
| `-ResultSize Unlimited` | Elimina el límite predeterminado de 1000 entradas |
| `Where-Object` | Filtra en el cliente por remitentes del dominio propio, ya que `-Sender` solo acepta direcciones exactas |
| `@{n=…; e=…}` | Columna calculada: resume el campo de colección `Recipients` en una cadena separada por comas |

</details>

## Paso 2: Leer el último evento

Todo el diagnóstico depende del **último** `EventId` del mensaje. Le indica cuál es el siguiente espacio de búsqueda.

| Último EventId | Significado | Siguiente paso |
|---|---|---|
| `RECEIVE`, después nada | El mensaje está bloqueado | Comprobar las colas |
| `SEND` o `SENDEXTERNAL` | Transferido correctamente | Seguir buscando en el siguiente salto |
| `FAIL` | Fallo definitivo | Leer la causa en `RecipientStatus` |
| `DEFER` | Hay un reintento en curso | Comprobar la cola y el sistema de destino |
| `DROP` o `POISONMESSAGE` | Descartado | Regla de transporte o agente |
| `DELIVER` | Entregado en un buzón local | Comprobar las reglas del buzón |
| `RESOLVE` | Se ha reescrito el destinatario | Leer la dirección de destino en la entrada |

`RESOLVE` es el paso intermedio más revelador en entornos Hybrid, porque ahí se ve la reescritura a la dirección de enrutamiento en la nube:

```text
EventId : RESOLVE
Source  : ROUTING
Sender  : absender@example.com
To      : BENUTZER@example.mail.onmicrosoft.com
```

Si aparece la dirección `onmicrosoft.com` esperada, el objeto de destinatario está configurado correctamente y puede dar el tema por resuelto. Si sigue apareciendo la dirección original, falta la dirección de destino en el objeto local y Exchange intenta realizar la entrega localmente.

Si el mensaje está bloqueado, la cola suele mostrar la causa en texto claro:

```powershell
Get-Queue -Server SRV-MAIL01 |
    Where-Object { $_.MessageCount -gt 0 } |
    Format-Table Identity, DeliveryType, Status, MessageCount, NextHopDomain, LastError `
        -AutoSize -Wrap
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Server` | Servidor cuyas colas de transporte se consultan |
| `Where-Object` | Oculta las colas vacías y muestra solo las que tienen mensajes en espera |
| `-AutoSize -Wrap` | Evita que se trunque la columna larga `LastError` |

</details>

## Fuente de error 1: el seguimiento depende del servidor y muchas entradas son copias sombra

Si en la salida ve pares de `HARECEIVE` y `HADISCARD`, a menudo con el añadido `ExplicitlyDiscarded`, ese servidor **no ha procesado** el mensaje. Solo mantenía una copia sombra en el marco de Shadow Redundancy, mientras otro servidor realizaba la entrega real. En cuanto el servidor principal informa de éxito, el asociado descarta su copia.

Así se ve cuando solo ha consultado el servidor equivocado:

```text
Timestamp           EventId    SourceContext
---------           -------    -------------
11.08.2026 09:47:13 HARECEIVE  1a2aae6b-f3a3-4e04-9233-7de460b92223
11.08.2026 09:48:07 HADISCARD  ExplicitlyDiscarded;1a2aae6b-f3a3-4e04-9233-7de460b92223
```

Dos líneas, sin error ni entrega. Quien concluya de ello que el mensaje ha desaparecido busca en el lugar equivocado. El procesamiento real se encuentra en el seguimiento del servidor asociado.

En la práctica, esto significa dos cosas. Primero, estas líneas no indican un problema, sino funcionamiento normal. Segundo, debe consultar obligatoriamente todos los servidores de transporte.

## Fuente de error 2: `Format-Table` trunca precisamente las columnas decisivas

La causa del error se encuentra en `RecipientStatus`, y este campo es largo. En una tabla, o bien desaparece por completo o se trunca. Precisamente esto hace que vea el `FAIL`, pero no la causa, y empiece a adivinar.

Por tanto, en cuanto encuentre un caso de error, cambie a `Format-List` y expanda los campos de colección:

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

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Server` | Servidor de transporte cuyo registro de seguimiento se consulta |
| `-Start` | Límite temporal inferior de la búsqueda |
| `-ResultSize Unlimited` | Elimina el límite predeterminado de 1000 entradas |
| `-Recipients` | Filtra mensajes enviados a esta dirección de destinatario |
| `-EventId FAIL` | Solo entradas con error definitivo de entrega |
| `Format-List` | Muestra cada campo en su propia línea y en toda su longitud; no se trunca nada |
| `@{n=…; e=…}` | Campos calculados: convierten los campos de colección `Recipients` y `RecipientStatus` en cadenas legibles |

</details>

Y así se aprecia la diferencia. Primero la vista de tabla, que no explica nada:

```text
Timestamp           EventId ConnectorId
---------           ------- -----------
11.08.2026 09:47:13 FAIL    Outbound-to-O365
```

Luego el mismo mensaje como lista:

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

Con ello el diagnóstico queda claro sin necesidad de hacer una sola suposición: el extremo remoto objeta al remitente. `LED` contiene la respuesta SMTP completa, `FQDN` y `IP` identifican el sistema que respondió, y `LRT` el momento del último intento.

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

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Start` | Límite temporal inferior, aquí las últimas ocho horas |
| `-EventId FAIL` | Solo entregas con fallo definitivo |
| `-ResultSize Unlimited` | Elimina el límite predeterminado de 1000 entradas |
| `Where-Object` | Filtra por el código de estado SMTP analizado en el campo `RecipientStatus` |
| `Group-Object` | Agrupa por dominio del remitente, la parte posterior a `@` |
| `Sort-Object Count -Descending` | El dominio más frecuente arriba |

</details>

Sustituya `5.1.8` por el código de estado que está investigando. La salida responde a la pregunta en una sola línea:

```text
Count Name
----- ----
    9 example-test.com
```

Un único dominio de remitente significa: problema muy delimitado, no es un incidente; puede seguir investigando con calma. Si aparecieran veinte dominios distintos, tendría una interrupción en curso y todo lo demás debería esperar. Hacer esta distinción tan pronto es, por experiencia, lo que más tiempo ahorra.

## Fuente de error 3: el `ConnectorId` no identifica el Receive Connector real

Esta es la fuente de error más costosa, porque la salida parece fiable. El correo que un cliente o sistema externo entrega en el puerto 25 llega primero al **Front End Transport**. Este reenvía el mensaje al **Transport Service** en el puerto 2525. Message Tracking solo se escribe allí; Front End Transport no escribe su propio seguimiento.

La consecuencia se aprecia en esta línea:

```text
EventId        : RECEIVE
ConnectorId    : SRV-MAIL01\Default SRV-MAIL01
ClientIp       : 10.0.1.11
ClientHostname : srv-mail01.intern.example.com
```

El `ConnectorId` identifica el conector interno del puerto 2525, y el `ClientIp` es la dirección del **servidor proxy**, no la del remitente original. Message Tracking simplemente no indica cuál de los conectores configurados en el puerto 25 alcanzó realmente un sistema. Quien crea este dato busca el error en un conector que ni siquiera ha participado.

Hay dos formas de obtener la respuesta. La primera es reconstruirlo mediante la configuración:

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object { Get-ReceiveConnector -Server $_ } |
    Format-List Identity, Enabled,
        @{n='Bindings';       e={$_.Bindings -join ','}},
        @{n='RemoteIPRanges'; e={$_.RemoteIPRanges -join ','}},
        PermissionGroups, AuthMechanism
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Server` | Servidor cuyos Receive Connectors se enumeran |
| `Format-List` | Longitudes completas de campo; `RemoteIPRanges` y `PermissionGroups` se truncarían en tablas |
| `@{n=…; e=…}` | Campos calculados: resumen los campos de colección `Bindings` y `RemoteIPRanges` en cadenas separadas por comas |

</details>

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

Determine la IP de origen del sistema que realiza la entrega y busque el conector cuyo `RemoteIPRanges` la contenga. Si no pertenece a ninguno de los conectores restringidos, queda el conector frontend predeterminado, que normalmente acepta todo el espacio de direcciones. También aquí utilice `Format-List`, pues `RemoteIPRanges` y `PermissionGroups` se truncan regularmente en tablas.

La segunda vía es el protocolo SMTP, que merece una sección propia.

## El protocolo SMTP: la única fuente completa

El protocolo de Front End Transport registra toda la sesión SMTP: a qué conector se dirigió, qué IP se conectó y qué se dijeron cliente y servidor. Es la única fuente que resuelve el problema descrito anteriormente con el `ConnectorId`.

### Activar el registro

De forma predeterminada, el registro está **desactivado** en la mayoría de los conectores. Se activa por conector:

```powershell
Set-ReceiveConnector -Identity "SRV-MAIL01\Default Frontend SRV-MAIL01" `
    -ProtocolLoggingLevel Verbose
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Identity` | El conector que se debe modificar en la forma `Server\Connectorname` |
| `-ProtocolLoggingLevel Verbose` | Activa el registro SMTP; `None` lo vuelve a desactivar |

</details>

Para conexiones salientes, haga lo mismo con `Set-SendConnector`. Recuerde volver a establecer el valor en `None` después del análisis, pues el registro detallado consume espacio en disco y genera cantidades considerables de datos con un volumen elevado.

### Dónde se encuentran los archivos

Exchange separa los protocolos por servicio y dirección. No es necesario codificar las rutas de forma fija: consúltelas:

```powershell
Get-FrontendTransportService SRV-MAIL01 |
    Format-List ReceiveProtocolLogPath, SendProtocolLogPath,
        ReceiveProtocolLogMaxAge, ReceiveProtocolLogMaxDirectorySize

Get-TransportService SRV-MAIL01 |
    Format-List ReceiveProtocolLogPath, SendProtocolLogPath
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `SRV-MAIL01` | Parámetro posicional `-Identity`: el servidor que se consulta |
| `ReceiveProtocolLogPath`, `SendProtocolLogPath` | Rutas de almacenamiento de los protocolos para conexiones entrantes y salientes, respectivamente |
| `ReceiveProtocolLogMaxAge` | Antigüedad máxima de los archivos de protocolo; los más antiguos se eliminan |
| `ReceiveProtocolLogMaxDirectorySize` | Límite superior de uso de espacio del directorio de protocolos |

</details>

Normalmente se encuentran bajo la ruta de instalación en `TransportRoles\Logs\FrontEnd\ProtocolLog\SmtpReceive` para Front End Transport y en `TransportRoles\Logs\Hub\ProtocolLog\SmtpReceive` para Transport Service. **Este es el punto central:** las conexiones de cliente en el puerto 25 solo se encuentran en la ruta `FrontEnd`; la ruta `Hub` contiene únicamente el tráfico interno de reenvío en 2525.

Tenga en cuenta la retención. `ReceiveProtocolLogMaxAge` suele estar configurado en 30 días, mientras que `ReceiveProtocolLogMaxDirectorySize` limita adicionalmente el uso de espacio. Con un volumen elevado, el límite de tamaño se alcanza mucho antes que el límite de antigüedad y entonces sus protocolos solo tendrán unos pocos días.

### Comprender el formato

Los archivos son CSV con cabeceras que comienzan con `#`. Las columnas más importantes son `date-time`, `connector-id`, `session-id`, `sequence-number`, `local-endpoint`, `remote-endpoint`, `event` y `data`.

La columna decisiva es `event`, un único carácter:

| Carácter | Significado |
|---|---|
| `+` | Conexión establecida |
| `-` | Conexión finalizada |
| `>` | El servidor envía al cliente |
| `<` | El cliente envía al servidor |
| `*` | Información del servidor, sin tráfico SMTP |

Puede reconocer una sesión por el valor común `session-id`; `sequence-number` indica el orden dentro de la sesión. Un fragmento típico tiene este aspecto:

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

Aquí figura todo lo que faltaba en Message Tracking: el conector **real** (`smtp-noauth`), la IP de origen **real** (`10.0.20.22`) y el nombre con el que el sistema se identifica en `EHLO`.

### Buscar de forma específica

Para casos individuales, un filtro de texto es mucho más rápido que una evaluación de objetos. Busque la dirección del remitente o el nombre de `EHLO` y obtenga el identificador de sesión:

```powershell
$pfad = (Get-FrontendTransportService SRV-MAIL01).ReceiveProtocolLogPath
Select-String -Path "$pfad\*.log" -Pattern "dienst@example-test.com" -SimpleMatch |
    Select-Object -First 5 Line
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Path "$pfad\*.log"` | Busca en todos los archivos de protocolo de la ruta consultada anteriormente |
| `-Pattern` | El término de búsqueda, aquí la dirección del remitente |
| `-SimpleMatch` | Trata el patrón como texto en lugar de expresión regular; así el punto de la dirección no requiere escape |
| `-First 5` | Limita la salida a los cinco primeros resultados |

</details>

Con el `session-id` encontrado, obtenga la sesión completa:

```powershell
Select-String -Path "$pfad\*.log" -Pattern "08DEF44EC454A414" -SimpleMatch |
    ForEach-Object { $_.Line } | Select-Object -First 40
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Pattern` | El identificador de sesión del primer resultado |
| `-SimpleMatch` | Búsqueda literal sin evaluación de expresiones regulares |
| `-First 40` | Limita la salida a las primeras 40 líneas de la sesión |

</details>

Si solo quiere saber qué conectores reciben tráfico, cuente los establecimientos de conexión. Con archivos grandes es varios órdenes de magnitud más rápido que analizar cada línea:

```powershell
Select-String -Path "$pfad\*.log" -Pattern ',\+,' |
    ForEach-Object { ($_.Line -split ',')[1] } |
    Group-Object | Sort-Object Count -Descending |
    Format-Table Count, Name -AutoSize
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Pattern ',\+,'` | Expresión regular para el evento `+` (establecimiento de conexión) entre dos comas CSV; el signo más está escapado |
| `ForEach-Object { … -split ',' }` | Divide la línea coincidente por comas y toma la segunda columna, `connector-id` |
| `Group-Object` | Cuenta los establecimientos de conexión por conector |
| `Sort-Object Count -Descending` | El conector más utilizado arriba |

</details>

```text
Count Name
----- ----
51479 SRV-MAIL01\Default Frontend SRV-MAIL01
50756 SRV-MAIL01\smtp-auth SRV-MAIL01
19405 SRV-MAIL01\smtp-intern SRV-MAIL01
15789 SRV-MAIL01\smtp-noauth SRV-MAIL01
```

Esta distribución responde a una pregunta que no puede responderse con Message Tracking: ¿qué vías utilizan realmente sus aplicaciones? Antes de modificar un conector, es la cifra más importante de todas.

### Si no se registró nada

Si falta cualquier línea en el momento en cuestión, hay tres motivos habituales: el registro estaba desactivado en el conector correspondiente, el límite de retención ya ha eliminado el archivo o está mirando la ruta equivocada, es decir, el directorio `Hub` en lugar del `FrontEnd`. Compruébelos en este orden.

## Paso 4: Comprobar permisos

Si se rechaza una entrega o, a la inversa, se permite más de lo previsto, el camino pasa por los permisos del conector. Hay una particularidad técnica: `Get-ADPermission` requiere el **DistinguishedName**. Si pasa la identidad habitual en la forma `Server\Connectorname`, la llamada falla en una sesión remota con el mensaje engañoso de que el objeto no se encuentra.

```powershell
$dn = (Get-ReceiveConnector "SRV-MAIL01\Default Frontend SRV-MAIL01").DistinguishedName
Get-ADPermission -Identity $dn -User "NT AUTHORITY\ANONYMOUS LOGON" |
    Where-Object { $_.ExtendedRights -like "*ms-Exch-SMTP*" } |
    Format-Table User, @{n='Rights'; e={$_.ExtendedRights}} -AutoSize
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Identity $dn` | El objeto que se debe comprobar como DistinguishedName; la forma `Server\Connectorname` falla en sesiones remotas |
| `-User` | Limita la salida a los permisos de este principal de seguridad, aquí el acceso anónimo |
| `Where-Object` | Filtra por los Extended Rights relevantes para SMTP |

</details>

```text
User                         Rights
----                         ------
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Submit
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Accept-Any-Sender
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Accept-Authoritative-Domain-Sender
```

La evaluación es más sencilla de lo que parece si distingue cuatro derechos:

| Derecho | Significado |
|---|---|
| `ms-Exch-SMTP-Submit` | Puede realizar entregas |
| `ms-Exch-SMTP-Accept-Any-Sender` | Puede utilizar direcciones de remitente arbitrarias |
| `ms-Exch-SMTP-Accept-Authoritative-Domain-Sender` | Puede presentarse como dominio propio |
| `ms-Exch-SMTP-Accept-Any-Recipient` | **Puede reenviar a dominios externos** |

Los tres primeros forman el conjunto estándar necesario para la entrega anónima y la recepción de correo de Internet. Solo el cuarto derecho convierte un conector de entrada en un relay. En un conector que acepta desde todo el espacio de direcciones, es un relay abierto. En un conector con restricción estricta de IP, en cambio, es la vía habitual e intencionada para que los servidores de aplicaciones puedan enviar externamente.

No confunda `Accept-Any-Sender` con `Accept-Any-Recipient`. El primero es inocuo y necesario; el segundo es la configuración relevante para la seguridad.

## Paso 5: Prueba de control mediante entrega propia

Si el análisis sigue siendo ambiguo, realice usted mismo la entrega. Así controla completamente el remitente, el destinatario y el punto de entrega:

```powershell
Send-MailMessage -SmtpServer 10.0.1.11 -Port 25 `
    -From 'test@example.com' `
    -To 'empfaenger@example.net' `
    -Subject 'Diagnose, bitte ignorieren' `
    -Body 'Testeinlieferung' `
    -Encoding UTF8
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-SmtpServer` | Host de destino de la entrega, aquí deliberadamente como dirección IP para alcanzar un punto final concreto |
| `-Port 25` | Puerto de destino; 25 para entrega no autenticada de servidor a servidor |
| `-From` | Remitente de envolvente y cabecera del mensaje de prueba |
| `-To` | Dirección del destinatario |
| `-Subject` | Línea de asunto |
| `-Body` | Texto del mensaje |
| `-Encoding UTF8` | Codificación de caracteres para asunto y texto; evita problemas con caracteres especiales |

</details>

`Send-MailMessage` está oficialmente en desuso, pero para fines de diagnóstico sigue siendo la herramienta más rápida y está disponible en todos los servidores Windows. Si tiene éxito, no muestra salida, lo cual puede resultar desconcertante.

Si prueba una conexión TLS en el puerto 587 y el extremo remoto presenta un certificado que no coincide con el nombre utilizado, por ejemplo porque se dirige a la dirección IP, la llamada falla con un error de certificado. Para la prueba puede suspender la comprobación durante la sesión:

```powershell
[Net.ServicePointManager]::ServerCertificateValidationCallback = { $true }
```

Esto solo es válido para la sesión actual de PowerShell. Establézcalo conscientemente y nunca en scripts que se ejecuten en producción.

Si llega el mensaje de prueba y quiere saber qué ha ocurrido con él durante el trayecto, el [analizador de cabeceras de correo](https://rafaelpfister.ch/tools/header-analyzer) ayuda: descompone las cabeceras, traza el recorrido por los saltos y muestra los resultados de las comprobaciones de autenticación, completamente local en el navegador, sin que el mensaje abandone su dispositivo.

## Exchange Online: la misma pregunta, otra herramienta

En el tenant rigen otras reglas, y este es el punto donde fallan los procedimientos habituales. Cuente con estas diferencias:

| | Exchange OnPrem | Exchange Online |
|---|---|---|
| Consulta | `Get-MessageTrackingLog` | `Get-MessageTraceV2` |
| Granularidad | Cada evento de transporte | Una línea por mensaje y destinatario |
| Conector visible | Sí, con limitaciones, véase arriba | **No** |
| Dependencia del servidor | Sí, consultar por servidor | No aplica |
| Protocolo SMTP | Disponible | **No disponible** |
| Retención | Su configuración | 90 días mediante el cmdlet, como máximo 10 días por consulta |
| Retraso | Casi inmediato | Algunos minutos |

Las tres consecuencias más importantes en la práctica: **no hay asignación de conector**; se utiliza `FromIP` y `ToIP` como alternativa. **No hay protocolo SMTP**; la conversación SMTP no se puede reconstruir. Y los datos aparecen **con retraso**; un mensaje recién enviado no aparece inmediatamente.

### La consulta básica

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-4) `
    -EndDate (Get-Date) `
    -RecipientAddress "empfaenger@example.com" `
    -ResultSize 1000 |
  Sort-Object Received |
  Format-Table Received, SenderAddress, RecipientAddress, Status, FromIP, Size -AutoSize
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-StartDate` | Límite temporal inferior de la consulta, aquí las últimas cuatro horas |
| `-EndDate` | Límite temporal superior; el cmdlet requiere ambos límites |
| `-RecipientAddress` | Filtra mensajes enviados a esta dirección de destinatario |
| `-ResultSize 1000` | Máximo de líneas de esta página; el límite superior es 5000 |

</details>

```text
Received            SenderAddress          RecipientAddress          Status    FromIP
--------            -------------          ----------------          ------    ------
11.08.2026 08:27:16 emma@partner.example   empfaenger@example.com    Delivered 10.0.20.23
11.08.2026 09:05:24 dienst@example-test.com empfaenger@example.com   Failed    10.0.20.23
```

Los valores más importantes de `Status`: `Delivered`, `Failed`, `Pending`, `Quarantined`, `FilteredAsSpam` y `Expanded` para listas de distribución expandidas. `Pending` significa que aún se están realizando intentos de entrega, no que algo esté averiado.

### Los detalles de un mensaje

El estado por sí solo no indica la causa. Para ello necesita la vista detallada, que requiere el identificador del mensaje de la consulta básica:

```powershell
$n = Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-4) -EndDate (Get-Date) `
        -RecipientAddress "empfaenger@example.com" -ResultSize 100 |
     Where-Object { $_.Status -eq 'Failed' } | Select-Object -First 1

Get-MessageTraceDetailV2 -MessageTraceId $n.MessageTraceId `
    -RecipientAddress $n.RecipientAddress |
  Format-List Date, Event, Action, Detail
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-MessageTraceId` | Identificador único del mensaje de la consulta básica; obligatorio |
| `-RecipientAddress` | Destinatario cuyos pasos de procesamiento se muestran; también obligatorio, pues un mensaje puede tener varios destinatarios |

</details>

Ahí aparecen los pasos de procesamiento en el servicio, como aplicaciones de reglas, decisiones de filtrado y la causa de un rechazo.

### Más allá de diez días

`Get-MessageTraceV2` proporciona datos de los últimos 90 días, pero como máximo de 10 días por consulta; divida periodos más largos en varias consultas. Como alternativa, existe la búsqueda histórica, que se ejecuta de forma asíncrona y proporciona el resultado como CSV, con un intervalo de hasta 90 días:

```powershell
Start-HistoricalSearch -ReportTitle "Analyse Nachtlauf" `
    -StartDate (Get-Date).AddDays(-45) `
    -EndDate (Get-Date).AddDays(-30) `
    -ReportType MessageTrace `
    -SenderAddress "dienst@example-test.com" `
    -NotifyAddress "admin@example.com"

Get-HistoricalSearch | Format-Table JobId, ReportTitle, Status, SubmitDate -AutoSize
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-ReportTitle` | Nombre libremente elegible para el trabajo, con el que posteriormente se podrá encontrar el resultado |
| `-StartDate`, `-EndDate` | Periodo investigado, hasta 90 días atrás |
| `-ReportType MessageTrace` | Tipo de informe; `MessageTrace` proporciona el resumen de mensajes como CSV |
| `-SenderAddress` | Filtra por esta dirección de remitente |
| `-NotifyAddress` | Destinatario de la notificación de finalización; debe ser una dirección de un Accepted Domain del tenant |

</details>

Reserve tiempo: estos trabajos pueden tardar horas según el volumen.

### Fuente de error 4: la ausencia de resultados no prueba la ausencia de tráfico

Esta es la fuente de error más sutil en el tenant. `Get-MessageTraceV2` entrega los resultados por páginas, con un máximo de 5000 líneas por llamada. Con un volumen elevado, una página puede cubrir solo unos pocos minutos aunque haya consultado siete días. Si después filtra localmente, por ejemplo por una IP de origen, estará filtrando un fragmento minúsculo.

Se reconoce por la advertencia que indica que existen más resultados:

```text
WARNING: There are more results, use the following command to get more.
Get-MessageTraceV2 -StartDate "2026-08-11T07:25:19Z" -EndDate "2026-08-11T09:05:46Z"
  -StartingRecipientAddress "naechster@example.com" -ResultSize 5000
```

Si aparece, su análisis es **incompleto**. Si no se devuelve ningún resultado, la conclusión correcta es: no se encontró en el fragmento. No es: no existe.

Hay dos salidas correctas. Puede reducir la ventana temporal hasta que una página la cubra completamente, reconocible por la ausencia de la advertencia. O puede avanzar por todas las páginas utilizando los datos de continuación de la advertencia. Para responder a la pregunta de si algo **nunca** ocurre, una comprobación de configuración es de todos modos superior: si un sistema no tiene ruta hacia un destino, no puede entregar allí, independientemente de cualquier ventana de observación.

El análisis completo de todas las direcciones que realizan entregas es un tema propio, con puntos delicados específicos en la interpretación. Se trata en [¿Quién realiza realmente entregas en su tenant? Agregar direcciones IP de entrega](https://rafaelpfister.ch/blog/einliefernde-ip-adressen-aggregieren).

## Un procedimiento que ha demostrado ser eficaz

En resumen, este orden ha demostrado ser el más rápido. Buscar el mensaje en todos los servidores y determinar el último evento. En caso de fallo, cambiar inmediatamente a `Format-List` y leer la respuesta SMTP completa, en vez de deducir a partir del tipo de evento. Después aclarar el alcance, es decir, agrupar y contar. Solo cuando el caso esté claramente delimitado, reconstruir la vía de entrega mediante la configuración del conector y el protocolo SMTP. Y por último, si es necesario, verificar mediante una entrega propia.

Los mayores consumidores de tiempo son siempre los mismos: se lee una tabla truncada en lugar del mensaje de error completo, se confunden copias sombra con pasos de procesamiento, se cree el `ConnectorId` del seguimiento y se considera una muestra vacía como prueba. Quien conozca estos cuatro puntos suele llegar al nivel correcto en pocos minutos.

## Historial de cambios

- **25 de octubre de 2026:** Corregido: Get-MessageTraceV2 proporciona datos de los últimos 90 días, pero como máximo de 10 días por consulta. Se actualizaron los enlaces de fuentes a las nuevas direcciones de Microsoft Learn.

## Fuentes

1.  [Message tracking in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): descripción de campos y lista completa de tipos de eventos en Message Tracking.

2.  [Protocol logging in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/protocol-logging): ubicaciones de almacenamiento, formato y retención de los protocolos SMTP, incluido Front End Transport.

3.  [Shadow redundancy in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/shadow-redundancy): explica los eventos relacionados con copias sombra y su descarte.

4.  [Mail routing in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing): interacción entre Front End Transport y Transport Service, base del comportamiento de proxy.

5.  [Receive connectors in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/receive-connectors): enlaces, grupos de permisos y mecanismos de autenticación.

6.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2): sucesor de Get-MessageTrace, incluida la lógica de páginas, la lista de campos y el intervalo de consulta (90 días, máximo 10 días por consulta).

7.  [Start-HistoricalSearch](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/start-historicalsearch): seguimiento asíncrono de mensajes durante hasta 90 días.
