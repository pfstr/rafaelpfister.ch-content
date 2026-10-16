---
title: "Pruebas funcionales y de carga: comprobar la infraestructura en condiciones controladas"
blatt: "testing"
description: "Metodología para pruebas funcionales y de carga en entornos de mensajería e infraestructura: métricas, proceso de prueba con línea base, ráfaga y soak, generadores de carga y sumideros, marcado de las ejecuciones de prueba y evaluación mediante percentiles en lugar de medias."
fakten:
  - label: Proceso básico
    wert: Prueba funcional → línea base → ráfaga → soak
    href: https://jmeter.apache.org/usermanual/best-practices.html
  - label: Cuatro métricas
    wert: Tasa de aceptación · latencia de sesión · latencia de extremo a extremo · comportamiento de la cola
  - label: Evaluación
    wert: Percentiles p50 · p95 · p99, no medias
    href: https://sre.google/sre-book/monitoring-distributed-systems/
  - label: Generadores de carga
    wert: smtp-source · Postal · JMeter · scripts propios
    href: https://www.postfix.org/smtp-source.1.html
  - label: Sumideros
    wert: smtp-sink · bhm · aiosmtpd · transporte discard
    href: https://www.postfix.org/smtp-sink.1.html
  - label: Prueba de control
    wert: recuento independiente en el lado receptor
  - label: Marcado
    wert: cabecera o marcador de asunto único por ejecución de prueba
  - label: Autoprueba del generador
    wert: la ejecución contra el sumidero demuestra la tasa objetivo alcanzable
  - label: Regla básica
    wert: solo sistemas propios o expresamente autorizados
werbung:
  - newsletter
ctaThemen:
  - smtp-mailflow
translationSourceHash: f6d1ef180221f41242e1d32b5fb8fb2e631894e3ccd32a3b3322a33b808da210
translationModel: gpt-5.6-terra
translatedAt: 2026-10-14T09:40:02.933Z
translationReview: automatic
---

# Pruebas funcionales y de carga: comprobar la infraestructura en condiciones controladas

Que un entorno soporte su carga máxima, que un conjunto de reglas se comporte igual tras una modificación o que un montaje de almacenamiento resista patrones de acceso reales no se demuestra en la ficha técnica, sino en las pruebas. Esta página reúne la metodología común a todos los tipos de prueba: qué se mide, cómo se estructura un proceso de prueba, qué herramientas generan y absorben carga, y cómo convertir datos en bruto en conclusiones fiables.

La regla más importante precede a cualquier metodología: las pruebas de carga y de perturbación solo deben realizarse en sistemas propios o en entornos expresamente autorizados para ello. Una ráfaga contra infraestructura ajena es un ataque; una prueba de correo con remitentes inventados contra destinos de producción genera backscatter y entradas en listas de bloqueo.

## Separar las pruebas funcionales de las pruebas de carga

Una prueba responde siempre a una única pregunta, y las dos preguntas fundamentales requieren configuraciones distintas. La **prueba funcional** verifica si una ruta funciona correctamente: una única transacción SMTP con control total de cada paso, un único acceso a archivo, una única ruta de reglas a través de un gateway. La **prueba de carga** comprueba cómo se comporta el sistema bajo volumen: rendimiento, distribución de latencia, colas y consumo de recursos.

El orden es obligatorio: primero la prueba funcional y después la carga. Una ráfaga a través de una ruta defectuosa no mide nada; solo produce diez mil errores idénticos. Un contador de errores del 100 por ciento con tiempos de respuesta cercanos a cero casi siempre significa que el error ocurre antes de la red y que la prueba nunca ha llegado al sistema objetivo.

## Las cuatro métricas

Las pruebas de carga en entornos multietapa miden cuatro aspectos distintos, y cada uno exige su propia perspectiva:

1. **Tasa de aceptación:** ¿Cuántas operaciones por segundo acepta el primer salto? Es el valor que los generadores de carga proporcionan directamente.
2. **Latencia de sesión:** ¿Cuánto dura una transacción individual? Bajo carga, este valor suele aumentar mucho antes de que caiga la tasa de aceptación.
3. **Latencia de extremo a extremo:** ¿Cuánto tarda una operación en llegar al destino a través de todas las etapas intermedias? Es la métrica que perciben los usuarios.
4. **Comportamiento de la cola:** ¿Hasta qué punto crece la cola durante el pico y con qué rapidez se vacía después? Un sistema que acepta todo y lo procesa durante horas supera la prueba de aceptación, pero aun así suspende.

