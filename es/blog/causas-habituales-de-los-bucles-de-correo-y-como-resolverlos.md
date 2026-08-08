---
title: "Causas habituales de los bucles de correo y cómo resolverlos"
navTitle: "Resolver bucles de correo"
description: "Cómo identificar y resolver sistemáticamente bucles de correo SMTP en Exchange Online, entornos híbridos y gateways de correo previos mediante NDR, encabezados, Message Trace, objetos de destinatario y conectores."
date: "2026-08-07"
kategorie: "Exchange OnPrem / híbrido"
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
url: https://rafaelpfister.ch/es/blog/causas-habituales-de-los-bucles-de-correo-y-como-resolverlos
translationSourceHash: 5353684681217adafc789a3b28ec218fa927e18d801c82c437ae281e1e1017bd
translationModel: gpt-5.6-terra
translatedAt: 2026-08-08T14:00:36.352Z
translationReview: automatic
---

# Causas habituales de los bucles de correo y cómo resolverlos

Un bucle de correo se produce cuando al menos dos sistemas de transporte se entregan repetidamente el mismo mensaje. Ninguno de los sistemas se reconoce como destino final, pero ambos conocen un siguiente salto aparentemente adecuado. El bucle solo termina cuando un servidor detecta que se ha superado el número permitido de estaciones de transporte y genera un NDR.

En Exchange, hay dos mensajes especialmente reveladores:

- `554 5.4.6 Hop count exceeded - possible mail loop` suele ser generado por el Exchange local.
- `554 5.4.14 Hop count exceeded - possible mail loop ATTR34` es generado por Exchange Online.

El límite de saltos no es la causa, sino la protección frente a una repetición infinita. Por tanto, aumentarlo no soluciona nada. Hay que encontrar el punto en el que el mensaje, contrariamente a la arquitectura de destino, se devuelve a un sistema por el que ya ha pasado.

## Reconocer el patrón de bucle en el encabezado

El NDR y los encabezados completos del mensaje original deben guardarse antes de realizar cualquier cambio. Las líneas `Received` se leen de abajo arriba: la línea inferior corresponde al primer salto documentado y la superior, al más reciente.

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

No todos los nombres de host de Microsoft que aparecen varias veces constituyen ya un bucle. Exchange Online procesa internamente los mensajes a través de varios roles de transporte. Lo sospechoso es el retorno repetido entre los mismos límites administrativos, por ejemplo, entre Exchange Online y un gateway local. Las marcas de tiempo, la IP remitente, el host receptor y `Message-ID` ayudan a identificar claramente el ciclo.

Para el análisis inicial, deben responderse estas preguntas:

1. ¿Qué sistema generó el NDR?
2. ¿Qué dos o tres saltos se repiten?
3. ¿Qué sistema debería haber entregado finalmente el mensaje?
4. ¿Debido a qué decisión de dominio, destinatario, conector o regla se reenvió?
5. ¿Qué cambio afectó por última vez al flujo de correo?

## Diagnóstico en Exchange Online

Con `Get-MessageTraceV2` se puede examinar el procesamiento de los últimos 90 días; se permite un máximo de diez días por consulta. Una ventana temporal reducida y la dirección concreta del destinatario proporcionan los resultados más útiles:

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

Los detalles de un resultado muestran eventos de transporte individuales:

```powershell
$trace | ForEach-Object {
    Get-MessageTraceDetailV2 `
        -MessageTraceId $_.MessageTraceId `
        -RecipientAddress $_.RecipientAddress
} | Format-Table Date,Event,Action,Detail -AutoSize
```

A continuación, se revisan conjuntamente el dominio, el destinatario y los conectores:

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

Lo decisivo no es si un objeto individual parece plausible. El tipo de dominio, el tipo real de destinatario y el conector aplicable deben describir conjuntamente el mismo destino.

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

Un `SEND` hacia Exchange Online, seguido de un nuevo `RECEIVE` del mismo mensaje desde Exchange Online, hace visible la devolución. Con `MessageId` y `NetworkMessageId` se evita confundir distintos mensajes de prueba.

## Las causas más frecuentes de un vistazo

