---
title: "Gestionar un boletín propio con Cloudflare Workers y D1"
navTitle: "Boletín en Workers"
description: "La plantilla abierta proporciona suscripción, cancelación, cola y base de datos en su propia cuenta de Cloudflare. Un botón de despliegue configura Worker, D1 y CI sin necesidad de servidor local."
date: "2026-07-22"
kategorie: "Cloudflare Workers"
timeToRead: "8 min de lectura"
themen:
  - cloudflare-workers
slug: "gestionar-tu-propia-newsletter-con-cloudflare-workers-y-d1"
translationOf: "serverloser-newsletter-cloudflare-workers-d1"
translationId: article-4e7139acdb90923b
translationReview: automatic
translationSourceHash: ad5b78d6330d06a17259e464c0fb8bb9713b3fdf5cd6c77ac1d300d9fea2a48e
translatedAt: 2026-09-04T08:39:14.413Z
url: https://rafaelpfister.ch/es/blog/gestionar-tu-propia-newsletter-con-cloudflare-workers-y-d1
translationModel: gpt-5.6-terra
---

# Gestionar un boletín propio con Cloudflare Workers y D1

Con un servicio de boletines alojado, la lista de destinatarios queda en manos del proveedor y los costes suelen aumentar con el número de suscriptores. Un servidor propio ofrece más control, pero implica trabajo continuo: actualizaciones, supervisión, copias de seguridad y operación de un sistema que quizá solo envía una vez por semana.

