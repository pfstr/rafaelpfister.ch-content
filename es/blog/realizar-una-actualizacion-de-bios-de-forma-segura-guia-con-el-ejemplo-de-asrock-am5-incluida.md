---
title: "Realizar una actualización de BIOS de forma segura: guía con el ejemplo de ASRock AM5, incluida la preparación de BitLocker"
navTitle: "Actualización de BIOS"
description: "El proceso completo de una actualización de BIOS usando como ejemplo una placa ASRock AM5: determinar la versión, verificar la descarga mediante hash, suspender correctamente BitLocker, arrancar en UEFI (incluso si F2 no funciona), actualizar con Instant Flash y configurar sensatamente los ajustes después de la actualización."
date: "2026-08-26"
kategorie: "PC y hardware"
timeToRead: "8 min de lectura"
themen:
  - pc-hardware
produkte:
  - "windows-client"
protokolle:
  - "troubleshooting"
  - "releases"
slug: "realizar-una-actualizacion-de-bios-de-forma-segura-guia-con-el-ejemplo-de-asrock-am5-incluida"
translationId: "article-82840b2d159b9367"
translationOf: bios-update-asrock-am5-sicher-durchfuehren
translationSourceHash: 555b16e753b2ac5dec357741b071ed6aa33de367a2197a8dbb10fef7c9f6a946
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:14:53.259Z
translationReview: automatic
url: https://rafaelpfister.ch/es/blog/realizar-una-actualizacion-de-bios-de-forma-segura-guia-con-el-ejemplo-de-asrock-am5-incluida
---

Una actualización de BIOS forma parte de las tareas de mantenimiento que se realizan pocas veces y que, por ello, vuelven a suscitar preguntas cada vez: ¿qué versión es la correcta?, ¿cómo llega de forma segura a la placa?, ¿y qué hay que tener en cuenta antes y después? Esta guía documenta todo el proceso utilizando como ejemplo una ASRock A620I Lightning WiFi (socket AM5) con el método propio del fabricante, Instant Flash. Los pasos son aplicables a cualquier placa base moderna; los puntos críticos (BitLocker, Fast Boot y restablecimiento de ajustes) son independientes del fabricante.

## Cuándo conviene actualizar la BIOS

Hay tres motivos que justifican la intervención. Primero, las correcciones de seguridad: las vulnerabilidades de firmware solo se pueden cerrar mediante una actualización de BIOS, y los registros de cambios de los fabricantes suelen mencionarlas de forma escueta. Segundo, la compatibilidad: la compatibilidad con nuevas generaciones de CPU y una mejor compatibilidad de memoria llegan exclusivamente mediante nuevas versiones de firmware; en AM5, a través del firmware de referencia AGESA de AMD, que los fabricantes de placas integran en sus versiones de BIOS. Tercero, la estabilidad: si un sistema se reinicia espontáneamente y el registro de eventos solo registra Kernel-Power 41 con `BugcheckCode=0`, el fallo se produjo a nivel de hardware o firmware, sin intervención de Windows; las causas típicas son tensiones inestables y el entrenamiento de memoria, precisamente la capa que mantienen las versiones de AGESA. Entradas como "Improve memory compatibility and system stability" o un manejo revisado de EXPO en los registros de cambios indican que una actualización aborda este tipo de problemas. En cambio, si un sistema funciona de forma estable y no le afectan las vulnerabilidades corregidas, esperar es legítimo; una actualización de BIOS sin motivo es un riesgo sin beneficio.

## Paso 1: Determinar el estado actual

Antes de descargar nada, necesita dos datos: el modelo exacto de la placa y la versión de BIOS instalada. PowerShell proporciona ambos sin reiniciar:

```powershell
Get-CimInstance Win32_ComputerSystem | Select-Object Manufacturer, Model
Get-CimInstance Win32_BIOS | Select-Object SMBIOSBIOSVersion, ReleaseDate
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Win32_ComputerSystem` | Argumento posicional ClassName: clase CIM con el fabricante y el modelo del sistema |
| `Win32_BIOS` | Clase CIM con los datos del firmware, incluidos la versión y la fecha |
| `Select-Object <eigenschaften>` | reduce la salida a las propiedades indicadas |

</details>

Anote la versión. La necesitará más adelante para comprobar el éxito y, al leer los registros de cambios, querrá saber qué versiones está omitiendo.

## Paso 2: Descargar la BIOS y verificar la suma de comprobación

