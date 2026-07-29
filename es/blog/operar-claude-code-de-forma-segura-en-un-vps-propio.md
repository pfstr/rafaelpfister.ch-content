---
title: "Operar Claude Code de forma segura en un VPS propio"
navTitle: "VPS para Claude"
description: "Un VPS Debian reforzado mantiene las sesiones de Claude Code accesibles de forma permanente. La guía abarca desde la cuenta de usuario y las claves SSH hasta el firewall, la higiene de datos, tmux y el acceso seguro desde el iPhone."
date: "2026-07-21"
kategorie: "Claude"
timeToRead: "12 min de lectura"
themen:
  - "claude"
slug: "operar-claude-code-de-forma-segura-en-un-vps-propio"
translationOf: "claude-code-vps-debian-absichern"
url: "https://rafaelpfister.ch/es/blog/operar-claude-code-de-forma-segura-en-un-vps-propio"
---

En el propio ordenador, una sesión de Claude Code termina involuntariamente como muy tarde cuando el portátil entra en suspensión o se interrumpe la conexión de red. Un VPS sigue funcionando y es accesible desde varios dispositivos. Al mismo tiempo, permanece conectado al Internet público y recibe escaneos automatizados poco después de iniciarse.

Esta guía combina ambos requisitos: Claude Code permanece disponible en una sesión de `tmux`, mientras que el servidor Debian solo ofrece al exterior una conexión SSH protegida con claves. El refuerzo no es específico de Claude y también sirve para otros servidores Linux accesibles públicamente.

## Por qué un VPS puede tener sentido

Frente a una instalación exclusivamente local, el servidor ofrece tres ventajas prácticas:

- **Persistencia.** En una sesión de `tmux`, Claude sigue ejecutándose aunque se desconecte la conexión SSH. Una tarea que tarda diez minutos o una hora termina sin que el portátil tenga que permanecer abierto.
- **Accesibilidad.** La misma sesión es accesible desde el ordenador de escritorio, el portátil y el iPhone. Se inicia una tarea en el escritorio y se consulta el resultado durante el trayecto.
- **Control de datos.** Uno mismo decide qué hay en el servidor. Sin servicio de sincronización ni credenciales incluidas accidentalmente en copias de seguridad, siempre que la migración se realice con cuidado (véase más abajo).

`tmux` es únicamente una función de disponibilidad y comodidad, no una medida de seguridad. El trabajo real está en la protección.

## Situación inicial

La base es Debian 13 (Trixie), instalado de forma mínima, sin escritorio ni servicios de red adicionales. El proveedor ofrece un firewall previo que actúa independientemente del sistema operativo. El objetivo es un servidor en el que solo SSH sea accesible desde el exterior, y únicamente con claves protegidas mediante frase de contraseña.

## 1. Actualizar el sistema

Justo después de la instalación, actualizar todo el estado de paquetes:

```bash
sudo apt update
sudo apt full-upgrade
```

`full-upgrade`, a diferencia de `upgrade`, también resuelve dependencias que requieren paquetes nuevos o eliminados. En un sistema recién instalado, esta es la forma correcta de aplicar realmente todas las actualizaciones de seguridad disponibles. Reiniciar una vez tras actualizaciones del kernel.

## 2. Usuario propio en lugar de root

Trabajar como root conlleva riesgos innecesarios: cualquier error tipográfico afecta a todo el sistema, y el inicio de sesión directo de root es lo primero que intentan los ataques automatizados. Por eso, crear un usuario propio (aquí `claude`) con permisos sudo para los casos en los que se necesiten:

```bash
sudo adduser claude
sudo usermod -aG sudo claude
```

A partir de ahora, toda la administración se realiza mediante `claude` y `sudo`, ya no mediante acceso directo como root.

## 3. Claves Ed25519 con frase de contraseña, una por dispositivo

El inicio de sesión debe funcionar exclusivamente mediante claves SSH, no mediante contraseñas. Ed25519 es el estándar actual: breve, rápido y criptográficamente sólido. Es decisivo que la clave se genere en el cliente, es decir, en el PC y no en el servidor, y que esté protegida con una frase de contraseña. La frase de contraseña es la segunda línea de defensa si la clave privada cae alguna vez en manos equivocadas.

En el PC:

```bash
ssh-keygen -t ed25519 -C "pc-thinkpad"
```

El comentario (`-C`) identifica el dispositivo. Esto resulta útil más adelante: se genera una clave propia para cada dispositivo, una para el PC y otra independiente para el iPhone. Si se pierde un dispositivo, se elimina específicamente su clave pública de `~/.ssh/authorized_keys`, sin tener que desplegar de nuevo todos los demás accesos.

