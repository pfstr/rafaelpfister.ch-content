---
title: "smtp-source sin instalar Postfix: extraer herramientas de pruebas de carga del RPM"
navTitle: "Extraer smtp-source"
description: "smtp-source y smtp-sink forman parte de Postfix, pero también funcionan sin un servidor de correo instalado. Cómo extraer ambas herramientas del paquete en RHEL, por qué la ejecución desde /tmp puede fallar debido a la opción de montaje noexec y qué bibliotecas deben incluirse."
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
url: https://rafaelpfister.ch/es/blog/smtp-source-sin-instalar-postfix-extraer-herramientas-de-pruebas-de-carga-del-rpm
translationSourceHash: 2b4bda3ea22f49c9d5269ec15b0c1dbfd779ccc6d03ad5b234aba738e5bb119f
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:24:25.507Z
translationReview: automatic
---

# smtp-source sin instalar Postfix: extraer herramientas de pruebas de carga del RPM

Para las pruebas de carga SMTP, `smtp-source` es una buena elección: la herramienta abre sesiones paralelas, las mantiene activas durante varios mensajes y, por tanto, reproduce el comportamiento de conexión de un remitente masivo de forma mucho más realista que las herramientas de prueba que establecen una conexión nueva para cada correo. Su contraparte, `smtp-sink`, acepta correos y los descarta sin entregar nada. Ambas forman parte de Postfix.

Ahí reside precisamente el problema: a menudo no hay Postfix instalado en el sistema desde el que se desea realizar las pruebas. Tampoco es deseable instalarlo en una appliance de pasarela de correo, ya que un Postfix adicional incorpora su propia configuración en `/etc/postfix` y un servicio del sistema que, en el peor de los casos, ocupa el puerto 25 y bloquea así el sistema de correo real. Además, está la cuestión de qué opina el soporte del fabricante sobre paquetes instalados posteriormente en su appliance.

Sin embargo, ambas herramientas también pueden utilizarse sin instalación: descargar el RPM, extraer los binarios junto con las bibliotecas y listo. El procedimiento tiene dos particularidades, que este artículo muestra con un sistema RHEL 8. No necesita permisos de root, solo acceso a las fuentes de paquetes.

## ¿Ya está disponible smtp-source?

Primero compruebe si la herramienta ya se encuentra en el sistema. `smtp-source` puede estar fuera del PATH habitual, según la distribución:

```bash
command -v smtp-source || \
  ls /usr/sbin/smtp-source /usr/lib/postfix/sbin/smtp-source 2>/dev/null
```

Si la salida permanece vacía, también falta el paquete correspondiente. En sistemas RPM, confirme esto y compruebe al mismo tiempo si los repositorios ofrecen Postfix:

```bash
rpm -qa | grep -i postfix
```

```bash
yum list available postfix
```

En el sistema de prueba no había Postfix instalado, pero el repositorio BaseOS ofrecía `postfix-3.5.8-8.el8_10` . Con ello, el camino queda libre: el paquete puede descargarse sin instalarlo.

## Descargar únicamente el RPM

`yum download` (del paquete de complementos `dnf-plugins-core`, normalmente presente en RHEL 8) descarga un paquete en el directorio actual sin instalarlo. Funciona sin permisos de root siempre que el directorio de destino sea escribible:

```bash
cd /tmp && yum download postfix
```

Si yum muestra `No such command: download`, falta el complemento. Con permisos de root puede lograr lo mismo mediante el comando de instalación con `--downloadonly`:

```bash
sudo yum install --downloadonly --downloaddir=/tmp postfix
```

Si no dispone de ninguna de las dos opciones, queda el rodeo a través de un segundo sistema con la misma versión de RHEL: descargue allí el RPM y cópielo al sistema de destino mediante `scp`.

## Extraer binarios y bibliotecas

`rpm2cpio` convierte el RPM en un flujo de archivo cpio, del que `cpio` extrae selectivamente rutas concretas. Además de los dos binarios, también necesita las bibliotecas de Postfix, ya que en RHEL las herramientas están enlazadas dinámicamente con `libpostfix-*.so`:

```bash
cd /tmp && rpm2cpio postfix-*.rpm | \
  cpio -idmv ./usr/sbin/smtp-source ./usr/sbin/smtp-sink \
  './usr/lib64/postfix/*'
```

Los archivos quedan después bajo `/tmp/usr/`. Las rutas comienzan por `./`, porque cpio espera las rutas exactamente como aparecen en el archivo.

## Problema 1: /tmp está montado con noexec

El intento obvio de iniciar directamente desde /tmp falla en sistemas reforzados:

```text
bash: /tmp/usr/sbin/smtp-sink: Permission denied
[1]+  Exit 126
```

El código de salida 126 pese a tener correctamente establecido el bit de ejecución es el síntoma típico de un sistema de archivos con la opción de montaje `noexec`. El kernel deniega entonces toda ejecución de programas desde ese sistema de archivos, independientemente de los permisos del archivo. Puede comprobarlo directamente:

```bash
mount | grep ' /tmp '
```