Descargue la BIOS exclusivamente desde la página del producto del fabricante, nunca desde portales de terceros. ASRock publica la suma de comprobación SHA256 para cada versión; tras la descarga, compárela antes de que el archivo se acerque siquiera a una memoria USB:

```powershell
Get-FileHash .\A620I_Lightning_WiFi_4.43.zip -Algorithm SHA256
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `.\A620I_…_4.43.zip` | Argumento posicional Path: el archivo que se debe comprobar |
| `-Algorithm SHA256` | Método hash; debe coincidir con el tipo de suma de comprobación publicado por el fabricante |

</details>

Si el valor no coincide con el indicado por el fabricante, la descarga está dañada o manipulada: no la flashee. Tras descomprimirlo, queda un único archivo ROM, en el ejemplo `A62IRW_4.43.ROM` de 32 MB.

## Paso 3: Preparar la memoria USB

El mecanismo de flasheo en el UEFI (en ASRock, "Instant Flash"; en otros fabricantes, Q-Flash, EZ Flash o M-Flash) lee la memoria directamente desde el firmware. Esto significa que solo FAT32 se reconoce de forma fiable; NTFS y exFAT no. Casi todas las memorias compradas ya vienen en FAT32; puede comprobarlo así:

```powershell
Get-CimInstance Win32_LogicalDisk -Filter "DriveType=2" |
  Select-Object DeviceID, FileSystem, VolumeName
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Win32_LogicalDisk` | Clase CIM de las unidades lógicas |
| `-Filter "DriveType=2"` | Filtro WQL para medios extraíbles; oculta discos duros y unidades de CD |
| `Select-Object DeviceID, FileSystem, VolumeName` | muestra la letra de unidad, el sistema de archivos y el nombre del volumen |

</details>

Copie el archivo ROM en el directorio raíz de la memoria USB. Solo es necesario reformatearla si el sistema de archivos no es adecuado. La capacidad de la memoria no es relevante: el archivo es más pequeño que cualquier capacidad habitual hoy en día.

Una nota sobre la elección del método: muchas placas ofrecen además un botón BIOS Flashback, que flashea sin CPU y sin un sistema funcional. Es la vía de rescate para una placa que ya no arranca. Para un sistema en funcionamiento, Instant Flash en el UEFI es el método correcto y más sencillo. Las herramientas de flasheo basadas en Windows no son necesarias ni recomendables en las plataformas actuales.

## Paso 4: Suspender BitLocker; de lo contrario, se solicitará la clave

Este es el punto que falta en muchas guías. Si la unidad del sistema está cifrada con BitLocker (en Windows 11 con una cuenta Microsoft, suele activarse automáticamente), BitLocker vincula la clave a los valores de medición del TPM. Una actualización de BIOS modifica esos valores y, en el siguiente arranque, Windows solicita la clave de recuperación de 48 dígitos. Quien no la tenga a mano se encontrará ante un sistema inaccesible.

BitLocker incluye un mecanismo propio para este escenario. En una PowerShell con permisos de administrador:

```powershell
Suspend-BitLocker -MountPoint C: -RebootCount 2
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-MountPoint C:` | El volumen afectado, aquí la unidad del sistema |
| `-RebootCount 2` | Número de reinicios durante los cuales la protección permanece suspendida (de 0 a 15; 0 = hasta la reactivación manual) |

</details>

El valor 2 cubre los dos reinicios pendientes (uno hacia el UEFI y otro después del flasheo); después, la protección se reactiva por sí sola y sella la clave frente a las nuevas mediciones. Independientemente de ello, compruebe antes que puede localizar la clave de recuperación, por ejemplo en la cuenta Microsoft en aka.ms/myrecoverykey o mediante `manage-bde -protectors -get C:`.

## Paso 5: Acceder al UEFI, incluso si F2 no responde

La vía clásica mediante F2 o Supr al encender suele fallar en sistemas modernos: con Fast Boot activado, el firmware inicializa el teclado USB solo después del POST, por lo que la pulsación no llega. Sin embargo, no depende de esa tecla: Windows puede dirigir el siguiente reinicio directamente a la configuración del UEFI. En una PowerShell con permisos de administrador:

```powershell
shutdown /r /fw /t 5
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `/r` | Reinicia en lugar de apagar |
| `/fw` | establece la variable de firmware que dirige el siguiente arranque directamente a la configuración del UEFI; solo junto con una opción de apagado como `/r`, requiere permisos de administrador |
| `/t 5` | Tiempo de espera en segundos hasta la ejecución |

