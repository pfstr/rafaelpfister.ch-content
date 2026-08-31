---
title: "smtp-source sin instalar Postfix: extraer herramientas de pruebas de carga del RPM"
navTitle: "Extraer smtp-source"
description: "smtp-source y smtp-sink forman parte de Postfix, pero también funcionan sin un servidor de correo instalado. Cómo extraer ambas herramientas del paquete en RHEL, por qué ejecutarlas desde /tmp puede fallar debido a la opción de montaje noexec y qué bibliotecas deben incluirse."
date: "2026-08-27"
kategorie: "SMTP y flujo de correo"
timeToRead: "7 min de lectura"
themen:
  - smtp-mailflow
  - smtp-lasttests
produkte:
  - "uebergreifend"
protokolle:
  - "smtp"
  - "testing"
  - "troubleshooting"
slug: "smtp-source-sin-instalar-postfix-extraer-herramientas-de-pruebas-de-carga-del-rpm"
translationId: "article-d0a27da11509d24b"
translationOf: smtp-source-ohne-postfix-installation
translationSourceHash: fd4ad6beb5036817db9b7758653a2b7d015a6adb15d7b4a0b47f94161e34b4e6
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:54:04.869Z
translationReview: automatic
url: https://rafaelpfister.ch/es/blog/smtp-source-sin-instalar-postfix-extraer-herramientas-de-pruebas-de-carga-del-rpm
---

# smtp-source sin instalar Postfix: extraer herramientas de pruebas de carga del RPM

Para las pruebas de carga SMTP, `smtp-source` es una buena opción: la herramienta abre sesiones paralelas, las mantiene abiertas durante varios mensajes y, por tanto, reproduce el comportamiento de conexión de un remitente masivo de forma mucho más realista que las herramientas de prueba que abren una conexión nueva por cada correo. Su equivalente, `smtp-sink`, acepta correos y los descarta sin entregar nada. Ambas forman parte de Postfix.

Ahí está precisamente el problema: a menudo no hay Postfix instalado en el sistema desde el que se desea realizar la prueba. Tampoco es recomendable instalarlo en un appliance de pasarela de correo, ya que un Postfix adicional incorpora su propia configuración en `/etc/postfix` y un servicio del sistema que, en el peor de los casos, ocupa el puerto 25 y bloquea así el sistema de correo propiamente dicho. Además, está la cuestión de qué opina el soporte del fabricante sobre paquetes instalados posteriormente en su appliance.

Sin embargo, ambas herramientas también pueden utilizarse sin instalación: descargar el RPM, extraer los binarios junto con las bibliotecas y listo. El proceso presenta dos particularidades que este artículo muestra en un sistema RHEL 8. No necesita permisos de root, solo acceso a las fuentes de paquetes.

## ¿Ya está disponible smtp-source?

Primero, compruebe si la herramienta ya se encuentra en el sistema. Dependiendo de la distribución, `smtp-source` se encuentra fuera del PATH habitual:

```bash
command -v smtp-source || \
  ls /usr/sbin/smtp-source /usr/lib/postfix/sbin/smtp-source 2>/dev/null
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `command -v smtp-source` | Muestra la ruta si el programa está en el PATH; de lo contrario, no muestra nada |
| `/usr/sbin/... /usr/lib/postfix/sbin/...` | Las ubicaciones habituales fuera del PATH (RHEL y Debian/Ubuntu) |
| `2>/dev/null` | Suprime los mensajes de error de `ls` para rutas inexistentes |

</details>

Si la salida permanece vacía, también falta el paquete correspondiente. En sistemas RPM, confírmelo y compruebe al mismo tiempo si los repositorios ofrecen Postfix:

```bash
rpm -qa | grep -i postfix
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-q` | Modo de consulta de rpm |
| `-a` | Enumera todos los paquetes instalados |
| `grep -i postfix` | Filtra la lista sin distinguir entre mayúsculas y minúsculas |

</details>

```bash
yum list available postfix
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `list available` | Muestra únicamente paquetes disponibles en los repositorios pero no instalados |
| `postfix` | Limita la salida al paquete buscado |

</details>

En el sistema de prueba no estaba instalado Postfix, pero el repositorio BaseOS ofrecía `postfix-3.5.8-8.el8_10` . Con ello queda libre el camino: el paquete puede descargarse sin instalarlo.

## Descargar únicamente el RPM

`yum download` (del paquete de plugins `dnf-plugins-core`, habitualmente presente en RHEL 8) descarga un paquete en el directorio actual sin instalarlo. Funciona sin permisos de root siempre que el directorio de destino sea escribible:

```bash
cd /tmp && yum download postfix
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `cd /tmp` | Cambia a un directorio escribible; `yum download` guarda el RPM en el directorio actual |
| `download` | Subcomando de `dnf-plugins-core`: descarga el paquete sin instalarlo |
| `postfix` | Nombre del paquete que se debe descargar |

</details>

Si yum informa `No such command: download`, falta el plugin. Con permisos de root puede lograr lo mismo mediante el comando de instalación con `--downloadonly`:

```bash
sudo yum install --downloadonly --downloaddir=/tmp postfix
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `--downloadonly` | Se detiene tras la descarga; no se instala nada |
| `--downloaddir=/tmp` | Directorio de destino para el RPM descargado |
| `postfix` | Nombre del paquete |

</details>

Si no dispone de ninguna de las dos opciones, queda el rodeo mediante un segundo sistema con la misma versión de RHEL: descargue allí el RPM y cópielo al sistema de destino con `scp`.

## Extraer binarios y bibliotecas

`rpm2cpio` convierte el RPM en un flujo de archivo cpio, del cual `cpio` extrae de forma selectiva rutas individuales. Además de los dos binarios, también necesita las bibliotecas de Postfix, ya que en RHEL las herramientas están enlazadas dinámicamente con `libpostfix-*.so`:

```bash
cd /tmp && rpm2cpio postfix-*.rpm | \
  cpio -idmv ./usr/sbin/smtp-source ./usr/sbin/smtp-sink \
  './usr/lib64/postfix/*'
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `rpm2cpio postfix-*.rpm` | Convierte el RPM en un flujo de archivo cpio en stdout |
| `-i` | Modo de extracción de cpio (copy-in) |
| `-d` | Crea los directorios que falten durante la extracción |
| `-m` | Conserva las fechas de modificación de los archivos |
| `-v` | Enumera cada archivo extraído |
| `./usr/sbin/smtp-source ./usr/sbin/smtp-sink` | Los dos binarios, rutas exactamente como en el archivo (con `./` inicial) |
| `'./usr/lib64/postfix/*'` | Las bibliotecas de Postfix; el patrón está entre comillas para que lo evalúe cpio y no la shell |

</details>

Los archivos quedan después bajo `/tmp/usr/`.

## Problema 1: /tmp está montado con noexec

El inicio obvio directamente desde /tmp falla en sistemas reforzados:

```text
bash: /tmp/usr/sbin/smtp-sink: Permission denied
[1]+  Exit 126
```

El código de salida 126 pese a tener correctamente establecido el bit de ejecución es el síntoma típico de un sistema de archivos con la opción de montaje `noexec`. En ese caso, el kernel rechaza toda ejecución de programas desde ese sistema de archivos, independientemente de los permisos del archivo. Puede comprobarlo directamente:

```bash
mount | grep ' /tmp '
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `mount` | Sin argumentos, enumera todos los sistemas de archivos montados con sus opciones de montaje |
| `' /tmp '` | Patrón de búsqueda con un espacio antes y después, para que solo coincida con el punto de montaje `/tmp` y no, por ejemplo, con `/var/tmp` |

</details>

La solución: copie los binarios y las bibliotecas a un directorio cuyo sistema de archivos permita la ejecución, por ejemplo, su propio directorio personal:

```bash
mkdir -p ~/bin && \
  cp /tmp/usr/sbin/smtp-source /tmp/usr/sbin/smtp-sink \
     /tmp/usr/lib64/postfix/libpostfix-*.so ~/bin/ && \
  chmod +x ~/bin/smtp-source ~/bin/smtp-sink
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `mkdir -p ~/bin` | Crea el directorio de destino; sin error si ya existe |
| `cp ... ~/bin/` | Copia los dos binarios y las bibliotecas `libpostfix-*.so` al directorio ejecutable |
| `chmod +x` | Establece el bit de ejecución en ambos binarios |

</details>

Tenga en cuenta que `noexec` también afecta a la carga de bibliotecas compartidas. Por tanto, no basta con mover solo los binarios y dejar las bibliotecas en /tmp.

## Problema 2: la ruta de las bibliotecas

Sin más indicaciones, el enlazador dinámico busca las bibliotecas de Postfix en `/usr/lib64/postfix`, donde no se encuentran al no haber instalación:

```text
smtp-sink: error while loading shared libraries: libpostfix-global.so:
cannot open shared object file: No such file or directory
```

`LD_LIBRARY_PATH` añade el directorio propio a la ruta de búsqueda del enlazador. La variable se antepone a cada invocación:

```bash
LD_LIBRARY_PATH=~/bin ~/bin/smtp-source ...
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `LD_LIBRARY_PATH=~/bin` | Añade `~/bin` a la ruta de búsqueda del enlazador dinámico para esta única invocación |
| `~/bin/smtp-source` | Invocación mediante la ruta completa, ya que `~/bin` no tiene por qué estar en el PATH |

