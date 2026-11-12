---
title: "RustDesk 1.5.3: conexión con aprobación, destinos de relay verificados e IPC de Windows reforzado"
navTitle: "RustDesk 1.5.3"
description: "RustDesk 1.5.3 incorpora conexiones sin contraseña con aprobación de la otra parte, acepta solicitudes de relay solo para direcciones ofrecidas y autentica las tuberías IPC en Windows. Las estrategias del servidor ya no pueden modificar ajustes sensibles. Se recomienda actualizar todos los clientes."
date: "2026-11-11"
kategorie: "Asistencia remota y soporte"
timeToRead: "5 min de lectura"
themen:
  - fernwartung
produkte:
  - "rustdesk"
protokolle:
  - "releases"
  - "haertung"
slug: "rustdesk-1-5-3-conexion-con-aprobacion-destinos-de-relay-verificados-e-ipc-de-windows-reforzado"
translationId: article-557eece28ec889f1
translationOf: rustdesk-1-5-3
url: https://rafaelpfister.ch/es/blog/rustdesk-1-5-3-conexion-con-aprobacion-destinos-de-relay-verificados-e-ipc-de-windows-reforzado
translationSourceHash: 3d06a4983979b612a9c43bd0a3eacbb0e13d9692a772e95478c351673d707815
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T03:34:21.698Z
translationReview: automatic
---

# RustDesk 1.5.3: conexión con aprobación, destinos de relay verificados e IPC de Windows reforzado

RustDesk publicó la versión 1.5.3 del cliente el 9 de noviembre de 2026, ocho días después de la 1.5.1. La página de lanzamientos no incluye una versión 1.5.2. El lanzamiento incorpora una nueva función (establecimiento de conexión con aprobación de la otra parte), un cambio en la lógica de relay y 22 correcciones de errores. Varias de ellas afectan a procesos relevantes para la seguridad: validación de direcciones de relay y UUID de relay, autenticación de las Named Pipes en Windows, claves de configuración bloqueadas en las estrategias del servidor y, de nuevo, la transferencia del instalador en Windows. El fabricante no menciona una CVE ni un Security Advisory al respecto (a fecha de 11 de noviembre de 2026). El lanzamiento está dirigido a todas las personas que operan clientes RustDesk, en especial en Windows y con servidor propio. El estado anterior, con actualizaciones firmadas y Trusted Devices V2, se describe en el artículo [RustDesk 1.5.1: actualizaciones firmadas, Trusted Devices V2 y refuerzo de la transferencia de archivos](/blog/rustdesk-1-5-1).

## Las versiones de un vistazo

| Versión | Fecha | Tipo | Puntos clave |
|---|---|---|---|
| 1.5.3 | 09.11.2026 | Lanzamiento del cliente | Request approval, relay solo a direcciones ofrecidas, autenticación IPC en Windows, claves bloqueadas en estrategias, correcciones del instalador (F-Secure, Junction), correcciones para macOS, DRM, Wayland, Android e iOS |

El servidor de código abierto (`rustdesk-server`) no ha recibido un nuevo lanzamiento; la última versión sigue siendo la 1.1.16, del 20 de julio de 2026.

## Nueva función: Request approval

Con el PR #16784 se puede establecer una conexión sin contraseña de RustDesk. La otra parte debe aceptarla en un diálogo. Según el PR, en este modo el cliente no envía ninguna contraseña desde ninguna fuente, tampoco desde el almacenamiento local ni la libreta de direcciones. Se ignora una contraseña proporcionada explícitamente. La selección aparece como el submenú «Request approval» en el desplegable de ID y en los menús de las tarjetas de pares, respectivamente con los tipos de conexión compatibles con la plataforma y la otra parte (conexión, transferencia de archivos y otros).

Para scripts y accesos directos, el PR menciona el parámetro de línea de comandos `--request-approval` y el parámetro URI `request-approval=true`. El modo se mantiene al reconectar, al mover una sesión a otra ventana y en conexiones adicionales desde la barra de herramientas remota. Las conexiones normales conservan su comportamiento de contraseña anterior.

Según el PR, se aplican los siguientes límites:

- Los métodos de autenticación disponibles no se negocian con la otra parte. Si allí solo se permite el inicio de sesión con contraseña o se aplican restricciones en la pantalla de inicio de sesión, estas se mantienen como hasta ahora.
- Si ya existe una ventana para el mismo par y el mismo tipo de conexión, RustDesk la activa sin cambiar el método de inicio de sesión. Para una nueva conexión con aprobación, cierre primero la sesión existente.
- Con la multiplexación de túneles, se puede reutilizar un túnel ya aprobado sin volver a solicitar confirmación.

El PR no incluye información sobre la compatibilidad con versiones anteriores de la otra parte.

## Relay solo a direcciones ofrecidas

