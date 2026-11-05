---
title: "RustDesk 1.5.1: actualizaciones firmadas, Trusted Devices V2 y refuerzo de la transferencia de archivos"
navTitle: "RustDesk 1.5.1"
description: "RustDesk 1.5.1 verifica las actualizaciones de escritorio mediante metadatos firmados, introduce Trusted Devices V2 y refuerza la transferencia de archivos, el instalador de Windows y el inicio de sesión automático. Además, incluye unas 35 correcciones de errores, principalmente para la captura DRM y Wayland. Se recomienda actualizar los clientes; el servidor permanece sin cambios."
date: "2026-11-02"
kategorie: "Mantenimiento remoto y soporte"
timeToRead: "5 min de lectura"
themen:
  - fernwartung
produkte:
  - "rustdesk"
protokolle:
  - "releases"
slug: "rustdesk-1-5-1-actualizaciones-firmadas-trusted-devices-v2-y-refuerzo-de-la-transferencia-de"
translationId: article-c95ef39dfcc09875
translationOf: rustdesk-1-5-1
url: https://rafaelpfister.ch/es/blog/rustdesk-1-5-1-actualizaciones-firmadas-trusted-devices-v2-y-refuerzo-de-la-transferencia-de
translationSourceHash: aa77142cfee63f49b15c18d6d1aa69368ec9e6ac75dee970c2907402fc21fc91
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:35:28.194Z
translationReview: automatic
---

# RustDesk 1.5.1: actualizaciones firmadas, Trusted Devices V2 y refuerzo de la transferencia de archivos

RustDesk publicó la versión 1.5.1 del cliente el 1 de noviembre de 2026, poco más de un mes después de la 1.5.0 (30 de septiembre de 2026). La versión incorpora cinco funciones nuevas, entre ellas la comprobación de metadatos de actualización firmados y una nueva versión de Trusted Devices para la autenticación de dos factores, tres cambios y unas 35 correcciones de errores. Varias de ellas refuerzan procesos relevantes para la seguridad: validación de rutas de descarga, transferencia protegida del instalador en Windows, advertencia sobre servidores API sin cifrar y un inicio de sesión automático más restringido. El fabricante no menciona ninguna CVE ni Security Advisory al respecto (a 2 de noviembre de 2026). La versión está dirigida a todos quienes utilizan clientes RustDesk en Windows, macOS, Linux, Android o iOS, especialmente con servidor propio. La configuración y el funcionamiento de un servidor RustDesk propio se describen en el artículo [Configurar RustDesk: la alternativa de código abierto a TeamViewer](/blog/rustdesk-teamviewer-alternative).

## Las versiones de un vistazo

| Versión | Fecha | Tipo | Puntos clave |
|---|---|---|---|
| 1.5.1 | 01.11.2026 | Versión de cliente | metadatos de actualización firmados, Trusted Devices V2, atajos de teclado, validación de rutas de descarga, refuerzo del instalador en Windows, advertencia sobre servidor API HTTP, correcciones para DRM, Wayland, X11 y QSV |

El servidor de código abierto (`rustdesk-server`) no se ve afectado por esta versión. Su última versión es la 1.1.16, del 20 de julio de 2026.

## Metadatos de actualización firmados

Con el PR #15914, el actualizador de escritorio en Windows y macOS obtiene su información de metadatos de versión firmados. Cada versión incluye los recursos `rustdesk-update.json` (versión, URL, nombre de archivo, tamaño y SHA-256 por artefacto) y `rustdesk-update.json.sig` (firma Ed25519). El cliente verifica la firma antes de evaluar los metadatos y solo acepta URL de descarga bajo la versión de `rustdesk/rustdesk`. Si la comprobación falla, la actualización se cancela; no hay alternativa con datos sin firmar. Los instaladores descargados se conservan en un directorio protegido hasta su inicio y se vuelven a hashear antes de ejecutarse con privilegios elevados.

