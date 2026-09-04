---
title: "Cuando el registro llena el disco: cómo limitar correctamente log4j2 RollingFile, con TotemoMail como ejemplo"
navTitle: "Espacio en disco de log4j2"
description: "Un volumen de registros lleno puede, en el peor de los casos, paralizar toda la pasarela. Por qué la combinación de rotación por tiempo y tamaño sin %i genera un único archivo enorme, cómo strategy.max limita la retención, qué papel desempeña el nivel de registro y dónde oculta TotemoMail estos valores."
date: "2026-09-04"
kategorie: "TotemoMail"
timeToRead: "9 min de lectura"
themen:
  - totemomail
produkte:
  - "totemomail"
protokolle:
  - "troubleshooting"
  - "storage"
slug: "cuando-el-registro-llena-el-disco-como-limitar-correctamente-log4j2-rollingfile-con-totemomail"
translationId: "article-c400eee99d90052d"
translationOf: log4j2-rollingfile-plattenplatz-totemomail
url: https://rafaelpfister.ch/es/blog/cuando-el-registro-llena-el-disco-como-limitar-correctamente-log4j2-rollingfile-con-totemomail
translationSourceHash: 39952348654f81231356634fc8b434cbfecdea73118db7ff1add02720283792b
translationModel: gpt-5.6-terra
translatedAt: 2026-09-04T08:17:20.758Z
translationReview: automatic
---

# Cuando el registro llena el disco: cómo limitar correctamente log4j2 RollingFile, con TotemoMail como ejemplo

Una pasarela de correo basada en Java escribe cantidades sorprendentes en modo DEBUG. Un solo día de carga puede generar varios gigabytes de registro de trazas y, si el volumen de registros tiene poco espacio asignado, se llena. La consecuencia es desagradable: el proceso Java ya no puede escribir en su registro, el framework de logging entra en estado de error y, aun después de liberar espacio, no vuelve a escribir hasta reiniciarse. En una pasarela de correo, un disco lleno también puede interferir con el spooling y la entrega. El desencadenante casi siempre es una rotación de registros configurada, pero que no funciona como se supone.

Este artículo explica la rotación de log4j2 precisamente en este punto, primero de forma general y después específicamente para TotemoMail (que se basa en Apache James y log4j2). La clave es un único dato fácil de pasar por alto en el patrón de archivo.

## Cómo rota log4j2

El `RollingFileAppender` de log4j2 combina dos componentes: una o varias **TriggeringPolicies** deciden *cuándo* se rota, y una **RolloverStrategy** decide *cómo* se nombran los archivos de archivo y cuántos se conservan. Es habitual usar dos policies simultáneamente:

- `TimeBasedTriggeringPolicy`: rota por tiempo, normalmente a diario.
- `SizeBasedTriggeringPolicy`: rota en cuanto el archivo activo alcanza un tamaño, por ejemplo, 100 MB.

Durante el rollover, el archivo activo se renombra y archiva. El nombre del archivo de archivo lo determina el `filePattern`, que contiene dos marcadores de posición cuya interacción marca la diferencia decisiva.

<details class="options-details">
<summary>Opciones de un vistazo</summary>

| Marcador de posición | Significado |
|---|---|
| `%d{...}` | Fecha/hora del rollover según el patrón especificado, p. ej., `%d{yyyy-MM-dd}` para el día |
| `%i` | El índice calculado del archivo de archivo, un contador que aumenta con cada rollover |
| `%03i` | El mismo índice, rellenado con ceros hasta tres posiciones |
| `.gz` / `.zip` al final del patrón | El archivo se comprime durante el rollover |

</details>

La referencia completa se encuentra en la documentación de log4j2 sobre Rolling File Appender; la tabla anterior solo enumera los elementos esenciales para la rotación por tamaño y tiempo.

## La trampa de %i