La solución: copie los binarios y las bibliotecas a un directorio cuyo sistema de archivos permita la ejecución, por ejemplo, su propio directorio home:

```bash
mkdir -p ~/bin && \
  cp /tmp/usr/sbin/smtp-source /tmp/usr/sbin/smtp-sink \
     /tmp/usr/lib64/postfix/libpostfix-*.so ~/bin/ && \
  chmod +x ~/bin/smtp-source ~/bin/smtp-sink
```

Tenga en cuenta que `noexec` también afecta a la carga de bibliotecas compartidas. Por tanto, no basta con mover solo los binarios y dejar las bibliotecas en /tmp.

## Problema 2: la ruta de bibliotecas

Sin indicaciones adicionales, el enlazador dinámico busca las bibliotecas de Postfix en `/usr/lib64/postfix`, donde no se encuentran al no existir instalación:

```text
smtp-sink: error while loading shared libraries: libpostfix-global.so:
cannot open shared object file: No such file or directory
```

`LD_LIBRARY_PATH` añade el directorio propio a la ruta de búsqueda del enlazador. La variable se antepone a cada llamada:

```bash
LD_LIBRARY_PATH=~/bin ~/bin/smtp-source ...
```

Con `ldd ~/bin/smtp-source` puede comprobar de antemano si se pueden resolver todas las dependencias. Aparte de las bibliotecas de Postfix, las herramientas solo dependen de bibliotecas estándar del sistema.

## Prueba de funcionamiento en loopback

Puede comprobar que todo funciona sin enviar un solo correo real: `smtp-sink` escucha como receptor de descarte en un puerto alto, mientras que `smtp-source` realiza la entrega. Todo el tráfico permanece en localhost.

```bash
LD_LIBRARY_PATH=~/bin ~/bin/smtp-sink -v 127.0.0.1:2525 100 &
```

```bash
LD_LIBRARY_PATH=~/bin ~/bin/smtp-source -s 2 -m 10 -l 5120 \
  -f test@example.com -t test@example.com 127.0.0.1:2525
```

| Opción | Efecto |
|---|---|
| `-v` (smtp-sink) | Registra cada paso del diálogo de las conexiones aceptadas |
| `127.0.0.1:2525` | smtp-sink escucha solo en localhost, puerto 2525 |
| `100` | Backlog: longitud máxima de la cola de conexiones en espera según listen(2) |
| `-s 2` | Dos sesiones SMTP paralelas |
| `-m 10` | Diez mensajes en total, distribuidos entre las sesiones |
| `-l 5120` | Tamaño del mensaje en bytes (sin cabecera), aquí 5 KB |
| `-f` / `-t` | Dirección del remitente y del destinatario |

Si tiene éxito, `smtp-source` no genera ninguna salida, mientras que smtp-sink muestra para cada mensaje el diálogo SMTP completo, desde `HELO` hasta `QUIT`. Después, detenga el proceso en segundo plano y elimine los restos de /tmp:

```bash
kill %1
```

```bash
rm -rf /tmp/usr /tmp/postfix-*.rpm
```

## Indicaciones para la prueba de carga real

Para mediciones de rendimiento fiables, el generador de carga debe estar en una máquina independiente del mismo segmento de red, no en el propio objeto de prueba. Si `smtp-source` se ejecuta en la pasarela evaluada, el generador y el sistema de correo compiten por CPU y E/S, y la medición refleja esa competencia en lugar de la capacidad real. Localmente en el sistema de destino, la herramienta extraída resulta útil sobre todo para pruebas funcionales de las reglas y primeras comprobaciones de plausibilidad.

En cuanto la prueba se realiza contra el puerto 25 real, se trata de correos reales que atraviesan las reglas de la pasarela y que, según la configuración, se entregan. Por ello, utilice direcciones de destinatario con un destino controlado: un buzón de prueba dedicado, una regla que descarte los remitentes de prueba o un dominio de descarte previsto por el proveedor para este fin. Las direcciones de producción no deben utilizarse en una prueba de carga.

El procedimiento descrito es aplicable, más allá de las dos herramientas SMTP, a cualquier programa de línea de comandos incluido en un paquete cuya instalación no sea una opción en el sistema de destino. La combinación de `yum download`, `rpm2cpio` y un directorio ejecutable en el home es la misma en todos los sistemas RPM.

## Fuentes

1.  [Postfix: smtp-source(1)](https://www.postfix.org/smtp-source.1.html): página de manual con todos los parámetros del generador de carga, incluido el control de sesiones y mensajes.

2.  [Postfix: smtp-sink(1)](https://www.postfix.org/smtp-sink.1.html): página de manual del receptor de prueba, entre otras cosas con opciones para retrasos artificiales y respuestas de error.

3.  [Red Hat: How to download a package without installing it](https://access.redhat.com/solutions/10154): documenta `yum download` y la alternativa `--downloadonly`.

4.  [man7.org: mount(8)](https://man7.org/linux/man-pages/man8/mount.8.html): descripción de la opción de montaje `noexec` y su efecto sobre la ejecución de programas.
