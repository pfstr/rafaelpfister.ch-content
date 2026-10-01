---
title: "Rescuezilla: migrar Windows a una nueva SSD, con presentación de la herramienta"
navTitle: "Migración con Rescuezilla"
description: "Rescuezilla es una solución de creación de imágenes gratuita y compatible con Clonezilla, con interfaz gráfica. El artículo presenta la herramienta y muestra la migración completa de una instalación de Windows a una nueva SSD: preparación en Windows, memoria USB de arranque, copia de seguridad, comprobación, restauración y tareas posteriores."
date: "2026-09-29"
kategorie: "PC y hardware"
timeToRead: "10 min de lectura"
themen:
  - pc-hardware
produkte:
  - "windows-client"
protokolle:
  - "backup-dr"
  - "migration"
slug: "rescuezilla-migrar-windows-a-una-nueva-ssd-con-presentacion-de-la-herramienta"
translationId: "article-d9438c5774d0167c"
translationOf: rescuezilla-windows-migration
url: https://rafaelpfister.ch/es/blog/rescuezilla-migrar-windows-a-una-nueva-ssd-con-presentacion-de-la-herramienta
translationSourceHash: 33320c51c0a77d69bc5804e2c7e69d14c63731a23df6ef55969b37b043a49cd5
translationModel: gpt-5.6-terra
translatedAt: 2026-09-30T09:51:27.905Z
translationReview: automatic
---

Quien quiera trasladar una instalación de Windows a una nueva SSD o a un equipo nuevo sin reinstalar Windows necesita una herramienta que haga una copia de seguridad y restaure todo el disco con todas sus particiones. Rescuezilla es una de esas herramientas: gratuita, de código abierto y con una interfaz gráfica que puede utilizarse incluso sin conocimientos de Linux. Este artículo presenta la herramienta y describe la migración a una SSD del mismo tamaño o mayor. Para una SSD de destino más pequeña hay una guía específica: [Migrar Windows a una SSD más pequeña con Rescuezilla](/blog/windows-kleinere-ssd-rescuezilla).

## Qué es Rescuezilla

Rescuezilla es un sistema Live basado en Ubuntu que se inicia desde una memoria USB. Funciona independientemente del sistema operativo instalado y, por ello, también realiza copias de seguridad de volúmenes que Windows bloquea mientras está en ejecución. El proyecto surgió en 2019 como un fork de Redo Backup and Recovery, que en aquel momento llevaba siete años sin mantenimiento. Desde la versión 2.0 (2020), Rescuezilla escribe imágenes en formato Clonezilla: una copia de seguridad creada con Rescuezilla puede restaurarse con Clonezilla y viceversa. La licencia es GPL-3.0. La versión actual es la 2.6.2 de mayo de 2026, basada en Ubuntu 26.04 LTS con Partclone 0.3.47.

El trabajo propiamente dicho lo realiza Partclone. Reconoce los sistemas de archivos habituales (NTFS, FAT, ext4 y otros) y lee únicamente los bloques ocupados. Por ello, una partición de 1 TB con 350 GB de datos produce una imagen de unos 215 GB (comprimida con gzip), no de 1 TB. Las particiones sin un sistema de archivos reconocido, como la partición reservada de Microsoft, Rescuezilla las guarda bloque a bloque con `dd`; estos archivos tienen en la imagen la extensión `.dd-ptcl-img`.

<details class="options-details">
<summary>Funciones principales</summary>

| Función | Finalidad |
|---|---|
| Backup | guarda las particiones seleccionadas de un disco, incluida la tabla de particiones, como imagen en una unidad local o recurso compartido de red (SMB, SSH) |
| Restore | restaura una imagen en un disco, opcionalmente sobrescribiendo la tabla de particiones |
| Verify Image | comprueba que una imagen existente esté completa y se pueda leer |
| Clone | copia un disco directamente a un segundo disco, sin almacenamiento intermedio |
| Image Explorer (beta) | monta una imagen en modo de solo lectura para extraer archivos individuales |
| Imágenes de VM | además de imágenes de Clonezilla, lee VDI, VMDK, VHDX, QCOW2 e imágenes Raw |
| Herramientas adicionales | GParted, gestor de archivos, navegador web y herramientas para recuperar archivos eliminados en el escritorio Live |
| CLI | línea de comandos experimental (desde la 2.5) para Backup, Verify, Restore y Clone |

