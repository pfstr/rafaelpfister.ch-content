---
title: "Inicio de sesión en HIN Mail: Webmail, token de Outlook, contraseña olvidada y HIN Mail Global"
navTitle: "Inicio de sesión en HIN Mail"
description: "Dónde y cómo iniciar sesión en HIN Mail: Webmail en webmail.hin.ch, token de correo para Outlook y smartphone, inicio de sesión sin HIN Client mediante código SMS o aplicación Authenticator, restablecimiento de contraseña y apertura de HIN Mail Global como destinatario sin cuenta HIN."
date: "2026-10-05"
kategorie: "Pasarela HIN"
timeToRead: "7 min de lectura"
themen:
  - hin-gateway
produkte:
  - "hin"
  - "outlook"
protokolle:
  - "troubleshooting"
  - "verschluesselung"
related:
  - hin-plattformerneuerung-2026
  - hin-mailgateway-backup-disaster-recovery
slug: "inicio-de-sesion-en-hin-mail-webmail-token-de-outlook-contrasena-olvidada-y-hin-mail-global"
translationId: "article-6f466c817c7c7c66"
translationOf: hin-mail-login
url: https://rafaelpfister.ch/es/blog/inicio-de-sesion-en-hin-mail-webmail-token-de-outlook-contrasena-olvidada-y-hin-mail-global
translationSourceHash: 8f863faa139fd22182bf59e849447e241e339e4fcc4e6a073e6c1a89344f8a06
translationModel: gpt-5.6-terra
translatedAt: 2026-10-06T10:40:02.958Z
translationReview: automatic
---

# Inicio de sesión en HIN Mail: Webmail, token de Outlook, contraseña olvidada y HIN Mail Global

Detrás de la búsqueda «HIN Mail Login» suelen existir tres situaciones diferentes: es miembro de HIN y desea leer sus correos en el navegador, desea configurar HIN Mail en Outlook o en el smartphone, o ha recibido como paciente, paciente o entidad externa un correo HIN cifrado y no tiene ninguna cuenta HIN. El inicio de sesión funciona de forma distinta en los tres casos.

| Situación | Dirección | Nombre de usuario | Contraseña |
|---|---|---|---|
| Webmail en el navegador | `webmail.hin.ch` | ID de HIN | Contraseña de HIN, protegida mediante HIN Client, código SMS o aplicación Authenticator |
| Outlook, Thunderbird, Apple Mail, smartphone | `imap.mail.hin.ch`, `smtp.mail.hin.ch` | ID de HIN | Token de correo de `apps.hin.ch` |
| Administración, token, contraseña de inicialización | `apps.hin.ch` | ID de HIN | Contraseña de HIN más segundo factor |
| Destinatario sin cuenta HIN (HIN Mail Global) | Adjunto `secure-email.html` o enlace en el correo | Número de móvil | Código SMS |

## HIN Webmail: inicio de sesión en el navegador

Puede acceder al webmail en `https://webmail.hin.ch`. Inicie sesión con su ID de HIN (el nombre de inicio de sesión de HIN, p. ej., `hmuster`) y su contraseña de HIN. El segundo factor lo proporciona uno de los siguientes métodos:

- **HIN Client:** Si el cliente está en ejecución en el puesto de trabajo y ha iniciado sesión, el webmail se abre sin ninguna consulta adicional. Con HIN Client 4.0, el inicio de sesión se realiza directamente en el navegador: introduzca la dirección protegida por HIN y será redirigido al inicio de sesión.
- **Código SMS (mTAN):** Tras la contraseña recibirá un código en el número de móvil registrado. Este método es adecuado para dispositivos sin HIN Client, como un portátil privado.
- **Aplicación HIN Authenticator:** La aplicación (App Store y Google Play, «HIN Authenticator») confirma el inicio de sesión en el smartphone.

El código SMS y la aplicación Authenticator deben activarse previamente. Si faltan ambos y HIN Client no está disponible, no es posible iniciar sesión en el webmail. Por ello, configure al menos una alternativa mientras el cliente siga funcionando.

En el webmail puede redactar y gestionar correos, mantener contactos y libretas de direcciones, buscar en el directorio de participantes de HIN y crear varias firmas. Debajo del nombre de usuario, una barra muestra el espacio libre del buzón; si está en rojo, debe eliminar correos o ampliar el almacenamiento.

## HIN Mail en Outlook y en el smartphone: inicio de sesión con token de correo

Los programas de correo no inician sesión con la contraseña de HIN, sino con un **token de correo**. Esta es la causa más frecuente de que Outlook solicite repetidamente la contraseña: la contraseña de HIN se rechaza al iniciar sesión mediante IMAP.

Así se genera el token:

1. Abra `https://apps.hin.ch` e inicie sesión con el ID de HIN (HIN Client, código SMS o aplicación Authenticator).
2. Seleccione la sección «HIN Mail».
3. Haga clic en «Añadir dispositivo» y seleccione «Mail Clients» como tipo de dispositivo.
4. Copie el token mostrado e introdúzcalo como contraseña en el programa de correo.

