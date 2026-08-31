---
title: "Sin contraseña en servidores Linux: configurar el inicio de sesión con clave SSH mediante PuTTY, Pageant y compañía"
navTitle: "PuTTY sin contraseña"
description: "Quien accede a diario a servidores Linux como administrador introduce el nombre de usuario y la contraseña con cada inicio de sesión por contraseña. Un par de claves SSH lo reduce a un doble clic: generar la clave con PuTTYgen, almacenar la clave pública en el servidor y cargarla en Pageant. La misma clave funciona en WinSCP, MobaXterm y OpenSSH, y quien lo desee puede acceder directamente a la shell de una cuenta de servicio."
date: "2026-08-28"
kategorie: "Linux y SSH"
timeToRead: "9 min de lectura"
themen:
  - totemomail
  - windows-client
produkte:
  - "totemomail"
  - "uebergreifend"
protokolle:
  - "ssh"
  - "haertung"
slug: "sin-contrasena-en-servidores-linux-configurar-el-inicio-de-sesion-con-clave-ssh-mediante-putty"
translationId: "article-9f94fa6eb8b95bcf"
aiPrompt: |
  Du bist mein Linux- und SSH-Assistent. Hilf mir Schritt für Schritt, einen passwortlosen SSH-Login von Windows auf meine Linux-Server einzurichten: 1. Ein Ed25519-Schlüsselpaar mit PuTTYgen erzeugen und den Public Key in authorized_keys eintragen. 2. Die PuTTY-Session mit Schlüsseldatei und Auto-login username vervollständigen und Pageant mit Autostart einrichten. 3. Optional ein Remote command hinterlegen, das mich direkt in die Shell eines Service-Accounts bringt, samt minimaler NOPASSWD-Regel unter /etc/sudoers.d. Weise mich auf typische Fehler hin: Key im falschen Home-Verzeichnis, mehrzeiliger Public Key, falsche Berechtigungen, sudoers-Befehl stimmt nicht exakt mit dem Remote command überein.
translationOf: putty-ssh-login-service-account-shell
url: https://rafaelpfister.ch/es/blog/sin-contrasena-en-servidores-linux-configurar-el-inicio-de-sesion-con-clave-ssh-mediante-putty
translationSourceHash: e95b27e9a86f59dfb0808afee63664493f5961b983f807f31ef9ee7a36f6fb3e
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:37:31.467Z
translationReview: automatic
---

# Sin contraseña en servidores Linux: configurar el inicio de sesión con clave SSH mediante PuTTY, Pageant y compañía

Quien accede a diario a servidores Linux como administrador repite los mismos datos en cada conexión mediante inicio de sesión por contraseña: nombre de usuario, contraseña y, en el caso de cuentas de servicio, un comando sudo después. Con un par de claves SSH, todo esto desaparece. Tras la configuración, un doble clic en la sesión guardada abre una shell lista para usar, y la misma clave funciona en PuTTY, WinSCP, MobaXterm y el cliente OpenSSH de Windows.

Esta guía configura por completo el inicio de sesión sin contraseña desde Windows: generar el par de claves, almacenar la clave pública en el servidor, completar la sesión de PuTTY y configurar Pageant como agente de claves. También incluye la resolución del problema más habitual (el servidor sigue solicitando la contraseña a pesar de la clave) y, como ampliación, el acceso directo a la shell de una cuenta de servicio como `totemo`.

## Por qué usar claves en lugar de contraseña

La ganancia de comodidad es el efecto más visible, pero no el más importante. Una clave Ed25519 es prácticamente inmune a los ataques de fuerza bruta, mientras que una contraseña solo es tan fuerte como su longitud y la disciplina de no reutilizarla en ningún sitio. En servidores cuyos usuarios se han migrado por completo a claves, la autenticación por contraseña puede desactivarse por completo en la configuración de sshd (`PasswordAuthentication no`), de modo que los intentos de inicio de sesión automatizados desde Internet no llegan a ninguna parte. Desactive la autenticación por contraseña solo cuando el inicio de sesión mediante clave funcione de forma comprobada y exista una segunda vía de acceso (consola, segunda clave).

