---
title: "Establishing a Microsoft Graph Connection in PowerShell: Reading a Mailbox and Sending Emails"
navTitle: "Graph in PowerShell"
description: "EWS will be discontinued for Exchange Online on October 1, 2026. This step-by-step guide walks you through the entire process of setting up the Graph connection in PowerShell: app registration, certificate-based app-only sign-in, reading mailboxes, downloading ZIP attachments, and sending emails, with specific code for each step."
date: "2026-07-11"
kategorie: "Totemomail"
timeToRead: "5 min to read"
themen:
  - "microsoft-365-exchange"
slug: "microsoft-graph-powershell-mailbox-connection"
translationOf: "microsoft-graph-powershell-postfach-anbindung"
url: "https://rafaelpfister.ch/en/blog/microsoft-graph-powershell-mailbox-connection"
---

# Establishing a Microsoft Graph Connection in PowerShell: Reading a Mailbox and Sending Emails

Exchange Web Services (EWS) will be disabled for Exchange Online on **October 1, 2026**. Anyone who accesses mailboxes via script (for example, to retrieve automatically delivered files) must switch to the Microsoft Graph API. Setting up the complete Graph integration in PowerShell covers app registration, certificate-based authentication, reading and downloading attachments, and sending emails, each with the specific code.

A good example of this is an unattended script that downloads ZIP attachments from a mailbox, unpacks them, and then sends a report. Placeholders such as `example.com` and replace the tenant/app IDs with your own values.

## 1\. Requirements

Three modules of the Microsoft Graph SDK are sufficient, not the entire Meta module `Microsoft.Graph`:

```powershell
Install-Module Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions -Scope AllUsers
```

## 2\. App Registration in Entra ID

Unsupervised scripts do not log in a user, but rather an app with its own permissions (App-Only). In the [Log in to the Admin Center](https://entra.microsoft.com) Under "App registrations," create a new registration and grant it two permissions under "API permissions" → "Microsoft Graph" → "Application permissions":

-   `Mail.ReadWrite`: Read emails and move them after processing them
    
-   `Mail.Send`: Send the report email
    

After that **Grant admin consent** Click and make a note of the tenant ID and application (client) ID.

## 3\. Certificate Instead of Client Secret

App-Only authentication uses a client secret or a certificate. For scheduled tasks, the certificate is the cleaner option: The private key remains in the certificate store, and there is no password in the script. Generate the certificate on the server running the task and export the public key:

```powershell
$cert = New-SelfSignedCertificate -Subject "CN=eCall-Graph" `
    -CertStoreLocation "Cert:\LocalMachine\My" `
    -KeyExportPolicy NonExportable -KeySpec Signature `
    -KeyLength 2048 -NotAfter (Get-Date).AddYears(2)

Export-Certificate -Cert $cert -FilePath .\eCall-Graph.cer
$cert.Thumbprint   # -> im Skript als Thumbprint verwenden
```

The exported `.cer`Upload the file to the app registration under "Certificates & Secrets." The Scheduled Tasks account needs read access to the private key (`certlm.msc` → Certificate → All Tasks → Manage Private Keys).

## 4\. Restrict access to individual mailboxes

Application permissions otherwise apply tenant-wide: the app would be able to read every mailbox in the tenant. An Application Access Policy restricts it to a mail-enabled security group containing the permitted mailboxes (Exchange Online PowerShell, one-time setup):

```powershell
New-ApplicationAccessPolicy -AppId "<App-ID>" `
    -PolicyScopeGroupId "graph-mailboxes@example.com" `
    -AccessRight RestrictAccess `
    -Description "eCall Graph: nur Log-Postfach"

# Wirksamkeit pruefen
Test-ApplicationAccessPolicy -AppId "<App-ID>" -Identity "ecall-logs@example.com"
```

## 5\. Establish a connection

The login process uses the tenant ID, app ID, and certificate thumbprint, all without any user interaction:

```powershell
$TenantId   = "00000000-0000-0000-0000-000000000000"
$ClientId   = "00000000-0000-0000-0000-000000000000"
$Thumbprint = "0000000000000000000000000000000000000000"
$Mailbox    = "ecall-logs@example.com"

Import-Module Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions
Connect-MgGraph -TenantId $TenantId -ClientId $ClientId `
    -CertificateThumbprint $Thumbprint -NoWelcome
```

## 6\. Read emails and download ZIP attachments

The gist of it: Go through your inbox, save ZIP attachments, unzip them, and move the processed email to "Deleted Items." The key step is downloading via the `/$value`\-endpoint with `Invoke-MgGraphRequest -OutputFilePath`, which streams the raw content directly to a file and works reliably even with large attachments:

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

If there are more than 100 emails with `Get-MgUserMessage -All` or paging; one batch is usually sufficient for a monthly run.

## 7\. Send a report email via Graph

Also `Send-MailMessage` is outdated. Through the same app registration (the `Mail.Send` permission), the email goes out directly via Graph, in this case with a file as a base64-encoded attachment:

```powershell
$pfad = "D:\Reports
eport.csv"
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

## 8\. Run without supervision

When scheduled as a task, the script runs without requiring a login because the certificate is stored in the account's store:

```powershell
$action  = New-ScheduledTaskAction -Execute "powershell.exe" `
    -Argument '-NoProfile -ExecutionPolicy Bypass -File "D:\Scripts\graph-import.ps1"'
$trigger = New-ScheduledTaskTrigger -Daily -At 06:00
Register-ScheduledTask -TaskName "eCall-Graph-Import" -Action $action -Trigger $trigger `
    -User "DOMAIN\svc-ecall" -Password (Read-Host "Passwort")
```

The complete script (including logging, error handling, and the actual billing logic) is available as a working example on GitHub: [github.com/pfstr/eCall-Log-Analyzer](https://github.com/pfstr/eCall-Log-Analyzer)

## Sources

1.  [Microsoft – «Retirement of Exchange Web Services in Exchange Online»](https://techcommunity.microsoft.com/blog/exchange/retirement-of-exchange-web-services-in-exchange-online/3924440) — Announcement and end date (October 1, 2026) for the discontinuation of EWS in Exchange Online.
    
2.  [Microsoft Learn – «Get access without a user (App-only)»](https://learn.microsoft.com/en-us/graph/auth-v2-service) — App-only authentication against Microsoft Graph using a certificate.
    
3.  [Microsoft Learn – «Limiting application permissions to specific mailboxes»](https://learn.microsoft.com/en-us/graph/auth-limit-mailbox-access) — Application Access Policy to restrict the app to specific mailboxes.
