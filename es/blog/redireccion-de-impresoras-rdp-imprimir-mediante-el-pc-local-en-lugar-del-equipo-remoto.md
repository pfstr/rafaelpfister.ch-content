---
title: "Redirección de impresoras RDP: imprimir mediante el PC local en lugar del equipo remoto"
navTitle: "Redirección de impresoras RDP"
description: "Los trabajos de impresión de la sesión RDP deben llegar a la impresora situada junto al usuario, no al equipo remoto. La configuración se encuentra en tres lugares: en el cliente RDP, en el archivo .rdp y en el sistema de destino. También se explica cómo gestionar la advertencia «Editor desconocido» e incluye una lista de comprobación para la resolución de problemas."
date: "2026-08-24"
kategorie: "Cliente de Windows"
timeToRead: "5 min de lectura"
themen:
  - windows-client
produkte:
  - "windows-client"
protokolle:
  - "troubleshooting"
slug: "redireccion-de-impresoras-rdp-imprimir-mediante-el-pc-local-en-lugar-del-equipo-remoto"
translationId: "article-12521248666e9809"
draft: false
translationOf: rdp-druckerumleitung-lokale-drucker
url: https://rafaelpfister.ch/es/blog/redireccion-de-impresoras-rdp-imprimir-mediante-el-pc-local-en-lugar-del-equipo-remoto
translationSourceHash: a4f12f591e9dcb86f8ebdd3ff8af1008a130c3ec65424abe789ad4d6446eb4c2
translationModel: gpt-5.6-terra
translatedAt: 2026-08-25T04:14:39.126Z
translationReview: automatic
---

# Redirección de impresoras RDP: imprimir mediante el PC local en lugar del equipo remoto

Un usuario trabaja mediante Escritorio remoto en un equipo remoto y quiere imprimir en la impresora que tiene a su lado. Precisamente para ello existe la redirección de impresoras: el cliente RDP registra las impresoras locales en la sesión, el trabajo de impresión vuelve al cliente a través del canal RDP y se imprime allí. En el sistema de destino, la impresora aparece con el sufijo **(redireccionada, sesión n)**. Por lo general, no se necesitan controladores en el equipo remoto: Windows utiliza el controlador universal **Remote Desktop Easy Print**; el controlador de impresora adecuado solo debe estar instalado en el cliente local.

La redirección solo se aplica al establecer la conexión. Tras cada cambio en la configuración, la sesión debe desconectarse por completo y volver a conectarse; no basta con minimizar la ventana RDP.

## Lado del cliente: activar la redirección

La vía más rápida es la interfaz gráfica: iniciar `mstsc`, seleccionar **Mostrar opciones**, ir a la pestaña **Recursos locales**, marcar **Impresoras** y guardar la conexión en la pestaña **General**. Quien trabaje con archivos .rdp puede añadir la línea correspondiente directamente en el archivo; los archivos .rdp son archivos de texto sencillos y se pueden editar con cualquier editor:

```text
redirectprinters:i:1
```

Un obstáculo al usar accesos directos sin archivo .rdp: si la conexión se inicia con `mstsc /v:hostname`, se aplican los ajustes del archivo oculto `Default.rdp` en la carpeta Documentos del usuario. Si falta allí la línea `redirectprinters:i:1`, la impresora no aparecerá aunque aparentemente todo esté configurado correctamente. Este fragmento añade la línea de forma idempotente (la `0` existente pasa a ser `1` y se añade la línea si falta) y muestra el resultado para comprobarlo:

```powershell
$f = "$env:USERPROFILE\Documents\Default.rdp"
if (Test-Path $f) {
    $c = Get-Content $f
    if ($c -match 'redirectprinters') {
        $c -replace 'redirectprinters:i:0', 'redirectprinters:i:1' | Set-Content $f
    } else {
        Add-Content $f 'redirectprinters:i:1'
    }
} else {
    Set-Content $f 'redirectprinters:i:1'
}
Select-String -Path $f -Pattern 'redirectprinters'
```

Dos trampas más en el lado del cliente: en primer lugar, Windows guarda para cada equipo de destino, en `HKCU\Software\Microsoft\Terminal Server Client\LocalDevices`, qué redirecciones permitió por última vez el usuario en el diálogo de seguridad; esta selección guardada sobrescribe la configuración predeterminada del archivo .rdp. Al eliminar la clave se restablece el estado. En segundo lugar, el valor de registro `DisablePrinterRedirection` (DWORD, valor 1) bajo `HKLM\Software\Microsoft\Terminal Server Client` desactiva por completo la redirección de impresoras en el cliente; en dispositivos administrados conviene comprobarlo antes de comenzar a buscar el error en la sesión.