| Patrón | Causa habitual | Solución |
| --- | --- | --- |
| Los destinatarios desconocidos oscilan entre dos sistemas | El dominio aceptado está configurado como `InternalRelay`, pero ambas partes reenvían destinatarios desconocidos | Definir una responsabilidad clara; para la entrega completa en EXO, usar `Authoritative`, o definir un único salto final para un dominio dividido |
| EXO envía al Exchange local y este lo devuelve inmediatamente a EXO | El conector híbrido o Centralized Mail Transport ya no se ajusta a la ubicación del buzón | Comprobar la configuración de HCW y `RouteAllMessagesViaOnPremises`; desactivar la ruta centralizada obsoleta o corregir la resolución local de destinatarios |
| El mensaje oscila entre EXO y un gateway de seguridad, firma o cifrado | Los mensajes devueltos vuelven a cumplir la regla de salida | Usar como excepción el encabezado establecido por el gateway o el mecanismo documentado de prevención de bucles; autenticar de forma inequívoca los conectores de entrada y salida |
| Solo un destinatario se ve afectado | `targetAddress` obsoleto o incorrecto, tipo de RemoteMailbox erróneo o direcciones proxy contradictorias | Determinar la fuente de autoridad, corregir el objeto de destinatario allí y sincronizarlo |
| Solo se producen bucles en mensajes reenviados | Una regla de transporte, el reenvío de buzón o una regla de bandeja de entrada vuelve a dirigir a la ruta original | Desactivar la regla, corregir el destino y definir una excepción sólida |
| Solo se ve afectado un subdominio o una aplicación | El dominio principal no cubre correctamente el subdominio en la ruta de conector prevista | Configurar explícitamente el subdominio como dominio aceptado y en el conector de envío adecuado |
| Todos los mensajes entran en bucle después de un cambio de gateway o DNS | El host inteligente o MX apunta a la entrada del sistema emisor | Corregir el siguiente salto y comprobar por separado los destinos DNS, NAT y del balanceador de carga |

## Causa 1: Tipo incorrecto del dominio aceptado

Un dominio authoritative significa que todos los destinatarios válidos de ese dominio se conocen en la organización de Exchange; los destinatarios desconocidos se rechazan. Un dominio de retransmisión interna significa que parte de los destinatarios se encuentra en otro sistema y debe reenviarse mediante un conector de envío o de salida.

La configuración problemática se produce cuando Exchange Online envía destinatarios desconocidos a un sistema local y este tampoco trata ese mismo dominio de forma definitiva, sino que lo devuelve a Exchange Online mediante MX o un host inteligente.

```powershell
Get-AcceptedDomain -Identity contoso.com |
    Format-List DomainName,DomainType,MatchSubDomains
```

Si todos los destinatarios están en Exchange Online tras completar una migración, `Authoritative` suele ser el estado objetivo correcto:

```powershell
# Ejecutar solo después de comprobar completamente los destinatarios y el enrutamiento.
Set-AcceptedDomain -Identity contoso.com -DomainType Authoritative
```

En un dominio dividido real, `InternalRelay` puede ser correcto. Sin embargo, se necesita un conector claro hacia el sistema que conoce a los destinatarios restantes. Este destino no debe reenviar direcciones desconocidas de vuelta al punto de origen.

## Causa 2: Conectores híbridos superpuestos y Centralized Mail Transport

Centralized Mail Transport enruta deliberadamente los mensajes salientes de Exchange Online a través del Exchange local. Esto es útil para determinados requisitos de cumplimiento, pero genera rutas de transporte adicionales. Si la opción permanece activa después de una migración, aunque el sistema local reenvíe los mensajes a Exchange Online mediante su propio MX, puede formarse un ciclo.

```powershell
Get-OutboundConnector -IncludeTestModeConnectors |
    Format-Table Name,Enabled,ConnectorType,RouteAllMessagesViaOnPremises,
        RecipientDomains,SmartHosts,UseMXRecord -AutoSize
```

Varios conectores con ámbitos superpuestos también son sospechosos. Microsoft recomienda un conector local dedicado para el flujo de correo híbrido; una reparación mediante Hybrid Configuration Wizard suele ser más segura que cambios individuales aislados.

