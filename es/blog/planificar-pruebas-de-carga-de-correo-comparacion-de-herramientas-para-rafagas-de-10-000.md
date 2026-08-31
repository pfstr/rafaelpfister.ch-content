---
title: "Planificar pruebas de carga de correo: comparación de herramientas para ráfagas de 10.000 correos en Linux y Windows"
navTitle: "Pruebas de carga de correo"
description: "Quien migra una pasarela o dimensiona un entorno de correo necesita cifras fiables en lugar de intuiciones. Qué herramientas generan ráfagas de varias decenas de miles de correos, cómo elaborar un plan de pruebas limpio y cómo evaluar los resultados a partir de los registros."
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
translationSourceHash: 2fd0b1bd0748b9fb44be85907a946bbf85604b5eb7c85107170fa7443068efd7
translationModel: gpt-5.6-terra
translatedAt: 2026-08-30T09:27:09.908Z
translationReview: required
url: https://rafaelpfister.ch/es/blog/planificar-pruebas-de-carga-de-correo-comparacion-de-herramientas-para-rafagas-de-10-000
---

# Planificar pruebas de carga de correo: comparación de herramientas para ráfagas de 10.000 correos en Linux y Windows

Solo es posible comprobar si una nueva pasarela de correo soporta la carga máxima de un proceso nocturno de facturación mediante una prueba de carga. Quien sustituye una appliance, dimensiona un entorno de Exchange o planea el envío de un boletín a través de su propia infraestructura necesita antes cifras fiables: ¿cuántos mensajes por segundo acepta el sistema?, ¿cómo se comporta la cola bajo presión?, ¿y a partir de qué punto comienzan los aplazamientos? Este artículo compara los generadores de carga habituales en Linux y Windows y muestra cómo planificar, realizar y evaluar una prueba con ráfagas de varias decenas de miles de correos.

La regla más importante, antes de empezar: las pruebas de carga pertenecen exclusivamente a la propia infraestructura o a un entorno de pruebas expresamente autorizado para ello. Una ráfaga contra sistemas ajenos es un ataque, y una prueba con direcciones de remitente inventadas contra destinos productivos genera backscatter que lleva a listas de bloqueo. Una configuración correcta consta de un generador de carga, el sistema que se va a probar y un sumidero controlado que acepta y descarta los correos al final.

## Qué debe medir una prueba de carga de correo

Antes de hablar de una herramienta, conviene preguntarse qué magnitud interesa realmente. En la práctica son cuatro distintas, y requieren configuraciones de prueba diferentes:

1. **Tasa de aceptación:** ¿Cuántos mensajes por segundo acepta el primer salto mediante SMTP? Esta es la medición clásica de rendimiento y el valor que los generadores de carga proporcionan directamente.
2. **Latencia de sesión:** ¿Cuánto dura una transacción SMTP individual, desde el establecimiento de la conexión hasta el `250` después de `DATA`? Bajo carga, este valor a menudo aumenta mucho antes de que caiga la tasa de aceptación.
3. **Latencia de extremo a extremo:** ¿Cuánto tarda un mensaje desde su entrega inicial hasta su llegada al sumidero, pasando por todas las estaciones intermedias? Esta es la magnitud que perciben los usuarios.
4. **Comportamiento de la cola:** ¿Hasta qué punto crece la cola durante la ráfaga y con qué rapidez se vacía después? Una pasarela que acepta 50.000 correos y luego los procesa durante tres horas supera la prueba de aceptación, pero aun así suspende.

Una prueba que solo mide la tasa de aceptación dice poco sobre un entorno de varias etapas con pasarela, nivel de cifrado y servidor de destino. Precisamente ahí es decisiva la visión de extremo a extremo.

## El perfil de carga determina la herramienta

Además de la magnitud de medición, una segunda pregunta determina la elección de la herramienta y con frecuencia se omite: ¿qué comportamiento de conexión tiene la carga que se desea simular? Hay que distinguir dos perfiles de carga.

