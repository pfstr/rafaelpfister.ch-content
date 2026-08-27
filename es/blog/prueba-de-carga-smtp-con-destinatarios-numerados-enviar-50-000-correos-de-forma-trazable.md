---
title: "Prueba de carga SMTP con destinatarios numerados: enviar 50'000 correos de forma trazable"
navTitle: "Pruebas de carga numeradas"
description: "Una prueba de carga solo es tan buena como su evaluación. Con la opción -N, smtp-source numera cada correo mediante la dirección del destinatario sin sacrificar rendimiento. Cómo estructurar la ejecución con 50'000 correos, cuántas sesiones tienen sentido y cómo encontrar automáticamente los números que faltan."
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
url: https://rafaelpfister.ch/es/blog/prueba-de-carga-smtp-con-destinatarios-numerados-enviar-50-000-correos-de-forma-trazable
translationSourceHash: a2ec75884c06a6d736ea9b5895211ddc4cbba252c7ddf491752e1bec5ab1a24d
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:21:31.175Z
translationReview: automatic
---

# Prueba de carga SMTP con destinatarios numerados: enviar 50'000 correos de forma trazable

Quien realiza una prueba de carga con 50'000 correos quiere poder responder después a dos preguntas: ¿han llegado todos y, si no, cuáles faltan? Con correos de prueba idénticos solo se puede contar, y una diferencia entre 49'987 y 50'000 no dice cuándo ni dónde se perdieron los 13 mensajes que faltan. En cambio, si cada correo lleva un número consecutivo, el conteo se convierte en una comparación: cada número se puede localizar individualmente en los logs del sistema de destino, los huecos muestran el momento de la pérdida y se puede comprobar el orden de entrega.

La reacción habitual es usar un script que incrementa el asunto. Funciona, pero reduce el rendimiento, pues el generador de carga `smtp-source` del paquete Postfix fija el asunto en cada invocación, y un bucle con una invocación por correo obliga a establecer una conexión independiente para cada mensaje. La mejor identificación de mensajes ya viene incorporada: la opción `-N` numera la dirección del destinatario para cada mensaje, dentro de una única invocación con sesiones paralelas. Para la evaluación, la dirección del destinatario es tan útil como el asunto, pues aparece en todos los logs de seguimiento.

