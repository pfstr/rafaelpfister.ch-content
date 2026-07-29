---
title: "Acceder a Exchange Online con PowerShell: sustituir EWS por Microsoft Graph"
navTitle: "Graph en PowerShell"
description: "EWS finaliza en Exchange Online el 1 de octubre de 2026. Así puede registrar una aplicación, autenticar un script de PowerShell mediante certificado, limitar el acceso a buzones individuales y procesar mensajes y adjuntos a través de Microsoft Graph."
date: "2026-07-11"
kategorie: "Totemomail"
timeToRead: "5 min de lectura"
themen:
  - "microsoft-365-exchange"
slug: "acceder-a-exchange-online-con-powershell-sustituir-ews-por-microsoft-graph"
translationOf: "microsoft-graph-powershell-postfach-anbindung"
url: "https://rafaelpfister.ch/es/blog/acceder-a-exchange-online-con-powershell-sustituir-ews-por-microsoft-graph"
---

# Acceder a Exchange Online con PowerShell: sustituir EWS por Microsoft Graph

Microsoft retirará Exchange Web Services (EWS) de Exchange Online el **1 de octubre de 2026**. Por ello, los scripts que recuperan mensajes o adjuntos de un buzón deben migrar a Microsoft Graph.

El ejemplo de este artículo se ejecuta sin inicio de sesión de usuario: descarga adjuntos ZIP de un buzón, los descomprime, mueve los mensajes procesados y, a continuación, envía un informe. Para ello se necesita un registro de aplicación, un certificado y dos permisos de Graph. Valores como `example.com`, el ID de inquilino y el ID de aplicación son marcadores de posición.

## 1. Módulos de PowerShell necesarios

Bastan tres módulos del SDK de Microsoft Graph, no todo el metapaquete `Microsoft.Graph`:

```powershell
Install-Module Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions -Scope AllUsers
```

## 2\. Registro de aplicación en Entra ID