Un **remitente masivo con sesiones abiertas** es el perfil de carga de procesos de facturación, nóminas y sistemas de boletines: un único sistema establece pocas conexiones y envía por ellas cientos o miles de mensajes seguidos. La sobrecarga de conexión se produce una vez por sesión, no una vez por mensaje, y la pasarela ve pocas conexiones con muchas transacciones.

**Muchos remitentes independientes** son el perfil de carga de los entornos de aplicaciones y del tráfico de usuarios: numerosos sistemas entregan mensajes individuales, cada uno mediante su propia conexión. Aquí, el establecimiento de conexión, incluidos TLS y AUTH, forma parte de cada mensaje.

Para dimensionar un envío masivo, cuenta el primer perfil de carga, y para ello el generador de carga debe poder mantener abiertas las sesiones: `smtp-source` lo hace (muchos mensajes distribuidos entre pocas sesiones), al igual que Postal y los scripts propios con conexión persistente. JMeter no puede hacerlo; los motivos se explican en la sección de Windows. Por tanto, para la carga máxima de un proceso de facturación este criterio de sesión es determinante, no la plataforma; en Windows, la vía pasa por WSL.

## Herramientas en Linux

**smtp-source y smtp-sink** del paquete Postfix son el estándar para carga SMTP bruta y están disponibles en prácticamente cualquier sistema que tenga Postfix instalado. `smtp-source` genera mensajes con tamaño, paralelismo y cantidad configurables; `smtp-sink` es su equivalente: un servidor SMTP que acepta y descarta todo. Una ráfaga de 10.000 correos con 50 sesiones paralelas y mensajes de 5 KB cabe en una sola línea:

```bash
time smtp-source -s 50 -m 10000 -l 5120 -c -f last@test.example -t senke@test.example gateway.test.example:25
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `time` | Mide la duración total de la ejecución; de ella se obtiene la tasa en correos por segundo |
| `-s 50` | 50 sesiones SMTP paralelas |
| `-m 10000` | Número total de mensajes, distribuidos entre las sesiones |
| `-l 5120` | Tamaño del cuerpo del mensaje en bytes (sin cabeceras), aquí 5 KB |
| `-c` | Contador continuo de mensajes enviados como indicador de progreso |
| `-f last@test.example` | Dirección del remitente |
| `-t senke@test.example` | Dirección del destinatario |
| `gateway.test.example:25` | Host y puerto de destino para la entrega inicial |

</details>

Límites importantes: `smtp-source` no mide percentiles de latencia y los mensajes son sintéticos y uniformes. Sin embargo, para responder a «cuánto acepta el sistema como máximo» sigue siendo la primera opción, porque incluso con hardware modesto genera decenas de miles de mensajes por minuto y el generador prácticamente nunca se convierte en el cuello de botella.

**Postal** es el benchmark clásico de servidor de correo dedicado en Linux. Varía automáticamente remitente, destinatario y tamaño de mensaje, mantiene una tasa objetivo durante periodos prolongados y escribe estadísticas por minuto. Por ello resulta más adecuado que `smtp-source` para pruebas de resistencia, es decir, carga sostenida durante horas. El `bhm` asociado (Black Hole Mailer) asume el papel de sumidero. Postal es antiguo, pero está diseñado exactamente para ello y se incluye en los repositorios de paquetes de la mayoría de las distribuciones.

**swaks** no es un generador de carga, pero debe formar parte de todo plan de pruebas. Ejecuta una transacción SMTP individual con control completo de cada paso: autenticación, STARTTLS, cabeceras arbitrarias y adjuntos. Antes de cada prueba de carga debe realizarse una ejecución de swaks como prueba funcional, para que la ráfaga no falle por un destinatario incorrecto o un problema de TLS y deje la medición sin valor. En un bucle con `xargs -P` también se puede abusar de swaks como pequeño generador de carga, pero para decenas de miles de correos la sobrecarga de procesos es demasiado alta.

**Scripts propios** en Python (smtplib, aiosmtplib) o Go son la vía adecuada cuando la prueba requiere mezclas realistas de mensajes: distintos tamaños, adjuntos reales, números variables de destinatarios por transacción y casos de error específicos. El esfuerzo es mayor, pero a cambio el script mide exactamente lo que verá el propio entorno más adelante y puede registrar marcas de tiempo por mensaje para evaluar la latencia.

## Herramientas en Windows

**Apache JMeter** es la herramienta adecuada en Windows cuando el perfil de carga consiste en muchos remitentes independientes o cuando los percentiles, la mezcla de mensajes y los informes son prioritarios. El SMTP Sampler integrado admite autenticación, STARTTLS, adjuntos y archivos EML como fuente de mensajes, y el mecanismo de JMeter proporciona lo que falta en las herramientas de Postfix: grupos de hilos para perfiles de carga escalonados, percentiles de tiempo de respuesta, tasas de error e informes. Para ráfagas que superan unos pocos miles de correos por minuto se aplica la regla habitual de JMeter: usar la GUI solo para crear el plan de pruebas y ejecutar la medición en modo CLI; de lo contrario, se mide la interfaz.

Debe conocerse una limitación del SMTP Sampler: JMeter no puede mantener abiertas las sesiones SMTP. Cada ejecución de muestra abre una conexión nueva, recorre el diálogo completo de handshake TCP, EHLO, opcionalmente STARTTLS y AUTH, envía exactamente un mensaje y termina la conexión con QUIT. No es posible representar varios mensajes a través de la misma conexión abierta, como hacen los remitentes masivos con reutilización de sesión; en cambio, `smtp-source` distribuye muchos mensajes entre pocas sesiones abiertas. El motivo está en la arquitectura: JMeter es un framework de pruebas de carga multiprotocolo, no una herramienta SMTP. Su modelo de ejecución trata cada sampler como una unidad cerrada, medida de forma independiente, porque solo así los temporizadores, las aserciones y la evaluación de percentiles funcionan de forma uniforme para todos los protocolos compatibles. En consecuencia, el SMTP Sampler es una capa ligera sobre la biblioteca JavaMail, que como API de cliente establece y cierra una conexión para cada operación de envío; la reutilización de conexiones entre muestras, como la que ofrece el sampler HTTP con Keep-Alive, nunca se implementó para SMTP. Para la medición, esto significa que JMeter genera el perfil de carga de muchos remitentes individuales, no el de un remitente masivo con sesión abierta. El rendimiento medido incluye por mensaje toda la sobrecarga de conexión y TLS, y los límites de conexión de la pasarela actúan en consecuencia antes que con la reutilización de sesión. Para el perfil de carga de remitente masivo de un proceso de facturación, JMeter no es por tanto la herramienta adecuada; en Windows, la vía WSL con `smtp-source` es la mejor opción.

**PowerShell con MailKit** es la vía con herramientas integradas. El antiguo `Send-MailMessage` está marcado por Microsoft como obsoleto y recomienda la migración; MailKit se puede cargar mediante NuGet y paralelizar desde PowerShell 7 con runspaces. De forma realista, permite algunos cientos hasta pocos miles de correos por minuto, suficiente para pruebas funcionales y de regresión, pero demasiado poco para medir la carga máxima. La ventaja: el script se ejecuta sin instalación adicional en cualquier puesto de trabajo de administración y puede escribir los resultados directamente como CSV para su evaluación.

**Microsoft Exchange Load Generator (LoadGen)** fue durante años la herramienta oficial para someter entornos de Exchange a carga con perfiles de usuario simulados (Outlook, ActiveSync, OWA). Microsoft dejó de mantenerlo después de Exchange 2013 y retiró la descarga. Para carga SMTP pura, LoadGen ya era de todos modos la herramienta equivocada; quien hoy quiera simular carga de buzones de Exchange no dispone de una herramienta oficial y es mejor que pruebe directamente la ruta SMTP.

**WSL** merece un apartado propio: quien trabaje en una máquina Windows pero necesite herramientas Linux puede instalar `smtp-source` y Postal en una distribución WSL y disponer así de todas las herramientas Linux sin una VM de pruebas independiente. Para las cargas tratadas aquí, la ruta de red de WSL no constituye un cuello de botella relevante.

## Comparación

| Herramienta | Plataforma | Fortaleza | Límite |
|---|---|---|---|
| smtp-source / smtp-sink | Linux (Postfix) | Máxima carga bruta con un esfuerzo mínimo, generador y sumidero de una sola fuente | Sin percentiles de latencia, mensajes uniformes |
| Postal / bhm | Linux | Carga sostenida con tasa objetivo, mensajes variados y estadísticas por minuto | Herramientas anticuadas, evaluación a construir manualmente |
| swaks | Linux, Windows (Perl) | Prueba individual totalmente controlable, ideal como comprobación funcional antes de la ráfaga | No es un generador de carga |
| JMeter (SMTP Sampler) | Windows, Linux (Java) | Perfiles de carga, percentiles, informes, fuentes de mensajes EML | Sobrecarga de Java, modo GUI inadecuado para tasas altas, una conexión por mensaje (sin reutilización de sesión) |
| PowerShell + MailKit | Windows | Sin instalación adicional en cualquier equipo de administración, salida CSV | Rendimiento limitado, paralelización a construir manualmente |
| Script propio (Python/Go) | ambos | Mezcla realista de mensajes, puntos de medición propios | Esfuerzo de desarrollo, validar el generador manualmente |

## El sumidero: adónde van los correos

La mitad infravalorada de la configuración de prueba es el destino. Han demostrado su valía tres variantes:

- **smtp-sink** o `bhm` como agujero negro: aceptan todo, descartan todo y miden la cadena de transporte pura. Si se desea, `smtp-sink` puede generar retrasos artificiales de respuesta y códigos de error, y con ello comprobar también el comportamiento del sistema bajo prueba ante un destino lento o que responde con errores.
- **Postfix con transporte discard** como sumidero más realista, cuando el destino debe ser también un servidor SMTP completo con gestión de cola.
- **Unos pocos buzones semilla reales** además del sumidero, para comprobar por muestreo que los mensajes llegan intactos en contenido, incluido el nivel de cifrado o firma.

Las herramientas con interfaz web, como Mailpit, están pensadas para el desarrollo y con decenas de miles de correos se convierten rápidamente en el cuello de botella. No son adecuadas como sumidero para una prueba de carga; la medición acabaría evaluando la herramienta de análisis en lugar del sistema bajo prueba.

## Planificar la prueba

Una prueba fiable se desarrolla en tres fases, cada una con su propia pregunta:

1. **Línea base:** Una carga moderada y conocida (aproximadamente el 10 % del pico esperado) durante unos minutos. Proporciona valores de referencia para la latencia y el consumo de recursos, y detecta errores de configuración antes de que queden ocultos en la medición de ráfaga.
2. **Ráfaga:** La medición real de carga máxima, por ejemplo 10.000 a 50.000 correos tan rápido como sea posible o con una tasa objetivo definida. Varias ejecuciones con paralelismo creciente muestran dónde se aplana la tasa de aceptación y se dispara la latencia.
3. **Resistencia:** La carga diaria esperada durante varias horas. Solo aquí aparecen fugas de memoria, particiones de spool llenas, rotación de registros bajo carga y límites de conexión que una ráfaga corta nunca alcanza.

Para la mezcla de mensajes se aplica: tan realista como sea necesario. Una medición exclusivamente con correos de texto de 5 KB sobreestima varias veces el rendimiento de un entorno cuyo día a día incluye adjuntos PDF. Tiene sentido usar una mezcla del propio inventario, por ejemplo un 70 % de mensajes pequeños, un 25 % con adjunto típico y un 5 % de mensajes grandes. TLS también debe formar parte de la prueba si producción utiliza TLS: el handshake cuesta por conexión bastante más que la propia transmisión del mensaje, y los generadores que abren una conexión nueva por correo miden de otro modo principalmente la terminación TLS.

Para la posterior evaluación, cada mensaje de prueba recibe un marcador único, lo más sencillo es una cabecera propia como `X-Loadtest-Id` con número de ejecución y marca de tiempo, además de una convención de asunto reconocible. Así se pueden separar limpiamente las ejecuciones de prueba en los registros, tanto entre sí como del tráfico restante, y los correos de prueba se pueden eliminar de forma selectiva de cuarentenas y diarios después de la ejecución.

Tres puntos que se olvidan habitualmente en la planificación: primero, los límites de tasa y el tarpitting en la ruta de prueba; una pasarela que limita después de 100 correos por minuto por IP de origen solo probará su propia limitación (excluirla deliberadamente para la medición de carga máxima, dejarla conscientemente activa para comprobar el realismo). Segundo, DNS: si el sistema de prueba resuelve dominios de destinatarios o realiza consultas DNSBL para cada mensaje, debe incluirse un resolvedor en el entorno de pruebas; de lo contrario, la prueba medirá el DNS ascendente. Tercero, el propio generador: antes de la primera ejecución contra el sistema de destino, debe ejecutarse el generador directamente contra el sumidero para demostrar que puede producir la tasa objetivo.

## Evaluar

Los valores de medición del generador de carga son solo la mitad de la verdad, pues muestran la perspectiva del entregador. La otra mitad se encuentra en los registros del sistema bajo prueba.

En Postfix, el registro de correo proporciona por mensaje los campos `delay` y `delays`, este último desglosado por tiempo en la cola, establecimiento de conexión y transmisión. Se puede evaluar una ejecución de prueba con herramientas integradas:

```bash
grep "status=sent" /var/log/mail.log |
  grep -o "delay=[0-9.]*" | cut -d= -f2 | sort -n |
  awk '{a[NR]=$1} END {print "n="NR, "p50="a[int(NR*0.5)], "p95="a[int(NR*0.95)], "p99="a[int(NR*0.99)], "max="a[NR]}'
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `grep "status=sent" /var/log/mail.log` | Filtra el registro de correo por mensajes entregados correctamente |
| `grep -o "delay=[0-9.]*"` | `-o` muestra solo la coincidencia misma, aquí el campo `delay` con su valor |
| `cut -d= -f2` | Separa por `=` (`-d`) y conserva el segundo campo (`-f2`), es decir, el valor numérico |
| `sort -n` | Ordena numéricamente en lugar de alfabéticamente; requisito para calcular percentiles |
| `awk '…'` | Recopila los valores ordenados en un array e imprime cantidad, p50, p95, p99 y máximo |