Solo la clave pública debe llegar al servidor. La clave privada nunca abandona el dispositivo. Al final, `authorized_keys` contiene exclusivamente claves públicas, cada una con el comentario de su dispositivo:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...pc  pc-thinkpad
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...ios iphone-15
```

Transferir inicialmente la clave pública del PC. Mientras el inicio de sesión por contraseña siga activo, la forma más sencilla es:

```bash
ssh-copy-id claude@SERVER
```

Después, comprobar que el inicio de sesión con clave funciona antes de desactivar el inicio de sesión por contraseña en el siguiente paso. Los permisos de archivo deben ser correctos; de lo contrario, sshd ignora el archivo: `~/.ssh` en `700`, `authorized_keys` en `600`.

## 4. Reforzar SSH: sin root, sin contraseña

La configuración del servidor se encuentra en `/etc/ssh/sshd_config` y, en Debian 13, en archivos drop-in bajo `/etc/ssh/sshd_config.d/`. Los cambios deben ir en un archivo drop-in propio; así el archivo principal permanece intacto y las actualizaciones de paquetes no sobrescriben nada. Crear el archivo `/etc/ssh/sshd_config.d/99-haertung.conf`:

```text
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
```

Esto desactiva el inicio de sesión directo de root y el inicio de sesión mediante contraseña. A partir de ahora, solo podrá entrar quien posea una clave privada adecuada. Antes de recargar, comprobar sintácticamente la configuración:

```bash
sudo sshd -t
```

Si `sshd -t` no informa nada, el archivo es válido. Solo entonces recargar:

```bash
sudo systemctl reload ssh
```

**Importante:** Mantener abierta la sesión SSH existente y probar el nuevo acceso en una segunda terminal. Solo cuando se haya confirmado que el inicio de sesión con clave funciona allí debe cerrarse la sesión antigua. Esta medida de precaución reduce prácticamente a cero el riesgo de quedarse bloqueado. De lo contrario, un error en la configuración cuesta el acceso completo.

## 5. Mover SSH a un puerto poco habitual

El puerto estándar 22 es probado por bots las 24 horas del día. Cambiar a un puerto alto elegido libremente (en el ejemplo, `61417`) hace que la mayor parte de este ruido automatizado no llegue a nada. Esto no es una mejora de seguridad en sentido estricto: cambiar el puerto no sustituye una autenticación fuerte, solo reduce el volumen de registros y la carga de escaneos. La obligación de usar claves del paso 4 sigue siendo la protección real.

El puerto elegido no es arbitrario. IANA distingue tres zonas: **0–1023 (puertos bien conocidos)** están reservados para servicios estándar (SSH en el 22, HTTP en el 80, HTTPS en el 443), requieren root para vincularse y no tienen cabida en un puerto SSH elegido por uno mismo; estos son precisamente los puertos que esperan los escáneres y también los servicios estándar que se instalen después. **1024–49151 (puertos registrados)** se asignan a aplicaciones individuales previa solicitud, como 3306 (MySQL), 5432 (PostgreSQL), 6379 (Redis) u 8080/8443 como alternativas HTTP habituales; un puerto elegido al azar en este rango puede entrar fácilmente en conflicto más adelante con software que espera exactamente su puerto registrado. **49152–65535 (puertos dinámicos/privados)** no están asignados a ningún servicio según IANA y están previstos para fines temporales y privados, el rango correcto para un puerto permanente elegido por uno mismo.

Se mantiene una salvedad: muchos sistemas Linux, incluido Debian, usan parte de ese mismo rango como puerto de origen para sus propias conexiones salientes (`net.ipv4.ip_local_port_range`, por defecto alrededor de 32768–60999). Un servicio que escucha permanentemente no colisiona realmente con ello, ya que el kernel no asigna un puerto que ya está vinculado, pero un puerto por encima de 60999 también evita esta imprecisión teórica. Por eso, el ejemplo de este artículo (`61417`) se encuentra deliberadamente ahí. Antes del cambio, comprobar además con `ss -lntup` (véase el paso 7) que el puerto elegido no esté ya ocupado en el propio servidor.

En Debian 13 hay una dificultad: SSH puede iniciarse mediante activación de socket de systemd. Si es el caso, la indicación `Port` en `sshd_config` simplemente se ignora; entonces el puerto debe configurarse en el socket. Primero comprobar cuál es el caso:

```bash
systemctl is-enabled ssh.socket
```

Si el comando responde `enabled`, SSH se ejecuta mediante el socket. Entonces cambiar el puerto allí:

```bash
sudo systemctl edit ssh.socket
```

Introducir las siguientes líneas en el editor. La primera línea vacía `ListenStream=` elimina el puerto 22 preconfigurado; la segunda establece el nuevo:

```text
[Socket]
ListenStream=
ListenStream=61417
```

A continuación, aplicar los cambios:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
```