El principio es el siguiente: la clave privada permanece en su equipo Windows y la pública se almacena en el servidor. Al establecer la conexión, el servidor comprueba si la otra parte posee la clave privada correspondiente, sin que esta abandone jamás el equipo.

## Paso 1: generar el par de claves con PuTTYgen

1. Inicie **PuTTYgen** (parte del paquete PuTTY), seleccione **Ed25519** como tipo, haga clic en **Generate** y mueva el ratón sobre el área.
2. Introduzca una **Passphrase** en ambos campos y guarde la clave privada con **Save private key** como archivo `.ppk`.
3. Copie por completo el campo de texto superior (la línea que comienza con `ssh-ed25519 AAAA...`). Es la clave pública en el formato que espera el servidor.

Guarde la clave privada con frase de contraseña. Sin frase de contraseña, cualquier copia del archivo es un acceso listo al servidor; con frase de contraseña, el archivo por sí solo no vale nada. La desventaja en comodidad desaparece casi por completo con Pageant (paso 4). Una clave sin frase de contraseña solo es aceptable para automatización desatendida, no para el inicio de sesión interactivo.

## Paso 2: almacenar la clave pública en el servidor

En el servidor, tras iniciar sesión como el usuario con el que se conectará en el futuro:

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo 'ssh-ed25519 AAAA... kommentar' >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
chmod go-w ~
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Comando / opción | Efecto |
|---|---|
| `mkdir -p ~/.ssh` | Crea el directorio SSH; `-p` suprime el error si ya existe |
| `chmod 700 ~/.ssh` | Solo el propietario puede leer, escribir y acceder al directorio |
| `echo '...' >> ~/.ssh/authorized_keys` | Añade la clave pública como una línea nueva al archivo (`>>` en lugar de `>`, de lo contrario sobrescribirá claves existentes) |
| `chmod 600 ~/.ssh/authorized_keys` | Solo el propietario puede leer y escribir el archivo de claves |
| `chmod go-w ~` | Revoca el permiso de escritura al grupo y a otros en el directorio home |

</details>

La última línea parece poco llamativa, pero es necesaria: un directorio home con permisos de escritura para el grupo o para todos hace que el demonio SSH ignore silenciosamente la clave, y el servidor siga solicitando la contraseña sin indicar el motivo.

## Paso 3: completar la sesión de PuTTY

1. Abra PuTTY, seleccione la sesión guardada y cárguela con **Load**.
2. En **Connection → SSH → Auth → Credentials**, seleccione el archivo `.ppk` en **Private key file for authentication**.
3. En **Connection → Data**, introduzca el nombre de usuario en **Auto-login username**; de lo contrario, PuTTY seguirá solicitándolo en cada conexión.
4. Vuelva a la categoría **Session**, seleccione de nuevo el nombre de la sesión y haga clic en **Save**.

El error de uso más frecuente: omitir **Load** antes de modificar o **Save** después. Sin Load, solo edita la configuración predeterminada; sin Save, el cambio se pierde al iniciar PuTTY la próxima vez.

## Paso 4: Pageant, frase de contraseña una vez por sesión de Windows

Pageant es el agente de claves de PuTTY. Mantiene la clave privada descifrada en memoria, por lo que la frase de contraseña solo se solicita una vez por sesión de Windows:

1. Inicie Pageant (el icono aparece en la bandeja del sistema).
2. Haga clic con el botón derecho en el icono, seleccione **Add Key**, elija el archivo `.ppk` e introduzca la frase de contraseña.
3. A partir de ese momento, todas las conexiones funcionarán sin solicitudes hasta el siguiente reinicio.

Para que Pageant se inicie automáticamente, cree un acceso directo en la carpeta de inicio automático (`Win+R`, después `shell:startup`) y pase la clave como argumento:

```text
"C:\Program Files\PuTTY\pageant.exe" "C:\Pfad\zum\schluessel.ppk"
```

Windows solicitará la frase de contraseña una vez después de iniciar sesión; el resto de la jornada laboral transcurrirá sin hacerlo.

## Cuando el servidor sigue solicitando la contraseña

La resolución de problemas comienza en el **Event Log** de PuTTY (clic con el botón derecho en la barra de título de la sesión de terminal). Allí se indica si la clave se ofreció siquiera:

| Hallazgo en el Event Log | Causa y solución |
|---|---|
| No hay ninguna entrada sobre una clave pública | El archivo `.ppk` no está almacenado en la sesión guardada o se editó la sesión equivocada. Cargue la sesión, establezca la clave y guarde. |
| `Server refused our key` | El servidor no encuentra o no acepta la clave: home incorrecto, formato incorrecto o permisos incorrectos (véase más abajo). |
| `Access granted`, seguido de solicitud de contraseña | El inicio de sesión con clave funcionó; la solicitud procede de un programa posterior, normalmente sudo. Véase la ampliación más abajo. |

Las tres causas más frecuentes de `Server refused our key`:

- **Clave en el home equivocado.** La clave pública debe estar en el archivo `authorized_keys` del usuario con el que se establece la conexión. Si al añadirla ya cambió a otra cuenta con `sudo -u` o `su`, el archivo se crea en su home en lugar de en el suyo. `whoami` antes de añadirla muestra en el home de quién se almacenará la clave.
- **Formato incorrecto.** La clave pública debe estar como una única línea en `authorized_keys`, en el formato del campo de texto superior de PuTTYgen. El archivo de **Save public key** tiene un formato distinto de varias líneas (`---- BEGIN SSH2 PUBLIC KEY ----`) y no funciona en `authorized_keys`.
- **Permisos.** `~/.ssh` con `700`, `authorized_keys` con `600`, y el directorio home sin permisos de escritura para el grupo ni para todos.

Si el hallazgo sigue sin estar claro, en el servidor ayuda revisar `/var/log/secure` o `journalctl -u sshd`; allí el demonio SSH explica el rechazo.

## La misma clave en otras herramientas

La configuración en el servidor es independiente de la herramienta; la clave puede reutilizarse en todas ellas:

| Herramienta | Configuración |
|---|---|
| **WinSCP** | Utiliza directamente archivos `.ppk` (Inicio de sesión → Avanzado → SSH → Autenticación) y usa automáticamente Pageant si la clave está cargada allí |
| **MobaXterm** | Archivo `.ppk` en Session settings → SSH → Advanced → Use private key; también entiende el formato OpenSSH |
| **FileZilla** | Añada el archivo `.ppk` en Ajustes → SFTP o deje Pageant en ejecución |
| **OpenSSH (Windows Terminal, `ssh`)** | Requiere el formato OpenSSH: expórtelo en PuTTYgen mediante **Conversions → Export OpenSSH key** y guárdelo en `~/.ssh/` |

Para el cliente OpenSSH, el inicio de sesión cómodo incluye una entrada en `~/.ssh/config` en el equipo Windows:

```text
Host mailgw
    HostName server.example.com
    User mmuster
    IdentityFile ~/.ssh/id_ed25519
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Host mailgw` | Alias de libre elección; después basta con `ssh mailgw` como comando |
| `HostName` | Nombre real del servidor o dirección IP |
| `User` | Nombre de usuario; equivale a Auto-login username en PuTTY |
| `IdentityFile` | Ruta a la clave privada en formato OpenSSH |

</details>

## Ampliación: acceder directamente a la shell de la cuenta de servicio

Muchos servidores Linux en entornos de aplicaciones y correo no se administran con la cuenta personal, sino mediante una cuenta de servicio: TotemoMail se ejecuta bajo `totemo`, y otras pasarelas y aplicaciones tienen sus propias cuentas funcionales. Por ello, después del inicio de sesión se ejecuta rutinariamente el mismo comando:

```bash
sudo -u totemo /bin/bash -l
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `sudo` | Ejecuta el siguiente comando con otros privilegios y registra la llamada |
| `-u totemo` | El usuario de destino es `totemo` en vez del predeterminado `root` |
| `/bin/bash` | El comando que se ejecutará: una nueva shell Bash |
| `-l` | Inicia Bash como shell de inicio de sesión; lee el perfil del usuario de destino (`.bash_profile`, variables de entorno, rutas) |

</details>

El parámetro `-l` es decisivo para las cuentas de servicio: sin una shell de inicio de sesión faltan las variables de entorno del perfil de la cuenta funcional, como rutas a directorios de aplicaciones o instalaciones de Java, y los comandos propios de la aplicación fallan con mensajes de error engañosos.

El inicio de sesión SSH directo como cuenta de servicio sería aún más corto, pero normalmente no es posible por buenas razones: las cuentas funcionales a menudo no tienen contraseña o tienen una shell de inicio de sesión bloqueada, y un acceso directo compartido por varias personas eliminaría el rastro de auditoría personalizado. Al pasar por sudo, sigue siendo posible comprobar qué persona cambió y cuándo a la shell de la cuenta de servicio. La siguiente automatización no cambia esto; solo ahorra la tarea de teclear.

### Remote command en la sesión de PuTTY

PuTTY permite almacenar un comando por sesión guardada que se ejecuta después del inicio de sesión en lugar de la shell normal:

1. Cargue la sesión con **Load**.
2. En el árbol de la izquierda, vaya a **Connection → SSH** (el nodo principal, no un subapartado).
3. Introduzca en el campo **Remote command**: `sudo -u totemo /bin/bash -l`
4. Guarde en **Session**.

Debe conocer tres particularidades del Remote command:

- Un `exit` en la shell de la cuenta de servicio cierra completamente la conexión, en lugar de devolverle a su shell personal. El comando sustituye la shell de inicio de sesión; no la anida.
- Si ocasionalmente trabaja en el servidor con la cuenta personal, guarde una segunda sesión sin Remote command (cargue la sesión, vacíe el campo y guárdela con un nombre nuevo).
- Las herramientas de transferencia de archivos como WinSCP o `pscp` no se ven afectadas. Establecen sus propias conexiones e ignoran el Remote command de la sesión de PuTTY.

Si la conexión se cierra inmediatamente tras establecerse o sudo informa de que falta un terminal: compruebe en **Connection → SSH → TTY** que **Don't allocate a pseudo-terminal** no esté marcado. Por defecto no lo está. Importante para el paso 2 anterior: con Remote command activo, al añadir la clave pública ya está usando la cuenta de servicio; sin embargo, la clave pertenece al home del usuario personal.

En el cliente OpenSSH, dos líneas en la entrada `Host` hacen lo mismo: `RequestTTY yes` y `RemoteCommand sudo -u totemo /bin/bash -l`.

### sudo sin solicitud de contraseña

Después de la clave y el Remote command queda una única entrada: la solicitud de contraseña de sudo. Solo desaparece mediante la configuración de sudoers en el servidor, y para ello se necesitan permisos de root. En un servidor corporativo gestionado, esto es una solicitud al administrador del servidor, no una configuración de PuTTY.

La regla debe ir en un archivo independiente bajo `/etc/sudoers.d/` y editarse con `visudo`:

```bash
visudo -f /etc/sudoers.d/totemo-shell
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `visudo` | Abre el archivo sudoers en el editor y comprueba la sintaxis antes de guardar |
| `-f /etc/sudoers.d/totemo-shell` | Edita el archivo indicado en lugar del archivo central `/etc/sudoers` |

</details>

Contenido del archivo, con el nombre de usuario personal (aquí `mmuster` como ejemplo):

