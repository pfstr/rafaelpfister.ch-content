---
title: "Prueba de carga SMTP con destinatarios numerados: enviar cada correo de forma trazable"
navTitle: "Pruebas de carga numeradas"
description: "Una prueba de carga solo es tan buena como su evaluación. Con la opción -N, smtp-source numera cada correo mediante la dirección del destinatario sin sacrificar el rendimiento. Cómo estructurar la ejecución, cuántas sesiones tienen sentido y cómo encontrar automáticamente los números que faltan."
date: "2026-08-27"
kategorie: "SMTP y flujo de correo"
timeToRead: "8 min de lectura"
themen:
  - smtp-lasttests
  - smtp-mailflow
produkte:
  - "uebergreifend"
protokolle:
  - "smtp"
  - "testing"
slug: "prueba-de-carga-smtp-con-destinatarios-numerados-enviar-50-000-correos-de-forma-trazable"
translationId: "article-57f09c758baf6e1e"
translationOf: smtp-lasttest-nummerierte-empfaenger
translationSourceHash: 7145f2b49fb0b141d9c74d009d7c480ce4d119b4c97236e2ed7d92a39f65a1c5
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:48:18.456Z
translationReview: automatic
url: https://rafaelpfister.ch/es/blog/prueba-de-carga-smtp-con-destinatarios-numerados-enviar-50-000-correos-de-forma-trazable
---

# Prueba de carga SMTP con destinatarios numerados: enviar cada correo de forma trazable

Quien realiza una prueba de carga quiere poder responder después a dos preguntas: ¿han llegado todos los correos y, si no, cuáles faltan? Con correos de prueba idénticos solo se puede contar, y un contador con 13 mensajes ausentes no dice nada sobre cuándo ni dónde se perdieron. En cambio, si cada correo lleva un número consecutivo, el recuento se convierte en una comparación: cada número se puede localizar individualmente en los registros del sistema de destino, los huecos muestran el momento de la pérdida y se puede comprobar el orden de entrega.

La reacción refleja habitual es un script que incrementa el asunto. Funciona, pero reduce el rendimiento, pues el generador de carga `smtp-source` del paquete Postfix fija el asunto por llamada, y un bucle con una llamada por correo obliga a establecer una conexión propia para cada mensaje. La mejor identificación de mensajes ya está integrada: la opción `-N` numera la dirección del destinatario para cada mensaje dentro de una sola llamada con sesiones paralelas. Para la evaluación, la dirección del destinatario es tan útil como el asunto, pues aparece en todos los registros de seguimiento.

