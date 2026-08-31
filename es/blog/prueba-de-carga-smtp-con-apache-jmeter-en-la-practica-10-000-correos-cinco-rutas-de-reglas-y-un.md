---
title: "Prueba de carga SMTP con Apache JMeter en la práctica: 10.000 correos, cinco rutas de reglas y un informe HTML"
navTitle: "Prueba de carga con JMeter"
description: "Una prueba de carga realizada de principio a fin: plan de prueba con una mezcla de mensajes a lo largo de las rutas de ruleset de una pasarela de cifrado, configuración portátil sin instalación, 10.000 correos en ráfaga y evaluación mediante el informe HTML de JMeter, incluidos los problemas que realmente surgieron."
date: "2026-08-24"
kategorie: "SMTP y flujo de correo"
timeToRead: "11 min de lectura"
themen:
  - smtp-mailflow
  - testing
  - totemomail
produkte:
  - "uebergreifend"
  - "totemomail"
  - "apache-james"
protokolle:
  - "testing"
  - "smtp"
  - "troubleshooting"
related:
  - mail-lasttest-tools-linux-windows-vergleich
image: "../images/jmeter-report-dashboard.png"
slug: "prueba-de-carga-smtp-con-apache-jmeter-en-la-practica-10-000-correos-cinco-rutas-de-reglas-y-un"
translationId: "article-fc3f25272e051f92"
aiPrompt: |
  Du bist mein Assistent für Mail-Lasttests mit Apache JMeter. Hilf mir Schritt für Schritt, einen SMTP-Lasttest aufzubauen: portables Setup (JRE + JMeter ohne Installation), lokale SMTP-Senke mit aiosmtpd, Testplan mit Thread Group, Throughput Controllern für den Nachrichtenmix und SMTP Samplern, Lauf im CLI-Modus mit HTML-Report und Auswertung der Perzentile pro Nachrichtenklasse. Frage zuerst nach Zielsystem, Nachrichtenklassen und gewünschtem Volumen.
translationOf: jmeter-smtp-lasttest-html-report
translationSourceHash: 26c09e391d2252b6203dceb5dc45edd23beba797820fe0b95273bf48a9afc181
translationModel: gpt-5.6-terra
translatedAt: 2026-08-30T09:23:45.131Z
translationReview: required
url: https://rafaelpfister.ch/es/blog/prueba-de-carga-smtp-con-apache-jmeter-en-la-practica-10-000-correos-cinco-rutas-de-reglas-y-un
---

# Prueba de carga SMTP con Apache JMeter en la práctica: 10.000 correos, cinco rutas de reglas y un informe HTML

El [artículo de resumen sobre las pruebas de carga de correo](/blog/mail-lasttest-tools-linux-windows-vergleich) comparó las herramientas y esbozó el plan de prueba. A continuación se presenta la ejecución práctica: una prueba de carga completa con JMeter con 10.000 correos, una mezcla de mensajes a lo largo de rutas reales de reglas de la pasarela y el informe HTML como evaluación. Todos los valores mostrados proceden de la ejecución real, incluidos los errores que se produjeron durante el proceso.

El escenario reproduce un proyecto real: una pasarela de cifrado de correo electrónico basada en Apache James (Totemomail) está conectada como bucle de smarthost detrás de Exchange Online y decide para cada mensaje sobre el cifrado, la firma y el enrutamiento especial. El ruleset de Mailet cuenta para ello con varias rutas: desencadenantes en el asunto como (sec), (sign) y (unsec), palabras clave como VERTRAULICH para el enrutamiento a una pasarela sectorial y la ruta estándar con comprobación de certificados y alternativa en texto sin cifrar. Una prueba de carga que solo entregue un tipo de mensaje mediría siempre la misma ruta a través de este conjunto de reglas; por ello, el plan de prueba representa cinco clases cuya proporción de mezcla corresponde al tráfico esperado.

