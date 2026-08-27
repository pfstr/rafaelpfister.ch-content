---
title: "Realizar una actualización de BIOS de forma segura: guía con el ejemplo de ASRock AM5, incluida la preparación de BitLocker"
navTitle: "Actualización de BIOS"
description: "El proceso completo de una actualización de BIOS con el ejemplo de una placa ASRock AM5: determinar la versión, verificar la descarga mediante hash, pausar BitLocker correctamente, arrancar en UEFI (incluso si F2 no funciona), actualizar mediante Instant Flash y configurar adecuadamente los ajustes después de la actualización."
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
url: https://rafaelpfister.ch/es/blog/realizar-una-actualizacion-de-bios-de-forma-segura-guia-con-el-ejemplo-de-asrock-am5-incluida
translationSourceHash: 60fff28a10b0f91f3d59996b00afe614f2230b9831514dcf01a1f496b99f4fbd
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:26:29.013Z
translationReview: automatic
---

Una actualización de BIOS forma parte de las tareas de mantenimiento que se realizan pocas veces y que, por ello, vuelven a plantear dudas cada vez: ¿qué versión es la correcta, cómo se lleva de forma segura a la placa y qué hay que tener en cuenta antes y después? Esta guía documenta el proceso completo con el ejemplo de una ASRock A620I Lightning WiFi (socket AM5) mediante el método propio del fabricante, Instant Flash. Los pasos son aplicables a cualquier placa base moderna; los aspectos críticos (BitLocker, Fast Boot y restablecimiento de ajustes) son independientes del fabricante.

## Cuándo conviene actualizar la BIOS

Hay tres motivos que justifican la intervención. Primero, las correcciones de seguridad: las vulnerabilidades de firmware solo pueden cerrarse mediante una actualización de BIOS, y los registros de cambios de los fabricantes suelen mencionarlas de forma escueta. Segundo, la compatibilidad: la compatibilidad con nuevas generaciones de CPU y una mayor compatibilidad de memoria llegan exclusivamente mediante nuevas versiones de firmware; en AM5, a través del firmware de referencia AGESA de AMD, que los fabricantes de placas integran en sus versiones de BIOS. Tercero, la estabilidad: si un sistema se reinicia espontáneamente y el registro de eventos solo registra Kernel-Power 41 con `BugcheckCode=0`, el fallo se ha producido a nivel de hardware o firmware, sin intervención de Windows; las causas típicas son voltajes inestables y el entrenamiento de memoria, precisamente la capa que mantienen las versiones de AGESA. Entradas como "Improve memory compatibility and system stability" o un manejo revisado de EXPO en los registros de cambios indican que una actualización aborda estos problemas. En cambio, si un sistema funciona de forma estable y no está afectado por las vulnerabilidades corregidas, esperar es legítimo; actualizar la BIOS sin motivo es un riesgo sin beneficio.

## Paso 1: Determinar el estado actual

Antes de descargar nada, necesita dos datos: el modelo exacto de la placa y la versión de BIOS instalada. PowerShell proporciona ambos sin reiniciar:

```powershell
Get-CimInstance Win32_ComputerSystem | Select-Object Manufacturer, Model
Get-CimInstance Win32_BIOS | Select-Object SMBIOSBIOSVersion, ReleaseDate
```

Anote la versión. La necesitará más adelante para comprobar el resultado, y al leer los registros de cambios querrá saber qué versiones se está saltando.

## Paso 2: Descargar la BIOS y verificar la suma de comprobación

Descargue la BIOS exclusivamente desde la página del producto del fabricante, nunca desde portales de terceros. ASRock publica la suma de comprobación SHA256 para cada versión; tras la descarga, compárela antes de que el archivo se acerque siquiera a una memoria USB:

```powershell
Get-FileHash .\A620I_Lightning_WiFi_4.43.zip -Algorithm SHA256
```

Si el valor no coincide con el indicado por el fabricante, la descarga está dañada o ha sido manipulada: no actualice. Tras descomprimirla, queda un único archivo ROM, en el ejemplo `A62IRW_4.43.ROM` de 32 MB.

## Paso 3: Preparar la memoria USB

El mecanismo de actualización del UEFI (en ASRock, "Instant Flash"; en otros fabricantes, Q-Flash, EZ Flash o M-Flash) lee la memoria directamente desde el firmware. Esto significa que solo FAT32 se reconoce de forma fiable; NTFS y exFAT no. Casi cualquier memoria USB comprada ya viene en FAT32; puede comprobarlo así:

```powershell
Get-CimInstance Win32_LogicalDisk -Filter "DriveType=2" |
  Select-Object DeviceID, FileSystem, VolumeName
```

Copie el archivo ROM en el directorio raíz de la memoria USB. Solo es necesario reformatearla si el sistema de archivos no es adecuado. El tamaño de la memoria no es relevante: el archivo es menor que cualquier capacidad habitual hoy en día.

Una nota sobre la elección del método: muchas placas también ofrecen un botón BIOS Flashback, que actualiza sin CPU y sin un sistema funcional. Es la vía de rescate para una placa que ya no arranca. Para un sistema en funcionamiento, Instant Flash en el UEFI es la opción correcta y más sencilla. Las herramientas de actualización basadas en Windows no son necesarias ni recomendables en plataformas actuales.

## Paso 4: Pausar BitLocker o se solicitará la clave