Aquí es precisamente donde está el error que llena discos. Quien nombre solo por fecha, es decir, `filePattern = trace.log.%d{yyyy-MM-dd}`, y al mismo tiempo establezca una policy de tamaño de 100 MB, no obtendrá muchos archivos de 100 MB al día, sino uno solo que seguirá creciendo sin freno. La rotación por tamaño no tiene un destino propio al que pueda escribir el siguiente bloque, porque el patrón no contiene ningún contador. La documentación de log4j2 es clara al respecto:

> When combined with a time-based triggering policy, the filePattern attribute of the Appender should contain an `%i` conversion pattern. Otherwise, the target file will be overwritten on each rollover.

Sin `%i`, la combinación de rotación por tiempo y tamaño es, por tanto, defectuosa; según la estrategia, el archivo se sobrescribe o crece más allá del tamaño configurado. En la práctica, significa que el límite de 100 MB nunca se aplica, un día de carga escribe todo en un archivo y este alcanza varios gigabytes. La corrección consiste en añadirlo al patrón:

```text
filePattern = trace.log.%d{yyyy-MM-dd}.%i
```

Así, cada rollover de 100 MB crea su propio archivo indexado (`trace.log.2026-09-04.1`, `.2`, `.3`) y el límite de tamaño funciona como se espera.

## Retención mediante strategy.max

El índice es también el requisito para que funcione la retención. La `DefaultRolloverStrategy` posee un atributo `max` que indica el número máximo de archivos de archivo conservados; por encima de ese límite se eliminan los más antiguos. Sin `%i` no hay ningún índice que `max` pueda contar, por lo que tampoco se elimina nada y los archivos antiguos fechados se acumulan.

<details class="options-details">
<summary>Opciones explicadas</summary>

| Atributo | Efecto |
|---|---|
| `max` | Número máximo de archivos de archivo conservados; por encima de este se eliminan los más antiguos |
| `min` | Valor de índice mínimo (predeterminado: 1) |
| `fileIndex="min"` | El archivo más reciente recibe el índice `min`, el más antiguo `max` |
| `fileIndex="max"` (predeterminado) | El archivo más antiguo recibe el índice `min`, el más reciente `max` |
| `fileIndex="nomax"` | Nunca se elimina nada; los nuevos archivos reciben índices consecutivamente crecientes |

</details>

El tamaño y la cantidad determinan el límite total: 100 MB por archivo multiplicado por `max=10` limita el registro a aproximadamente un gigabyte, independientemente de cuánto se escriba. Quien necesite un control más preciso de la antigüedad en lugar de la cantidad puede añadir a la estrategia una acción `Delete` con `IfLastModified` (antigüedad) o `IfAccumulatedFileSize` (tamaño total); para la mayoría de los casos basta la combinación de tamaño por archivo y `max`.

## El nivel de registro como verdadero factor de volumen

La rotación y la retención limitan el consumo de espacio, pero no cambian cuánto se escribe en primer lugar. El mayor factor es el nivel de registro. Una pasarela en producción que funciona con DEBUG registra cada paso de procesamiento de cada mensaje y, bajo carga, esto suma gigabytes al día. Para el funcionamiento normal, el nivel debe estar en INFO o superior; DEBUG es una herramienta para análisis puntuales, no para uso permanente. Si el nivel está en INFO y además la rotación por tamaño con `%i` está configurada correctamente, ambos elementos se complementan: INFO mantiene baja la cantidad diaria y la rotación limita incluso una anomalía de DEBUG.

## Dónde guarda TotemoMail estos valores

En TotemoMail, estos ajustes no se encuentran en un `log4j2.xml` local, lo que puede desviar fácilmente la búsqueda de errores. La configuración se genera en tiempo de ejecución a partir de properties con el prefijo `totemo.log4j2.*`, y estas properties se gestionan centralmente mediante la consola de gestión (sección Logging + Tracking). Por ello, buscar `log4j2.xml` en el sistema de archivos no da resultados; un `log4j.xml` en el directorio de configuración pertenece a un componente incluido (openjms) y no tiene relación con el registro de trazas.

Las properties relevantes y su significado:

<details class="options-details">
<summary>Opciones explicadas</summary>