Según el PR, la firma acredita autenticidad e integridad, pero no actualidad: no incluye protección contra la reinstalación de versiones anteriores correctamente firmadas (rollback). La variante portátil en Windows sigue descomprimiéndose e iniciándose desde un directorio modificable por el usuario; según el PR, su refuerzo se ha trasladado a una rama independiente.

## Trusted Devices V2

El PR #16548 sustituye la confianza anterior basada en el ID de hardware por tokens emitidos por el host. Para un nuevo registro siguen siendo necesarios la contraseña permanente y TOTP; después, un token de Trusted Device solo evita introducir el TOTP. El host almacena únicamente el hash SHA-256 del token de forma cifrada, y el controlador vincula su token a la clave pública verificada del host. En conexiones IP directas no se ofrece confianza, ya que allí no se verifica la identidad del host.

Para la operación, son relevantes los siguientes puntos del PR:

- **Validez:** configurable en 1, 7, 30 o 90 días, o ilimitada; el valor predeterminado es 90 días. Las relaciones de confianza pueden renovarse, eliminarse individualmente o borrarse en conjunto desde la configuración.
- **Revocación:** Un cambio de contraseña, 2FA o de la configuración de Trusted Device revoca los tokens afectados. El host mantiene como máximo 128 entradas.
- **Versiones mixtas:** Las relaciones de confianza existentes y no caducadas siguen siendo válidas tras actualizar el host. Los nuevos registros requieren la versión 1.5.1 en ambos lados; en sesiones con versiones mixtas, RustDesk solicita TOTP y no ofrece «Trust this device».
- **Downgrade:** Según el PR, no es seguro, ya que las versiones anteriores no pueden leer las relaciones de confianza importadas.
- **Limitaciones conocidas:** Una revocación se guarda primero localmente y se incorpora de forma asíncrona a la configuración raíz; un reinicio antes de la sincronización puede restaurar una relación de confianza anterior. La fecha de caducidad depende del reloj del sistema del host.

Por ello, planifique la actualización de modo que hosts y controladores pasen a la 1.5.1 en un plazo breve, y evite volver a la 1.5.0.

## Otras correcciones relevantes para la seguridad