La configuración del servidor:

| Configuración | Correo entrante (IMAP) | Correo saliente (SMTP) |
|---|---|---|
| Servidor | `imap.mail.hin.ch` | `smtp.mail.hin.ch` |
| Puerto | `993` | `587` |
| Cifrado | SSL/TLS | STARTTLS |
| Nombre de usuario | ID de HIN | ID de HIN |
| Contraseña | Token de correo | Token de correo |
| Autenticación | normal | Activar «El servidor de correo saliente requiere autenticación» |

El nombre de usuario es el ID de HIN, no la dirección de correo electrónico. Outlook propone la dirección de correo electrónico durante la configuración automática; por tanto, la configuración debe realizarse manualmente mediante «Opciones avanzadas» o «Configuración manual».

El token está vinculado a su identidad HIN y puede revocarse y generarse de nuevo en `apps.hin.ch` en cualquier momento. Se recomienda un token propio para cada dispositivo: si pierde un smartphone, bloquee únicamente ese dispositivo y Outlook en el PC de la consulta seguirá funcionando. Una vez configurado el token, Outlook ya no necesita HIN Client para consultar el correo.

## Primer inicio de sesión sin HIN Client: activar la identidad

Las nuevas identidades HIN pueden activarse sin HIN Client. Necesita el nombre de usuario y la contraseña de inicialización de la documentación de HIN.

1. Abra el enlace de activación en el centro de servicios (`servicecenter.hin.ch`).
2. Introduzca el nombre de usuario y la contraseña de inicialización.
3. Establezca una contraseña propia: al menos 10 caracteres con números, mayúsculas, minúsculas y caracteres especiales.
4. Configure inmediatamente un segundo factor (código SMS o aplicación Authenticator).
5. A continuación, inicie sesión en `apps.hin.ch`.

Dispone de 10 minutos para la activación. Después, la contraseña de inicialización se restablece por motivos de seguridad y deberá solicitar una nueva.

## Contraseña de HIN olvidada

El restablecimiento depende de si se ha configurado un método de inicio de sesión alternativo.

**Con código SMS o aplicación Authenticator:**

1. Inicie sesión en `apps.hin.ch` mediante el método alternativo.
2. Seleccione la pestaña «HIN Client» y genere una nueva contraseña de inicialización.
3. En HIN Client, seleccione «Registro», introduzca el nombre de inicio de sesión y la nueva contraseña de inicialización.
4. Establezca una nueva contraseña y confirme «Registrar nueva identidad HIN».

**Sin método alternativo ni dispositivo con sesión iniciada**, solo queda el soporte telefónico de HIN (0848 830 740, de lunes a viernes de 08:00 a 18:00). Anote la nueva contraseña de inicialización antes de cerrar la ventana.

## HIN Mail Global: abrir correo cifrado sin cuenta HIN

Las consultas médicas, hospitales y laboratorios envían mensajes a pacientes o entidades sin afiliación a HIN mediante HIN Mail Global. El correo le llega como un correo electrónico normal, pero el contenido está cifrado en el adjunto `secure-email.html` o detrás del enlace «Leer mensaje cifrado». Aquí no existe un inicio de sesión HIN en sentido estricto; el acceso se realiza mediante su número de móvil.

**La primera vez:**

1. Descargue y abra el adjunto `secure-email.html` (o haga clic en el enlace del correo). Con servicios de webmail como Gmail, Bluewin o GMX, primero guárdelo y luego ábralo.
2. Seleccione el idioma e introduzca su propio número de móvil.
3. Introduzca el código recibido por SMS. Se mostrará el mensaje.

**En mensajes posteriores**, basta con abrir el adjunto; el código SMS se envía automáticamente al número registrado.

Tenga en cuenta:

- El mensaje puede consultarse mediante el enlace durante **30 días**; después, el acceso caduca. Guarde o imprima los contenidos importantes dentro de este plazo.
- Solo el destinatario original puede abrir los mensajes reenviados.
- Puede responder de forma cifrada.
- Un dispositivo puede marcarse como de confianza; entonces no se solicitará SMS durante un año.
- Si no recibe ningún código en 15 minutos, puede solicitar uno nuevo.
- Son compatibles las dos versiones actuales de Chrome, Safari, Edge y Firefox.

Si tiene problemas con HIN Mail Global, el soporte de correo de HIN puede ayudarle en el +41 58 670 48 69. Solo el remitente puede responder preguntas sobre el contenido del mensaje.

## Errores frecuentes al iniciar sesión en HIN Mail

