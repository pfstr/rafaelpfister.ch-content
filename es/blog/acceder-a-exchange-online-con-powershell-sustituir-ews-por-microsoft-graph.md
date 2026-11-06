---
title: "Acceder a Exchange Online con PowerShell: sustituir EWS por Microsoft Graph"
navTitle: "Graph en PowerShell"
description: "Microsoft retirará gradualmente EWS de Exchange Online a partir de octubre de 2026 y lo deshabilitará por completo en abril de 2027. Así puede registrar una aplicación, autenticar un script de PowerShell mediante certificado, limitar el acceso a buzones concretos y procesar mensajes y adjuntos con Microsoft Graph."
date: "2026-07-11"
kategorie: "Totemomail"
timeToRead: "5 min de lectura"
themen:
  - microsoft-365-exchange
slug: "acceder-a-exchange-online-con-powershell-sustituir-ews-por-microsoft-graph"
translationOf: "microsoft-graph-powershell-postfach-anbindung"
translationId: article-4c6a02c79b7bf0fe
translationReview: automatic
translationSourceHash: 38a149cf8f2933d92be1702594f7e41492b0a1924a56a20808172cc32be4a498
translatedAt: 2026-11-06T03:20:50.156Z
translationModel: gpt-5.6-terra
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/acceder-a-exchange-online-con-powershell-sustituir-ews-por-microsoft-graph
---

# Acceder a Exchange Online con PowerShell: sustituir EWS por Microsoft Graph

Microsoft retirará gradualmente Exchange Web Services (EWS) de Exchange Online a partir de **octubre de 2026**; en **abril de 2027**, EWS estará completamente deshabilitado. Por ello, los scripts que recuperan mensajes o adjuntos de un buzón deben migrar a Microsoft Graph.

El ejemplo de este artículo se ejecuta sin inicio de sesión de usuario: descarga adjuntos ZIP de un buzón, los descomprime, mueve los mensajes procesados y, a continuación, envía un informe. Para ello se necesita un registro de aplicación, un certificado y dos permisos de Graph. Valores como `example.com`, el ID de tenant y el ID de aplicación son marcadores de posición.

## 1. Módulos de PowerShell necesarios

Bastan tres módulos del SDK de Microsoft Graph, no el metamódulo completo `Microsoft.Graph`:

```powershell
Install-Module Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions -Scope AllUsers
```

<details class="options-details">
<summary>Explicación de las opciones</summary>

| Opción | Efecto |
|---|---|
| `Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions` | Los tres submódulos necesarios como argumento posicional `-Name`: autenticación, cmdlets de correo y acciones como `Send-MgUserMail` |
| `-Scope AllUsers` | Instala los módulos para todo el equipo en `Program Files`; necesario para que también estén disponibles para la cuenta de servicio configurada posteriormente para la tarea programada (requiere derechos de administrador) |

</details>

## 2\. Registro de aplicación en Entra ID

