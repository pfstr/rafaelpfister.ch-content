---
title: "Controles clave para administradores de Totemomail: detener el servidor, revisar las colas y depurarlas de forma controlada"
navTitle: "Controles de Totemomail"
description: "Los controles más importantes para operar una puerta de enlace de totemomail: detener el servicio mediante systemd y el script de control de Tanuki, contar las colas por repositorio, inspeccionar mensajes individuales, depurarlas de forma controlada y volver a iniciar el servicio."
date: "2026-08-28"
kategorie: "Totemomail"
timeToRead: "9 min de lectura"
themen:
  - totemomail
produkte:
  - "totemomail"
  - "apache-james"
protokolle:
  - "smtp"
  - "troubleshooting"
slug: "controles-clave-para-administradores-de-totemomail-detener-el-servidor-revisar-las-colas-y"
translationId: "article-3a0a526ab6e38a06"
translationOf: totemomail-server-stoppen-queues-bereinigen
url: https://rafaelpfister.ch/es/blog/controles-clave-para-administradores-de-totemomail-detener-el-servidor-revisar-las-colas-y
translationSourceHash: bc887dcd4aa82db7e020247f75b86528f0fa331e1643c28a215a1638587197a6
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:41:01.632Z
translationReview: automatic
---

# Controles clave para administradores de Totemomail: detener el servidor, revisar las colas y depurarlas de forma controlada

Para operar una puerta de enlace de totemomail (actualmente Kiteworks Email Protection Gateway), hay cuatro pasos de trabajo esenciales: detener correctamente el servicio, registrar el estado de las colas, inspeccionar mensajes individuales y depurar las colas de forma controlada antes de volver a iniciar el servicio.

Estos pasos son necesarios tanto durante el mantenimiento planificado como ante incidencias, por ejemplo, cuando una regla defectuosa, un destino inaccesible o una prueba de carga ha llenado las colas. Este artículo muestra cada paso con los comandos concretos, incluida la cuestión de cómo detener correctamente el servicio. El modelo de procesamiento subyacente (procesadores, repositorios, formatos de archivo) se describe en el artículo [Comprender el enrutamiento de correo entre totemomail y Exchange Online](/blog/totemomail-m365) descrito.

Todas las rutas hacen referencia a una instalación en `/opt/totemomail` con el usuario de servicio `totemo`. Adapte las rutas a su entorno.

## Cómo se inicia y detiene totemomail

Antes de detener un servicio, debe saber cómo se ejecuta. En totemomail intervienen tres capas:

- Una **unidad systemd** `totemomail.service` como nivel de control más externo.
- El **script de control** `/opt/totemomail/bin/totemomail`, que invoca la unidad al iniciar y detener.
- El **Tanuki Java Service Wrapper**: un proceso `wrapper` nativo que inicia, supervisa y puede reiniciar el proceso Java propiamente dicho en caso de fallo.

Puede comprobar esta estructura en su sistema sin tener permiso para leer el archivo de unidad. `systemctl show` consulta las propiedades directamente a systemd y funciona incluso si el archivo en `/etc/systemd/system/` solo puede leerlo root:

```bash
systemctl show totemomail.service -p Type -p User -p ExecStart -p ExecStop \
  -p KillMode -p TimeoutStopUSec --no-pager
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `show totemomail.service` | Muestra las propiedades de ejecución de la unidad tal como systemd las ha cargado |
| `-p <Property>` | Limita la salida a la propiedad indicada; puede especificarse varias veces |
| `--no-pager` | Imprime directamente en la consola en lugar de abrir un paginador como `less` |

</details>

Una salida típica tiene este aspecto:

```text
Type=oneshot
TimeoutStopUSec=1min 30s
ExecStart={ path=/opt/totemomail/bin/totemomail ; argv[]=/opt/totemomail/bin/totemomail start ; ... }
ExecStop={ path=/opt/totemomail/bin/totemomail ; argv[]=/opt/totemomail/bin/totemomail stop ; ... }
User=totemo
KillMode=control-group
```

De ella se pueden extraer las propiedades importantes: `systemctl stop totemomail` invoca el script de control con el argumento `stop`, espera hasta 90 segundos a que finalice correctamente y, después, termina mediante `KillMode=control-group` todos los procesos restantes de la unidad. Por tanto, detener mediante systemd equivale a invocar directamente el script, pero además realiza una limpieza si el script se queda bloqueado.

El estado `active (exited)` en `systemctl status totemomail` es normal en esta configuración y no es un error: la unidad es `Type=oneshot`, el script de inicio termina tras iniciar el servicio y el wrapper sigue ejecutándose como un demonio independiente que systemd solo administra indirectamente. Por ello, el estado de la unidad no indica si el servicio está realmente activo; la lista de procesos sí lo hace:

```bash
ps -ef | grep -E 'wrapper|TotemoBootStrapper' | grep -v grep
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-e` | Muestra todos los procesos, no solo los de la sesión actual |
| `-f` | Formato de salida completo con la línea de comandos íntegra |
| `grep -E 'wrapper\|TotemoBootStrapper'` | Filtra por el proceso wrapper y la clase principal de Java |
| `grep -v grep` | Elimina de la lista de resultados los propios procesos grep |

</details>

En funcionamiento normal aparecen dos procesos: el `wrapper` nativo (iniciado con `../conf/wrapper.conf` y el archivo PID `totemomail.pid`) y el proceso Java con la clase principal `ch.totemo.bootstrapper.TotemoBootStrapper`. Si falta uno de los dos, el servicio no se ha iniciado por completo.

## Paso 1: detener el servicio

Antes de realizar cualquier trabajo en las colas, detenga primero el servicio. Mientras totemomail está en ejecución, acepta mensajes, procesa las colas y entrega correo; solo al detenerlo se congela el estado para el análisis.

```bash
sudo systemctl stop totemomail
```

A continuación, compruebe que los procesos wrapper y Java han finalizado:

```bash
ps -ef | grep -E 'wrapper|TotemoBootStrapper' | grep -v grep
```

La salida debe estar vacía. Además, desaparece el archivo PID `/opt/totemomail/bin/totemomail.pid`. Si un proceso sigue activo tras expirar el tiempo de espera de detención, systemd lo termina mediante el grupo de control; en ese caso, revise `journalctl -u totemomail` antes de continuar.

No olvide el nivel anterior: durante la detención, los mensajes recién recibidos se acumulan en el sistema que los entrega, por ejemplo, en la cola de Exchange o en el relay anterior. Esto es intencionado. Los remitentes fiables vuelven a entregar automáticamente tras el reinicio.

## Paso 2: registrar el estado de las colas

Las colas de totemomail son repositorios de correo basados en archivos del Apache James subyacente. Se encuentran dentro del directorio de aplicación de James, aquí `/opt/totemomail/mailer/apps/james/var/mail/`. Cada subdirectorio es un repositorio; cada mensaje consta de dos archivos: `*.FileStreamStore` contiene el mensaje MIME completo y `*.FileObjectStore` el objeto de estado serializado con metadatos.

Puede obtener una vista general contando los archivos `FileObjectStore` por directorio:

```bash
for d in /opt/totemomail/mailer/apps/james/var/mail/*/; do \
  printf '%-22s %s\n' "$(basename "$d")" \
  "$(find "$d" -maxdepth 1 -name '*.FileObjectStore' | wc -l)"; \
done
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `for d in .../*/` | Itera por todos los directorios de repositorios (el `/` final restringe el resultado a directorios) |
| `printf '%-22s %s\n'` | Formatea la salida en dos columnas; `%-22s` rellena el nombre alineado a la izquierda hasta 22 caracteres |
| `basename "$d"` | Reduce la ruta completa al nombre del directorio |
| `find "$d" -maxdepth 1` | Busca solo directamente en el directorio, sin subdirectorios |
| `-name '*.FileObjectStore'` | Cuenta un archivo por mensaje; el equivalente de stream duplicaría la cifra |
| `wc -l` | Cuenta los archivos encontrados |

</details>

El resultado es una línea por cola con el número de mensajes, por ejemplo:

```text
DBUnavailable          0
error                  12
incoming               121
outgoing               0
spool                  0
```

Los repositorios estándar significan lo siguiente: `spool` contiene mensajes aceptados aún no procesados, `incoming` mensajes que deben entregarse internamente, `outgoing` mensajes salientes, `error` mensajes fallidos y `DBUnavailable` mensajes aparcados debido a un backend inaccesible. Según la configuración, pueden existir otros repositorios para rutas especiales; todos siguen el mismo esquema de archivos.

Si `find` se ejecuta desde un directorio al que el usuario de servicio no tiene acceso (por ejemplo, el directorio personal de otro usuario tras `sudo -u totemo`), aparece en cada llamada la advertencia `Failed to restore initial working directory`. Es inofensiva y desaparece después de ejecutar `cd ~`.

## Paso 3: examinar los mensajes

Los números por sí solos no bastan para tomar una decisión. Antes de borrar nada, debe saber qué hay en las colas: ¿mensajes no deseados a causa de una incidencia o correos legítimos que deberían entregarse tras el reinicio?

Los archivos `FileStreamStore` son mensajes RFC 822 sin modificar. Por ello, los encabezados más importantes se pueden consultar directamente:

```bash
for f in /opt/totemomail/mailer/apps/james/var/mail/incoming/*.FileStreamStore; do \
  awk 'BEGIN{IGNORECASE=1} /^(From|To|Subject|Date):/{print} /^\r?$/{exit}' "$f"; \
  echo ---; \
done | less
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `BEGIN{IGNORECASE=1}` | Compara los nombres de encabezados sin distinguir entre mayúsculas y minúsculas (GNU awk) |
| `/^(From\|To\|Subject\|Date):/{print}` | Emite solo las cuatro líneas de encabezado relevantes |
| `/^\r?$/{exit}` | Se detiene en la línea vacía entre el encabezado y el cuerpo; no se lee el contenido del mensaje |
| `echo ---` | Línea separadora entre mensajes |
| `less` | Permite paginar en lugar de desplazarse por muchos mensajes |

</details>

En volúmenes grandes, la distribución es más reveladora que la vista individual. Los remitentes más frecuentes se muestran con:

```bash
grep -him1 '^From:' /opt/totemomail/mailer/apps/james/var/mail/incoming/*.FileStreamStore \
  | sort | uniq -c | sort -rn | head
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-h` | Suprime los nombres de archivo en la salida para que los remitentes idénticos se agrupen |
| `-i` | Ignora las diferencias entre mayúsculas y minúsculas |
| `-m1` | Solo la primera coincidencia por archivo (el encabezado, no líneas `From:` citadas en el cuerpo) |
| `sort \| uniq -c` | Agrupa líneas de remitente idénticas y las cuenta |
| `sort -rn \| head` | Ordena de forma descendente por frecuencia y muestra los diez más frecuentes |

</details>

Si domina un único remitente o un único asunto con cientos de copias, esto indica un bucle o un envío masivo mal dirigido; esos mensajes son candidatos para la depuración. Consultar las marcas de tiempo de los archivos (`ls -lt`) también delimita el periodo y muestra si hay mensajes legítimos más antiguos entre ellos.

## Paso 4: depurar de forma controlada

Solo ahora se borra y, aun así, con un paso intermedio: primero mueva el contenido a un directorio de copia de seguridad en lugar de eliminarlo directamente. El resultado para el funcionamiento del correo es el mismo (la cola queda vacía), pero el paso es reversible y posteriormente se pueden restaurar mensajes legítimos individuales desde la copia de seguridad o seguir utilizándolos como `.eml`.

```bash
mkdir -p /opt/totemomail/queue-backup-$(date +%F)
mv /opt/totemomail/mailer/apps/james/var/mail/incoming/* \
   /opt/totemomail/queue-backup-$(date +%F)/
```

Importante: los directorios de repositorios permanecen, solo se mueve su contenido. Además, los archivos stream y object de un mensaje pertenecen juntos; quien elimine solo uno de los dos dejará archivos huérfanos que generarán errores en el registro en el próximo inicio.

Si la copia de seguridad se ha comprobado o el contenido carece indudablemente de valor (por ejemplo, solo mensajes de pruebas de carga), elimine todo el contenido de las colas en todos los repositorios:

```bash
find /opt/totemomail/mailer/apps/james/var/mail/ -mindepth 2 -maxdepth 2 -type f \
  \( -name '*.FileStreamStore' -o -name '*.FileObjectStore' \) -delete
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-mindepth 2 -maxdepth 2` | Afecta solo a archivos directamente en los directorios de repositorios, no a `var/mail` propiamente dicho ni a niveles más profundos |
| `-type f` | Solo archivos normales; los directorios se conservan |
| `\( -name ... -o -name ... \)` | Ambos tipos de archivo de un mensaje, stream y objeto de estado |
| `-delete` | Elimina directamente las coincidencias; ejecútelo primero sin esta opción para revisar la lista de coincidencias |

</details>

Después, ejecute el mismo recuento que en el paso 2: todos los repositorios deben mostrar 0.

## Paso 5: volver a iniciar el servicio

```bash
sudo systemctl start totemomail
```

El inicio invoca el script de control con `start`, que daemoniza el wrapper; a continuación, el wrapper inicia el proceso Java. Compruebe ambos mediante la lista de procesos de la primera sección y revise los archivos de registro en `/opt/totemomail/bin/`: `wrapper.log` registra el inicio del wrapper y de la JVM, mientras que `console.log` y `console.err` registran las salidas de la propia aplicación.

Como cierre, realice una prueba funcional con un único mensaje de prueba a través de la puerta de enlace antes de volver a habilitar el flujo de correo habitual. Y si una regla o un bucle de correo había llenado las colas: corrija primero la causa y luego vuelva a permitir el tráfico. De lo contrario, tendrá que empezar de nuevo el registro del estado de las colas.

## Resumen

| Paso | Comando | Comprobación |
|---|---|---|
| Detener | `sudo systemctl stop totemomail` | Filtro de `ps` vacío, archivo PID eliminado |
| Contar el contenido | Bucle de `find` sobre `var/mail/*/` | Número por repositorio |
| Inspeccionar | Extracto de encabezados con `awk`, estadística de remitentes con `grep` | Separar mensajes no deseados de los legítimos |
| Depurar | `mv` a copia de seguridad, después `find ... -delete` | El recuento muestra 0 en todas partes |
| Iniciar | `sudo systemctl start totemomail` | Procesos, `wrapper.log`, mensaje de prueba |

## Fuentes

1.  [Apache James Server 2: Provided Mailets](https://james.apache.org/server/2/provided_mailets.html): Documentación de los mailets y repositorios en los que se basa la estructura de colas de totemomail.

2.  [Tanuki Software: Java Service Wrapper](https://wrapper.tanukisoftware.com/doc/english/introduction.html): Funcionamiento del wrapper, que inicia y supervisa el proceso Java de totemomail, incluido el archivo PID y `wrapper.conf`.

3.  [systemd.service(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html): Significado de `Type=oneshot`, `ExecStop` y `TimeoutStopSec` en unidades que invocan un script de control externo.

4.  [systemd.kill(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.kill.html): `KillMode=control-group` como mecanismo de seguridad que termina los procesos restantes de la unidad tras el script de detención.

5.  [RFC 5322: Internet Message Format](https://datatracker.ietf.org/doc/html/rfc5322): Estructura de los encabezados de mensajes que se consultan al inspeccionar los archivos `FileStreamStore`.