Este es el punto que falta en muchas guías. Si la unidad del sistema está cifrada con BitLocker (en Windows 11, a menudo se activa automáticamente al usar una cuenta Microsoft), BitLocker vincula la clave a las mediciones del TPM. Una actualización de BIOS modifica estas mediciones y, en el siguiente arranque, Windows solicita la clave de recuperación de 48 dígitos. Quien no la tenga a mano se encontrará con un sistema inaccesible.

BitLocker incluye su propio mecanismo para este escenario. En una PowerShell con derechos de administrador:

```powershell
Suspend-BitLocker -MountPoint C: -RebootCount 2
```

El parámetro `RebootCount 2` cubre los dos reinicios pendientes (uno al UEFI y otro después de la actualización); después, la protección se reactiva por sí sola y sella la clave con las nuevas mediciones. Independientemente de ello, compruebe previamente que la clave de recuperación se puede localizar, por ejemplo, en la cuenta Microsoft en aka.ms/myrecoverykey o mediante `manage-bde -protectors -get C:`.

## Paso 5: Entrar en el UEFI, incluso si F2 no responde

La vía clásica mediante F2 o Supr al encender suele fallar en sistemas modernos: con Fast Boot activado, el firmware inicializa el teclado USB solo después del POST, por lo que la pulsación no llega. Sin embargo, no depende de la tecla: Windows puede dirigir el siguiente reinicio directamente a la configuración del UEFI. En una PowerShell con derechos de administrador:

```powershell
shutdown /r /fw /t 5
```

Si el comando informa del error 203 ("The system could not find the environment option that was entered"), casi siempre faltan derechos de administrador: sin elevación, el proceso no puede establecer la variable de firmware necesaria, y el mensaje de error no indica esta causa. Una segunda vía sin variable de firmware pasa por el entorno de recuperación: `shutdown /r /o`, después Solucionar problemas, Opciones avanzadas y Configuración de firmware UEFI.

## Paso 6: Actualizar con Instant Flash

En el UEFI encontrará Instant Flash en el menú Tool. La herramienta enumera todos los archivos ROM de la memoria USB; tras seleccionarlo, comprueba el archivo, actualiza y reinicia automáticamente. Durante los pocos minutos que dura, se aplica la única regla estricta de todo el proceso: no interrumpa la alimentación ni apague el equipo. Una actualización interrumpida es el único paso de esta guía que realmente puede dejar la placa sin posibilidad de arrancar (y entonces requerirá la vía de rescate Flashback mencionada).

## Paso 7: Ajustes posteriores, porque la actualización lo restablece todo

Tras la actualización, todos los ajustes de BIOS quedan en los valores de fábrica. Esto es intencionado y ofrece una oportunidad de diagnóstico: ahora la RAM funciona sin perfil EXPO a la velocidad base JEDEC. Si actualizó por problemas de estabilidad, déjela deliberadamente así durante una o dos semanas. Si los fallos dejan de producirse, el perfil de memoria estaba implicado y podrá volver a probar EXPO específicamente con el nuevo firmware. La diferencia cotidiana entre 4800 y 6000 MT/s apenas se percibe fuera de los benchmarks; un equipo estable vale cada punto de benchmark.

De todos modos, merece la pena revisar dos ajustes en el UEFI: quien haya tenido reinicios en reposo puede establecer, en Advanced, AMD CBS, la opción "Power Supply Idle Control" en "Typical Current Idle"; esto atenúa una incompatibilidad conocida de algunas fuentes de alimentación con los estados de inactividad profundos de las CPU Ryzen. Y quien quiera volver a entrar en la configuración mediante F2 en el futuro puede desactivar Fast Boot.

La comprobación del resultado de vuelta en Windows:

```powershell
Get-CimInstance Win32_BIOS | Select-Object SMBIOSBIOSVersion
Get-CimInstance Win32_PhysicalMemory |
  Select-Object PartNumber, ConfiguredClockSpeed
manage-bde -status C:
```

La primera línea debe mostrar la nueva versión, la segunda la frecuencia de memoria esperada y BitLocker debe volver a indicar "Protección activada". Con ello, la actualización queda completada y documentada. Si se actualizó por problemas de estabilidad, solo la observación durante las semanas siguientes mostrará si se han resuelto; lo más sencillo es revisar las nuevas entradas Kernel-Power 41 en el registro de eventos del sistema.

## Fuentes

1.  [ASRock A620I Lightning WiFi, descargas de BIOS](https://pg.asrock.com/mb/AMD/A620I%20Lightning%20WiFi/index.asp#BIOS): lista de versiones con registros de cambios, sumas de comprobación SHA256 y los métodos de actualización compatibles de la placa del ejemplo.

2.  [Microsoft Learn: Suspend-BitLocker](https://learn.microsoft.com/en-us/powershell/module/bitlocker/suspend-bitlocker): referencia para pausar la protección de BitLocker, incluido el parámetro RebootCount.

3.  [Microsoft Learn: Advanced troubleshooting for Event ID 41](https://learn.microsoft.com/en-us/troubleshoot/windows-client/performance/event-id-41-restart): explicación de Kernel-Power 41 y del significado de BugcheckCode 0.

4.  [Microsoft Learn: comando shutdown](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/shutdown): documentación de los parámetros /fw y /o para reiniciar en el UEFI o en el entorno de recuperación, respectivamente.
