---
title: "Proton Drive CLI: gestionar Proton Drive desde scripts y servidores"
navTitle: "Proton Drive CLI"
description: "Desde junio de 2026, Proton ofrece una herramienta oficial de línea de comandos para Proton Drive. El artículo describe los comandos, el inicio de sesión en servidores sin escritorio, las estrategias de conflictos para scripts y las limitaciones frente a Rclone."
date: "2026-10-01"
kategorie: "Proton Drive"
timeToRead: "9 min de lectura"
themen:
  - proton-drive
produkte:
  - "proton-drive"
protokolle:
  - "storage"
  - "backup-dr"
related:
  - proton-drive-linux-status
  - rclone-mount-in-docker-container
slug: "proton-drive-cli-controlar-proton-drive-desde-scripts-y-servidores"
translationId: "article-97376b998fdaec4c"
translationOf: proton-drive-cli
translationSourceHash: ce73cccdecdcdea611f1ead28872dab00bc33a7106e2eaefd51bee4241d8631c
translationModel: gpt-5.6-terra
translatedAt: 2026-10-27T09:16:15.071Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/proton-drive-cli-controlar-proton-drive-desde-scripts-y-servidores
---

El 9 de junio de 2026, Proton publicó la **Proton Drive CLI**, una herramienta oficial de línea de comandos para Windows, macOS y Linux. Se basa en el mismo SDK que las aplicaciones oficiales de Drive, cifra de extremo a extremo y está disponible como un único archivo ejecutable `proton-drive`. El código fuente se encuentra en el repositorio público del SDK en `cli/`.

La CLI está pensada para operaciones individuales y puntuales: subir archivos tras una compilación, hacer una copia de seguridad programada de una carpeta, comprobar o revocar comparticiones. No sincroniza en segundo plano ni monta un sistema de archivos. Actualmente está disponible la versión **0.9.0 del 5 de octubre de 2026**; el número de versión indica que los comandos y las opciones todavía pueden cambiar (la versión 0.8.0 renombró las estrategias de conflictos con un cambio incompatible).

La clasificación respecto a las demás opciones para Linux (Rclone, cliente de escritorio anunciado) se encuentra en el artículo de estado [Proton Drive en Linux](/blog/proton-drive-linux-status).

## Resumen de comandos

Los comandos se organizan en grupos: `proton-drive <gruppe> <befehl> [optionen] [argumente]`. Los nombres de grupo se pueden abreviar siempre que sigan siendo inequívocos; para `filesystem` existe además el alias `fs`. Sin argumentos, se inicia una shell interactiva. La ayuda completa se obtiene con `proton-drive help` o `proton-drive <gruppe> <befehl> --help`.

<details class="options-details">
<summary>Resumen de opciones</summary>

| Grupo / comando | Función |
|---|---|
| `auth login` / `auth logout` | Inicio de sesión mediante el navegador; el cierre de sesión elimina las credenciales locales y las cachés |
| `filesystem list <pfad>` | Mostrar el contenido de una carpeta; `/` muestra las áreas raíz |
| `filesystem info` / `size` | Metadatos de un elemento o tamaño de una carpeta, incluido el contenido de la papelera |
| `filesystem upload` / `download` | Subir o descargar archivos y carpetas |
| `filesystem create-folder`, `rename`, `copy`, `move` | Crear, renombrar, copiar y mover carpetas |
| `filesystem trash` / `restore` | Mover a la papelera o restaurar |
| `filesystem delete` / `empty-trash` | Eliminar permanentemente o vaciar `/trash` |
| `sharing status <pfad>` | Mostrar miembros, invitaciones abiertas y configuración de enlaces |
| `sharing invite` / `remove` | Invitar a personas por correo electrónico o revocar el acceso |
| `sharing set-url` / `remove-url` | Crear, modificar o eliminar un enlace público |
| `sharing leave` / `report` | Abandonar una compartición recibida o denunciarla como abuso |
| `invitation list` / `accept` / `reject` | Gestionar invitaciones recibidas |
| `album …`, `photo timeline`, `photo upload`, `photo download` | Proton Photos: álbumes y línea de tiempo |
| `takeout run` | Exportar una copia sin conexión de la cuenta a una carpeta local (desde 0.9.0) |
| `version` | Mostrar versiones de la CLI y del SDK |
| `--json` (`-j`) | Salida JSON legible por máquinas, en cualquier comando |
| `--verbose` (`-v`) | Salida de registro directamente en la consola |
| `--help` (`-h`) | Ayuda sobre el comando correspondiente |

