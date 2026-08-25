---
title: "Planificar pruebas de carga de correo: comparación de herramientas para ráfagas de 10.000 correos en Linux y Windows"
navTitle: "Pruebas de carga de correo"
description: "Quien migra una pasarela o dimensiona un entorno de correo necesita cifras fiables en lugar de intuiciones. Qué herramientas generan ráfagas de varias decenas de miles de correos, cómo elaborar un plan de pruebas riguroso y cómo evaluar los resultados a partir de los registros."
date: "2026-08-24"
kategorie: "SMTP y flujo de correo"
timeToRead: "12 min de lectura"
themen:
  - smtp-mailflow
  - testing
produkte:
  - "uebergreifend"
protokolle:
  - "testing"
  - "smtp"
  - "tcp"
  - "tls"
  - "troubleshooting"
slug: "planificar-pruebas-de-carga-de-correo-comparacion-de-herramientas-para-rafagas-de-10-000"
translationId: "article-14a98de0cef45565"
aiPrompt: |
  Du bist mein Assistent für Mail-Lasttests. Hilf mir Schritt für Schritt, einen Lasttest gegen mein eigenes Test-Mailgateway zu planen: Zieldefinition (Durchsatz, Latenz, Queue-Verhalten), Wahl des Lastgenerators (smtp-source, Postal, JMeter oder Skript), Aufbau einer Mail-Senke, Testablauf (Baseline, Burst, Soak) und Auswertung der Logs mit Perzentilen. Frage zuerst nach Plattform, Zielsystem und erwartetem Mailvolumen.
translationOf: mail-lasttest-tools-linux-windows-vergleich
url: https://rafaelpfister.ch/es/blog/planificar-pruebas-de-carga-de-correo-comparacion-de-herramientas-para-rafagas-de-10-000
translationSourceHash: c9b76f3c9887117756e07c71a3dc30d1ee99aeb8a322c50dee994a07df46cb97
translationModel: gpt-5.6-terra
translatedAt: 2026-08-25T04:12:12.966Z
translationReview: required
---

# Planificar pruebas de carga de correo: comparación de herramientas para ráfagas de 10.000 correos en Linux y Windows

Si una nueva pasarela de correo soporta la carga punta de una noche de ejecución de facturas no se demuestra en la ficha técnica, sino en las pruebas. Quien sustituye una appliance, dimensiona un entorno de Exchange o planifica el envío de una newsletter a través de su propia infraestructura necesita antes cifras fiables: ¿cuántos mensajes por segundo acepta el sistema, cómo se comporta la cola bajo presión y a partir de qué punto empiezan los aplazamientos? Este artículo compara los generadores de carga habituales en Linux y Windows y muestra cómo planificar, realizar y evaluar una prueba con ráfagas de varias decenas de miles de correos.

Ante todo, la regla más importante: las pruebas de carga pertenecen exclusivamente a la propia infraestructura o a un entorno de pruebas expresamente autorizado para ello. Una ráfaga contra sistemas ajenos es un ataque, y una prueba con direcciones de remitente inventadas contra destinos de producción genera backscatter que acaba en listas de bloqueo. Una configuración correcta consta de un generador de carga, el sistema que se va a probar y un sumidero controlado que acepta y descarta los correos al final.

## Qué debe medir una prueba de carga de correo

Antes de hablar de una herramienta, conviene preguntarse qué magnitud interesa realmente. En la práctica son cuatro distintas, y requieren configuraciones de prueba diferentes:

1. **Tasa de aceptación:** ¿Cuántos mensajes por segundo acepta el primer salto mediante SMTP? Esta es la medición clásica de rendimiento y el valor que proporcionan directamente los generadores de carga.
2. **Latencia de sesión:** ¿Cuánto dura una transacción SMTP individual desde el establecimiento de la conexión hasta `250` después de `DATA`? Bajo carga, este valor suele aumentar mucho antes de que caiga la tasa de aceptación.
3. **Latencia de extremo a extremo:** ¿Cuánto tarda un mensaje desde su entrega inicial hasta su llegada al sumidero, pasando por todas las estaciones intermedias? Esta es la magnitud que perciben los usuarios.
4. **Comportamiento de la cola:** ¿Hasta qué punto crece la cola durante la ráfaga y con qué rapidez se vacía después? Una pasarela que acepta 50.000 correos y luego los procesa durante tres horas supera la prueba de aceptación, pero aun así falla.

Una prueba que solo mide la tasa de aceptación dice poco sobre un entorno de varias capas con pasarela, capa de cifrado y servidor de destino. Precisamente ahí es decisiva la perspectiva de extremo a extremo.

## Herramientas en Linux

**smtp-source y smtp-sink** del paquete Postfix son el estándar para carga SMTP bruta y están disponibles prácticamente en cualquier sistema que tenga Postfix instalado. `smtp-source` genera mensajes con tamaño, paralelismo y cantidad configurables; `smtp-sink` es la contraparte: un servidor SMTP que acepta y descarta todo. Una ráfaga de 10.000 correos con 50 sesiones paralelas y mensajes de 5 KB se logra con una sola línea:

```bash
time smtp-source -s 50 -m 10000 -l 5120 -c -f last@test.example -t senke@test.example gateway.test.example:25
```

La opción `-c` cuenta en directo los mensajes enviados, mientras que `time` proporciona la duración total y, con ello, la tasa. Limitaciones importantes: `smtp-source` no mide percentiles de latencia y los mensajes sintéticos son uniformes. Sin embargo, para responder a «cuánto acepta el sistema como máximo» sigue siendo la primera opción, porque incluso en hardware modesto genera decenas de miles de mensajes por minuto y el generador prácticamente nunca se convierte en el cuello de botella.

**Postal** es el benchmark clásico de servidor de correo dedicado en Linux. Varía automáticamente remitentes, destinatarios y tamaño de los mensajes, mantiene una tasa objetivo durante periodos prolongados y escribe estadísticas por minuto. Por ello es más adecuado que `smtp-source` para pruebas de soak, es decir, carga sostenida durante horas. El `bhm` asociado (Black Hole Mailer) asume el papel de sumidero. Postal es antiguo, pero está diseñado precisamente para ello y se incluye en los repositorios de paquetes de la mayoría de las distribuciones.

**swaks** no es un generador de carga, pero debe formar parte de todo plan de pruebas. Ejecuta una única transacción SMTP con control completo sobre cada paso: autenticación, STARTTLS, cabeceras arbitrarias y adjuntos. Antes de cada prueba de carga debe ejecutarse swaks como prueba funcional, para que la ráfaga no falle por un destinatario incorrecto o un problema de TLS y deje la medición sin valor. En un bucle con `xargs -P` también puede utilizarse swaks como pequeño generador de carga, pero para decenas de miles de correos la sobrecarga de procesos es demasiado elevada.

**Scripts propios** en Python (smtplib, aiosmtplib) o Go son la vía adecuada cuando la prueba necesita mezclas de mensajes realistas: distintos tamaños, adjuntos reales, número variable de destinatarios por transacción y casos de error específicos. El esfuerzo es mayor, pero a cambio el script mide exactamente lo que verá después el propio entorno y puede registrar marcas de tiempo por mensaje para evaluar la latencia.

## Herramientas en Windows

**Apache JMeter** es la primera recomendación en Windows. El SMTP Sampler integrado admite autenticación, STARTTLS, adjuntos y archivos EML como fuente de mensajes, y la mecánica de JMeter proporciona lo que falta en las herramientas de Postfix: grupos de hilos para perfiles de carga escalonados, percentiles de tiempo de respuesta, tasas de error e informes. Para ráfagas de más de unos miles de correos por minuto se aplica la regla habitual de JMeter: usar la GUI solo para crear el plan de pruebas y ejecutar la medición propiamente dicha en modo CLI; de lo contrario, se estará midiendo la interfaz.