| Property | Significado |
|---|---|
| `totemo.log4j2.appender.a1.filePattern` | El patrón de archivo; aquí debe incluirse `%i` |
| `totemo.log4j2.appender.a1.policies.size.size` | Tamaño por archivo para la SizeBasedTriggeringPolicy, p. ej., `100MB` |
| `totemo.log4j2.appender.a1.strategy.max` | Número de archivos de archivo conservados |
| `totemo.log4j2.rootLogger.level` | Nivel del logger raíz de log4j2 |
| `totemo.log.priority` | Prioridad de registro global de la aplicación, el verdadero interruptor DEBUG |
| `totemo.tracking` | Nivel de detalle del seguimiento de mensajes; `debug` genera las líneas por Mailet |

</details>

Es importante su doble naturaleza: los loggers de log4j2 pueden estar en `warn` o `error` y, aun así, generar una avalancha de DEBUG en el registro de trazas, porque `totemo.log.priority` y `totemo.tracking` actúan como interruptores propios y superiores. Quien quiera reducir el volumen debe establecer `totemo.log.priority` en INFO y cambiar `totemo.tracking` de `debug` a `on`; esto elimina las líneas detalladas de procesamiento. Como los valores se gestionan mediante la consola, se aplican a todo el clúster, y algunos requieren reiniciar la instancia para surtir efecto (esto se indica en la property correspondiente).

## El reinicio después de llenarse el disco

Un detalle fácil de pasar por alto: después de que el disco se haya llenado una vez, el logging no vuelve por sí solo, aunque se libere espacio. El appender de archivo permanece en su estado de error hasta que se reinicia el proceso Java. Se reconoce porque la pasarela sigue aceptando y procesando correo (el banner SMTP muestra la hora correcta), pero el registro de trazas queda detenido en el momento en que se llenó el disco. Un reinicio controlado de la instancia restaura el logging y activa al mismo tiempo los ajustes modificados del appender, como el nuevo `filePattern`.

## Diagnóstico en pocos comandos

La partición llena y su causante pueden identificarse rápidamente. Primero se muestra qué sistema de archivos está afectado:

```bash
df -h
```

Si el volumen de registros está al 100 por ciento, una lista ordenada por tamaño identifica al principal causante:

```bash
du -sh /pfad/zu/logs/* | sort -rh | head
```

Si aparece un único archivo diario de varios gigabytes en lugar de muchos archivos indexados pequeños, se trata de la trampa de `%i`. Tras la corrección y un reinicio, la lista de archivos confirma que la rotación funciona:

```bash
ls -laht /pfad/zu/logs/trace.log*
```

Se esperan `trace.log` más archivos indexados `trace.log.<datum>.1`, `.2` y así sucesivamente, cada uno aproximadamente del tamaño máximo configurado.

## Resumen

Quien utilice log4j2 con rotación por tiempo y tamaño necesita obligatoriamente un `%i` en el `filePattern`; de lo contrario, un único archivo crecerá sin freno y el límite de tamaño no tendrá efecto. Mediante `strategy.max` (junto con el índice), el número de archivos limita el consumo de espacio, y el nivel de registro decide el volumen en origen. En TotemoMail, estos valores se encuentran en la consola de gestión bajo `totemo.log4j2.*`, así como en los interruptores superiores `totemo.log.priority` y `totemo.tracking`; después de llenarse el disco, debe reiniciarse la instancia para que el logging vuelva a escribir.

## Fuentes

1.  [Apache Logging Services: Log4j RollingFileAppender](https://logging.apache.org/log4j/2.x/manual/appenders/rolling-file.html): referencia sobre filePattern, las TriggeringPolicies y la DefaultRolloverStrategy, incluida la indicación sobre `%i` con rotación basada en tiempo.

2.  [Apache Logging Services: Log4j Architecture](https://logging.apache.org/log4j/2.x/manual/architecture.html): contextualización de Appender, Layout y jerarquía de logger, para comprender el logger raíz y el nivel de registro.
