---
title: "NDR, DSN, Bounce: distinguir correctamente los informes de no entrega"
navTitle: "NDR y bounces"
description: "NDR, DSN, bounce, reject, backscatter: los términos relacionados con entregas fallidas se utilizan a menudo como sinónimos, pero designan cosas diferentes. Qué definen las RFC, quién genera cada mensaje, cómo se estructura una DSN y por qué la diferencia entre reject y bounce determina el backscatter."
date: "2026-08-28"
kategorie: "SMTP y flujo de correo"
timeToRead: "10 min de lectura"
themen:
  - smtp-mailflow
produkte:
  - "uebergreifend"
protokolle:
  - "smtp"
  - "troubleshooting"
slug: "ndr-dsn-bounce-distinguir-correctamente-los-informes-de-no-entrega"
translationId: "article-5c5164049a129fa4"
aiPrompt: |
  Du bist mein Mailflow-Assistent. Ich füge dir gleich eine Unzustellbarkeitsmeldung (NDR/DSN) ein. Analysiere sie Schritt für Schritt: 1. Welcher Server hat die Meldung erzeugt (Reporting-MTA bzw. Generating server)? 2. Wurde die Mail in der SMTP-Session abgewiesen oder nach Annahme zurückgeschickt? 3. Was bedeuten SMTP-Antwortcode und Enhanced Status Code (RFC 3463) konkret? 4. Liegt die Ursache beim Absender, beim Empfänger oder auf dem Transportweg? 5. Welche nächsten Diagnose-Schritte empfiehlst du?
translationOf: ndr-dsn-bounce-unterschiede
url: https://rafaelpfister.ch/es/blog/ndr-dsn-bounce-distinguir-correctamente-los-informes-de-no-entrega
translationSourceHash: e526de6f4a454b4f4975eac3e8a406ab5b30314c624bf12c69f87bec99fdd0e7
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:33:30.993Z
translationReview: automatic
---

# NDR, DSN, Bounce: distinguir correctamente los informes de no entrega

Un correo no llega y en el ticket aparece «bounce», «NDR», «Mailer-Daemon» o «mensaje de error del servidor». En el día a día de la administración, estos términos se usan como sinónimos aunque designan cosas diferentes: un reject en la sesión SMTP no es un correo de devolución, una notificación de retraso no es un error de entrega y una confirmación de lectura no tiene nada que ver con la no entrega. Quien distingue correctamente los términos encuentra la causa más rápido, porque cada tipo de mensaje indica algo distinto sobre dónde se encuentra el problema en la ruta de transporte y quién puede resolverlo.

## DSN: el término genérico de las RFC

El término genérico formal es Delivery Status Notification (DSN), definido en las RFC 3461 a 3464. Una DSN es un correo generado automáticamente que informa al remitente sobre el estado de entrega de su mensaje. Lo decisivo: una DSN no informa solo de fallos. El campo `Action` de la parte legible por máquina tiene cinco valores:

| Action | Significado |
|---|---|
| `failed` | La entrega ha fallado definitivamente; no se volverá a intentar enviar el correo |
| `delayed` | La entrega está retrasada; el servidor sigue intentándolo |
| `delivered` | Entregado correctamente (confirmación de entrega, solo previa solicitud explícita) |
| `relayed` | Transferido a un servidor que no genera DSN por sí mismo |
| `expanded` | Entregado a una lista de distribución y expandido |

Por tanto, el informe de no entrega es solo un caso especial: una DSN con `Action: failed`. Microsoft denomina precisamente a este caso especial Non-Delivery Report (NDR). El término NDR procede del mundo de Exchange, pero actualmente se utiliza entre fabricantes. Para ser precisos: todo NDR es una DSN, pero no toda DSN es un NDR.

La notificación de retraso (`Action: delayed`) merece especial atención, porque en soporte se interpreta habitualmente como un error de entrega. Un asunto típico es «Delivery delayed» o «Entrega retrasada». El correo permanece entonces en la cola del servidor remitente, que sigue intentándolo, normalmente durante uno o dos días. Solo cuando expira la vida útil de la cola se genera el NDR definitivo. Un usuario que reenvía el correo al recibir una notificación de retraso genera duplicados en cuanto el sistema de destino vuelve a estar disponible.

## Reject o bounce: la distinción más importante

