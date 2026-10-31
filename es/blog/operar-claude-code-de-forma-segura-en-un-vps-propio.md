---
title: "Operar Claude Code de forma segura en un VPS propio"
navTitle: "VPS para Claude"
description: "Un VPS Debian reforzado mantiene las sesiones de Claude Code disponibles de forma permanente. La guía abarca desde la cuenta de usuario y las claves SSH hasta el firewall, la higiene de datos, tmux y el acceso seguro desde el iPhone."
date: "2026-07-21"
kategorie: "Claude"
timeToRead: "12 min de lectura"
themen:
  - claude
slug: "operar-claude-code-de-forma-segura-en-un-vps-propio"
translationOf: "claude-code-vps-debian-absichern"
translationId: article-f932e9e537d7704a
translationReview: required
translationSourceHash: 0c584eb684bd92b439d97b20aae84cd15468c5c30d1e277d7fde7bd3fab9a37e
translatedAt: 2026-10-31T03:43:40.163Z
translationModel: gpt-5.6-terra
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/operar-claude-code-de-forma-segura-en-un-vps-propio
---

En un ordenador propio, una sesión de Claude Code termina involuntariamente como muy tarde cuando el portátil entra en suspensión o se interrumpe la conexión de red. Un VPS sigue funcionando y es accesible desde varios dispositivos. Al mismo tiempo, permanece conectado a la Internet pública y se escanea automáticamente poco después de iniciarse.

Esta guía combina ambos requisitos: Claude Code permanece disponible en una sesión de `tmux`, mientras que el servidor Debian solo ofrece al exterior una conexión SSH protegida mediante claves. El endurecimiento no es específico de Claude y también es adecuado para otros servidores Linux accesibles públicamente.

## Por qué un VPS puede tener sentido

Frente a una instalación exclusivamente local, el servidor ofrece tres ventajas prácticas:

- **Persistencia.** En una sesión de `tmux`, Claude sigue ejecutándose aunque se desconecte la conexión SSH. Una tarea que tarda diez minutos o una hora termina sin que el portátil tenga que permanecer abierto.
- **Accesibilidad.** Se puede acceder a la misma sesión desde el ordenador de sobremesa, el portátil y el iPhone. Se inicia una tarea en el escritorio y se consulta el resultado durante el trayecto.
- **Control de datos.** Uno mismo decide qué reside en el servidor. Sin servicio de sincronización ni credenciales incluidas accidentalmente en la copia, siempre que la migración se realice con cuidado (véase más abajo).

`tmux` es únicamente una función de disponibilidad y comodidad, no una medida de seguridad. El trabajo real está en la protección.

## Situación de partida

La base es Debian 13 (Trixie), instalado de forma mínima, sin escritorio ni servicios de red adicionales. El proveedor ofrece un firewall previo que actúa independientemente del sistema operativo. El objetivo es un servidor en el que solo SSH sea accesible desde el exterior, y únicamente con claves protegidas por frase de contraseña.

## 1. Actualizar el sistema

Inmediatamente después de la instalación, actualizar todos los paquetes:

```bash
sudo apt update
sudo apt full-upgrade
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `update` | Vuelve a leer las listas de paquetes de todas las Fuentes configuradas |
| `full-upgrade` | Actualiza todos los paquetes y, para ello, también puede instalar paquetes nuevos o eliminar los existentes |

</details>

A diferencia de `upgrade`, `full-upgrade` también resuelve dependencias que requieren paquetes nuevos o eliminados. En un sistema recién instalado, esta es la forma adecuada de instalar realmente todas las actualizaciones de seguridad disponibles. Reiniciar una vez después de las actualizaciones del kernel.

## 2. Usuario propio en lugar de root

Trabajar como root supone un riesgo innecesario: cualquier error tipográfico afecta a todo el sistema, y el inicio de sesión directo de root es lo primero que intentan los ataques automatizados. Por ello, crear un usuario propio (aquí `claude`) con permisos sudo para los casos en que se necesiten:

```bash
sudo adduser claude
sudo usermod -aG sudo claude
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-a` | Añadir: complementa la lista de grupos del usuario en lugar de sustituirla; solo es válido junto con `-G` |
| `-G sudo` | Grupo(s) adicional(es) al que se incorpora el usuario |
| `claude` | El usuario afectado; en `adduser` el nombre de la cuenta nueva que se va a crear |

</details>

A partir de ahora, toda la administración se realiza mediante `claude` y `sudo`, ya no mediante acceso directo como root.

## 3. Claves Ed25519 con frase de contraseña, una por dispositivo

El inicio de sesión debe realizarse exclusivamente mediante claves SSH, no contraseñas. Ed25519 es el estándar actual: breve, rápido y criptográficamente sólido. Lo decisivo es que la clave se genere en el cliente, es decir, en el PC y no en el servidor, y que esté protegida con una frase de contraseña. Esta es la segunda línea de defensa si la clave privada llegara a caer en manos equivocadas.

En el PC:

```bash
ssh-keygen -t ed25519 -C "pc-thinkpad"
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-t ed25519` | Tipo de clave, en este caso el método elíptico Ed25519 |
| `-C "pc-thinkpad"` | Comentario que se añade a la clave pública |

</details>

El comentario (`-C`) identifica el dispositivo. Esto resulta útil más adelante: se genera una clave propia para cada dispositivo, una para el PC y otra independiente para el iPhone. Si se pierde un dispositivo, se elimina específicamente su clave pública de `~/.ssh/authorized_keys` sin tener que volver a distribuir todos los demás accesos.

Solo la clave pública pertenece al servidor. La clave privada nunca abandona el dispositivo. Al final, `authorized_keys` contiene exclusivamente claves públicas, cada una con el comentario de su dispositivo:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...pc  pc-thinkpad
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...ios iphone-15
```