</details>

Una limitación es importante para las migraciones: Rescuezilla no reduce particiones. El disco de destino debe llegar, como mínimo, hasta el final de la última partición guardada. Si es más pequeño, se requiere un trabajo previo con GParted, descrito en la guía enlazada anteriormente.

## Imagen o clon

Rescuezilla ofrece dos opciones para una migración. Con el **clonado**, los discos de origen y destino están conectados al mismo tiempo y Rescuezilla copia directamente. Esto ahorra tiempo y un tercer soporte de almacenamiento, pero exige que ambos discos estén conectados simultáneamente al equipo o a un adaptador. Con la **creación de imágenes**, primero se genera una imagen en una unidad externa, que después se restaura en el nuevo disco. Tarda más, pero tiene una ventaja: la imagen permanece como copia de seguridad completa del estado anterior, incluso si algo sale mal durante la restauración. Por ello, para una migración la imagen es la opción más segura, y es la variante que describe este artículo.

## Requisitos

1.  **Memoria USB** para Rescuezilla. Su contenido se borrará al escribir la imagen de arranque.

2.  **Unidad externa** para la imagen, con espacio libre aproximadamente igual al tamaño de los datos ocupados. Tanto exFAT como NTFS funcionan.

3.  **Disco de destino**, al menos tan grande como el disco de origen. Si es más pequeño, siga primero la guía para SSD más pequeñas.

4.  **Clave de recuperación de BitLocker**, si C: está cifrado. Disponible en aka.ms/myrecoverykey o en el portal Entra para dispositivos administrados.

## Paso 1: Preparar Windows

Antes de la copia de seguridad, compruebe tres puntos en una PowerShell con derechos de administrador.

**BitLocker.** Partclone no puede leer un volumen BitLocker como NTFS. Entonces Rescuezilla lo guarda bloque a bloque, como cualquier partición sin sistema de archivos reconocido, y la imagen tendrá el tamaño de toda la partición. Para una migración, es recomendable desactivar BitLocker antes y volver a activarlo después del traslado:

```powershell
manage-bde -status C:
manage-bde -off C:
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-status` | muestra el grado de cifrado y el estado de protección del volumen |
| `-off` | descifra el volumen por completo; continúa ejecutándose en segundo plano |
| `C:` | argumento posicional: el volumen afectado |

</details>

Espere hasta que `manage-bde -status C:` muestre el valor `Fully Decrypted`. En dispositivos administrados mediante Intune, una directiva puede volver a activar BitLocker; por ello, compruebe de nuevo el estado justo antes de reiniciar.

**Hibernación e inicio rápido.** Con el inicio rápido activado, Windows no se apaga por completo y NTFS se considera todavía en uso. En ese caso, Partclone se interrumpe. Un comando desactiva ambos:

```powershell
powercfg /h off
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `/h off` | forma abreviada de `/hibernate off`: desactiva la hibernación y el inicio rápido, se elimina `hiberfil.sys` |

</details>

**Estado del sistema de archivos.** Si está establecido el bit de datos pendientes, Partclone se interrumpe con el mensaje de que el volumen está «scheduled for a check or it was shutdown uncleanly». Compruébelo previamente:

```powershell
fsutil dirty query C:
```

Si el comando informa `is Dirty`, programe una comprobación para el siguiente inicio con `chkdsk C: /f`, reinicie y vuelva a comprobarlo.

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `dirty query C:` | fsutil: consulta si está establecido el bit de datos pendientes del volumen |
| `/f` | chkdsk: corrige errores; en el volumen del sistema, programa la comprobación para el siguiente inicio |

</details>

## Paso 2: Crear e iniciar la memoria USB de arranque

Descargue la imagen ISO desde la página de lanzamientos de GitHub del proyecto. La variante estándar lleva el nombre en clave de Ubuntu en el nombre de archivo; en la 2.6.2 es `rescuezilla-2.6.2-64bit.resolute.iso`. Escríbala en la memoria USB con un programa de escritura de imágenes como balenaEtcher; la página del proyecto recomienda este programa para Windows, macOS y Linux.

El propio Windows inicia el arranque desde la memoria USB:

```powershell
shutdown /r /o /t 0
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `/r` | reinicia en lugar de apagar |
| `/o` | inicia las opciones de arranque avanzadas; solo junto con `/r` |
| `/t 0` | tiempo de espera en segundos hasta la ejecución |

