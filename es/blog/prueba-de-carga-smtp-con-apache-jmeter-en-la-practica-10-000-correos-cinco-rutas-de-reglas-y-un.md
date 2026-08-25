---
title: "Prueba de carga SMTP con Apache JMeter en la práctica: 10.000 correos, cinco rutas de reglas y un informe HTML"
navTitle: "Prueba de carga con JMeter"
description: "Una prueba de carga realizada de principio a fin: plan de pruebas con mezcla de mensajes a lo largo de las rutas de Ruleset de una gateway de cifrado, configuración portátil sin instalación, 10.000 correos en ráfaga y evaluación mediante el informe HTML de JMeter, incluidos los obstáculos que realmente surgieron."
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
url: https://rafaelpfister.ch/es/blog/prueba-de-carga-smtp-con-apache-jmeter-en-la-practica-10-000-correos-cinco-rutas-de-reglas-y-un
translationSourceHash: a41d58b7a4a717db179b3fec1ef8fac7961ff3ee12069f65627ddb48338aef0a
translationModel: gpt-5.6-terra
translatedAt: 2026-08-25T04:09:19.070Z
translationReview: required
---

# Prueba de carga SMTP con Apache JMeter en la práctica: 10.000 correos, cinco rutas de reglas y un informe HTML

El [artículo de resumen sobre pruebas de carga de correo](/blog/mail-lasttest-tools-linux-windows-vergleich) comparó las herramientas y esbozó el plan de pruebas. Este artículo pone la prueba en práctica: una prueba de carga completa con JMeter con 10.000 correos, una mezcla de mensajes a lo largo de rutas de reglas reales de la gateway y el informe HTML como evaluación. Todos los valores mostrados proceden de la ejecución real, incluidos los errores que surgieron durante el proceso.

El escenario está inspirado en un proyecto real: una gateway de cifrado de correo electrónico basada en Apache James (Totemomail) funciona como un bucle de smarthost detrás de Exchange Online y decide para cada mensaje el cifrado, la firma y el enrutamiento especial. El Ruleset de Mailet dispone para ello de varias rutas: disparadores en el asunto como (sec), (sign) y (unsec), palabras clave como VERTRAULICH para el enrutamiento a una gateway sectorial y la ruta estándar con comprobación de certificados y alternativa en texto sin cifrar. Una prueba de carga que solo entregue un tipo de mensaje mediría siempre la misma ruta a través de este conjunto de reglas; por ello, el plan de pruebas representa cinco clases cuya proporción corresponde al tráfico esperado.

## La configuración: sin necesidad de instalar nada

La prueba se ejecutó en una máquina Windows sin Java ni JMeter. Ambos pueden utilizarse de forma portátil, lo cual es decisivo en puestos de trabajo de administración con permisos de instalación restringidos: JRE Temurin como ZIP desde Adoptium, JMeter como ZIP desde apache.org, descomprimir ambos, establecer `JAVA_HOME` en el directorio de la JRE y listo.

```bash
export JAVA_HOME="$PWD/jdk-21-jre"
export PATH="$JAVA_HOME/bin:$PATH"
./apache-jmeter-5.6.3/bin/jmeter -n -t gateway-lasttest.jmx -l lauf.jtl -e -o report
```

Como sumidero se utilizó una caja negra SMTP local basada en aiosmtpd, poco más de 40 líneas de Python: acepta cada mensaje con `250`, descarta el contenido, lleva el recuento y asigna cada correo a una clase según la línea de asunto. Este recuento independiente en el lado receptor es el control de la prueba; si las cifras del generador y del sumidero no coinciden, algo se ha perdido por el camino.

```python
from aiosmtpd.controller import Controller

class SinkHandler:
    def __init__(self):
        self.count = 0

    async def handle_DATA(self, server, session, envelope):
        self.count += 1
        # Extraer el asunto de la cabecera para las estadísticas de clase,
        # El contenido se descarta
        return "250 Message accepted for delivery"

controller = Controller(SinkHandler(), hostname="127.0.0.1", port=2525)
controller.start()
```

Importante para la interpretación: el generador y el sumidero se ejecutaron en la misma máquina, sin TLS ni red entre ambos. Por tanto, las cifras medidas no dicen nada sobre una gateway, sino que son la autoprueba del generador del artículo de resumen: la demostración de que la configuración de carga puede generar la tasa objetivo y el límite superior con el que se compararán posteriormente las mediciones frente al sistema de prueba real.

## El plan de pruebas: cinco clases de mensajes, una proporción