</details>

Las rutas de Proton Drive siempre son rutas POSIX, también en Windows. La raíz `/` contiene áreas virtuales: `/my-files` (archivos propios), `/devices` (equipos respaldados), `/shared-by-me`, `/shared-with-me`, `/trash` y las áreas de Photos `/photos`, `/albums`, `/photos-shared-by-me`, `/photos-shared-with-me` y `/photos-trash`.

## Instalación en Linux

Proton proporciona las compilaciones en una página de descargas propia, cada una con suma de comprobación SHA-512. Para Linux hay cinco variantes:

| Compilación | Uso |
|---|---|
| `linux/x64` | Estándar para sistemas x86-64 actuales |
| `linux/x64-baseline` | x86-64 sin AVX2, por ejemplo, dispositivos NAS y CPU de servidor antiguas |
| `linux/arm64` | Servidores ARM y ordenadores monoplaca con glibc |
| `linux/x64-musl`, `linux/arm64-musl` | Distribuciones con musl en lugar de glibc, por ejemplo Alpine Linux e imágenes de contenedor basadas en ella |

Si la compilación estándar se interrumpe al iniciarse con `Illegal instruction`, a la CPU le falta la extensión AVX2; entonces la compilación `x64-baseline` es la elección correcta. El archivo incorpora el entorno de ejecución Bun y no necesita más dependencias:

```bash
chmod +x proton-drive
sudo install -m 0755 proton-drive /usr/local/bin/proton-drive
proton-drive version
```

Sin derechos de administrador basta con copiar el archivo a `~/.local/bin` si este directorio está en `PATH`.

## Inicio de sesión, también en servidores sin escritorio

`auth login` no solicita la contraseña en la línea de comandos. La CLI intenta abrir un navegador y también muestra la URL de inicio de sesión. Esta URL se puede abrir **en otro dispositivo**; el terminal espera hasta que allí se complete el inicio de sesión. La autenticación de dos factores se realiza normalmente en el navegador. De este modo, el inicio de sesión también funciona mediante SSH en un servidor sin interfaz gráfica.

```bash
proton-drive auth login
```

Tras iniciar sesión correctamente, la CLI guarda la sesión, no la contraseña. La variable de entorno `PROTON_DRIVE_CREDENTIALS_STORE` determina dónde se almacena:

