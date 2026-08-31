---
title: "Migración a EXO sin Remote Move"
navTitle: "Migración a EXO sin Remote Move"
description: "Cómo aprovisionar buzones locales de Exchange de forma controlada como nuevos buzones vacíos de Exchange Online: copia de seguridad PST, aprobación mediante CSV, RemoteMailbox, sincronización, validación y rollback."
date: "2026-08-07"
kategorie: "Exchange OnPrem / Híbrido"
timeToRead: "12 min de lectura"
themen:
  - exchange-onprem-hybrid
  - microsoft-365-exchange
  - active-directory-entra
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange"
protokolle:
  - "powershell"
  - "migration"
related:
  - typische-ursachen-fuer-mail-loops-und-deren-behebung
  - totemomail-m365
  - ghost-sender-exchange-online-nebeneingang
slug: "migracion-a-exo-sin-remote-move"
translationId: "article-8f3c1b7a62d94e50"
draft: false
translationOf: exchange-mailboxen-ohne-remote-move-cloud-neu-erstellen
translationSourceHash: dc64d2c419e3ac0f4dd730785b3cd7f37c3f23effd2317feb4d61a46fa33401a
translationModel: gpt-5.6-terra
translatedAt: 2026-08-31T10:21:13.365Z
translationReview: automatic
url: https://rafaelpfister.ch/es/blog/migracion-a-exo-sin-remote-move
---

# Migración a EXO sin Remote Move

Un Hybrid Remote Move es la vía habitual para mover un buzón local junto con su contenido a Exchange Online. No todas las organizaciones permiten esta ruta de migración. Si una directiva de seguridad excluye los Remote Moves, puede ser aceptable adoptar deliberadamente otro enfoque: el buzón local se respalda como PST, se desvincula del usuario de AD sincronizado y se aprovisiona para el mismo usuario un buzón nuevo y vacío en Exchange Online.

Esta vía **no es una migración de buzón**. No transfiere mensajes, calendarios, reglas ni permisos a la nube. El PST sirve exclusivamente como copia de seguridad y no se importa en este escenario. Por tanto, el procedimiento solo es adecuado si un buzón de destino vacío es aceptable desde el punto de vista operativo y se ha autorizado expresamente la pérdida de la configuración activa del buzón.

## Estado objetivo y requisitos estrictos

Después del cutover, se conserva el mismo usuario de AD. Sin embargo, localmente ya no se administra como `UserMailbox`, `SharedMailbox`, `RoomMailbox` o `EquipmentMailbox`, sino como `RemoteMailbox`. Tras la sincronización, este objeto representa el nuevo buzón en Exchange Online.

El estado deseado es el siguiente:

1. El buzón local cuenta con una copia de seguridad completa en PST.
2. El buzón local está desconectado, pero aún no se ha eliminado definitivamente dentro de la retención configurada.
3. El usuario de AD existente está habilitado como RemoteMailbox.
4. Se conservan la dirección principal, los alias y el `LegacyExchangeDN` anterior.
5. Entra Connect ha sincronizado los cambios.
6. Para los buzones de usuario se ha asignado un plan de servicio de Exchange Online.
7. Exchange Online muestra un buzón real en la nube y el flujo de correo termina allí.

Además, antes de comenzar se debe aclarar lo siguiente:

- El recurso compartido para PST es accesible mediante UNC. El grupo `Exchange Trusted Subsystem` tiene allí permisos de lectura y escritura.
- La cuenta ejecutora cuenta con el rol de administración `Mailbox Import Export`.
- El PST es únicamente la copia de seguridad acordada; no se prevé ninguna importación posterior.
- La retención, Litigation Hold, eDiscovery y los requisitos regulatorios se han revisado por separado.
- Se han inventariado las delegaciones, Send-As, Send-on-Behalf, los reenvíos, las reglas de bandeja de entrada, los dispositivos móviles y los accesos de aplicaciones.
- Durante la exportación y la conmutación, los mensajes entrantes se retienen de forma controlada en el gateway previo. Los usuarios y las aplicaciones ya no deben poder escribir en el buzón de origen.
- La retención de la base de datos de buzones local cubre la ventana de rollback.

## Por qué una lista de aprobación CSV es indispensable

Una canalización directa como `Get-Mailbox | Disable-Mailbox` es demasiado arriesgada para este proceso. También podría incluir buzones del sistema, de Discovery u otros buzones no aprobados. Por ello, el siguiente procedimiento utiliza dos aprobaciones explícitas:

- `Action=CUTOVER` determina qué fila puede conmutarse realmente.
- `PstVerified=YES` confirma que el archivo de exportación se ha revisado técnica y organizativamente.

Primero solo se genera el inventario:

```powershell
$CsvPath = "C:\Migration\mailboxes.csv"
$RemoteRoutingDomain = "contoso.mail.onmicrosoft.com"

Get-Mailbox -ResultSize Unlimited -RecipientTypeDetails `
    UserMailbox,SharedMailbox,RoomMailbox,EquipmentMailbox |
    Sort-Object PrimarySmtpAddress |
    ForEach-Object {
        [pscustomobject]@{
            Identity             = $_.Identity
            DisplayName          = $_.DisplayName
            PrimarySmtpAddress   = $_.PrimarySmtpAddress.ToString()
            Alias                = $_.Alias
            SourceType           = $_.RecipientTypeDetails.ToString()
            ArchiveStatus        = $_.ArchiveStatus.ToString()
            ServerName           = $_.ServerName
            Database             = $_.Database.ToString()
            RemoteRoutingAddress = "$($_.Alias)@$RemoteRoutingDomain"
            Action               = "REVIEW"
            PstVerified          = "NO"
        }
    } |
    Export-Csv -Path $CsvPath -NoTypeInformation -Encoding UTF8
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Get-Mailbox -ResultSize Unlimited` | Elimina el límite predeterminado de 1000 resultados; sin este parámetro, faltan buzones en el inventario de entornos grandes |
| `-RecipientTypeDetails UserMailbox,SharedMailbox,RoomMailbox,EquipmentMailbox` | Limita la consulta a los cuatro tipos de buzón que se van a conmutar; los buzones del sistema y Discovery quedan excluidos |
| `Sort-Object PrimarySmtpAddress` | Ordena la salida por la dirección SMTP principal para que el archivo CSV mantenga un orden estable durante la revisión operativa |
| `Export-Csv -Path` | Ruta de destino del archivo CSV |
| `-NoTypeInformation` | Suprime la cabecera de tipo `#TYPE ...`, que las versiones antiguas de PowerShell escriben de otro modo como primera línea |
| `-Encoding UTF8` | Escribe el archivo codificado en UTF-8 para conservar correctamente los caracteres especiales en los nombres para mostrar |

</details>

A continuación, el archivo se depura desde el punto de vista operativo. Solo los buzones realmente aprobados reciben `Action=CUTOVER`. Los buzones del sistema y los objetos especiales no deben figurar en esta lista.

## Fase 1: Respaldar el buzón principal y el archivo como PST

`New-MailboxExportRequest` solo escribe en una ruta UNC. Se genera un nombre de archivo único para cada buzón. Un archivo en línea activo del Exchange local se exporta por separado:

```powershell
$CsvPath = "C:\Migration\mailboxes.csv"
$PstShare = "\\fileserver\exchange-pst$"
$BatchName = "EXO-NewMailbox-20260807"

$targets = Import-Csv $CsvPath | Where-Object Action -eq "CUTOVER"

foreach ($row in $targets) {
    $safeName = ($row.PrimarySmtpAddress -replace '[^a-zA-Z0-9@._-]', '_')
    $primaryPath = "$PstShare\$safeName.pst"

    New-MailboxExportRequest `
        -Mailbox $row.Identity `
        -FilePath $primaryPath `
        -Name "Primary-$safeName" `
        -BatchName $BatchName

    if ($row.ArchiveStatus -eq "Active") {
        New-MailboxExportRequest `
            -Mailbox $row.Identity `
            -IsArchive `
            -FilePath "$PstShare\$safeName-archive.pst" `
            -Name "Archive-$safeName" `
            -BatchName $BatchName
    }
}
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Import-Csv $CsvPath` | Lee la lista de aprobación; cada fila se convierte en un objeto con las columnas CSV como propiedades |
| `Where-Object Action -eq "CUTOVER"` | Procesa únicamente las filas aprobadas explícitamente |
| `New-MailboxExportRequest -Mailbox` | Buzón de origen de la exportación (aquí, la Identity de la fila CSV) |
| `-FilePath` | Ruta de destino del archivo PST; debe ser una ruta UNC, el cmdlet rechaza las rutas locales |
| `-Name` | Nombre único de la solicitud; permite asignar posteriormente de forma específica la exportación principal y la del archivo |
| `-BatchName` | Agrupa todas las solicitudes de una ejecución bajo un nombre de lote; base para consultar el estado y limpiar |
| `-IsArchive` | Exporta el archivo en línea en lugar del buzón principal; de ahí la segunda solicitud por cada buzón con archivo activo |

</details>

La exportación solo queda aprobada cuando **cada** solicitud tiene el estado `Completed`:

```powershell
Get-MailboxExportRequest -BatchName $BatchName |
    Get-MailboxExportRequestStatistics -IncludeReport |
    Format-Table DisplayName,Status,PercentComplete,FailureCode,Message -AutoSize

