---
title: "Redirección de impresoras RDP: imprimir a través del PC local en lugar del equipo remoto"
navTitle: "Redirección de impresoras RDP"
description: "Los trabajos de impresión de la sesión RDP deben llegar a la impresora junto al usuario, no al equipo remoto. La configuración se encuentra en tres lugares: en el cliente RDP, en el archivo .rdp y en el sistema de destino. También se explica cómo tratar la advertencia «Editor desconocido» e incluye una lista de comprobación para solucionar problemas."
date: "2026-08-24"
kategorie: "Cliente Windows"
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
translationSourceHash: 2cb3845d308ebda202c6c33b20cbe791ddfbeeb584341876bdc340e0febf65b5
translationModel: gpt-5.6-terra
translatedAt: 2026-08-30T09:29:43.475Z
translationReview: automatic
url: https://rafaelpfister.ch/es/blog/redireccion-de-impresoras-rdp-imprimir-mediante-el-pc-local-en-lugar-del-equipo-remoto
---

# Redirección de impresoras RDP: imprimir a través del PC local en lugar del equipo remoto

Un usuario trabaja mediante Escritorio remoto en un equipo remoto y quiere imprimir en la impresora que tiene a su lado. Para eso existe la redirección de impresoras: el cliente RDP registra las impresoras locales en la sesión, el trabajo de impresión vuelve al cliente a través del canal RDP y se imprime allí. En el sistema de destino, la impresora aparece con el añadido **(redireccionada, sesión n)**. Por lo general, no se necesitan controladores en el equipo remoto: Windows utiliza el controlador universal **Remote Desktop Easy Print**; el controlador de impresora correspondiente solo debe estar instalado en el cliente local.

La redirección solo se aplica al establecer la conexión. Después de cada cambio en la configuración, la sesión debe desconectarse por completo y volver a conectarse; no basta con minimizar la ventana RDP.

## Lado del cliente: activar la redirección

La forma más sencilla de activar la redirección de impresoras es mediante la interfaz gráfica: iniciar `mstsc`, seleccionar **Mostrar opciones**, abrir la pestaña **Recursos locales**, marcar **Impresoras** y guardar la conexión en la pestaña **General**. Quienes trabajen con archivos .rdp pueden ajustar la línea directamente en el archivo; los archivos .rdp son simples archivos de texto y se pueden editar con cualquier editor:

```text
redirectprinters:i:1
```

Una particularidad de los accesos directos sin archivo .rdp: si la conexión se inicia con `mstsc /v:hostname`, se aplican los ajustes del archivo oculto `Default.rdp` en la carpeta Documentos del usuario. Si falta allí la línea `redirectprinters:i:1`, la impresora no aparece, aunque aparentemente todo esté configurado correctamente. Este fragmento añade la línea de forma idempotente (la `0` existente pasa a ser `1`, y se añade la línea si falta) y muestra el resultado para comprobarlo:

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

Hay otras dos fuentes de error en el lado del cliente: primero, Windows guarda para cada equipo de destino, en `HKCU\Software\Microsoft\Terminal Server Client\LocalDevices`, qué redirecciones permitió por última vez el usuario en el diálogo de seguridad; esta selección guardada sobrescribe la configuración predeterminada del archivo .rdp. Eliminar la clave restablece el estado. Segundo, el valor del Registro `DisablePrinterRedirection` (DWORD, valor 1) en `HKLM\Software\Microsoft\Terminal Server Client` desactiva por completo la redirección de impresoras en el cliente; en dispositivos administrados conviene revisarlo antes de empezar a buscar el problema en la sesión.

## Lado del servidor: permitir la redirección

En el sistema de destino, la directiva **No permitir la redirección de impresoras del cliente** (Configuración del equipo → Plantillas administrativas → Componentes de Windows → Servicios de Escritorio remoto → Host de sesión de Escritorio remoto → Redirección de impresoras) determina el resultado. Si está *Habilitada*, no se crean impresoras de cliente, independientemente de lo que solicite el cliente. Se aplica el principio de la configuración más restrictiva: si uno de los dos lados bloquea la redirección, no se produce.