Transferir inicialmente la clave pública del PC. Mientras el inicio de sesión con contraseña siga activo, la forma más sencilla es:

```bash
ssh-copy-id claude@SERVER
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `claude@SERVER` | Usuario y host de destino; la clave pública estándar se añade allí a `~/.ssh/authorized_keys` |

</details>

Después, comprobar que el inicio de sesión con clave funciona antes de desactivar la autenticación por contraseña en el siguiente paso. Los permisos de archivo deben ser correctos; de lo contrario, sshd ignorará el archivo: `~/.ssh` en `700`, `authorized_keys` en `600`.

## 4. Endurecer SSH: sin root, sin contraseña

La configuración del servidor se encuentra en `/etc/ssh/sshd_config` y (en Debian 13) en archivos drop-in bajo `/etc/ssh/sshd_config.d/`. Los cambios deben incluirse en un archivo drop-in propio; así, el archivo principal permanece intacto y las actualizaciones de paquetes no sobrescriben nada. Crear el archivo `/etc/ssh/sshd_config.d/99-haertung.conf`:

```text
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
```

Esto desactiva el inicio de sesión directo de root y la autenticación por contraseña. A partir de ahora, solo podrá entrar quien posea una clave privada adecuada. Antes de recargar, comprobar sintácticamente la configuración:

```bash
sudo sshd -t
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-t` | Modo de prueba: comprueba la validez del archivo de configuración y las claves sin iniciar el servicio |

</details>

Si `sshd -t` no informa nada, el archivo es válido. Solo entonces recargar:

```bash
sudo systemctl reload ssh
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `reload` | Indica al servicio que recargue su configuración sin interrumpir las conexiones existentes |
| `ssh` | La unidad de destino, en este caso el servicio OpenSSH |

</details>

**Importante:** dejar abierta la sesión SSH existente y probar el nuevo acceso en una segunda terminal. Solo cuando se haya comprobado que el inicio de sesión con clave funciona allí, se puede cerrar la sesión anterior. Esta medida de precaución reduce a prácticamente cero el riesgo de quedarse bloqueado. De lo contrario, un error en la configuración puede costar todo el acceso.

## 5. Mover SSH a un puerto poco habitual

El puerto estándar 22 es probado por bots las 24 horas del día. Cambiar a un puerto alto elegido libremente (en el ejemplo, `61417`) hace que la mayor parte de este ruido automatizado no llegue a ninguna parte. Esto no supone expresamente una ganancia de seguridad en sentido estricto: cambiar de puerto no sustituye una autenticación fuerte, solo reduce el volumen de registros y la carga de escaneo. La obligatoriedad de claves del paso 4 sigue siendo la protección real.

