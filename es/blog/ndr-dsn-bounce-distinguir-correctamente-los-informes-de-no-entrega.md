---
title: "NDR, DSN, Bounce: cómo distinguir correctamente los avisos de no entrega"
navTitle: "NDR y bounces"
description: "NDR, DSN, Bounce, Reject, Backscatter: los términos relacionados con entregas fallidas se utilizan a menudo como sinónimos, pero designan cosas diferentes. Qué definen las RFC, quién genera cada aviso, cómo está estructurado un DSN y por qué la diferencia entre Reject y Bounce determina el Backscatter."
date: "2026-08-28"
kategorie: "SMTP y flujo de correo"
timeToRead: "10 min de lectura"
themen:
  - smtp-mailflow
protokolle:
  - "smtp"
  - "troubleshooting"
slug: "ndr-dsn-bounce-distinguir-correctamente-los-informes-de-no-entrega"
translationId: "article-5c5164049a129fa4"
translationOf: ndr-dsn-bounce-unterschiede
translationSourceHash: 1500e27283dc4c53456810f133d12d81e1f60d7e874b6f3450b0a50fb47d4cf2
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:20:40.525Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/ndr-dsn-bounce-distinguir-correctamente-los-informes-de-no-entrega
---

# NDR, DSN, Bounce: cómo distinguir correctamente los avisos de no entrega

Un correo no llega y en el ticket se indica, según el caso, «Bounce», «NDR», «Mailer-Daemon» o «mensaje de error del servidor». En el día a día de la administración, estos términos se usan como sinónimos aunque designan cosas diferentes: un Reject durante la sesión SMTP no es un correo de devolución, un aviso de retraso no es un error de entrega y una confirmación de lectura no tiene nada que ver con la no entrega. Quien diferencia correctamente los términos encuentra antes la causa, porque cada tipo de aviso indica algo distinto sobre dónde se encuentra el problema en la ruta de transporte y quién puede resolverlo.

## DSN: el término genérico de las RFC

El término formal genérico es Delivery Status Notification (DSN), definido en las RFC 3461 a 3464. Un DSN es un correo generado automáticamente que informa al remitente sobre el estado de entrega de su mensaje. Importante: un DSN no solo informa de fallos. El campo `Action` en la parte legible por máquina admite cinco valores:

| Action | Significado |
|---|---|
| `failed` | La entrega ha fallado definitivamente; no se volverá a intentar enviar el correo |
| `delayed` | La entrega se ha retrasado; el servidor seguirá intentándolo |
| `delivered` | Entregado correctamente (confirmación de entrega, solo previa solicitud explícita) |
| `relayed` | Transferido a un servidor que no genera DSN por sí mismo |
| `expanded` | Entregado a una lista de distribución y expandido |

Por tanto, el aviso de no entrega es solo un caso especial: un DSN con `Action: failed`. Microsoft denomina precisamente a este caso especial Non-Delivery Report (NDR). El término NDR procede del mundo de Exchange, pero actualmente se utiliza entre fabricantes. Si se quiere ser preciso: todo NDR es un DSN, pero no todo DSN es un NDR.

El aviso de retraso (`Action: delayed`) merece especial atención porque en soporte se confunde habitualmente con un error de entrega. Un asunto típico es «Delivery delayed» o «Entrega retrasada». El correo permanece entonces en la cola del servidor remitente, que sigue intentándolo, normalmente durante uno o dos días. Solo cuando expira la vida útil de la cola se genera el NDR definitivo. Un usuario que reenvía el correo tras recibir un aviso de retraso genera duplicados en cuanto el sistema de destino vuelve a estar disponible.

## Reject o Bounce: la distinción más importante

Antes de abordar los demás términos, conviene explicar la decisión técnica central, ya que determina qué servidor genera un aviso.

**Reject (rechazo durante la sesión):** El servidor receptor rechaza el correo durante la sesión SMTP, con un código de respuesta 5xx a `RCPT TO` o después de `DATA`. Nunca acepta el correo y no genera ningún correo de respuesta. La obligación de informar al remitente corresponde al servidor de envío: el MTA remitente ve la respuesta 5xx y genera entonces el NDR para su usuario local. En este caso, el NDR que lee el usuario procede de su propio servidor, pero cita el mensaje de error del servidor remoto.

