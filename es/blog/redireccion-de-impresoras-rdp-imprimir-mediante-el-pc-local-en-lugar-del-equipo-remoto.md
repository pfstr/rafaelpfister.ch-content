---
title: "Redirección de impresoras RDP: imprimir a través del PC local en lugar del equipo remoto"
navTitle: "Redirección de impresoras RDP"
description: "Los trabajos de impresión de la sesión RDP deben salir por la impresora junto al usuario, no por el equipo remoto. La configuración se encuentra en tres lugares: en el cliente RDP, en el archivo .rdp y en el sistema de destino. También se explica cómo gestionar la advertencia «Editor desconocido» e incluye una lista de comprobación para la resolución de problemas."
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
translationSourceHash: 1ca6c78df61e05f90d1aefc5b20b611e5c6a4bf03908d0466da114c2823e31cd
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:33:22.009Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/redireccion-de-impresoras-rdp-imprimir-mediante-el-pc-local-en-lugar-del-equipo-remoto
---

# Redirección de impresoras RDP: imprimir a través del PC local en lugar del equipo remoto

Un usuario trabaja mediante Escritorio remoto en un equipo remoto y quiere imprimir en la impresora que tiene a su lado. Para eso existe la redirección de impresoras: el cliente RDP registra las impresoras locales en la sesión, el trabajo de impresión vuelve al cliente a través del canal RDP y se imprime allí. En el sistema de destino, la impresora aparece con el añadido **(redireccionada, sesión n)**. Por lo general, no se necesitan controladores en el equipo remoto: Windows utiliza el controlador universal **Remote Desktop Easy Print**; basta con que el controlador de impresora adecuado esté instalado en el cliente local.

La redirección solo se aplica al establecer la conexión. Después de cada cambio en la configuración, hay que desconectar por completo la sesión y volver a conectarse; no basta con minimizar la ventana RDP.

## Lado del cliente: activar la redirección

La forma más sencilla de activar la redirección de impresoras es mediante la interfaz gráfica: iniciar `mstsc`, seleccionar **Mostrar opciones**, abrir la pestaña **Recursos locales**, marcar **Impresoras** y guardar la conexión en la pestaña **General**. Quien trabaje con archivos .rdp puede ajustar la línea directamente en el archivo; los archivos .rdp son simples archivos de texto y se pueden editar con cualquier editor:

```text
redirectprinters:i:1
```

Hay una particularidad con los accesos directos sin archivo .rdp: si la conexión se inicia con `mstsc /v:hostname`, se aplican los ajustes del archivo oculto `Default.rdp` en la carpeta Documentos del usuario. Si falta allí la línea `redirectprinters:i:1`, la impresora no aparece, aunque aparentemente todo esté configurado correctamente. Este fragmento añade la línea de forma idempotente (la `0` existente se convierte en `1`, y se añade una línea si falta) y muestra el resultado para comprobarlo:

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

Hay otras dos fuentes de errores en el lado del cliente: en primer lugar, Windows guarda por equipo de destino en `HKCU\Software\Microsoft\Terminal Server Client\LocalDevices` qué redirecciones autorizó por última vez el usuario en el diálogo de seguridad; esta selección guardada sobrescribe el valor predeterminado del archivo .rdp. Eliminar la clave restablece el estado. En segundo lugar, el valor del Registro `DisablePrinterRedirection` (DWORD, valor 1) bajo `HKLM\Software\Microsoft\Terminal Server Client` desactiva completamente la redirección de impresoras en el cliente; en dispositivos administrados conviene comprobarlo antes de empezar a buscar errores en la sesión.

## Lado del servidor: permitir la redirección

En el sistema de destino, la directiva **No permitir la redirección de impresoras cliente** (Configuración del equipo → Plantillas administrativas → Componentes de Windows → Servicios de Escritorio remoto → Host de sesión de Escritorio remoto → Redirección de impresoras) toma la decisión. Si está en *Habilitada*, no se crean impresoras de cliente, independientemente de lo que solicite el cliente. Se aplica el principio de la configuración más restrictiva: si una de las dos partes bloquea la redirección, esta no se realiza.

Sin directiva de grupo, el mismo mecanismo se controla mediante el Registro: `fDisableCpm` bajo `HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp` (0 = redirección permitida, 1 = bloqueada). Además, el servicio **Cola de impresión** debe estar en ejecución en el sistema de destino; sin el spooler tampoco se crean impresoras redirigidas.

En la misma categoría de GPO hay otras dos configuraciones útiles: **Usar primero el controlador de impresión Easy Print de Escritorio remoto** (el valor predeterminado y, por lo general, la elección correcta) y **Establecer la impresora predeterminada del cliente como impresora predeterminada de la sesión**.

## La advertencia «Editor desconocido»

Al abrir un archivo .rdp sin firmar que solicita redirecciones de dispositivos, el cliente muestra una advertencia de seguridad con casillas para los distintos recursos. Las casillas marcadas o desmarcadas allí solo se aplican a ese inicio de conexión, pero se guardan en la clave `LocalDevices` mencionada anteriormente y, por tanto, influyen silenciosamente en futuras conexiones. Quien se pregunte por qué la casilla de la impresora vuelve a faltar pese a que el archivo .rdp es correcto, casi siempre encontrará la causa allí.

