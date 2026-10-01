---
title: "Migrar Windows a una SSD más pequeña con Rescuezilla: de 1 TB a 512 GB"
navTitle: "Migrar a SSD más pequeña"
description: "Rescuezilla solo restaura una imagen en un disco que llegue al menos hasta el final de la última partición. Esta guía muestra, usando como ejemplo una NVMe de 1 TB, cómo reducir C: con GParted, mover la partición de recuperación, eliminar el indicador Dirty y restaurar la nueva imagen en una SSD de 512 GB."
date: "2026-09-29"
kategorie: "PC y hardware"
timeToRead: "9 min de lectura"
themen:
  - pc-hardware
produkte:
  - "windows-client"
protokolle:
  - "backup-dr"
  - "migration"
slug: "migrar-windows-a-una-ssd-mas-pequena-con-rescuezilla-de-1-tb-a-512-gb"
translationId: "article-6947213279383f27"
translationOf: windows-kleinere-ssd-rescuezilla
url: https://rafaelpfister.ch/es/blog/migrar-windows-a-una-ssd-mas-pequena-con-rescuezilla-de-1-tb-a-512-gb
translationSourceHash: ad4485736511552671c4e469ddaa672e9439cffb3e74d43bd8dd7333eaf662d3
translationModel: gpt-5.6-terra
translatedAt: 2026-09-30T09:55:31.137Z
translationReview: automatic
---

Rescuezilla es un sistema Live gratuito para realizar copias de seguridad y restaurar unidades completas, compatible con el formato de imagen de Clonezilla (presentación de la herramienta y procedimiento básico: [Rescuezilla: migrar Windows a una SSD nueva](/blog/rescuezilla-windows-migration)). Mientras el disco de destino tenga el mismo tamaño o sea mayor, basta con hacer una copia de seguridad y restaurarla. Si es más pequeño, la restauración se cancela aunque los datos ocupados cabrían en él. Este artículo documenta la migración de un sistema Windows 11 desde una NVMe de 1 TB (C: con 930 GB, de los cuales unos 350 GB están ocupados) a una NVMe de 512 GB, incluidos los mensajes de error que aparecen durante el proceso.

## Por qué falla la restauración en el disco más pequeño

Rescuezilla realiza una copia de cada partición por separado con Partclone y también guarda la tabla de particiones. Durante la restauración, escribe esta tabla sin modificar en el disco de destino. Rescuezilla no puede reducir una partición. Por ello, antes de empezar comprueba si la última partición se encuentra íntegramente en el disco de destino y, de no ser así, se cancela con este mensaje:

```text
The source partition table's final partition (/dev/nvme0n1p4:
1000203091968 bytes) must refer to a region completely within
the destination disk (512110190592 bytes).
```

Una instalación típica de Windows tiene cuatro particiones: la partición del sistema EFI (200 MB), la partición reservada de Microsoft (16 MB), C: y la partición de recuperación con Windows RE (aquí, 904 MiB). Windows coloca la partición de recuperación al final del disco. Por tanto, no basta con reducir C:: también debe desplazarse hacia delante la partición de recuperación, directamente detrás de C:.

El procedimiento que describe la wiki de Rescuezilla para este caso es: reducir el disco de origen con GParted, crear una nueva imagen y restaurar esa imagen. Una imagen creada previamente del disco sin modificar se conserva como copia de seguridad hasta que el nuevo sistema funcione.

## Paso 1: calcular el tamaño de destino

Una SSD vendida como de 512 GB tiene 512'110'190'592 bytes, es decir, 476,9 GiB (Rescuezilla también muestra este valor). De ahí hay que restar las particiones EFI, MSR y de recuperación, que suman algo más de 1,1 GiB. Para C: quedan por tanto apenas 475 GiB. Con algo de margen, **470 GiB = 481'280 MiB** es un valor de destino razonable. Los aproximadamente 6 GB que quedarán libres al final de la nueva SSD no son relevantes.

