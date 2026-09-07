---
title: "Acceder a Exchange Online con PowerShell: sustituir EWS por Microsoft Graph"
navTitle: "Graph en PowerShell"
description: "EWS finaliza en Exchange Online el 1 de octubre de 2026. Así puede registrar una aplicación, autenticar un script de PowerShell mediante certificado, restringir el acceso a buzones individuales y procesar mensajes y archivos adjuntos con Microsoft Graph."
date: "2026-07-11"
kategorie: "Totemomail"
timeToRead: "5 min de lectura"
themen:
  - microsoft-365-exchange
slug: "acceder-a-exchange-online-con-powershell-sustituir-ews-por-microsoft-graph"
translationOf: "microsoft-graph-powershell-postfach-anbindung"
translationId: article-4c6a02c79b7bf0fe
translationReview: automatic
translationSourceHash: 66e214f25e8088562270157199f9ff55fcd9828362abf145170a12f287cd0f6c
translatedAt: 2026-09-05T07:56:07.512Z
url: https://rafaelpfister.ch/es/blog/acceder-a-exchange-online-con-powershell-sustituir-ews-por-microsoft-graph
translationModel: gpt-5.6-terra
---

# Acceder a Exchange Online con PowerShell: sustituir EWS por Microsoft Graph

Microsoft retirará Exchange Web Services (EWS) de Exchange Online el **1 de octubre de 2026**. Por ello, los scripts que recuperan mensajes o archivos adjuntos de un buzón deben migrar a Microsoft Graph.

El ejemplo de este artículo se ejecuta sin inicio de sesión de usuario: descarga archivos adjuntos ZIP de un buzón, los descomprime, mueve los mensajes procesados y, a continuación, envía un informe. Para ello se necesitan un registro de aplicación, un certificado y dos permisos de Graph. Valores como `example.com`, el ID de inquilino y el ID de aplicación son marcadores de posición.

## 1. Módulos de PowerShell necesarios

Bastan tres módulos del SDK de Microsoft Graph, no todo el metamódulo `Microsoft.Graph`:

```powershell
Install-Module Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions -Scope AllUsers
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions` | Los tres submódulos necesarios como argumento posicional `-Name`: autenticación, cmdlets de correo y acciones como `Send-MgUserMail` |
| `-Scope AllUsers` | Instala los módulos para todo el equipo en `Program Files`; necesario para que también estén disponibles para la cuenta de servicio configurada posteriormente para la tarea programada (requiere derechos de administrador) |

</details>

## 2\. Registro de aplicación en Entra ID

