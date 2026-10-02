---
title: "Proton Drive CLI: controlar Proton Drive desde scripts y servidores"
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
url: https://rafaelpfister.ch/es/blog/proton-drive-cli-controlar-proton-drive-desde-scripts-y-servidores
translationSourceHash: e71e82deea6466312d0d95edc2c96890fc0810219a21308b4cb9346df3f32341
translationModel: gpt-5.6-terra
translatedAt: 2026-10-02T09:59:31.082Z
translationReview: automatic
---

El 9 de junio de 2026, Proton publicó **Proton Drive CLI**, una herramienta oficial de línea de comandos para Windows, macOS y Linux. Se basa en el mismo SDK que las aplicaciones oficiales de Drive, cifra de extremo a extremo y está disponible como un único archivo ejecutable `proton-drive`. El código fuente se encuentra en el repositorio público del SDK en `cli/`.

La CLI está pensada para operaciones individuales y puntuales: subir archivos después de una compilación, realizar una copia de seguridad programada de una carpeta, comprobar o revocar comparticiones. No sincroniza en segundo plano ni monta un sistema de archivos. Actualmente la versión es la **0.8.0 del 13 de agosto de 2026**; el número de versión indica que los comandos y las opciones aún pueden cambiar (la versión 0.8.0 renombró las estrategias de conflicto con un cambio incompatible).

La clasificación entre las demás opciones de Linux (Rclone, cliente de escritorio anunciado) se explica en el artículo de estado [Proton Drive en Linux](/blog/proton-drive-linux-status).

## Resumen de comandos

Los comandos están organizados en grupos: `proton-drive <gruppe> <befehl> [optionen] [argumente]`. Los nombres de grupo pueden abreviarse siempre que sigan siendo inequívocos; para `filesystem` existe además el alias `fs`. Sin argumentos se inicia una shell interactiva. La ayuda completa se obtiene con `proton-drive help` o `proton-drive <gruppe> <befehl> --help`.

<details class="options-details">
<summary>Resumen de opciones</summary>

| Grupo / comando | Función |
|---|---|
| `auth login` / `auth logout` | Inicio de sesión mediante el navegador; el cierre de sesión elimina los datos de inicio de sesión y las cachés locales |
| `filesystem list <pfad>` | Lista el contenido de una carpeta; `/` muestra las áreas raíz |
| `filesystem info` / `size` | Metadatos de un elemento o tamaño de una carpeta, incluido el contenido de la papelera |
| `filesystem upload` / `download` | Sube o descarga archivos y carpetas |
| `filesystem create-folder`, `rename`, `copy`, `move` | Crea carpetas, cambia nombres, copia y mueve |
| `filesystem trash` / `restore` | Mueve a la papelera o restaura |
| `filesystem delete` / `empty-trash` | Elimina definitivamente o vacía `/trash` |
| `sharing status <pfad>` | Muestra miembros, invitaciones pendientes y ajustes de enlaces |
| `sharing invite` / `remove` | Invita a personas por correo electrónico o revoca el acceso |
| `sharing set-url` / `remove-url` | Crea, modifica o elimina un enlace público |
| `sharing leave` / `report` | Abandona una compartición recibida o la denuncia como abuso |
| `invitation list` / `accept` / `reject` | Gestiona las invitaciones recibidas |
| `album …`, `photo timeline`, `photo upload`, `photo download` | Proton Photos: álbumes y cronología |
| `version` | Muestra las versiones de la CLI y del SDK |
| `--json` (`-j`) | Salida JSON legible por máquinas, para todos los comandos |
| `--verbose` (`-v`) | Salida de registro directamente en la consola |
| `--help` (`-h`) | Ayuda sobre el comando correspondiente |

</details>