## Proceso de prueba: línea base, ráfaga y soak

Una prueba fiable se desarrolla en tres etapas, cada una con su propia cuestión. La **línea base** establece, con una carga moderada y conocida, los valores de referencia de latencia y consumo de recursos, y detecta errores de configuración antes de que queden ocultos en la medición de picos. La **ráfaga** es la medición de la carga máxima propiamente dicha; varias ejecuciones con paralelismo creciente muestran dónde se aplana la tasa de aceptación y aumenta considerablemente la latencia. El **soak** aplica la carga sostenida esperada durante horas; solo aquí se manifiestan fugas de memoria, particiones de spool llenas y límites de conexión que una ráfaga corta nunca alcanza.

Esto incluye tres reglas prácticas. Primera, la **autoprueba del generador**: antes de la primera ejecución contra el sistema objetivo, el generador se ejecuta directamente contra el sumidero; esto demuestra que la configuración puede generar la tasa objetivo y proporciona el límite superior con el que se compararán las mediciones posteriores. Segunda, el **marcado**: cada operación de prueba lleva un marcador único (cabecera propia, convención de asunto, número de ejecución), para poder separar limpiamente las ejecuciones en los registros y eliminar por completo los datos de prueba después. Tercera, la **prueba de control**: un recuento independiente en el lado receptor confirma que ha llegado lo que el generador ha enviado; si las cifras no coinciden, algo se ha perdido en el camino.

## Herramientas: generador y sumidero

Una prueba de carga siempre consta de tres partes: generador de carga, sistema de prueba y sumidero controlado. Para carga SMTP, los generadores habituales son [smtp-source](https://www.postfix.org/smtp-source.1.html) del paquete Postfix (máxima carga bruta con esfuerzo mínimo), [Postal](https://doc.coker.com.au/projects/postal/) (carga sostenida con tasa objetivo y mensajes variables), [Apache JMeter](https://jmeter.apache.org/) (perfiles de carga, percentiles e informe HTML; la primera opción en Windows) y scripts propios cuando la prueba requiere mezclas de operaciones realistas. [swaks](https://www.jetmore.org/john/code/swaks/) no es un generador de carga, pero debe utilizarse como probador funcional antes de cada ráfaga.

En el lado receptor, un sumidero acepta todo y lo descarta: [smtp-sink](https://www.postfix.org/smtp-sink.1.html), `bhm` incluido con Postal o un pequeño servidor [aiosmtpd](https://aiosmtpd.aio-libs.org/) en unas pocas líneas de Python, que además realiza el recuento de control. Las herramientas de desarrollo con interfaz web no son adecuadas como sumideros para pruebas de carga; con decenas de miles de operaciones, de lo contrario se mediría la herramienta de análisis en lugar del sistema de prueba.

## Evaluación: percentiles, pivote y evolución

Los datos en bruto solo se convierten en conclusiones mediante tres pasos de evaluación. **Percentiles en lugar de medias:** una media de dos segundos puede significar que todo tarda dos segundos, o que el 95 por ciento se completa en medio segundo y el resto tarda considerablemente más; p50, p95 y p99 distinguen estos casos. **Pivotar los errores:** la distribución temporal de los códigos de error muestra cuándo un sistema empieza a limitar y qué mecanismo de protección interviene primero. **Evolución en lugar de estado final:** el rendimiento y la profundidad de la cola representados a lo largo del tiempo muestran la subida gradual, la meseta y la bajada; para la interpretación cuenta la meseta, no la media de toda la ejecución, pues esta incluye la subida y la bajada y subestima la tasa sostenida alcanzada.

Por último, la evaluación incluye una delimitación honesta: cada informe de prueba deja constancia de lo que la ejecución demuestra y de lo que no. Una medición sin TLS no dice nada sobre los costes de handshake de TLS; un sumidero local no dice nada sobre el comportamiento de la cola del destino real; una ráfaga no dice nada sobre fugas de memoria en funcionamiento continuo. Estos límites determinan las siguientes ejecuciones de prueba.

Los artículos siguientes aplican la metodología a casos concretos: comparación de herramientas y planificación de pruebas de carga de correo, una ejecución completa de JMeter con informe HTML, diagnóstico SMTP por capas con herramientas integradas, un plan de pruebas para montajes de almacenamiento en la nube y el método científico detrás de experimentos controlados en el flujo de correo.