**Bounce (aceptación con devolución posterior):** El servidor receptor acepta el correo con `250 OK` y solo después detecta que no puede entregarlo, por ejemplo porque el buzón no existe, la cuota está llena o un servidor posterior lo rechaza. Ahora es responsable del mensaje y debe enviar él mismo un DSN al remitente. Este correo de devolución posterior es el Bounce en sentido estricto.

Para solucionar problemas, la diferencia es de utilidad inmediata: si el NDR muestra el servidor propio como sistema generador, el correo fue rechazado durante la sesión o ni siquiera llegó a salir. Si aparece un servidor externo como remitente del aviso, la otra parte aceptó inicialmente el correo y el problema se encuentra después de su punto de aceptación, invisible para el remitente.

Del ámbito del marketing proceden otros dos términos de Bounce que no aparecen en ninguna RFC: Hard Bounce para errores definitivos (5xx, `Action: failed`) y Soft Bounce para errores temporales (4xx, `Action: delayed`). Para las plataformas de envío masivo, la distinción es fundamental, porque los Hard Bounces deberían llevar a una depuración inmediata de las listas. Técnicamente, son los mismos mecanismos descritos anteriormente.

## Los términos de un vistazo

| Término | Qué es | Quién genera el aviso | Estándar |
|---|---|---|---|
| DSN | Término genérico: aviso de estado de entrega (failed, delayed, delivered, relayed, expanded) | El MTA responsable del correo | RFC 3461 a 3464 |
| NDR | DSN con `Action: failed`; término de Microsoft para el aviso de no entrega | MTA remitente (tras Reject) o MTA receptor (tras aceptación) | RFC 3464, documentación de Microsoft |
| Reject | Rechazo 5xx durante la sesión SMTP en curso; no es un correo independiente | Nadie; el MTA remitente genera a partir de ello un NDR | RFC 5321 |
| Bounce | Correo de devolución después de una aceptación previa | MTA receptor | RFC 5321, RFC 3464 |
| Hard/Soft Bounce | Clasificación de marketing: definitivo (5xx) frente a temporal (4xx) | igual que Bounce | sin RFC |
| Aviso de retraso | DSN con `Action: delayed`; el correo sigue en la cola | MTA remitente o de retransmisión | RFC 3464 |
| Backscatter | NDR enviados a direcciones de remitente falsificadas, normalmente provocados por spam | MTA receptores mal configurados | sin RFC, término antiabuso |
| MDN / confirmación de lectura | Aviso sobre la visualización o eliminación por parte del destinatario | Cliente de correo del destinatario | RFC 8098 |
| Aviso de ausencia | Respuesta automática de un buzón al que se ha llegado | Servidor de buzón o de groupware | RFC 3834 |

## Estructura de un DSN

Los DSN conformes con el estándar utilizan el tipo MIME `multipart/report; report-type=delivery-status` con tres partes: una explicación legible para personas, una parte legible por máquina del tipo `message/delivery-status` y, opcionalmente, el mensaje original o sus cabeceras. La parte legible por máquina es la más valiosa para el diagnóstico porque sus campos están estandarizados:

```text
Reporting-MTA: dns; mail01.example.net
Received-From-MTA: dns; client.example.org

Final-Recipient: rfc822; max.muster@example.com
Action: failed
Status: 5.1.1
Remote-MTA: dns; mx.example.com
Diagnostic-Code: smtp; 550 5.1.1 <max.muster@example.com>:
    Recipient address rejected: User unknown
```

| Campo | Significado |
|---|---|
| `Reporting-MTA` | El servidor que generó este DSN; primera indicación de responsabilidad |
| `Final-Recipient` | La dirección del destinatario a la que se refiere el estado (un bloque por destinatario) |
| `Action` | Uno de los cinco valores de estado (failed, delayed, delivered, relayed, expanded) |
| `Status` | Enhanced Status Code según RFC 3463, p. ej., `5.1.1` |
| `Remote-MTA` | El servidor remoto con el que habló el MTA informante |
| `Diagnostic-Code` | La respuesta SMTP literal del servidor remoto; a menudo, la línea más reveladora |

