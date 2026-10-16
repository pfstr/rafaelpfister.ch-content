---
title: "Totemomail y Kiteworks EPG: gateway de correo y cambio de plataforma"
blatt: "totemomail"
description: "Totemomail y su sucesor Kiteworks Email Protection Gateway para administradores de mensajería: arquitectura SMTP en línea, políticas, S/MIME, OpenPGP y TLS, WebMail, pila criptográfica Java, estado de claves, identidad, licencias, migración, alta disponibilidad, monitorización y recuperación."
fakten:
  - label: Línea de productos
    wert: totemomail → Kiteworks Email Protection Gateway
    href: https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf
  - label: Función del sistema
    wert: gateway de cifrado de correo electrónico basado en políticas
    href: https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf
  - label: Transporte
    wert: SMTP en la ruta de mensajes
    href: https://marketplace.microsoft.com/en/marketplace/apps/totemo.totemo-azr-tm7?tab=overview
  - label: Procedimientos
    wert: S/MIME · OpenPGP · TLS
    href: https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf
  - label: Sin clave del destinatario
    wert: entrega mediante WebMail o portal
    href: https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf
  - label: Decisión
    wert: políticas centrales y selección automática de claves
    href: https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf
  - label: Núcleo criptográfico heredado
    wert: Totemo Security Platform · TCM basado en Java
    href: https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/3500
  - label: Plataforma EPG
    wert: Kiteworks Hardened Virtual Appliance
    href: https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf
  - label: Almacenamiento de datos EPG
    wert: base de datos integrada no administrada directamente por el cliente
    href: https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf
  - label: Observabilidad
    wert: evidencia de flujo de correo · Syslog · auditoría de plataforma
    href: https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf
  - label: Modelos operativos de EPG
    wert: on-premises · recursos propios en la nube · hosting de tenant único
    href: https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf
  - label: Origen
    wert: totemo AG, Suiza · fundada en 2001 · adquirida por Kiteworks en 2022
    href: https://www.kiteworks.com/sites/default/files/resources/on-the-hunt-kiteworks-secure-content-strategy.pdf
werbung:
  - tools
  - newsletter
ctaThemen:
  - totemomail
translationSourceHash: d949453b9a80abb64f76a8ecdee768813548ae2987b18b071cc23d33b0c5df21
translationModel: gpt-5.6-terra
translatedAt: 2026-10-15T08:55:06.125Z
translationReview: automatic
---

# Totemomail y Kiteworks EPG: gateway de correo y cambio de plataforma

Totemomail era una familia de productos de la empresa suiza totemo AG para el cifrado centralizado de correo electrónico. Tras la adquisición por Kiteworks, según el fabricante, el código se integró en la plataforma Kiteworks y continuó como **Email Protection Gateway**, EPG. Por tanto, los nombres no designan dos productos totalmente independientes, pero tampoco el mismo entorno operativo: una appliance Totemomail existente y un EPG dentro de la Kiteworks Hardened Virtual Appliance tienen límites distintos de plataforma, base de datos, administración y recuperación ([Kiteworks: FAQ for totemomail customers](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf), [Kiteworks: EPG capability brief](https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf)).

