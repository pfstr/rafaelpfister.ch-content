---
title: "SMTP: transporte, cola y responsabilidad de entrega"
blatt: "smtp"
description: "Simple Mail Transfer Protocol desde la perspectiva de administración: funciones de MUA, MSA, MTA y MDA, sobre y contenido del mensaje, autómata de estados SMTP, límite de aceptación, enrutamiento MX, cola y reintentos, códigos de respuesta, extensiones ESMTP, submission, política TLS y diagnóstico por capas."
fakten:
  - label: Nombre completo
    wert: Simple Mail Transfer Protocol
    href: https://datatracker.ietf.org/doc/html/rfc5321
  - label: Modelo básico
    wert: Command/Reply y Store-and-Forward
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-2.3.1
  - label: Objeto de transporte
    wert: Sobre más contenido del mensaje
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-2.3.1
  - label: Transporte
    wert: flujo de datos fiable y ordenado; en Internet, TCP
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-1.1
  - label: De servidor a servidor
    wert: SMTP sobre TCP 25
    href: https://datatracker.ietf.org/doc/html/rfc6409#section-1
  - label: Submission
    wert: 587 con STARTTLS · 465 con TLS implícito
    href: https://datatracker.ietf.org/doc/html/rfc8314#section-3
  - label: Funciones principales
    wert: MUA · MSA · MTA · MDA o LMTP
    href: https://datatracker.ietf.org/doc/html/rfc5598#section-4
  - label: Capacidades
    wert: EHLO negocia extensiones ESMTP en cada sesión
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-2.2
  - label: Límite de aceptación
    wert: 250 tras End-of-Data transfiere la responsabilidad
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-4.1.1.4
  - label: Clases de error
    wert: 4xx temporal · 5xx permanente · x.y.z como detalle
    href: https://datatracker.ietf.org/doc/html/rfc3463#section-2
  - label: Dependencias
    wert: DNS · TCP · TLS · SASL · MIME · cola persistente
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-2.3
  - label: Historia normativa
    wert: RFC 821 → RFC 2821 → RFC 5321
    href: https://datatracker.ietf.org/doc/html/rfc5321#appendix-C
werbung:
  - tools
  - newsletter
ctaThemen:
  - smtp-mailflow
translationSourceHash: bd51241025647a47c1c52f82747a0f7eee7d7b8e42d57f52057e7123d6bc3825
translationModel: gpt-5.6-terra
translatedAt: 2026-10-13T11:15:52.316Z
translationReview: automatic
---

# SMTP: transporte, cola y responsabilidad de entrega