</details>

En las opciones de arranque avanzadas, seleccione «Usar un dispositivo» y la memoria USB. Como alternativa, al encender se abre el menú de arranque del firmware (según el fabricante, F8, F11 o F12). Rescuezilla se inicia con Secure Boot activado; no es necesario desactivarlo. Si la pantalla permanece negra después de seleccionar el idioma, use la entrada «Graphical Fallback Mode» en el menú de arranque de la memoria USB.

## Paso 3: Crear la copia de seguridad

Rescuezilla se inicia automáticamente en el escritorio. El asistente guía por la copia de seguridad en pasos numerados:

1.  Seleccione **Backup**.

2.  *Step 1: Select Drive To Backup*: seleccione el disco del sistema. El modelo y el tamaño ayudan a distinguirlos; los nombres de dispositivo (`nvme0n1`, `sda`) pueden variar entre dos arranques.

3.  *Step 2: Select Partitions to Save*: deje marcadas todas las particiones. Para una restauración arrancable, necesita la partición de sistema EFI, la partición reservada de Microsoft, C: y la partición de recuperación.

4.  *Step 3: Select Destination Drive*: la unidad externa (Local) o un recurso compartido de red (Network).

5.  *Step 4: Select Destination Folder*: la carpeta de destino en la unidad. Rescuezilla crea en ella una subcarpeta con marca de tiempo.

6.  *Step 5: Name Your Backup*: opcionalmente, una descripción; aparecerá después en la lista de imágenes.

7.  *Step 6: Customize Compression Settings*: la configuración predeterminada gzip es adecuada en la mayoría de los casos.

8.  *Step 7: Confirm Backup Configuration*: revise los datos e inicie el proceso.

Para 350 GB de datos en una SSD NVMe, la copia de seguridad en una SSD USB externa tardó unos 20 minutos. Al final, Rescuezilla informa para cada partición si se guardó correctamente. Si aparece un error en una partición, la imagen está incompleta, aunque la carpeta exista.

## Paso 4: Comprobar la imagen

Compruebe la imagen antes de restaurarla. **Verify Image** en el menú principal lee todas las partes y verifica que estén completas y se puedan leer. En la lista de imágenes, la columna **Partitions** muestra las particiones guardadas con su tamaño; un triángulo amarillo de advertencia marca imágenes con particiones ausentes.

La carpeta de la imagen también puede examinarse en Windows. Los archivos más importantes:

| Archivo | Contenido |
|---|---|
| `nvme0n1-pt.parted` | tabla de particiones en formato de texto, con el inicio y el final de cada partición en sectores |
| `nvme0n1-gpt-1st`, `nvme0n1-gpt-2nd` | copias binarias de la GPT primaria y de respaldo |
| `nvme0n1p3.ntfs-ptcl-img.gz.aa`, `.ab`, … | imagen Partclone de C:, comprimida con gzip y dividida en partes de 4 GB |
| `nvme0n1p2.dd-ptcl-img.gz.aa` | copia bloque a bloque de una partición sin sistema de archivos reconocido |
| `clonezilla-img` | registro de la copia de seguridad con el resultado de cada partición |
| `Info-*.txt` | información de hardware y SMART del sistema de origen en el momento de la copia de seguridad |

A partir de `nvme0n1-pt.parted` se puede determinar si la imagen cabe en el disco de destino: el sector final de la última partición más uno, multiplicado por 512 bytes, debe ser menor que el tamaño del disco de destino en bytes.

## Paso 5: Restaurar en la nueva SSD

Instale la nueva SSD e inicie Rescuezilla de nuevo.

1.  Seleccione **Restore**.

2.  *Step 1: Select Image Location*: la unidad externa con la imagen.

3.  *Step 2: Select Backup Image*: seleccione la imagen según la fecha y los tamaños de partición.

4.  *Step 3: Select Drive To Restore*: la nueva SSD. Compruebe dos veces el modelo y el tamaño: se sobrescribirán todos los datos de esta unidad.

5.  *Step 4: Select Partitions to Restore*: deje marcadas todas las particiones, así como **Overwrite partition table**.