Desde el punto de vista funcional, el papel principal sigue siendo el de un gateway de correo: procesa correos electrónicos conforme a directivas centrales y utiliza S/MIME, OpenPGP o TLS; para destinatarios sin claves adecuadas se dispone de una solución WebMail o de portal. La documentación del fabricante describe la gestión automática de claves y certificados, así como el uso con clientes de correo habituales sin un complemento de cliente específico de EPG como modelo básico ([Kiteworks: Enterprise packaging and EPG features](https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf), [Microsoft Marketplace: totemomail Encryption Gateway](https://marketplace.microsoft.com/en/marketplace/apps/totemo.totemo-azr-tm7?tab=overview)).

Para los administradores de mensajería, la línea de productos es ante todo una cadena de límites de confianza. El gateway acepta un mensaje mediante SMTP, clasifica la dirección y las identidades, determina una política, busca claves o un canal alternativo, modifica en su caso el contenido MIME y entrega el mensaje al siguiente salto. Con WebMail se añade un estado propio de web, cuentas, claves y almacenamiento. La afirmación «el mensaje está cifrado» está incompleta sin indicar el procedimiento, el tramo protegido, el titular de la clave y el momento ([RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321), [RFC 8551](https://datatracker.ietf.org/doc/html/rfc8551), [RFC 9580](https://datatracker.ietf.org/doc/html/rfc9580)).

La explicación sigue un mensaje a través de las reglas EPG, la selección de claves y los procedimientos de cifrado hasta la entrega directa o WebMail. A continuación se tratan por separado el cambio de Totemomail a Kiteworks, la operación, el diagnóstico y la recuperación.

Totemomail y Kiteworks EPG se sitúan como gateway en línea en la ruta SMTP. Un mensaje se acepta, se asigna a una identidad y una política, se protege con un procedimiento adecuado y después se entrega mediante SMTP o portal. El cambio de plataforma modifica la implementación y la operación, no esta tarea básica.

## Enfoque de arquitectura: gateway en línea con política central

En el funcionamiento como gateway, el sistema de correo interno envía los mensajes salientes mediante [SMTP](/kb/smtp) a Totemomail o EPG. Los mensajes entrantes llegan al gateway directamente a través de MX o mediante un sistema de seguridad de correo anterior. El gateway termina la sesión SMTP, procesa el mensaje y abre una nueva sesión para reenviarlo. Un handshake TCP o TLS correcto todavía no constituye la aceptación del mensaje; la transferencia de responsabilidad solo tiene lugar con la respuesta SMTP positiva al final de la fase DATA ([RFC 5321, sección 4.1.1.4](https://datatracker.ietf.org/doc/html/rfc5321#section-4.1.1.4), [RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207)).

La descripción pública del producto EPG distingue varias rutas de protección. «Encryption Gateway» cubre el cifrado y descifrado de gateway externos. «Internal Encryption» protege los mensajes en su tránsito y almacenamiento en una plataforma de correo en la nube. Otros componentes documentados son Directory Proxy, Application Integration Proxy y un conector para RMS o Microsoft Information Protection. Qué funciones están licenciadas y realmente activadas debe obtenerse de la configuración propia y del contrato; el nombre del producto por sí solo no lo demuestra ([Kiteworks: Enterprise packaging and licensing](https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf)).

Para Microsoft 365, la integración de transporte sigue siendo un problema de conectores. Exchange Online debe enrutar los mensajes específicamente al gateway y aceptar el tráfico de retorno basándose en una identidad fiable de la contraparte. Un MX delante de un gateway de terceros también modifica qué IP de origen ve Exchange Online para la evaluación de spam y suplantación. Microsoft documenta conectores y Enhanced Filtering para ello; ambos están separados de la política criptográfica de EPG ([Microsoft: Configure mail flow using connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow), [Microsoft: Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)).

## Pila tecnológica y límites verificables del producto

El núcleo criptográfico de la plataforma histórica Totemomail está bien documentado públicamente. La documentación NIST-CMVP describe el **Totemo Cryptographic Module**, TCM, como biblioteca Java común de Totemo Security Platform, TSP. La validación de 2019 menciona Totemo Appliance OS 2.3, JRE 8 y un módulo criptográfico de software con AES, RSA, ECDSA, familia SHA, HMAC y generador de números aleatorios; la entrada tiene ahora estatus histórico y, por tanto, no demuestra una adquisición actualmente admisible ni un modo de algoritmo concreto instalado ([NIST CMVP Certificate 3500](https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/3500), [Totemo TCM Security Policy](https://csrc.nist.gov/CSRC/media/projects/cryptographic-module-validation-program/documents/security-policies/140sp3500.pdf)).

La política de seguridad de NIST delimita expresamente TCM como componente de TSP. No documenta un flujo de correo completo de Totemomail ni una implementación universal de cola, base de datos o MTA. Por ello, esos detalles no deben transferirse de una instalación ajena de Apache James o de una única observación de campo a todas las versiones de Totemomail o EPG. Para un documento operativo son determinantes la configuración exportada del producto, la matriz de soporte, el inventario de paquetes o servicios y el manual del fabricante de la generación realmente instalada.

Con EPG, el límite se desplaza a la Kiteworks Hardened Virtual Appliance. Las FAQ de migración describen que el sistema operativo, el servidor web y la base de datos están incluidos en la appliance y que los administradores de clientes no reciben acceso directo al sistema operativo ni a la base de datos. Las actualizaciones se aplican como una unidad de appliance probada. Esto reduce la operación de componentes por parte del cliente, pero convierte los procedimientos de copia de seguridad, clúster, actualización y soporte de la appliance en la superficie de control principal ([Kiteworks: FAQ for totemomail customers](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf)).

| Espacio de estado | Totemomail heredado | Kiteworks EPG | Evidencia administrativa |
|---|---|---|---|
| SMTP y enrutamiento | Configuración de gateway, MTA y destino de la instalación | EPG más modelo de red y roles de Kiteworks | Correo de prueba, respuesta SMTP, ID de cola/flujo de correo, siguiente salto |
| Política | Reglas y desencadenantes de Totemomail | Política EPG integrada más controles de plataforma | Política exportada, prioridad, estado activado, matriz de pruebas |
| Claves | TSP/TCM, certificados y claves OpenPGP | Inventario migrado de claves y certificados EPG | Huella digital, titular, vencimiento, historial, prueba de descifrado |
| Entrega web | Totemomail WebMail y cuentas de destinatarios | Portal EPG dentro de la plataforma Kiteworks | Registro, inicio de sesión, conservación, apertura de mensaje antiguo |
| Base de datos | Base de datos externa o local según instalación | Base de datos integrada sin acceso directo de administrador | Trabajo de copia de seguridad, procedimiento de restauración, vía de soporte del fabricante |
| Registros | Registros de producto, flujo de correo y sistema | Registros normalizados de plataforma y EPG, exportación Syslog | ID de mensaje/evento correlacionable, base temporal, entrada SIEM |

La comparación se basa en las FAQ de migración del fabricante y en las descripciones de funciones EPG publicadas. Cuando los detalles de Totemomail dependen de la instalación, esto se mantiene deliberadamente como punto de verificación en lugar de presentarlo como una propiedad inventada del producto ([Kiteworks: totemomail-to-EPG migration FAQ](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf), [Kiteworks: Top reasons for EPG](https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-totemomail.svg?v=20260813" title="Interaktive Infografik: Totemomail- und Kiteworks-EPG-Mailflow, Policy, Schlüssel, WebMail, Plattformmigration, Monitoring und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-totemomail.svg?v=20260813">Abrir el gráfico interactivo directamente</a>.
</iframe>

## Flujo de correo SMTP y riesgo de bucles

Una ruta de gateway necesita una responsabilidad inequívoca para cada dirección. En la salida, el sistema de correo no debe entregar opcionalmente de forma directa o a través de EPG si solo la ruta del gateway ejecuta la política requerida. En la entrada debe estar definido si MX, filtros previos o EPG autentican al remitente original y qué salto alcanza el servidor de correo interno. Varios conectores bidireccionales sin una autoridad clara sobre destinatarios y dominios pueden hacer que los mensajes circulen entre sistemas ([RFC 5321: Trace information and loop detection](https://datatracker.ietf.org/doc/html/rfc5321#section-6.3), [Microsoft: Mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud)).

Para cada ruta productiva, los administradores deberían documentar conjuntamente los siguientes valores:

- red de origen o identidad de pares basada en certificados y nombre EHLO esperado;
- dirección local de aceptación, puerto, requisitos de STARTTLS y certificado de cliente;
- responsabilidad autoritativa, de relay o de dominio dividido;
- dirección de la política, clasificación de usuarios y destinatarios;
- siguiente salto, resolución DNS o Smarthost y requisito TLS;
- comportamiento cuando no se pueda acceder a un componente de política, claves, directorio o portal;
- número máximo de saltos, antigüedad de la cola y responsabilidades de reintento y rebote.

La norma SMTP exige añadir líneas Received en cada reenvío y describe la detección de bucles mediante la ruta de trazado. Para diagnóstico y auditoría, el ID de evento del gateway debe poder correlacionarse con Internet `Message-ID`, sobre, ventana temporal y traza posterior. Un asunto visible no es un identificador estable y puede ser modificado por una política o la entrega web ([RFC 5321, sección 4.4](https://datatracker.ietf.org/doc/html/rfc5321#section-4.4), [RFC 5322: Message-ID](https://datatracker.ietf.org/doc/html/rfc5322#section-3.6.4)).

Después de la aceptación SMTP, el motor de políticas decide, basándose en remitente, destinatario, directorio y claves disponibles, qué ruta de protección está permitida y es posible.

## Motor de políticas y decisión de identidad

Kiteworks describe EPG como una capa de decisión automática controlada por políticas. Dependiendo del alcance de funciones licenciado, las reglas pueden basarse en dominios, usuarios, ubicaciones, características de contenido o etiquetas de Microsoft Information Protection. Sin embargo, la automatización no elimina la necesidad de confiar en las entradas: una cabecera de remitente, un desencadenante en el asunto o una etiqueta establecida desde el exterior no debe tener sin verificación el mismo efecto que una identidad interna autenticada ([Kiteworks: EPG capability brief](https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf), [Kiteworks: totemo and Kiteworks better together](https://www.kiteworks.com/sites/default/files/resources/kiteworks-product-brief-top-5-reasons-totemo-kiteworks-better-together.pdf)).

Un modelo de reglas fiable separa cuatro preguntas:

1. **Dirección:** interno a externo, externo a interno, interno a interno o aplicación a externo.
2. **Identidad:** el sobre SMTP, el `From:` visible, la sesión autenticada, el conector, el objeto de directorio y el tenant no son lo mismo.
3. **Acción:** firmar, cifrar, recifrar, entrega por portal, reenviar sin cambios, rechazar temporalmente o rebotar permanentemente.
4. **Ruta de error:** clave ausente, cadena no válida, directorio no accesible, error de política y siguiente salto no accesible requieren resultados separados.

Antes de los cambios debe ejecutarse una pequeña matriz de pruebas con exactamente un remitente y un destinatario por clase de identidad. Para cada fila se registran la regla esperada, el procedimiento, el remitente visible, el canal del destinatario, la característica del registro y el comportamiento ante errores. Solo cuando estos resultados son reproducibles, la política no solo está configurada, sino demostrada.

## Procedimientos de protección y recifrado

[S/MIME](https://datatracker.ietf.org/doc/html/rfc8551) utiliza CMS y certificados X.509. El gateway necesita una clave privada para firma o descifrado y un certificado público adecuado para el cifrado del destinatario. La ruta de certificación, el nombre o dirección de correo, Key Usage, validez e información de revocación forman parte de la comprobación ([RFC 5652](https://datatracker.ietf.org/doc/html/rfc5652), [RFC 5280](https://datatracker.ietf.org/doc/html/rfc5280), [RFC 6960](https://datatracker.ietf.org/doc/html/rfc6960)).

[OpenPGP](https://datatracker.ietf.org/doc/html/rfc9580) utiliza un modelo propio de claves, firmas y paquetes. La huella digital, User ID, capacidades, vencimiento y revocación deben inventariarse independientemente de X.509. Una clave pública aprendida automáticamente solo es tan fiable como el proceso documentado de aprendizaje y confirmación. La documentación del producto menciona S/MIME y OpenPGP juntos, pero no los convierte en modelos de confianza intercambiables ([Kiteworks: Enterprise EPG feature matrix](https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf)).

[SMTP-TLS](/kb/tls#tls-policy-bei-smtp) protege una conexión de transporte individual. Con STARTTLS oportunista, la entrega puede volver a texto claro sin una política adicional. Según el entorno, puede establecerse una identidad de pares vinculante mediante comprobación de certificados, DANE o MTA-STS; aun así, no protege el mensaje frente a los MTA involucrados ([RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207), [RFC 7672](https://datatracker.ietf.org/doc/html/rfc7672), [RFC 8461](https://datatracker.ietf.org/doc/html/rfc8461)).

Durante el recifrado, el gateway descifra un procedimiento entrante y genera otro nuevo para el siguiente tramo. Esto permite la inspección de contenido o el cambio de protocolo, pero crea un momento de texto claro dentro del límite del gateway. El catálogo de funciones EPG describe expresamente escenarios en los que los mensajes protegidos internamente se descifran brevemente en el gateway para inspeccionarlos o convertirlos al estándar del destinatario ([Kiteworks: Internal Encryption and Encryption Gateway](https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf)). DLP, antivirus, archivos temporales, volcados de memoria, registros de depuración y acceso de administrador pasan entonces a formar parte del límite criptográfico.

Si falta una clave de destinatario utilizable directamente, la entrega por portal desplaza el acceso al texto claro a una aplicación web. De este modo, la identidad web, la sesión y el mensaje almacenado pasan a formar parte del modelo operativo.

## WebMail y entrega por portal

Las FAQ oficiales de migración confirman que EPG proporciona una función equivalente a Totemomail WebMail para destinatarios sin cifrado de correo electrónico adecuado. Las guías públicas de operadores para Totemomail muestran el flujo de usuario típico: notificación, contraseña inicial o enlace de registro, cambio de contraseña, inicio de sesión en el portal y selección entre WebMail, S/MIME u OpenPGP ([Kiteworks: Migration FAQ, pregunta 7](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf), [L-Bank: Comunicación segura por correo electrónico](https://www.l-bank.info/allgemein/rechtliches/e-secure/sichere-e-mail-kommunikation.html)).

La entrega por portal es una ruta independiente de almacenamiento e identidad. El correo de notificación no contiene necesariamente el mensaje confidencial; el contenido puede estar en el portal o en un sobre protegido con contraseña. Por tanto, para los administradores son relevantes como mínimo el nombre de host, el certificado TLS, el estado de cuenta, los procedimientos de contraseña y restablecimiento, el período de conservación, las claves de mensaje, el historial de entrega y el proceso de eliminación. La variante elegida debe proceder de la política propia; las guías externas de operadores prueban posibles funciones de Totemomail, pero no valores predeterminados globales.

Una comprobación de estado del portal no debe terminar en HTTP 200. La prueba sintética debe incluir registro o cuenta de prueba existente, inicio de sesión, apertura de un mensaje conocido de antemano, descarga de un adjunto, respuesta y cierre de sesión. Después de una restauración debe comprobarse especialmente si los contenidos antiguos del portal siguen siendo legibles con sus claves históricas.

## Directorio y claves con recuento de licencias

Kiteworks documenta la integración LDAP/AD, los roles y las políticas centrales en el nivel de plataforma. Para un gateway, el directorio puede clasificar remitentes y destinatarios, proporcionar condiciones de grupo y vincular identidades con certificados o claves. Tan pronto como una regla depende de una consulta de directorio, el endpoint [LDAP](/kb/ldap#bind-sasl-und-tls-vertrauensgrenzen), Search Base, filtro, cuenta de enlace, confianza TLS, timeout, caché y failover forman parte de la ruta de mensajes ([Kiteworks: security and integration overview](https://www.kiteworks.com/sites/default/files/resources/accellion-brochure-external-file-sharing-security-and-governance-platform_redirected_20191210.pdf), [RFC 4511](https://datatracker.ietf.org/doc/html/rfc4511), [RFC 4513](https://datatracker.ietf.org/doc/html/rfc4513)).

El inventario de claves no es un subproducto de la importación de usuarios. Deben inventariarse las claves privadas de usuarios y dominios, las claves públicas de destinatarios, certificados CA e intermedios, el estado de confianza, información de revocación, conexiones de keyserver o MPKI y la asignación a mensajes históricos. Una limpieza del directorio nunca debe eliminar sin verificación una clave privada necesaria para mensajes antiguos.

Un documento publicado de empaquetado de Kiteworks definía los puestos EPG de modo que un usuario interno consumía un puesto al enviar o recibir un mensaje cifrado o al enviar un mensaje firmado; los usuarios internos que pasaban sin cifrar ni firmar se trataban allí de forma diferente. El documento es una descripción contractual fechada, no una garantía de licencia atemporal ([Kiteworks: Enterprise 8.0 licensing and packaging](https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf)). Por ello, para la operación deben gestionarse por separado tres valores: identidades técnicamente conocidas, identidades realmente activas criptográficamente y la métrica licenciada contractualmente del pedido propio.

Por tanto, al cambiar a EPG no solo deben transferirse o reconstruirse deliberadamente las rutas SMTP, sino también reglas, identidades, claves, estados de portal y evidencias.

## Migración de Totemomail a EPG

Kiteworks no describe la migración como un simple cambio de nombre in situ. Una transferencia automatizada copia información de la base de datos existente de Totemomail a la base de datos EPG. La antigua base de datos externa debe desmantelarse después de retirar el sistema Totemomail porque ya no está sincronizada. EPG contiene su propia base de datos, sistema operativo y servicios web en la Hardened Virtual Appliance ([Kiteworks: Migration FAQ](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf)).

El alcance completo de una migración incluye como mínimo:

- dominios, rutas, identidades de relay y conector, y políticas TLS;
- reglas activas, prioridades, desencadenantes, excepciones y rutas de error;
- claves S/MIME, OpenPGP, de dominio, TLS y CA, incluido el historial;
- cuentas WebMail, mensajes de portal, conservación, plantillas y nombre público;
- integraciones de directorio, SSO, SIEM, Syslog, DLP, archivado y aplicaciones;
- roles, administradores y accesos de auditoría y soporte;
- métrica de licencias, inventario de usuarios activos y límites contractuales;
- copia de seguridad, restauración, HA, DNS, monitorización y runbooks.

La prueba de aceptación debe comprobar datos históricos además de correos nuevos. Para cada tipo de cifrado antiguo, se abre o descifra un mensaje creado antes de la migración. Para cada dirección, se envía un nuevo correo de prueba con y sin clave de destinatario disponible. Además, se comprueban casos negativos como certificado no válido, siguiente salto no accesible y directorio no disponible. Solo entonces puede eliminarse el gateway antiguo de DNS, conectores, listas de permitidos y monitorización.

## Alta disponibilidad con copia de seguridad y recuperación

Las FAQ de migración de EPG mencionan clúster integrado, copia de seguridad/restauración, recuperación ante desastres, alta disponibilidad y distribución como características de la Hardened Virtual Appliance. Sin embargo, no indican que cada función EPG individual reciba automáticamente el mismo RPO y RTO. La cola de correo, la base de datos, las claves, el contenido del portal, el registro de auditoría, la configuración y el almacenamiento externo de objetos o archivos deben asignarse por separado según la arquitectura ([Kiteworks: Migration FAQ – Hardened Virtual Appliance](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf)).

| Objeto de recuperación | Evidencia mínima | Brecha típica |
|---|---|---|
| Configuración del gateway | Estado objetivo exportado o respaldado y restauración en sistema de sustitución | La configuración existe, faltan secretos o certificados |
| Claves privadas | Lista de huellas digitales antes y después de la restauración; descifrado histórico | Existen claves actuales, se perdió la rotación antigua |
| Política | Estado activado más matriz de pruebas positiva y negativa | La exportación no corresponde a la regla realmente activa |
| Estado de correo y portal | Objetos de prueba de cola y WebMail conocidos de antemano después de la restauración | La plataforma se inicia, el mensaje antiguo es ilegible |
| Clúster | Failover con prueba de SMTP, HTTPS, claves y registros | La VIP cambia, pero no el estado backend o la IP saliente |
| Auditoría y Syslog | Copia externa de eventos con correlación temporal y de mensajes | La restauración local elimina el historial de investigación |

En Totemomail heredado, el procedimiento de copia de seguridad depende de la versión y la instalación. La afirmación de EPG sobre copia de seguridad integrada no debe trasladarse retrospectivamente a una antigua base de datos externa o a un almacenamiento WebMail independiente. La vía de soporte del fabricante, la documentación instalada y una prueba de restauración aislada son las fuentes primarias en este caso.

## Monitorización y operación de seguridad

Kiteworks confirma para EPG la exportación a uno o varios destinos Syslog y la integración con Splunk. La documentación de la plataforma describe registros y metadatos normalizados sobre correo electrónico y otros canales de comunicación. Para los administradores de mensajería, la correlación funcional sigue siendo decisiva: la auditoría de plataforma, el evento SMTP, la decisión de política, la operación de claves, el acceso al portal y la traza posterior del mensaje deben reunirse mediante tiempo, identidad e identificador de mensaje ([Kiteworks: Migration FAQ – syslog](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf), [Kiteworks: EPG logging and audit](https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf)).

Un panel de administración debería mostrar como mínimo aceptaciones y rechazos, antigüedad de la cola por siguiente salto, errores de política, número y tipo de acciones criptográficas, claves ausentes o no válidas, vencimiento de certificados, latencia LDAP, errores de inicio de sesión en el portal, estado de disco y base de datos, replicación de clúster, antigüedad de copias de seguridad y pérdida de Syslog. Una cola vacía solo es un resultado positivo si, al mismo tiempo, los mensajes se aceptan y se entregan correctamente al siguiente salto.

Syslog mediante UDP no ofrece entrega garantizada. Para datos de auditoría que deban protegerse, deben planificarse transporte TLS, almacenamiento en búfer, autenticación del destino, hora correcta y una alarma de fallo ([RFC 5424](https://datatracker.ietf.org/doc/html/rfc5424), [RFC 5425](https://datatracker.ietf.org/doc/html/rfc5425)). Los registros pueden contener remitente, destinatario, asunto, información de certificados o actividades de portal; por ello, la minimización de datos y el control de acceso forman parte de la política de registro.

Los avisos de seguridad históricos muestran que la interfaz web también forma parte de la superficie de ataque. Las entradas NVD documentan, entre otras, vulnerabilidades anteriores de CSRF, acceso a objetos y XSS en componentes web de Totemomail. De ello no se deriva ninguna afirmación sobre un estado de parche arbitrario sin comprobar, pero sí la obligación permanente de incluir avisos del fabricante, exposición, estado de soporte y pruebas de portal web en la gestión de parches ([NVD: CVE-2018-6563](https://nvd.nist.gov/vuln/detail/CVE-2018-6563), [NVD: CVE-2020-7918](https://nvd.nist.gov/vuln/detail/CVE-2020-7918), [NVD: CVE-2024-28063](https://nvd.nist.gov/vuln/detail/CVE-2024-28063)).

El diagnóstico sigue un mensaje de prueba desde la aceptación, pasando por la decisión de política y claves, hasta la salida SMTP o el acceso al portal.

## Herramientas de diagnóstico

El diagnóstico comienza en la ruta de correo publicada y después sigue los eventos EPG hasta la entrega o WebMail. La migración de plataforma y el estado de licencia solo se comprueban cuando se conoce el paso de procesamiento afectado.

### DNS y rutas de correo públicas

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Totemomail- und EPG-DNS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName -Type MX example.ch
Resolve-DnsName epg.example.ch -Type A,AAAA
Resolve-DnsName _25._tcp.epg.example.ch -Type TLSA
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig +short MX example.ch
dig +short A epg.example.ch
dig +short AAAA epg.example.ch
dig +short TLSA _25._tcp.epg.example.ch
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) y [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) muestran el MX público, la dirección del gateway y, en su caso, DANE-TLSA. Las vistas de resolvedores internos y externos deben comprobarse por separado. En Microsoft 365, el MX describe solo el primer salto de Internet; los conectores y reglas de transporte determinan la ruta posterior.

### Accesibilidad TCP, SMTP y TLS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Totemomail- und EPG-TLS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Test-NetConnection epg.example.ch -Port 25 -InformationLevel Detailed
Test-NetConnection epg.example.ch -Port 443 -InformationLevel Detailed
curl.exe --verbose --ssl-reqd smtp://epg.example.ch:25
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nc -vz epg.example.ch 25
nc -vz epg.example.ch 443
openssl s_client -starttls smtp -connect epg.example.ch:25 \
  -servername epg.example.ch -showcerts
```

  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) y [`nc`](https://man.openbsd.org/nc) solo demuestran la ruta TCP. [`curl.exe`](https://curl.se/docs/manpage.html) de Windows solicita SMTP-STARTTLS; [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) muestra capacidades, handshake, SNI y cadena de certificados. Solo una transacción SMTP completa con respuesta DATA positiva demuestra la aceptación.

### Mensaje SMTP de prueba autorizado

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Totemomail- und EPG-SMTP-Test">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
curl.exe --verbose --ssl-reqd --url smtp://epg.example.ch:25 `
  --mail-from test-sender@example.ch `
  --mail-rcpt test-recipient@example.net `
  --upload-file .\epg-test.eml
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
swaks --server epg.example.ch --port 25 --tls \
  --from test-sender@example.ch --to test-recipient@example.net \
  --data epg-test.eml
```

  </div>
</div>

[`curl`](https://curl.se/docs/manpage.html) y [`swaks`](https://jetmore.org/john/code/swaks/) envían aquí un mensaje de prueba controlado. El remitente, el destinatario y el destino deben estar expresamente autorizados para ello. Deben registrarse la respuesta SMTP final, el ID del gateway, la política activada, la clave utilizada o ruta de portal, el siguiente salto y la llegada efectiva. Un `250` en `RCPT TO` todavía no constituye la aceptación del contenido del mensaje.

### Ruta WebMail o de portal

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Totemomail- und EPG-Portaldiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Invoke-WebRequest -Method Head -Uri https://securemail.example.ch/
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
curl --head --verbose https://securemail.example.ch/
openssl s_client -connect securemail.example.ch:443 \
  -servername securemail.example.ch -showcerts
```

  </div>
</div>

[`Invoke-WebRequest`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest), [`curl`](https://curl.se/docs/manpage.html) y [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) comprueban la interfaz HTTP y TLS. Un código de estado correcto no demuestra ni el inicio de sesión ni el acceso a claves. Una prueba de navegador sintética debe abarcar inicio de sesión, apertura de un mensaje conocido, adjunto y respuesta.

### Ruta de paquetes en el punto de medición correcto

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Totemomail- und EPG-Paketerfassung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
pktmon filter remove
pktmon filter add EPG-SMTP -p 25
pktmon start --capture --pkt-size 0 --file-name epg.etl
pktmon stop
pktmon pcapng epg.etl -o epg.pcapng
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
tcpdump -ni any -s 0 -w epg.pcap 'tcp port 25 or tcp port 443'
```

  </div>
</div>

[`pktmon`](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon) y [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) solo ven tráfico en el punto de medición elegido. Un cliente administrativo no observa automáticamente la ruta entre Exchange Online, el equilibrador de carga y EPG. Para el análisis son adecuados un host autorizado, un puerto espejo o Cloud Flow Logs; SMTP antes de STARTTLS y los metadatos web deben conservarse con la protección correspondiente.

## Historia técnica

Totemo se fundó en 2001 en Suiza y desarrolló soluciones para cifrado de correo electrónico, Managed File Transfer y comunicación móvil. Las validaciones NIST de 2012 y 2019 documentan el desarrollo del Totemo Cryptographic Module basado en Java y de Totemo Security Platform en sistemas operativos de appliance propios ([Entrust Partner Directory: Totemo](https://www.entrust.com/partners/directory/totemo), [NIST: TCM Security Policy 2012](https://csrc.nist.gov/csrc/media/projects/cryptographic-module-validation-program/documents/security-policies/140sp1884.pdf), [NIST CMVP Certificate 3500](https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/3500)).

Kiteworks adquirió totemo en enero de 2022. La documentación del fabricante describe la integración del código Totemomail en la plataforma Kiteworks y el cambio de nombre a Email Protection Gateway. EPG combinó las funciones existentes de cifrado de correo electrónico con la Hardened Virtual Appliance, roles comunes, registros normalizados y otros canales de comunicación de Kiteworks ([Kiteworks: acquisition and EPG integration](https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf), [Kiteworks: Migration FAQ](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf), [Kiteworks: January 2022 acquisition](https://www.kiteworks.com/sites/default/files/resources/on-the-hunt-kiteworks-secure-content-strategy.pdf)).

La transición es técnicamente relevante: el producto antiguo podía funcionar con una base de datos externa; EPG asume sus datos en una base de datos integrada y desplaza el mantenimiento del sistema operativo y de la base de datos al límite de la appliance. Por ello, los runbooks, la monitorización y la recuperación no deben limitarse a sustituir términos de producto, sino que deben reasignar superficies de control y datos y configuraciones almacenados.

## Fuentes

- [Kiteworks – FAQ for totemomail Customers Updating to EPG](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf)
- [Kiteworks – Top 6 Reasons to Add Email Protection Gateway](https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf)
- [Kiteworks – Enterprise 8.0 Licensing and Packaging](https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf)
- [Microsoft Marketplace – totemomail Encryption Gateway](https://marketplace.microsoft.com/en/marketplace/apps/totemo.totemo-azr-tm7?tab=overview)
- [IETF RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321)
- [IETF RFC 8551 – S/MIME 4.0](https://datatracker.ietf.org/doc/html/rfc8551)
- [IETF RFC 9580 – OpenPGP](https://datatracker.ietf.org/doc/html/rfc9580)
- [IETF RFC 3207 – SMTP STARTTLS](https://datatracker.ietf.org/doc/html/rfc3207)
- [Microsoft – Configure mail flow using connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow)
- [Microsoft – Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)
- [NIST CMVP – Certificate 3500, Totemo Cryptographic Module](https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/3500)
- [NIST – Totemo Cryptographic Module Security Policy 2019](https://csrc.nist.gov/CSRC/media/projects/cryptographic-module-validation-program/documents/security-policies/140sp3500.pdf)
- [Microsoft – Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud)
- [RFC 5322: Message-ID](https://datatracker.ietf.org/doc/html/rfc5322)
- [Kiteworks – Top 5 Reasons totemo and Kiteworks Are Better Together](https://www.kiteworks.com/sites/default/files/resources/kiteworks-product-brief-top-5-reasons-totemo-kiteworks-better-together.pdf)
- [IETF RFC 5652 – Cryptographic Message Syntax](https://datatracker.ietf.org/doc/html/rfc5652)
- [IETF RFC 5280 – X.509 PKI Certificate and CRL Profile](https://datatracker.ietf.org/doc/html/rfc5280)
- [IETF RFC 6960 – Online Certificate Status Protocol](https://datatracker.ietf.org/doc/html/rfc6960)
- [IETF RFC 7672 – SMTP Security via DANE](https://datatracker.ietf.org/doc/html/rfc7672)
- [IETF RFC 8461 – MTA Strict Transport Security](https://datatracker.ietf.org/doc/html/rfc8461)
- [L-Bank – Comunicación segura por correo electrónico con Totemomail](https://www.l-bank.info/allgemein/rechtliches/e-secure/sichere-e-mail-kommunikation.html)
- [Kiteworks – Security and Integration Overview](https://www.kiteworks.com/sites/default/files/resources/accellion-brochure-external-file-sharing-security-and-governance-platform_redirected_20191210.pdf)
- [IETF RFC 4511 – LDAP Protocol](https://datatracker.ietf.org/doc/html/rfc4511)
- [IETF RFC 4513 – LDAP Authentication Methods and Security Mechanisms](https://datatracker.ietf.org/doc/html/rfc4513)
- [IETF RFC 5424 – Syslog Protocol](https://datatracker.ietf.org/doc/html/rfc5424)
- [IETF RFC 5425 – TLS Transport Mapping for Syslog](https://datatracker.ietf.org/doc/html/rfc5425)
- [NVD – CVE-2018-6563](https://nvd.nist.gov/vuln/detail/CVE-2018-6563)
- [NVD – CVE-2020-7918](https://nvd.nist.gov/vuln/detail/CVE-2020-7918)
- [NVD – CVE-2024-28063](https://nvd.nist.gov/vuln/detail/CVE-2024-28063)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [ISC BIND 9 – dig manpage](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc manpage](https://man.openbsd.org/nc)
- [curl – command line manpage](https://curl.se/docs/manpage.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [swaks – SMTP test tool](https://jetmore.org/john/code/swaks/)
- [Microsoft Learn – Invoke-WebRequest](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest)
- [Microsoft Learn – pktmon](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon)
- [tcpdump – manual page](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [Entrust Partner Directory – Totemo](https://www.entrust.com/partners/directory/totemo)
- [NIST – Totemo Cryptographic Module Security Policy 2012](https://csrc.nist.gov/csrc/media/projects/cryptographic-module-validation-program/documents/security-policies/140sp1884.pdf)
- [Kiteworks – The Deal: On the Hunt](https://www.kiteworks.com/sites/default/files/resources/on-the-hunt-kiteworks-secure-content-strategy.pdf)