**PowerShell con MailKit** es la opción basada en herramientas integradas. El anteriormente habitual `Send-MailMessage` está marcado por Microsoft como obsoleto y recomienda migrar; MailKit puede cargarse mediante NuGet y paralelizarse desde PowerShell 7 con Runspaces. De forma realista, permite enviar de varios cientos a unos pocos miles de correos por minuto, suficiente para pruebas funcionales y de regresión, pero insuficiente para medir la carga máxima. La ventaja es que el script se ejecuta sin instalación adicional en cualquier puesto de trabajo de administración y puede escribir los resultados directamente como CSV para su evaluación.

**Microsoft Exchange Load Generator (LoadGen)** fue durante años la herramienta oficial para someter entornos de Exchange a carga con perfiles de usuario simulados (Outlook, ActiveSync, OWA). Microsoft dejó de mantenerlo después de Exchange 2013 y retiró la descarga. Para carga SMTP pura, LoadGen ya era en cualquier caso la herramienta equivocada; quien quiera simular hoy la carga de buzones de Exchange no dispone de una herramienta oficial y es mejor que pruebe directamente la ruta SMTP.

**WSL** merece un apartado propio: quien trabaja en una máquina Windows pero necesita herramientas de Linux instala `smtp-source` y Postal en una distribución WSL y obtiene así todo el conjunto de herramientas de Linux sin una VM de pruebas independiente. Para las cargas aquí tratadas, la ruta de red de WSL no supone un cuello de botella relevante.

## Comparación

| Herramienta | Plataforma | Ventaja | Límite |
|---|---|---|---|
| smtp-source / smtp-sink | Linux (Postfix) | Máxima carga bruta con mínimo esfuerzo, generador y sumidero de la misma solución | Sin percentiles de latencia, mensajes uniformes |
| Postal / bhm | Linux | Carga sostenida con tasa objetivo, mensajes variables, estadísticas por minuto | Herramientas antiguas, la evaluación debe construirse manualmente |
| swaks | Linux, Windows (Perl) | Prueba individual totalmente controlable, ideal como comprobación funcional antes de la ráfaga | No es un generador de carga |
| JMeter (SMTP Sampler) | Windows, Linux (Java) | Perfiles de carga, percentiles, informes, fuentes de mensajes EML | Sobrecarga de Java, trampa de la GUI a tasas altas |
| PowerShell + MailKit | Windows | Sin instalación adicional en cualquier equipo de administración, salida CSV | Rendimiento limitado, hay que construir la paralelización |
| Script propio (Python/Go) | ambos | Mezcla de mensajes realista, puntos de medición propios | Esfuerzo de desarrollo, validar el generador por cuenta propia |

## El sumidero: adónde van los correos

La mitad infravalorada de la configuración de prueba es el destino. Tres variantes han demostrado su eficacia:

- **smtp-sink** o `bhm` como agujero negro: acepta todo, descarta todo y mide la cadena de transporte pura. `smtp-sink` puede generar, si se desea, retardos artificiales de respuesta y códigos de error, y así comprobar también el comportamiento del sistema de prueba ante un destino lento o resistente.
- **Postfix con transporte discard** como sumidero más realista, si el propio destino debe ser un servidor SMTP completo con puesta en cola.
- **Unos pocos buzones semilla reales** además del sumidero, para comprobar de forma puntual que los mensajes llegan intactos en su contenido, incluida la capa de cifrado o firma.

Las herramientas con interfaz web como Mailpit están pensadas para el desarrollo y con decenas de miles de correos se convierten rápidamente en el cuello de botella. No son adecuadas como sumidero para una prueba de carga; la medición evaluaría la herramienta de análisis en lugar del sistema de prueba.

## Planificar la prueba