Los datos ocupados deben estar por debajo de este valor. Este comando, ejecutado en PowerShell con derechos de administrador, muestra hasta qué punto Windows podría reducir un volumen:

```powershell
$s = Get-PartitionSupportedSize -DriveLetter C
"{0:N1} GB" -f ($s.SizeMin / 1GB)
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-DriveLetter C` | Volumen cuyas posibles dimensiones se consultan |
| `SizeMin` | Tamaño mínimo al que Windows podría reducir por sí mismo el volumen |
| `"{0:N1} GB" -f` | Da formato al valor en bytes como gigabytes con un decimal |

</details>

Si `SizeMin` está claramente por encima de la cantidad de datos ocupados, hay archivos inamovibles (archivo de paginación, puntos de restauración, MFT) que impiden la reducción mediante las herramientas integradas de Windows. GParted mueve estos datos al reducir la partición, por lo que ese límite no se aplica allí.

## Paso 2: desactivar BitLocker y la hibernación

GParted no puede leer ni reducir un volumen cifrado con BitLocker, y Partclone no puede respaldarlo como NTFS. Compruebe el estado en PowerShell con derechos de administrador:

```powershell
manage-bde -status C:
```

Si no aparece `Fully Decrypted`, desactive BitLocker y espere a que finalice el descifrado:

```powershell
manage-bde -off C:
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-status` | Muestra el nivel de cifrado, el método y el estado de protección del volumen |
| `-off` | Descifra completamente el volumen y elimina los protectores de claves |
| `C:` | Argumento posicional: el volumen afectado |

</details>

El descifrado se ejecuta en segundo plano y, según la cantidad de datos, puede tardar una hora o más. En dispositivos administrados mediante Intune, una directiva puede volver a activar BitLocker poco después. Por ello, compruebe de nuevo el estado inmediatamente antes de iniciar GParted.

Además, la hibernación debe estar desactivada. Con el inicio rápido activo, Windows no se apaga por completo, sino que guarda el estado del kernel en `hiberfil.sys`. Entonces NTFS se considera aún en uso y GParted rechaza cualquier modificación. Un comando desactiva conjuntamente la hibernación y el inicio rápido, y elimina `hiberfil.sys`:

```powershell
powercfg /h off
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `/h off` | Forma abreviada de `/hibernate off`: desactiva la hibernación y el inicio rápido; se elimina `hiberfil.sys` |

</details>

## Paso 3: iniciar Rescuezilla

Windows puede dirigir el siguiente inicio directamente al menú de arranque con las opciones avanzadas. Allí seleccione «Usar un dispositivo» y la memoria USB con Rescuezilla:

```powershell
shutdown /r /o /t 0
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `/r` | Reinicia en lugar de apagar |
| `/o` | Inicia en las opciones de arranque avanzadas (Windows RE); solo junto con `/r` |
| `/t 0` | Tiempo de espera en segundos hasta la ejecución |

</details>

Como alternativa, al encender se abre el menú de arranque del firmware; en placas ASRock, con F11. Si no aparece «Usar un dispositivo», `shutdown /r /fw /t 0` lleva directamente a la configuración UEFI, donde puede seleccionarse la unidad de arranque para el siguiente inicio.

## Paso 4: reducir C: y mover la partición de recuperación

En el escritorio de Rescuezilla, inicie **Partition Editor** (GParted), no Rescuezilla.

1.  Seleccione el disco de origen arriba a la derecha. Preste atención al tamaño (aquí, 931.51 GiB) para no editar por error el disco externo de copia de seguridad o un segundo disco interno.

2.  Haga clic con el botón derecho en la partición NTFS con C: → **Resize/Move**. Introduzca el valor de destino en el campo **New size (MiB)**, aquí `481280`, y confirme con la tecla Tab. **Free space preceding** se mantiene sin cambios. Confirme con **Resize/Move**.

3.  Debajo de C: aparece ahora una línea `unallocated`, y debajo la partición de recuperación (NTFS, unos 900 MiB, indicadores `hidden, diag`). Haga clic con el botón derecho en la partición de recuperación → **Resize/Move**.