| Valor | Ubicación de almacenamiento |
|---|---|
| `keychain` (estándar) | Almacén de claves del sistema operativo: Windows Credential Manager, macOS Keychain y, en Linux, libsecret (GNOME Keyring, KWallet) |
| `pass` | Entrada cifrada con GPG `ch.proton.drive/drive-sdk-cli/auth-session` en el gestor de contraseñas [pass](https://www.passwordstore.org/) |
| `unsafe_file` | Archivo de texto sin cifrar `auth-session.json` en el directorio de datos; según Proton, solo para pruebas |

En un servidor sin sesión de escritorio, normalmente no hay un llavero libsecret desbloqueado. Para este caso existe, desde la versión 0.6.0, la opción `pass`. El usuario con el que se ejecutan los scripts necesita para ello un almacén de contraseñas inicializado y una clave GPG que `gpg-agent` pueda desbloquear sin entrada interactiva. La sesión debe encontrarse mediante la misma variable en cada llamada, por lo que también debe establecerse en trabajos de Cron y unidades de systemd:

```bash
export PROTON_DRIVE_CREDENTIALS_STORE=pass
proton-drive auth login
```

Esto supone una mejora frente a Rclone: la contraseña y la clave TOTP no se almacenan en el servidor, y `auth logout` finaliza el acceso. Sin embargo, la sesión sigue teniendo el alcance completo de la cuenta. No existe ninguna restricción a carpetas individuales ni acceso de solo lectura. Por ello, para procesos automatizados sigue siendo más seguro utilizar una cuenta de Proton propia.

La caché, los datos de la aplicación y los registros se encuentran en Linux en los directorios XDG (`~/.cache/proton-drive-cli`, `~/.local/share/proton-drive-cli`, `~/.local/state/proton-drive-cli`). Con `PROTON_DRIVE_CACHE_DIR` se pueden reunir los tres en un único directorio, por ejemplo para un contenedor con un volumen montado. De forma predeterminada, la CLI escribe los registros con el nivel `DEBUG`; `PROTON_DRIVE_LOG_LEVEL=WARNING` reduce la cantidad.

## Subir y descargar en scripts

De forma interactiva, la CLI pregunta qué hacer ante cada conflicto de nombre. En scripts esto no es posible: con `--json` se desactiva la consulta interactiva. Por tanto, establezca siempre de forma explícita la estrategia de conflictos para archivos y carpetas.

```bash
proton-drive filesystem upload --json \
  --file-conflict-strategy create-new-revision \
  --folder-conflict-strategy merge \
  --skip-thumbnails \
  /srv/export/berichte /my-files/backup
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Función |
|---|---|
| `--json` (`-j`) | Emitir el resultado como JSON; desactiva las consultas interactivas |
| `--file-conflict-strategy` (`-f`) | Comportamiento si existe un archivo con el mismo nombre: `create-new-revision` (nueva versión del archivo existente), `rename` (añadir sufijo), `replace` (mover el archivo remoto a la papelera y subir el local), `skip` |
| `--folder-conflict-strategy` (`-d`) | Comportamiento si ya existe la carpeta: `merge` (fusionar contenidos), `rename`, `replace`, `skip` |
| `--skip-thumbnails` (`-t`) | No generar miniaturas; ahorra tiempo de procesamiento en imágenes |
| `/srv/export/berichte` | Origen local; se admiten varios orígenes |
| `/my-files/backup` | Carpeta de destino en Proton Drive (último argumento) |

</details>

`create-new-revision` es la opción adecuada para copias de seguridad: Proton Drive conserva las versiones anteriores de un archivo y, desde la versión 0.7.0, la CLI omite automáticamente los archivos con contenido sin cambios. Sin embargo, la CLI no sincroniza: los archivos eliminados localmente permanecen en Proton Drive. Quien necesite un espejo con eliminaciones sigue dependiendo de `rclone sync`.

La descarga funciona de forma análoga. Las estrategias difieren porque aquí se sobrescribe el lado local:

```bash
proton-drive filesystem download --json \
  --file-conflict-strategy remove \
  --folder-conflict-strategy merge \
  /my-files/backup/berichte /srv/restore
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Función |
|---|---|
| `--file-conflict-strategy` (`-f`) | `rename`, `remove` (eliminar el archivo local y descargar la versión remota) o `skip` |
| `--folder-conflict-strategy` (`-d`) | `merge`, `rename`, `remove` o `skip` |
| `/my-files/backup/berichte` | Origen en Proton Drive; se admiten varios orígenes |
| `/srv/restore` | Carpeta de destino local (último argumento) |

</details>

La CLI omite Proton Docs y Proton Sheets al descargar; actualmente no se pueden exportar como archivos.

Una subida periódica puede programarse con un temporizador de systemd o Cron. Después se puede evaluar la salida JSON con `jq`, por ejemplo para enviar un aviso al sistema de monitorización.

## Gestionar comparticiones

Para bajas de personal o auditorías, la gestión de comparticiones suele ser más útil que la transferencia de archivos. `sharing status` muestra para un elemento todos los miembros, las invitaciones abiertas y la configuración de un enlace público:

```bash
proton-drive sharing status --json /my-files/projekte/kunde-a
```

Una invitación con permisos de lectura:

```bash
proton-drive sharing invite \
  --user person@example.com \
  --role viewer \
  /my-files/projekte/kunde-a
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Función |
|---|---|
| `--user` (`-u`) | Dirección de correo electrónico de la persona invitada; se puede indicar varias veces |
| `--role` (`-r`) | Rol, `viewer` de forma predeterminada; otros roles según `--help` (p. ej., `editor`) |
| `--message` (`-m`) | Mensaje en el correo de invitación; se envía en **texto sin cifrar** |
| `--include-node-name` (`-n`) | Incluir el nombre del elemento en el correo de invitación; también en texto sin cifrar |
| `/my-files/projekte/kunde-a` | Elemento que se compartirá |

</details>

Un enlace público con contraseña y fecha de caducidad:

```bash
proton-drive sharing set-url \
  --role viewer \
  --password 'Linkpasswort' \
  --expiration 2026-12-31 \
  /my-files/projekte/kunde-a/bericht.pdf
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Función |
|---|---|
| `--role` | `viewer` (predeterminado) o `editor` |
| `--password` | Contraseña propia para el enlace |
| `--expiration` | Fecha de caducidad en formato ISO (`JJJJ-MM-TT`) |
| `/my-files/…/bericht.pdf` | Elemento para el que se crea o modifica el enlace |

</details>

Una contraseña proporcionada en la línea de comandos queda en el historial de la shell y es visible en la lista de procesos durante la ejecución. Por ello, en scripts debe proceder de una variable o de un almacén de secretos. `sharing remove-url` vuelve a eliminar el enlace sin afectar a los miembros directos; `sharing remove --user …` revoca el acceso de personas concretas.

## Takeout: exportación de la cuenta

Desde la versión 0.9.0 del 5 de octubre de 2026, la CLI incluye el comando `takeout run`. Exporta una copia sin conexión de la cuenta a una carpeta local, opcionalmente con `--include my-files`, `devices`, `photos` y `revisions` (todas las versiones anteriores de los archivos). Sin al menos una indicación `--include` no exporta nada, y `revisions` requiere `my-files` o `devices`. Para cada carpeta escribe un archivo `manifest.json` que describe la exportación; no modifica nada en la cuenta. El comando todavía no admite Proton Docs ni Proton Sheets. Para una copia de seguridad local completa del contenido de Proton Drive, es la vía más directa.

## Limitaciones frente a Rclone

| Requisito | Proton Drive CLI 0.9.0 | Rclone (backend `protondrive`) |
|---|---|---|
| Compatibilidad oficial | Sí, por Proton, código abierto | No, ingeniería inversa, beta |
| Inicio de sesión | Navegador, también en otro dispositivo; sesión en el almacén de claves o `pass` | Contraseña y clave TOTP en el archivo de configuración |
| Subida, descarga | Sí, con estrategias de conflictos y versionado | Sí |
| Espejo con eliminaciones (`sync`) | No | Sí |
| Montar sistema de archivos (FUSE) | No | Sí |
| Comparticiones, invitaciones, enlaces | Sí | No |
| Proton Photos | Sí | No |
| Acceso de alcance limitado | No | No |

Para copias de seguridad, artefactos de compilación y gestión de comparticiones, la CLI es la mejor opción porque cuenta con soporte oficial y no necesita una contraseña almacenada. Para un montaje, como el que necesita por ejemplo un [archivo documental Paperless](/blog/paperless-dokumente-clouddienst-auslagern), y para espejos con eliminaciones, Rclone sigue siendo necesario por ahora. La mayor carencia es la misma en ambos casos: Proton no ofrece acceso para máquinas que pueda limitarse a carpetas individuales o a acceso de lectura.

## Historial de cambios

- **25 de octubre de 2026:** Actualizado a la versión 0.9.0 del 5 de octubre de 2026. Añadido: el comando takeout run ya está publicado, incluidas indicaciones sobre las opciones --include necesarias y la falta de compatibilidad con Proton Docs y Sheets.

## Fuentes

1.  [Proton: Introducing Proton Drive CLI](https://proton.me/blog/proton-drive-cli): el anuncio del 9 de junio de 2026 con casos de uso y salida JSON.

2.  [Proton Support: Using Proton Drive CLI](https://proton.me/support/drive-cli): guía para la descarga, el inicio de sesión y los comandos básicos.

3.  [Proton Drive CLI: Downloads](https://proton.me/download/drive/cli/index.html): versión actual 0.9.0, todas las compilaciones de plataforma con sumas de comprobación SHA-512.

4.  [ProtonDriveApps/sdk: cli/README.md](https://github.com/ProtonDriveApps/sdk/blob/main/cli/README.md): variables de entorno, ubicaciones de almacenamiento, almacenes de credenciales y la nota sobre la compilación `x64-baseline`.

5.  [ProtonDriveApps/sdk: cli/CHANGELOG.md](https://github.com/ProtonDriveApps/sdk/blob/main/cli/CHANGELOG.md): historial de versiones de 0.4.2 a 0.9.0, incluido el soporte de `pass` (0.6.0), la omisión de archivos sin cambios (0.7.0) y el comando Takeout (0.9.0).

6.  [ProtonDriveApps/sdk: cli/src/commands](https://github.com/ProtonDriveApps/sdk/tree/main/cli/src/commands): código fuente de los comandos con opciones, estrategias de conflictos y el comando Takeout.

7.  [Proton for Business: Proton Drive CLI](https://proton.me/business/drive/cli): casos de uso de Proton para empresas, como revocar comparticiones cuando empleados dejan la empresa.

8.  [Rclone: Proton Drive](https://rclone.org/protondrive/): el backend de la comunidad con funciones de montaje y sincronización como comparación.