Get-MailboxExportRequest -BatchName $BatchName |
    Where-Object Status -ne "Completed"
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Get-MailboxExportRequest -BatchName` | Enumera todas las solicitudes de exportación del lote indicado |
| `Get-MailboxExportRequestStatistics -IncludeReport` | Añade a las estadísticas el informe detallado del historial, que contiene las causas de error de solicitudes individuales |
| `Format-Table ... -AutoSize` | Muestra en formato tabular las propiedades indicadas; `-AutoSize` ajusta el ancho de las columnas al contenido |
| `Where-Object Status -ne "Completed"` | Filtra todas las solicitudes que aún no han finalizado o han fallado; la salida debe estar vacía antes de continuar |

</details>

Además, se deben comprobar la existencia, el tamaño, la legibilidad, la incorporación a la copia de seguridad y la protección de acceso de los archivos. Solo entonces se establece `PstVerified=YES` para la fila CSV correspondiente.

## Fase 2: Respaldar los datos del buzón y los atributos de Exchange

Antes del primer cambio, se crea una instantánea legible por máquina para cada buzón. Es más importante que una captura de pantalla, porque posteriormente se pueden reconstruir con precisión los alias, los GUID y el `LegacyExchangeDN`:

```powershell
$SnapshotPath = "C:\Migration\Snapshots"
New-Item -ItemType Directory -Path $SnapshotPath -Force | Out-Null

Import-Csv "C:\Migration\mailboxes.csv" |
    Where-Object { $_.Action -eq "CUTOVER" -and $_.PstVerified -eq "YES" } |
    ForEach-Object {
        $mailbox = Get-Mailbox -Identity $_.Identity
        $safeName = ($_.PrimarySmtpAddress -replace '[^a-zA-Z0-9@._-]', '_')

        $mailbox |
            Select-Object Identity,DistinguishedName,ExchangeGuid,ArchiveGuid,
                RecipientTypeDetails,PrimarySmtpAddress,EmailAddresses,
                LegacyExchangeDN,Alias,Database,ServerName |
            Export-Clixml "$SnapshotPath\$safeName.xml"
    }
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `New-Item -ItemType Directory -Path ... -Force` | Crea el directorio de instantáneas; `-Force` suprime el error si ya existe |
| `Get-Mailbox -Identity` | Obtiene el objeto de buzón actual para la fila CSV correspondiente |
| `Select-Object Identity,...,ServerName` | Reduce el objeto a los atributos necesarios para una reconstrucción posterior (GUID, direcciones, `LegacyExchangeDN`, base de datos) |
| `Export-Clixml` | Serializa el objeto como CLIXML conservando el tipo; a diferencia de CSV, los valores múltiples como `EmailAddresses` se conservan íntegramente y pueden volver a leerse mediante `Import-Clixml` |

</details>

Las delegaciones y los reenvíos requieren exportaciones independientes. Como mínimo, se debe respaldar por separado la siguiente información:

```powershell
$mailbox = Get-Mailbox -Identity user01@contoso.com

$mailbox | Format-List ForwardingAddress,ForwardingSmtpAddress,DeliverToMailboxAndForward
Get-MailboxPermission -Identity $mailbox.Identity
Get-ADPermission -Identity $mailbox.DistinguishedName
Get-InboxRule -Mailbox $mailbox.Identity
Get-CalendarProcessing -Identity $mailbox.Identity -ErrorAction SilentlyContinue
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Format-List ForwardingAddress,ForwardingSmtpAddress,DeliverToMailboxAndForward` | Muestra los tres atributos de reenvío del buzón en formato de lista |
| `Get-MailboxPermission -Identity` | Enumera permisos de buzón como Full Access |
| `Get-ADPermission -Identity` | Enumera permisos de AD en el objeto de usuario, incluido Send-As; aquí espera el Distinguished Name |
| `Get-InboxRule -Mailbox` | Enumera las reglas de bandeja de entrada del lado del servidor del buzón |
| `Get-CalendarProcessing -Identity` | Muestra la configuración de reservas; relevante para buzones de salas y equipos |
| `-ErrorAction SilentlyContinue` | Suprime el error para los tipos de buzón sin configuración de reservas, para que la copia de seguridad no se interrumpa |