4.  Borre el valor del campo **Free space preceding (MiB)**, introduzca `0` y confirme con Tab. **New size** permanece igual; **Free space following** pasa a abarcar todo el espacio libre. Si tras pulsar Tab aparece `1` en lugar de `0`, se debe a la alineación a MiB enteros y es correcto. Confirme con **Resize/Move**.

5.  Confirme con OK una advertencia que indica que el desplazamiento podría impedir el arranque. Se refiere a particiones con cargador de arranque; Windows arranca desde la partición del sistema EFI, que permanece sin cambios.

6.  El orden ahora es: EFI, MSR, C:, partición de recuperación, `unallocated`. Hasta aquí, las operaciones solo están programadas. Solo al hacer clic en la marca verde (**Apply All Operations**) se ejecutan los cambios.

Windows RE vuelve a encontrar su partición después de desplazarla porque conserva su número de partición. `reagentc /info` sigue mostrando `Enabled` con la ruta `harddisk0\partition4\Recovery\WindowsRE`.

La guía de Rescuezilla reduce únicamente la última partición. Esto basta si C: es la última partición. En una instalación estándar de Windows 10 u 11, la partición de recuperación está detrás, por lo que moverla es obligatorio.

## Paso 5: eliminar el indicador Dirty

Tras la reducción, la siguiente copia de seguridad se cancela en C: con este mensaje:

```text
ntfsclone-ng.c: NTFS Volume '/dev/nvme0n1p3' is scheduled for a check
or it was shutdown uncleanly. Please boot Windows or fix it by fsck.
```

La causa es intencionada: `ntfsresize`, que GParted utiliza para NTFS, marca el sistema de archivos para comprobación antes de cada cambio de tamaño y deja esa marca activa. Según la página del manual, Windows debe ejecutar `chkdsk` en el siguiente inicio. Sin embargo, en el caso descrito, el indicador seguía activo tras un inicio normal de Windows y `Get-Volume C` informaba `Full Repair Needed`. Puede comprobar el estado en PowerShell con derechos de administrador:

```powershell
fsutil dirty query C:
```

Si el comando informa `Volume - C: is Dirty`, ejecute la comprobación en el siguiente inicio:

```powershell
chkdsk C: /f
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `dirty query C:` | fsutil: consulta si está establecido el bit Dirty del volumen |
| `C:` | chkdsk: el volumen que se debe comprobar |
| `/f` | Corrige los errores encontrados; en el volumen del sistema, programa la comprobación para el siguiente inicio |

</details>

Responda S a la pregunta de si se debe ejecutar la comprobación en el próximo reinicio y reinicie Windows normalmente. Después, `fsutil dirty query C:` debe mostrar el mensaje `is NOT Dirty`, y `Get-Volume C` debe volver a informar `Healthy`. Solo entonces inicie Rescuezilla de nuevo.

## Paso 6: crear y comprobar la nueva imagen

Ahora cree con Rescuezilla una nueva imagen del disco reducido. El tamaño de la imagen será prácticamente el mismo que el de la primera copia de seguridad (aquí, unos 214 GB), ya que Partclone solo guarda bloques ocupados. Rescuezilla guarda cada copia de seguridad en su propia carpeta con marca de tiempo, por ejemplo `2026-09-29-1343-img-rescuezilla`.

En la lista de selección de imágenes aparecerán todas las imágenes una junto a otra. La columna **Partitions** muestra los tamaños: la imagen anterior a la reducción contiene `ntfs 930.4GB`, la nueva `ntfs 470GB`. Un intento de copia de seguridad cancelado aparece con un triángulo de advertencia amarillo; contiene la tabla de particiones antigua y provoca durante la restauración exactamente el mensaje de error de la primera sección. Elimine esas carpetas para no seleccionarlas por error.

También puede comprobarse de antemano si una imagen cabe en el disco de destino consultando el archivo `<disk>-pt.parted` de la carpeta de la imagen. Contiene la tabla de particiones guardada en sectores de 512 bytes:

```text
Number  Start       End         Size        File system  Name
 1      2048s       411647s     409600s     fat32        EFI system partition
 2      411648s     444415s     32768s                   Microsoft reserved partition
 3      444416s     986105855s  985661440s  ntfs         Basic data partition
 4      986105856s  987957247s  1851392s    ntfs