```text
mmuster ALL=(totemo) NOPASSWD: /bin/bash -l
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Componente | Efecto |
|---|---|
| `mmuster` | La regla solo se aplica a este usuario |
| `ALL=` | En todos los hosts (relevante para archivos sudoers distribuidos centralmente) |
| `(totemo)` | Solo para comandos ejecutados como usuario de destino `totemo`, no como root |
| `NOPASSWD:` | Sin solicitud de contraseña para los siguientes comandos |
| `/bin/bash -l` | Exactamente este comando con exactamente este argumento, nada más |

</details>

Dos aspectos determinan si la regla se aplica y si es razonable:

- **Coincidencia exacta.** El comando de la regla debe corresponder al Remote command, incluido el argumento. Si PuTTY contiene `sudo -u totemo /bin/bash -l`, la regla debe permitir `/bin/bash -l`. Una regla para `/bin/bash` sin `-l` no cubre la llamada con `-l`, y sudo seguirá solicitando la contraseña.
- **Alcance mínimo.** La regla permite un único comando como un único usuario de destino. No concede permisos de root ni habilita comandos arbitrarios. En esta forma, también es una solicitud habitual y justificable en servidores gestionados. El registro de sudo se conserva íntegramente; cada cambio a la shell de la cuenta de servicio sigue figurando en el registro.

`visudo` no es opcional: comprueba la sintaxis antes de guardar. Un error tipográfico escrito directamente en el archivo puede dejar sudo inutilizable para todos los usuarios del sistema. Por la misma razón, es preferible un archivo independiente bajo `/etc/sudoers.d/` a editar el archivo central `/etc/sudoers`; sobrevive a las actualizaciones de paquetes y puede eliminarse de nuevo sin riesgo.

## El resultado

Tras la configuración, el inicio de sesión es así: doble clic en la sesión de PuTTY y la shell está lista. Sin nombre de usuario, sin contraseña y, con la ampliación, tampoco sin solicitud de sudo. La situación de seguridad no ha empeorado; en un aspecto incluso ha mejorado:

| Aspecto | Antes | Después |
|---|---|---|
| Autenticación | Contraseña en cada inicio de sesión | Clave Ed25519 con frase de contraseña, mantenida en Pageant |
| Identidad de inicio de sesión | Cuenta personal | Cuenta personal sin cambios |
| Registro de sudo | Cada cambio en el registro | Cada cambio sigue en el registro |
| Alcance de NOPASSWD | Ninguno | Un comando, un usuario de destino, sin root |

## Fuentes

1.  [PuTTY User Manual, capítulo 4: Configuring PuTTY](https://the.earth.li/~sgtatham/putty/0.83/htmldoc/Chapter4.html): Documentación de la configuración de sesiones, incluidos Remote command (panel SSH), Auto-login username (panel Data) y Pseudo-Terminal (panel TTY).

2.  [PuTTY User Manual, capítulo 8: Using public keys for SSH authentication](https://the.earth.li/~sgtatham/putty/0.83/htmldoc/Chapter8.html): PuTTYgen, tipos de claves, frases de contraseña, exportación OpenSSH y almacenamiento de la clave pública en el servidor.

3.  [PuTTY User Manual, capítulo 9: Using Pageant for authentication](https://the.earth.li/~sgtatham/putty/0.83/htmldoc/Chapter9.html): Funcionamiento del agente, carga de claves e inicio con una clave como argumento de línea de comandos.

4.  [Manual ssh_config(5)](https://man.openbsd.org/ssh_config.5): Configuración de cliente del cliente OpenSSH, incluidos alias de host, IdentityFile, RequestTTY y RemoteCommand.

5.  [Manual sudoers(5)](https://www.sudo.ws/docs/man/sudoers.man/): Sintaxis de las reglas sudoers, especificación Runas y etiqueta NOPASSWD.

6.  [Manual sshd(8), sección AUTHORIZED_KEYS FILE FORMAT](https://man.openbsd.org/sshd.8): Formato del archivo authorized_keys y requisitos de permisos de archivos.