El puerto elegido no es arbitrario. La IANA distingue tres zonas: los **0–1023 (puertos bien conocidos)** están reservados a servicios estándar (SSH en el 22, HTTP en el 80, HTTPS en el 443), requieren root para enlazarse y no tienen cabida en un puerto SSH elegido por uno mismo; son precisamente los puertos que esperan los escáneres, al igual que los servicios estándar que se instalen más adelante. Los **1024–49151 (puertos registrados)** se asignan previa solicitud a aplicaciones individuales, como 3306 (MySQL), 5432 (PostgreSQL), 6379 (Redis) u 8080/8443 como alternativas HTTP habituales; un puerto elegido al azar de este rango puede entrar fácilmente en conflicto posteriormente con software que espera exactamente su puerto registrado. Los **49152–65535 (puertos dinámicos/privados)** no están asignados a ningún servicio según la IANA y están previstos para fines temporales y privados, el rango adecuado para un puerto permanente elegido por uno mismo.

Queda una salvedad: muchos sistemas Linux, incluido Debian, usan parte de ese mismo rango como puerto de origen para sus propias conexiones salientes (`net.ipv4.ip_local_port_range`, de forma predeterminada alrededor de 32768–60999). Un servicio que escucha permanentemente no entra realmente en conflicto por ello, pues el kernel no asigna un puerto ya enlazado, pero un puerto por encima de 60999 evita también esta imprecisión teórica. Por eso, el ejemplo de este artículo (`61417`) se encuentra deliberadamente ahí. Antes del cambio, comprobar además con `ss -lntup` (véase el paso 7) que el puerto elegido no esté ya ocupado en el propio servidor.

En Debian 13 hay una particularidad: SSH puede iniciarse mediante activación por socket de systemd. Si este es el caso, la indicación `Port` en `sshd_config` simplemente se ignora; entonces el puerto debe establecerse en el socket. Primero, comprobar qué caso se aplica:

```bash
systemctl is-enabled ssh.socket
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `is-enabled` | Muestra si la unidad está activada para el inicio del sistema |
| `ssh.socket` | La unidad de socket del servicio SSH |

</details>

Si el comando responde con `enabled`, SSH se ejecuta a través del socket. Entonces cambiar el puerto allí:

```bash
sudo systemctl edit ssh.socket
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `edit` | Crea un archivo de sobrescritura drop-in para la unidad y lo abre en el editor |
| `ssh.socket` | La unidad de socket que se va a sobrescribir |

</details>

Introducir las siguientes líneas en el editor. La primera línea `ListenStream=` vacía elimina el puerto 22 preconfigurado; la segunda establece el nuevo:

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

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `daemon-reload` | Vuelve a leer todos los archivos de unidad, incluido el archivo de sobrescritura recién creado |
| `restart ssh.socket` | Reinicia la unidad de socket para que escuche en el nuevo puerto |

</details>

Si la activación por socket no está activa (`disabled`), se debe añadir en su lugar `Port 61417` al archivo drop-in del paso 4, seguido de `sudo sshd -t` y `sudo systemctl restart ssh`.

También aquí se aplica lo siguiente: primero abrir el nuevo puerto en el firewall (siguiente paso), luego conectarse y probar, y dejar abierta la sesión anterior hasta confirmar el acceso a través del nuevo puerto.

## 6. Firewall: cerrado por defecto

El firewall previo del proveedor es el límite más eficaz porque intercepta los paquetes antes de que lleguen siquiera al sistema operativo. Dos reglas básicas:

- **Acción predeterminada de entrada en DROP.** Se descarta todo lo que no esté expresamente permitido, sin comentarios ni respuesta al remitente.
- **Una única excepción:** TCP entrante al puerto de destino `61417`. No es necesario que haya nada más accesible desde el exterior.

El tráfico saliente permanece permitido. Es intencionado: el servidor debe descargar paquetes, sincronizar la hora y alcanzar la API para Claude Code. Un filtrado restrictivo de salida aporta poca protección adicional en un servidor individual, pero hace que la operación sea notablemente más engorrosa.

Quien desee defensa en profundidad adicional puede duplicar las mismas reglas en el host con `nftables` o `ufw`. Para la configuración descrita, el firewall del proveedor es suficiente.

## 7. Comprobar la superficie de ataque

Después del endurecimiento, comprobar qué ofrece realmente el servidor al exterior. Bastan dos comandos. Primero: ¿qué servicios escuchan en qué direcciones?

```bash
sudo ss -lntup
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-l` | Muestra solo sockets en escucha |
| `-n` | Salida numérica: los puertos y las direcciones no se resuelven a nombres |
| `-t` | Incluye sockets TCP |
| `-u` | Incluye sockets UDP |
| `-p` | Muestra el proceso detrás de cada socket; para ello se necesita `sudo` |

</details>