- **Rutas de descarga (PR #16601):** Las descargas rechazan nombres de archivo y rutas inseguros que salgan del directorio de destino seleccionado o pasen por enlaces simbólicos. Las listas de archivos remotos se verifican antes de la descarga; sobrescribir archivos existentes y reanudar descargas interrumpidas requieren confirmación.
- **Instalador de Windows (PR #16554, #16562):** Los scripts para transferir el instalador con privilegios elevados ya no se encuentran en `System32`, sino en un directorio bajo Windows Temp con DACL restringida (Administrators y SYSTEM). El servicio se controla mediante el control nativo de servicios. Uno de los motivos fue que Bitdefender bloqueaba el script Batch temporal (Issue #16557).
- **Servidor API sin cifrar (PR #16616):** La configuración del servidor muestra una advertencia cuando el servidor API está vacío o utiliza una dirección HTTP remota. HTTPS y las direcciones locales no generan advertencia. La conexión no se bloquea.
- **Inicio de sesión automático (PR #16520):** El inicio de sesión automático solo introduce la contraseña del sistema operativo en la pantalla de inicio de sesión o de bloqueo.
- **Privacy Mode (PR #16507):** Una solicitud de Privacy Mode rechazada ya no desactiva el Privacy Mode de otra conexión.

Si utiliza un servidor propio con una dirección `http://` como servidor API, aparecerá la nueva advertencia para sus usuarios. Cambie el servidor API a HTTPS antes de distribuir la versión 1.5.1 para evitar consultas.

## Nuevas funciones y cambios

Además de los puntos ya mencionados, la 1.5.1 incorpora atajos de teclado (PR #14933), un botón en la libreta de direcciones para abrir la consola web y, en Android, transferencia directa de archivos a carpetas SAF. Se modificaron la codificación de contenidos de imagen estáticos tras el desplazamiento (representación más nítida) y los presupuestos de actualización para MediaCodec. Se eliminó el procesamiento de `ConfigUpdate`.

## Correcciones de errores por plataforma

- **Linux, captura DRM:** tamaño correcto en pantallas rotadas, no se activan pantallas en reposo al iniciar, descarte de fotogramas duplicados y frecuencia de imagen adaptada al codificador.
- **Linux, Wayland:** captura PipeWire sin tiempos de espera, asignación correcta del flujo del portal a la salida, distribución del teclado durante la entrada, y una solicitud abierta del portal ya no bloquea el servidor.
- **Linux, general:** usuario activo correcto en Flatpak, bloqueo al cerrar la ventana GTK, solicitud de sudo sin el incorrecto «Sorry, try again.» y con soporte para sudo-rs, nombres de salida de xrandr, consulta de monitores RandR bajo X11.
- **Windows:** lectura de `host=` desde rutas de programa con caracteres no ASCII, ejecutable nombrado explícitamente para la ventana Privacy, bloqueos del codificador QSV.
- **macOS:** enlace a la configuración de Screen Recording, soltar el botón del ratón.
- **Móvil:** ya no se produce un Mouse-Down tardío tras soltar el botón flotante.
- **Red:** error HTTP API 407 mediante proxies HTTPS, servidores de ID IPv6 sin corchetes al derivar el servidor API, un mensaje de error en lugar de dos cuando fallan ambos intentos de conexión WebRTC.
- **Otros:** se conservan las impresoras eliminadas, búfer de repetición en el terminal, navegación por teclado en la lista de archivos, carga de pares sin directorio de aplicación establecido, clics en tarjetas de pares en el cliente web.

## Procedimiento recomendado

La versión objetivo para todos los clientes es la 1.5.1. Actualice hosts y controladores, en la medida de lo posible, durante el mismo periodo para que funcionen los nuevos registros de Trusted Device; según el PR #16548, un downgrade a la 1.5.0 no es seguro. Antes del despliegue, compruebe si su propio servidor API es accesible mediante HTTPS. El servidor de código abierto no necesita actualizarse para esta versión. Las notas de la versión no contienen información sobre Known Issues ni plazos de soporte (a 2 de noviembre de 2026).

## Fuentes

1.  [GitHub: RustDesk Release 1.5.1](https://github.com/rustdesk/rustdesk/releases/tag/1.5.1): notas de la versión con fecha, nuevas funciones, cambios y correcciones de errores.

2.  [GitHub: RustDesk Release 1.5.0](https://github.com/rustdesk/rustdesk/releases/tag/1.5.0): versión anterior del 30 de septiembre de 2026.

3.  [GitHub: PR #15914, verify signed update metadata](https://github.com/rustdesk/rustdesk/pull/15914): proceso de verificación de firma, falta de protección contra rollback, variante portátil.

4.  [GitHub: PR #16548, Trusted Devices V2](https://github.com/rustdesk/rustdesk/pull/16548): modelo de token, compatibilidad con versiones anteriores, downgrade y limitaciones conocidas.

5.  [GitHub: PR #16601, Download path validation](https://github.com/rustdesk/rustdesk/pull/16601): verificación de nombres de archivo, rutas y enlaces simbólicos en descargas.

6.  [GitHub: PR #16554, scripts del instalador en directorios Temp protegidos](https://github.com/rustdesk/rustdesk/pull/16554): traslado desde System32, DACL y relación con Bitdefender.

7.  [GitHub: PR #16616, advertencia sobre servidor API inseguro](https://github.com/rustdesk/rustdesk/pull/16616): condiciones y texto de la nueva advertencia.

8.  [GitHub: RustDesk Security Advisories](https://github.com/rustdesk/rustdesk/security/advisories): no hay advisories publicados (a 2 de noviembre de 2026).

9.  [GitHub: rustdesk-server Releases](https://github.com/rustdesk/rustdesk-server/releases): última versión del servidor 1.1.16, del 20 de julio de 2026.

10.  [RustDesk: configurar la alternativa de código abierto a TeamViewer](/blog/rustdesk-teamviewer-alternative): configuración del cliente y del servidor propio.