El **Simple Mail Transfer Protocol** (SMTP) lleva un mensaje de un proceso de correo al siguiente. El cliente se conecta a un servidor, consulta sus capacidades, indica el remitente y los destinatarios del sobre, y solo entonces transmite el contenido del mensaje. El servidor responde a cada paso con un código numérico. De esta secuencia aparentemente sencilla surgen el enrutamiento, la puesta en cola, los reintentos y la cuestión decisiva de qué servidor tiene en ese momento la responsabilidad del mensaje ([RFC 5321, secciones 1 y 2](https://datatracker.ietf.org/doc/html/rfc5321#section-1)).

Para los administradores de mensajería, la característica decisiva reside en la **transferencia de responsabilidad**. Un establecimiento TCP correcto solo significa que un listener es accesible. Un `RCPT TO` aceptado reserva un destinatario dentro de la transacción en curso. Solo una finalización positiva tras el contenido completo del mensaje transfiere al sistema SMTP receptor la responsabilidad de entregar el mensaje o informar posteriormente de un error al remitente del sobre ([RFC 5321, secciones 4.1.1.4 y 6.1](https://datatracker.ietf.org/doc/html/rfc5321#section-4.1.1.4)). Por ello, el ID de cola, el código de respuesta, el MTA remoto, el estado del destinatario y la traza `Received` no son detalles de registro secundarios, sino la cadena de prueba de una entrega.

La explicación sigue un mensaje desde el cliente que lo presenta, pasando por la sesión SMTP, el sobre y la cola, hasta el siguiente servidor de correo o buzón. DNS, reintentos, códigos de estado, seguridad y diagnóstico se profundizan donde influyen en el recorrido.

## Funciones y responsabilidades

SMTP conoce inicialmente solo **cliente** y **servidor**. En una cadena de entrega real, estas funciones cambian en cada salto: un Mail Transfer Agent acepta un mensaje como servidor, lo almacena en su cola y se convierte él mismo en cliente en el siguiente intento de entrega. Solo a partir de este cambio de funciones pueden interpretarse correctamente los registros y las responsabilidades ([RFC 5321, sección 2.3.3](https://datatracker.ietf.org/doc/html/rfc5321#section-2.3.3), [RFC 5598, sección 4](https://datatracker.ietf.org/doc/html/rfc5598#section-4), [RFC 6409, sección 2.1](https://datatracker.ietf.org/doc/html/rfc6409#section-2.1)).

| Función | Tarea | Límite típico de confianza y estado |
|---|---|---|
| **MUA** – Message User Agent | crea un mensaje y muestra los mensajes recibidos | identidad del usuario, borradores locales, libreta de direcciones y presentación; sin servicio de relay público |
| **MSA** – Message Submission Agent | acepta nuevos mensajes de usuarios o aplicaciones autorizados | identidad autenticada, autorización del remitente, política de submission y devolución inmediata de errores |
| **MTA** – Message Transfer Agent | enruta y transmite mensajes aceptados a otros MTA | cola persistente, planificador de reintentos, selección de destino DNS, política TLS y responsabilidad de entrega |
| **Gateway o relay** | realiza conversiones entre zonas de seguridad, organizaciones o sistemas de mensajería | política, reescritura, inspección de contenido, conversión de protocolo y cola propia |
| **MDA** – Message Delivery Agent | entrega a buzón, archivo, pipe o almacén propietario | resolución final del destinatario, cuota y consistencia del buzón |
| **Servidor LMTP** | acepta un mensaje del MTA para la entrega local | informa un estado independiente por destinatario tras el contenido y no necesita una cola de relay a largo plazo |

El MSA no es simplemente un MTA en otro puerto. Message Submission separa la presentación controlada de mensajes nuevos del transporte abierto entre dominios de correo. Entre otras cosas, un MSA puede comprobar los derechos del remitente, completar campos obligatorios faltantes y adaptar los mensajes a reglas locales; por regla general, un MTA posterior no debería modificar arbitrariamente el contenido de un mensaje ya terminado ([RFC 6409, secciones 1, 6 y 8](https://datatracker.ietf.org/doc/html/rfc6409#section-1)). Este límite es especialmente relevante para las firmas DKIM: cualquier modificación posterior de encabezados firmados o del cuerpo puede invalidar la firma.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-smtp-zustellpfad.svg?v=20260813" title="Interaktive Infografik: SMTP-Rollen, Sitzungsablauf, Envelope, Queue und Zustellverantwortung" loading="lazy">
  <a href="/images/kb-interaktiv-smtp-zustellpfad.svg?v=20260813">Abrir infografía sobre la ruta de entrega SMTP y el límite de aceptación</a>
</iframe>

El gráfico muestra límites lógicos. El MSA, el MTA saliente, el gateway y el MTA entrante pueden ejecutarse como servicios separados, coincidir en un mismo sistema o quedar ocultos por un proveedor de nube. Los estándares no prescriben una arquitectura de procesos. Sin embargo, administrativamente la separación sigue siendo necesaria, porque cada transferencia aceptada puede generar una nueva cola, un nuevo estado de reintento y un nuevo responsable.

## Sobre y contenido del mensaje

Antes de examinar el diálogo en detalle, deben separarse dos áreas de datos: el **sobre** para el transporte y el **contenido del mensaje** para destinatarios y programas de correo. El sobre consta de un Reverse Path de `MAIL FROM` y uno o varios Forward Paths de `RCPT TO`. El contenido sigue a `DATA` e incluye un área de encabezados conforme al Internet Message Format y el cuerpo, a menudo con estructura MIME ([RFC 5321, sección 2.3.1](https://datatracker.ietf.org/doc/html/rfc5321#section-2.3.1), [RFC 5322, sección 2.1](https://datatracker.ietf.org/doc/html/rfc5322#section-2.1)).

| Elemento | Objeto de protocolo | Para qué se utiliza realmente |
|---|---|---|
| `MAIL FROM:<bounce@example.net>` | Envelope Reverse Path | destino de los informes de no entrega y entrada para SPF; puede estar vacío para DSN |
| `RCPT TO:<user@example.ch>` | Envelope Forward Path | enrutamiento real y decisión de destinatario; repetible por destinatario |
| `From: Author <author@example.net>` | Encabezado del mensaje | identidad visible del autor y dominio de referencia DMARC; no es un comando de enrutamiento SMTP |
| `To:` y `Cc:` | Encabezados del mensaje | indicaciones visibles de destino para personas y MUA; no tienen que corresponder a todos los destinatarios del sobre |
| `Bcc:` | Información de creación | los destinatarios Bcc se entregan mediante direcciones del sobre, sin revelar su dirección a las demás copias |
| `Message-ID:` | Encabezado del mensaje | identificador de contenido del mensaje; no es lo mismo que el ID de cola o el ID de sobre DSN |
| `Return-Path:` | Encabezado de traza generado en la entrega final | conserva el Reverse Path utilizado en último lugar; no debe copiarse como línea de remitente establecida libremente desde un mensaje entrante |

Los servidores SMTP no pueden derivar la ruta final de `To:`, `Cc:` ni `From:`. A la inversa, las direcciones del sobre no tienen que aparecer en los encabezados del mensaje. Por eso funcionan Bcc, las listas de distribución, los reenvíos, SRS y las direcciones de rebote separadas. La separación también explica por qué un mensaje puede parecer visualmente en el cliente proceder de `ceo@example.ch`, mientras que su remitente del sobre utiliza otro dominio. SPF considera la ruta SMTP; DMARC establece además una referencia con el `From:` visible. Los detalles de estas comprobaciones se tratan en [autenticación de correo](/kb/mail-auth) ([RFC 5321, apéndice C](https://datatracker.ietf.org/doc/html/rfc5321#appendix-C), [RFC 5322, sección 3.6](https://datatracker.ietf.org/doc/html/rfc5322#section-3.6)).

Un Reverse Path vacío se transmite como `MAIL FROM:<>`. Evita bucles de informes de error y es obligatorio para Delivery Status Notifications. Por tanto, un servidor que rechaza remitentes vacíos de forma general daña la ruta de retorno del sistema de correo ([RFC 5321, sección 3.6.3](https://datatracker.ietf.org/doc/html/rfc5321#section-3.6.3)).

## Sesión y transacción SMTP

En una [conexión TCP](/kb/tcp) se ejecuta una **sesión SMTP**; dentro de ella pueden seguir varios mensajes como **transacciones de correo** separadas. El servidor comienza con un greeting `220`; el cliente se identifica mediante `EHLO` y recibe una lista de capacidades ESMTP asociada a la sesión. Después, una transacción consta de `MAIL`, al menos un `RCPT` aceptado y `DATA`, o bien un sustituto negociado como `BDAT` ([RFC 5321, secciones 3.1 y 4.1](https://datatracker.ietf.org/doc/html/rfc5321#section-3.1)).

1. **Conexión y greeting:** El cliente se conecta; el servidor responde con `220`. Si ya aquí se retrasa o rechaza, aún no existe una transacción de correo.
2. **Negociación de capacidades:** `EHLO client.example` pone a ambas partes en el estado inicial y proporciona extensiones como `SIZE`, `STARTTLS`, `PIPELINING` o `SMTPUTF8`. Un cliente no puede asumir que otro nodo del clúster o una sesión posterior ofrezca las mismas capacidades.
3. **Actualización TLS opcional:** Tras un `STARTTLS` correcto, se descartan el estado SMTP y las capacidades aprendidas previamente. El cliente vuelve a enviar `EHLO` dentro del canal protegido ([RFC 3207, sección 4](https://datatracker.ietf.org/doc/html/rfc3207#section-4)).
4. **Remitente del sobre:** `MAIL FROM` abre una transacción y vacía los búferes anteriores de sobre y datos. Los parámetros ESMTP como `SIZE`, `BODY` o `SMTPUTF8` se adjuntan a este comando.
5. **Destinatarios:** Cada `RCPT TO` recibe una respuesta individual. Un servidor puede aceptar dos destinatarios y rechazar permanentemente un tercero; entonces el cliente envía el contenido para el subconjunto aceptado.
6. **Contenido:** Tras `DATA`, si está preparado, sigue `354`. El cliente transmite encabezados y cuerpo como líneas terminadas en CRLF, escapa los puntos al inicio de línea y termina con una línea que consiste solo en `.`.
7. **Finalización:** La respuesta tras este marcador End-of-Data decide si el servidor asume el mensaje completo para los destinatarios previamente aceptados. `RSET` descarta la transacción en curso sin terminar la sesión; `QUIT` la cierra ordenadamente.

SMTP está orientado a lockstep, pero ESMTP-`PIPELINING` puede enviar grupos de comandos permitidos sin esperar cada respuesta individual. Las respuestas permanecen en orden de entrada y deben seguir asignándose correctamente a cada comando ([RFC 2920](https://datatracker.ietf.org/doc/html/rfc2920)). Por ello, una captura de paquetes con varias líneas `MAIL`-/`RCPT` consecutivas no constituye automáticamente una infracción del protocolo.

Con ello el diálogo está completo, pero la responsabilidad de entrega aún no se ha aclarado. Solo cambia con una respuesta muy concreta del servidor receptor.

## Límite de aceptación: 250 tras End-of-Data

El límite SMTP más importante se encuentra después del contenido del mensaje. Un `250` a `MAIL FROM` solo acepta el Reverse Path. Un `250` a `RCPT TO` acepta un destinatario para la transacción actual. Solo un `250` tras el marcador End-of-Data significa que el servidor ha asumido la responsabilidad del mensaje. Después debe entregarlo o, si detecta un error posteriormente, generar un informe de no entrega; el cliente emisor puede retirar su copia local de la responsabilidad activa de entrega ([RFC 5321, secciones 4.1.1.4 y 6.1](https://datatracker.ietf.org/doc/html/rfc5321#section-4.1.1.4)).

Esta semántica hace crítica la última parte del diálogo. Si la conexión se interrumpe tras el punto, pero antes de recibir una respuesta completa, el cliente no sabe con certeza si el servidor almacenó el mensaje. Debe poder intentar la entrega de nuevo; por ello son posibles los duplicados. SMTP no ofrece una garantía global de exactamente una vez. En consecuencia, los sistemas necesitan seguimiento idempotente mediante ID de cola, `Message-ID`, destinatario, hora, tamaño y, si procede, una referencia de transacción propietaria, sin equiparar entre sí estos identificadores.

Los filtros de contenido, DLP, análisis de malware o firma DKIM deberían integrarse en la canalización de aceptación de modo que el servidor sepa antes de la respuesta positiva final si puede asumir responsabilidad persistente. Un proxy anterior que informe prematuramente `250` y después pierda internamente el mensaje ya no podrá devolver el error al cliente SMTP original. Precisamente aquí una arquitectura debe establecer si la aceptación se confirma solo tras una escritura durable, tras la replicación, tras el análisis o ya tras la transferencia a un broker interno. Se trata de una característica del producto, no de un requisito del estándar SMTP, y debe acreditarse mediante documentación del fabricante y pruebas de error y restauración.

Tras la aceptación, el MTA debe decidir adónde envía el mensaje a continuación. En el correo de Internet, esta decisión comienza en DNS.

## Enrutamiento mediante DNS y MX

Para la entrega directa por Internet, el MTA emisor determina el siguiente destino a partir del dominio del destinatario del sobre. Consulta [DNS](/kb/dns) en busca de registros MX, ordena los candidatos por valor Preference ascendente y resuelve los nombres Exchange seleccionados a direcciones A o AAAA. Varios destinos con la misma Preference forman candidatos equivalentes; un MX de respaldo con valor superior no es una réplica de la cola primaria, sino un receptor SMTP independiente con su propia política y responsabilidad de entrega ([RFC 5321, sección 5](https://datatracker.ietf.org/doc/html/rfc5321#section-5)).

Si falta un conjunto MX-RRset, SMTP trata históricamente el dominio como un MX implícito sobre el propio nombre de dominio e intenta sus direcciones. En cambio, un **Null MX** explícito con Preference `0` y Exchange `.` declara que un dominio no acepta correo electrónico ([RFC 7505](https://datatracker.ietf.org/doc/html/rfc7505)). `NXDOMAIN`, `NODATA`, `SERVFAIL`, los errores de validación DNSSEC y un MX accesible sin servicio SMTP operativo son estados distintos y deben permanecer diferenciados en el motivo de cola.

Las rutas de correo internas suelen usar siguientes saltos estáticos, smarthosts, conectores o tablas de enrutamiento en vez de selección MX pública. El destinatario del sobre sigue siendo la dirección de entrega; la política local de enrutamiento solo decide a qué sistema siguiente se le confía. Por ello, en cada diagnóstico deben anotarse tres nombres: el dominio destinatario, el nombre del siguiente salto seleccionado y la dirección IP realmente conectada.

Aunque el MTA conozca el destino, la entrega no siempre es posible de inmediato. Por ello, los errores temporales pasan a la cola en vez de provocar la pérdida del mensaje.

## Cola, reintento y DSN

Un MTA completo es un sistema Store-and-Forward. Si no puede alcanzar inmediatamente el siguiente salto o este responde temporalmente, el MTA conserva el mensaje y el sobre en una cola persistente, programa un nuevo intento y administra el estado por destinatario. RFC 5321 exige una estrategia de reintento, pero no prescribe un algoritmo único para todos los productos. Recomienda reintentos diferidos, una notificación ante retrasos prolongados y una conservación de varios días antes del abandono definitivo ([RFC 5321, sección 4.5.4](https://datatracker.ietf.org/doc/html/rfc5321#section-4.5.4)). Por tanto, valores de producto como backoff, duración máxima de cola, paralelismo y concurrencia por destino deben consultarse en la configuración concreta del MTA.

Un registro de cola razonable contiene como mínimo:

- contenido del mensaje inmutable o un blob referenciado;
- remitente del sobre y estado de cada destinatario del sobre;
- momento de aceptación, siguiente intento y límite de expiración;
- transporte elegido, siguiente salto, última dirección y decisión TLS/de política;
- último código SMTP y de estado mejorado, junto con el texto remoto;
- ID de cola local, así como IDs correlacionables de saltos anteriores o posteriores;
- estados para active, deferred, hold, delivered, bounced y quarantined;
- estado de replicación, sistema de archivos o base de datos, si la aceptación debe tener alta disponibilidad.

Los errores temporales normalmente se reintentan. Los errores permanentes antes de la propia aceptación se devuelven de forma síncrona al cliente SMTP emisor. Los errores permanentes tras la propia aceptación generan una **Delivery Status Notification** (DSN) al Reverse Path. RFC 3464 define para ello un mensaje `multipart/report` legible por máquina, con campos por mensaje y por destinatario como `Reporting-MTA`, `Final-Recipient`, `Action`, `Status` y, opcionalmente, `Remote-MTA` o `Diagnostic-Code` ([RFC 3464](https://datatracker.ietf.org/doc/html/rfc3464)).

Un rebote no es automáticamente fiable. Las DSN pueden falsificarse, revelar el contenido original o generar backscatter cuando el remitente del sobre se ha establecido de forma abusiva. Un gateway debería rechazar mensajes no válidos durante la sesión SMTP original siempre que sea posible, en lugar de aceptarlos y enviarlos posteriormente a una dirección posiblemente falsificada.

## Códigos de respuesta y significado para el diagnóstico

Las respuestas SMTP tienen un código base de tres dígitos. El rango del primer dígito describe el estado inmediato; el texto está destinado a personas y puede ser específico del producto ([RFC 5321, sección 4.2](https://datatracker.ietf.org/doc/html/rfc5321#section-4.2)).

| Clase | Significado en el diálogo | Consecuencia para el proceso emisor |
|---|---|---|
| `2xx` | finalización positiva | ejecutar el siguiente paso del protocolo; tras End-of-Data, registrar la responsabilidad como transferida |
| `3xx` | respuesta intermedia positiva | enviar más datos, por ejemplo, tras `354` el contenido del mensaje |
| `4xx` | finalización negativa transitoria | conservar el mensaje o destinatario afectado en la cola y volver a intentarlo según la política de reintentos |
| `5xx` | finalización negativa permanente | no repetir sin cambios; informar síncronamente al cliente anterior o generar una DSN tras una aceptación previa |

Los **Enhanced Status Codes** complementan el formato `x.y.z`. `x` indica éxito, temporal o permanente; `y` designa el área general de fallo y `z` el detalle. Así, `5.1.1` pertenece a la clase de direcciones e indica un buzón de destino inexistente, mientras que `4.4.x` clasifica problemas de red y enrutamiento. Los significados registrados figuran en el [registro IANA de Enhanced Status Codes](https://www.iana.org/assignments/smtp-enhanced-status-codes/smtp-enhanced-status-codes.xhtml); un texto de fabricante no debe evaluarse en contradicción con la semántica numérica ([RFC 3463](https://datatracker.ietf.org/doc/html/rfc3463), [RFC 5248](https://datatracker.ietf.org/doc/html/rfc5248)).

El mismo código puede afectar a algo distinto según la fase del protocolo. `550` en `MAIL FROM` rechaza el remitente o su autorización; en `RCPT TO` rechaza un destinatario concreto y, tras el contenido, la transacción completa. Por ello, un ticket sin host remoto, hora, fase, códigos base, código mejorado y texto de respuesta completo está incompleto.

## ESMTP y extensiones

Extended SMTP utiliza `EHLO` para negociar capacidades por conexión. IANA mantiene el [registro vinculante de SMTP Service Extensions](https://www.iana.org/assignments/smtp/smtp.xhtml). Un cliente solo puede utilizar una extensión si se ha ofrecido en la sesión actual; tras STARTTLS debe consultar de nuevo la lista.

| Palabra clave EHLO | Efecto técnico | Relevancia administrativa |
|---|---|---|
| `SIZE` | el servidor puede indicar un límite de tamaño fijo; el cliente puede declarar el tamaño esperado en `MAIL FROM` | evita transferencias de datos innecesarias, pero aún no garantiza la aceptación ([RFC 1870](https://datatracker.ietf.org/doc/html/rfc1870)) |
| `PIPELINING` | permite grupos de comandos definidos sin un round trip por comando | aumenta el rendimiento en rutas con latencia; las respuestas deben correlacionarse exactamente en orden ([RFC 2920](https://datatracker.ietf.org/doc/html/rfc2920)) |
| `8BITMIME` | permite transporte de cuerpos MIME de 8 bits con `BODY=8BITMIME` | no autoriza datos binarios arbitrarios ni modifica las reglas de encabezados ([RFC 6152](https://datatracker.ietf.org/doc/html/rfc6152)) |
| `CHUNKING` | introduce `BDAT` con conteos explícitos de bytes como alternativa a `DATA` | sin transparencia de puntos; requisito para `BINARYMIME` ([RFC 3030](https://datatracker.ietf.org/doc/html/rfc3030)) |
| `SMTPUTF8` | permite buzones internacionalizados y UTF-8 en los encabezados definidos para ello | cada salto posterior requerido debe admitir la capacidad; de lo contrario surge un error de interoperabilidad ([RFC 6531](https://datatracker.ietf.org/doc/html/rfc6531)) |
| `DSN` | transporta parámetros para notificaciones de entrega en `MAIL` y `RCPT` | diferencia informes solicitados de éxito, retraso y error ([RFC 3461](https://datatracker.ietf.org/doc/html/rfc3461)) |
| `STARTTLS` | cambia del diálogo en texto claro a un canal TLS | restablece el estado SMTP; en transferencia MTA, la seguridad depende además de la política ([RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207)) |
| `AUTH` | integra autenticación SASL en SMTP y enumera mecanismos | previsto para submission; los mecanismos ofrecidos pueden cambiar tras STARTTLS ([RFC 4954](https://datatracker.ietf.org/doc/html/rfc4954)) |

La deriva de capacidades entre nodos de clúster es un problema operativo frecuente. Si solo algunos listeners ofrecen `SMTPUTF8`, el mismo valor de `SIZE` o mecanismos AUTH idénticos, se producen errores de entrega aparentemente aleatorios. Por ello, la monitorización no debe comprobar solo un VIP, sino cada ruta de backend relevante con el mismo SNI, nombre EHLO y red de origen.

Hasta aquí se trataba del transporte entre servidores. La presentación por usuarios y aplicaciones es un servicio propio con otros puertos, identidades y riesgos de abuso.

## Submission, relay y autenticación

El puerto **25** sirve para la transferencia de MTA a MTA. El puerto **587** identifica Message Submission; el puerto **465** está registrado para submission con TLS activo desde el inicio como servicio `submissions`. El puerto 465 no es un puerto MX alternativo: la entrega MX pública no conoce indicación de puerto y utiliza el puerto 25 ([RFC 6409, secciones 1 y 3](https://datatracker.ietf.org/doc/html/rfc6409#section-3.1), [RFC 8314, sección 7.3](https://datatracker.ietf.org/doc/html/rfc8314#section-7.3), [IANA Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml)).

Un servicio de submission combina autenticación y autorización. `AUTH` utiliza SASL y puede transportar mecanismos como PLAIN sobre TLS o tokens Bearer basados en OAuth. La identidad autenticada, la identidad de autorización solicitada y el remitente del sobre permitido son magnitudes distintas. Un inicio de sesión correcto no autoriza automáticamente a utilizar cualquier dominio de remitente ni cualquier remitente del sobre ([RFC 4422, sección 3.4](https://datatracker.ietf.org/doc/html/rfc4422#section-3.4), [RFC 4954](https://datatracker.ietf.org/doc/html/rfc4954), [RFC 7628](https://datatracker.ietf.org/doc/html/rfc7628)).

Un relay abierto surge cuando un MTA reenvía mensajes de fuentes no autorizadas a destinos no locales. Por tanto, una configuración segura decide por separado sobre dominios de destinatarios locales, redes de origen fiables, submission autenticada y conectores explícitos. Una allowlist de IP es una autorización de red y no sustituye a una identidad de usuario; con NAT, egreso de nube o relays compartidos puede habilitar un círculo de emisores mucho mayor del previsto.

Solo cuando están definidas la función y el puerto puede determinarse la política TLS adecuada: submission debe estar protegida y autenticada, mientras que el transporte MTA abierto necesita políticas adicionales por dominio.

## Política TLS en SMTP

STARTTLS protege la conexión individual cuando ambas partes realizan la actualización. Sin embargo, en la transferencia MTA abierta, STARTTLS puro sigue siendo oportunista: un atacante activo puede eliminar la oferta o manipular destinos DNS, y un emisor sin política adicional puede volver a texto claro. [DANE para SMTP](https://datatracker.ietf.org/doc/html/rfc7672) vincula los datos TLSA mediante DNSSEC a la selección del destino MX. [MTA-STS](https://datatracker.ietf.org/doc/html/rfc8461) publica mediante DNS y HTTPS una política MX y TLS basada en PKIX. [TLS-RPT](https://datatracker.ietf.org/doc/html/rfc8460) informa de errores agregados con dichas políticas. El artículo [TLS](/kb/tls) trata los detalles criptográficos, las rutas de certificados y los procedimientos de prueba.

Submission es diferente: aquí el cliente y el servicio están asociados administrativamente, y el usuario no debe volver silenciosamente a texto claro. RFC 8314 recomienda TLS implícito para Mail Submission y Access, y exige que el texto claro se trate como obsoleto. SMTP AUTH con mecanismos de contraseña o Bearer necesita un canal adecuadamente protegido; un servidor no debería ofrecer mecanismos débiles antes de STARTTLS ([RFC 8314, sección 3](https://datatracker.ietf.org/doc/html/rfc8314#section-3), [RFC 4954, sección 14](https://datatracker.ietf.org/doc/html/rfc4954#section-14)).

## Entrega local con LMTP

El **Local Mail Transfer Protocol** (LMTP) utiliza un diálogo similar a SMTP para la transferencia a un sistema sin cola de correo propia a largo plazo, por ejemplo, un servicio de buzones. La diferencia esencial aparece tras el contenido del mensaje: LMTP proporciona una respuesta final propia para cada destinatario previamente aceptado. Así, el almacén de correo puede entregar correctamente a un destinatario y rechazar a otro por cuota o error local, sin que el MTA anterior deba tratar toda la transacción como una unidad ([RFC 2033, sección 4.2](https://datatracker.ietf.org/doc/html/rfc2033#section-4.2)).

LMTP no desplaza la responsabilidad fuera del sistema. Si el almacén de correo rechaza temporalmente a un destinatario, el MTA anterior debe seguir conservando ese destinatario en su cola. En los sockets Unix desaparece la ruta de red IP, pero no la necesidad de observar permisos del socket, backpressure, tiempos de espera, respuestas por destinatario y el límite de persistencia.

## Implementación y pila tecnológica

SMTP define el protocolo en el cable, no la arquitectura interna de software. En la práctica, un MTA conecta varias capas técnicas:

| Capa | Componentes típicos | Estado operativo crítico |
|---|---|---|
| Selección de servicio | resolvedor DNS, MX/A/AAAA, tablas de enrutamiento estáticas | caché, estado DNSSEC, selección del siguiente salto e IP conectada |
| Transporte de red | pila TCP, listener, balanceador de carga, proxy | límites de conexión, tiempos de espera, conservación de IP de origen y backpressure |
| Protección de canal | biblioteca TLS, motor de certificados y políticas | SNI, truststore, decisión DANE/MTA-STS y fallback |
| Frontend de protocolo | analizador SMTP, autómata de estados, AUTH/SASL, límites de tasa | estado de sesión, capacidades ofrecidas y límites de abuso |
| Política y contenido | resolución de direcciones, antispam, malware, DLP, firma | orden determinista, tiempos de espera y comportamiento ante fallo del escáner |
| Persistencia | archivos de cola/spool, base de datos o blob store | escritura durable antes de `250`, espacio libre, replicación y recuperación |
| Planificador y enrutamiento | temporizadores de reintento, concurrencia por destino, tablas de transporte | equidad, backoff, antigüedad de cola y dominios bloqueados |
| Adaptador de entrega | SMTP, LMTP, buzón local, archivo, pipe o API | estado por destinatario, idempotencia y límite de transferencia |
| Observabilidad | registros estructurados, ID de cola, métricas, trazas y DSN | correlación entre saltos sin pérdida de contenido sensible |

Las implementaciones organizan estas capas de manera distinta. [Postfix](https://www.postfix.org/OVERVIEW.html) divide recepción, limpieza, Queue Manager y agentes de entrega en procesos cooperativos, y separa `incoming`, `active` y `deferred` como estados de cola. [Exim](https://www.exim.org/exim-html-current/doc/html/spec_html/ch-how_exim_receives_and_delivers_mail.html) almacena sobre/encabezados y cuerpo en el spool, y deja que los queue runners procesen los intentos de entrega vencidos. [Apache James](/kb/apache-james) se ejecuta sobre la JVM y combina servidores de protocolo, cola y una canalización Mailet con backends de almacenamiento intercambiables. Los tres hablan SMTP, pero sus modelos de procesos, colas, extensiones y recuperación no son intercambiables.

El lenguaje de programación y el runtime influyen en el empaquetado, el proceso de parches, el perfil de memoria y la extensibilidad, no en la semántica del diálogo SMTP. Postfix se compila con una toolchain C nativa para sistemas tipo Unix ([Postfix INSTALL](https://www.postfix.org/INSTALL.html)); la [documentación de Exim](https://www.exim.org/exim-html-2.00/doc/html/oview.html) indica ANSI C como lenguaje de implementación. En cambio, Apache James se describe como plataforma Java/Scala sobre la JVM ([Apache James](https://james.apache.org/)). Un servicio en la nube puede representar las mismas funciones RFC mediante microservicios propietarios, colas distribuidas y API. Por tanto, para la aprobación operativa deben documentarse no solo las capacidades EHLO, sino también la garantía de persistencia, el modelo de reintentos, la exportación de cola, la copia de seguridad, la gestión de claves, los límites de escalado y las zonas de fallo.

Del flujo de entrega se deriva una resolución de problemas fija: determinar el destino, comprobar la conexión y el diálogo SMTP, evaluar la respuesta y después seguir la cola y el siguiente salto.

## Diagnóstico desde DNS hasta la cola

Un diagnóstico SMTP fiable sigue la ruta de entrega y mantiene separadas las capas de observación. Las pruebas deben realizarse con dominios de prueba y destinatarios autorizados; no deben utilizarse servidores ajenos para pruebas de relay, validación de direcciones o carga.

### Determinar el destino MX y las direcciones

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-DNS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName example.ch -Type MX -DnsOnly
Resolve-DnsName mx1.example.ch -Type A -DnsOnly
Resolve-DnsName mx1.example.ch -Type AAAA -DnsOnly</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig +noall +answer example.ch MX
dig +noall +answer mx1.example.ch A
dig +noall +answer mx1.example.ch AAAA</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) y [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) muestran datos DNS publicados. La salida debe comprobarse respecto a Preference, Null MX, cadenas CNAME, IPv4/IPv6 y el resolvedor realmente consultado. Una respuesta DNS no acredita ni la función del listener ni la de TLS.

### Comprobar la accesibilidad TCP del salto elegido

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-TCP-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection mx1.example.ch -Port 25 -InformationLevel Detailed
Test-NetConnection submit.example.ch -Port 587 -InformationLevel Detailed
Test-NetConnection submit.example.ch -Port 465 -InformationLevel Detailed</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">nc -vz -w 5 mx1.example.ch 25
nc -vz -w 5 submit.example.ch 587
nc -vz -w 5 submit.example.ch 465</code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) y [`nc`](https://man.openbsd.org/nc) comprueban aquí únicamente la resolución DNS y el establecimiento TCP. Un éxito en el puerto 25 no indica si el greeting SMTP llega a tiempo; un éxito en 587 o 465 no dice nada sobre la política TLS y AUTH.

### Comprobar STARTTLS, EHLO y el certificado

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-STARTTLS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">openssl.exe s_client -connect mx1.example.ch:25 -starttls smtp `
  -name probe.example.net -servername mx1.example.ch `
  -verify_hostname mx1.example.ch -verify_return_error -brief</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">openssl s_client -connect mx1.example.ch:25 -starttls smtp \
  -name probe.example.net -servername mx1.example.ch \
  -verify_hostname mx1.example.ch -verify_return_error -brief</code></pre>
  </div>
</div>

[`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) ejecuta con `-starttls smtp` el diálogo en texto claro necesario, establece con `-name` el nombre SMTP EHLO y con `-servername` SNI. `-verify_hostname` y `-verify_return_error` convierten el mero handshake en una prueba explícita de nombre PKIX. El resultado describe OpenSSL con su truststore, no automáticamente la política del MTA de producción.

### Probar de forma controlada el sobre y la fase de respuesta

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für kontrollierten SMTP-Dialog">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">swaks.exe --server mx1.example.ch --port 25 --tls `
  --ehlo probe.example.net --from sender@example.net `
  --to test-recipient@example.ch --quit-after RCPT</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">swaks --server mx1.example.ch --port 25 --tls \
  --ehlo probe.example.net --from sender@example.net \
  --to test-recipient@example.ch --quit-after RCPT</code></pre>
  </div>
</div>

[`swaks`](https://jetmore.org/john/code/swaks/) registra comandos y respuestas SMTP, y puede terminar deliberadamente la prueba tras `RCPT` sin enviar datos de mensaje. La herramienta no es un componente del sistema en Windows; `swaks.exe` o una instalación de Perl debe proceder de una fuente controlada. Una prueba RCPT solo puede realizarse contra sistemas y direcciones propios o explícitamente autorizados, porque activa políticas, directorios y límites de tasa.

### Examinar listeners y paquetes en el servidor

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-Listener- und Paketprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-NetTCPConnection -State Listen -LocalPort 25,465,587
pktmon filter add SMTP25 -p 25
pktmon start --capture --pkt-size 0
pktmon stop
pktmon etl2pcap PktMon.etl --out smtp.pcapng</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">ss -ltnp '( sport = :25 or sport = :465 or sport = :587 )'
tcpdump -ni any -s 0 -w smtp.pcap 'tcp port 25 or tcp port 465 or tcp port 587'</code></pre>
  </div>
</div>

[`Get-NetTCPConnection`](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection) y [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) asignan listeners y estados TCP a procesos locales. [`pktmon`](https://learn.microsoft.com/windows-server/networking/technologies/pktmon/pktmon) y [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) capturan paquetes; las capturas de producción pueden contener direcciones de sobre, contenidos sin cifrar y datos de autenticación, y deben protegerse, limitarse y eliminarse en consecuencia. [`Wireshark`](https://www.wireshark.org/docs/wsug_html/index.html) es adecuado para analizar handshakes TCP, SMTP y TLS, pero no sustituye ni los registros de cola ni los de política.

### Asignar el error a un límite

| Observación | Límite probable | Siguiente comprobación fiable |
|---|---|---|
| sin MX o Null MX | publicación DNS del dominio destinatario | respuesta autoritativa, estado DNSSEC, TTL de caché negativa |
| tiempo de espera TCP a uno de varios MX | ruta de red o MX individual | cada dirección MX y familia IP por separado, SYN/SYN-ACK y ruta de retorno |
| conexión establecida, sin `220` | listener, proxy, tarpitting o sobrecarga | latencia del greeting, límites de conexión, registros de frontend y backend |
| falta `STARTTLS` solo en conexiones individuales | deriva de capacidades, proxy o política por red de origen | cada nodo de clúster, EHLO antes/después de TLS, ruta del balanceador |
| `4xx` en `MAIL FROM` | política de remitente, tasa o recursos | código completo, IP de origen, remitente del sobre, registro de cola/política |
| `5xx` en un `RCPT TO` | decisión de destinatario o relay | destinatario del sobre afectado, Accepted Domains, Directory Edge |
| `4xx` tras End-of-Data | escáner, almacenamiento, cola o política de contenido | canalización de aceptación local, espacio de spool, latencia del escáner, sin toma de cola |
| `250` tras End-of-Data, DSN posterior | entrega posterior fallida tras la propia aceptación | MTA informante, MTA remoto, ID de cola, estado DSN y destinatario original |
| la cola crece para muchos dominios | planificador local, DNS, red o recursos | distribución por antigüedad, tasas de entrega, estados active/deferred, dependencias globales |
| la cola crece solo para un dominio | MX remoto, política TLS o throttling por destino | último error por MX/IP, DANE/MTA-STS, concurrencia y hora de reintento |
| falta el buzón pese a un salto SMTP correcto | MDA/LMTP o salto de enrutamiento interno | última línea `Received` fiable, respuesta LMTP por destinatario, registros del almacén |
| duplicados esporádicos tras tiempo de espera | respuesta incierta en el límite de aceptación | momento de interrupción respecto al `250` final, correlación de ID de cola y Message-ID |

## Observabilidad y modelo operativo

Un monitor SMTP no debe limitarse a enviar un correo de prueba. Debe hacer visibles los estados individuales del sistema:

- latencia de Greeting, EHLO, STARTTLS, AUTH y End-of-Data por separado;
- conexiones, sesiones, transacciones activas y rechazos por fase;
- capacidades ESMTP ofrecidas por listener y backend;
- tasa de aceptación, número de destinatarios, tamaño de mensaje y latencia de escritura durable;
- volumen de cola por antigüedad, destino, transporte, estado y último error;
- tasa de reintentos, mensaje más antiguo, entregas que expiran y generación de DSN;
- destino DNS, IP conectada, IPv4/IPv6, modo TLS, cipher y resultado de política;
- códigos de respuesta como campos estructurados en lugar de exclusivamente texto libre;
- correlación de ID de edge, ID interno de cola, ID de cola remoto, `Message-ID` y destinatario;
- estado LMTP/buzón por destinatario, así como rutas de cuarentena y liberación;
- capacidad de spool, latencia de I/O, estado de base de datos o broker y retraso de replicación;
- límites de protección para registros y capturas, porque el sobre, los encabezados y los contenidos contienen datos personales o confidenciales.

Las copias de seguridad de la cola por sí solas no bastan. La restauración debe considerar configuración, tablas de enrutamiento, certificados y claves, metadatos de cola, blobs de mensajes, cuarentena, estado de firma e idempotencia. Tras una restauración, una copia de destinatario ya entregada no debe reenviarse de forma incontrolada; al mismo tiempo, no deben perderse los destinatarios aún pendientes de un mensaje entregado parcialmente. Por ello, RPO y RTO deben definirse para cada componente de estado y probarse con una cola real con varios destinatarios.

## Historia técnica

SMTP surgió de procedimientos de correo ARPANET anteriores. El [RFC 821](https://datatracker.ietf.org/doc/html/rfc821) de Jon Postel estandarizó en 1982 el modelo todavía reconocible de SMTP emisor, SMTP receptor, `MAIL`, `RCPT`, `DATA` y respuestas de tres dígitos. El formato de mensaje se describió por separado ese mismo año en RFC 822. Esta separación entre protocolo de transferencia y formato de mensaje se ha mantenido.

Con MIME, SMTP creció más allá del texto puro de 7 bits. El marco ESMTP introdujo `EHLO`, capacidades registradas y parámetros en `MAIL` y `RCPT`; su versión de 1995 figuraba en [RFC 1869](https://datatracker.ietf.org/doc/html/rfc1869). Extensiones basadas en él como `SIZE`, `8BITMIME`, `PIPELINING`, `AUTH`, `STARTTLS`, `CHUNKING`, DSN y, posteriormente, `SMTPUTF8` desarrollaron el protocolo de forma incremental, sin crear una pila paralela de «SMTP 2».

[RFC 2821](https://datatracker.ietf.org/doc/html/rfc2821) consolidó SMTP en 2001 y sustituyó RFC 821 y varios documentos complementarios. [RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321) siguió en 2008 como otra consolidación; el Internet Message Format correspondiente figura en [RFC 5322](https://datatracker.ietf.org/doc/html/rfc5322). Message Submission se especificó inicialmente por separado y se consolidó en RFC 6409. Así, la evolución técnica avanzó mediante funciones más claras, registros de extensiones, direcciones internacionalizadas, informes de entrega y política de transporte, mientras que el diálogo básico con estado se mantuvo compatible.

Las arquitecturas de software también evolucionaron en direcciones distintas. La compatibilidad histórica con Sendmail marcó las interfaces locales de línea de comandos; Postfix apostó por procesos cooperativos separados por privilegios y colas explícitas, Exim por un enfoque de router/transporte y spool muy configurable, y sistemas JVM como Apache James hicieron extensible el procesamiento mediante componentes y Mailets. Las plataformas en nube distribuyen las mismas funciones lógicas entre frontends, brokers, servicios de política y colas regionales. Para los administradores, la semántica RFC sigue siendo el denominador común, mientras que la persistencia, escalabilidad y recuperación siempre deben acreditarse de manera específica para cada producto.

## Fuentes

- [RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321) – modelo SMTP, enrutamiento, comandos, respuestas, cola, reintentos, traza y responsabilidad de entrega.
- [RFC 5598, sección 4](https://datatracker.ietf.org/doc/html/rfc5598)
- [RFC 6409 – Message Submission for Mail](https://datatracker.ietf.org/doc/html/rfc6409) – separación entre submission y relay, funciones, puerto 587 y política MSA.
- [RFC 5322 – Internet Message Format](https://datatracker.ietf.org/doc/html/rfc5322) – encabezados, cuerpo, campos de dirección, Message-ID y campos de traza.
- [RFC 3207 – SMTP Service Extension for Secure SMTP over TLS](https://datatracker.ietf.org/doc/html/rfc3207) – STARTTLS y restablecimiento del estado SMTP.
- [RFC 2920 – SMTP Command Pipelining](https://datatracker.ietf.org/doc/html/rfc2920) – grupos de comandos ordenados sin un round trip por comando.
- [RFC 7505 – A Null MX No Service Resource Record](https://datatracker.ietf.org/doc/html/rfc7505) – identificación explícita de dominios sin capacidad de correo.
- [RFC 3464 – Delivery Status Notification Format](https://datatracker.ietf.org/doc/html/rfc3464) – estructura DSN legible por máquina y estado de destinatarios.
- [IANA – SMTP Enhanced Status Codes](https://www.iana.org/assignments/smtp-enhanced-status-codes/smtp-enhanced-status-codes.xhtml) – significados registrados de los códigos de detalle.
- [RFC 3463 – Enhanced Mail System Status Codes](https://datatracker.ietf.org/doc/html/rfc3463) – estructura y clases de códigos de estado ampliados.
- [RFC 5248 – SMTP Enhanced Status Code Registry](https://datatracker.ietf.org/doc/html/rfc5248) – modelo de registro para Enhanced Codes.
- [IANA – Simple Mail Transfer Protocol Parameters](https://www.iana.org/assignments/smtp/smtp.xhtml) – comandos SMTP, extensiones y palabras clave EHLO.
- [RFC 1870 – SMTP SIZE Extension](https://datatracker.ietf.org/doc/html/rfc1870) – negociación de tamaño y rechazo antes del contenido.
- [RFC 6152 – SMTP Service Extension for 8-bit MIME Transport](https://datatracker.ietf.org/doc/html/rfc6152) – 8BITMIME y parámetro BODY.
- [RFC 3030 – CHUNKING and BINARYMIME](https://datatracker.ietf.org/doc/html/rfc3030) – transferencia BDAT y mensajes MIME binarios.
- [RFC 6531 – SMTP Extension for Internationalized Email](https://datatracker.ietf.org/doc/html/rfc6531) – SMTPUTF8 y buzones internacionales.
- [RFC 3461 – SMTP Service Extension for Delivery Status Notifications](https://datatracker.ietf.org/doc/html/rfc3461) – parámetros de sobre para solicitudes DSN.
- [RFC 4954 – SMTP Service Extension for Authentication](https://datatracker.ietf.org/doc/html/rfc4954) – comando AUTH y perfil SASL para SMTP.
- [RFC 8314, sección 7.3](https://datatracker.ietf.org/doc/html/rfc8314)
- [IANA – Service Name and Transport Protocol Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) – asignación de SMTP, Submission y Submissions a puertos.
- [RFC 4422, sección 3.4](https://datatracker.ietf.org/doc/html/rfc4422)
- [RFC 7628 – SASL Mechanisms for OAuth](https://datatracker.ietf.org/doc/html/rfc7628) – mecanismos SASL basados en OAuth para SMTP y otros protocolos.
- [RFC 7672 – SMTP Security via DANE TLS](https://datatracker.ietf.org/doc/html/rfc7672) – política de transporte SMTP vinculada a DNSSEC y TLSA.
- [RFC 8461 – SMTP MTA Strict Transport Security](https://datatracker.ietf.org/doc/html/rfc8461) – política MX y TLS basada en PKIX.
- [RFC 8460 – SMTP TLS Reporting](https://datatracker.ietf.org/doc/html/rfc8460) – informes agregados sobre errores TLS de transporte.
- [RFC 2033, sección 4.2](https://datatracker.ietf.org/doc/html/rfc2033)
- [Postfix – Architecture Overview](https://www.postfix.org/OVERVIEW.html) – funciones de proceso, Queue Manager y agentes de entrega.
- [Exim – How Exim receives and delivers mail](https://www.exim.org/exim-html-current/doc/html/spec_html/ch-how_exim_receives_and_delivers_mail.html) – modelo de spool, enrutamiento, transporte y queue runner.
- [Postfix – Installation From Source Code](https://www.postfix.org/INSTALL.html) – requisitos nativos de compilación, plataforma y toolchain.
- [Exim – resumen técnico](https://www.exim.org/exim-html-2.00/doc/html/oview.html) – ANSI C como lenguaje de implementación y características técnicas del producto.
- [Apache James Project](https://james.apache.org/) – arquitectura de JVM, Java/Scala y componentes.
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) – consultas DNS en Windows.
- [BIND 9 – manual de dig](https://bind9.readthedocs.io/en/latest/manpages.html) – consultas DNS en sistemas Unix.
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) – accesibilidad TCP en Windows.
- [OpenBSD – nc(1)](https://man.openbsd.org/nc) – conexiones TCP con Netcat.
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/) – diagnóstico de STARTTLS, SNI y certificados.
- [Swaks – Swiss Army Knife for SMTP](https://jetmore.org/john/code/swaks/) – prueba de transacciones y extensiones SMTP.
- [Microsoft Learn – Get-NetTCPConnection](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection) – listeners y conexiones TCP locales.
- [Linux man-pages – ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html) – diagnóstico de sockets y estados TCP.
- [Microsoft Learn – Packet Monitor](https://learn.microsoft.com/windows-server/networking/technologies/pktmon/pktmon) – captura de paquetes de Windows y conversión PCAPNG.
- [manual de tcpdump(1)](https://www.tcpdump.org/manpages/tcpdump.1.html) – diagnóstico basado en paquetes en sistemas Unix.
- [Guía de usuario de Wireshark](https://www.wireshark.org/docs/wsug_html/index.html) – análisis interactivo de trazas de red y protocolo.
- [RFC 821 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc821) – especificación SMTP histórica de 1982.
- [RFC 1869 – SMTP Service Extensions](https://datatracker.ietf.org/doc/html/rfc1869) – marco ESMTP histórico y modelo EHLO.
- [RFC 2821 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc2821) – consolidación SMTP de 2001.
- [RFC 3848 – ESMTP and LMTP Transmission Types](https://datatracker.ietf.org/doc/html/rfc3848) – valores `with` como ESMTPSA en encabezados Received.
