---
title: "Proton Drive en Linux: situación a octubre de 2026"
navTitle: "Proton Drive y Linux"
description: "El cliente oficial para Linux está anunciado, pero todavía no está disponible. Para scripts y servidores existe desde junio de 2026 la CLI oficial de Proton Drive; Proton Drive sigue pudiéndose montar únicamente con Rclone. Lo que falta es un acceso de máquina limitado a carpetas o tareas concretas."
date: "2026-10-01"
kategorie: "Proton Drive"
timeToRead: "8 min de lectura"
themen:
  - proton-drive
  - rclone
related:
  - proton-drive-cli
  - paperless-dokumente-clouddienst-auslagern
  - rclone-mount-in-docker-container
slug: "proton-drive-en-linux-situacion-en-julio-de-2026"
translationOf: "proton-drive-linux-status"
translationId: article-ca282447e0b9acff
translationModel: gpt-5.6-terra
translatedAt: 2026-10-02T10:02:29.572Z
translationReview: automatic
translationSourceHash: 73500e1be526e5e93bbd7bf789b1cc500976140cb4cb9542404e1e7a56b40ce0
url: https://rafaelpfister.ch/es/blog/proton-drive-en-linux-situacion-en-julio-de-2026
---

Proton Drive ofrece clientes de sincronización propios para Windows y macOS desde 2023. En Linux, hasta ahora existen la interfaz web, herramientas de la comunidad y, desde junio de 2026, una aplicación oficial de línea de comandos, pero todavía no un cliente de sincronización. En un servidor, la situación es aún más difícil, porque ni la sincronización de escritorio ni el inicio de sesión interactivo encajan bien.

Esta visión general describe la situación a 1 de octubre de 2026. Se basa en las hojas de ruta publicadas, el código fuente de la CLI de Proton Drive y una prueba práctica del backend de Rclone [como repositorio documental para Paperless-ngx](/blog/paperless-dokumente-clouddienst-auslagern).

**Actualización del 1 de octubre de 2026:** La primera versión, del 26 de julio, describía la aplicación de línea de comandos únicamente como una herramienta en el repositorio del SDK. Sin embargo, Proton ya la había publicado oficialmente el 9 de junio de 2026 como **Proton Drive CLI**, con compilaciones listas para Windows, macOS y Linux. La sección correspondiente y la tabla de recomendaciones se han revisado; los detalles se encuentran en el artículo específico sobre la [Proton Drive CLI](/blog/proton-drive-cli).

## El cliente para Linux está anunciado, pero aún no tiene fecha

En junio de 2026, Proton confirmó expresamente por primera vez que se está desarrollando un cliente para Linux. Se está creando sobre el nuevo SDK unificado y deberá utilizar la misma base técnica que las aplicaciones para Windows y macOS. A principios de octubre de 2026, sigue sin haber ni fecha ni beta pública.

Importante para situarlo: será un **cliente de sincronización de escritorio**. Para el escritorio, resuelve el problema. Sin embargo, para aplicaciones de servidor, un cliente de sincronización es la herramienta equivocada, ya que un servicio debe leer archivos directamente de Proton Drive y escribirlos allí. Un cliente de sincronización mantiene una copia local completa, precisamente lo que se quiere evitar cuando el almacenamiento es limitado.

## Rclone sigue siendo necesario para montajes y duplicaciones

En Linux, Rclone con su backend de `protondrive` es actualmente la herramienta más versátil. Puede copiar y sincronizar archivos y, como única solución disponible, proporcionar Proton Drive como un directorio local mediante un **montaje FUSE**. Hay dos limitaciones importantes:

**Está en beta y utiliza una API reconstruida.** Proton no documenta públicamente su API de Drive; el backend se basa en ingeniería inversa. En las pruebas funcionó de forma fiable, pero limitó las secuencias rápidas de llamadas con listas de directorios inconsistentes.

**Para el funcionamiento desatendido, Rclone solicita la clave TOTP.** El asistente de configuración denomina el campo `otp_secret_key`. Se refiere a la clave permanente de la configuración de 2FA, no al código de seis dígitos que muestra en ese momento una aplicación autenticadora. Rclone almacena este valor de forma ofuscada y genera por sí mismo un código TOTP válido en cada inicio de sesión.

Quien introduzca por error un código de un solo uso actual puede completar el primer inicio de sesión. Sin embargo, la siguiente autenticación volverá a fallar con el error 8002, porque Rclone no puede utilizar de nuevo el mismo código.

De este modo, la cuenta sigue protegida frente a una contraseña robada de forma aislada. Sin embargo, un servidor comprometido expone tanto la contraseña como la clave TOTP. Por ello, para accesos automatizados se recomienda una **cuenta de Proton dedicada**.

El comportamiento de este tipo de montaje en entornos Docker, incluidos dos problemas no documentados, se explica en el [artículo específico sobre Rclone en contenedores](/blog/rclone-mount-in-docker-container).

## La CLI oficial cubre scripts y copias de seguridad

El 9 de junio de 2026, Proton publicó la **Proton Drive CLI**, un único archivo ejecutable `proton-drive` para Windows, macOS y Linux. Se basa en el mismo SDK que las aplicaciones oficiales; el código fuente se encuentra en el repositorio público del SDK. Actualmente es la versión 0.8.0, del 13 de agosto de 2026, con compilaciones para x86-64 (también sin AVX2), ARM64 y distribuciones musl como Alpine.