Un script desatendido inicia sesión como aplicación con sus propios permisos. Cree un nuevo registro en el [Centro de administración de Entra](https://entra.microsoft.com) en **App registrations** y asigne estos dos permisos en **API permissions → Microsoft Graph → Application permissions**:

-   `Mail.ReadWrite`: leer correos y moverlos tras su procesamiento
    
-   `Mail.Send`: enviar el correo del informe
    

A continuación, conceda el consentimiento de administrador y anote el ID de inquilino y el ID de aplicación (cliente).

## 3\. Certificado en lugar de secreto de cliente

Un inicio de sesión solo de aplicación funciona con un secreto de cliente o un certificado. Para tareas programadas, un certificado es la mejor opción: la clave privada permanece en el almacén de certificados y no hay ninguna contraseña en el script. Cree el certificado en el servidor que ejecutará la tarea y exporte únicamente la parte pública:

```powershell
$cert = New-SelfSignedCertificate -Subject "CN=eCall-Graph" `
    -CertStoreLocation "Cert:\LocalMachine\My" `
    -KeyExportPolicy NonExportable -KeySpec Signature `
    -KeyLength 2048 -NotAfter (Get-Date).AddYears(2)

Export-Certificate -Cert $cert -FilePath .\eCall-Graph.cer
$cert.Thumbprint   # -> usar como huella digital en el script
```

Cargue el archivo `.cer`\-Datei exportado en el registro de aplicación, en Certificates & secrets. La cuenta de la tarea programada necesita permiso de lectura sobre la clave privada (`certlm.msc` → certificado → All Tasks → Manage Private Keys).

## 4\. Limitar el acceso a buzones individuales

Inicialmente, los permisos de aplicación se aplican a todos los buzones del inquilino. Por ello, limite la aplicación a un grupo de seguridad habilitado para correo mediante una Application Access Policy. Este paso se ejecuta una sola vez en Exchange Online PowerShell:

```powershell
New-ApplicationAccessPolicy -AppId "<ID-de-aplicacion>" `
    -PolicyScopeGroupId "graph-mailboxes@example.com" `
    -AccessRight RestrictAccess `
    -Description "eCall Graph: solo buzón de registros"

# Comprobar la efectividad
Test-ApplicationAccessPolicy -AppId "<ID-de-aplicacion>" -Identity "ecall-logs@example.com"
```

## 5\. Establecer la conexión

El inicio de sesión utiliza el ID de inquilino, el ID de aplicación y la huella digital del certificado, sin ninguna interacción del usuario:

```powershell
$TenantId   = "00000000-0000-0000-0000-000000000000"
$ClientId   = "00000000-0000-0000-0000-000000000000"
$Thumbprint = "0000000000000000000000000000000000000000"
$Mailbox    = "ecall-logs@example.com"

Import-Module Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions
Connect-MgGraph -TenantId $TenantId -ClientId $ClientId `
    -CertificateThumbprint $Thumbprint -NoWelcome
```

## 6. Leer mensajes y descargar adjuntos ZIP

Ahora el script puede recorrer la bandeja de entrada, guardar y descomprimir adjuntos ZIP, y mover los mensajes procesados a «Elementos eliminados». La descarga se realiza mediante el punto de conexión `/$value` y `Invoke-MgGraphRequest -OutputFilePath`. Así, el contenido sin procesar se guarda directamente en un archivo, sin mantener un adjunto grande completo en memoria:

```powershell
Add-Type -AssemblyName System.IO.Compression.FileSystem
$Zielordner = "D:\Import\{0:yyyyMMdd_HHmmss}" -f (Get-Date)

$messages = Get-MgUserMessage -UserId $Mailbox -Top 100 `
    -Property id, subject, hasAttachments

foreach ($msg in $messages) {
    $ordner = Join-Path $Zielordner $msg.Id
    New-Item -Path $ordner -ItemType Directory -Force | Out-Null

    $anhaenge = Get-MgUserMessageAttachment -UserId $Mailbox -MessageId $msg.Id |
        Where-Object {
            $_.AdditionalProperties['@odata.type'] -eq '#microsoft.graph.fileAttachment' -and
            $_.Name -like '*.zip'
        }

    foreach ($att in $anhaenge) {
        $zip = Join-Path $ordner $att.Name
        $uri = "https://graph.microsoft.com/v1.0/users/$Mailbox/messages/$($msg.Id)/attachments/$($att.Id)/`$value"
        Invoke-MgGraphRequest -Method GET -Uri $uri -OutputFilePath $zip
        [System.IO.Compression.ZipFile]::ExtractToDirectory($zip, $ordner)
    }

    # verarbeitete Mail in "Geloeschte Elemente" verschieben
    Move-MgUserMessage -UserId $Mailbox -MessageId $msg.Id -DestinationId "deleteditems" | Out-Null
}
```

Para más de 100 correos, trabaje con `Get-MgUserMessage -All` o paginación; para una ejecución mensual, normalmente basta con un lote.

## 7\. Enviar el correo del informe mediante Graph

También `Send-MailMessage` está obsoleto. Con el mismo registro de aplicación (permiso `Mail.Send`), el correo se envía directamente a través de Graph, aquí con un archivo como adjunto codificado en base64:

```powershell
$pfad = "D:\Reports\report.csv"
$body = @{
    message = @{
        subject = "eCall Report"
        body    = @{ contentType = "HTML"; content = "<b>Lauf erfolgreich</b>" }
        toRecipients = @(@{ emailAddress = @{ address = "empfaenger@example.com" } })
        attachments  = @(@{
            "@odata.type" = "#microsoft.graph.fileAttachment"
            name          = Split-Path $pfad -Leaf
            contentBytes  = [Convert]::ToBase64String([IO.File]::ReadAllBytes($pfad))
        })
    }
    saveToSentItems = $true
}
Send-MgUserMail -UserId "reporting@example.com" -BodyParameter $body
```

## 8\. Ejecutar sin supervisión

Como tarea programada, el script se ejecuta sin inicio de sesión porque el certificado se encuentra en el almacén de la cuenta:

```powershell
$action  = New-ScheduledTaskAction -Execute "powershell.exe" `
    -Argument '-NoProfile -ExecutionPolicy Bypass -File "D:\Scripts\graph-import.ps1"'
$trigger = New-ScheduledTaskTrigger -Daily -At 06:00
Register-ScheduledTask -TaskName "eCall-Graph-Import" -Action $action -Trigger $trigger `
    -User "DOMAIN\svc-ecall" -Password (Read-Host "Contraseña")
```

El ejemplo completo con registro y gestión de errores está disponible en GitHub: [pfstr/eCall-Log-Analyzer](https://github.com/pfstr/eCall-Log-Analyzer).

## Fuentes

1.  [Microsoft – «Retirement of Exchange Web Services in Exchange Online»](https://techcommunity.microsoft.com/blog/exchange/retirement-of-exchange-web-services-in-exchange-online/3924440): anuncio y fecha límite (1 de octubre de 2026) para el fin de EWS en Exchange Online.
    
2.  [Microsoft Learn – «Get access without a user (App-only)»](https://learn.microsoft.com/en-us/graph/auth-v2-service): autenticación solo de aplicación con Microsoft Graph mediante certificado.
    
3.  [Microsoft Learn – «Limiting application permissions to specific mailboxes»](https://learn.microsoft.com/en-us/graph/auth-limit-mailbox-access): Application Access Policy para limitar la aplicación a buzones individuales.