</details>

Estas configuraciones no se transfieren automáticamente al nuevo buzón en la nube.

## Fase 3: Desconectar el buzón local y habilitar RemoteMailbox

El cutover propiamente dicho es breve, pero tiene consecuencias. `Disable-Mailbox` elimina los atributos de Exchange del usuario de AD y desconecta el buzón local. Los datos del buzón se conservan como disconnected mailbox hasta que expire la retención de la base de datos. Justo después, `Enable-RemoteMailbox` habilita el mismo usuario de AD para Exchange Online.

El siguiente script procesa exclusivamente filas aprobadas dos veces. Conserva la dirección SMTP principal, todas las direcciones proxy existentes y el `LegacyExchangeDN` anterior como dirección X500. La entrada X500 evita NDR al responder a mensajes antiguos o al utilizar entradas antiguas de autocompletado de Outlook.

```powershell
$CsvPath = "C:\Migration\mailboxes.csv"
$PstShare = "\\fileserver\exchange-pst$"

$targets = Import-Csv $CsvPath |
    Where-Object { $_.Action -eq "CUTOVER" -and $_.PstVerified -eq "YES" }

foreach ($row in $targets) {
    $safeName = ($row.PrimarySmtpAddress -replace '[^a-zA-Z0-9@._-]', '_')
    $pstPath = "$PstShare\$safeName.pst"

    if (-not (Test-Path $pstPath)) {
        throw "PST fehlt: $pstPath"
    }

    if ((Get-Item $pstPath).Length -eq 0) {
        throw "PST ist leer: $pstPath"
    }

    $mailbox = Get-Mailbox -Identity $row.Identity -ErrorAction Stop
    $primary = $mailbox.PrimarySmtpAddress.ToString()
    $legacyDn = $mailbox.LegacyExchangeDN

    if ($primary -ine $row.PrimarySmtpAddress) {
        throw "Primaere SMTP-Adresse stimmt nicht mit der Freigabeliste ueberein: $($row.Identity)"
    }

    $preservedAddresses = @($mailbox.EmailAddresses | ForEach-Object ToString)

    Disable-Mailbox -Identity $row.Identity -Confirm:$false

    $enableParams = @{
        Identity             = $row.Identity
        Alias                = $row.Alias
        PrimarySmtpAddress   = $primary
        RemoteRoutingAddress = $row.RemoteRoutingAddress
        ACLableSyncedObjectEnabled = $true
    }

    switch ($row.SourceType) {
        "SharedMailbox"    { $enableParams.Shared = $true }
        "RoomMailbox"      { $enableParams.Room = $true }
        "EquipmentMailbox" { $enableParams.Equipment = $true }
        "UserMailbox"      { }
        default             { throw "Nicht unterstuetzter Mailbox-Typ: $($row.SourceType)" }
    }

    Enable-RemoteMailbox @enableParams

    $orderedAddresses = @(
        "SMTP:$primary"
        $preservedAddresses |
            Where-Object { $_ -notmatch '^SMTP:' -or $_.Substring(5) -ine $primary }
        "smtp:$($row.RemoteRoutingAddress)"
        "X500:$legacyDn"
    )

    $seen = [System.Collections.Generic.HashSet[string]]::new(
        [System.StringComparer]::OrdinalIgnoreCase
    )
    $finalAddresses = foreach ($address in $orderedAddresses) {
        if ($seen.Add($address)) { $address }
    }

    Set-RemoteMailbox -Identity $row.Identity -EmailAddressPolicyEnabled $false
    Set-RemoteMailbox -Identity $row.Identity -EmailAddresses $finalAddresses
    Set-RemoteMailbox -Identity $row.Identity -PrimarySmtpAddress $primary

    Get-RemoteMailbox -Identity $row.Identity |
        Format-List DisplayName,RecipientTypeDetails,PrimarySmtpAddress,
            RemoteRoutingAddress,EmailAddresses
}
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Get-Mailbox -Identity ... -ErrorAction Stop` | Obtiene el buzón de origen; `-ErrorAction Stop` convierte un error de búsqueda en un error que interrumpe la ejecución, en lugar de continuar silenciosamente |
| `Disable-Mailbox -Identity` | Elimina los atributos de Exchange del usuario de AD y desconecta el buzón local; los datos permanecen como disconnected mailbox en la base de datos |
| `-Confirm:$false` | Suprime la confirmación interactiva; aquí la aprobación se realiza mediante la lista CSV, no en el prompt |
| `Enable-RemoteMailbox -Identity` | Habilita el mismo usuario de AD como RemoteMailbox para Exchange Online |
| `-Alias` | Restablece el alias de Exchange al valor de la lista de aprobación |
| `-PrimarySmtpAddress` | Conserva la dirección SMTP principal existente |
| `-RemoteRoutingAddress` | Dirección de destino en el dominio de enrutamiento `mail.onmicrosoft.com`, mediante la cual el Exchange local alcanza el buzón en la nube |
| `-ACLableSyncedObjectEnabled` | Marca el objeto como compatible con ACL para que permisos como Full Access puedan evaluarse en Exchange Online después de la sincronización |
| `-Shared` / `-Room` / `-Equipment` | Crea el tipo especial correspondiente en lugar de un buzón de usuario; el script establece exactamente un modificador según el tipo de origen |
| `Set-RemoteMailbox -EmailAddressPolicyEnabled $false` | Excluye el objeto de la directiva de direcciones de correo para que no sobrescriba las direcciones establecidas manualmente |
| `-EmailAddresses` | Establece la lista completa de direcciones sin duplicados, incluidas las direcciones proxy antiguas, la dirección de enrutamiento y la entrada X500 |
| `Get-RemoteMailbox -Identity` | Consulta de control del resultado inmediatamente después de la conmutación |