La columna de direcciones es decisiva: un servicio en `0.0.0.0` o `[::]` es accesible desde el exterior; uno en `127.0.0.1` o `[::1]` solo localmente. En un estado protegido, solo SSH debería aparecer públicamente. Pueden aparecer servicios como `chronyd` (sincronización horaria), pero solo enlazados a direcciones locales. Si `chronyd` escucha exclusivamente en `127.0.0.1` y `::1`, no es accesible desde el exterior y, por tanto, no es problemático.

Segundo: ¿hay servicios del sistema fallidos que indiquen un problema de configuración?

```bash
systemctl --failed
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `--failed` | Enumera exclusivamente las unidades en estado de error |

</details>

La respuesta debería ser `0 loaded units listed`, ni un solo servicio fallido. Las unidades con errores no son solo un problema operativo, sino también potencialmente de seguridad, si detrás hay un servicio de red iniciado a medias o mal configurado.

## 8. Instalar y operar Claude Code

Claude Code no requiere un entorno de ejecución Node.js independiente. Anthropic recomienda el instalador nativo (`curl -fsSL https://claude.ai/install.sh | bash`), que actualiza Claude Code por sí mismo en segundo plano; para Debian, existe alternativamente un repositorio apt firmado que se actualiza mediante las actualizaciones normales del sistema. Tras la instalación, autenticar la CLI de nuevo en el servidor; no cargar las credenciales locales (más sobre ello enseguida).

Para la operación permanente, `tmux`:

```bash
tmux new -s claude
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `new` | Crea una sesión nueva |
| `-s claude` | Asigna el nombre de sesión con el que se retomará posteriormente |

</details>

Iniciar Claude dentro de la sesión. Con `Ctrl-b`, seguido de `d`, se separa de la sesión sin terminarla; Claude sigue ejecutándose. Para volver:

```bash
tmux attach -t claude
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `attach` | Vuelve a conectar la terminal con una sesión en ejecución |
| `-t claude` | Selecciona la sesión de destino por su nombre |

</details>

De este modo, una tarea en curso sobrevive a conexiones interrumpidas, cambios de dispositivo y al descanso nocturno del portátil.

## 9. Higiene de datos durante la migración

La parte más delicada al trasladarse al servidor no es la técnica, sino qué se lleva consigo. Tres reglas:

- **No almacenar claves privadas en el servidor.** En `authorized_keys` solo hay claves públicas. Las claves privadas permanecen en los dispositivos finales.
- **No copiar credenciales indiscriminadamente.** Los archivos locales sensibles, como `.credentials.json`, no deben trasladarse al VPS sin comprobación. En su lugar, hay que autenticarse de nuevo en el servidor.
- **Primero llevar la configuración a una carpeta de migración.** No escribir directamente las memorias y la configuración existentes de Claude en las rutas de configuración activas; transferirlas primero a una carpeta de migración separada y comprobar allí qué se desea adoptar realmente. Lo que ya no se necesita, como entradas MCP antiguas o ajustes huérfanos, se deja deliberadamente atrás en lugar de arrastrarlo sin más.

## 10. Vistas previas web mediante un túnel SSH

Para vistas previas web, por ejemplo de un servidor de desarrollo local iniciado por Claude, es grande la tentación de abrir simplemente otro puerto. No se debe hacer. Cada puerto abierto adicional es superficie de ataque adicional. En su lugar, la vista previa se ejecuta mediante un túnel de puerto SSH cifrado: el servicio escucha solo localmente en el servidor y SSH lo reenvía al cliente.

Hacer accesible desde el PC un servicio que se ejecuta localmente en el puerto 4321:

```bash
ssh -p 61417 -L 4321:localhost:4321 claude@SERVER
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-p 61417` | Puerto en el que escucha el servidor SSH (el elegido en el paso 5) |
| `-L 4321:localhost:4321` | Reenvío de puerto local: las conexiones al puerto local 4321 se reenvían a través del túnel a `localhost:4321` desde la perspectiva del servidor |
| `claude@SERVER` | Usuario y host de destino de la conexión SSH |

</details>

Después, abrir `http://localhost:4321` en el navegador local. El tráfico circula íntegramente por la conexión SSH autenticada existente, sin tener que abrir un solo puerto adicional en el firewall.

## Acceso desde el iPhone

El acceso durante los desplazamientos funciona con el mismo modelo de seguridad que desde el PC. Solo se necesita un cliente SSH con gestión de claves. Son habituales **Termius**, **Blink Shell** y **Secure ShellFish**; todos pueden generar claves Ed25519 y almacenarlas en el llavero de iOS, en parte protegidas con Face ID.

