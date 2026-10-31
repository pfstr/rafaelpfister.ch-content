---
title: "Causas habituales de los bucles de correo y cómo solucionarlos"
navTitle: "Solucionar bucles de correo"
description: "Cómo identificar y solucionar sistemáticamente los bucles de correo SMTP en Exchange Online, entornos híbridos y gateways de correo previos mediante NDR, encabezados, Message Trace, objetos de destinatario y conectores."
date: "2026-08-07"
kategorie: "Exchange local / híbrido"
timeToRead: "12 min de lectura"
themen:
  - exchange-onprem-hybrid
  - microsoft-365-exchange
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange"
protokolle:
  - "smtp"
  - "powershell"
  - "troubleshooting"
related:
  - exchange-mailboxen-ohne-remote-move-cloud-neu-erstellen
  - totemomail-m365
  - ghost-sender-exchange-online-nebeneingang
slug: "causas-habituales-de-los-bucles-de-correo-y-como-resolverlos"
translationId: "article-4c91e7b2a8605fd3"
draft: false
translationOf: typische-ursachen-fuer-mail-loops-und-deren-behebung
translationSourceHash: 6bd8a9a875aa7795711db421084241c47d7b2c1f24abb9ba64bc1b18128d86b7
translationModel: gpt-5.6-terra
translatedAt: 2026-10-30T03:37:49.596Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/causas-habituales-de-los-bucles-de-correo-y-como-resolverlos
---

# Causas habituales de los bucles de correo y cómo solucionarlos

Un bucle de correo se produce cuando al menos dos sistemas de transporte se entregan repetidamente el mismo mensaje. Ninguno de los sistemas se reconoce como destino final, pero ambos conocen un supuesto siguiente salto adecuado. El bucle solo termina cuando un servidor considera superado el número permitido de estaciones de transporte y genera un NDR.

En Exchange, dos mensajes son especialmente reveladores:

- `554 5.4.6 Hop count exceeded - possible mail loop` suele generarlo el Exchange local.
- `554 5.4.14 Hop count exceeded - possible mail loop ATTR34` lo genera Exchange Online.

El límite de saltos no es la causa, sino la protección frente a una repetición infinita. Por tanto, aumentarlo no soluciona nada. Hay que localizar el punto en el que el mensaje, contrariamente a la arquitectura de destino, se devuelve a un sistema por el que ya ha pasado.

## Reconocer el patrón del bucle en los encabezados

El NDR y los encabezados completos del mensaje original deben guardarse antes de realizar cambios. Las líneas `Received` se leen de abajo arriba: la línea inferior es el salto documentado más temprano y la superior el más reciente.

Un bucle suele mostrarse como una secuencia recurrente:

```text
Internet
  → Exchange Online Protection
  → Mailgateway
  → Exchange Online Protection
  → Mailgateway
  → Exchange Online Protection
  → ...
```

No todos los nombres de host de Microsoft que aparecen varias veces indican ya un bucle. Exchange Online procesa internamente los mensajes a través de varios roles de transporte. Lo llamativo es el retorno repetido entre los mismos límites administrativos, por ejemplo, entre Exchange Online y un gateway local. Las marcas de tiempo, la IP emisora, el host receptor y `Message-ID` ayudan a identificar inequívocamente el ciclo.

Para el primer análisis se responden estas preguntas:

1. ¿Qué sistema generó el NDR?
2. ¿Qué dos o tres saltos se repiten?
3. ¿Qué sistema debería haber entregado definitivamente el mensaje?
4. ¿A causa de qué decisión de dominio, destinatario, conector o regla se reenvió?
5. ¿Qué cambio ha afectado recientemente al flujo de correo?

## Diagnóstico en Exchange Online

Con `Get-MessageTraceV2` se puede examinar el procesamiento de los últimos 90 días; se permite un máximo de diez días por consulta. Una ventana de tiempo reducida y la dirección concreta del destinatario proporcionan los resultados más útiles:

```powershell
$start = (Get-Date).AddHours(-2)
$end = Get-Date
$recipient = "user01@contoso.com"

$trace = Get-MessageTraceV2 `
    -RecipientAddress $recipient `
    -StartDate $start `
    -EndDate $end `
    -ResultSize 5000

$trace |
    Select-Object Received,SenderAddress,RecipientAddress,Subject,
        Status,FromIP,ToIP,MessageTraceId,MessageId |
    Sort-Object Received
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-RecipientAddress` | Filtra el seguimiento por la dirección de destinatario especificada |
| `-StartDate` / `-EndDate` | Ventana temporal de la consulta; se permite un máximo de diez días por consulta |
| `-ResultSize 5000` | Número máximo de entradas devueltas |
| `Select-Object …` | Reduce la salida a los campos relevantes para el análisis del bucle |
| `Sort-Object Received` | Ordena los resultados cronológicamente según la hora de aceptación |

</details>

Los detalles de un resultado muestran eventos de transporte individuales:

```powershell
$trace | ForEach-Object {
    Get-MessageTraceDetailV2 `
        -MessageTraceId $_.MessageTraceId `
        -RecipientAddress $_.RecipientAddress
} | Format-Table Date,Event,Action,Detail -AutoSize
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-MessageTraceId` | ID de seguimiento único procedente del resultado de `Get-MessageTraceV2` |
| `-RecipientAddress` | Dirección del destinatario del resultado; requerida junto con el ID de seguimiento para la consulta detallada |
| `Format-Table … -AutoSize` | Ajusta los anchos de columna al contenido para que los detalles de los eventos sigan siendo legibles |

</details>

A continuación se recopilan conjuntamente el dominio, el destinatario y los conectores:

```powershell
Get-AcceptedDomain |
    Format-Table Name,DomainName,DomainType,MatchSubDomains -AutoSize

Get-EXORecipient -Identity $recipient |
    Format-List DisplayName,RecipientTypeDetails,PrimarySmtpAddress,
        ExternalEmailAddress,EmailAddresses

Get-OutboundConnector -IncludeTestModeConnectors |
    Format-List Name,Enabled,ConnectorType,RecipientDomains,SmartHosts,
        UseMXRecord,RouteAllMessagesViaOnPremises,TlsSettings

Get-InboundConnector |
    Format-List Name,Enabled,ConnectorType,SenderDomains,SenderIPAddresses,
        TlsSenderCertificateName,RequireTls,RestrictDomainsToIPAddresses,
        RestrictDomainsToCertificate
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Identity $recipient` | Selecciona el objeto de destinatario mediante dirección, alias o nombre |
| `-IncludeTestModeConnectors` | Incluye también los conectores en modo de prueba en la salida |
| `Format-Table … -AutoSize` | Vista de tabla con anchos de columna según el contenido |
| `Format-List …` | Vista de lista de las propiedades indicadas, adecuada para valores largos como listas de direcciones |

</details>

Lo decisivo no es que un objeto individual parezca plausible. El tipo de dominio, el tipo real de destinatario y el conector aplicable deben describir juntos el mismo destino.

## Diagnóstico en el Exchange local

En un entorno híbrido, el mismo destinatario también se comprueba localmente. Las consultas distinguen entre un buzón local real, una RemoteMailbox y un MailUser:

```powershell
Get-Recipient -Identity $recipient |
    Format-List DisplayName,RecipientType,RecipientTypeDetails,
        PrimarySmtpAddress,EmailAddresses

Get-Mailbox -Identity $recipient -ErrorAction SilentlyContinue |
    Format-List RecipientTypeDetails,ServerName,Database,PrimarySmtpAddress

Get-RemoteMailbox -Identity $recipient -ErrorAction SilentlyContinue |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,RemoteRoutingAddress

Get-MailUser -Identity $recipient -ErrorAction SilentlyContinue |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,ExternalEmailAddress
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Identity $recipient` | Selecciona el objeto mediante dirección, alias o nombre |
| `-ErrorAction SilentlyContinue` | Suprime el mensaje de error si el objeto no existe en el tipo respectivo; la consulta simplemente no devuelve ningún resultado |

</details>

Para la ruta de transporte se necesitan los conectores de envío y recepción, así como los registros de seguimiento:

```powershell
Get-SendConnector |
    Format-List Name,Enabled,AddressSpaces,DNSRoutingEnabled,SmartHosts,
        SourceTransportServers,CloudServicesMailEnabled,TlsDomain

Get-ReceiveConnector |
    Format-List Identity,Enabled,Bindings,RemoteIPRanges,PermissionGroups

$servers = Get-ExchangeServer |
    Where-Object { $_.IsMailboxServer -or $_.IsHubTransportServer }

$servers |
    Get-MessageTrackingLog `
        -Start $start `
        -End $end `
        -Recipients $recipient `
        -ResultSize Unlimited |
    Select-Object Timestamp,ServerHostname,ClientHostname,Source,EventId,
        ConnectorId,Sender,Recipients,MessageId,NetworkMessageId |
    Sort-Object Timestamp
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Where-Object { … }` | Limita la lista de servidores a servidores de buzón y de transporte de concentrador, es decir, los roles con registros de seguimiento |
| `-Start` / `-End` | Ventana temporal para la búsqueda en los registros |
| `-Recipients $recipient` | Filtra los eventos de seguimiento con esta dirección de destinatario |
| `-ResultSize Unlimited` | Elimina el límite predeterminado de 1000 entradas devueltas |
| `Select-Object …` | Reduce la salida a los campos relevantes para el análisis de la ruta |
| `Sort-Object Timestamp` | Ordena cronológicamente los eventos de todos los servidores |

</details>

Un `SEND` a Exchange Online, seguido de un nuevo `RECEIVE` del mismo mensaje desde Exchange Online, hace visible la devolución. Con `MessageId` y `NetworkMessageId` se evita confundir distintos mensajes de prueba entre sí.

## Las causas más frecuentes de un vistazo

| Patrón | Causa habitual | Solución |
| --- | --- | --- |
| Los destinatarios desconocidos oscilan entre dos sistemas | El dominio aceptado está configurado como `InternalRelay`, pero ambos lados reenvían destinatarios desconocidos | Definir una responsabilidad inequívoca; si la entrega se realiza íntegramente en EXO, usar `Authoritative`, o para un dominio dividido definir un único salto final |
| EXO envía al Exchange local, que lo devuelve inmediatamente a EXO | El conector híbrido o Centralized Mail Transport ya no coincide con la ubicación del buzón | Comprobar la configuración de HCW y `RouteAllMessagesViaOnPremises`; desactivar la ruta centralizada obsoleta o corregir la resolución local de destinatarios |
| El mensaje oscila entre EXO y un gateway de seguridad, firma o cifrado | Los mensajes devueltos vuelven a cumplir la regla de salida | Usar como excepción el encabezado establecido por el gateway o el mecanismo documentado de prevención de bucles; autenticar inequívocamente los conectores de entrada y salida |
| Solo se ve afectado un destinatario | `targetAddress` obsoleto o incorrecto, tipo RemoteMailbox incorrecto o direcciones proxy contradictorias | Determinar la fuente de autoridad, corregir allí el objeto de destinatario y sincronizarlo |
| Solo se repiten los mensajes reenviados | Una regla de transporte, un reenvío de buzón o una regla de bandeja de entrada vuelve a dirigirse a la ruta original | Desactivar la regla, corregir el destino y definir una excepción fiable |
| Solo se ve afectado un subdominio o una aplicación | El dominio superior no cubre correctamente el subdominio en la ruta de conector esperada | Configurar explícitamente el subdominio como dominio aceptado y en el conector de envío adecuado |
| Todos los mensajes se repiten tras un cambio de gateway o DNS | El host inteligente o MX apunta a la entrada del sistema emisor | Corregir el siguiente salto y comprobar por separado los destinos de DNS, NAT y equilibrador de carga |

## Causa 1: Tipo incorrecto del dominio aceptado

Un dominio authoritative significa que todos los destinatarios válidos de ese dominio son conocidos en la organización de Exchange; los destinatarios desconocidos se rechazan. Un dominio de retransmisión interna significa que parte de los destinatarios se encuentra en otro sistema y debe reenviarse mediante un conector de envío o salida.

La configuración problemática surge cuando Exchange Online envía destinatarios desconocidos a un sistema local y este tampoco trata de forma definitiva el mismo dominio, sino que lo devuelve a Exchange Online mediante MX o un host inteligente.

```powershell
Get-AcceptedDomain -Identity contoso.com |
    Format-List DomainName,DomainType,MatchSubDomains
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Identity contoso.com` | Selecciona el dominio aceptado que se debe comprobar |
| `Format-List …` | Muestra como lista el nombre de dominio, el tipo de dominio y la cobertura de subdominios |