Esta configuración de prueba envía, a diferencia de una prueba funcional de loopback pura, a otro sistema a través de la red. Si no hay Postfix instalado en el sistema de origen, el artículo [smtp-source sin instalación de Postfix](https://rafaelpfister.ch/blog/smtp-source-ohne-postfix-installation) muestra cómo extraer las herramientas del RPM.

## Las opciones más importantes de smtp-source

Como orientación, estas son las opciones que aparecen en este artículo, traducidas libremente de la página de manual:

<details class="options-details">
<summary>Resumen de opciones</summary>

| Opción | Significado |
|---|---|
| `-s n` | Número de sesiones SMTP paralelas (predeterminado: 1) |
| `-m n` | Número total de mensajes que se enviarán (predeterminado: 1) |
| `-l n` | Tamaño del cuerpo del mensaje en bytes, sin cabeceras |
| `-f adresse` | Dirección del remitente |
| `-t adresse` | Dirección del destinatario (predeterminada: `foo@hostname`) |
| `-S text` | Línea de asunto, fija para todos los mensajes de la llamada |
| `-F datei` | Envía las cabeceras y el cuerpo sin modificar desde un archivo; sustituye `-l` y `-S` |
| `-N` | Numera la dirección del destinatario por mensaje (contador por proceso; la posición y el valor inicial dependen de la versión, véase más abajo) |
| `-r n` | Número de destinatarios por mensaje (predeterminado: 1), generación de direcciones como con `-N` |
| `-d` | No desconecta después de un mensaje; envía el siguiente por la misma conexión |
| `-c` | Muestra un contador en curso que aumenta con cada `DATA` completado |
| `-w n` | Tiempo de espera fijo de n segundos entre mensajes (por sesión) |
| `-v` | Salida detallada para la resolución de problemas |
| `host:port` | Destino de la entrega mediante TCP; sin indicar puerto se usa el puerto SMTP estándar |

</details>

La lista completa, incluidas las opciones de TLS, LMTP y temporización, se encuentra en la página de manual de `smtp-source(1)`; su equivalente para el lado receptor es `smtp-sink(1)` y se utilizará más abajo en la evaluación.

## Cómo -N numera los destinatarios

`-N` activa un contador por proceso que se incorpora a la dirección del destinatario. Tres características determinan la configuración de la prueba, y las tres se pueden consultar en el código fuente de `smtp-source.c`:

En primer lugar, el formato exacto de la dirección depende de la versión de Postfix. Postfix 3.5, como el que incluye RHEL 8, antepone el número a toda la dirección (`RCPT TO:<%d%s>`): de `-t test@example.com` se obtienen `1test@example.com`, `2test@example.com` y así sucesivamente, y el contador empieza en 1. Las versiones actuales de Postfix añaden en cambio el número al final de la parte local y empiezan en 0 (`test0@` hasta `test49999@`); para esta variante, la página de manual recomienda el direccionamiento con plus (`-t 'test+@example.com'` produce `test+0@` y los siguientes), de modo que un sistema de destino con subdireccionamiento asigne todo al mismo buzón. Compruebe el formato antes de la ejecución grande con unos pocos correos contra un `smtp-sink` o en el registro del destino; de ello dependen la cantidad esperada y el patrón de búsqueda de la evaluación.

En segundo lugar, el contador es global al proceso y lo comparten todas las sesiones paralelas. Con `-s 8`, las ocho sesiones asignan los números en común y cada número aparece exactamente una vez. El orden entre las sesiones no es determinista, pero la integridad del conjunto de números está garantizada.

En tercer lugar, el valor inicial no se puede configurar: 1 en Postfix 3.5 y 0 en las versiones actuales. Por tanto, los correos llevan los números del 1 al total indicado por `-m`, o del 0 al total menos 1, y la cantidad esperada para la comparación debe ajustarse a ello.

## La ejecución de prueba en una sola llamada

La cantidad de correos que abarca la ejecución no cambia el procedimiento; `-m` determina el total, y los ejemplos de este artículo usan 50'000 como marcador arbitrario.

```bash
smtp-source -c -d -N -s 8 -m 50000 -l 5120 \
  -f lasttest@example.com \
  -t test@example.com \
  gateway.example.com:25
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-c` | Contador en curso de entregas completadas como indicador de progreso de una sola línea |
| `-d` | Las conexiones permanecen abiertas durante todos los mensajes; sin `-d`, se abre una conexión nueva por mensaje |
| `-N` | Numeración de destinatarios: añade el contador por proceso a la parte local |
| `-s 8` | Ocho sesiones SMTP paralelas |
| `-m 50000` | Número total de mensajes, distribuidos entre las sesiones |
| `-l 5120` | Tamaño del mensaje en bytes (sin cabeceras), aquí 5 KB |
| `-f` | Dirección del remitente |
| `-t` | Dirección base del destinatario; `-N` la convierte en `1test@`, `2test@` y así sucesivamente (Postfix 3.5), o en `test0@`, `test1@` y así sucesivamente (versiones actuales) |
| `gateway.example.com:25` | Host y puerto de destino |

</details>

`-d` es decisivo para el perfil de carga: sin esta opción, `smtp-source` cierra la conexión después de cada mensaje y abre una nueva para el siguiente; con `-d`, las ocho conexiones permanecen abiertas y entregan todos los mensajes uno tras otro, como lo hace un remitente masivo.

Se omite deliberadamente el conocido `-v` de las pruebas funcionales: registra cada diálogo SMTP individual desde `HELO` hasta `QUIT` y genera cientos de miles de líneas de registro en una ejecución grande sin aportar valor a la evaluación. En su lugar, `-c` proporciona el resumen con el que se puede seguir el progreso de la ejecución en tiempo real. Un `time` antepuesto proporciona la duración total para calcular la tasa.

El requisito para todo el enfoque es que el sistema de destino acepte las direcciones generadas. Un `smtp-sink`, un dominio catch-all, un dominio de descarte del proveedor o una puerta de enlace que resuelva los destinatarios solo tras la aceptación cumplen este requisito. En cambio, si el destino comprueba cada destinatario contra un directorio, rechazará las direcciones numeradas y solo quedará la variante del asunto.

## Establecer cabeceras propias

Algunas pruebas de carga necesitan una cabecera propia, por ejemplo como marcador para que la puerta de enlace identifique los correos de prueba o se aplique una regla. `smtp-source` no tiene una opción para ello, pero `-F` lee un mensaje completamente preformateado de un archivo, donde puede incluirse cualquier cabecera deseada. El archivo consta de las líneas de cabecera, una línea vacía y el cuerpo, con todas las líneas terminadas en `\r\n`:

```bash
{ printf 'X-Lasttest: aktiv\r\n'
  printf 'Subject: Lasttest\r\n'
  printf '\r\n'
  head -c 5120 /dev/zero | tr '\0' 'x'
  printf '\r\n'; } > lasttest.eml
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `head -c 5120` | Genera los primeros 5120 bytes de la entrada, aquí de `/dev/zero` |
| `tr '\0' 'x'` | Sustituye cada byte nulo por el carácter `x` y genera así el texto de relleno de 5 KB |
| `> lasttest.eml` | Escribe el mensaje compuesto en el archivo para `-F` |

</details>

```bash
smtp-source -c -d -N -s 8 -m 50000 -F lasttest.eml \
  -f lasttest@example.com \
  -t test@example.com \
  gateway.example.com:25
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-F datei` | Envía las cabeceras y el cuerpo sin modificar desde el archivo; sustituye el contenido de mensaje generado |

</details>

Dos consecuencias: `-F` sustituye `-l` y `-S`, porque el tamaño y el asunto ahora proceden del archivo (por eso ambos deben incluirse allí). En cambio, `-N` sigue activo y los destinatarios continúan numerándose; la cabecera es idéntica en todos los mensajes, ya que procede del archivo fijo.

## ¿Cuántas sesiones?

La forma más fiable de determinar el número de sesiones adecuado es medirlo, usando exactamente las mismas opciones que en la ejecución principal prevista: misma fuente de mensajes (el mismo archivo `-F` o el mismo `-l`), mismo remitente, mismo destino. Solo se reduce la cantidad a 2'000 por nivel y se varía `-s`. Una breve ejecución de calibración con un número creciente de sesiones muestra a partir de cuándo las sesiones adicionales ya no aportan nada:

```bash
for s in 1 2 4 8 16 32; do
  t0=$(date +%s%N)
  smtp-source -d -N -s "$s" -m 2000 -F lasttest.eml \
    -f lasttest@example.com -t '@blackhole.example.com' \
    gateway.example.com:25
  t1=$(date +%s%N)
  echo "$s Sessions: $(( 2000000000000 / (t1 - t0) )) Mails/s"
done
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `date +%s%N` | Genera los segundos Unix seguidos directamente por la parte de nanosegundos como un único número |
| `-d` | Las conexiones permanecen abiertas durante todos los mensajes del nivel |
| `-N` | Numeración de destinatarios mediante el contador por proceso |
| `-s "$s"` | Número de sesiones paralelas, de 1 a 32 por iteración del bucle |
| `-m 2000` | 2'000 mensajes por nivel de medición |
| `-F lasttest.eml` | El mismo archivo de mensajes que en la ejecución principal prevista |
| `-f` | Dirección del remitente |
| `-t '@blackhole.example.com'` | Dirección base del destinatario con parte local vacía en un dominio de descarte |
| `gateway.example.com:25` | Host y puerto de destino |

</details>

Dos detalles de la llamada: aquí se prescinde deliberadamente de `-c` para que no aparezcan contadores en curso entre las líneas de medición; el bucle entrega exactamente una línea de resultado por nivel. Además, la parte local vacía en `-t` funciona bien junto con la numeración en un dominio de descarte: con el contador antepuesto de Postfix 3.5 se obtienen direcciones de destinatario puramente numéricas (`1@blackhole.example.com`, `2@…`), lo que mantiene clara la evaluación en los registros.

En detalle, ocurre lo siguiente: el bucle exterior recorre los números de sesiones de 1 a 32 en incrementos de duplicación. Antes y después de cada ejecución, `date +%s%N` guarda la hora actual como un número grande: los segundos Unix seguidos directamente por la parte de nanosegundos. Entre ambas operaciones, `smtp-source` envía 2'000 mensajes (el contenido, las cabeceras y el tamaño proceden del archivo `-F`) mediante el respectivo número de conexiones paralelas que permanecen abiertas gracias a `-d`; el bucle espera hasta que la llamada termina por completo. La línea de `echo` convierte la diferencia de tiempo en una tasa: 2'000 correos divididos por la duración de ejecución en segundos, aunque esta se expresa en nanosegundos. De 2'000 por 10⁹ se obtiene la constante `2000000000000`. La aritmética de Bash `$(( ))` calcula con enteros y trunca los decimales, lo cual es suficientemente preciso para esta medición.

Tres indicaciones prácticas: `%N` solo proporciona nanosegundos con GNU date (como ocurre en RHEL y en la mayoría de los sistemas Linux; BusyBox y macOS no lo admiten). La ejecución completa envía 6 × 2'000 = 12'000 correos, que también necesitan una dirección de destinatario controlada, y la numeración `-N` vuelve a empezar en cada llamada con el valor inicial. Si una llamada de `smtp-source` termina con un mensaje de error, la tasa de esa línea carece de significado; primero corrija la causa y vuelva a medir.

La salida esperada es una línea por nivel. Con valores de ejemplo inventados pero típicos, se ve así:

```text
1 Sessions: 11 Mails/s
2 Sessions: 21 Mails/s
4 Sessions: 40 Mails/s
8 Sessions: 71 Mails/s
16 Sessions: 79 Mails/s
32 Sessions: 80 Mails/s
```

La interpretación es la siguiente: mientras la tasa se duplica aproximadamente con el número de sesiones, las sesiones paralelas cubren el tiempo de espera de las respuestas del destino; el cuello de botella es entonces la latencia del trayecto, no la capacidad. A partir del punto en el que la curva se aplana (en el ejemplo, entre 8 y 16 sesiones), el sistema de destino está saturado o la fuente ha alcanzado su límite. Elija el valor más pequeño a partir del cual la tasa ya no aumente de forma apreciable, en el ejemplo de 8 a 16; más sesiones solo aumentan la carga por paralelismo, no el rendimiento. Para la ejecución principal, la tasa medida también permite estimar la duración esperada: el total de `-m` dividido por la tasa.

## Evaluación en el lado receptor

Si hay un receptor de prueba propio en el sistema de destino, `smtp-sink` también se encarga del registro:

```bash
smtp-sink -c -d "mails/%Y%m%d-%H%M%S." 0.0.0.0:2525 200
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-c` | Contadores en curso en lugar del diálogo SMTP completo |
| `-d "mails/…"` | En el sink: volcado, no mantenimiento de conexión. Escribe cada mensaje aceptado en un archivo independiente (patrón de nombre mediante strftime), incluida una cabecera `X-Rcpt-Args` con la dirección del destinatario |
| `0.0.0.0:2525` | Escucha en todas las interfaces en el puerto 2525 |
| `200` | Backlog: longitud máxima de la cola de conexiones en espera según listen(2) |

</details>

Tras la ejecución, extraiga los números recibidos y compárelos con la cantidad esperada. Como los números no llevan ceros iniciales, ambas listas se rellenan a una longitud fija antes de compararlas, para que la ordenación alfabética de `comm` corresponda a la numérica. El patrón de búsqueda se ajusta al formato de dirección de Postfix 3.5 (número antes de la dirección); en las versiones actuales, use respectivamente `test[0-9]+@` y `seq` a partir de 0:

```bash
grep -rhoE '[0-9]+test@example\.com' mails/ | \
  grep -oE '^[0-9]+' | sort -u | \
  awk '{printf "%08d\n", $1}' | sort > empfangen.txt
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `grep -r` | Busca recursivamente en el directorio `mails/` |
| `grep -h` | Suprime los nombres de archivo antes de las coincidencias |
| `grep -o` | Genera solo la parte de dirección coincidente, no la línea completa |
| `grep -E` | Expresiones regulares extendidas, aquí para `[0-9]+` |
| `sort -u` | Ordena y elimina duplicados (cada número una vez) |
| `awk '{printf "%08d\n", $1}'` | Rellena cada número con ceros iniciales hasta ocho posiciones |
| `sort` | Ordena los números rellenados para compararlos con `comm` |

</details>

```bash
seq 1 50000 | awk '{printf "%08d\n", $1}' | \
  comm -23 - empfangen.txt
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `seq 1 50000` | Genera la cantidad esperada de números; el valor final corresponde al total enviado indicado por `-m` |
| `comm -23` | Suprime la columna 2 (solo en el archivo 2) y la columna 3 (en ambos); quedan las líneas presentes solo en la cantidad esperada |
| `-` | Lee la primera lista de comparación desde la tubería en lugar de un archivo |
| `empfangen.txt` | Segunda lista de comparación: los números realmente recibidos |

</details>

`comm -23` muestra exactamente los números que están en la cantidad esperada pero no en la lista de recepción: los correos que faltan. Una salida vacía significa entrega completa. Si aparecen números duplicados (reconocibles por la diferencia entre `sort` y `sort -u`), algún sistema ha duplicado el mensaje durante el trayecto, lo que también es un hallazgo.

Si el destino es un sistema similar a producción en lugar de un smtp-sink, su registro asume el papel de los archivos de volcado. En un servidor Exchange, por ejemplo, `Get-MessageTrackingLog -Recipients` o un filtro por la dirección del destinatario proporciona los números recibidos; en un sistema Postfix, un `grep` sobre `to=` y la dirección base a través del registro de correo. Esta es precisamente la ventaja del número en la dirección: el destinatario aparece en todo seguimiento de mensajes, mientras que el asunto puede faltar según el sistema o requerir activación previa.

## Cuando el número debe estar en el asunto

Algunas evaluaciones dependen del asunto, por ejemplo cuando el sistema de destino reescribe las direcciones de destinatario o los registros muestran el destinatario solo de forma enmascarada. Entonces queda la variante de bucle: una llamada de `smtp-source` por correo con `-m 1` y un asunto que la shell incrementa, distribuida entre varios workers paralelos con rangos de números contiguos.

```bash
worker() {
  local i
  for ((i = $1; i <= $2; i++)); do
    smtp-source -s 1 -m 1 -l 5120 \
      -S "$(printf 'Lasttest %05d' "$i")" \
      -f lasttest@example.com -t test@example.com \
      gateway.example.com:25 || echo "$i" >> fehlend.log
  done
}
for w in 0 1 2 3; do
  worker $(( w * 12500 + 1 )) $(( (w + 1) * 12500 )) &
done
wait
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-s 1` | Una sesión por llamada; los cuatro workers proporcionan el paralelismo |
| `-m 1` | Exactamente un mensaje por llamada, para poder establecer el asunto por correo |
| `-l 5120` | Tamaño del mensaje en bytes (sin cabeceras), aquí 5 KB |
| `-S "$(printf 'Lasttest %05d' "$i")"` | Asunto con el número consecutivo rellenado a cinco posiciones |
| `-f` / `-t` | Dirección del remitente y del destinatario |
| `gateway.example.com:25` | Host y puerto de destino |

</details>

El coste es un establecimiento completo de conexión por correo: handshake TCP, banner, `HELO`, envío, `QUIT`. Por tanto, esta ejecución no mide el rendimiento máximo del sistema de destino, sino un caso deliberadamente intensivo en conexiones. Determine el número de workers de forma análoga a la ejecución de calibración anterior, solo que con el bucle de workers en vez de `-s`. Los ceros iniciales en el asunto evitan tener que reformatear durante la comparación, como necesita la variante `-N`.

## Reglas para pruebas contra otros sistemas

En cuanto la prueba sale del sistema propio, se aplican tres condiciones. Primera: el operador del sistema de destino debe estar informado y haber aceptado la ventana temporal; una prueba de carga parece un ataque o una oleada de spam para cualquier monitorización. Segunda: la dirección de destinatario debe terminar de forma controlada, en un buzón de prueba dedicado, una regla de descarte en el destino o un dominio de descarte previsto para ello por el proveedor; las direcciones de producción no pertenecen a una prueba de carga. Tercera: antes de empezar debe estar definido un criterio de interrupción, por ejemplo una cola creciente en el destino o una tasa de errores superior a un umbral, y alguien debe observar estos valores durante la ejecución.

Con estos tres puntos y la numeración, la ejecución no solo proporciona al final una cifra de rendimiento, sino una afirmación verificable: qué correos han llegado, cuáles faltan y dónde se vieron por última vez durante el trayecto.

## Fuentes

1.  [Postfix: smtp-source(1)](https://www.postfix.org/smtp-source.1.html): página de manual del generador de carga; describe el comportamiento de `-N` en la versión actual (contador en la parte local, direccionamiento con plus).

2.  [Código fuente de Postfix 3.5.8: smtp-source.c](https://github.com/vdukhovni/postfix/blob/v3.5.8/postfix/src/smtpstone/smtp-source.c): demuestra que, en la versión de RHEL 8, el número se antepone (`RCPT TO:<%d%s>`) con valor inicial 1; en la [versión actual](https://github.com/vdukhovni/postfix/blob/master/postfix/src/smtpstone/smtp-source.c) el número se añade en cambio a la parte local, a partir de 0.

3.  [Postfix: smtp-sink(1)](https://www.postfix.org/smtp-sink.1.html): página de manual del receptor de prueba con las opciones de volcado y las cabeceras X registradas.

4.  [GNU Coreutils: comm](https://www.gnu.org/software/coreutils/manual/html_node/comm-invocation.html): comparación de conjuntos de dos listas ordenadas, aquí para comparar los números esperados y recibidos.