Sin directiva de grupo, el mismo mecanismo se controla mediante el Registro: `fDisableCpm` en `HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp` (0 = redirección permitida, 1 = bloqueada). Además, el servicio **Cola de impresión** debe estar en ejecución en el sistema de destino; sin el spooler tampoco se crean impresoras redireccionadas.

En la misma categoría de GPO se encuentran otros dos ajustes útiles: **Usar primero el controlador de impresora Easy Print de Escritorio remoto** (el valor predeterminado y, por lo general, la opción correcta) y **Establecer la impresora predeterminada del cliente como impresora predeterminada de la sesión**.

## La advertencia «Editor desconocido»

Al abrir un archivo .rdp sin firmar que solicita redirecciones de dispositivos, el cliente muestra una advertencia de seguridad con casillas de verificación para cada recurso. Las casillas marcadas o desmarcadas allí solo se aplican a ese inicio de conexión, pero se guardan en la clave `LocalDevices` mencionada anteriormente y, por tanto, influyen silenciosamente en futuras conexiones. Quien se pregunte por qué la casilla de impresora vuelve a faltar pese a que el archivo .rdp es correcto, casi siempre encontrará allí la causa.

Hay tres formas de gestionar la advertencia, en orden creciente de esfuerzo. Primera: iniciar la conexión con `mstsc /v:hostname` en lugar de mediante el archivo .rdp; sin archivo no hay comprobación del editor y los ajustes proceden de `Default.rdp`. Segunda: aprobar previamente las redirecciones para el equipo de destino mediante el Registro; así desaparece la parte de recursos del diálogo:

```powershell
$key = "HKCU:\Software\Microsoft\Terminal Server Client\LocalDevices"
New-Item -Path $key -Force | Out-Null
Set-ItemProperty -Path $key -Name "hostname-oder-ip" -Value 0xFFFFFFFF -Type DWord
```

Tercera, la vía correcta para archivos .rdp distribuidos en la empresa: firmar el archivo con `rdpsign.exe` y un certificado, y registrar la huella digital del certificado mediante GPO como editor de confianza. Para puestos de trabajo individuales, el esfuerzo rara vez merece la pena; para archivos de conexión distribuidos centralmente, es la solución adecuada.

## Lista de comprobación para solucionar problemas

Si la impresora no aparece en la sesión, comprobar en este orden:

1. **¿Se ha vuelto a conectar?** La redirección solo se aplica al establecer la conexión, no en una sesión existente.
2. **¿El archivo correcto?** En los accesos directos, comprobar qué archivo .rdp se abre realmente; con `mstsc /v:` cuenta `Default.rdp`.
3. **¿Selección guardada?** Comprobar o eliminar la clave `LocalDevices` en el cliente.
4. **¿Bloqueo del cliente?** `DisablePrinterRedirection` en `HKLM\Software\Microsoft\Terminal Server Client` no debe tener el valor 1.
5. **¿Bloqueo del servidor?** Comprobar la GPO «No permitir la redirección de impresoras del cliente» o bien `fDisableCpm` en el sistema de destino.
6. **¿Spooler?** El servicio Cola de impresión debe estar en ejecución en el sistema de destino.
7. **Comprobación en la sesión:** `Get-Printer | Where-Object DriverName -eq "Remote Desktop Easy Print"` enumera las impresoras redireccionadas junto con el ID de sesión.

## Fuentes

1.  [Supported RDP properties](https://learn.microsoft.com/en-us/azure/virtual-desktop/rdp-properties): referencia de todas las propiedades .rdp, incluida redirectprinters con valores y valor predeterminado.

2.  [Configure printer redirection over the Remote Desktop Protocol](https://learn.microsoft.com/en-us/azure/virtual-desktop/redirection-configure-printers): Easy Print, configuración de GPO e Intune, DisablePrinterRedirection y la prueba con Get-Printer.

3.  [rdpsign](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/rdpsign): referencia del comando para firmar archivos .rdp mediante la huella digital de un certificado.
