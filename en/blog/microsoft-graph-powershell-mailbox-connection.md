---
title: "Accessing Exchange Online with PowerShell: Replacing EWS with Microsoft Graph"
navTitle: "Graph in PowerShell"
description: "EWS ends in Exchange Online on 1 October 2026. Here is how to register an app, authenticate a PowerShell script using a certificate, restrict access to individual mailboxes, and process messages and attachments via Microsoft Graph."
date: "2026-07-11"
kategorie: "Totemomail"
timeToRead: "5 min read"
themen:
  - "microsoft-365-exchange"
slug: "microsoft-graph-powershell-mailbox-connection"
translationOf: "microsoft-graph-powershell-postfach-anbindung"
url: "https://rafaelpfister.ch/en/blog/microsoft-graph-powershell-mailbox-connection"
---

# Accessing Exchange Online with PowerShell: Replacing EWS with Microsoft Graph

Microsoft is retiring Exchange Web Services (EWS) in Exchange Online on **1 October 2026**. Scripts that retrieve messages or attachments from a mailbox must therefore move to Microsoft Graph.

The example in this article runs without user sign-in: it downloads ZIP attachments from a mailbox, extracts them, moves processed messages, and then sends a report. It requires an app registration, a certificate, and two Graph permissions. Values such as `example.com`, tenant ID and app ID are placeholders.

## 1. Required PowerShell modules

Three Microsoft Graph SDK modules are sufficient; the entire `Microsoft.Graph` meta-module is not required:

```powershell
Install-Module Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions -Scope AllUsers
```

## 2\. App registration in Entra ID

An unattended script signs in as an application with its own permissions. Create a new registration in the [Entra Admin Centre](https://entra.microsoft.com) under **App registrations** and assign these two permissions under **API permissions → Microsoft Graph → Application permissions**:

-   `Mail.ReadWrite`: read emails and move them after processing
    
-   `Mail.Send`: send the report email
    

Then grant admin consent and note down the tenant ID and Application (Client) ID.

## 3\. A certificate instead of a client secret

An app-only sign-in works with a client secret or certificate. A certificate is the better choice for scheduled tasks: the private key remains in the certificate store, so the script contains no password. Create the certificate on the server that will run the script and export only the public part:

```powershell
$cert = New-SelfSignedCertificate -Subject "CN=eCall-Graph" `
    -CertStoreLocation "Cert:\LocalMachine\My" `
    -KeyExportPolicy NonExportable -KeySpec Signature `
    -KeyLength 2048 -NotAfter (Get-Date).AddYears(2)

Export-Certificate -Cert $cert -FilePath .\eCall-Graph.cer
$cert.Thumbprint   # -> use as the thumbprint in the script
```

Upload the exported `.cer`\-file to the app registration under Certificates & secrets. The Scheduled Task account requires read access to the private key (`certlm.msc` → certificate → All Tasks → Manage Private Keys).

## 4\. Restrict access to individual mailboxes

Initially, application permissions apply to all mailboxes in the tenant. Therefore, restrict the app to a mail-enabled security group using an Application Access Policy. Run this step once in Exchange Online PowerShell:

```powershell
New-ApplicationAccessPolicy -AppId "<App-ID>" `
    -PolicyScopeGroupId "graph-mailboxes@example.com" `
    -AccessRight RestrictAccess `
    -Description "eCall Graph: log mailbox only"

# Check effectiveness
Test-ApplicationAccessPolicy -AppId "<App-ID>" -Identity "ecall-logs@example.com"
```

## 5\. Establish the connection

Sign-in uses the tenant ID, app ID and certificate thumbprint, entirely without user interaction:

```powershell
$TenantId   = "00000000-0000-0000-0000-000000000000"
$ClientId   = "00000000-0000-0000-0000-000000000000"
$Thumbprint = "0000000000000000000000000000000000000000"
$Mailbox    = "ecall-logs@example.com"

Import-Module Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions
Connect-MgGraph -TenantId $TenantId -ClientId $ClientId `
    -CertificateThumbprint $Thumbprint -NoWelcome
```

## 6. Read messages and download ZIP attachments

The script can now process the inbox, save and extract ZIP attachments, and move processed messages to “Deleted Items”. Downloads use the `/$value` endpoint and `Invoke-MgGraphRequest -OutputFilePath`. This writes the raw content directly to a file without keeping a large attachment entirely in memory:

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

For more than 100 emails, work with `Get-MgUserMessage -All` or paging; one batch is usually sufficient for a monthly run.

## 7\. Send a report email via Graph

`Send-MailMessage` is also obsolete. Using the same app registration (permission `Mail.Send`), the email is sent directly through Graph, here with a file as a base64-encoded attachment:

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

## 8\. Run unattended

As a scheduled task, the script runs without sign-in because the certificate is stored in the account’s certificate store:

```powershell
$action  = New-ScheduledTaskAction -Execute "powershell.exe" `
    -Argument '-NoProfile -ExecutionPolicy Bypass -File "D:\Scripts\graph-import.ps1"'
$trigger = New-ScheduledTaskTrigger -Daily -At 06:00
Register-ScheduledTask -TaskName "eCall-Graph-Import" -Action $action -Trigger $trigger `
    -User "DOMAIN\svc-ecall" -Password (Read-Host "Password")
```

The complete example with logging and error handling is available on GitHub: [pfstr/eCall-Log-Analyzer](https://github.com/pfstr/eCall-Log-Analyzer).

## Sources

1.  [Microsoft – “Retirement of Exchange Web Services in Exchange Online”](https://techcommunity.microsoft.com/blog/exchange/retirement-of-exchange-web-services-in-exchange-online/3924440): announcement and deadline (1 October 2026) for the end of EWS in Exchange Online.
    
2.  [Microsoft Learn – “Get access without a user (App-only)”](https://learn.microsoft.com/en-us/graph/auth-v2-service): app-only authentication to Microsoft Graph using a certificate.
    
3.  [Microsoft Learn – “Limiting application permissions to specific mailboxes”](https://learn.microsoft.com/en-us/graph/auth-limit-mailbox-access): Application Access Policy for restricting the app to individual mailboxes.