</details>

Cuando todos los destinatarios están en Exchange Online tras finalizar una migración, `Authoritative` suele ser el estado de destino correcto:

```powershell
# Ejecutar solo después de comprobar completamente los destinatarios y el enrutamiento.
Set-AcceptedDomain -Identity contoso.com -DomainType Authoritative
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Identity contoso.com` | El dominio aceptado que se va a modificar |
| `-DomainType Authoritative` | Configura el dominio como authoritative: los destinatarios desconocidos se rechazan en lugar de reenviarse |

</details>

En un dominio dividido real, `InternalRelay` puede ser correcto. Sin embargo, entonces se necesita un conector claro hacia el sistema que conoce a los destinatarios restantes. Ese destino no debe volver a enviar las direcciones desconocidas al punto de origen.

## Causa 2: Conectores híbridos superpuestos y Centralized Mail Transport

Centralized Mail Transport dirige deliberadamente los mensajes salientes de Exchange Online a través del Exchange local. Esto es útil para determinados requisitos de cumplimiento, pero genera rutas de transporte adicionales. Si la opción permanece activa después de una migración, aunque el sistema local envíe mensajes de vuelta a Exchange Online a través de su propio MX, puede formarse un círculo.

```powershell
Get-OutboundConnector -IncludeTestModeConnectors |
    Format-Table Name,Enabled,ConnectorType,RouteAllMessagesViaOnPremises,
        RecipientDomains,SmartHosts,UseMXRecord -AutoSize
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-IncludeTestModeConnectors` | Incluye también los conectores en modo de prueba en la salida |
| `Format-Table … -AutoSize` | Vista de tabla de las propiedades de enrutamiento con anchos de columna según el contenido |

</details>

También deben comprobarse varios conectores con ámbito superpuesto. Microsoft recomienda un conector local dedicado para el flujo de correo híbrido; una reparación mediante Hybrid Configuration Wizard suele ser más segura que cambios individuales aislados.

Si se demuestra que Centralized Mail Transport ya no es necesario, la configuración puede desactivarse específicamente:

```powershell
# Solo después de comprobar los requisitos de cumplimiento y del gateway.
Set-OutboundConnector `
    -Identity "Outbound to On-Premises" `
    -RouteAllMessagesViaOnPremises:$false
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Identity "Outbound to On-Premises"` | El conector de salida que se va a modificar |
| `-RouteAllMessagesViaOnPremises:$false` | Desactiva Centralized Mail Transport: los mensajes salientes de Exchange Online dejan de pasar por el Exchange local |

</details>

## Causa 3: Un gateway vuelve a procesar sus mensajes devueltos

En un escenario de entrada y salida, Exchange Online envía un mensaje a un servicio adicional para firma, cifrado o archivado. Este lo devuelve posteriormente a Exchange Online. La regla de salida debe reconocer el mensaje devuelto; de lo contrario, se envía de nuevo al servicio.

La comprobación comienza con todas las reglas que seleccionan conectores, redirigen destinatarios o evalúan encabezados:

```powershell
Get-TransportRule |
    Sort-Object Priority |
    Format-List Name,State,Mode,Priority,FromScope,SentToScope,
        RedirectMessageTo,RouteMessageOutboundConnector,
        SetHeaderName,SetHeaderValue,ExceptIfHeaderContainsMessageHeader,
        ExceptIfHeaderContainsWords
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Sort-Object Priority` | Ordena las reglas según su orden de evaluación |
| `Format-List …` | Muestra las propiedades que seleccionan conectores, redirigen destinatarios o establecen encabezados, o bien los evalúan como excepción |

</details>

La excepción concreta debe seguir la documentación del fabricante del gateway. Es habitual utilizar un encabezado establecido por el servicio que no pueda falsificarse de forma fiable desde Internet. Además, los conectores de entrada deben identificar el servicio mediante certificado o IP fija de remitente. Una excepción general para todos los mensajes que parecen «internos» es demasiado amplia.

## Causa 4: El objeto de destinatario y el buzón real no están en el mismo lugar

Un objeto puede aparecer en Exchange Online como `MailUser`, aunque el buzón activo esté localmente. En un entorno híbrido sincronizado, esto no constituye automáticamente un duplicado. Tampoco una `ExternalEmailAddress`, que corresponda a la dirección SMTP principal, demuestra por sí sola una configuración incorrecta.