</details>

En Exchange, el Message Tracking Log es la fuente central. Para una ejecución de prueba con convención de asunto:

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

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Start` / `End` | Ventana temporal de la búsqueda en los registros; aquí se pasa mediante splatting (`@p`) |
| `MessageSubject "LOADTEST"` | Filtra mensajes cuyo asunto contiene el marcador |
| `ResultSize Unlimited` | Elimina el límite predeterminado de 1000 entradas devueltas |
| `Group-Object EventId` | Agrupa los eventos de seguimiento por tipo (RECEIVE, DELIVER, DEFER, …) |
| `Sort-Object Count -Descending` | Ordena los grupos de eventos de forma descendente por frecuencia |
| `Format-Table Name, Count` | Muestra la cantidad por tipo de evento |

</details>

La diferencia entre las marcas de tiempo de los eventos RECEIVE y DELIVER del mismo MessageId proporciona la latencia de extremo a extremo por mensaje; al exportarla como CSV se puede calcular a partir de ella la distribución de percentiles.

Al interpretar los resultados cuentan tres principios básicos. Primero: percentiles en lugar de medias. Un promedio de dos segundos puede significar que todo tarda dos segundos, o que el 95 % termina en medio segundo y el resto quedó esperando en la cola; p50, p95 y p99 distinguen estos casos. Segundo: analizar los códigos de respuesta SMTP. La distribución de las respuestas 4xx a lo largo del tiempo muestra cuándo el sistema empieza a limitar, y los códigos concretos (límite de conexión, protección de cola, greylisting) indican qué mecanismo actúa primero. Tercero: representar la profundidad de la cola a lo largo del tiempo, en Postfix mediante `qshape` o `postqueue -j`, en Exchange mediante `Get-Queue` cada minuto. El área bajo esta curva, no la tasa de aceptación, determina si el entorno absorbe una ráfaga o simplemente la almacena.

Junto con los registros de correo, las métricas del sistema bajo prueba deben formar parte de la evaluación: CPU, tiempos de espera de I/O en la partición de spool, descriptores de archivos y contadores de conexiones. El hallazgo más habitual en entornos de varias etapas es que el proceso de correo no limita, sino una etapa de inspección de contenido (antivirus, módulo de cifrado, DLP) con un número fijo de workers. Precisamente estos hallazgos son el verdadero valor de la prueba: identifican el ajuste que se debe realizar antes de que lo descubra producción.

## Conclusión

Para medir rápidamente la carga máxima en Linux, no hay alternativa a `smtp-source` con `smtp-sink`; Postal complementa el caso de carga sostenida. En Windows, JMeter proporciona la medición más completa, PowerShell con MailKit cubre las pruebas funcionales y de regresión, y WSL lleva las herramientas Linux al puesto de administración cuando hace falta. Más importante que la herramienta es el plan: medición separada de aceptación, latencia y comportamiento de la cola, una mezcla realista de mensajes, una ejecución de prueba marcada y una evaluación que incluya percentiles y registros del sistema de destino en lugar de limitarse al contador del generador.

## Fuentes

1.  [smtp-source(1), manual de Postfix](https://www.postfix.org/smtp-source.1.html): Referencia del generador de carga con todas las opciones de paralelismo, tamaño de mensaje y TLS.

2.  [smtp-sink(1), manual de Postfix](https://www.postfix.org/smtp-sink.1.html): Referencia del sumidero de correo, incluidos los retrasos artificiales y las respuestas de error.

3.  [Documentación de Postal, Russell Coker](https://doc.coker.com.au/projects/postal/): Descripción del benchmark de servidor de correo con tasa objetivo, variación de mensajes y sumidero bhm.

4.  [swaks, John Jetmore](https://www.jetmore.org/john/code/swaks/): El comprobador funcional SMTP para la verificación previa de cada ruta de prueba.

5.  [Apache JMeter Component Reference: SMTP Sampler](https://jmeter.apache.org/usermanual/component_reference.html): Funcionalidad del SMTP Sampler, incluidas Auth, TLS y fuentes EML.

6.  [Send-MailMessage, Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/send-mailmessage): Aviso oficial de Microsoft de que el cmdlet está obsoleto, con referencia a alternativas como MailKit.

7.  [MailKit, Jeffrey Stedfast](https://github.com/jstedfast/MailKit): La biblioteca de correo .NET para scripts de envío propios en PowerShell 7.

8.  [Get-MessageTrackingLog, Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog): Referencia para evaluar el Exchange Message Tracking Log después de una ejecución de prueba.

9.  [qshape(1), manual de Postfix](https://www.postfix.org/qshape.1.html): Herramienta para analizar la distribución de la cola durante y después de la ráfaga.