Importante para contextualizar: este plan de prueba genera el patrón de carga de muchos remitentes independientes, ya que JMeter abre una conexión propia para cada mensaje (los antecedentes se explican en las limitaciones al final). Para demostrar que un conjunto de reglas funciona correcta y suficientemente rápido bajo tráfico mixto paralelo, este es el patrón adecuado. Sin embargo, el plan no reproduce la carga máxima de un único remitente masivo con sesiones abiertas; para este patrón de carga, `smtp-source` del [artículo de resumen](/blog/mail-lasttest-tools-linux-windows-vergleich) es la herramienta adecuada.

## Las opciones más importantes de jmeter

Como orientación previa, las opciones de línea de comandos que aparecen en este artículo, traducidas libremente de la documentación:

<details class="options-details">
<summary>Resumen de opciones</summary>

| Opción | Significado |
|---|---|
| `-n` | Modo CLI (sin GUI): ejecuta el plan de prueba sin interfaz gráfica |
| `-t datei` | Ruta al archivo JMX que contiene el plan de prueba |
| `-l datei` | Ruta al archivo de resultados JTL en el que se escriben los valores medidos |
| `-e` | Genera directamente el informe del dashboard HTML tras la ejecución |
| `-o verzeichnis` | Directorio de destino para el informe; debe estar vacío o no existir todavía |
| `-g datei` | Genera posteriormente el informe a partir de un archivo JTL existente, sin una nueva ejecución |
| `-J<property>=<wert>` | Establece una propiedad de JMeter solo para esta invocación |

</details>