```

El final de la última partición (987'957'247 + 1 sectores × 512 bytes = 505,8 GB) está por debajo de los 512,1 GB del disco de destino. La imagen cabe.

## Paso 7: restaurar en la nueva SSD

1.  En Rescuezilla, seleccione **Restore**, la unidad con las imágenes y, a continuación, la imagen **nueva** (sin triángulo de advertencia y con la partición C: reducida).

2.  Seleccione la nueva SSD como destino. Compruebe dos veces el modelo y el tamaño: la restauración sobrescribe por completo el disco de destino.

3.  Deje marcadas todas las particiones, así como **Overwrite partition table**, e inicie la restauración.

4.  A continuación, arranque desde la nueva SSD. Lo más sencillo es desconectar antes el disco antiguo; de lo contrario, seleccione la nueva SSD en el menú de arranque del firmware.

## Tareas posteriores

En la nueva unidad, vuelva a activar lo que se desactivó para la migración. `powercfg /h on` activa la hibernación. Active BitLocker mediante Configuración → Privacidad y seguridad → Cifrado de dispositivo o con `manage-bde -on C:`; después, compruebe con `manage-bde -protectors -get C:` que la clave de recuperación esté guardada. Si se desactivó Secure Boot en UEFI para iniciar Rescuezilla, vuelva a activarlo; mientras esté desactivado, BitLocker registra el evento 810 en cada inicio.

Puede eliminar la imagen del disco de 1 TB sin modificar cuando Windows arranque correctamente desde la nueva SSD y todos los datos estén completos.

## Fuentes

1.  [Wiki de Rescuezilla: restaurar en un disco más pequeño](https://github.com/rescuezilla/rescuezilla/wiki/HOWTO:-Restoring-to-a-smaller-disk.-Eg,-1000GB-HDD-to-500GB-SSD): procedimiento oficial con GParted y motivo de la limitación durante la restauración.

2.  [Rescuezilla](https://rescuezilla.com/): página del proyecto con la descarga del sistema Live.

3.  [Manual de GParted](https://gparted.org/display-doc.php?name=help-manual): uso de Resize/Move, alineación a MiB y ejecución de operaciones programadas.

4.  [ntfsresize(8), página del manual de Ubuntu](https://manpages.ubuntu.com/manpages/noble/man8/ntfsresize.8.html): herramienta detrás de la reducción NTFS en GParted; describe la marca de comprobación establecida intencionadamente.

5.  [Microsoft Learn: chkdsk](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/chkdsk): parámetro `/f` y programación de la comprobación en el volumen del sistema.

6.  [Microsoft Learn: fsutil dirty](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/fsutil-dirty): consulta y significado del bit Dirty.

7.  [Microsoft Learn: manage-bde status](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-status): consulta del estado de BitLocker.

8.  [Microsoft Learn: manage-bde off](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-off): descifrado de un volumen.

9.  [Microsoft Learn: opciones de línea de comandos de Powercfg](https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/powercfg-command-line-options): opción `/hibernate` y su efecto en el inicio rápido.

10.  [Microsoft Learn: Get-PartitionSupportedSize](https://learn.microsoft.com/en-us/powershell/module/storage/get-partitionsupportedsize): tamaño mínimo y máximo de partición desde la perspectiva de Windows.

11.  [Microsoft Learn: opciones de línea de comandos de REAgentC](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/reagentc-command-line-options): comprobar el estado y la ubicación de Windows RE.

12.  [Microsoft Learn: shutdown](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/shutdown): parámetros `/o` y `/fw` para iniciar las opciones avanzadas o UEFI, respectivamente.