El PR #16858 cierra una brecha en la negociación de relay. Hasta ahora, la parte controlada aceptaba sin verificar la dirección de relay y el UUID de un mensaje `RequestRelay` retransmitido y se conectaba a la dirección indicada. Ahora, la parte controlada recuerda las direcciones de relay que se le han ofrecido, así como el relay que elegiría ella misma (como máximo 32 entradas, válidas durante diez minutos desde el último mensaje). Descarta las solicitudes con otro relay antes de la resolución de nombres. Además, ambas partes rechazan UUID vacíos o de más de 128 bytes, así como aquellos que contengan caracteres distintos de letras ASCII, dígitos, `-` y `_`. Según el PR, esto corresponde a la validación que hbbs ya realiza.

Una solicitud rechazada no recibe respuesta; por ello, el intento de relay del controlador termina en un tiempo de espera en lugar de un mensaje de error. Un relay que elimine de la configuración dejará de aceptarse, como máximo, diez minutos después de su último mensaje.

Para quienes operan un servidor propio, una limitación es relevante: si hbbs se ejecuta con la opción marcada como obsoleta `--mask`, se rechaza la alternativa a un relay tras un intento fallido de WebRTC si hbbs ha reescrito la dirección de relay para la LAN. Este punto seguía abierto al fusionarse el cambio. Compruebe si su instancia de hbbs utiliza `--mask` y pruebe las conexiones de relay después de la actualización.

## Otras correcciones relevantes para la seguridad