Si la activación de socket no está activa (`disabled`), se debe incluir `Port 61417` en el archivo drop-in del paso 4, seguido de `sudo sshd -t` y `sudo systemctl restart ssh`.

También aquí se aplica lo siguiente: primero abrir el nuevo puerto en el firewall (siguiente paso), después conectarse y probar, y mantener abierta la sesión antigua hasta confirmar el acceso mediante el nuevo puerto.

## 6. Firewall: cerrado por defecto

El firewall previo del proveedor es el límite más eficaz porque intercepta los paquetes antes de que lleguen siquiera al sistema operativo. Dos reglas básicas:

- **Acción predeterminada entrante en DROP.** Todo lo que no esté expresamente permitido se descarta, sin comentarios ni respuesta al remitente.
- **Una única excepción:** TCP entrante hacia el puerto de destino `61417`. No hace falta que nada más sea accesible desde el exterior.

El tráfico saliente permanece permitido. Esto es deliberado: el servidor debe poder descargar paquetes, sincronizar la hora y llegar a la API para Claude Code. Un filtrado saliente restrictivo aporta poca protección adicional en un servidor individual, pero hace la operación notablemente más incómoda.

Quien desee defensa en profundidad adicional puede duplicar las mismas reglas en el host mediante `nftables` o `ufw`. Para la configuración descrita, el firewall del proveedor es suficiente.

## 7. Comprobar la superficie de ataque

Tras el refuerzo, comprobar qué ofrece realmente el servidor al exterior. Bastan dos comandos. Primero: ¿qué servicios escuchan en qué direcciones?

```bash
sudo ss -lntup
```

`ss` lista todos los sockets TCP y UDP en escucha junto con su proceso asociado (`sudo` es necesario para ver los nombres de los procesos). Lo decisivo es la columna de dirección: un servicio en `0.0.0.0` o `[::]` es accesible desde el exterior; uno en `127.0.0.1` o `[::1]` solo es local. En estado protegido, únicamente SSH debería aparecer públicamente. Pueden aparecer servicios como `chronyd` (sincronización de hora), pero solo vinculados a direcciones locales. Si `chronyd` escucha exclusivamente en `127.0.0.1` y `::1`, no es accesible desde el exterior y, por tanto, no es problemático.

Segundo: ¿hay servicios del sistema fallidos que indiquen un problema de configuración?

```bash
systemctl --failed
```

La respuesta debería ser `0 loaded units listed`, ni un solo servicio fallido. Las unidades defectuosas no son solo un problema operativo, sino potencialmente también de seguridad si detrás hay un servicio de red iniciado a medias o configurado incorrectamente.

## 8. Instalar y operar Claude Code

Claude Code necesita un entorno de ejecución Node.js actualizado. Tras instalarlo, configurar la CLI según la guía oficial y autenticarse de nuevo en el servidor; no subir las credenciales locales (más sobre ello a continuación).

Para operar `tmux` de forma permanente:

```bash
tmux new -s claude
```

Dentro de la sesión, iniciar Claude. Con `Ctrl-b`, luego `d`, se desconecta de la sesión sin terminarla; Claude sigue ejecutándose. Para volver:

```bash
tmux attach -t claude
```

Así, una tarea en ejecución sobrevive a conexiones interrumpidas, cambios de dispositivo y al descanso nocturno del portátil.

## 9. Higiene de datos durante la migración

La parte más delicada de trasladarse al servidor no es la técnica, sino la cuestión de qué llevar consigo. Tres reglas:

- **No llevar claves privadas al servidor.** En `authorized_keys` solo hay claves públicas. Las claves privadas permanecen en los dispositivos finales.
- **No copiar credenciales indiscriminadamente.** Archivos locales sensibles como `.credentials.json` no deben llegar al VPS sin revisión. En su lugar, autenticarse de nuevo en el servidor.
- **Primero la configuración en una carpeta de migración.** No escribir directamente las memorias y configuraciones existentes de Claude en las rutas de configuración activas; transferirlas primero a una carpeta de migración independiente y comprobar allí qué se debe adoptar realmente. Lo que ya no se necesita, como entradas MCP antiguas o ajustes huérfanos, se deja deliberadamente atrás en lugar de migrarlo sin más.

## 10. Vistas previas web mediante un túnel SSH

Para vistas previas web, por ejemplo un servidor de desarrollo local iniciado por Claude, es tentador abrir simplemente otro puerto. No debe hacerse. Cada puerto abierto adicional es superficie de ataque adicional. En su lugar, la vista previa se ejecuta mediante un túnel de puertos SSH cifrado: el servicio escucha solo localmente en el servidor y SSH lo reenvía al cliente.