Las rutas en Proton Drive siempre son rutas POSIX, también en Windows. La raíz `/` contiene áreas virtuales: `/my-files` (archivos propios), `/devices` (equipos con copia de seguridad), `/shared-by-me`, `/shared-with-me`, `/trash`, así como las áreas de Photos `/photos`, `/albums`, `/photos-shared-by-me`, `/photos-shared-with-me` y `/photos-trash`.

## Instalación en Linux

Proton proporciona las compilaciones en una página de descargas propia, cada una con suma de comprobación SHA-512. Para Linux hay cinco variantes:

| Compilación | Uso |
|---|---|
| `linux/x64` | Estándar para sistemas x86-64 actuales |
| `linux/x64-baseline` | x86-64 sin AVX2, por ejemplo dispositivos NAS y CPU de servidor antiguas |
| `linux/arm64` | Servidores ARM y ordenadores monoplaca con glibc |
| `linux/x64-musl`, `linux/arm64-musl` | Distribuciones con musl en lugar de glibc, por ejemplo Alpine Linux e imágenes de contenedor basadas en ella |

Si la compilación estándar termina al iniciarse con `Illegal instruction`, la CPU no dispone de la extensión AVX2; en ese caso, la compilación `x64-baseline` es la opción correcta. El archivo incluye el entorno de ejecución Bun integrado y no necesita más dependencias:

```bash
chmod +x proton-drive
sudo install -m 0755 proton-drive /usr/local/bin/proton-drive
proton-drive version
```

Sin derechos de administrador, basta con copiar el archivo a `~/.local/bin` si este directorio está incluido en `PATH`.

## Inicio de sesión, también en servidores sin escritorio

`auth login` no solicita ninguna contraseña en la línea de comandos. La CLI intenta abrir un navegador y también muestra la URL de inicio de sesión. Esta URL se puede abrir **en otro dispositivo**; el terminal espera hasta que el inicio de sesión se complete allí. La autenticación de dos factores se realiza normalmente en el navegador. Así, el inicio de sesión también funciona mediante SSH en un servidor sin interfaz gráfica.

```bash
proton-drive auth login
```

Tras iniciar sesión correctamente, la CLI guarda la sesión, no la contraseña. Su ubicación la determina la variable de entorno `PROTON_DRIVE_CREDENTIALS_STORE`:

| Valor | Ubicación de almacenamiento |
|---|---|
| `keychain` (predeterminado) | Almacén de claves del sistema operativo: Windows Credential Manager, macOS Keychain y, en Linux, libsecret (GNOME Keyring, KWallet) |
| `pass` | Entrada cifrada con GPG `ch.proton.drive/drive-sdk-cli/auth-session` en el gestor de contraseñas [pass](https://www.passwordstore.org/) |
| `unsafe_file` | Archivo de texto sin cifrar `auth-session.json` en el directorio de datos; según Proton, solo para pruebas |

En un servidor sin sesión de escritorio, normalmente falta un llavero libsecret desbloqueado. Para este caso existe, desde la versión 0.6.0, la opción `pass`. El usuario con el que se ejecutan los scripts necesita para ello un almacén de contraseñas inicializado y una clave GPG que el `gpg-agent` pueda desbloquear sin entrada interactiva. La sesión debe encontrarse mediante la misma variable en cada llamada, por lo que también debe estar configurada en trabajos de Cron y unidades de systemd:

```bash
export PROTON_DRIVE_CREDENTIALS_STORE=pass
proton-drive auth login
```

Frente a Rclone, esto es una mejora: la contraseña y la clave TOTP no se almacenan en el servidor, y `auth logout` finaliza el acceso. Sin embargo, la sesión sigue teniendo acceso completo a la cuenta. No es posible limitarla a determinadas carpetas ni al acceso de solo lectura. Por ello, para procesos automatizados sigue siendo más seguro usar una cuenta de Proton independiente.

La caché, los datos de la aplicación y los registros se encuentran en Linux en los directorios XDG (`~/.cache/proton-drive-cli`, `~/.local/share/proton-drive-cli`, `~/.local/state/proton-drive-cli`). Con `PROTON_DRIVE_CACHE_DIR` se pueden colocar los tres en un único directorio, por ejemplo para un contenedor con un volumen montado. De forma predeterminada, la CLI escribe los registros con el nivel `DEBUG`; `PROTON_DRIVE_LOG_LEVEL=WARNING` reduce la cantidad.

## Subir y descargar en scripts

De forma interactiva, la CLI pregunta qué hacer ante cada conflicto de nombres. Esto no es posible en scripts: con `--json` se desactiva la consulta interactiva. Por ello, establezca siempre explícitamente la estrategia de conflicto para archivos y carpetas.

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
| `--json` (`-j`) | Devuelve el resultado como JSON; desactiva las consultas interactivas |
| `--file-conflict-strategy` (`-f`) | Comportamiento cuando existe un archivo con el mismo nombre: `create-new-revision` (nueva versión del archivo existente), `rename` (añade un sufijo), `replace` (archivo remoto a la papelera, se sube el local), `skip` |
| `--folder-conflict-strategy` (`-d`) | Comportamiento si la carpeta ya existe: `merge` (fusiona contenidos), `rename`, `replace`, `skip` |
| `--skip-thumbnails` (`-t`) | No genera miniaturas; ahorra tiempo de procesamiento con imágenes |
| `/srv/export/berichte` | Fuente local; se admiten varias fuentes |
| `/my-files/backup` | Carpeta de destino en Proton Drive (último argumento) |

</details>

`create-new-revision` es la elección adecuada para copias de seguridad: Proton Drive conserva las versiones anteriores de un archivo, y desde la versión 0.7.0 la CLI omite automáticamente los archivos cuyo contenido no ha cambiado. Sin embargo, la CLI no sincroniza: los archivos eliminados localmente se conservan en Proton Drive. Quien necesite un espejo con eliminaciones sigue dependiendo de `rclone sync`.

La descarga funciona de forma análoga. Las estrategias son diferentes porque aquí se sobrescribe el lado local:

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
| `--file-conflict-strategy` (`-f`) | `rename`, `remove` (elimina el archivo local y descarga la versión remota) o `skip` |
| `--folder-conflict-strategy` (`-d`) | `merge`, `rename`, `remove` o `skip` |
| `/my-files/backup/berichte` | Fuente en Proton Drive; se admiten varias fuentes |
| `/srv/restore` | Carpeta de destino local (último argumento) |

</details>

La CLI omite Proton Docs y Proton Sheets al descargar; actualmente no se pueden exportar como archivos.

Una subida periódica se puede programar con un temporizador de systemd o Cron. Después, la salida JSON se puede evaluar con `jq`, por ejemplo para enviar un aviso al sistema de monitorización.

## Gestionar comparticiones

Para bajas de personal o auditorías, la gestión de comparticiones suele ser más útil que la transferencia de archivos. `sharing status` muestra todos los miembros, invitaciones pendientes y la configuración de un enlace público de un elemento:

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
| `--role` (`-r`) | Rol, predeterminado `viewer`; otros roles según `--help` (p. ej., `editor`) |
| `--message` (`-m`) | Mensaje en el correo de invitación; se envía en **texto sin cifrar** |
| `--include-node-name` (`-n`) | Incluye el nombre del elemento en el correo de invitación; también en texto sin cifrar |
| `/my-files/projekte/kunde-a` | Elemento que se va a compartir |

</details>

Un enlace público con contraseña y fecha de vencimiento:

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
| `--expiration` | Fecha de vencimiento en formato ISO (`JJJJ-MM-TT`) |
| `/my-files/…/bericht.pdf` | Elemento para el que se crea o modifica el enlace |

</details>

Una contraseña pasada en la línea de comandos queda en el historial de la shell y es visible en la lista de procesos durante la ejecución. Por ello, en scripts debería proceder de una variable o de un almacén de secretos. `sharing remove-url` vuelve a eliminar el enlace sin afectar a los miembros directos; `sharing remove --user …` revoca el acceso a personas concretas.

## En desarrollo: Takeout

Desde el 10 de septiembre de 2026, el repositorio del SDK contiene otro comando, `takeout run`. Exporta una copia sin conexión de la cuenta a una carpeta local, opcionalmente con `--include my-files`, `devices`, `photos` y `revisions` (todas las versiones anteriores de archivos). En cada carpeta escribe un archivo `manifest.json` que describe la exportación; no modifica nada en la cuenta. El comando aún no está incluido en la versión publicada 0.8.0. Cuando aparezca, será la vía natural para una copia de seguridad local completa del contenido de Proton Drive.

## Limitaciones frente a Rclone

| Requisito | Proton Drive CLI 0.8.0 | Rclone (backend `protondrive`) |
|---|---|---|
| Compatibilidad oficial | Sí, por Proton, código abierto | No, ingeniería inversa, beta |
| Inicio de sesión | Navegador, también en otro dispositivo; sesión en el almacén de claves o `pass` | Contraseña y clave TOTP en el archivo de configuración |
| Subir, descargar | Sí, con estrategias de conflicto y versionado | Sí |
| Espejo con eliminaciones (`sync`) | No | Sí |
| Montar sistema de archivos (FUSE) | No | Sí |
| Comparticiones, invitaciones, enlaces | Sí | No |
| Proton Photos | Sí | No |
| Acceso de alcance limitado | No | No |

Para copias de seguridad, artefactos de compilación y gestión de comparticiones, la CLI es la mejor opción porque cuenta con soporte oficial y no requiere una contraseña almacenada. Para un montaje, como el que necesita por ejemplo un [archivo documental de Paperless](/blog/paperless-dokumente-clouddienst-auslagern), y para espejos con eliminaciones, Rclone sigue siendo necesario por ahora. La mayor carencia sigue siendo la misma en ambos casos: Proton no ofrece acceso de máquina que pueda limitarse a carpetas concretas o al acceso de solo lectura.

## Fuentes

1.  [Proton: Introducing Proton Drive CLI](https://proton.me/blog/proton-drive-cli): el anuncio del 9 de junio de 2026 con el propósito de uso y la salida JSON.

2.  [Proton Support: Using Proton Drive CLI](https://proton.me/support/drive-cli): guía para la descarga, el inicio de sesión y los comandos básicos.

3.  [Proton Drive CLI: Downloads](https://proton.me/download/drive/cli/index.html): versión actual 0.8.0, todas las compilaciones para plataformas con sumas de comprobación SHA-512.

4.  [ProtonDriveApps/sdk: cli/README.md](https://github.com/ProtonDriveApps/sdk/blob/main/cli/README.md): variables de entorno, ubicaciones de almacenamiento, almacenes de credenciales y la nota sobre la compilación `x64-baseline`.

5.  [ProtonDriveApps/sdk: cli/CHANGELOG.md](https://github.com/ProtonDriveApps/sdk/blob/main/cli/CHANGELOG.md): historial de versiones de 0.4.2 a 0.8.0, entre otros la compatibilidad con `pass` (0.6.0) y la omisión de archivos sin cambios (0.7.0).

6.  [ProtonDriveApps/sdk: cli/src/commands](https://github.com/ProtonDriveApps/sdk/tree/main/cli/src/commands): código fuente de los comandos con opciones, estrategias de conflicto y el comando Takeout aún no publicado.

7.  [Proton for Business: Proton Drive CLI](https://proton.me/business/drive/cli): escenarios de uso empresarial de Proton, por ejemplo revocar comparticiones cuando empleados dejan la empresa.

8.  [Rclone: Proton Drive](https://rclone.org/protondrive/): el backend comunitario con funciones de montaje y sincronización como comparación.