</details>

El script no pretende ser deliberadamente una herramienta de migración totalmente automatizada. Detiene la ejecución ante la primera discrepancia para que un administrador pueda evaluar la causa y el estado. Antes de un lote de producción, el código debe validarse con unos pocos buzones de prueba y las versiones de Exchange utilizadas.

## Fase 4: Sincronizar, asignar licencias y verificar

Tras el cambio local, se inicia un ciclo delta en el servidor de Entra Connect:

```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-PolicyType Delta` | Sincroniza únicamente los objetos modificados desde el último ciclo; la alternativa `Initial` sería una ejecución completa y considerablemente más larga |

</details>

Para los buzones de usuario, debe asignarse después un plan de servicio válido de Exchange Online, por ejemplo mediante licencias basadas en grupos. Los buzones compartidos, de salas y de equipos deben evaluarse conforme a las condiciones de licencia actuales de Microsoft y las funciones necesarias.

El aprovisionamiento es asíncrono. Microsoft suele indicar menos de 30 minutos para los cambios normales, pero en casos individuales puede tardar hasta 24 horas. Durante este tiempo, el flujo de correo previo debe retener los mensajes de forma controlada en lugar de entregarlos a un destino que aún no está listo.

La comprobación local debe mostrar ahora una RemoteMailbox:

```powershell
Get-RemoteMailbox -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,RemoteRoutingAddress,EmailAddresses

Get-Mailbox -Identity user01@contoso.com -ErrorAction SilentlyContinue
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Get-RemoteMailbox -Identity` | Debe devolver el objeto conmutado como RemoteMailbox |
| `Format-List RecipientTypeDetails,...` | Muestra el tipo, las direcciones y la dirección de enrutamiento para su comprobación en formato de lista |
| `Get-Mailbox -Identity ... -ErrorAction SilentlyContinue` | Contraprueba: la llamada ya no debe devolver nada, pues localmente ya no existe ningún buzón conectado; `-ErrorAction SilentlyContinue` suprime el mensaje de error esperado |

</details>

En Exchange Online se comprueba si el anterior MailUser se ha convertido en un buzón real:

```powershell
Get-EXORecipient -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,EmailAddresses

Get-EXOMailbox -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,ExchangeGuid
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Get-EXORecipient -Identity` | Muestra el tipo de destinatario en Exchange Online; se espera `UserMailbox` o el tipo especial, ya no `MailUser` |
| `Get-EXOMailbox -Identity` | Devuelve únicamente buzones reales en la nube; un resultado acredita que el aprovisionamiento ha finalizado |
| `Format-List ...,ExchangeGuid` | Enumera los atributos de control; el `ExchangeGuid` identifica inequívocamente el nuevo buzón en la nube |