El núcleo del plan es un Thread Group con 20 hilos, 10 segundos de ramp-up y 500 bucles, es decir, 10.000 iteraciones. Debajo hay cinco Throughput Controller en modo "Percent Executions", cada uno con exactamente un SMTP Sampler:

| Clase (etiqueta del Sampler) | Proporción | Ruta de reglas en la gateway |
|---|---|---|
| 01 Estándar sin disparador | 60 % | Comprobación de AutoGenerated, comprobación de certificados, alternativa en texto sin cifrar |
| 02 Disparador (sec) | 15 % | Sobre TRE para destinatarios sin certificado |
| 03 Disparador (sign) | 10 % | Certificate Exchange: firmar, adjuntar la clave |
| 04 Palabra clave VERTRAULICH | 10 % | Enrutamiento especial a la gateway sectorial |
| 05 Disparador (unsec) | 5 % | Texto sin cifrar forzado |

La división en cinco Sampler separados en lugar de un Sampler con asunto variable tiene un motivo práctico: el informe HTML agrupa todas las métricas por la etiqueta del Sampler. Cinco etiquetas generan cinco filas en las estadísticas con percentiles propios por clase; un único Sampler con asunto alimentado por CSV produciría una sola fila agregada y la diferencia entre las rutas de reglas sería invisible en la evaluación.

Cada Sampler rellena los campos habituales: host y puerto de destino como variables definidas por el usuario (`${zielhost}`, `${zielport}`), para que el mismo plan pueda ejecutarse sin modificaciones contra el sumidero, el entorno de pruebas o PreProd, además de remitente, destinatario, asunto con un marcador claro —en este caso la palabra LOADTEST en el asunto— y un cuerpo de texto de aproximadamente 1 a 2 KB. La opción "Include timestamp in subject" añade la hora de entrega en milisegundos; en una ejecución posterior contra un sistema real de varias etapas, esto permite calcular la latencia de extremo a extremo por mensaje junto con las horas de recepción del sumidero.

Un obstáculo de esta ejecución que se puede generalizar: el primer intento falló con 10.000 errores en 10 segundos, todos con `java.lang.ClassCastException ... NullProperty cannot be cast to ... CollectionProperty` en lugar de una respuesta SMTP. La causa fue un archivo JMX creado manualmente en el que faltaba la lista de cabeceras del Sampler; el Sampler requiere obligatoriamente la propiedad, aunque esté vacía. La lección no es tanto la propiedad concreta como el patrón: crear los planes de prueba en la GUI y guardarlos, no escribir XML a mano, y antes de cada ráfaga realizar una ejecución mínima y comprobar en el sumidero que el asunto y el contenido realmente llegan. Un contador de errores del 100 % con 0 ms de tiempo de respuesta casi siempre significa que el error ocurre antes de la red, es decir, que la prueba nunca llegó al sistema objetivo.

## La ejecución

La medición propiamente dicha se realiza en modo CLI; la GUI es solo el editor. Una única llamada genera la ejecución, los datos brutos y el informe:

```bash
jmeter -n -t gateway-lasttest.jmx -l lauf-10k.jtl -e -o report-10k
```

El summariser en la consola muestra el progreso en directo y el resultado final de la ejecución:

```text
summary =  10000 in 00:00:13 =  781.8/s Avg: 6 Min: 1 Max: 27 Err: 0 (0.00%)
```

10.000 mensajes en 12,8 segundos, 782 mensajes por segundo de media, sin errores. El sumidero confirmó independientemente exactamente 10.000 correos aceptados con la mezcla 6000 / 1500 / 1000 / 1000 / 500; la proporción de los Throughput Controller coincidió, por tanto, mensaje a mensaje.

## El informe HTML

El argumento a favor de JMeter frente a generadores más ligeros como smtp-source es la evaluación, y el informe Dashboard la proporciona sin trabajo adicional:

![Dashboard de JMeter de la ejecución: APDEX 1.000 para las cinco clases, Requests Summary con 100 por ciento PASS, tabla estadística con percentiles por clase de mensaje](../images/jmeter-report-dashboard.png)

La tabla estadística es la parte más importante del informe. Para cada etiqueta del Sampler, es decir, para cada clase de mensaje, muestra el número, la tasa de errores, la media, la mediana, los percentiles 90, 95 y 99, el máximo y el rendimiento. En la ejecución concreta: mediana de 7 ms, p95 de 11 ms, p99 de 12 ms, máximo de 27 ms, y prácticamente idénticos para las cinco clases. Con un sumidero local que trata cada mensaje de la misma forma, esta es exactamente la imagen esperada y, al mismo tiempo, el valor de referencia: si más adelante se ejecuta el mismo plan contra la gateway real y la clase (sec) muestra de repente varias veces la mediana estándar, ese será el trabajo adicional de la ruta de cifrado, aislado limpiamente por rama de reglas.