El procedimiento corresponde al paso 3, solo que en el iPhone:

1. Generar una clave Ed25519 propia para el iPhone en el cliente SSH; no copiar la clave del PC. La clave privada permanece en el llavero del dispositivo.
2. Añadir la clave pública del iPhone como línea adicional en `~/.ssh/authorized_keys` en el servidor, con un comentario descriptivo (`iphone-15`).
3. Crear la conexión en el cliente: dirección del servidor, usuario `claude`, puerto `61417`, la clave del iPhone como autenticación.

Precisamente por eso merece la pena una clave independiente por dispositivo: si se pierde el iPhone, se elimina en el servidor la línea de `iphone-15` de `authorized_keys`, y el dispositivo queda bloqueado, mientras que el acceso desde el PC y todas las demás claves siguen funcionando sin cambios.

Tras conectarse, recuperar la sesión de Claude en ejecución con `tmux attach -t claude` y continuar trabajando donde se dejó en el escritorio. El túnel de puerto del paso 10 también funciona desde iOS; Termius y Secure ShellFish admiten reenvío de puertos.

## Lista de comprobación

En resumen, el procedimiento completo:

1. Debian 13 instalado y completamente actualizado con `apt full-upgrade`.
2. Usuario propio `claude` con permisos sudo; ya no se utiliza el inicio de sesión directo de root.
3. Claves Ed25519 protegidas con frase de contraseña, una por dispositivo, solo claves públicas en `authorized_keys`.
4. sshd reforzado: `PermitRootLogin no`, `PasswordAuthentication no`; comprobado con `sshd -t` antes de recargar y mantenida abierta la sesión existente hasta la prueba.
5. SSH en el puerto 61417; con activación por socket se establece en `ssh.socket`, de lo contrario en la configuración de sshd.
6. Firewall del proveedor: DROP predeterminado para conexiones entrantes, única excepción TCP 61417; salidas permitidas.
7. Superficie de ataque comprobada con `ss -lntup` (solo SSH público, `chronyd` local) y `systemctl --failed` (sin errores).
8. Claude Code autenticado de nuevo en el servidor, operación en una sesión de `tmux`.
9. Higiene de datos: sin claves privadas ni credenciales en el servidor; configuración comprobada primero mediante una carpeta de migración.
10. Sin puertos adicionales; las vistas previas web se ejecutan mediante un túnel SSH.

Tras esta configuración, desde el exterior solo es accesible SSH en el puerto establecido, y también allí exclusivamente con una clave protegida con frase de contraseña. Claude Code se ejecuta independientemente del dispositivo final en `tmux`; las vistas previas web siguen siendo accesibles mediante túneles SSH sin abrir un puerto adicional.

## Historial de cambios

- **25 de octubre de 2026:** Corregido: Claude Code no requiere un entorno de ejecución Node.js independiente; se añaden el instalador nativo recomendado y el repositorio apt firmado para Debian. Se actualizaron los enlaces de Fuentes a las nuevas direcciones de la documentación de Claude Code.

## Fuentes

1.  [OpenSSH Manual – sshd_config(5)](https://man.openbsd.org/sshd_config): Referencia de todas las directivas de sshd, incluidas `PermitRootLogin`, `PasswordAuthentication` y `PubkeyAuthentication`.

2.  [Debian Wiki – SSH](https://wiki.debian.org/SSH): Indicaciones específicas de Debian sobre la configuración SSH, incluidos los archivos drop-in bajo `/etc/ssh/sshd_config.d/`.

3.  [systemd.socket(5) – freedesktop.org](https://www.freedesktop.org/software/systemd/man/latest/systemd.socket.html): Funcionamiento de la activación por socket y de la directiva `ListenStream=`, relevante para el cambio de puerto SSH en Debian 13.

4.  [ss(8) – iproute2 Manpage](https://man7.org/linux/man-pages/man8/ss.8.html): Opciones de `ss` para enumerar sockets en escucha junto con su proceso y dirección de enlace.

5.  [Claude Code – Documentación oficial](https://code.claude.com/docs/en/overview): Instalación, autenticación y funcionamiento de Claude Code.

6.  [Claude Code – Advanced setup](https://code.claude.com/docs/en/setup): Requisitos del sistema, instalador nativo, repositorio apt y actualizaciones automáticas.