Una prueba fiable se desarrolla en tres etapas, cada una con su propia cuestión:

1. **Línea base:** Una carga moderada y conocida (aproximadamente el 10 % del pico esperado) durante unos minutos. Proporciona los valores de referencia de latencia y consumo de recursos, y descubre errores de configuración antes de que se pierdan en la medición de la ráfaga.
2. **Ráfaga:** La medición de carga punta propiamente dicha, por ejemplo, entre 10.000 y 50.000 correos lo más rápido posible o con una tasa objetivo definida. Varias ejecuciones con paralelismo creciente muestran dónde se aplana la tasa de aceptación y se dispara la latencia.
3. **Soak:** La carga diaria esperada durante varias horas. Solo aquí se manifiestan fugas de memoria, particiones de spool llenas, rotación de registros bajo carga y límites de conexión que una ráfaga corta nunca alcanza.

En cuanto a la mezcla de mensajes, debe ser tan realista como sea necesario. Una medición exclusivamente con correos de texto de 5 KB sobreestima varias veces el rendimiento de un entorno cuyo día a día incluye adjuntos PDF. Conviene utilizar una mezcla procedente de los propios datos, por ejemplo, 70 % pequeños, 25 % con adjunto típico y 5 % grandes. TLS también debe formar parte de la prueba si producción utiliza TLS: el handshake cuesta considerablemente más por conexión que la transmisión de los mensajes, y los generadores que abren una conexión nueva por correo miden de otro modo principalmente la terminación TLS.

Para la evaluación posterior, cada mensaje de prueba recibe un marcador inequívoco, lo más sencillo es una cabecera propia como `X-Loadtest-Id` con número de ejecución y marca de tiempo, además de una convención de asunto reconocible. Esto permite separar limpiamente las ejecuciones de prueba entre sí y del tráfico restante en los registros, y eliminar selectivamente los correos de prueba de cuarentenas y diarios después de la ejecución.

Hay tres puntos que se olvidan regularmente en la planificación: primero, los límites de tasa y el tarpitting en la ruta de prueba; una pasarela que limita después de 100 correos por minuto y por IP de origen solo probará su propia limitación (excluirlo específicamente para medir la carga máxima, mantenerlo deliberadamente para comprobar el realismo). Segundo, DNS: si el sistema de prueba resuelve dominios de destinatarios o realiza consultas DNSBL para cada mensaje, debe incluirse un resolver en el entorno de pruebas; de lo contrario, la prueba mide el DNS ascendente. Tercero, el propio generador: antes de la primera ejecución contra el sistema de destino, debe realizarse una ejecución del generador directamente contra el sumidero para demostrar que el generador puede producir la tasa objetivo.

## Evaluar

Los valores medidos por el generador de carga son solo la mitad de la verdad, pues muestran la perspectiva del remitente. La otra mitad se encuentra en los registros del sistema de prueba.

En Postfix, el registro de correo proporciona por mensaje los campos `delay` y `delays`, este último desglosado por tiempo en la cola, establecimiento de conexión y transmisión. Una evaluación de una ejecución de prueba se realiza con herramientas integradas:

```bash
grep "status=sent" /var/log/mail.log |
  grep -o "delay=[0-9.]*" | cut -d= -f2 | sort -n |
  awk '{a[NR]=$1} END {print "n="NR, "p50="a[int(NR*0.5)], "p95="a[int(NR*0.95)], "p99="a[int(NR*0.99)], "max="a[NR]}'
```

En el lado de Exchange, el Message Tracking Log es la fuente central. Para una ejecución de prueba con convención de asunto:

```powershell
$p = @{
    Start          = "24.08.2026 14:00"
    End            = "24.08.2026 15:00"
    MessageSubject = "LOADTEST"
    ResultSize     = "Unlimited"
}
Get-MessageTrackingLog @p | Group-Object EventId |
    Sort-Object Count -Descending | Format-Table Name, Count
```