Desde la actualización de seguridad de abril de 2026 (CVE-2026-26151), Conexión a Escritorio remoto muestra un nuevo diálogo de seguridad cada vez que se abre un archivo .rdp, antes de establecer la conexión. Todas las redirecciones solicitadas por el archivo, incluidas las impresoras, están desactivadas de forma predeterminada y deben activarse individualmente. Los archivos sin firmar muestran además el aviso «Caution: Unknown remote connection». Según Microsoft, las conexiones iniciadas directamente con un nombre de equipo en Conexión a Escritorio remoto no se ven afectadas. Como solución alternativa temporal, Microsoft documenta el valor del Registro `RedirectionWarningDialogVersion` (DWORD, valor 1) bajo `HKLM\Software\Policies\Microsoft\Windows NT\Terminal Services\Client`, que recupera el diálogo anterior; la compatibilidad con él puede desaparecer con una futura actualización.

Hay tres formas de gestionar la advertencia, por orden creciente de esfuerzo. Primera: iniciar la conexión mediante `mstsc /v:hostname` en lugar de usar el archivo .rdp; sin archivo no hay comprobación del editor y la configuración procede de `Default.rdp`. Tampoco aparece el diálogo de seguridad introducido en abril de 2026. Segunda: aprobar previamente las redirecciones para el equipo de destino mediante el Registro; así desaparece la parte de recursos del diálogo anterior. Microsoft no documenta si esta aprobación previa también se aplica al nuevo diálogo de seguridad desde abril de 2026:

```powershell
$key = "HKCU:\Software\Microsoft\Terminal Server Client\LocalDevices"
New-Item -Path $key -Force | Out-Null
Set-ItemProperty -Path $key -Name "hostname-oder-ip" -Value 0xFFFFFFFF -Type DWord
```

Tercera, la vía adecuada para archivos .rdp distribuidos en la empresa: firmar el archivo con `rdpsign.exe` y un certificado, y almacenar la huella digital del certificado mediante GPO como editor de confianza. Para puestos de trabajo individuales rara vez merece la pena el esfuerzo; para archivos de conexión distribuidos de forma centralizada, es la solución correcta.

## Lista de comprobación para la resolución de problemas

Si la impresora no aparece en la sesión, comprobar en este orden:

1. **¿Se volvió a conectar?** La redirección solo se aplica al establecer la conexión, no en una sesión existente.
2. **¿Archivo correcto?** En los accesos directos, comprobar qué archivo .rdp se abre realmente; con `mstsc /v:` cuenta `Default.rdp`.
3. **¿Selección guardada?** Comprobar o eliminar la clave `LocalDevices` en el cliente. Al iniciar mediante un archivo .rdp, comprobar además si la impresora se activó en el diálogo de seguridad; desde abril de 2026 está desactivada de forma predeterminada.
4. **¿Bloqueo del cliente?** `DisablePrinterRedirection` bajo `HKLM\Software\Microsoft\Terminal Server Client` no debe tener el valor 1.
5. **¿Bloqueo del servidor?** Comprobar la GPO «No permitir la redirección de impresoras cliente» o `fDisableCpm` en el sistema de destino.
6. **¿Spooler?** El servicio Cola de impresión debe estar en ejecución en el sistema de destino.
7. **Comprobación en la sesión:** `Get-Printer | Where-Object DriverName -eq "Remote Desktop Easy Print"` enumera las impresoras redirigidas junto con el ID de sesión.

## Historial de cambios

- **25 de octubre de 2026:** Añadido: desde la actualización de seguridad de abril de 2026 (CVE-2026-26151), un nuevo diálogo al abrir archivos .rdp muestra todas las redirecciones, incluidas las impresoras, desactivadas de forma predeterminada; se añade también el valor del Registro RedirectionWarningDialogVersion y un paso de comprobación adicional.

## Fuentes

1.  [Supported RDP properties](https://learn.microsoft.com/en-us/azure/virtual-desktop/rdp-properties): referencia de todas las propiedades .rdp, incluida redirectprinters con valores y valor predeterminado.

2.  [Configure printer redirection over the Remote Desktop Protocol](https://learn.microsoft.com/en-us/azure/virtual-desktop/redirection-configure-printers): Easy Print, configuración de GPO e Intune, DisablePrinterRedirection y la prueba con Get-Printer.

3.  [rdpsign](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/rdpsign): referencia de comandos para firmar archivos .rdp mediante la huella digital de un certificado.

4.  [Understanding security warnings when opening Remote Desktop (RDP) files](https://learn.microsoft.com/en-us/windows-server/remote/remote-desktop-services/remotepc/understanding-security-warnings): nuevo diálogo de seguridad desde la actualización de abril de 2026, redirecciones desactivadas de forma predeterminada y el valor del Registro RedirectionWarningDialogVersion.