Antes de abordar los demás términos, es necesario explicar la bifurcación técnica central, ya que determina qué servidor genera un mensaje.

**Reject (rechazo en la sesión):** El servidor receptor rechaza el correo ya durante la sesión SMTP, con un código de respuesta 5xx a `RCPT TO` o después de `DATA`. Nunca acepta el correo y tampoco genera un correo de respuesta. La obligación de informar al remitente recae en el servidor de envío: el MTA remitente ve la respuesta 5xx y genera entonces el NDR para su usuario local. En este caso, el NDR que lee el usuario procede de su propio servidor, pero cita el mensaje de error del servidor remoto.

**Bounce (aceptación con devolución posterior):** El servidor receptor acepta el correo con `250 OK` y solo después detecta que no puede entregarlo, por ejemplo porque el buzón no existe, la cuota está llena o un servidor posterior lo rechaza. En ese momento asume la responsabilidad del mensaje y debe enviar él mismo una DSN al remitente. Este correo de devolución posterior es el bounce en sentido estricto.

Para solucionar problemas, la diferencia es directamente útil: si el NDR indica el propio servidor como sistema generador, el correo fue rechazado durante la sesión o ni siquiera llegó a salir. Si un servidor externo aparece como remitente del mensaje, la otra parte aceptó inicialmente el correo y el problema está detrás de su punto de aceptación, invisible para el remitente.

Del ámbito del marketing proceden otros dos términos relacionados con bounces que no aparecen en ninguna RFC: hard bounce para errores definitivos (5xx, `Action: failed`) y soft bounce para errores temporales (4xx, `Action: delayed`). Para las plataformas de mailing, la distinción es fundamental, ya que los hard bounces deberían llevar a una depuración inmediata de la lista. Técnicamente, son los mismos mecanismos descritos anteriormente.

## Los términos de un vistazo

| Término | Qué es | Quién genera el mensaje | Estándar |
|---|---|---|---|
| DSN | Término genérico: mensaje de estado sobre la entrega (failed, delayed, delivered, relayed, expanded) | El MTA que asume la responsabilidad del correo | RFC 3461 a 3464 |
| NDR | DSN con `Action: failed`; término de Microsoft para el informe de no entrega | MTA remitente (tras un reject) o MTA receptor (tras la aceptación) | RFC 3464, documentación de Microsoft |
| Reject | Rechazo 5xx durante la sesión SMTP activa; no es un correo independiente | Nadie; el MTA remitente lo convierte en un NDR | RFC 5321 |
| Bounce | Correo de devolución tras haberse aceptado el mensaje | MTA receptor | RFC 5321, RFC 3464 |
| Hard/Soft Bounce | Clasificación de marketing: definitivo (5xx) frente a temporal (4xx) | como el bounce | sin RFC |
| Notificación de retraso | DSN con `Action: delayed`; el correo sigue en la cola | MTA remitente o de retransmisión | RFC 3464 |
| Backscatter | NDR enviados a direcciones de remitente falsificadas, normalmente provocados por spam | MTA receptores mal configurados | sin RFC, término antiabuso |
| MDN / confirmación de lectura | Mensaje sobre la visualización o eliminación por parte del destinatario | Cliente de correo del destinatario | RFC 8098 |
| Respuesta de ausencia | Respuesta automática de un buzón alcanzado | Servidor de buzón o groupware | RFC 3834 |

## Estructura de una DSN

Las DSN conformes al estándar utilizan el tipo MIME `multipart/report; report-type=delivery-status` con tres partes: una explicación legible por humanos, una parte legible por máquina de tipo `message/delivery-status` y, opcionalmente, el mensaje original o sus encabezados. La parte legible por máquina es la más valiosa para el diagnóstico, ya que sus campos están normalizados:

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
| `Reporting-MTA` | El servidor que generó esta DSN; primera pista sobre la responsabilidad |
| `Final-Recipient` | La dirección del destinatario a la que se refiere el estado (un bloque por destinatario) |
| `Action` | Uno de los cinco valores de estado (failed, delayed, delivered, relayed, expanded) |
| `Status` | Enhanced Status Code según RFC 3463, p. ej., `5.1.1` |
| `Remote-MTA` | El servidor remoto con el que habló el MTA informante |
| `Diagnostic-Code` | La respuesta SMTP literal del servidor remoto; a menudo la línea más reveladora |