Si se ha comprobado que Centralized Mail Transport ya no es necesario, la configuración puede desactivarse específicamente:

```powershell
# Solo después de comprobar los requisitos de cumplimiento y del gateway.
Set-OutboundConnector `
    -Identity "Outbound to On-Premises" `
    -RouteAllMessagesViaOnPremises:$false
```

## Causa 3: Un gateway vuelve a procesar sus mensajes devueltos

En un escenario de entrada y salida, Exchange Online envía un mensaje a un servicio adicional para firma, cifrado o archivado. Este lo devuelve posteriormente a Exchange Online. La regla de salida debe reconocer el mensaje devuelto; de lo contrario, se enviará de nuevo al servicio.

La comprobación comienza con todas las reglas que seleccionan conectores, redirigen destinatarios o evalúan encabezados:

```powershell
Get-TransportRule |
    Sort-Object Priority |
    Format-List Name,State,Mode,Priority,FromScope,SentToScope,
        RedirectMessageTo,RouteMessageOutboundConnector,
        SetHeaderName,SetHeaderValue,ExceptIfHeaderContainsMessageHeader,
        ExceptIfHeaderContainsWords
```

La excepción concreta debe seguir la documentación del fabricante del gateway. Es habitual utilizar un encabezado establecido por el servicio que no pueda falsificarse de forma fiable desde Internet. Además, los conectores de entrada deben identificar el servicio mediante certificado o IP de remitente fija. Una excepción general para todos los mensajes que parecen «internos» es demasiado amplia.

## Causa 4: El objeto de destinatario y el buzón real no están en el mismo lugar

Un objeto puede aparecer en Exchange Online como `MailUser`, aunque el buzón activo se encuentre localmente. En un entorno híbrido sincronizado, esto no es automáticamente un duplicado. Asimismo, una `ExternalEmailAddress`, que corresponde a la dirección SMTP principal, no demuestra por sí sola una configuración incorrecta.

Lo determinante es la combinación de todas las consultas:

- `Get-Mailbox` local devuelve un resultado: el buzón activo está local.
- `Get-RemoteMailbox` local devuelve un resultado: el destino administrado está en Exchange Online.
- `Get-EXOMailbox` devuelve un resultado: existe un buzón real en la nube.
- `Get-EXORecipient` devuelve solo un MailUser: el objeto es un destino de enrutamiento, no un buzón en la nube.

Son problemáticos los objetos obsoletos tras una migración, los dominios de enrutamiento remoto incorrectos o los valores `targetAddress` establecidos manualmente cuyo dominio vuelve a conducir por la misma ruta de transporte. Los cambios deben realizarse en la fuente de autoridad: en entornos sincronizados, por tanto, con herramientas de administración de Exchange locales y no editando directamente atributos individuales en Exchange Online.

## Causa 5: Los reenvíos y las reglas de transporte forman un ciclo

Una regla puede redirigir de la dirección A a B, mientras que B vuelve a enviar a A mediante una segunda regla, un reenvío de buzón o un sistema externo. Estos bucles suelen afectar solo a determinados destinatarios o tipos de mensaje.

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

La solución no consiste solo en desactivar temporalmente una regla. Debe resolverse toda la cadena, y las reglas para servicios externos necesitan una excepción que reconozca de forma fiable los mensajes ya procesados.

## Causa 6: MX, host inteligente o subdominio apunta de vuelta

Un gateway puede necesitar internamente un siguiente salto diferente del que usan los remitentes externos. Si utiliza simplemente el MX público para el reenvío, este podría volver a apuntar al propio gateway. El mismo problema se produce si un host inteligente vuelve a su propio listener mediante NAT o balanceo de carga.

```powershell
Resolve-DnsName -Type MX contoso.com
Resolve-DnsName -Type MX app.contoso.com

Get-SendConnector |
    Format-List Name,AddressSpaces,DNSRoutingEnabled,SmartHosts
```

Los subdominios merecen una comprobación independiente. Microsoft documenta casos en los que un subdominio de aplicación debe crearse explícitamente como dominio de retransmisión interna y sincronizarse con los sistemas Edge:

```powershell
New-AcceptedDomain `
    -Name "app.contoso.com" `
    -DomainName app.contoso.com `
    -DomainType InternalRelay

Start-EdgeSynchronization
```

Estos comandos no son una solución universal. Solo son adecuados si `app.contoso.com` se entrega realmente fuera de la organización de Exchange y el conector de envío tiene un siguiente salto inequívoco.

## Procedimiento seguro ante un bucle activo

Durante la incidencia, primero debe detenerse la multiplicación. Según la arquitectura, se desactiva de forma controlada la regla de transporte desencadenante o el conector específico, o bien el gateway retiene la cola afectada. Antes de ello, se exportan la configuración y los ejemplos de mensajes.

A continuación, se realiza una prueba con exactamente un remitente, un destinatario y una línea de asunto claramente identificable. El mensaje se sigue de extremo a extremo mediante encabezados, Message Trace y registros de seguimiento locales. Solo cuando termina en el destino previsto se vuelve a abrir gradualmente el flujo de correo.

No se recomienda:

- aumentar los límites de saltos
- modificar varios conectores al mismo tiempo
- alternar por sospecha los dominios aceptados entre `Authoritative` y `InternalRelay`
- volver a inyectar repetidamente una cola problemática sin comprobarla
- corregir directamente en AD o Exchange Online atributos de Exchange sincronizados
- desactivar comprobaciones de TLS, IP o certificados como supuesto arreglo rápido

## Comprobación final

Tras la corrección, la documentación debe contener exactamente una declaración para cada dominio relevante: ¿Qué sistema conoce al destinatario, qué conector es aplicable y qué host es el siguiente salto final?

La validación técnica incluye al menos:

- mensaje de prueba externo e interno
- destinatario desconocido del mismo dominio
- destinatario en cada lado de un dominio dividido real
- mensaje saliente con el gateway o Centralized Mail Transport activado
- encabezados sin secuencia de saltos recurrente
- Message Trace con `Delivered` o la transferencia prevista
- seguimiento local sin un nuevo `RECEIVE` después de un `SEND` hacia el mismo destino
- validación de conectores para todos los conectores que sigan siendo necesarios

Un bucle de correo resuelto solo se considera finalizado cuando no solo llega el correo de prueba, sino que también los destinatarios desconocidos y las rutas alternativas de flujo de correo terminan de forma definida. Ahí es donde se producen la mayoría de las recaídas.

## Fuentes

1. [Microsoft Learn – Fix NDR error 5.4.6 or 5.4.14 in Exchange Online](https://learn.microsoft.com/en-us/troubleshoot/exchange/email-delivery/ndr/fix-error-code-5-4-6-through-5-4-20-in-exchange-online): significado de los NDR de Exchange y causas habituales en dominios aceptados y conectores híbridos.

2. [Microsoft Learn – Manage accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains): diferencias entre `Authoritative` y `InternalRelay`.

3. [Microsoft Learn – Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains): responsabilidad, dominios de retransmisión y búsqueda de destinatarios en el Exchange local.

4. [Microsoft Learn – Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing): rutas de transporte esperadas con y sin Centralized Mail Transport.

5. [Microsoft Learn – Validate connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/validate-connectors): validación de conectores e indicaciones sobre varios conectores que coinciden simultáneamente.

6. [Microsoft Learn – Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud): patrones de flujo de correo compatibles con servicios de terceros previos.

7. [Microsoft Learn – Mail flow rules in Exchange Online](https://learn.microsoft.com/en-us/exchange/security-and-compliance/mail-flow-rules/mail-flow-rules): procesamiento, prioridad, acciones y excepciones de las reglas de transporte.

8. [Microsoft Learn – Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2?view=exchange-ps): búsqueda de mensajes en el transporte de Exchange Online.

9. [Microsoft Learn – Search message tracking logs](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/search-message-tracking-logs): seguimiento local de mensajes en todos los servidores Exchange.

10. [Microsoft Learn – Hop count exceeded for an on-premises application subdomain](https://learn.microsoft.com/en-us/troubleshoot/exchange/mailflow/hop-count-exceeded-possible-mail-loop): escenario documentado de subdominio/EdgeSync con un dominio de retransmisión interna explícito.