Lo determinante es la combinación de todas las consultas:

- `Get-Mailbox` local devuelve un resultado: el buzón activo está localmente.
- `Get-RemoteMailbox` local devuelve un resultado: el destino administrado está en Exchange Online.
- `Get-EXOMailbox` devuelve un resultado: existe un buzón real en la nube.
- `Get-EXORecipient` solo devuelve un MailUser: el objeto es un destino de enrutamiento, no un buzón en la nube.

Son problemáticos los objetos obsoletos tras una migración, los dominios de enrutamiento remoto incorrectos o los valores `targetAddress` establecidos manualmente cuya dirección de dominio vuelve por la misma ruta de transporte. Los cambios se realizan en la fuente de autoridad: en entornos sincronizados, por tanto, con herramientas de administración de Exchange localmente y no editando directamente atributos individuales en Exchange Online.

## Causa 5: Los reenvíos y las reglas de transporte forman un círculo

Una regla puede redirigir de la dirección A a B, mientras que B vuelve a enviar a A mediante una segunda regla, un reenvío de buzón o un sistema externo. Estos bucles suelen afectar solo a destinatarios o tipos de mensaje concretos.

```powershell
Get-TransportRule |
    Sort-Object Priority |
    Select-Object Name,State,Mode,Priority,RedirectMessageTo,
        BlindCopyTo,AddToRecipients,RouteMessageOutboundConnector

Get-Mailbox -ResultSize Unlimited |
    Select-Object DisplayName,PrimarySmtpAddress,
        ForwardingAddress,ForwardingSmtpAddress,DeliverToMailboxAndForward

Get-InboxRule -Mailbox user01@contoso.com |
    Select-Object Name,Enabled,Priority,ForwardTo,RedirectTo,ForwardAsAttachmentTo
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Sort-Object Priority` | Ordena las reglas de transporte según su orden de evaluación |
| `-ResultSize Unlimited` | Elimina el límite predeterminado de 1000 buzones devueltos |
| `-Mailbox user01@contoso.com` | Buzón cuyas reglas de bandeja de entrada se consultan |
| `Select-Object …` | Reduce la salida a los destinos de reenvío y redirección |

</details>

La solución no consiste únicamente en desactivar temporalmente una regla. Debe resolverse toda la cadena, y las reglas para servicios externos necesitan una excepción que reconozca de forma segura los mensajes ya procesados.

## Causa 6: MX, host inteligente o subdominio apunta de vuelta

Un gateway puede necesitar internamente un siguiente salto diferente al de los remitentes externos. Si para el reenvío utiliza simplemente el MX público, este puede apuntar de nuevo al propio gateway. El mismo problema se produce si un host inteligente vuelve a su propio listener mediante NAT o equilibrio de carga.

```powershell
Resolve-DnsName -Type MX contoso.com
Resolve-DnsName -Type MX app.contoso.com

Get-SendConnector |
    Format-List Name,AddressSpaces,DNSRoutingEnabled,SmartHosts
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Type MX` | Consulta los registros MX en lugar de los registros A predeterminados |
| `contoso.com` / `app.contoso.com` | Dominio que se consulta como argumento posicional (parámetro `-Name`) |
| `Format-List …` | Muestra por cada conector de envío los espacios de direcciones, el modo de enrutamiento y los hosts inteligentes |

</details>

Los subdominios requieren una comprobación propia. Microsoft documentó en un artículo de soporte posteriormente retirado casos en los que un subdominio de aplicación debe crearse explícitamente como dominio de retransmisión interna y sincronizarse con los sistemas Edge:

```powershell
New-AcceptedDomain `
    -Name "app.contoso.com" `
    -DomainName app.contoso.com `
    -DomainType InternalRelay