Un script desatendido se autentica como aplicación con sus propios permisos. Cree un nuevo registro en el [Centro de administración de Entra](https://entra.microsoft.com) y asigne estos dos permisos en **API permissions → Microsoft Graph → Application permissions**:

-   `Mail.ReadWrite`: leer correos y moverlos tras su procesamiento
    
-   `Mail.Send`: enviar el correo del informe
    
A continuación, conceda el consentimiento del administrador y anote el ID de inquilino y el ID de aplicación (cliente).

## 3\. Certificado en lugar de secreto de cliente

Una autenticación solo de aplicación funciona con un secreto de cliente o un certificado. Para tareas programadas, un certificado es la mejor opción: la clave privada permanece en el almacén de certificados y el script no contiene ninguna contraseña. Cree el certificado en el servidor que lo ejecutará y exporte únicamente la parte pública:

```powershell
$cert = New-SelfSignedCertificate -Subject "CN=eCall-Graph" `
    -CertStoreLocation "Cert:\LocalMachine\My" `
    -KeyExportPolicy NonExportable -KeySpec Signature `
    -KeyLength 2048 -NotAfter (Get-Date).AddYears(2)

Export-Certificate -Cert $cert -FilePath .\eCall-Graph.cer
$cert.Thumbprint   # -> im Skript als Thumbprint verwenden
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `New-SelfSignedCertificate -Subject` | Nombre del sujeto del certificado; sirve únicamente para identificarlo en el almacén de certificados |
| `-CertStoreLocation "Cert:\LocalMachine\My"` | Guarda el certificado en el almacén del equipo, no en el del usuario; así está disponible independientemente del usuario que haya iniciado sesión |
| `-KeyExportPolicy NonExportable` | Impide la exportación de la clave privada; no abandona el servidor |
| `-KeySpec Signature` | Crea una clave de firma; la autenticación de la aplicación la utiliza para firmar la aserción de solicitud de token |
| `-KeyLength 2048` | Longitud de clave RSA de 2048 bits |
| `-NotAfter (Get-Date).AddYears(2)` | Fecha de expiración dentro de dos años; después se debe renovar y volver a cargar el certificado |
| `Export-Certificate -Cert` | Objeto de certificado que se va a exportar |
| `-FilePath` | Archivo de destino; como `.cer` contiene únicamente la parte pública |

</details>

Cargue el archivo `.cer`\-exportado en el registro de aplicación, en Certificates & secrets. La cuenta de la tarea programada necesita permiso de lectura sobre la clave privada (`certlm.msc` → certificado → All Tasks → Manage Private Keys).

## 4\. Restringir el acceso a buzones individuales

Los permisos de aplicación se aplican inicialmente a todos los buzones del inquilino. Por ello, limite la aplicación mediante una Application Access Policy a un grupo de seguridad habilitado para correo. Este paso se ejecuta una sola vez en Exchange Online PowerShell:

```powershell
New-ApplicationAccessPolicy -AppId "<App-ID>" `
    -PolicyScopeGroupId "graph-mailboxes@example.com" `
    -AccessRight RestrictAccess `
    -Description "eCall Graph: nur Log-Postfach"

# Comprobar la efectividad
Test-ApplicationAccessPolicy -AppId "<App-ID>" -Identity "ecall-logs@example.com"
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `New-ApplicationAccessPolicy -AppId` | ID de aplicación (cliente) del registro de aplicación al que se aplica la directiva |
| `-PolicyScopeGroupId` | Grupo de seguridad habilitado para correo cuyos miembros definen el ámbito |
| `-AccessRight RestrictAccess` | Restringe la aplicación a los buzones del grupo; la alternativa `DenyAccess` bloquearía precisamente esos buzones |
| `-Description` | Texto libre para documentar la directiva |
| `Test-ApplicationAccessPolicy -Identity` | Comprueba para un buzón concreto si la aplicación puede acceder a él (`AccessCheckResult: Granted` o `Denied`) |

</details>

## 5\. Establecer la conexión

La autenticación utiliza el ID de inquilino, el ID de aplicación y la huella digital del certificado, sin ninguna interacción del usuario:

```powershell
$TenantId   = "00000000-0000-0000-0000-000000000000"
$ClientId   = "00000000-0000-0000-0000-000000000000"
$Thumbprint = "0000000000000000000000000000000000000000"
$Mailbox    = "ecall-logs@example.com"

Import-Module Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions
Connect-MgGraph -TenantId $TenantId -ClientId $ClientId `
    -CertificateThumbprint $Thumbprint -NoWelcome
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Connect-MgGraph -TenantId` | Inquilino ante el que se autentica la aplicación |
| `-ClientId` | ID de aplicación (cliente) del registro de aplicación |
| `-CertificateThumbprint` | Selecciona el certificado de autenticación de su almacén local mediante su huella digital; la combinación de `-ClientId` y certificado permite una autenticación solo de aplicación sin usuario |
| `-NoWelcome` | Suprime el mensaje de bienvenida tras la autenticación; resulta útil para salidas de scripts y registros |

</details>

## 6. Leer mensajes y descargar archivos adjuntos ZIP

Ahora el script puede recorrer la bandeja de entrada, guardar y descomprimir archivos adjuntos ZIP, así como mover los mensajes procesados a «Elementos eliminados». La descarga se realiza mediante el punto de conexión `/$value` y `Invoke-MgGraphRequest -OutputFilePath`. De este modo, el contenido sin procesar se guarda directamente en un archivo, sin mantener por completo un archivo adjunto grande en memoria:

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

    # mover el correo procesado a "Elementos eliminados"
    Move-MgUserMessage -UserId $Mailbox -MessageId $msg.Id -DestinationId "deleteditems" | Out-Null
}
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Add-Type -AssemblyName System.IO.Compression.FileSystem` | Carga el ensamblado .NET con la clase `ZipFile` para descomprimir |
| `Get-MgUserMessage -UserId` | Buzón cuyos mensajes se leen; en la autenticación solo de aplicación, la indicación es obligatoria |
| `-Top 100` | Limita la consulta a un máximo de 100 mensajes por llamada |
| `-Property id, subject, hasAttachments` | Solicita solo los campos necesarios; esto reduce la respuesta y acelera la llamada |
| `Get-MgUserMessageAttachment -MessageId` | Mensaje cuyos archivos adjuntos se enumeran |
| `Invoke-MgGraphRequest -Method GET` | Método HTTP de la llamada directa a la API de Graph |
| `-Uri` | Punto de conexión llamado; el `/$value` añadido devuelve el contenido sin procesar del archivo adjunto en lugar de un objeto JSON |
| `-OutputFilePath` | Escribe la respuesta directamente en el archivo de destino, sin mantener el archivo adjunto completo en memoria |
| `Move-MgUserMessage -DestinationId "deleteditems"` | Mueve el mensaje procesado a la carpeta de destino; `deleteditems` es el nombre de carpeta conocido para «Elementos eliminados» |

</details>

Si hay más de 100 correos, trabaje con `Get-MgUserMessage -All` o paginación; para una ejecución mensual suele bastar un lote.

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

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-UserId` | Buzón en cuyo nombre se envía el correo; debe estar cubierto por el ámbito de la Application Access Policy |
| `-BodyParameter` | El mensaje completo como tabla hash en el esquema de Graph: `message` con asunto, cuerpo, destinatarios y archivos adjuntos, así como `saveToSentItems` para guardarlo en «Elementos enviados» |

</details>

## 8\. Ejecutar sin supervisión

Como tarea programada, el script se ejecuta sin autenticación interactiva porque el certificado está en el almacén de la cuenta:

```powershell
$action  = New-ScheduledTaskAction -Execute "powershell.exe" `
    -Argument '-NoProfile -ExecutionPolicy Bypass -File "D:\Scripts\graph-import.ps1"'
$trigger = New-ScheduledTaskTrigger -Daily -At 06:00
Register-ScheduledTask -TaskName "eCall-Graph-Import" -Action $action -Trigger $trigger `
    -User "DOMAIN\svc-ecall" -Password (Read-Host "Passwort")
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `New-ScheduledTaskAction -Execute` | Programa que se va a ejecutar, aquí `powershell.exe` |
| `-Argument` | Línea de comandos del programa: `-NoProfile` omite los scripts de perfil, `-ExecutionPolicy Bypass` omite la directiva de scripts para esta llamada y `-File` indica el script |
| `New-ScheduledTaskTrigger -Daily -At 06:00` | Desencadenador diario a las 06:00 horas |
| `Register-ScheduledTask -TaskName` | Nombre de la tarea en el Programador de tareas |
| `-Action` / `-Trigger` | Vincula la acción y el desencadenador creados previamente con la tarea |
| `-User` | Cuenta con la que se ejecuta la tarea; la clave privada debe ser legible en su almacén de certificados |
| `-Password (Read-Host "Passwort")` | Solicita interactivamente la contraseña de la cuenta para que la tarea pueda iniciarse incluso sin un usuario conectado; así no queda almacenada en el script ni en el archivo de historial |

</details>

El ejemplo completo con registro y gestión de errores está disponible en GitHub: [pfstr/eCall-Log-Analyzer](https://github.com/pfstr/eCall-Log-Analyzer).

## Fuentes

1.  [Microsoft – «Retirement of Exchange Web Services in Exchange Online»](https://techcommunity.microsoft.com/blog/exchange/retirement-of-exchange-web-services-in-exchange-online/3924440): anuncio y fecha límite (1 de octubre de 2026) para el fin de EWS en Exchange Online.
    
2.  [Microsoft Learn – «Get access without a user (App-only)»](https://learn.microsoft.com/en-us/graph/auth-v2-service): autenticación solo de aplicación con certificado frente a Microsoft Graph.
    
3.  [Microsoft Learn – «Limiting application permissions to specific mailboxes»](https://learn.microsoft.com/en-us/graph/auth-limit-mailbox-access): Application Access Policy para restringir la aplicación a buzones individuales.
