---
title: "Migración a EXO sin Remote Move"
navTitle: "Migración a EXO sin Remote Move"
description: "Cómo aprovisionar de forma controlada buzones locales de Exchange como buzones nuevos y vacíos de Exchange Online: copia de seguridad PST, aprobación mediante CSV, RemoteMailbox, sincronización, validación y rollback."
date: "2026-08-07"
kategorie: "Exchange OnPrem / Hybrid"
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
url: https://rafaelpfister.ch/es/blog/migracion-a-exo-sin-remote-move
translationSourceHash: 861f11b6e2f1e316ca773f049637fa2ac6ed5efdab5ec74d8c28178f3ea7e98c
translationModel: gpt-5.6-terra
translatedAt: 2026-08-08T13:48:25.067Z
translationReview: automatic
---

# Migración a EXO sin Remote Move

Un Hybrid Remote Move es la vía habitual para trasladar un buzón local de Exchange, incluido su contenido, a Exchange Online. No todas las organizaciones permiten esta vía de migración. Si una política de seguridad excluye los Remote Moves, puede ser aceptable un enfoque deliberadamente distinto: se realiza una copia de seguridad del buzón local como PST, se desvincula del usuario de AD sincronizado y se aprovisiona un buzón nuevo y vacío en Exchange Online para el mismo usuario.

Esta vía **no es una migración de buzón**. No transfiere mensajes, calendarios, reglas ni permisos a la nube. El PST sirve exclusivamente como copia de seguridad y no se importa en este escenario. Por tanto, el procedimiento solo es adecuado si se acepta desde el punto de vista funcional un buzón de destino vacío y se aprueba expresamente la pérdida de la configuración activa del buzón.

## Estado objetivo y requisitos estrictos

Después del cutover, se conserva el mismo usuario de AD. Sin embargo, localmente ya no se administra como `UserMailbox`, `SharedMailbox`, `RoomMailbox` o `EquipmentMailbox`, sino como `RemoteMailbox`. Tras la sincronización, este objeto representa el nuevo buzón en Exchange Online.

El estado deseado es el siguiente:

1. Se ha realizado una copia de seguridad completa del buzón local como PST.
2. El buzón local está desconectado, pero aún no se ha eliminado definitivamente dentro de la retención configurada.
3. El usuario de AD existente está habilitado como RemoteMailbox.
4. Se conservan la dirección principal, los alias y el antiguo `LegacyExchangeDN`.
5. Entra Connect ha sincronizado los cambios.
6. Para los buzones de usuario se ha asignado un plan de servicio de Exchange Online.
7. Exchange Online muestra un buzón real en la nube y el flujo de correo termina allí.

Además, antes de empezar deben aclararse los siguientes puntos:

- El recurso compartido para PST es accesible mediante UNC. El grupo `Exchange Trusted Subsystem` dispone allí de permisos de lectura y escritura.
- La cuenta ejecutora tiene el rol de administración `Mailbox Import Export`.
- El PST es únicamente la copia de seguridad acordada; no está prevista ninguna importación posterior.
- La retención, Litigation Hold, eDiscovery y los requisitos normativos se han revisado por separado.
- Se han inventariado las delegaciones, Send-As, Send-on-Behalf, los reenvíos, las reglas de bandeja de entrada, los dispositivos móviles y los accesos de aplicaciones.
- Durante la exportación y el cambio, los mensajes entrantes se retienen de forma controlada en el gateway anterior. Los usuarios y las aplicaciones ya no pueden escribir en el buzón de origen.
- La retención de la base de datos de buzones local cubre la ventana de rollback.

## Por qué es indispensable una lista de aprobación CSV

Una canalización directa como `Get-Mailbox | Disable-Mailbox` es demasiado arriesgada para este procedimiento. También podría incluir buzones de sistema, de Discovery u otros buzones no aprobados. Por ello, el siguiente proceso trabaja con dos aprobaciones explícitas:

- `Action=CUTOVER` determina qué fila puede cambiarse realmente.
- `PstVerified=YES` confirma que el archivo de exportación se ha revisado técnica y organizativamente.

Primero, solo se genera el inventario:

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

A continuación, el archivo se depura desde el punto de vista funcional. Solo los buzones realmente aprobados reciben `Action=CUTOVER`. Los buzones de sistema y los objetos especiales no deben incluirse en esta lista.

## Fase 1: Guardar el buzón principal y el archivo como PST

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

La exportación solo queda aprobada cuando **cada** solicitud tiene el estado `Completed`:

```powershell
Get-MailboxExportRequest -BatchName $BatchName |
    Get-MailboxExportRequestStatistics -IncludeReport |
    Format-Table DisplayName,Status,PercentComplete,FailureCode,Message -AutoSize

Get-MailboxExportRequest -BatchName $BatchName |
    Where-Object Status -ne "Completed"
```

Además, deben verificarse la existencia, el tamaño, la legibilidad, la incorporación a la copia de seguridad y la protección de acceso de los archivos. Solo entonces se establece `PstVerified=YES` para la fila CSV correspondiente.

## Fase 2: Guardar los datos del buzón y los atributos de Exchange

Antes del primer cambio, se crea una instantánea legible por máquina para cada buzón. Es más importante que una captura de pantalla, ya que posteriormente pueden reconstruirse con exactitud los alias, los GUID y el `LegacyExchangeDN`:

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

Las delegaciones y los reenvíos requieren exportaciones propias. Como mínimo, debe guardarse por separado la siguiente información:

```powershell
$mailbox = Get-Mailbox -Identity user01@contoso.com

$mailbox | Format-List ForwardingAddress,ForwardingSmtpAddress,DeliverToMailboxAndForward
Get-MailboxPermission -Identity $mailbox.Identity
Get-ADPermission -Identity $mailbox.DistinguishedName
Get-InboxRule -Mailbox $mailbox.Identity
Get-CalendarProcessing -Identity $mailbox.Identity -ErrorAction SilentlyContinue
```

Estas configuraciones no se transfieren automáticamente al nuevo buzón en la nube.

## Fase 3: Desconectar el buzón local y habilitar RemoteMailbox

El cutover propiamente dicho es breve, pero tiene consecuencias. `Disable-Mailbox` elimina los atributos de Exchange del usuario de AD y desconecta el buzón local. Los datos del buzón permanecen como buzón desconectado hasta que expire la retención de la base de datos. Inmediatamente después, `Enable-RemoteMailbox` habilita el mismo usuario de AD para Exchange Online.

El siguiente script procesa exclusivamente filas con doble aprobación. Conserva la dirección SMTP principal, todas las direcciones proxy existentes y el antiguo `LegacyExchangeDN` como dirección X500. La entrada X500 evita NDR al responder a mensajes antiguos o al usar entradas antiguas de autocompletar de Outlook.

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

El script no pretende deliberadamente ser un motor de migración completamente autónomo. Detiene la ejecución ante la primera contradicción para que un administrador pueda evaluar la causa y el estado. Antes de un lote de producción, el código debe validarse con unos pocos buzones de prueba y con las versiones de Exchange utilizadas.

## Fase 4: Sincronizar, asignar licencias y verificar

Después del cambio local, se inicia un ciclo delta en el servidor de Entra Connect:

```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```

Para los buzones de usuario debe asignarse posteriormente un plan de servicio válido de Exchange Online, por ejemplo mediante licencias basadas en grupos. Los buzones compartidos, de sala y de equipo deben evaluarse conforme a las condiciones de licencia actuales de Microsoft y las funciones requeridas.

El aprovisionamiento es asíncrono. Microsoft suele indicar menos de 30 minutos para cambios normales, aunque en casos concretos puede tardar hasta 24 horas. Durante este tiempo, el flujo de correo anterior debe retener los mensajes de forma controlada, en lugar de entregarlos a un destino aún no preparado.

La comprobación local debe mostrar ahora una RemoteMailbox:

```powershell
Get-RemoteMailbox -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,RemoteRoutingAddress,EmailAddresses

Get-Mailbox -Identity user01@contoso.com -ErrorAction SilentlyContinue
```

En Exchange Online se comprueba si el MailUser anterior se ha convertido en un buzón real:

```powershell
Get-EXORecipient -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,EmailAddresses

Get-EXOMailbox -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,ExchangeGuid
```

El lote solo se considera completado cuando además las siguientes pruebas han sido satisfactorias:

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

El buzón de origen local no debe eliminarse durante la fase de validación con `Remove-StoreMailbox`. Mientras exista como buzón desconectado dentro de la retención del buzón, sigue existiendo una posibilidad técnica de reversión. Sin embargo, un rollback requiere una reversión controlada de los atributos de RemoteMailbox y volver a conectar el buzón local; al mismo tiempo, debe evitarse que existan dos destinos de entrega activos.

Por ello, antes de un rollback deben asegurarse el flujo de correo, el estado de sincronización y los mensajes ya recibidos en la nube. Una reversión no es una simple instrucción de una línea y debe formar parte del cambio como runbook probado.

Tras la aceptación satisfactoria, se limpian las solicitudes de exportación, se archivan los archivos PST conforme al concepto de protección y retención y se eliminan los permisos temporales en el recurso compartido de exportación:

```powershell
Get-MailboxExportRequest -BatchName $BatchName |
    Remove-MailboxExportRequest -Confirm:$false
```

Los buzones desconectados solo deben eliminarse definitivamente después de que expire la ventana de rollback acordada y conforme al concepto de retención.

## Conclusión

Si los Hybrid Remote Moves no están permitidos y no es necesario transferir datos de buzón a Exchange Online, un usuario de AD sincronizado existente puede cambiarse de forma controlada de un buzón local a un nuevo buzón en la nube. La parte crítica no es `Enable-RemoteMailbox`, sino el control del proceso a su alrededor: inventario completo, copia de seguridad PST verificada, aprobaciones explícitas, conservación de las direcciones proxy y X500, un flujo de correo controlado y una ventana de rollback real.

## Fuentes

1. [Microsoft Learn – Enable-RemoteMailbox](https://learn.microsoft.com/en-us/powershell/module/exchange/enable-remotemailbox?view=exchange-ps): Habilita un usuario de AD local existente para un buzón en el servicio basado en la nube y documenta los modificadores para buzones de usuario, compartidos, de sala y de equipo.

2. [Microsoft Learn – New-MailboxExportRequest](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-mailboxexportrequest?view=exchange-ps): Referencia para la exportación de buzones locales principales y archivados a archivos PST.

3. [Microsoft Learn – Mailbox import and export requests](https://learn.microsoft.com/en-us/exchange/mailbox-import-and-export-requests-exchange-2013-help): Requisitos para el recurso compartido de exportación, los permisos y Exchange Trusted Subsystem.

4. [Microsoft Learn – Disable or delete a mailbox in Exchange Server](https://learn.microsoft.com/en-us/exchange/recipients/disconnected-mailboxes/disable-or-delete-mailboxes): Comportamiento de `Disable-Mailbox`, eliminación de los atributos de Exchange y retención del buzón desconectado.

5. [Microsoft Learn – Disconnected mailboxes](https://learn.microsoft.com/en-us/exchange/recipients/disconnected-mailboxes/disconnected-mailboxes): Reconexión, restauración y eliminación definitiva de buzones desconectados.

6. [Microsoft Learn – Delays in provisioning of a user or mailbox](https://learn.microsoft.com/en-us/troubleshoot/exchange/user-and-shared-mailboxes/delays-provision-mailbox-sync-changes): Duración habitual y solución de problemas del aprovisionamiento en Exchange Online.

7. [Microsoft Learn – Move mailboxes between on-premises and Exchange Online organizations](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/move-mailboxes): El Hybrid Remote Move habitual como referencia y delimitación respecto a la nueva creación descrita aquí.