Una DSN se envía siempre con un remitente de sobre vacío (`MAIL FROM:<>`). No es una negligencia, sino un requisito de RFC 5321: el remitente vacío evita que una DSN no entregable genere otra DSN y que dos servidores se envíen mensajes de error sin fin. De ello se deriva una regla de configuración: un sistema de correo no debe rechazar de forma generalizada los mensajes con remitente de sobre vacío; de lo contrario, los informes legítimos de no entrega nunca llegarán a los propios usuarios.

Exchange y Exchange Online respetan el estándar en cuanto al formato, pero presentan el contenido de una forma propia: el usuario ve una página preparada con una explicación en texto claro y, debajo, «Generating server» (equivale a `Reporting-MTA`) y los datos sin procesar. Para el diagnóstico, siempre merece la pena revisar esta parte técnica inferior.

## Leer los Enhanced Status Codes

En el campo `Status` y, habitualmente, también en `Diagnostic-Code`, aparece un código de tres partes según RFC 3463: clase.asunto.detalle. La clase indica el carácter vinculante, y el asunto y el detalle indican la causa:

| Rango de códigos | Significado |
|---|---|
| `2.x.x` | Éxito (solo en confirmaciones de entrega) |
| `4.x.x` | Error temporal; el servidor volverá a intentarlo |
| `5.x.x` | Error definitivo; no habrá más intentos |
| `x.1.x` | Problema de direccionamiento, p. ej., `5.1.1` destinatario desconocido, `5.1.10` dominio sin MX |
| `x.2.x` | Problema de buzón, p. ej., `5.2.2` buzón lleno, `5.2.3` mensaje demasiado grande para el buzón |
| `x.3.x` | Problema del sistema de destino, p. ej., `4.3.2` el sistema no acepta nada en este momento |
| `x.4.x` | Red y enrutamiento, p. ej., `4.4.1` sin respuesta, `4.4.7` vida útil de la cola expirada |
| `x.5.x` | Error de protocolo en el diálogo SMTP |
| `x.7.x` | Política y seguridad, p. ej., `5.7.1` relé denegado o rechazo por política, `5.7.26` falta de autenticación (SPF/DKIM/DMARC) |

El código de respuesta SMTP clásico de tres dígitos (por ejemplo, `550`) y el Enhanced Status Code suelen aparecer juntos en una línea: `550 5.7.1 ...`. El código de tres dígitos controla el comportamiento del protocolo del servidor remitente; el código ampliado aporta la información de diagnóstico. Si hay contradicciones entre el código y el texto libre, el texto libre del servidor remoto suele ser la fuente más precisa, ya que muchos sistemas establecen códigos genéricos y escriben la causa real en el comentario, incluidos los ID de referencia para el soporte de la otra parte.

Téngase en cuenta que los rechazos `5.7.x` realizados por filtros de reputación y contenido suelen aportar deliberadamente poca información. Quien se fija únicamente en el código buscará en el lugar equivocado; la lista de bloqueo o el fabricante del filtro indicados en el texto libre conducen más rápido a la solución.

## Backscatter: el tipo dañino de bounce

El backscatter se produce cuando un servidor acepta primero spam con un remitente falsificado y después envía un NDR a la dirección falsificada. El NDR llega así a una persona no implicada cuya dirección ha sido utilizada indebidamente por el spammer. En grandes oleadas de spam, los afectados reciben miles de NDR de correos que nunca enviaron, y los servidores que generan masivamente estos NDR acaban ellos mismos en listas de bloqueo (como la lista Backscatterer de UCEPROTECT).

La solución se deriva directamente de la distinción entre reject y bounce: todo lo que pueda rechazarse debe rechazarse durante la sesión SMTP, no devolverse después de aceptarlo. En concreto, esto significa validar los destinatarios en el punto de aceptación más externo (la gateway perimetral conoce las direcciones válidas, mediante comparación con el directorio o Recipient Callout, en lugar de aceptarlo todo y dejar que falle internamente), rechazar spam y malware durante la sesión en vez de generar NDR de cuarentena, y no enviar NDR para mensajes clasificados como spam. Un reject no genera backscatter, porque con un remitente falsificado la respuesta 5xx llega al servidor del spammer, que no genera a partir de ella un NDR para la víctima.