Start-EdgeSynchronization
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Name "app.contoso.com"` | Nombre para mostrar del nuevo objeto de dominio aceptado |
| `-DomainName app.contoso.com` | El dominio SMTP para el que Exchange acepta mensajes |
| `-DomainType InternalRelay` | Parte de los destinatarios se encuentra fuera de la organización; los destinatarios desconocidos se reenvían mediante un conector de envío en lugar de rechazarse |

</details>

Estos comandos no son una solución universal. Solo son adecuados si `app.contoso.com` se entrega realmente fuera de la organización de Exchange y el conector de envío dispone de un siguiente salto inequívoco.

## Procedimiento seguro ante un bucle activo

Durante la incidencia, primero debe detenerse la multiplicación. Según la arquitectura, se desactiva de forma controlada la regla de transporte desencadenante o el conector específico, o el gateway retiene la cola afectada. Antes se exportan la configuración y los ejemplos de mensajes.

A continuación, se realiza una prueba con exactamente un remitente, un destinatario y una línea de asunto claramente identificable. El mensaje se sigue sin interrupciones mediante encabezados, Message Trace y registros de seguimiento locales. Solo cuando llegue al destino previsto se vuelve a abrir el flujo de correo de forma gradual.

No se recomienda:

- aumentar los límites de salto
- modificar varios conectores al mismo tiempo
- alternar dominios aceptados por suposición entre `Authoritative` y `InternalRelay`
- volver a inyectar repetidamente una cola problemática sin comprobarla
- corregir directamente en AD o Exchange Online atributos de Exchange sincronizados
- desactivar las comprobaciones de TLS, IP o certificados como supuesta solución rápida

## Comprobación final

Tras la corrección, la documentación debe contener exactamente una afirmación para cada dominio relevante: ¿qué sistema conoce al destinatario, qué conector es aplicable y qué host es el siguiente salto final?

La validación técnica incluye, como mínimo:

- mensaje de prueba externo e interno
- destinatario desconocido del mismo dominio
- destinatario en cada lado de un dominio dividido real
- mensaje saliente con gateway o Centralized Mail Transport activado
- encabezados sin secuencia de saltos recurrente
- Message Trace con `Delivered` o la transferencia esperada
- seguimiento local sin un nuevo `RECEIVE` después de un `SEND` al mismo destino
- validación de conectores para todos los conectores que siguen siendo necesarios

Un bucle de correo solucionado solo se considera cerrado cuando no solo llega el correo de prueba, sino que también los destinatarios desconocidos y las rutas alternativas de flujo de correo terminan de forma definida. Ahí es donde se producen la mayoría de las recaídas.

## Historial de cambios

- **25 de octubre de 2026:** Microsoft retiró el artículo de soporte sobre el escenario de subdominio y EdgeSync; la fuente remite ahora a la versión archivada del 19 de abril de 2026.

## Fuentes

1. [Microsoft Learn – Fix NDR error 5.4.6 or 5.4.14 in Exchange Online](https://learn.microsoft.com/en-us/troubleshoot/exchange/email-delivery/ndr/fix-error-code-5-4-6-through-5-4-20-in-exchange-online): significado de los NDR de Exchange y causas habituales en dominios aceptados y conectores híbridos.

2. [Microsoft Learn – Manage accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains): diferencias entre `Authoritative` y `InternalRelay`.

3. [Microsoft Learn – Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains): responsabilidad, dominios de retransmisión y búsqueda de destinatarios en Exchange local.

4. [Microsoft Learn – Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing): rutas de transporte esperadas con y sin Centralized Mail Transport.

5. [Microsoft Learn – Validate connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/validate-connectors): validación de conectores e indicaciones sobre varios conectores coincidentes al mismo tiempo.

6. [Microsoft Learn – Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud): patrones de flujo de correo compatibles con servicios de terceros previos.

7. [Microsoft Learn – Mail flow rules in Exchange Online](https://learn.microsoft.com/en-us/exchange/security-and-compliance/mail-flow-rules/mail-flow-rules): procesamiento, prioridad, acciones y excepciones de las reglas de transporte.

8. [Microsoft Learn – Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2?view=exchange-ps): búsqueda de mensajes en el transporte de Exchange Online.

9. [Microsoft Learn – Search message tracking logs](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/search-message-tracking-logs): seguimiento local de mensajes en todos los servidores Exchange.

10. [Microsoft Learn – Hop count exceeded for an on-premises application subdomain (versión archivada del 19 de abril de 2026)](https://web.archive.org/web/20260419203247/https://learn.microsoft.com/en-us/troubleshoot/exchange/mailflow/hop-count-exceeded-possible-mail-loop): escenario documentado de subdominio/EdgeSync con dominio de retransmisión interna explícito; Microsoft ha retirado entretanto el artículo.