## Lado del servidor: permitir la redirección

En el sistema de destino, la directiva **No permitir la redirección de impresoras de cliente** (Configuración del equipo → Plantillas administrativas → Componentes de Windows → Servicios de Escritorio remoto → Host de sesión de Escritorio remoto → Redirección de impresoras) es decisiva. Si está *Habilitada*, no se crean impresoras de cliente, independientemente de lo que solicite el cliente. Se aplica el principio de la configuración más restrictiva: si uno de los dos lados bloquea la redirección, esta no tiene lugar.

Sin directiva de grupo, el mismo mecanismo se controla mediante el Registro: `fDisableCpm` bajo `HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp` (0 = redirección permitida, 1 = bloqueada). Además, el servicio **Cola de impresión** debe estar en ejecución en el sistema de destino; sin el spooler tampoco se crean impresoras redireccionadas.

En la misma categoría de GPO se encuentran dos opciones útiles: **Usar primero el controlador de impresora Easy Print de Escritorio remoto** (predeterminado y normalmente la opción correcta) y **Establecer la impresora predeterminada del cliente como impresora predeterminada de la sesión**.

## La advertencia «Editor desconocido»

Al abrir un archivo .rdp sin firmar que solicita redirecciones de dispositivos, el cliente muestra una advertencia de seguridad con casillas para cada recurso. Las casillas marcadas o desmarcadas allí solo se aplican a ese inicio de conexión, pero se guardan en la clave `LocalDevices` mencionada anteriormente y, por tanto, influyen de forma silenciosa en futuras conexiones. Quien se pregunte por qué la casilla de la impresora vuelve a faltar pese a que el archivo .rdp es correcto encontrará casi siempre la causa allí.

Hay tres formas de gestionar la advertencia, en orden creciente de esfuerzo. Primera: iniciar la conexión mediante `mstsc /v:hostname` en lugar de hacerlo con el archivo .rdp; sin archivo no hay comprobación del editor y la configuración procede de `Default.rdp`. Segunda: aprobar previamente las redirecciones para el equipo de destino mediante el Registro; así desaparece la parte de recursos del diálogo:

```powershell
$key = "HKCU:\Software\Microsoft\Terminal Server Client\LocalDevices"
New-Item -Path $key -Force | Out-Null
Set-ItemProperty -Path $key -Name "hostname-oder-ip" -Value 0xFFFFFFFF -Type DWord
```

Tercera, la vía adecuada para archivos .rdp distribuidos en una empresa: firmar el archivo con `rdpsign.exe` y un certificado, y registrar la huella digital del certificado mediante GPO como editor de confianza. Para puestos de trabajo individuales, el esfuerzo rara vez merece la pena; para archivos de conexión distribuidos centralmente, es la solución correcta.

## Lista de comprobación para la resolución de problemas

Si la impresora no aparece en la sesión, comprobar lo siguiente en este orden:

1. **¿Se ha vuelto a conectar?** La redirección solo se aplica al establecer la conexión, no en una sesión existente.
2. **¿Es el archivo correcto?** En accesos directos, comprobar qué archivo .rdp se abre realmente; con `mstsc /v:` cuenta `Default.rdp`.
3. **¿Selección guardada?** Comprobar o eliminar la clave `LocalDevices` en el cliente.
4. **¿Bloqueo en el cliente?** `DisablePrinterRedirection` bajo `HKLM\Software\Microsoft\Terminal Server Client` no debe tener el valor 1.
5. **¿Bloqueo en el servidor?** Comprobar la GPO «No permitir la redirección de impresoras de cliente» o bien `fDisableCpm` en el sistema de destino.
6. **¿Spooler?** El servicio Cola de impresión debe estar en ejecución en el sistema de destino.
7. **Comprobación en la sesión:** `Get-Printer | Where-Object DriverName -eq "Remote Desktop Easy Print"` muestra las impresoras redireccionadas junto con su ID de sesión.

## Fuentes

1.  [Supported RDP properties](https://learn.microsoft.com/en-us/azure/virtual-desktop/rdp-properties): referencia de todas las propiedades de .rdp, incluida redirectprinters con valores y valor predeterminado.

2.  [Configure printer redirection over the Remote Desktop Protocol](https://learn.microsoft.com/en-us/azure/virtual-desktop/redirection-configure-printers): Easy Print, configuración mediante GPO e Intune, DisablePrinterRedirection y la prueba con Get-Printer.

3.  [rdpsign](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/rdpsign): referencia de comandos para firmar archivos .rdp mediante la huella digital de un certificado.