Un DSN siempre se envía con un remitente de sobre vacío (`MAIL FROM:<>`). No se trata de una negligencia, sino de una obligación de la RFC 5321: el remitente vacío evita que un DSN no entregable genere otro DSN y que dos servidores se envíen mensajes de error sin fin. De ello se deriva una regla de configuración: un sistema de correo no debe rechazar de forma generalizada los mensajes con remitente de sobre vacío; de lo contrario, los avisos legítimos de no entrega nunca llegarán a los propios usuarios.

Exchange y Exchange Online cumplen el estándar en cuanto al formato, pero presentan el contenido de una manera propia: el usuario ve una página preparada con una explicación en texto claro; debajo figuran «Generating server» (equivale a `Reporting-MTA`) y los datos sin procesar. Para el diagnóstico, siempre merece la pena revisar esta sección técnica inferior.

## Interpretar los Enhanced Status Codes

En el campo `Status` y normalmente también en `Diagnostic-Code` aparece un código de tres partes según RFC 3463: clase.asunto.detalle. La clase indica el carácter vinculante; el asunto y el detalle, la causa:

| Rango de código | Significado |
|---|---|
| `2.x.x` | Éxito (solo en confirmaciones de entrega) |
| `4.x.x` | Error temporal; el servidor lo volverá a intentar |
| `5.x.x` | Error definitivo; no se realizarán más intentos |
| `x.1.x` | Problema de direccionamiento, p. ej., `5.1.1` destinatario desconocido, `5.1.10` dominio sin MX |
| `x.2.x` | Problema de buzón, p. ej., `5.2.2` buzón lleno, `5.2.3` mensaje demasiado grande para el buzón |
| `x.3.x` | Problema del sistema de destino, p. ej., `4.3.2` el sistema no acepta mensajes en este momento |
| `x.4.x` | Red y enrutamiento, p. ej., `4.4.1` sin respuesta, `4.4.7` vida útil de la cola agotada |
| `x.5.x` | Error de protocolo en el diálogo SMTP |
| `x.7.x` | Política y seguridad, p. ej., `5.7.1` retransmisión denegada o rechazo por política, `5.7.26` falta de autenticación (SPF/DKIM/DMARC) |

El código clásico de respuesta SMTP de tres dígitos (por ejemplo, `550`) y el Enhanced Status Code suelen aparecer juntos en una línea: `550 5.7.1 ...`. El código de tres dígitos controla el comportamiento de protocolo del servidor remitente; el código ampliado proporciona el diagnóstico. Si hay contradicciones entre el código y el texto libre, el texto libre del servidor remoto suele ser la fuente más precisa, ya que muchos sistemas establecen códigos genéricos y escriben la causa real en el comentario, incluidos los ID de referencia para el soporte de la otra parte.

Tenga en cuenta que los rechazos `5.7.x` de filtros de reputación y contenido suelen proporcionar deliberadamente poca información. Quien solo se fija en el código buscará en el lugar equivocado; la lista de bloqueo o el fabricante del filtro indicado en el texto libre conduce antes a la solución.

## Backscatter: el tipo de Bounce perjudicial

El Backscatter se produce cuando un servidor acepta primero spam con un remitente falsificado y después envía un NDR a la dirección falsificada. El NDR llega así a una persona ajena cuya dirección ha sido utilizada indebidamente por el spammer. Durante grandes oleadas de spam, los afectados reciben miles de NDR de correos que nunca enviaron, y los servidores que generan estos NDR masivamente terminan ellos mismos en listas de bloqueo (por ejemplo, la lista Backscatterer de UCEPROTECT).

La solución se deduce directamente de la distinción entre Reject y Bounce: todo lo que pueda rechazarse debe rechazarse durante la sesión SMTP, no devolverse tras su aceptación. En concreto, esto significa validar destinatarios en el punto de aceptación más externo (el gateway perimetral conoce las direcciones válidas, mediante comparación con el directorio o Recipient Callout, en lugar de aceptarlo todo y dejar que falle internamente), rechazar spam y malware durante la sesión en vez de emitir NDR de cuarentena y no enviar NDR para mensajes clasificados como spam. Un Reject no genera Backscatter, pues con un remitente falsificado la respuesta 5xx llega al servidor del spammer, que no genera a partir de ella ningún NDR dirigido a la víctima.

## Qué no es un aviso de no entrega

Tres tipos de avisos llegan habitualmente a los tickets como si fueran lo mismo, pero no lo son:

**MDN (Message Disposition Notification, RFC 8098):** La confirmación de lectura. No la genera el sistema de transporte, sino el cliente de correo del destinatario, e informa de la visualización o eliminación del mensaje, no de su entrega. Por tanto, el tipo MIME se denomina `multipart/report; report-type=disposition-notification`. La ausencia de una confirmación de lectura no dice nada sobre la entrega; la mayoría de los clientes preguntan al usuario o suprimen por completo los MDN.

**Avisos de ausencia y autorespondedores (RFC 3834):** Un aviso de ausencia demuestra lo contrario de un error de entrega, pues presupone que el correo ha llegado al buzón. En las descripciones de tickets («recibo una respuesta automática, ¿mi correo llega?»), conviene preguntar qué aviso se ha recibido exactamente.

**Notificaciones de cuarentena:** Los avisos como el resumen de cuarentena de Microsoft 365 o de un gateway informan al destinatario sobre correos retenidos. Se envían al destinatario, no al remitente, y no siguen ningún estándar DSN. En este escenario, el remitente a menudo no recibe nada, lo que explica los casos en que un correo «desaparece sin mensaje de error».

## Lista de comprobación para el diagnóstico

Cuando haya un aviso, aclárelo en este orden:

1. ¿De qué tipo se trata: NDR (`Action: failed`), retraso (`Action: delayed`), MDN, autorespondedor o aviso de cuarentena? En caso de aviso de retraso: espere, no vuelva a enviar.
2. ¿Quién generó el aviso (`Reporting-MTA` o «Generating server»)? El servidor propio significa Reject o error interno; un servidor externo significa aceptación con fallo posterior en el otro extremo.
3. ¿Qué indican el estado y el código de diagnóstico? La clase 4 frente a la clase 5 separa lo temporal de lo definitivo, el asunto (`x.1` dirección, `x.2` buzón, `x.4` red, `x.7` política) delimita la causa, y el texto libre del servidor remoto proporciona los detalles.
4. Si no hay ningún aviso aunque el correo no llegue: revise el Message Tracking en el sistema propio y considere una cuarentena o filtrado silencioso en el otro extremo.

Los artículos sobre [Message Tracking y diagnóstico SMTP en el generador de comandos](https://rafaelpfister.ch/tools/command-builder) y el [Mail-Header-Analyzer](https://rafaelpfister.ch/tools/header-analyzer) para evaluar la ruta de transporte de un correo recibido muestran cómo reproducir específicamente cada ruta de entrega.

## Historial de cambios

- **25 de octubre de 2026:** Corregido: se ha establecido la dirección correcta para el enlace al Mail-Header-Analyzer. Se ha actualizado el enlace de fuentes a la nueva dirección de Microsoft Learn.

## Fuentes

1.  [RFC 3461: SMTP Service Extension for Delivery Status Notifications](https://www.rfc-editor.org/rfc/rfc3461): Extensión SMTP con la que los remitentes pueden solicitar y controlar DSN.

2.  [RFC 3463: Enhanced Mail System Status Codes](https://www.rfc-editor.org/rfc/rfc3463): Definición de los códigos de estado de tres partes (clase.asunto.detalle).

3.  [RFC 3464: An Extensible Message Format for Delivery Status Notifications](https://www.rfc-editor.org/rfc/rfc3464): Estructura del DSN como multipart/report, campos como Action, Status y Diagnostic-Code.

4.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): Reglas básicas sobre códigos de respuesta, transferencia de responsabilidad tras la aceptación y remitente de sobre vacío para mensajes de error.

5.  [RFC 8098: Message Disposition Notification](https://www.rfc-editor.org/rfc/rfc8098): Estándar para confirmaciones de lectura, para distinguirlas de los DSN.

6.  [RFC 3834: Recommendations for Automatic Responses to Electronic Mail](https://www.rfc-editor.org/rfc/rfc3834): Reglas para autorespondedores como los avisos de ausencia.

7.  [Microsoft Learn: Email non-delivery reports and SMTP errors in Exchange Online](https://learn.microsoft.com/en-us/troubleshoot/exchange/email-delivery/ndr/non-delivery-reports-in-exchange-online): Estructura de los NDR y lista de códigos desde la perspectiva de Exchange Online.

8.  [UCEPROTECT Backscatterer](https://www.backscatterer.org/): Lista de bloqueo para sistemas que generan Backscatter; explica los criterios de inclusión.
