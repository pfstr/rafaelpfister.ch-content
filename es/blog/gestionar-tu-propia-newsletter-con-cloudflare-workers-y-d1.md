---
title: "Gestionar tu propia newsletter con Cloudflare Workers y D1"
navTitle: "Newsletter en Workers"
description: "La plantilla abierta proporciona suscripción, baja, cola y base de datos en tu propia cuenta de Cloudflare. Un botón de despliegue configura Worker, D1 y CI sin servidor local."
date: "2026-07-22"
kategorie: "Cloudflare Workers"
timeToRead: "8 min de lectura"
themen:
  - cloudflare-workers
slug: "gestionar-tu-propia-newsletter-con-cloudflare-workers-y-d1"
translationOf: "serverloser-newsletter-cloudflare-workers-d1"
url: "https://rafaelpfister.ch/es/blog/gestionar-tu-propia-newsletter-con-cloudflare-workers-y-d1"
translationId: article-4e7139acdb90923b
translationReview: automatic
translationSourceHash: 90c100386e148f80be4d4be81dc928f373431ce83b5f6e2336cfb0daafd3945e
translatedAt: 2026-07-29T12:29:38.951Z
---

# Gestionar tu propia newsletter con Cloudflare Workers y D1

Con un servicio de newsletters alojado, la lista de destinatarios queda en manos del proveedor y los costes suelen aumentar con el número de suscriptores. Un servidor propio ofrece más control, pero implica trabajo continuo: actualizaciones, monitorización, copias de seguridad y operación de un sistema que quizá solo envía una vez por semana.