A diferencia de una prueba funcional de loopback pura, esta configuración de prueba envía a otro sistema a través de la red. Si no hay Postfix instalado en el sistema de origen, el artículo [smtp-source sin instalación de Postfix](https://rafaelpfister.ch/blog/smtp-source-ohne-postfix-installation) muestra cómo extraer las herramientas del RPM.

## Las opciones más importantes de smtp-source

Como orientación, a continuación se muestran las opciones que aparecen en este artículo, traducidas libremente de la página de manual:

| Opción | Significado |
|---|---|
| `-s n` | Número de sesiones SMTP paralelas (predeterminado: 1) |
| `-m n` | Número total de mensajes que se enviarán (predeterminado: 1) |
| `-l n` | Tamaño del texto del mensaje en bytes, sin cabeceras |
| `-f adresse` | Dirección del remitente |
| `-t adresse` | Dirección del destinatario (predeterminada: `foo@hostname`) |
| `-S text` | Línea de asunto, fija para todos los mensajes de la invocación |
| `-F datei` | Envía cabeceras y cuerpo sin modificar desde un archivo; sustituye `-l` y `-S` |
| `-N` | Numera la dirección del destinatario para cada mensaje (contador por proceso; posición y valor inicial según la versión; véase más abajo) |
| `-r n` | Número de destinatarios por mensaje (predeterminado: 1), formación de direcciones como con `-N` |
| `-d` | No desconectar tras un mensaje; enviar el siguiente por la misma conexión |
| `-c` | Mostrar un contador en curso que aumenta con cada `DATA` completado |
| `-w n` | Tiempo de espera fijo de n segundos entre mensajes (por sesión) |
| `-v` | Salida detallada para la resolución de problemas |
| `host:port` | Destino de entrega mediante TCP; sin especificar puerto, se usa el puerto SMTP estándar |

La lista completa, incluidas las opciones de TLS, LMTP y temporización, está en la página de manual de `smtp-source(1)`; su equivalente para el lado receptor es `smtp-sink(1)` y se utiliza más adelante en la evaluación.

## Cómo numera -N a los destinatarios

`-N` activa un contador por proceso que se incorpora a la dirección del destinatario. Tres propiedades determinan la configuración de la prueba, y las tres se pueden consultar en el código fuente de `smtp-source.c`:

En primer lugar, el formato exacto de la dirección depende de la versión de Postfix. Postfix 3.5, como el que incluye RHEL 8, antepone el número a toda la dirección (`RCPT TO:<%d%s>`): de `-t test@example.com` resultan `1test@example.com`, `2test@example.com` y así sucesivamente, y el contador empieza en 1. Las versiones actuales de Postfix insertan en cambio el número al final de la parte local y empiezan en 0 (`test0@` hasta `test49999@`); para esta variante, la página de manual recomienda el direccionamiento plus (`-t 'test+@example.com'` produce `test+0@` y los siguientes), para que un sistema de destino con subdireccionamiento asigne todo al mismo buzón. Compruebe el formato antes de la ejecución grande con unos pocos correos contra un `smtp-sink` o en el log del destino; de ello dependen el conjunto esperado y el patrón de búsqueda para la evaluación.

En segundo lugar, el contador abarca todo el proceso y se comparte entre todas las sesiones paralelas. Con `-s 8`, las ocho sesiones asignan conjuntamente los números; cada número aparece exactamente una vez. El orden entre sesiones no es determinista, pero la integridad del conjunto de números está garantizada.

En tercer lugar, el valor inicial no se puede configurar: 1 en Postfix 3.5 y 0 en las versiones actuales. Por tanto, los 50'000 correos llevan los números del 1 al 50'000 o del 0 al 49'999, respectivamente, y el conjunto esperado para la comparación debe ajustarse a ello.

## La ejecución de prueba en una sola invocación

```bash
smtp-source -c -d -N -s 8 -m 50000 -l 5120 \
  -f lasttest@example.com \
  -t test@example.com \
  gateway.example.com:25
```

| Opción | Efecto |
|---|---|
| `-c` | Contador en curso de entregas completadas como indicador de progreso de una línea |
| `-d` | Las conexiones permanecen abiertas para todos los mensajes; sin `-d`, se crea una conexión nueva por mensaje |
| `-N` | Numeración de destinatarios: añade el contador por proceso a la parte local |
| `-s 8` | Ocho sesiones SMTP paralelas |
| `-m 50000` | Número total de mensajes, repartidos entre las sesiones |
| `-l 5120` | Tamaño del mensaje en bytes (sin cabeceras), aquí 5 KB |
| `-f` | Dirección del remitente |
| `-t` | Dirección base del destinatario; `-N` la convierte en `1test@` hasta `50000test@` (Postfix 3.5) o `test0@` hasta `test49999@` (versiones actuales) |
| `gateway.example.com:25` | Host y puerto de destino |

`-d` es decisivo para el perfil de carga: sin esta opción, `smtp-source` desconecta después de cada mensaje y establece una conexión nueva para el siguiente; con `-d`, las ocho conexiones permanecen abiertas y entregan todos los mensajes uno tras otro, como lo haría un remitente masivo.

Se omite deliberadamente el conocido `-v` de las pruebas funcionales: registra cada diálogo SMTP individual desde `HELO` hasta `QUIT` y genera cientos de miles de líneas de log con 50'000 correos, sin aportar valor a la evaluación. `-c` proporciona en su lugar el resumen con el que se puede seguir el progreso de la ejecución en directo. Un `time` antepuesto proporciona la duración total para calcular la tasa.

Requisito para todo el enfoque: el sistema de destino debe aceptar las direcciones generadas. Un `smtp-sink`, un dominio catch-all, un dominio de descarte del proveedor o un gateway que resuelva los destinatarios solo después de aceptarlos cumplen este requisito. Si, en cambio, el destino comprueba cada destinatario contra un directorio, rechazará las direcciones numeradas y solo quedará la variante del asunto.

## Establecer cabeceras propias

Algunas pruebas de carga necesitan una cabecera propia, por ejemplo como marcador con el que el gateway identifica los correos de prueba o aplica una regla. `smtp-source` no dispone de una opción para ello, pero `-F` lee un mensaje completamente preformateado desde un archivo, donde se puede incluir cualquier cabecera deseada. El archivo consta de las líneas de cabecera, una línea vacía y el cuerpo; todas las líneas terminan con `\r\n`:

```bash
{ printf 'X-Lasttest: aktiv\r\n'
  printf 'Subject: Lasttest\r\n'
  printf '\r\n'
  head -c 5120 /dev/zero | tr '\0' 'x'
  printf '\r\n'; } > lasttest.eml
```

```bash
smtp-source -c -d -N -s 8 -m 50000 -F lasttest.eml \
  -f lasttest@example.com \
  -t test@example.com \
  gateway.example.com:25
```

| Opción | Efecto |
|---|---|
| `-F datei` | Envía cabeceras y cuerpo sin modificar desde el archivo; sustituye el contenido de mensaje generado |

Esto tiene dos consecuencias: `-F` sustituye `-l` y `-S`, porque el tamaño y el asunto ahora proceden del archivo (por lo que ambos deben incluirse allí). En cambio, `-N` sigue teniendo efecto: los destinatarios continúan numerándose; la cabecera es idéntica en todos los mensajes, ya que procede del archivo fijo.

## ¿Cuántas sesiones?

La forma más fiable de determinar el número adecuado de sesiones es medirlo con exactamente las mismas opciones que en la ejecución principal prevista: misma fuente de mensajes (el mismo archivo `-F` o el mismo `-l`), mismo remitente y mismo destino. Solo se reduce la cantidad a 2'000 por nivel y se varía `-s`. Una breve ejecución de calibración con un número creciente de sesiones muestra a partir de cuándo las sesiones adicionales ya no aportan nada:

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

Dos detalles de la invocación: aquí se prescinde deliberadamente de `-c` para que no haya salidas de contador en curso entre las líneas de medición; el bucle produce exactamente una línea de resultado por nivel. Además, la parte local vacía de `-t` funciona bien con la numeración en un dominio de descarte: con el contador antepuesto de Postfix 3.5 se generan direcciones de destinatario puramente numéricas (`1@blackhole.example.com`, `2@…`), lo que mantiene clara la evaluación en los logs.

En detalle ocurre lo siguiente: el bucle exterior recorre los números de sesiones del 1 al 32, duplicándolos en cada paso. Antes y después de cada ejecución, `date +%s%N` registra la hora actual como un número grande: segundos Unix seguidos directamente de la parte de nanosegundos. Entre ambos momentos, `smtp-source` envía 2'000 mensajes (el contenido, las cabeceras y el tamaño proceden del archivo `-F`) mediante el número correspondiente de conexiones paralelas que, gracias a `-d`, permanecen abiertas; el bucle espera a que la invocación termine por completo. La línea `echo` convierte la diferencia de tiempo en una tasa: 2'000 correos divididos por el tiempo de ejecución en segundos, aunque el tiempo de ejecución está expresado en nanosegundos. De 2'000 por 10⁹ resulta así la constante `2000000000000`. La aritmética de Bash `$(( ))` calcula con enteros y descarta los decimales, lo que es suficientemente preciso para esta medición.

Tres indicaciones prácticas: `%N` solo proporciona nanosegundos con GNU date (es el caso en RHEL y en la mayoría de sistemas Linux; BusyBox y macOS no lo admiten). La ejecución completa envía 6 × 2'000 = 12'000 correos; estos también necesitan una dirección de destinatario controlada, y la numeración `-N` vuelve a empezar desde el valor inicial en cada invocación. Si una invocación de `smtp-source` se interrumpe con un mensaje de error, la tasa de esa línea carece de significado; primero corrija la causa y después vuelva a medir.

La salida esperada es una línea por nivel. Con valores de ejemplo inventados, pero típicos, tendría este aspecto:

```text
1 Sessions: 11 Mails/s
2 Sessions: 21 Mails/s
4 Sessions: 40 Mails/s
8 Sessions: 71 Mails/s
16 Sessions: 79 Mails/s
32 Sessions: 80 Mails/s
```

La interpretación es la siguiente: mientras la tasa se duplique aproximadamente al aumentar el número de sesiones, las sesiones paralelas cubren el tiempo de espera de las respuestas del destino; el cuello de botella es entonces la latencia del trayecto, no la capacidad. A partir del punto en que la curva se aplana (en el ejemplo, entre 8 y 16 sesiones), el sistema de destino está saturado o el origen ha alcanzado su límite. Tome el valor más pequeño a partir del cual la tasa deja de aumentar de forma apreciable; en el ejemplo, de 8 a 16. Más sesiones solo aumentan entonces la carga mediante paralelismo, no el rendimiento. La tasa medida también permite estimar la duración esperada de la ejecución principal con 50'000 correos: a 71 correos/s, unos 12 minutos.

## Evaluación en el lado receptor

Si hay un receptor de prueba propio en el sistema de destino, `smtp-sink` también se encarga del registro:

```bash
smtp-sink -c -d "mails/%Y%m%d-%H%M%S." 0.0.0.0:2525 200
```

| Opción | Efecto |
|---|---|
| `-c` | Contadores en curso en lugar del diálogo SMTP completo |
| `-d "mails/…"` | En el sink: volcado, no mantenimiento de conexión. Escribe cada mensaje aceptado en un archivo separado (patrón de nombres mediante strftime), incluida una cabecera `X-Rcpt-Args` con la dirección del destinatario |
| `0.0.0.0:2525` | Escucha en todas las interfaces en el puerto 2525 |
| `200` | Backlog: longitud máxima de la cola de conexiones pendientes según listen(2) |

Después de la ejecución, extraiga los números recibidos y compárelos con el conjunto esperado. Como los números no llevan ceros a la izquierda, ambas listas se rellenan hasta una longitud fija antes de compararlas, para que la ordenación alfabética de `comm` coincida con la numérica. El patrón de búsqueda corresponde al formato de dirección de Postfix 3.5 (número antes de la dirección); para las versiones actuales, use respectivamente `test[0-9]+@` y `seq 0 49999`:

```bash
grep -rhoE '[0-9]+test@example\.com' mails/ | \
  grep -oE '^[0-9]+' | sort -u | \
  awk '{printf "%08d\n", $1}' | sort > empfangen.txt
```

```bash
seq 1 50000 | awk '{printf "%08d\n", $1}' | \
  comm -23 - empfangen.txt
```

`comm -23` muestra exactamente los números que están en el conjunto esperado pero no en la lista de recepción: los correos faltantes. Una salida vacía significa que la entrega fue completa. Si aparecen números duplicados (reconocibles por la diferencia entre `sort` y `sort -u`), algún sistema duplicó el mensaje en tránsito, lo que también es un hallazgo.

Si el destino es un sistema similar a producción en lugar de un smtp-sink, su registro asume el papel de los archivos de volcado. En un servidor Exchange, por ejemplo, `Get-MessageTrackingLog -Recipients` o un filtro por dirección de destinatario proporciona los números llegados; en un sistema Postfix, un `grep` sobre `to=` y la dirección base mediante el maillog. Esta es precisamente la ventaja del número en la dirección: el destinatario aparece en cada seguimiento de mensajes, mientras que el asunto puede faltar según el sistema o requerir activación previa.

## Cuando el número debe estar en el asunto

Algunas evaluaciones dependen del asunto, por ejemplo cuando el sistema de destino reescribe las direcciones de destinatario o los logs muestran el destinatario solo enmascarado. En ese caso queda la variante con bucle: una invocación de `smtp-source` por correo con `-m 1` y un asunto que la shell incrementa, distribuida entre varios workers paralelos con rangos de números consecutivos.

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

El coste es un establecimiento de conexión completo por correo: handshake TCP, banner, `HELO`, envío, `QUIT`. Por tanto, esta ejecución no mide el rendimiento máximo del sistema de destino, sino un caso deliberadamente intensivo en conexiones. Determine el número de workers de forma análoga a la ejecución de calibración anterior, pero con el bucle de workers en lugar de `-s`. Los ceros a la izquierda en el asunto evitan el reformateo necesario para la comparación en la variante `-N`.

## Reglas para pruebas contra otros sistemas

En cuanto la prueba sale del propio sistema, se aplican tres condiciones. Primera: el operador del sistema de destino está informado y ha aceptado la ventana temporal; 50'000 correos parecen un ataque o una oleada de spam para cualquier monitorización. Segunda: la dirección del destinatario termina en un destino controlado, ya sea un buzón de prueba dedicado, una regla de descarte en el destino o un dominio de descarte proporcionado para ello por el proveedor; las direcciones de producción no tienen cabida en una prueba de carga. Tercera: antes de empezar se fija un criterio de interrupción, por ejemplo una cola creciente en el destino o una tasa de errores superior a un umbral, y alguien observa estos valores durante la ejecución.

Con estos tres puntos y la numeración, la ejecución proporciona al final no solo una cifra de rendimiento, sino una afirmación verificable: cuáles de los 50'000 correos llegaron, cuáles faltan y dónde se vieron por última vez en el trayecto.

## Fuentes

1.  [Postfix: smtp-source(1)](https://www.postfix.org/smtp-source.1.html): página de manual del generador de carga; describe el comportamiento de `-N` en la versión actual (contador en la parte local, direccionamiento plus).

2.  [Código fuente de Postfix 3.5.8: smtp-source.c](https://github.com/vdukhovni/postfix/blob/v3.5.8/postfix/src/smtpstone/smtp-source.c): documenta para la versión de RHEL 8 el antepuesto del número (`RCPT TO:<%d%s>`) con valor inicial 1; en el [estado actual](https://github.com/vdukhovni/postfix/blob/master/postfix/src/smtpstone/smtp-source.c) el número se añade en cambio a la parte local, a partir de 0.

3.  [Postfix: smtp-sink(1)](https://www.postfix.org/smtp-sink.1.html): página de manual del receptor de prueba con las opciones de volcado y las cabeceras X registradas.

4.  [GNU Coreutils: comm](https://www.gnu.org/software/coreutils/manual/html_node/comm-invocation.html): comparación de conjuntos de dos listas ordenadas, aquí para cotejar los números esperados y recibidos.