</details>

Con `ldd ~/bin/smtp-source` puede comprobar de antemano si se pueden resolver todas las dependencias. Aparte de las bibliotecas de Postfix, las herramientas solo dependen de bibliotecas estándar del sistema.

## Prueba de funcionamiento en loopback

Puede comprobar que todo funciona sin un solo correo real: `smtp-sink` escucha como receptor de descarte en un puerto alto, mientras que `smtp-source` realiza la entrega. Todo el tráfico permanece en localhost.

```bash
LD_LIBRARY_PATH=~/bin ~/bin/smtp-sink -v 127.0.0.1:2525 100 &
```

```bash
LD_LIBRARY_PATH=~/bin ~/bin/smtp-source -s 2 -m 10 -l 5120 \
  -f test@example.com -t test@example.com 127.0.0.1:2525
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-v` (smtp-sink) | Registra cada paso del diálogo de las conexiones aceptadas |
| `127.0.0.1:2525` | smtp-sink escucha únicamente en localhost, puerto 2525 |
| `100` | Backlog: longitud máxima de la cola de conexiones en espera según listen(2) |
| `-s 2` | Dos sesiones SMTP paralelas |
| `-m 10` | Diez mensajes en total, distribuidos entre las sesiones |
| `-l 5120` | Tamaño del mensaje en bytes (sin encabezados), aquí 5 KB |
| `-f` / `-t` | Dirección del remitente y del destinatario |

</details>

Si tiene éxito, `smtp-source` no genera salida, mientras que smtp-sink muestra para cada mensaje el diálogo SMTP completo, desde `HELO` hasta `QUIT`. A continuación, detenga el proceso en segundo plano y elimine los restos de /tmp:

```bash
kill %1
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `%1` | Especificación de trabajo de la shell: finaliza el primer trabajo en segundo plano, aquí smtp-sink |

</details>

```bash
rm -rf /tmp/usr /tmp/postfix-*.rpm
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-r` | Elimina el árbol de directorios de forma recursiva |
| `-f` | Sin confirmaciones ni errores para rutas inexistentes |
| `/tmp/usr /tmp/postfix-*.rpm` | El árbol extraído y el RPM descargado |

</details>

## Indicaciones para la prueba de carga real

Para mediciones de rendimiento fiables, el generador de carga debe estar en una máquina independiente dentro del mismo segmento de red, no en el propio objeto de prueba. Si `smtp-source` se ejecuta en la pasarela evaluada, el generador y el sistema de correo compiten por CPU y E/S, y la medición muestra esa competencia en lugar de la capacidad real. Localmente en el sistema de destino, la herramienta extraída sirve sobre todo para pruebas funcionales del conjunto de reglas y para comprobaciones iniciales de plausibilidad.

En cuanto la prueba se realiza contra el puerto 25 real, se trata de correos reales que atraviesan el conjunto de reglas de la pasarela y que, según la configuración, se entregan. Por ello, utilice direcciones de destinatario que terminen en destinos controlados: un buzón de prueba dedicado, una regla que descarte los remitentes de prueba o un dominio de descarte previsto para ello por el proveedor. Las direcciones de producción no deben usarse en una prueba de carga.

El procedimiento descrito es útil más allá de las dos herramientas SMTP para cualquier programa de línea de comandos incluido en un paquete cuya instalación en el sistema de destino no sea una opción. La combinación de `yum download`, `rpm2cpio` y un directorio ejecutable en el directorio personal es igual en cualquier sistema RPM.

## Fuentes

1.  [Postfix: smtp-source(1)](https://www.postfix.org/smtp-source.1.html): página de manual con todos los parámetros del generador de carga, incluido el control de sesiones y mensajes.

2.  [Postfix: smtp-sink(1)](https://www.postfix.org/smtp-sink.1.html): página de manual del receptor de prueba, entre otras cosas con opciones para retrasos artificiales y respuestas de error.

3.  [Red Hat: How to download a package without installing it](https://access.redhat.com/solutions/10154): documenta `yum download` y la alternativa `--downloadonly`.

4.  [man7.org: mount(8)](https://man7.org/linux/man-pages/man8/mount.8.html): descripción de la opción de montaje `noexec` y su efecto sobre la ejecución de programas.