Para este caso de uso ligero bastan endpoints HTTP, una pequeña base de datos y una tarea de envío programada. Cloudflare Workers y D1 proporcionan precisamente estos componentes. Mi plantilla abierta los configura en su propia cuenta mediante un **botón Deploy to Cloudflare**. No se necesita una línea de comandos local ni un servidor que requiera mantenimiento permanente. El código fuente con licencia MIT está en [GitHub](https://github.com/pfstr/newsletter-template).

[![Deploy to Cloudflare](../images/serverloser-newsletter-cloudflare-workers-d1/deploy-to-cloudflare.svg)](https://deploy.workers.cloudflare.com/?url=https://github.com/pfstr/newsletter-template)

![El formulario de suscripción alojado de la plantilla](../images/serverloser-newsletter-cloudflare-workers-d1/newsletter-template-signup.png)

## Qué puede hacer la plantilla

- **Suscripción**: una página de suscripción alojada, un formulario integrable en su propio sitio web y un endpoint JSON
- **Cancelación con un clic**: conforme a RFC 8058, con un token individual por suscriptor
- **Datos obligatorios integrados**: cada correo electrónico recibe automáticamente un pie con enlace de cancelación y dirección postal; se almacenan las fechas y horas de consentimiento y cancelación
- **Envío**: en una página protegida se pueden introducir el asunto y el HTML, enviar un correo de prueba y poner la campaña en cola; una tarea en segundo plano envía por lotes y repite los intentos fallidos
- **Datos propios**: los suscriptores se almacenan en una base de datos D1 de su cuenta y se pueden exportar en cualquier momento
- **Opcional, desactivado por defecto**: doble opt-in, protección contra bots mediante Turnstile y envío automático de nuevos artículos del blog desde el feed RSS

## Arquitectura: un Worker, una base de datos

Todo el sistema consiste en un único Cloudflare Worker con dos handlers: `fetch` para HTTP (enrutado con Hono) y `scheduled` para el activador Cron, además de una base de datos D1. No hay un segundo servicio, ni un bróker de cola independiente, ni un backend de administración propio; incluso la cola de envío es solo una tabla D1.

| Ruta | Función |
| --- | --- |
| `GET /` | Página de suscripción alojada |
| `GET /embed` | Formulario transparente para integrar mediante iframe |
| `POST /api/subscribe` | Suscripción (CORS abierto para el propio sitio web) |
| `GET /confirm` | Enlace de confirmación para el doble opt-in |
| `GET/POST /unsubscribe` | Cancelación: página de confirmación mediante GET, ejecución mediante POST (un clic conforme a RFC 8058) |
| `GET /admin` | Página de envío (formulario) |
| `POST /api/send` | Poner la campaña en cola, protegida mediante token de administrador |

El modelo de datos incluye cuatro tablas: `subscribers` (correo electrónico como clave primaria, nombre, estado, tokens de cancelación y confirmación, una columna JSON para campos adicionales definidos por el usuario, así como marcas de tiempo de confirmación y cancelación), `campaigns` con asunto, contenido y contadores por envío, `outbox` como cola de envío (una fila por destinatario) y `sent_posts` para la deduplicación de envíos RSS.

## Despliegue sin línea de comandos

Más interesante que el código es el camino hacia un sistema operativo. El botón Deploy to Cloudflare lee la configuración de Wrangler del repositorio y realiza toda la configuración: clona el repositorio en su propia cuenta de GitHub, aprovisiona la base de datos D1, ejecuta las migraciones de esquema y configura CI, de modo que cada push se despliega automáticamente. Desde julio de 2025, el flujo de despliegue también solicita variables de entorno y secretos directamente en el formulario: en el caso de esta plantilla, la contraseña de administrador (`ADMIN_TOKEN`), el nombre y la dirección del remitente, el interruptor de doble opt-in y el tamaño del lote de envío (`SEND_BATCH`).

El resultado tras un clic y un formulario: la página de suscripción está activa en `https://<worker-name>.workers.dev` y recopila suscriptores. No se abre un terminal en ningún momento.

## Recopilar suscriptores

Para integrarlo en su propio sitio web hay tres opciones, en orden creciente de profundidad de integración. La más sencilla: compartir el enlace a la página de suscripción alojada. La más práctica para creadores de sitios (WordPress, Webflow, Squarespace, Framer): un iframe de una línea en cualquier bloque de inserción HTML.

```html
<iframe
  src="https://<worker-name>.workers.dev/embed"
  style="width:100%;max-width:420px;height:90px;border:0"
></iframe>
```

Quien quiera el formulario con su propio diseño puede publicar directamente en el endpoint:

```html
<form
  onsubmit="event.preventDefault();
  fetch('https://<worker-name>.workers.dev/api/subscribe', {
    method:'POST', headers:{'Content-Type':'application/json'},
    body: JSON.stringify({ email: this.email.value })
  }).then(()=>this.reset());"
>
  <input name="email" type="email" placeholder="you@example.com" required />
  <button>Abonnieren</button>
</form>
```

El formulario recopila por defecto el correo electrónico y, opcionalmente, el nombre. Los campos adicionales (empresa, país, …) se definen en un único archivo (`src/fields.ts`); aparecen automáticamente en ambos formularios y se guardan como JSON en la base de datos.

## Envío: proveedor propio en lugar de un vendor integrado

Para el envío de correo electrónico, la plantilla toma una decisión consciente: es **agnóstica respecto al proveedor**. El archivo `src/email.ts` contiene un único adaptador `sendEmail()` con un ejemplo comentado para una API HTTP genérica. Qué servicio de envío conecte allí queda a su elección. No hay ningún proveedor integrado de forma fija ni se presupone el registro en un servicio concreto. La recopilación de suscriptores ya funciona por completo sin configuración de envío; el envío se habilita en cuanto se implementa el adaptador y se configura el secreto del proveedor. Si el proveedor ofrece además un endpoint por lotes (una llamada API, muchos correos electrónicos), se puede añadir en el mismo archivo un adaptador opcional `sendEmailBatch()`; también hay un ejemplo comentado para ello.

El envío se gestiona desde la página `/admin`: introduzca el asunto y el HTML del correo, envíe una prueba a su propia dirección y, a continuación, ponga la campaña en cola para todos los suscriptores. En los correos están disponibles las etiquetas de combinación `{{unsubscribe_url}}`, `{{email}}` y `{{name}}`.

El envío propiamente dicho ocurre en segundo plano, según el patrón Transactional Outbox: `POST /api/send` escribe la campaña y una fila por destinatario en la base de datos y responde de inmediato. Después, una tarea Cron cada minuto entrega `SEND_BATCH` correos electrónicos por ejecución, 40 de forma predeterminada: elegido así para que cada ejecución se mantenga dentro de los límites de subrequests del plan Workers Free. Las filas se reclaman de forma atómica, por lo que las ejecuciones solapadas nunca pueden enviar dos veces; las entregas fallidas se repiten hasta tres veces y las ejecuciones interrumpidas se retoman tras diez minutos. Y quien se da de baja mientras su correo aún está en la cola ya no lo recibe: el opt-out también cancela los mensajes ya encolados.

## La cancelación y las pruebas forman parte del núcleo

Quien envía un boletín está sujeto a la legislación antispam y de protección de datos: la ley estadounidense CAN-SPAM Act, el RGPD y ePrivacy en la UE, y la UWG en Suiza. Una parte esencial de lo que se paga en los servicios de boletines es precisamente el cumplimiento de estas obligaciones. La plantilla asume su parte mecánica:

- **Pie obligatorio**: cada correo de campaña recibe automáticamente un pie con un enlace de cancelación funcional y la dirección postal del remitente (`SENDER_ADDRESS`); CAN-SPAM exige una dirección física en los correos comerciales. La página de envío advierte mientras falte la dirección.
- **Cabecera List-Unsubscribe conforme a RFC 8058** en cada envío: el botón nativo de cancelación en Gmail y Outlook, que Gmail y Yahoo exigen a los remitentes masivos desde 2024. La aplicación compone las cabeceras; el adaptador del proveedor propio solo debe reenviarlas.
- **Cancelación segura frente a escáneres**: el enlace de cancelación lleva a una página de confirmación con un único botón. Así, los escáneres de correo corporativo que consultan previamente cada enlace de un correo no pueden dar de baja a nadie por error; los clientes de correo utilizan directamente el POST de un clic.
- **Minimización de datos y prueba**: un opt-out tiene efecto inmediato, elimina el nombre y los campos adicionales y queda registrado con marca de tiempo, al igual que la suscripción y la confirmación de doble opt-in. Así se puede acreditar posteriormente el consentimiento (principio de responsabilidad del RGPD).
- **Enlace de privacidad**: con `PRIVACY_URL` configurado, aparece bajo el formulario de suscripción un enlace a su propia política de privacidad.

El operador sigue siendo responsable de líneas de remitente y asunto veraces, del envío únicamente a direcciones realmente suscritas y de la autenticación del dominio (SPF/DKIM/DMARC) en el servicio de envío. Nada de esto constituye asesoramiento jurídico.

## Opciones: doble opt-in, Turnstile, automatización RSS

Hay tres funciones integradas, pero desactivadas por defecto para que el sistema siga siendo operativo sin configuración:

- **Doble opt-in** (`DOUBLE_OPT_IN = "true"`): los nuevos suscriptores se guardan como `pending` y solo se activan tras hacer clic en un enlace de confirmación. Para Suiza (DSG) y la UE, este procedimiento es la opción más rigurosa.
- **Protección contra bots** con Cloudflare Turnstile: configure la clave de sitio y la clave secreta como variables; el widget aparece automáticamente en ambos formularios y el Worker verifica cada suscripción en el servidor. Sin un token válido, la suscripción se rechaza.
- **Envío automático RSS**: una tarea Cron comprueba cada 15 minutos el feed del propio blog (RSS 2.0 o Atom) y pone automáticamente los artículos nuevos en la cola de envío. Hay dos protecciones integradas: en la primera ejecución, el feed existente solo se marca como referencia (el archivo no se envía como una avalancha de correos), y cada ID de artículo se guarda en `sent_posts`, de modo que ninguna entrada se envía dos veces.

## Límites

La plantilla se ha mantenido deliberadamente mínima. El envío desde la cola entrega de forma predeterminada unas 40 correos electrónicos por minuto en el plan Free; por tanto, una campaña a 1'000 destinatarios tarda unos 25 minutos, algo irrelevante para un boletín. En el plan Workers de pago (10'000 subrequests por llamada en lugar de 50), `SEND_BATCH` se puede aumentar a varios cientos; con un adaptador por lotes (una llamada API, hasta unos 1'000 correos electrónicos), incluso el plan Free puede enviar listas grandes en pocos minutos. La entregabilidad depende, como en cualquier sistema, del propio dominio remitente: SPF, DKIM y DMARC deben estar verificados en el servicio de envío elegido; de lo contrario, el boletín acabará en spam. Y el valor predeterminado de opt-in único es el inicio más sencillo, pero no la variante de cumplimiento más conservadora; para ello está el interruptor.

En cuanto a los costes: Workers y D1 tienen cuotas generosas de nivel gratuito (entre otras, 100'000 solicitudes al día), que un formulario de suscripción y envíos semanales a una lista pequeña o mediana no agotan. Si se alcanza un límite, Cloudflare limita el servicio en el plan Free en lugar de emitir una factura.

## Probarlo

El código fuente, incluido el botón de despliegue, está en [GitHub](https://github.com/pfstr/newsletter-template); allí también se encuentra la documentación completa de las variables de configuración.

[![GitHub: pfstr/newsletter-template](../images/serverloser-newsletter-cloudflare-workers-d1/github-newsletter-template.svg)](https://github.com/pfstr/newsletter-template)

## Fuentes

1.  [pfstr/newsletter-template](https://github.com/pfstr/newsletter-template): código fuente de la plantilla (MIT) con botón de despliegue y documentación.

2.  [Deploy to Cloudflare buttons](https://developers.cloudflare.com/workers/platform/deploy-buttons/): aprovisionamiento automático de recursos, clonación del repositorio y CI durante el despliegue.

3.  [Deploy buttons: environment variables and secrets](https://developers.cloudflare.com/changelog/post/2025-07-01-workers-deploy-button-supports-environment-variables-and-secrets/): los secretos y las variables se solicitan en el formulario de despliegue desde julio de 2025.

4.  [Cloudflare D1](https://developers.cloudflare.com/d1/): SQLite sin servidor, utilizado aquí para suscriptores, registro de envíos y deduplicación RSS.

5.  [Cloudflare Turnstile](https://developers.cloudflare.com/turnstile/): protección contra bots sin acertijos CAPTCHA, activable opcionalmente en la plantilla.

6.  [RFC 8058](https://datatracker.ietf.org/doc/html/rfc8058): Signaling One-Click Functionality for List Email Headers; base del botón nativo de cancelación en Gmail y Outlook.

7.  [Workers limits](https://developers.cloudflare.com/workers/platform/limits/): límites de subrequests por llamada (50 en el plan Free, 10'000 en el plan de pago); de ello se deriva el tamaño de lote del envío desde la cola.

8.  [FTC: CAN-SPAM Act Compliance Guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business): obligaciones para correos comerciales, incluida la dirección postal y un opt-out funcional.