El modelo de inicio de sesión es más limpio que el del backend de Rclone:

- `auth login` muestra una URL de inicio de sesión que también puede abrirse **en otro dispositivo**; el inicio de sesión se realiza de forma normal, **incluida la autenticación de dos factores**, incluso mediante SSH en un servidor sin escritorio
- la sesión se guarda en el **almacén de claves del sistema operativo** (Keychain, Credential Manager, libsecret) o, desde la versión 0.6.0, en el gestor de contraseñas `pass`, que resulta más práctico en servidores sin sesión de escritorio
- después: cargar y descargar archivos, moverlos, enviarlos a la papelera, gestionar comparticiones, invitaciones y enlaces públicos, utilizar Proton Photos; en cada caso con `--json` para una salida legible por máquinas

Así, la contraseña y la clave TOTP no tienen que estar almacenadas en el servidor. Para copias de seguridad y artefactos de compilación, la CLI es hoy una mejor elección que Rclone. Se mantienen dos límites: la CLI **no puede montar un sistema de archivos** ni **crear una duplicación con eliminaciones**; carga y descarga, pero no sincroniza. En el repositorio ya se incluye un comando `takeout` para una exportación local completa, pero todavía no se ha publicado.

El propio SDK sigue siendo considerado por Proton como no apto para producción en aplicaciones de terceros; su lanzamiento está previsto entre finales de 2026 y principios de 2027. La CLI no se ve afectada, porque Proton la publica directamente.

## La verdadera carencia: accesos de máquina

El núcleo del problema está un nivel más abajo que el cliente o el SDK: **Proton no conoce los accesos de máquina.** No hay contraseña de aplicación, cuenta de servicio ni token de alcance limitado. Toda automatización, ya sea un script de copia de seguridad, un montaje de servidor o un trabajo de CI, debe utilizar las credenciales completas de la cuenta.

Como comparación: en los almacenamientos compatibles con S3, los pares de claves de acceso son lo habitual, revocables y restringibles a buckets o prefijos. Google y Microsoft disponen de contraseñas de aplicación y cuentas de servicio. En Proton, en cambio, es todo o nada: quien quiere dar a un servidor acceso a una carpeta le da acceso a toda la cuenta.

En un servicio cifrado de extremo a extremo, esto es más difícil que con S3, porque un acceso limitado también tendría que implicar material de claves limitado. Sin embargo, las sesiones de la CLI muestran que Proton domina tales construcciones. Una sesión ya es un acceso derivado y revocable, solo que con el alcance completo de la cuenta. Un «token de máquina oficial para exactamente esta carpeta, solo de lectura» sería el mayor avance individual para el uso en servidores, muy por delante de cualquier cliente.

## Recomendación según el caso de uso

| Caso de uso | Situación en octubre de 2026 |
|---|---|
| Sincronización de escritorio en Linux | Esperar al cliente anunciado; hasta entonces, sincronización con Rclone o interfaz web |
| Copia de seguridad de servidor (subir archivos) | [Proton Drive CLI](/blog/proton-drive-cli) con `filesystem upload` y estrategia de conflictos `create-new-revision`; compatible oficialmente, sin contraseña almacenada |
| Duplicación con eliminaciones | Rclone con `sync`; tener en cuenta el estado beta |
| Montaje de sistema de archivos para servicios | Rclone con `mount`, clave TOTP almacenada y cuenta dedicada; la única [vía probada en la práctica](/blog/paperless-dokumente-clouddienst-auslagern) |
| Automatización de scripts, gestión de comparticiones | Proton Drive CLI con `--json`; versión 0.x, los comandos aún pueden cambiar |

En el escritorio Linux se puede esperar al cliente anunciado o utilizar Rclone por el momento. En servidores, la CLI oficial ya se encarga de las copias de seguridad y la automatización; para un montaje, Rclone sigue siendo la única solución práctica. Sin embargo, una solución provisional funcional solo se convertirá en una plataforma robusta cuando Proton ofrezca accesos de máquina limitados y un montaje compatible oficialmente.

## Fuentes

1.  [OMG Ubuntu: Proton Drive client is (finally) coming to Linux](https://www.omgubuntu.co.uk/2026/06/proton-drive-linux-client): la confirmación de junio de 2026 de que el cliente para Linux está en desarrollo, sin fecha.

2.  [Proton: Product roadmaps for spring and summer 2026](https://proton.me/blog/2026-spring-summer-roadmaps): la hoja de ruta con el cliente para Linux sin plazo temporal y el SDK como base de las propias aplicaciones.

3.  [ProtonDriveApps/sdk en GitHub](https://github.com/ProtonDriveApps/sdk): el repositorio público del SDK con el código fuente y el registro de cambios de la CLI.

4.  [Proton: Introducing Proton Drive CLI](https://proton.me/blog/proton-drive-cli): la publicación oficial de la CLI el 9 de junio de 2026.

5.  [Proton Drive CLI: Downloads](https://proton.me/download/drive/cli/index.html): versión actual 0.8.0, del 13 de agosto de 2026, con todas las compilaciones para plataformas.

6.  [Proton Drive SDK preview](https://proton.me/blog/proton-drive-sdk-preview): la propia valoración de Proton: aún no apto para producción en aplicaciones de terceros.

7.  [Rclone: Proton Drive](https://rclone.org/protondrive/): el backend, incluida la indicación beta y la opción `otp_secret_key` para el inicio de sesión desatendido.
