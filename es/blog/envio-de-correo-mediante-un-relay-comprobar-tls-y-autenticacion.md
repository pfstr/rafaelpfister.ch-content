---
title: "Envío de correo a través de un relay: comprobar TLS y autenticación"
navTitle: "Relay: comprobar TLS"
description: "Una guía breve para Application Managers cuyas aplicaciones envían correos a través de un relay: qué tres configuraciones de la aplicación importan (puerto, modo TLS e inicio de sesión), cómo se llaman las opciones en entornos habituales y cómo un único correo de prueba demuestra mediante la cabecera Received que la conexión está realmente cifrada y autenticada."
date: "2026-08-28"
kategorie: "SMTP y flujo de correo"
timeToRead: "5 min de lectura"
themen:
  - smtp-mailflow
protokolle:
  - "smtp"
  - "tls"
  - "troubleshooting"
slug: "envio-de-correo-mediante-un-relay-comprobar-tls-y-autenticacion"
translationId: "article-734e79c4a87105e3"
translationOf: mail-relay-tls-authentisierung-pruefen
translationSourceHash: a1f56d0ba84377020bb56c927ee5d8789198061a0f941b463e3fd0e929262082
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T10:04:44.537Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/envio-de-correo-mediante-un-relay-comprobar-tls-y-autenticacion
---

# Envío de correo a través de un relay: comprobar TLS y autenticación

Muchas aplicaciones no envían correos directamente a Internet, sino que los entregan a un relay interno: el ERP sus confirmaciones de pedido, el sistema de monitorización sus alertas y el sistema de tickets sus notificaciones. El equipo de correo opera el relay; del lado de la aplicación, el responsable es el Application Manager. Por ello, durante una auditoría o un análisis de necesidades de protección, la pregunta llega hasta él: ¿se conecta la aplicación al relay de forma cifrada y se autentica correctamente?

La respuesta se encuentra en dos lugares, para los que no se necesita ninguna herramienta de correo ni acceso al relay: en la configuración SMTP de la propia aplicación y en la cabecera de un único correo de prueba. El equipo de correo es responsable de lo que ofrece el propio relay y de cómo cifra los correos hasta el destinatario; del lado de la aplicación basta con demostrar el propio tramo.

## Dónde se encuentran las configuraciones

La configuración SMTP se encuentra, según la aplicación, en uno de tres lugares: en la interfaz de administración (normalmente en «Correo electrónico», «Notificaciones», «SMTP» o «Servidor saliente»), en un archivo de configuración o en variables de entorno del despliegue (típicamente `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER` y variantes). Siempre se buscan los mismos datos: nombre del servidor, puerto, una opción de cifrado y las credenciales.

## Las tres configuraciones que importan

**Primero, el puerto y el modo TLS.** Ambos deben coincidir, ya que detrás de los valores seleccionables hay dos procedimientos distintos: con STARTTLS la conexión comienza en texto claro y luego cambia a TLS; con TLS implícito (normalmente llamado «SSL/TLS» o «SSL» en las interfaces) está cifrada desde el principio.

| Puerto | Configuración TLS en la aplicación | Evaluación |
|---|---|---|
| 587 | STARTTLS | Estado deseado para la entrega desde aplicaciones |
| 465 | SSL/TLS (implícito) | también correcto |
| 25 | ninguno o STARTTLS | habitual en relays con autorización por IP; active igualmente la opción TLS si el relay ofrece STARTTLS |
| cualquiera | «Ninguno» / «None» | Hallazgo: el envío se realiza en texto claro |
| cualquiera | «TLS si está disponible» / oportunista | Hallazgo: ante un problema, vuelve silenciosamente al texto claro; cambiar a TLS obligatorio |

Una combinación incorrecta (por ejemplo, «SSL/TLS» en el puerto 587) provoca interrupciones de conexión, no texto claro inadvertido. Las configuraciones de riesgo son las dos últimas filas de la tabla, porque en ellas la aplicación envía sin cifrar y sin mostrar un mensaje de error.

**Segundo, la validación del certificado.** Muchas aplicaciones ofrecen una opción como «No comprobar certificado», «Allow insecure» o `verify=false`, que suele activarse en proyectos de implantación porque el relay utiliza un certificado interno. La conexión sigue cifrada, pero la aplicación acepta cualquier interlocutor. Si la opción está activada, debe incluirse como hallazgo en el informe; la solución correcta es confiar en la CA interna en lugar de desactivar la validación.

**Tercero, el inicio de sesión.** Los relays admiten dos modelos: SMTP AUTH con nombre de usuario y contraseña, o autorización por IP sin cuenta. La variante aplicable figura en la autorización del relay del equipo de correo. Para SMTP AUTH, hay tres puntos que incluir en la lista de comprobación: el inicio de sesión se realiza con una cuenta de servicio dedicada de la aplicación (no con una cuenta personal que se desactivará en la próxima baja), la contraseña se almacena como secreto en lugar de en texto claro en un archivo de configuración, y la opción TLS está activa, ya que los métodos habituales PLAIN y LOGIN transmiten las credenciales en texto claro de otro modo.

## Cómo se llaman las configuraciones en entornos habituales