## Lo que no es un informe de no entrega

Tres tipos de mensajes llegan regularmente a los tickets agrupados con los anteriores, pero no pertenecen a esta categoría:

**MDN (Message Disposition Notification, RFC 8098):** La confirmación de lectura. No la genera el sistema de transporte, sino el cliente de correo del destinatario, e informa de la visualización o eliminación del mensaje, no de su entrega. El tipo MIME se llama correspondientemente `multipart/report; report-type=disposition-notification`. La ausencia de una confirmación de lectura no dice nada sobre la entrega; la mayoría de los clientes preguntan al usuario o suprimen por completo los MDN.

**Respuestas de ausencia y autorespondedores (RFC 3834):** Una respuesta de ausencia demuestra lo contrario de un error de entrega, pues presupone que el correo ha llegado al buzón. En las descripciones de tickets («recibo una respuesta automática, ¿llega mi correo?»), conviene preguntar qué mensaje concreto está presente.

**Notificaciones de cuarentena:** Mensajes como el resumen de cuarentena de Microsoft 365 o de una gateway informan al destinatario sobre correos retenidos. Se envían al destinatario, no al remitente, y no siguen ningún estándar DSN. En este escenario, el remitente a menudo no recibe nada, lo que explica los casos en los que un correo «desaparece sin mensaje de error».

## Lista de comprobación para el diagnóstico

Si hay un mensaje, aclare los siguientes puntos en este orden:

1. ¿De qué tipo se trata: NDR (`Action: failed`), retraso (`Action: delayed`), MDN, autorespondedor o aviso de cuarentena? Si es una notificación de retraso: espere, no reenvíe el mensaje.
2. ¿Quién generó el mensaje (`Reporting-MTA` o «Generating server»)? El propio servidor implica un reject o un error interno; un servidor externo implica aceptación con fallo posterior en el lado remoto.
3. ¿Qué indican el estado y el Diagnostic-Code? La clase 4 frente a la clase 5 separa lo temporal de lo definitivo; el asunto (`x.1` dirección, `x.2` buzón, `x.4` red, `x.7` política) delimita la causa, y el texto libre del servidor remoto proporciona los detalles.
4. Si no hay ningún mensaje aunque el correo no llega: compruebe el seguimiento de mensajes en el sistema propio y piense en cuarentena o filtrado silencioso en el lado remoto.

Los artículos sobre [Message Tracking y diagnóstico SMTP en el generador de comandos](https://rafaelpfister.ch/tools/command-builder) y el [analizador de encabezados de correo](https://rafaelpfister.ch/tools/mail-header-analyzer) muestran cómo reproducir posteriormente rutas de entrega concretas para analizar la ruta de transporte de un correo recibido.

## Fuentes

1.  [RFC 3461: SMTP Service Extension for Delivery Status Notifications](https://www.rfc-editor.org/rfc/rfc3461): extensión SMTP con la que los remitentes pueden solicitar y controlar DSN.

2.  [RFC 3463: Enhanced Mail System Status Codes](https://www.rfc-editor.org/rfc/rfc3463): definición de los códigos de estado de tres partes (clase.asunto.detalle).

3.  [RFC 3464: An Extensible Message Format for Delivery Status Notifications](https://www.rfc-editor.org/rfc/rfc3464): estructura de la DSN como multipart/report, campos como Action, Status y Diagnostic-Code.

4.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): reglas básicas sobre códigos de respuesta, transferencia de responsabilidad tras la aceptación y remitente de sobre vacío para mensajes de error.

5.  [RFC 8098: Message Disposition Notification](https://www.rfc-editor.org/rfc/rfc8098): estándar para confirmaciones de lectura, para distinguirlas de las DSN.

6.  [RFC 3834: Recommendations for Automatic Responses to Electronic Mail](https://www.rfc-editor.org/rfc/rfc3834): reglas para autorespondedores como las respuestas de ausencia.

7.  [Microsoft Learn: Email non-delivery reports and SMTP errors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/non-delivery-reports-in-exchange-online/non-delivery-reports-in-exchange-online): estructura de NDR y lista de códigos desde la perspectiva de Exchange Online.

8.  [UCEPROTECT Backscatterer](https://www.backscatterer.org/): lista de bloqueo para sistemas que generan backscatter; explica los criterios de inclusión.