6.  *Step 5: Confirm Restore Configuration*: compruebe e inicie el proceso.

La restauración tarda aproximadamente lo mismo que la copia de seguridad. Después, apague el equipo y retire la memoria USB.

## Paso 6: Iniciar desde la nueva unidad

Lo más sencillo es desconectar el disco antiguo antes del primer inicio. Si ambos discos están conectados, tendrán los mismos ID de partición y volumen, y el firmware podría elegir el antiguo para arrancar. Si va a seguir utilizando el disco antiguo, formatéelo solo después de comprobar el nuevo sistema.

Si la nueva SSD es mayor que la antigua, el espacio adicional queda como área sin asignar al final. Sin embargo, no se puede ampliar C: directamente en Administración de discos porque la partición de recuperación está entre C: y el espacio libre. Con GParted en la memoria USB de Rescuezilla, primero mueva la partición de recuperación al final (Resize/Move, **Free space following** a 0) y después amplíe C: a todo el espacio libre. Windows RE volverá a encontrar su partición porque conserva su número de partición; `reagentc /info` muestra el estado.

Si con la migración también cambia de equipo, Windows suele iniciarse sin ajustes e instala después los controladores que faltan. La activación se realiza mediante la licencia digital; tras cambiar la placa base, quizá solo después de iniciar sesión con la cuenta de Microsoft mediante el solucionador de problemas de activación.

## Tareas posteriores

En la nueva unidad, vuelva a activar lo que se desactivó para la migración. `powercfg /h on` activa la hibernación. Active BitLocker mediante Configuración → Privacidad y seguridad → Cifrado de dispositivo o con `manage-bde -on C:`; después, compruebe con `manage-bde -protectors -get C:` que existe una clave de recuperación y que está guardada.

Conserve la imagen en la unidad externa hasta que Windows haya funcionado varios días sin anomalías en la nueva unidad y haya comprobado todos los datos. Después puede eliminarla o archivarla como copia de seguridad del estado de entrega.

## Fuentes

1.  [Rescuezilla](https://rescuezilla.com/): página del proyecto con resumen de funciones, descarga y preguntas frecuentes.

2.  [Rescuezilla en GitHub](https://github.com/rescuezilla/rescuezilla): código fuente, historia del proyecto y lista de funciones en el README.

3.  [Lanzamientos de Rescuezilla](https://github.com/rescuezilla/rescuezilla/releases/latest): versión actual con notas de lanzamiento, lista de compatibilidad y variantes ISO.

4.  [Registro de cambios de Rescuezilla](https://raw.githubusercontent.com/rescuezilla/rescuezilla/master/CHANGELOG.md): introducción de CLI, Verify Image y Clone, así como la actualización del shim de Secure Boot.

5.  [Wiki de Rescuezilla: Restoring to a smaller disk](https://github.com/rescuezilla/rescuezilla/wiki/HOWTO:-Restoring-to-a-smaller-disk.-Eg,-1000GB-HDD-to-500GB-SSD): procedimiento oficial para discos de destino más pequeños que el original.

6.  [Partclone](https://partclone.org/): herramienta que Rescuezilla y Clonezilla usan para copias de seguridad conscientes del sistema de archivos.

7.  [Manual de GParted](https://gparted.org/display-doc.php?name=help-manual): mover y ampliar particiones.

8.  [Microsoft Learn: manage-bde status](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-status): consulta del estado de BitLocker.

9.  [Microsoft Learn: manage-bde off](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-off): descifrado de un volumen.

10.  [Microsoft Learn: Powercfg command-line options](https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/powercfg-command-line-options): opción `/hibernate` y su efecto en el inicio rápido.

11.  [Microsoft Learn: fsutil dirty](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/fsutil-dirty): consulta del bit de datos pendientes.

12.  [Microsoft Learn: chkdsk](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/chkdsk): comprobación y reparación de volúmenes NTFS.

13.  [Microsoft Learn: shutdown](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/shutdown): parámetro `/o` para iniciar las opciones de arranque avanzadas.

14.  [Microsoft Support: Reactivating Windows after a hardware change](https://support.microsoft.com/en-us/windows/reactivating-windows-after-a-hardware-change-2c0e962a-f04c-145b-6ead-fb3fc72b6665): activación mediante la licencia digital tras un cambio de placa base.