| Entorno | Cifrado | Inicio de sesión |
|---|---|---|
| Interfaces de administración (ERP, monitorización, appliances) | Lista desplegable «Cifrado»: None / STARTTLS / SSL-TLS | Campos de nombre de usuario/contraseña; vacíos = sin inicio de sesión |
| Java (Jakarta Mail, Spring) | `mail.smtp.starttls.enable=true` más `mail.smtp.starttls.required=true`; para el puerto 465, `mail.smtp.ssl.enable=true` | `mail.smtp.auth=true` |
| .NET | `SmtpClient.EnableSsl=true` (activa STARTTLS); MailKit: `SecureSocketOptions.StartTls` | `Credentials` o `Authenticate()` |
| PHP (PHPMailer) | `SMTPSecure='tls'` para 587, `'ssl'` para 465 | `SMTPAuth=true` |
| Python (smtplib) | `starttls()` después de establecer la conexión o `SMTP_SSL` para 465 | `login()` |
| Node.js (Nodemailer) | Puerto 465: `secure:true`; puerto 587: `secure:false` más `requireTLS:true` | `auth: {user, pass}` |

Dos puntos de esta tabla son, por experiencia, los hallazgos más frecuentes: en Java, `starttls.enable` por sí solo activa únicamente TLS oportunista; solo `starttls.required` impide la vuelta al texto claro. En Nodemailer, `secure:false` no significa «sin cifrar», sino «sin TLS implícito»; sin `requireTLS:true`, STARTTLS también sigue siendo oportunista.

## Comprobación: un correo de prueba y su cabecera Received

La configuración indica el estado deseado, pero no demuestra qué sucede en la conexión. La prueba está en la cabecera Received que el relay añade al recibir cada correo. Basta con enviar un correo de prueba desde la aplicación al propio buzón; allí, mostrar la cabecera del mensaje (Outlook: Archivo, Propiedades, encabezados de Internet; Gmail: Mostrar original) y leer la línea Received inferior, ya que las cabeceras crecen de abajo hacia arriba:

```text
Received: from app01.example.com (app01.example.com [10.1.2.3])
        by relay.example.com (Postfix) with ESMTPSA id 4XyZk12Fzq
        (version=TLSv1.3 cipher=TLS_AES_256_GCM_SHA384);
        Thu, 28 Aug 2026 09:15:04 +0200
```

La palabra clave después de `with` es el resumen del resultado de la comprobación. Las identificaciones están estandarizadas (registro IANA «Mail Transmission Types»):

| Identificación | Significado | Evaluación |
|---|---|---|
| `SMTP` / `ESMTP` | sin cifrar, sin inicio de sesión | Se requiere actuación si se exige TLS |
| `ESMTPS` | TLS, sin inicio de sesión | correcto con autorización por IP |
| `ESMTPA` | autenticado, pero sin TLS | crítico: las credenciales se transmitieron en texto claro |
| `ESMTPSA` | TLS y autenticado | estado deseado con SMTP AUTH |

Postfix y Exchange añaden entre paréntesis la versión de TLS y el cifrado, lo que también permite identificar versiones de protocolo obsoletas. Para evaluar cabeceras largas con varias estaciones, el [Mail-Header-Analyzer](https://rafaelpfister.ch/tools/header-analyzer) de este sitio web le ahorra el trabajo manual; funciona íntegramente de forma local en el navegador y la cabecera no sale de su equipo.

Si la cabecera no queda clara o un balanceador de carga anterior modifica la conexión, es el momento de consultar al equipo de correo: el registro del relay documenta para cada entrega si se negoció TLS y con qué cuenta se autenticó la aplicación.

## Lista de comprobación breve para el informe de auditoría

1. Se ha localizado y documentado la configuración SMTP de la aplicación (interfaz, archivo de configuración o variables de entorno).
2. El puerto y el modo TLS coinciden (587/STARTTLS o 465/SSL-TLS); no hay ninguna configuración «Ninguno» ni «TLS si está disponible».
3. La validación del certificado está activa; un «No comprobar certificado» activado se ha registrado como hallazgo.
4. Se ha aclarado el modelo de inicio de sesión: SMTP AUTH con cuenta de servicio y almacenamiento de secretos, o autorización por IP según la autorización del relay.
5. La cabecera Received del correo de prueba muestra `ESMTPSA` (con cuenta) o `ESMTPS` (con autorización por IP); `ESMTPA` y `ESMTP` son hallazgos.
6. Si se requiere cifrado hasta el destinatario: plantearlo como requisito al equipo de correo, ya que el tramo a partir del relay queda fuera de la aplicación.

## Historial de cambios

- **25 de octubre de 2026:** Se actualizaron los enlaces de las fuentes a las nuevas direcciones de IANA; la fuente de Jakarta Mail ahora remite a la documentación de Eclipse Angus Mail.

## Fuentes

1.  [RFC 3207: SMTP Service Extension for Secure SMTP over Transport Layer Security](https://www.rfc-editor.org/rfc/rfc3207): define STARTTLS y el cambio de la conexión en texto claro a TLS.

2.  [RFC 4954: SMTP Service Extension for Authentication](https://www.rfc-editor.org/rfc/rfc4954): define SMTP AUTH y procedimientos como PLAIN y LOGIN.

3.  [RFC 8314: Cleartext Considered Obsolete](https://www.rfc-editor.org/rfc/rfc8314): recomienda TLS implícito en el puerto 465 para la entrega desde clientes.

4.  [IANA: Mail Transmission Types](https://www.iana.org/assignments/smtp/smtp.xhtml#smtp-mail-transmission-types): registro de las identificaciones `with` de la cabecera Received (ESMTPS, ESMTPA, ESMTPSA).

5.  [Eclipse Angus Mail: Package org.eclipse.angus.mail.smtp](https://eclipse-ee4j.github.io/angus-mail/docs/api/org.eclipse.angus.mail/org/eclipse/angus/mail/smtp/package-summary.html): proveedor SMTP de la implementación Jakarta Mail; documenta las propiedades mail.smtp.starttls.enable, starttls.required y ssl.enable.