La diferencia entre las marcas de tiempo de los eventos RECEIVE y DELIVER de la misma MessageId proporciona la latencia de extremo a extremo por mensaje; exportada como CSV, permite calcular la distribución de percentiles.

Para la interpretación cuentan tres principios básicos. Primero: percentiles en lugar de medias. Una media de dos segundos puede significar que todo tarda dos segundos, o que el 95 % termina en medio segundo y el resto permanece en la cola; p50, p95 y p99 distinguen estos casos. Segundo: pivotar los códigos de respuesta SMTP. La distribución temporal de las respuestas 4xx muestra cuándo el sistema empieza a limitar, y los códigos implicados (límite de conexiones, protección de cola, greylisting) indican qué mecanismo actúa primero. Tercero: representar la profundidad de la cola a lo largo del tiempo, en Postfix mediante `qshape` o `postqueue -j`, y en Exchange mediante `Get-Queue` cada minuto. El área bajo esta curva, no la tasa de aceptación, decide si el entorno absorbe una ráfaga o simplemente la almacena.

Junto con los registros de correo, la evaluación debe incluir las métricas del sistema de prueba: CPU, tiempos de espera de E/S en la partición de spool, descriptores de archivo y contadores de conexión. El hallazgo más frecuente en entornos de varias capas es que no limita el proceso de correo, sino una capa de inspección de contenido (antivirus, módulo de cifrado, DLP) con un número fijo de workers. Precisamente este tipo de hallazgos es el verdadero valor de la prueba: identifica el ajuste necesario antes de que lo haga producción.

## Conclusión

Para medir rápidamente la carga máxima en Linux, no hay alternativa a `smtp-source` con `smtp-sink`; Postal complementa el caso de carga sostenida. En Windows, JMeter ofrece la medición más completa, PowerShell con MailKit cubre las pruebas funcionales y de regresión, y WSL lleva las herramientas de Linux al puesto de trabajo de administración cuando es necesario. Más importante que la herramienta es el plan: medición separada de aceptación, latencia y comportamiento de la cola, una mezcla de mensajes realista, una ejecución de prueba marcada y una evaluación que incorpore percentiles y registros del sistema de destino en lugar de limitarse al contador del generador.

## Fuentes

1.  [smtp-source(1), manual de Postfix](https://www.postfix.org/smtp-source.1.html): Referencia del generador de carga con todas las opciones de paralelismo, tamaño de mensaje y TLS.

2.  [smtp-sink(1), manual de Postfix](https://www.postfix.org/smtp-sink.1.html): Referencia del sumidero de correo, incluidos retardos artificiales y respuestas de error.

3.  [Documentación de Postal, Russell Coker](https://doc.coker.com.au/projects/postal/): Descripción del benchmark de servidor de correo con tasa objetivo, variación de mensajes y sumidero bhm.

4.  [swaks, John Jetmore](https://www.jetmore.org/john/code/swaks/): El comprobador funcional SMTP para la revisión previa de cada ruta de prueba.

5.  [Apache JMeter Component Reference: SMTP Sampler](https://jmeter.apache.org/usermanual/component_reference.html): Funcionalidad del SMTP Sampler, incluida autenticación, TLS y fuentes EML.

6.  [Send-MailMessage, Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/send-mailmessage): Aviso oficial de Microsoft de que el cmdlet está obsoleto, con referencia a alternativas como MailKit.

7.  [MailKit, Jeffrey Stedfast](https://github.com/jstedfast/MailKit): La biblioteca de correo .NET para scripts de envío propios en PowerShell 7.

8.  [Get-MessageTrackingLog, Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog): Referencia para evaluar el Exchange Message Tracking Log después de una ejecución de prueba.

9.  [qshape(1), manual de Postfix](https://www.postfix.org/qshape.1.html): Herramienta para analizar la distribución de la cola durante y después de la ráfaga.