El bloque APDEX situado encima condensa lo mismo en una cifra por clase —aquí 1.000 en todas, porque todas las respuestas quedaron muy por debajo del umbral de tolerancia de 500 ms—; los umbrales se pueden adaptar a objetivos de servicio propios en las propiedades del informe. El bloque Errors permanece vacío en esta ejecución, pero en pruebas contra sistemas reales es el primer lugar al que acudir: agrupa los errores por texto de respuesta, de modo que una limitación `421` del sistema objetivo se distingue inmediatamente de las interrupciones de conexión.

También hay un obstáculo aquí, y afecta a cualquier ráfaga corta: los gráficos de series temporales del informe utilizan por defecto una granularidad de un minuto. Una ejecución de 13 segundos queda así reducida a un único punto de datos, y las curvas bajo "Charts" parecen un error de medición. El informe puede regenerarse desde el archivo JTL existente, sin una nueva ejecución y con una resolución más fina:

```bash
jmeter -g lauf-10k.jtl -o report-fein -Jjmeter.reportgenerator.overall_granularity=1000
```

Con granularidad por segundos, el único punto se convierte en la evolución real de la carga:

![Hits per Second con granularidad de 1 segundo: aumento durante los 10 segundos de ramp-up hasta una meseta de unos 840 mensajes por segundo, seguido de una caída pronunciada al final de la prueba](../images/jmeter-report-hits-per-second.png)

La curva muestra el ramp-up de 10 segundos, una meseta de unos 840 mensajes por segundo y la caída al final, cuando los primeros hilos completan sus 500 bucles. Para la interpretación cuenta la meseta, no la media de toda la ejecución: la media de 782/s incluye el ramp-up y la fase final, por lo que subestima la tasa sostenida alcanzada.

## Lo que esta ejecución demuestra y lo que no

Esta ejecución demuestra lo siguiente: el plan de pruebas es funcionalmente correcto —ejecución mínima con control del contenido en el sumidero—, la proporción coincide exactamente y el generador alcanza en esta máquina al menos 840 mensajes por segundo sin TLS. Quien quiera probar con ello una gateway diseñada para 100 correos por segundo dispone de un margen de un factor de ocho y puede atribuir con confianza los cuellos de botella al sistema objetivo.

No demuestra nada más, y esta delimitación debe formar parte de todo informe de pruebas: ninguna afirmación sobre los costes del handshake TLS —la ruta real utiliza STARTTLS—, ninguna sobre el comportamiento de la cola de la gateway, ninguna sobre el tiempo de procesamiento de las rutas de reglas. Para ello, el mismo plan, con las variables `zielhost`/`zielport` dirigidas al entorno de pruebas de la gateway, muestra el resultado; la evaluación se realiza entonces de forma idéntica, complementada con los logs de la gateway y la observación de la cola del artículo de resumen. Precisamente esta reutilización —un plan para el sumidero, el entorno de pruebas y PreProd con una evaluación idéntica— es la verdadera razón para asumir una vez el esfuerzo de crear un plan de JMeter limpio.

## Fuentes

1.  [Apache JMeter User's Manual: Component Reference, SMTP Sampler](https://jmeter.apache.org/usermanual/component_reference.html): Referencia de los campos del Sampler, incluida la cabecera, la opción de marca de tiempo y el envío EML.

2.  [Apache JMeter: Generating Dashboard Report](https://jmeter.apache.org/usermanual/generating-dashboard.html): Generación del informe HTML a partir de la ejecución o posteriormente desde el JTL, incluidas las propiedades de granularidad y APDEX.

3.  [Apache JMeter: Test Plan, Logic Controllers](https://jmeter.apache.org/usermanual/component_reference.html#Throughput_Controller): Funcionamiento del Throughput Controller en modo Percent Executions para la mezcla de mensajes.

4.  [aiosmtpd, documentación](https://aiosmtpd.aio-libs.org/): El servidor SMTP basado en asyncio con el que se crea el sumidero en pocas líneas de Python.

5.  [Eclipse Temurin, Adoptium](https://adoptium.net/temurin/releases/): Archivos JRE portátiles para ejecutar JMeter sin instalar Java.
