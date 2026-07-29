---
title: "Proton Drive en Linux: situación en julio de 2026"
navTitle: "Proton Drive y Linux"
description: "El cliente oficial para Linux está anunciado, pero aún no está disponible. Actualmente, Proton Drive puede montarse en servidores con Rclone; el nuevo SDK muestra la dirección técnica. Lo que sigue faltando es un acceso de máquina limitado a carpetas o tareas concretas."
date: "2026-07-26"
kategorie: "Proton Drive"
timeToRead: "8 min de lectura"
themen:
  - "proton-drive"
  - "rclone"
related:
  - "paperless-dokumente-clouddienst-auslagern"
  - "rclone-mount-in-docker-container"
slug: "proton-drive-en-linux-situacion-en-julio-de-2026"
translationOf: "proton-drive-linux-status"
url: "https://rafaelpfister.ch/es/blog/proton-drive-en-linux-situacion-en-julio-de-2026"
---

Para Windows y macOS, Proton Drive ofrece sus propios clientes de sincronización desde 2023. En Linux, hasta ahora solo existen la interfaz web, herramientas de la comunidad y un SDK oficial en fase de vista previa. En un servidor, la situación es aún más complicada, ya que allí ni la sincronización de escritorio ni un inicio de sesión interactivo encajan bien.

Este resumen describe la situación en julio de 2026. Además de las hojas de ruta publicadas, se basa en una prueba práctica del backend de Rclone [como almacenamiento de documentos para Paperless-ngx](/blog/paperless-dokumente-clouddienst-auslagern).

## El cliente para Linux está anunciado, pero sigue sin fecha

En junio de 2026, Proton confirmó por primera vez de forma explícita que está desarrollando un cliente para Linux. Se basa en el nuevo SDK unificado y utilizará la misma base técnica que las aplicaciones para Windows y macOS. Aún no hay fecha ni beta pública.

Importante para contextualizarlo: será un **cliente de sincronización de escritorio**. Para el escritorio, resuelve el problema. Sin embargo, para aplicaciones de servidor, un cliente de sincronización es la herramienta equivocada, ya que un servicio debe leer archivos directamente desde Proton Drive y escribirlos allí. Un cliente de sincronización mantiene una copia local completa, justo lo que se quiere evitar cuando el almacenamiento es limitado.

## Hoy Rclone se encarga del trabajo práctico

En Linux, Rclone y su backend `protondrive` son actualmente la herramienta más versátil. Puede copiar y sincronizar archivos y, como única solución disponible, proporcionar Proton Drive como un directorio local mediante un **montaje FUSE**. Hay dos limitaciones importantes:

**Está en beta y se basa en una API reconstruida.** Proton no documenta públicamente su API de Drive; el backend se basa en ingeniería inversa. En las pruebas funcionó de forma fiable, pero limitó la velocidad con secuencias rápidas de llamadas y listados de directorios inconsistentes.

**Para funcionar sin supervisión, Rclone solicita la clave TOTP.** El asistente de configuración denomina el campo `otp_secret_key`. Se refiere a la clave permanente de la configuración de 2FA, no al código de seis dígitos que muestra en ese momento una aplicación de autenticación. Rclone guarda este valor de forma ofuscada y genera por sí mismo un código TOTP válido en cada inicio de sesión.

Quien introduzca por error un código de un solo uso actual podrá completar el primer inicio de sesión. Sin embargo, la siguiente reautenticación fallará con el error 8002, porque Rclone no puede volver a usar el mismo código.

Así, la cuenta sigue protegida frente al robo aislado de una contraseña. Sin embargo, un servidor comprometido expone tanto la contraseña como la clave TOTP. Por ello, para accesos automatizados se recomienda una **cuenta de Proton dedicada**.

Cómo se comporta un montaje así en entornos Docker, incluidas dos trampas no documentadas, se explica en el [artículo específico sobre Rclone en contenedores](/blog/rclone-mount-in-docker-container).

## El SDK oficial muestra hacia dónde va el desarrollo

Paralelamente, Proton está migrando sus aplicaciones a un **SDK oficial** para JavaScript y C#, con bindings para Swift y Kotlin. El repositorio público también incluye una herramienta de línea de comandos. Su modelo de inicio de sesión es más limpio que el del backend de Rclone:

- `auth login` abre el navegador; el inicio de sesión se realiza de forma habitual, **incluida la autenticación de dos factores**
- la sesión se guarda en el **almacén de claves del sistema operativo** (Keychain, Credential Manager, libsecret), y el SDK la renueva por sí mismo
- después: listar archivos con salida JSON legible por máquinas, subirlos y comprobar comparticiones

De este modo, la contraseña y la clave TOTP no tienen que figurar en un archivo de configuración. Sin embargo, para uso en servidores siguen existiendo tres límites: la CLI **no puede montar un sistema de archivos**, el inicio de sesión abre un navegador y Proton aún no considera que el SDK esté listo para producción en aplicaciones de terceros. Su lanzamiento está previsto entre finales de 2026 y principios de 2027.

## La verdadera carencia: accesos de máquina

El núcleo del problema está un nivel por debajo del cliente o el SDK: **Proton no conoce los accesos de máquina.** No hay contraseña de aplicación, cuenta de servicio ni token con alcance limitado. Toda automatización, ya sea un script de copia de seguridad, un montaje en servidor o un trabajo de CI, debe trabajar con las credenciales completas de la cuenta.

Como comparación: en almacenamientos compatibles con S3, los pares de claves de acceso son lo habitual, se pueden revocar y limitar a buckets o prefijos. Google y Microsoft disponen de contraseñas de aplicación y cuentas de servicio. En Proton, en cambio, todo o nada: quien quiera dar a un servidor acceso a una carpeta le da acceso a toda la cuenta.

Hay que reconocer que esto es más difícil en un servicio con cifrado de extremo a extremo que en S3, porque un acceso limitado también tendría que implicar material de claves limitado. Sin embargo, las sesiones del SDK demuestran que Proton domina este tipo de construcciones. Una sesión ya es un acceso derivado y revocable. Un «token de máquina oficial para esta carpeta concreta, solo de lectura» sería el mayor avance individual para el uso en servidores, mucho antes que cualquier cliente.

## Recomendación según el caso de uso

| Caso de uso | Situación en julio de 2026 |
|---|---|
| Sincronización de escritorio en Linux | Esperar al cliente anunciado; hasta entonces, sincronización con Rclone o interfaz web |
| Copia de seguridad en servidor (subir archivos) | Rclone con `copy` o `sync`; funciona, teniendo en cuenta su estado beta |
| Montaje de sistema de archivos para servicios | Rclone con `mount`, clave TOTP almacenada y cuenta dedicada; la única [vía probada en la práctica](/blog/paperless-dokumente-clouddienst-auslagern) |
| Automatización mediante scripts con inicio de sesión limpio | Seguir de cerca la CLI del SDK; aún es demasiado pronto para producción |

En el escritorio Linux se puede esperar al cliente anunciado o utilizar Rclone por ahora. En servidores, Rclone sigue siendo la única solución práctica de montaje. Sin embargo, una solución provisional funcional solo se convertirá en una plataforma sólida cuando Proton ofrezca accesos de máquina limitados y un montaje compatible oficialmente.

## Fuentes

1.  [OMG Ubuntu: el cliente de Proton Drive llega (por fin) a Linux](https://www.omgubuntu.co.uk/2026/06/proton-drive-linux-client): la confirmación de junio de 2026 de que el cliente para Linux está en desarrollo, sin fecha.

2.  [Proton: hojas de ruta de productos para la primavera y el verano de 2026](https://proton.me/blog/2026-spring-summer-roadmaps): la hoja de ruta con el cliente para Linux sin ventana temporal y el SDK como base de las propias aplicaciones.

3.  [ProtonDriveApps/sdk en GitHub](https://github.com/ProtonDriveApps/sdk): el repositorio público del SDK, incluida la CLI con inicio de sesión en el navegador y sesión en el almacén de claves.

4.  [Vista previa del SDK de Proton Drive](https://proton.me/blog/proton-drive-sdk-preview): la propia valoración de Proton: aún no está listo para producción en aplicaciones de terceros.

5.  [Rclone: Proton Drive](https://rclone.org/protondrive/): el backend, incluida la indicación de beta y la opción `otp_secret_key` para el inicio de sesión sin supervisión.