La lista completa se muestra con `jmeter -?`; las opciones se describen en el capítulo sobre el funcionamiento sin GUI del [JMeter User's Manual](https://jmeter.apache.org/usermanual/get-started.html).

## La configuración: sin necesidad de instalar nada

La prueba se ejecutó en una máquina Windows sin Java ni JMeter. Ambos pueden utilizarse de forma portátil, lo que resulta decisivo en puestos de trabajo de administración con permisos de instalación restringidos: Temurin-JRE como ZIP de Adoptium, JMeter como ZIP de apache.org, descomprimir ambos, establecer `JAVA_HOME` en el directorio de JRE y listo.

```bash
export JAVA_HOME="$PWD/jdk-21-jre"
export PATH="$JAVA_HOME/bin:$PATH"
./apache-jmeter-5.6.3/bin/jmeter -n -t gateway-lasttest.jmx -l lauf.jtl -e -o report
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `export JAVA_HOME=…` | Apunta al directorio JRE descomprimido; JMeter encuentra así el entorno de ejecución Java sin instalación |
| `export PATH=…` | Coloca los binarios de JRE al principio de la ruta de búsqueda |
| `-n` | Modo CLI sin interfaz gráfica |
| `-t gateway-lasttest.jmx` | El plan de prueba que se ejecutará |
| `-l lauf.jtl` | Archivo de resultados con los valores medidos de cada sampler |
| `-e` | Genera el informe HTML directamente después de la ejecución |
| `-o report` | Directorio de destino para el informe |

</details>

Como sumidero se utilizó una caja negra SMTP local basada en aiosmtpd, unas 40 líneas de Python: acepta cada mensaje con `250`, descarta el contenido, cuenta y asigna cada correo a una clase según la línea de asunto. Este conteo independiente en el lado receptor es el control de la prueba; si las cifras del generador y del sumidero no coinciden, algo se ha perdido por el camino.

```python
from aiosmtpd.controller import Controller

class SinkHandler:
    def __init__(self):
        self.count = 0

    async def handle_DATA(self, server, session, envelope):
        self.count += 1
        # Extraer el asunto de la cabecera para las estadísticas por clase,
        # Se descarta el contenido
        return "250 Message accepted for delivery"

controller = Controller(SinkHandler(), hostname="127.0.0.1", port=2525)
controller.start()
```

Importante para contextualizar: el generador y el sumidero se ejecutaron en la misma máquina, sin TLS ni red entre ambos. Por tanto, los valores medidos no indican nada sobre una pasarela, sino que constituyen la autoprueba del generador del artículo de resumen: la prueba de que la configuración de carga puede generar siquiera la tasa objetivo, y el límite superior con el que se compararán más adelante las mediciones frente al sistema de prueba real.

## El plan de prueba: cinco clases de mensajes, una proporción de mezcla

El núcleo del plan es un Thread Group con 20 hilos, 10 segundos de ramp-up y 500 bucles, es decir, 10.000 iteraciones. Debajo hay cinco Throughput Controller en modo "Percent Executions", cada uno con exactamente un SMTP Sampler:

| Clase (etiqueta del sampler) | Proporción | Ruta de reglas en la pasarela |
|---|---|---|
| 01 Estándar sin desencadenante | 60 % | Comprobación AutoGenerated, comprobación de certificados, alternativa en texto sin cifrar |
| 02 Desencadenante (sec) | 15 % | Sobre TRE para destinatarios sin certificado |
| 03 Desencadenante (sign) | 10 % | Certificate Exchange: firmar, enviar la clave |
| 04 Palabra clave VERTRAULICH | 10 % | Enrutamiento especial a la pasarela sectorial |
| 05 Desencadenante (unsec) | 5 % | Texto sin cifrar forzado |

La división en cinco samplers independientes en lugar de un solo sampler con asunto variable tiene una razón práctica: el informe HTML agrupa todas las métricas por la etiqueta del sampler. Cinco etiquetas generan cinco filas en las estadísticas con sus propios percentiles por clase; un único sampler con un asunto alimentado por CSV daría una sola fila agrupada, y la diferencia entre las rutas de reglas sería invisible en la evaluación.

Cada sampler rellena los campos habituales: host y puerto de destino como variables personalizadas (`${zielhost}`, `${zielport}`), de modo que el mismo plan pueda ejecutarse sin cambios contra el sumidero, el entorno de prueba o PreProd, además de remitente, destinatario, asunto con un marcador claro (aquí la palabra LOADTEST en el asunto) y un cuerpo de texto de aproximadamente 1 a 2 KB. La opción "Include timestamp in subject" añade al asunto el momento de entrega en milisegundos; en una ejecución posterior contra un sistema real de varias etapas, junto con los momentos de recepción del sumidero se podrá calcular la latencia de extremo a extremo por mensaje.

Un error de esta ejecución que se puede generalizar: el primer intento falló con 10.000 errores en 10 segundos, todos con `java.lang.ClassCastException ... NullProperty cannot be cast to ... CollectionProperty` en lugar de una respuesta SMTP. La causa fue un archivo JMX creado manualmente en el que faltaba la lista de encabezados del sampler; el sampler espera obligatoriamente esa propiedad, aunque esté vacía. La lección es menos la propiedad concreta que el patrón: crear los planes de prueba en la GUI y guardarlos, no escribir XML manualmente, y antes de cada ráfaga hacer una ejecución mínima y comprobar en el sumidero que el asunto y el contenido lleguen realmente. Un contador de errores del 100 por cien con un tiempo de respuesta de 0 ms casi siempre significa que el error se produce antes de la red, es decir, que la prueba nunca llegó al sistema de destino.

## La ejecución

La medición se ejecuta en modo CLI; la GUI solo sirve de editor. Una sola invocación genera la ejecución, los datos sin procesar y el informe:

```bash
jmeter -n -t gateway-lasttest.jmx -l lauf-10k.jtl -e -o report-10k
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-n` | Modo CLI: el plan de prueba se ejecuta sin GUI; solo el summariser escribe en la consola |
| `-t gateway-lasttest.jmx` | El plan de prueba creado en la GUI |
| `-l lauf-10k.jtl` | Datos sin procesar de la ejecución; el informe puede generarse de nuevo posteriormente a partir de este archivo |
| `-e` | Genera el informe inmediatamente después de la ejecución |
| `-o report-10k` | Directorio de destino para el informe HTML |

</details>

El summariser en la consola muestra el progreso en directo, el resultado final de la ejecución:

```text
summary =  10000 in 00:00:13 =  781.8/s Avg: 6 Min: 1 Max: 27 Err: 0 (0.00%)
```

10.000 mensajes en 12,8 segundos, 782 mensajes por segundo de media, sin errores. El sumidero confirmó independientemente exactamente 10.000 correos aceptados con la mezcla 6000 / 1500 / 1000 / 1000 / 500, por lo que la proporción de mezcla de los Throughput Controller se cumplió exactamente hasta el último mensaje.

## El informe HTML

El argumento a favor de JMeter frente a generadores más ligeros como smtp-source es la evaluación, y el informe del dashboard la ofrece sin trabajo adicional:

![Dashboard de JMeter de la ejecución: APDEX 1.000 para las cinco clases, Requests Summary con 100 por ciento PASS, tabla de estadísticas con percentiles por clase de mensajes](../images/jmeter-report-dashboard.png)

La tabla de estadísticas es la parte más importante del informe. Para cada etiqueta de sampler, es decir, para cada clase de mensaje, muestra cantidad, tasa de errores, promedio, mediana, percentiles 90, 95 y 99, máximo y rendimiento. En la ejecución concreta: mediana de 7 ms, p95 de 11 ms, p99 de 12 ms, máximo de 27 ms, prácticamente idénticos en las cinco clases. Con un sumidero local que trata todos los mensajes por igual, esta es exactamente la imagen esperada y al mismo tiempo el valor de referencia: si más adelante el mismo plan se ejecuta contra la pasarela real y la clase (sec) muestra de repente varias veces la mediana estándar, ese será el trabajo adicional de la ruta de cifrado, aislado limpiamente por rama de reglas.

El bloque APDEX de arriba condensa lo mismo en una cifra por clase (aquí, 1.000 en todas, porque todas las respuestas estuvieron muy por debajo del umbral de tolerancia de 500 ms); los umbrales pueden adaptarse a objetivos de servicio propios en las propiedades del informe. El bloque Errors permanece vacío en esta ejecución, pero en las pruebas contra sistemas reales es el primer lugar que consultar: agrupa los errores por texto de respuesta, de modo que una limitación `421` del sistema de destino se distingue de inmediato de las interrupciones de conexión.

También aquí hay un error de evaluación típico, que afecta a cualquier ráfaga corta: los gráficos de series temporales del informe funcionan por defecto con una granularidad de un minuto. Una ejecución de 13 segundos se reduce así a un único punto de datos, y las curvas bajo "Charts" parecen un error de medición. El informe puede regenerarse a partir del archivo JTL existente sin una nueva ejecución y con una resolución más fina:

```bash
jmeter -g lauf-10k.jtl -o report-fein -Jjmeter.reportgenerator.overall_granularity=1000
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-g lauf-10k.jtl` | Genera el informe a partir del archivo JTL existente, sin volver a ejecutar la prueba |
| `-o report-fein` | Nuevo directorio de destino; el directorio de informe existente no se modifica |
| `-Jjmeter.reportgenerator.overall_granularity=1000` | Establece la granularidad de los gráficos para esta invocación en 1.000 ms en lugar del minuto predeterminado |

</details>

Con granularidad de segundos, el punto único se convierte en el perfil real de carga:

![Hits per Second con granularidad de 1 segundo: aumento durante los 10 segundos de ramp-up hasta una meseta de unas 840 mensajes por segundo y luego una caída pronunciada al final de la prueba](../images/jmeter-report-hits-per-second.png)

La curva muestra los 10 segundos de ramp-up, una meseta de unas 840 mensajes por segundo y la caída final, cuando los primeros hilos completan sus 500 bucles. Para la interpretación cuenta la meseta, no el promedio de toda la ejecución: la media de 782/s incluye el ramp-up y el descenso, y subestima la tasa sostenida alcanzada.

## Lo que esta ejecución demuestra y lo que no

Tras esta ejecución queda demostrado lo siguiente: el plan de prueba es funcionalmente correcto (ejecución mínima con control de contenido en el sumidero), la proporción de mezcla es exacta y el generador alcanza en esta máquina al menos 840 mensajes por segundo sin TLS. Quien quiera probar con ello una pasarela diseñada para 100 correos por segundo dispone de un margen de un factor ocho y puede atribuir con tranquilidad los cuellos de botella al sistema de destino.

No demuestra nada más, y esta delimitación debe formar parte de todo informe de prueba: no dice nada sobre los costes del handshake TLS (la ruta real utiliza STARTTLS), ni sobre el comportamiento de la cola de la pasarela, ni sobre el tiempo de procesamiento de las rutas de reglas. Para ello, el mismo plan, con las variables `zielhost`/`zielport` modificadas, apunta al entorno de prueba de la pasarela; la evaluación se ejecuta entonces de forma idéntica, complementada con los registros de la pasarela y la observación de la cola del artículo de resumen. Precisamente esta reutilización —un plan para el sumidero, el entorno de prueba y PreProd con una evaluación idéntica— es la verdadera razón para invertir una vez el esfuerzo en un plan JMeter bien preparado.

Una limitación de la propia herramienta también forma parte de esta delimitación: JMeter no puede mantener abiertas las sesiones SMTP. El SMTP Sampler abre una nueva conexión para cada mensaje, recorre EHLO, en su caso STARTTLS y AUTH, y la termina tras exactamente una transacción con QUIT. Por tanto, los 840 mensajes por segundo incluyen un establecimiento completo de conexión por mensaje. Un remitente masivo que envía cientos de mensajes mediante una sesión abierta genera en la pasarela un patrón de carga diferente, con menos conexiones y más transacciones por conexión, y los límites de conexión se aplican antes con la carga de JMeter. La razón es la arquitectura del framework: JMeter mide cada sampler como una unidad independiente y autocontenida, para que los temporizadores, las aserciones y los percentiles funcionen de igual manera para todos los protocolos compatibles, y el SMTP Sampler es una capa sobre la biblioteca JavaMail, que como API de cliente conecta y desconecta para cada operación de envío. No existe reutilización de conexión para SMTP como el Keep-Alive del sampler HTTP. Para el patrón de carga de un remitente masivo con sesión abierta, `smtp-source` o un script propio son más adecuados; la comparación de herramientas del artículo de resumen lo sitúa en contexto.

## Fuentes

1.  [Apache JMeter User's Manual: Component Reference, SMTP Sampler](https://jmeter.apache.org/usermanual/component_reference.html): referencia de los campos del sampler, incluidos encabezados, opción de marca de tiempo y envío EML.

2.  [Apache JMeter: Generating Dashboard Report](https://jmeter.apache.org/usermanual/generating-dashboard.html): generación del informe HTML a partir de la ejecución o posteriormente desde el JTL, incluidas las propiedades de granularidad y APDEX.

3.  [Apache JMeter: Test Plan, Logic Controllers](https://jmeter.apache.org/usermanual/component_reference.html#Throughput_Controller): funcionamiento del Throughput Controller en modo Percent Executions para la mezcla de mensajes.

4.  [aiosmtpd, documentación](https://aiosmtpd.aio-libs.org/): el servidor SMTP basado en asyncio con el que se crea el sumidero en pocas líneas de Python.

5.  [Eclipse Temurin, Adoptium](https://adoptium.net/temurin/releases/): archivos JRE portátiles para ejecutar JMeter sin instalar Java.

6.  [Apache JMeter: Getting Started, Non-GUI Mode](https://jmeter.apache.org/usermanual/get-started.html): resumen de las opciones de línea de comandos para el funcionamiento CLI, incluidas `-n`, `-t`, `-l`, `-e`, `-o`, `-g` y `-J`.