</details>

El lote solo se considera finalizado cuando, además, las siguientes pruebas se han realizado correctamente:

- Entrega desde el exterior y desde el interior
- Envío al exterior y al interior
- Respuesta a un mensaje antiguo para comprobar la dirección X500
- Inicio de sesión con Outlook y Outlook en la Web
- Delegaciones y Send-As
- Reenvíos y reglas de transporte
- Reservas de salas y equipos
- Aplicaciones, escáneres y relés SMTP
- Message Trace con entrega al nuevo buzón en la nube

## Rollback y limpieza

El buzón local de origen no debe eliminarse durante la fase de validación con `Remove-StoreMailbox`. Mientras exista dentro de la retención del buzón como disconnected mailbox, sigue habiendo una posibilidad técnica de reversión. Sin embargo, un rollback requiere una reversión controlada de los atributos de RemoteMailbox y volver a conectar el buzón local; al mismo tiempo, se debe evitar que existan dos destinos de entrega activos.

Por tanto, antes de un rollback se deben asegurar el flujo de correo, el estado de sincronización y los mensajes ya recibidos en la nube. Una reversión no es una simple línea de comando y debe formar parte del cambio como runbook probado.

Tras la aceptación satisfactoria, se limpian las solicitudes de exportación, los archivos PST se archivan conforme al concepto de protección y retención, y se eliminan los permisos temporales en el recurso compartido de exportación:

```powershell
Get-MailboxExportRequest -BatchName $BatchName |
    Remove-MailboxExportRequest -Confirm:$false
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Get-MailboxExportRequest -BatchName` | Selecciona exactamente las solicitudes de exportación del lote finalizado |
| `Remove-MailboxExportRequest -Confirm:$false` | Elimina las solicitudes sin confirmación; los propios archivos PST no se ven afectados |

</details>

Los disconnected mailboxes solo deben eliminarse definitivamente una vez transcurrida la ventana de rollback acordada y conforme al concepto de retención.

## Conclusión

Si no se permiten Hybrid Remote Moves y no es necesario trasladar datos de buzón a Exchange Online, un usuario de AD sincronizado existente puede conmutarse de forma controlada de un buzón local a un buzón nuevo en la nube. La parte crítica no es `Enable-RemoteMailbox`, sino el control del proceso que lo rodea: inventario completo, copia de seguridad PST verificada, aprobaciones explícitas, conservación de las direcciones proxy y X500, un flujo de correo controlado y una ventana de rollback real.

## Fuentes

1. [Microsoft Learn – Enable-RemoteMailbox](https://learn.microsoft.com/en-us/powershell/module/exchange/enable-remotemailbox?view=exchange-ps): Habilita un usuario de AD local existente para un buzón en el servicio basado en la nube y documenta los modificadores para buzones de usuario, compartidos, de salas y de equipos.

2. [Microsoft Learn – New-MailboxExportRequest](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-mailboxexportrequest?view=exchange-ps): Referencia para exportar buzones locales principales y archivados a archivos PST.

3. [Microsoft Learn – Mailbox import and export requests](https://learn.microsoft.com/en-us/exchange/mailbox-import-and-export-requests-exchange-2013-help): Requisitos para el recurso compartido de exportación, los permisos y Exchange Trusted Subsystem.

4. [Microsoft Learn – Disable or delete a mailbox in Exchange Server](https://learn.microsoft.com/en-us/exchange/recipients/disconnected-mailboxes/disable-or-delete-mailboxes): Comportamiento de `Disable-Mailbox`, eliminación de los atributos de Exchange y retención del buzón desconectado.

5. [Microsoft Learn – Disconnected mailboxes](https://learn.microsoft.com/en-us/exchange/recipients/disconnected-mailboxes/disconnected-mailboxes): Volver a conectar, restaurar y eliminar definitivamente buzones desconectados.

6. [Microsoft Learn – Delays in provisioning of a user or mailbox](https://learn.microsoft.com/en-us/troubleshoot/exchange/user-and-shared-mailboxes/delays-provision-mailbox-sync-changes): Duración habitual y solución de problemas en el aprovisionamiento en Exchange Online.

7. [Microsoft Learn – Move mailboxes between on-premises and Exchange Online organizations](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/move-mailboxes): El Hybrid Remote Move habitual como referencia y diferenciación respecto a la nueva implementación descrita aquí.