Para este caso de uso ligero bastan endpoints HTTP, una pequeña base de datos y una tarea de envío programada. Cloudflare Workers y D1 proporcionan exactamente estos componentes. Mi plantilla abierta los configura en tu propia cuenta mediante un **botón Deploy to Cloudflare**. No se necesita una línea de comandos local ni un servidor que requiera mantenimiento permanente. El código fuente bajo licencia MIT está en [GitHub](https://github.com/pfstr/newsletter-template).

[![Deploy to Cloudflare](../images/serverloser-newsletter-cloudflare-workers-d1/deploy-to-cloudflare.svg)](https://deploy.workers.cloudflare.com/?url=https://github.com/pfstr/newsletter-template)

![El formulario de suscripción alojado de la plantilla](../images/serverloser-newsletter-cloudflare-workers-d1/newsletter-template-signup.png)

## Qué puede hacer la plantilla

- **Suscripción**: una página de suscripción alojada, un formulario incrustable para tu propio sitio web y un endpoint JSON
- **Baja con un clic**: conforme a RFC 8058, con un token individual por suscriptor
- **Información obligatoria integrada**: cada correo electrónico recibe automáticamente un pie de página con enlace de baja y dirección postal; se guardan los momentos de consentimiento y baja
- **Envío**: en una página protegida se pueden introducir el asunto y el HTML, enviar un correo de prueba y poner la campaña en cola; una tarea en segundo plano envía por lotes y repite los intentos fallidos
- **Datos propios**: los suscriptores se almacenan en una base de datos D1 de tu cuenta y se pueden exportar en cualquier momento
- **Opcional, desactivado por defecto**: doble opt-in, protección contra bots mediante Turnstile y envío automático de nuevos artículos del blog desde el feed RSS

## Arquitectura: un Worker, una base de datos

Todo el sistema es un único Cloudflare Worker con dos handlers: `fetch` para HTTP (enrutado con Hono) y `scheduled` para el disparador Cron, además de una base de datos D1. No hay un segundo servicio, ni un broker de colas independiente, ni un backend de administración propio; incluso la cola de envío es solo una tabla D1.

| Ruta | Función |
| --- | --- |
| `GET /` | Página de suscripción alojada |
| `GET /embed` | Formulario transparente para incrustar mediante iframe |
| `POST /api/subscribe` | Suscripción (CORS abierto para tu propio sitio web) |
| `GET /confirm` | Enlace de confirmación para doble opt-in |
| `GET/POST /unsubscribe` | Baja: página de confirmación mediante GET, ejecución mediante POST (un clic según RFC 8058) |
| `GET /admin` | Página de envío (formulario) |
| `POST /api/send` | Poner la campaña en cola, protegida mediante token de administrador |

El modelo de datos incluye cuatro tablas: `subscribers` (correo electrónico como clave primaria, nombre, estado, tokens de baja y confirmación, una columna JSON para campos adicionales definidos por el usuario, así como marcas de tiempo de confirmación y baja), `campaigns` con asunto, contenido y contadores por envío, `outbox` como cola de envío (una fila por destinatario) y `sent_posts` para la deduplicación del envío RSS.

## Despliegue sin línea de comandos

La parte más interesante no es el código, sino el camino hasta tener el sistema en funcionamiento. El botón Deploy to Cloudflare lee la configuración de Wrangler del repositorio y realiza toda la instalación: clona el repositorio en tu propia cuenta de GitHub, aprovisiona la base de datos D1, ejecuta las migraciones de esquema y configura CI para que cada push se despliegue automáticamente. Desde julio de 2025, el flujo de despliegue también solicita variables de entorno y secretos directamente en el formulario: en el caso de esta plantilla, la contraseña de administrador (`ADMIN_TOKEN`), el nombre y la dirección del remitente, el interruptor de doble opt-in y el tamaño del lote de envío (`SEND_BATCH`).

El resultado tras un clic y un formulario: la página de suscripción está activa en `https://<worker-name>.workers.dev` y recopila suscriptores. No se abre un terminal en ningún momento.

## Recopilar suscriptores

Para integrarlo en tu propio sitio web hay tres opciones, en orden creciente de profundidad de integración. La más sencilla: compartir el enlace a la página de suscripción alojada. La más práctica para creadores de sitios (WordPress, Webflow, Squarespace, Framer): una línea de iframe en cualquier bloque de inserción HTML.

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

El formulario recopila por defecto el correo electrónico y, opcionalmente, el nombre. Puedes definir campos adicionales (empresa, país, …) en un único archivo (`src/fields.ts`); aparecerán automáticamente en ambos formularios y se guardarán como JSON en la base de datos.

## Envío: proveedor propio en lugar de un proveedor integrado

Para el envío de correo electrónico, la plantilla toma una decisión deliberada: es **agnóstica respecto al proveedor**. El archivo `src/email.ts` contiene un único adaptador `sendEmail()` con un ejemplo comentado para una API HTTP genérica. Qué servicio de envío conectes ahí es decisión tuya. No hay ningún proveedor integrado de forma fija ni se presupone el registro en un servicio concreto. La recopilación de suscriptores ya funciona por completo sin configuración de envío; el envío se habilita en cuanto se implementa el adaptador y se establece el secreto del proveedor. Si el proveedor ofrece además un endpoint por lotes (una llamada API, muchos correos), se puede añadir en el mismo archivo un adaptador opcional `sendEmailBatch()`; también hay un ejemplo comentado para ello.

El envío se gestiona desde la página `/admin`: introduce el asunto y el HTML del correo, envía una prueba a tu propia dirección y, después, pon la campaña en cola para todos los suscriptores. En los correos están disponibles las etiquetas de combinación `{{unsubscribe_url}}`, `{{email}}` y `{{name}}`.

El envío propiamente dicho ocurre en segundo plano, siguiendo el patrón Transactional Outbox: `POST /api/send` escribe la campaña y una fila por destinatario en la base de datos y responde de inmediato. A continuación, una tarea Cron por minuto entrega `SEND_BATCH` correos por ejecución, 40 de forma predeterminada: elegido para que cada ejecución se mantenga dentro de los límites de subsolicitudes del plan gratuito de Workers. Las filas se reclaman de forma atómica, por lo que las ejecuciones superpuestas nunca pueden enviar dos veces; las entregas fallidas se repiten hasta tres veces y las ejecuciones interrumpidas se reanudan tras diez minutos. Y quien se da de baja mientras su propio correo aún está en la cola ya no lo recibe: el opt-out también cancela los mensajes que ya estaban en cola.

## La baja y las pruebas forman parte del núcleo

Quien envía una newsletter está sujeto a la legislación antispam y de protección de datos: la CAN-SPAM Act estadounidense, el RGPD y ePrivacy en la UE, y la UWG en Suiza. Una parte esencial de lo que se paga a los servicios de newsletter es precisamente el cumplimiento de estas obligaciones. La plantilla se encarga de la parte mecánica:

- **Pie de página obligatorio**: cada correo de campaña recibe automáticamente un pie con un enlace de baja funcional y la dirección postal del remitente (`SENDER_ADDRESS`); CAN-SPAM exige una dirección física en los correos comerciales. La página de envío avisa mientras falte la dirección.
- **Cabecera List-Unsubscribe según RFC 8058** en cada envío: el botón nativo de baja de Gmail y Outlook, que Gmail y Yahoo exigen a los remitentes masivos desde 2024. La aplicación compone las cabeceras; el adaptador del proveedor propio solo tiene que reenviarlas.
- **Baja segura frente a escáneres**: el enlace de baja lleva a una página de confirmación con un único botón. Los escáneres de correo corporativos que consultan previamente todos los enlaces de un correo no pueden dar de baja a nadie por accidente; los clientes de correo usan directamente el POST de un clic.
- **Minimización de datos y prueba**: un opt-out surte efecto de inmediato, elimina el nombre y los campos adicionales y queda registrado con marca de tiempo, al igual que la suscripción y la confirmación de doble opt-in. Así, el consentimiento se puede demostrar posteriormente (responsabilidad proactiva del RGPD).
- **Enlace de privacidad**: con `PRIVACY_URL` configurado, aparece bajo el formulario de suscripción un enlace a tu propia política de privacidad.

La responsabilidad de mantener líneas de remitente y asunto veraces, enviar solo a direcciones realmente suscritas y autenticar el dominio (SPF/DKIM/DMARC) ante el servicio de envío recae en el operador. Nada de esto constituye asesoramiento jurídico.

## Opciones: doble opt-in, Turnstile, automatización RSS

Hay tres funciones integradas, pero desactivadas por defecto para que el sistema siga funcionando sin configuración:

- **Doble opt-in** (`DOUBLE_OPT_IN = "true"`): los nuevos suscriptores se guardan como `pending` y solo se activan tras hacer clic en un enlace de confirmación. Para Suiza (DSG) y la UE, este procedimiento es la opción más rigurosa.
- **Protección contra bots** con Cloudflare Turnstile: basta con establecer la clave de sitio y la clave secreta como variables; el widget aparece automáticamente en ambos formularios y el Worker verifica cada suscripción en el servidor. Sin un token válido, la suscripción se rechaza.
- **Envío automático RSS**: una tarea Cron comprueba cada 15 minutos el feed del propio blog (RSS 2.0 o Atom) y pone automáticamente los nuevos artículos en la cola de envío. Hay dos salvaguardas integradas: en la primera ejecución, el feed existente solo se marca como referencia (por tanto, el archivo no se envía como una avalancha de correos), y cada ID de artículo se registra en `sent_posts`, para que ninguna publicación se envíe dos veces.

## Límites

La plantilla se mantiene deliberadamente minimalista. El envío mediante cola entrega en el plan gratuito unas 40 correos por minuto de forma predeterminada; por tanto, una campaña para 1.000 destinatarios tarda unos 25 minutos, algo irrelevante para una newsletter. En el plan de pago de Workers (10.000 subsolicitudes por llamada en lugar de 50), `SEND_BATCH` puede aumentarse a cientos; con un adaptador por lotes (una llamada API, hasta unos 1.000 correos), incluso el plan gratuito envía listas grandes en pocos minutos. Como en cualquier sistema, la entregabilidad depende del propio dominio remitente: SPF, DKIM y DMARC deben estar verificados con el servicio de envío elegido; de lo contrario, la newsletter acabará en spam. Y el valor predeterminado de opt-in único es la forma más sencilla de empezar, pero no la variante de cumplimiento más conservadora; para ello está el interruptor.

En cuanto a costes: Workers y D1 tienen cuotas generosas de nivel gratuito (entre otras, 100.000 solicitudes al día) que un formulario de suscripción y envíos semanales a una lista pequeña o mediana no agotan. Si se alcanza un límite, Cloudflare limita la velocidad en el plan gratuito en lugar de emitir una factura.

## Probarlo

El código fuente, incluido el botón de despliegue, está en [GitHub](https://github.com/pfstr/newsletter-template); allí también se encuentra la documentación completa de las variables de configuración.

[![GitHub: pfstr/newsletter-template](../images/serverloser-newsletter-cloudflare-workers-d1/github-newsletter-template.svg)](https://github.com/pfstr/newsletter-template)

## Fuentes

1.  [pfstr/newsletter-template](https://github.com/pfstr/newsletter-template): código fuente de la plantilla (MIT) con botón de despliegue y documentación.

2.  [Deploy to Cloudflare buttons](https://developers.cloudflare.com/workers/platform/deploy-buttons/): aprovisionamiento automático de recursos, clonación del repositorio y CI durante el despliegue.

3.  [Deploy buttons: environment variables and secrets](https://developers.cloudflare.com/changelog/post/2025-07-01-workers-deploy-button-supports-environment-variables-and-secrets/): los secretos y las variables se solicitan en el formulario de despliegue desde julio de 2025.

4.  [Cloudflare D1](https://developers.cloudflare.com/d1/): SQLite sin servidor, usado aquí para suscriptores, registro de envíos y deduplicación RSS.

5.  [Cloudflare Turnstile](https://developers.cloudflare.com/turnstile/): protección contra bots sin acertijos CAPTCHA, activable opcionalmente en la plantilla.

6.  [RFC 8058](https://datatracker.ietf.org/doc/html/rfc8058): Signaling One-Click Functionality for List Email Headers; base del botón nativo de baja en Gmail y Outlook.

7.  [Workers limits](https://developers.cloudflare.com/workers/platform/limits/): límites de subsolicitudes por llamada (50 en el plan gratuito, 10.000 en el plan de pago); de ahí se deriva el tamaño de lote del envío mediante cola.

8.  [FTC: CAN-SPAM Act Compliance Guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business): obligaciones para correos comerciales, incluida la dirección postal y un opt-out funcional.