| Síntoma | Causa | Solución |
|---|---|---|
| Outlook solicita constantemente la contraseña | Se introdujo la contraseña de HIN en lugar del token de correo | Genere un token en `apps.hin.ch` e introdúzcalo como contraseña |
| El inicio de sesión en Outlook falla pese al token | Dirección de correo electrónico como nombre de usuario | Utilice el ID de HIN como nombre de usuario |
| El envío falla, la recepción funciona | Autenticación SMTP no activada, puerto o cifrado incorrectos | Utilice el puerto `587` con STARTTLS y active la autenticación |
| El webmail solicita un código que nunca llega | Número de móvil desactualizado o código SMS no activado | Utilice la aplicación Authenticator o inicie sesión mediante HIN Client y actualice el número |
| Se rechaza la contraseña de inicialización | Se superó el plazo de 10 minutos | Solicite una nueva contraseña de inicialización |
| El token funcionaba y de repente dejó de hacerlo | Token revocado en `apps.hin.ch` o dispositivo eliminado | Genere un nuevo token para este dispositivo |
| HIN Mail Global: el enlace ya no abre nada | Plazo de 30 días vencido | Pida al remitente que lo envíe de nuevo |

## HIN Client 4.0: qué cambia en el inicio de sesión

HIN distribuirá HIN Client 4.0 automáticamente hasta el 26 de octubre de 2026; quien no reciba una actualización automática podrá descargarlo en `download.hin.ch`. La contraseña existente seguirá siendo válida y se confirmará una vez al iniciar sesión por primera vez con la nueva versión. A partir de entonces, el inicio de sesión en servicios protegidos por HIN, como el webmail, se realizará directamente en el navegador.

La versión 4.0 requiere Windows 11 con TPM o macOS 14 con Security Chip. Los dispositivos que no cumplan estos requisitos ya no podrán utilizar el cliente. El acceso al webmail seguirá siendo posible mediante código SMS o aplicación Authenticator, y la consulta de correo en Outlook mediante el token de correo. Encontrará detalles sobre los plazos de renovación de la plataforma en el artículo [Renovación de la plataforma HIN 2026](/blog/hin-plattformerneuerung-2026).

## Consultas y hospitales con servidor de correo propio

Las organizaciones con su propia pasarela de correo HIN no inician sesión en HIN Mail. Los buzones se encuentran en su propio Exchange o en Exchange Online, y la pasarela (un dispositivo SEPPmail) cifra los correos en su tránsito hacia y desde la comunidad HIN. En este caso, los problemas de inicio de sesión afectan al sistema de correo propio, no a la plataforma HIN. Esta pasarela será sustituida por la nueva HIN Gateway («Stargate»).

<aside class="offer-box">
  <span class="offer-box__tag">Comprobación gratuita</span>
  <p><strong>¿Gestiona su propia pasarela de correo HIN?</strong> Revisaré su entorno y le indicaré qué debe hacerse antes de la migración a «Stargate».</p>
  <a class="offer-box__cta" href="/stargate">Registrarse ahora</a>
</aside>

## Fuentes

1.  [Soporte de HIN: HIN Mail y Mobile](https://support.hin.ch/de/thema/hin-mail-mobile/): resumen con dirección de webmail e instrucciones para Outlook, Apple Mail, Thunderbird, iOS y Android.

2.  [Soporte de HIN: ¿Cómo funciona el webmail?](https://support.hin.ch/de/service/hin-mail-und-mobile/wie-funktioniert-das-webmail.cfm): funciones del webmail e indicador de almacenamiento.

3.  [Soporte de HIN: Instrucciones del servicio de token de correo para Mail Clients](https://support.hin.ch/de/hin-mail-mobile/mail-token-service-mail-clients/): generación de tokens en apps.hin.ch, servidores IMAP y SMTP, puertos y cifrado.

4.  [Soporte de HIN: ¿Qué es un token?](https://support.hin.ch/de/service/hin-mail-und-mobile/was-ist-ein-token.cfm): token de correo y token de acceso, vinculación con la identidad HIN.

5.  [Soporte de HIN: Activar identidad HIN (sin HIN Client)](https://support.hin.ch/de/hin-2fa/hin-identitaet-aktivieren/): activación en el centro de servicios, plazo de 10 minutos, reglas de contraseña, segundo factor.

6.  [Soporte de HIN: Contraseña olvidada](https://support.hin.ch/de/thema/hin-client/passwort-vergessen.cfm): restablecimiento mediante contraseña de inicialización, soporte telefónico sin método alternativo.

7.  [Soporte de HIN: HIN Mail para personas no miembros](https://support.hin.ch/en/service/hin-mail-to-non-members.cfm): disponibilidad durante 30 días, registro por SMS, dispositivos de confianza, navegadores compatibles, teléfono de soporte.

8.  [HIN: HIN Mail Global](https://www.hin.ch/services/hin-mail/hin-mail-global/): envío de correos cifrados a destinatarios sin afiliación a HIN.

9.  [Blog de HIN: Ya está disponible el nuevo HIN Client](https://www.hin.ch/de/blog/2026/neuer-hin-client.cfm): inicio de sesión en navegador, requisitos del sistema con TPM o Security Chip, despliegue hasta el 26 de octubre de 2026.

10.  [HIN Authenticator en App Store](https://apps.apple.com/ch/app/hin-authenticator/id1535944002): aplicación para el segundo factor.