- **IPC de Windows (PR #16755):** Según el PR, otros usuarios podían crear anteriormente instancias adicionales del servidor de las Named Pipes para la IPC principal y de servicio, y así interceptar conexiones a RustDesk, leer datos IPC o falsificar respuestas de configuración. Los clientes ahora abren las tuberías sin este permiso y autentican el servidor antes del intercambio de datos. Las tuberías de servicio deben pertenecer a SYSTEM. Según el PR, sigue siendo posible un denegación de servicio si un atacante ocupa primero el nombre de la tubería, y las versiones anteriores con reglas de acceso más abiertas no podrán establecer conexión hasta actualizarse.
- **Estrategias (PR #16696):** Las estrategias distribuidas por el servidor mediante heartbeat ya no pueden modificar ajustes sensibles. Entre las claves bloqueadas están `api-server`, `custom-rendezvous-server`, `relay-server`, `key`, `2fa`, `bot`, `enable-trusted-devices`, `trusted-device-expiration-days` y `websocket-port`. Siguen siendo configurables, por ejemplo, `approve-mode`, `verification-method`, `access-mode`, `whitelist`, `id-whitelist` y `allow-auto-update`. Las claves rechazadas se registran sin su valor. El autor califica expresamente el cambio como un refuerzo limitado: las respuestas de estrategia siguen sin autenticarse.
- **Instalador de Windows (PR #16889, #16908, #16871):** La transferencia mediante una Named Pipe introducida con la 1.5.1 se ha eliminado de nuevo, ya que, según el PR, impedía la instalación en sistemas con F-Secure DeepGuard y «Stop service» (error `0x800700E8`). La protección de los archivos preparados se basa ahora en las DACL restrictivas de la 1.5.1 y en directorios de staging propios cuyos handles permanecen abiertos durante la transferencia. Si `C:\Windows\Temp` se redirige mediante una Junction, el instalador utiliza en su lugar la carpeta ProgramData del sistema, en vez de abortar con un error.
- **macOS (PR #16729, #16879):** El Privacy Mode se desactiva al finalizar la sesión antes del bloqueo, y el daemon del servicio se inicia directamente en lugar de mediante `sh`.
- **Lista blanca de IP (PR #16692):** Se ignoran los espacios alrededor de las entradas de la lista blanca de IP.

Si la instalación en Windows falla con F-Secure, pruebe la 1.5.3 en un dispositivo afectado antes del despliegue general. Según el PR #16889, la confirmación de la corrección de F-Secure por parte de quien la reportó seguía pendiente al fusionarse el cambio. Si hasta ahora distribuye valores de configuración como servidores API, ID o relay mediante estrategias, deberá configurarlos en el futuro por otra vía.

## Correcciones de errores por plataforma

- **Windows:** La desinstalación continúa tras errores en la sincronización de configuración, se vuelve a intentar la IPC de servicio antes de cerrar la GUI y el marcador de propietario del portapapeles se escribe primero.
- **Linux:** La pantalla de inicio de sesión con greetd y Wayland se proporciona mediante DRM, los contenidos de pantalla sin cambios no se codifican de nuevo con DRM y los errores de captura de Wayland sin asignación aparecen con su mensaje real en lugar de «x11 expected».
- **Android:** Ya no se produce un bloqueo con contenidos del portapapeles de más de 1 MB, los fotogramas se copian antes de liberar la imagen y el hebreo y el indonesio se asignan correctamente desde códigos de configuración regional antiguos.
- **iOS:** La pantalla de inicio sigue el Dark Mode del sistema.
- **Otros:** El reenvío de puertos mantiene abierto el handle de KCP y envía un motivo de cierre, las barras laterales de la ventana de aceptación permanecen en el monitor actual en Windows, los dispositivos de entrada de audio se buscan únicamente entre los dispositivos de entrada y el tamaño mínimo de la ventana principal de Sciter ya no se escala dos veces.

## Problemas conocidos

- **Protección contra rollback en actualizaciones firmadas (desde la 1.5.1):** pendiente. Las notas de lanzamiento de la 1.5.3 no mencionan ningún cambio al respecto.
- **Variante portable en Windows (desde la 1.5.1):** Según el PR #15914, el refuerzo se había trasladado a una rama independiente. Las notas de lanzamiento de la 1.5.3 no mencionan ningún cambio al respecto.
- **Trusted Devices V2 (desde la 1.5.1):** Los límites conocidos (revocación asíncrona, caducidad según el reloj del sistema del host) no se abordan en las notas de lanzamiento de la 1.5.3.
- **Alternativa de relay con `--mask` (nuevo):** pendiente según el PR #16858.
- **Nombre de tubería ocupado previamente (nuevo):** según el PR #16755, sigue siendo posible una denegación de servicio.
- **Consulta UAC cancelada (nuevo):** según el PR #16889, los archivos preparados permanecen en el directorio protegido y un administrador puede eliminarlos.
- **Respuestas de estrategia (nuevo):** siguen sin autenticarse. En despliegues con HTTP en lugar de HTTPS, las opciones que aún se permiten son manipulables según el PR #16696.

## Procedimiento recomendado

La versión objetivo para todos los clientes es la 1.5.3. Puesto que, según el PR #16755, las versiones anteriores no pueden establecer conexión con las tuberías IPC reforzadas hasta actualizarse, actualice conjuntamente todos los componentes de RustDesk de un dispositivo en Windows. Antes del despliegue, compruebe si distribuye direcciones de servidor o claves mediante estrategias, si hbbs se ejecuta con `--mask` y si se utiliza F-Secure en los sistemas de destino. El servidor API debe ser accesible mediante HTTPS, como ya se recomendaba para la 1.5.1. El servidor de código abierto no necesita actualizarse para este lanzamiento. Las notas de lanzamiento no incluyen información sobre plazos de soporte (a fecha de 11 de noviembre de 2026).

## Fuentes

1.  [GitHub: lanzamiento de RustDesk 1.5.3](https://github.com/rustdesk/rustdesk/releases/tag/1.5.3): notas de lanzamiento con fecha, nueva función, cambio y correcciones de errores.

2.  [GitHub: lanzamientos de RustDesk](https://github.com/rustdesk/rustdesk/releases): lista de versiones sin lanzamiento 1.5.2.

3.  [GitHub: PR #16784, request approval](https://github.com/rustdesk/rustdesk/pull/16784): proceso, parámetros y límites de la conexión con aprobación.

4.  [GitHub: PR #16858, relay solo a direcciones ofrecidas](https://github.com/rustdesk/rustdesk/pull/16858): lista de permitidos para direcciones de relay, validación de UUID y limitación con `--mask`.

5.  [GitHub: PR #16755, Windows IPC server auth](https://github.com/rustdesk/rustdesk/pull/16755): protección de las Named Pipes, compatibilidad y riesgo restante.

6.  [GitHub: PR #16696, claves bloqueadas en estrategias](https://github.com/rustdesk/rustdesk/pull/16696): opciones bloqueadas y aún permitidas, falta de autenticación de las respuestas.

7.  [GitHub: PR #16889, eliminación de la transferencia por tubería en el instalador](https://github.com/rustdesk/rustdesk/pull/16889): relación con F-Secure DeepGuard y comportamiento tras una consulta UAC cancelada.

8.  [GitHub: PR #16908, protección del directorio del instalador](https://github.com/rustdesk/rustdesk/pull/16908): directorios de staging propios con handles abiertos.

9.  [GitHub: PR #16871, staging del instalador con Junction](https://github.com/rustdesk/rustdesk/pull/16871): uso de ProgramData cuando Windows Temp está redirigido.

10.  [GitHub: RustDesk Security Advisories](https://github.com/rustdesk/rustdesk/security/advisories): no hay advisories publicados (a fecha de 11 de noviembre de 2026).

11.  [GitHub: lanzamientos de rustdesk-server](https://github.com/rustdesk/rustdesk-server/releases): última versión del servidor 1.1.16, del 20 de julio de 2026.

12.  [RustDesk 1.5.1: actualizaciones firmadas, Trusted Devices V2 y refuerzo de la transferencia de archivos](/blog/rustdesk-1-5-1): estado del lanzamiento anterior.