Hacer accesible desde el PC un servicio que se ejecuta localmente en el puerto 4321:

```bash
ssh -p 61417 -L 4321:localhost:4321 claude@SERVER
```

Después, abrir `http://localhost:4321` en el navegador local. Todo el tráfico circula por la conexión SSH autenticada existente, sin tener que abrir un solo puerto adicional en el firewall.

## Acceso desde el iPhone

El acceso durante los desplazamientos funciona con el mismo modelo de seguridad que desde el PC. Solo se necesita un cliente SSH con gestión de claves. Son habituales **Termius**, **Blink Shell** y **Secure ShellFish**; todos pueden generar claves Ed25519 y almacenarlas en el llavero de iOS, en algunos casos protegidas mediante Face ID.

El procedimiento corresponde al paso 3, solo que en el iPhone:

1. Generar una clave Ed25519 propia para el iPhone en el cliente SSH, sin copiar la clave del PC. La clave privada permanece en el llavero del dispositivo.
2. Añadir la clave pública del iPhone como línea adicional en `~/.ssh/authorized_keys` en el servidor, con un comentario descriptivo (`iphone-15`).
3. Crear la conexión en el cliente: dirección del servidor, usuario `claude`, puerto `61417`, la clave del iPhone como autenticación.

Precisamente por eso merece la pena disponer de una clave independiente por dispositivo: si se pierde el iPhone, se elimina del servidor la única línea `iphone-15` de `authorized_keys`, y el dispositivo queda bloqueado mientras que el acceso desde el PC y todas las demás claves siguen funcionando sin cambios.

Tras conectarse, recuperar la sesión de Claude en ejecución con `tmux attach -t claude` y continuar trabajando donde se dejó en el escritorio. El túnel de puertos del paso 10 también funciona desde iOS; Termius y Secure ShellFish admiten redirección de puertos.

## Lista de comprobación

En resumen, el proceso completo:

1. Debian 13 instalado y completamente actualizado con `apt full-upgrade`.
2. Usuario propio `claude` con permisos sudo; el inicio de sesión directo de root ya no se utiliza.
3. Claves Ed25519 protegidas con frase de contraseña, una por dispositivo; solo claves públicas en `authorized_keys`.
4. sshd reforzado: `PermitRootLogin no`, `PasswordAuthentication no`; comprobado con `sshd -t` antes de recargar, manteniendo abierta la sesión existente hasta la prueba.
5. SSH en el puerto 61417, configurado en `ssh.socket` con activación de socket; de lo contrario, en la configuración de sshd.
6. Firewall del proveedor: DROP predeterminado entrante, única excepción TCP 61417; salida permitida.
7. Superficie de ataque comprobada con `ss -lntup` (solo SSH público, `chronyd` local) y `systemctl --failed` (sin errores).
8. Claude Code autenticado de nuevo en el servidor, operado en una sesión de `tmux`.
9. Higiene de datos: sin claves privadas ni credenciales en el servidor; configuración comprobada primero mediante una carpeta de migración.
10. Sin puertos adicionales; las vistas previas web se ejecutan a través de un túnel SSH.

Tras esta configuración, desde el exterior solo es accesible SSH en el puerto establecido, y también allí exclusivamente con una clave protegida mediante frase de contraseña. Claude Code se ejecuta independientemente del dispositivo final en `tmux`; las vistas previas web siguen siendo accesibles mediante túneles SSH, sin abrir un puerto adicional.

## Fuentes

1.  [Manual de OpenSSH – sshd_config(5)](https://man.openbsd.org/sshd_config): Referencia de todas las directivas de sshd, incluidas `PermitRootLogin`, `PasswordAuthentication` y `PubkeyAuthentication`.

2.  [Wiki de Debian – SSH](https://wiki.debian.org/SSH): Indicaciones específicas de Debian sobre la configuración de SSH, incluidos los archivos drop-in bajo `/etc/ssh/sshd_config.d/`.

3.  [systemd.socket(5) – freedesktop.org](https://www.freedesktop.org/software/systemd/man/latest/systemd.socket.html): Funcionamiento de la activación de socket y de la directiva `ListenStream=`, relevante para cambiar el puerto SSH en Debian 13.

4.  [ss(8) – página de manual de iproute2](https://man7.org/linux/man-pages/man8/ss.8.html): Opciones de `ss` para listar sockets en escucha junto con el proceso y la dirección de vinculación.

5.  [Claude Code – Documentación oficial](https://docs.claude.com/en/docs/claude-code/overview): Instalación, autenticación y operación de Claude Code.