</details>

Si el comando informa del error 203 ("The system could not find the environment option that was entered"), casi siempre faltan permisos de administrador: sin elevación, el proceso no puede establecer la variable de firmware necesaria, y el mensaje de error no indica esa causa. Una segunda vía posible sin variable de firmware pasa por el entorno de recuperación: `shutdown /r /o`, luego Solucionar problemas, Opciones avanzadas, Configuración de firmware UEFI.

## Paso 6: Flashear con Instant Flash

En el UEFI encontrará Instant Flash en el menú Tool. La herramienta lista todos los archivos ROM de la memoria USB; tras seleccionarlo, comprueba el archivo, lo flashea y se reinicia automáticamente. Durante esos pocos minutos se aplica la única regla estricta de todo el proceso: no interrumpa la alimentación eléctrica ni apague el equipo. Un flasheo interrumpido es el único paso de esta guía que realmente puede impedir que la placa arranque (y entonces requerirá la vía de rescate Flashback mencionada).

## Paso 7: Tareas posteriores, porque la actualización restablece todo

Después del flasheo, todos los ajustes de la BIOS vuelven a los valores de fábrica. Esto está previsto y ofrece una oportunidad de diagnóstico: la RAM funciona ahora sin perfil EXPO a la velocidad base JEDEC. Si ha flasheado por problemas de estabilidad, déjela así deliberadamente durante una o dos semanas. Si dejan de producirse los fallos, el perfil de memoria estaba implicado y podrá volver a probar EXPO específicamente con el nuevo firmware. La diferencia en el uso diario entre 4800 y 6000 MT/s apenas se percibe fuera de los benchmarks; un equipo estable vale cada punto de benchmark.

En cualquier caso, vale la pena revisar dos ajustes en el UEFI: quien haya tenido reinicios en reposo puede configurar, en Advanced, AMD CBS, la opción "Power Supply Idle Control" en "Typical Current Idle"; esto mitiga una incompatibilidad conocida de algunas fuentes de alimentación con los estados de reposo profundos de las CPU Ryzen. Y quien quiera volver a acceder a la configuración mediante F2 puede desactivar Fast Boot.

La comprobación del éxito de vuelta en Windows:

```powershell
Get-CimInstance Win32_BIOS | Select-Object SMBIOSBIOSVersion
Get-CimInstance Win32_PhysicalMemory |
  Select-Object PartNumber, ConfiguredClockSpeed
manage-bde -status C:
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Win32_BIOS` | Clase CIM con la versión del firmware; `SMBIOSBIOSVersion` debe mostrar ahora la nueva versión |
| `Win32_PhysicalMemory` | Clase CIM de los módulos de memoria; `ConfiguredClockSpeed` muestra la frecuencia realmente aplicada en MT/s |
| `-status` | manage-bde: muestra el estado de cifrado y protección del volumen |
| `C:` | Argumento posicional: el volumen que se debe comprobar |

</details>

La primera línea debe mostrar la nueva versión, la segunda la frecuencia de memoria esperada y BitLocker debe volver a indicar "Protección activada". Con ello, la actualización queda completada y documentada. Si se flasheó por problemas de estabilidad, solo la observación durante las semanas siguientes mostrará si se han resuelto; la forma más sencilla es revisar las nuevas entradas Kernel-Power 41 en el registro de eventos del sistema.

## Fuentes

1.  [ASRock A620I Lightning WiFi, descargas de BIOS](https://pg.asrock.com/mb/AMD/A620I%20Lightning%20WiFi/index.asp#BIOS): lista de versiones con registros de cambios, sumas de comprobación SHA256 y los métodos de actualización compatibles de la placa de ejemplo.

2.  [Microsoft Learn: Suspend-BitLocker](https://learn.microsoft.com/en-us/powershell/module/bitlocker/suspend-bitlocker): referencia para suspender la protección de BitLocker, incluido el parámetro RebootCount.

3.  [Microsoft Learn: Advanced troubleshooting for Event ID 41](https://learn.microsoft.com/en-us/troubleshoot/windows-client/performance/event-id-41-restart): clasificación de Kernel-Power 41 y del significado de BugcheckCode 0.

4.  [Microsoft Learn: comando shutdown](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/shutdown): documentación de los parámetros /fw y /o para reiniciar en el UEFI o en el entorno de recuperación, respectivamente.