Un script desatendido se autentica como aplicación con sus propios permisos. Cree un nuevo registro en el [Centro de administración de Entra](https://entra.microsoft.com) en **App registrations** y asigne estos dos permisos en **API permissions → Microsoft Graph → Application permissions**:

-   `Mail.ReadWrite`: leer correos y moverlos tras procesarlos
    
-   `Mail.Send`: enviar el correo del informe
    
A continuación, conceda el consentimiento de administrador y anote el ID de tenant y el ID de aplicación (cliente).

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
<summary>Explicación de las opciones</summary>

| Opción | Efecto |
|---|---|
| `New-SelfSignedCertificate -Subject` | Nombre del sujeto del certificado; sirve únicamente para reconocerlo en el almacén de certificados |
| `-CertStoreLocation "Cert:\LocalMachine\My"` | Guarda el certificado en el almacén del equipo, no en el del usuario; así está disponible independientemente del usuario que haya iniciado sesión |
| `-KeyExportPolicy NonExportable` | Impide exportar la clave privada; no abandona el servidor |
| `-KeySpec Signature` | Crea una clave de firma; la autenticación de la aplicación la usa para firmar la aserción de solicitud de token |
| `-KeyLength 2048` | Longitud de clave RSA de 2048 bits |
| `-NotAfter (Get-Date).AddYears(2)` | Fecha de vencimiento dentro de dos años; después deberá renovarse y cargarse de nuevo el certificado |
| `Export-Certificate -Cert` | Objeto de certificado que se va a exportar |
| `-FilePath` | Archivo de destino; contiene, como `.cer`, solo la parte pública |

</details>

Cargue el archivo `.cer`\-exportado en el registro de aplicación, en Certificates & secrets. La cuenta de la tarea programada necesita permiso de lectura para la clave privada (`certlm.msc` → certificado → All Tasks → Manage Private Keys).

## 4\. Restringir el acceso a buzones concretos

Inicialmente, los permisos de aplicación se aplican a todos los buzones del tenant. Por ello, limite la aplicación mediante una Application Access Policy a un grupo de seguridad habilitado para correo. Este paso se ejecuta una sola vez en Exchange Online PowerShell. Microsoft considera ahora RBAC for Applications como sustituto de las Application Access Policies; allí, el ámbito se define mediante un Management Scope o una Administrative Unit y el permiso se concede en Exchange Online en lugar de Entra ID.

```powershell
New-ApplicationAccessPolicy -AppId "<App-ID>" `
    -PolicyScopeGroupId "graph-mailboxes@example.com" `
    -AccessRight RestrictAccess `
    -Description "eCall Graph: nur Log-Postfach"

# Comprobar la aplicación
Test-ApplicationAccessPolicy -AppId "<App-ID>" -Identity "ecall-logs@example.com"
```

<details class="options-details">
<summary>Explicación de las opciones</summary>

| Opción | Efecto |
|---|---|
| `New-ApplicationAccessPolicy -AppId` | ID de aplicación (cliente) del registro de aplicación al que se aplica la directiva |
| `-PolicyScopeGroupId` | Grupo de seguridad habilitado para correo cuyos miembros definen el ámbito |
| `-AccessRight RestrictAccess` | Restringe la aplicación a los buzones del grupo; la alternativa `DenyAccess` bloquearía precisamente estos buzones |
| `-Description` | Texto libre para documentar la directiva |
| `Test-ApplicationAccessPolicy -Identity` | Comprueba si la aplicación puede acceder a un buzón concreto (`AccessCheckResult: Granted` o `Denied`) |

</details>

## 5\. Establecer la conexión

La autenticación utiliza el ID de tenant, el ID de aplicación y la huella digital del certificado, sin ninguna interacción del usuario:

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
<summary>Explicación de las opciones</summary>

| Opción | Efecto |
|---|---|
| `Connect-MgGraph -TenantId` | Tenant en el que se autentica la aplicación |
| `-ClientId` | ID de aplicación (cliente) del registro de aplicación |
| `-CertificateThumbprint` | Selecciona el certificado de autenticación del almacén local de certificados mediante su huella digital; la combinación de `-ClientId` y certificado permite una autenticación solo de aplicación sin usuario |
| `-NoWelcome` | Suprime el mensaje de bienvenida después de la autenticación; útil para salidas de scripts y registros |

</details>

## 6. Leer mensajes y descargar adjuntos ZIP

Ahora el script puede recorrer la bandeja de entrada, guardar y descomprimir adjuntos ZIP, y mover los mensajes procesados a «Elementos eliminados». La descarga se realiza mediante el endpoint `/$value` y `Invoke-MgGraphRequest -OutputFilePath`. Así, el contenido sin procesar llega directamente a un archivo, sin mantener un adjunto grande completo en la memoria:

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
<summary>Explicación de las opciones</summary>

| Opción | Efecto |
|---|---|
| `Add-Type -AssemblyName System.IO.Compression.FileSystem` | Carga el ensamblado .NET con la clase `ZipFile` para descomprimir |
| `Get-MgUserMessage -UserId` | Buzón cuyos mensajes se leen; es obligatorio indicarlo en la autenticación solo de aplicación |
| `-Top 100` | Limita la consulta a un máximo de 100 mensajes por llamada |
| `-Property id, subject, hasAttachments` | Solicita solo los campos necesarios; esto reduce la respuesta y acelera la llamada |
| `Get-MgUserMessageAttachment -MessageId` | Mensaje cuyos adjuntos se enumeran |
| `Invoke-MgGraphRequest -Method GET` | Método HTTP de la llamada directa a la API de Graph |
| `-Uri` | Endpoint invocado; el `/$value` añadido proporciona el contenido de archivo sin procesar del adjunto en lugar de un objeto JSON |
| `-OutputFilePath` | Escribe la respuesta directamente en el archivo de destino sin mantener el adjunto completo en la memoria |
| `Move-MgUserMessage -DestinationId "deleteditems"` | Mueve el mensaje procesado a la carpeta de destino; `deleteditems` es el nombre de carpeta conocido para «Elementos eliminados» |

</details>

Con más de 100 correos, trabaje con `Get-MgUserMessage -All` o paginación; para una ejecución mensual normalmente basta un lote.

## 7\. Enviar el correo del informe mediante Graph

También `Send-MailMessage` está obsoleto. Con el mismo registro de aplicación (permiso `Mail.Send`), el correo se envía directamente mediante Graph, aquí con un archivo como adjunto codificado en base64:

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
<summary>Explicación de las opciones</summary>

| Opción | Efecto |
|---|---|
| `-UserId` | Buzón en cuyo nombre se envía el correo; debe estar cubierto por el ámbito de la Application Access Policy |
| `-BodyParameter` | El mensaje completo como hashtable en el esquema de Graph: `message` con asunto, cuerpo, destinatarios y adjuntos, así como `saveToSentItems` para guardarlo en «Elementos enviados» |

</details>

## 8\. Ejecutar de forma desatendida

Como tarea programada, el script se ejecuta sin autenticación interactiva porque el certificado está en el almacén de la cuenta:

```powershell
$action  = New-ScheduledTaskAction -Execute "powershell.exe" `
    -Argument '-NoProfile -ExecutionPolicy Bypass -File "D:\Scripts\graph-import.ps1"'
$trigger = New-ScheduledTaskTrigger -Daily -At 06:00
Register-ScheduledTask -TaskName "eCall-Graph-Import" -Action $action -Trigger $trigger `
    -User "DOMAIN\svc-ecall" -Password (Read-Host "Passwort")
```

<details class="options-details">
<summary>Explicación de las opciones</summary>

| Opción | Efecto |
|---|---|
| `New-ScheduledTaskAction -Execute` | Programa que se ejecutará, aquí `powershell.exe` |
| `-Argument` | Línea de comandos del programa: `-NoProfile` omite scripts de perfil, `-ExecutionPolicy Bypass` omite la directiva de scripts para esta llamada y `-File` indica el script |
| `New-ScheduledTaskTrigger -Daily -At 06:00` | Desencadenador diario a las 06:00 |
| `Register-ScheduledTask -TaskName` | Nombre de la tarea en el Programador de tareas |
| `-Action` / `-Trigger` | Vincula la acción y el desencadenador creados anteriormente con la tarea |
| `-User` | Cuenta con la que se ejecuta la tarea; la clave privada debe poder leerse desde su almacén de certificados |
| `-Password (Read-Host "Passwort")` | Solicita interactivamente la contraseña de la cuenta para que la tarea pueda iniciarse también sin un usuario conectado; así no queda en el script ni en el archivo de historial |

</details>

El ejemplo completo, con registro y gestión de errores, está disponible en GitHub: [pfstr/ecall-log-analyzer](https://github.com/pfstr/ecall-log-analyzer).

## Historial de cambios

- **25 de octubre de 2026:** Corregido: EWS en Exchange Online se retirará gradualmente desde octubre de 2026 y no se deshabilitará por completo hasta abril de 2027. Añadido: RBAC for Applications como sustituto de Microsoft para las Application Access Policies, con una nueva fuente y el calendario actual.

## Fuentes

1.  [Microsoft – «Retirement of Exchange Web Services in Exchange Online»](https://techcommunity.microsoft.com/blog/exchange/retirement-of-exchange-web-services-in-exchange-online/3924440): Anuncio de 2023 con la fecha límite original de octubre de 2026 para el fin de EWS en Exchange Online.
    
2.  [Microsoft Learn – «Get access without a user (App-only)»](https://learn.microsoft.com/en-us/graph/auth-v2-service): Autenticación solo de aplicación con certificado para Microsoft Graph.
    
3.  [Microsoft Learn – «Role Based Access Control for Applications in Exchange Online»](https://learn.microsoft.com/en-us/exchange/permissions-exo/application-rbac): Restricción de una aplicación a buzones concretos; RBAC for Applications sustituye a las Application Access Policies, incluidos los pasos de migración.
    
4.  [Microsoft Learn – «Deprecation of Exchange Web Services in Exchange Online»](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/deprecation-of-ews-exchange-online): Calendario actual, retirada gradual desde octubre de 2026 y deshabilitación completa en abril de 2027.
