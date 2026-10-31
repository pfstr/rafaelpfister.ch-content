---
title: "Access Exchange Online with PowerShell: Replace EWS with Microsoft Graph"
navTitle: "Graph in PowerShell"
description: "Microsoft has been gradually retiring EWS in Exchange Online since October 2026, with full deactivation in April 2027. Learn how to register an app, authenticate a PowerShell script with a certificate, restrict access to individual mailboxes, and process messages and attachments through Microsoft Graph."
date: "2026-07-11"
kategorie: "TotemoMail"
timeToRead: "5 min read"
themen:
  - microsoft-365-exchange
slug: "microsoft-graph-powershell-mailbox-connection"
translationOf: "microsoft-graph-powershell-postfach-anbindung"
translationId: article-4c6a02c79b7bf0fe
translatedAt: 2026-10-31T03:45:15.092Z
translationReview: required
translationSourceHash: 38a149cf8f2933d92be1702594f7e41492b0a1924a56a20808172cc32be4a498
translationModel: gpt-5.6-terra
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/en/blog/microsoft-graph-powershell-mailbox-connection
---

# Access Exchange Online with PowerShell: Replace EWS with Microsoft Graph

Microsoft has been gradually retiring Exchange Web Services (EWS) in Exchange Online since **October 2026**; EWS will be fully disabled in **April 2027**. Scripts that retrieve messages or attachments from a mailbox must therefore switch to Microsoft Graph.

The example in this article runs without user sign-in: it downloads ZIP attachments from a mailbox, extracts them, moves processed messages, and then sends a report. This requires an app registration, a certificate, and two Graph permissions. Values such as `example.com`, tenant ID, and app ID are placeholders.

## 1. Required PowerShell modules

Three modules from the Microsoft Graph SDK are sufficient; you do not need the entire meta-module `Microsoft.Graph`:

```powershell
Install-Module Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions -Scope AllUsers
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions` | The three required submodules as a positional argument `-Name`: authentication, mail cmdlets, and actions such as `Send-MgUserMail` |
| `-Scope AllUsers` | Installs the modules machine-wide under `Program Files`; required so they are also available to the service account configured later for the Scheduled Task (requires administrator privileges) |

</details>

## 2\. App registration in Entra ID

An unattended script signs in as an application with its own permissions. Create a new registration in the [Entra Admin Center](https://entra.microsoft.com) under **App registrations**, then assign these two permissions under **API permissions → Microsoft Graph → Application permissions**:

-   `Mail.ReadWrite`: read emails and move them after processing
    
-   `Mail.Send`: send the report email
    

Then grant administrator consent and note the tenant ID and Application (Client) ID.

## 3\. Certificate instead of client secret

An app-only sign-in works with a client secret or a certificate. A certificate is the better choice for scheduled tasks: the private key remains in the certificate store, and the script contains no password. Create the certificate on the server that runs the script and export only the public portion:

```powershell
$cert = New-SelfSignedCertificate -Subject "CN=eCall-Graph" `
    -CertStoreLocation "Cert:\LocalMachine\My" `
    -KeyExportPolicy NonExportable -KeySpec Signature `
    -KeyLength 2048 -NotAfter (Get-Date).AddYears(2)

Export-Certificate -Cert $cert -FilePath .\eCall-Graph.cer
$cert.Thumbprint   # -> im Skript als Thumbprint verwenden
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `New-SelfSignedCertificate -Subject` | Certificate subject name; used only for recognition in the certificate store |
| `-CertStoreLocation "Cert:\LocalMachine\My"` | Stores the certificate in the computer store rather than the user store; this makes it available regardless of the signed-in user |
| `-KeyExportPolicy NonExportable` | Prevents export of the private key; it never leaves the server |
| `-KeySpec Signature` | Creates a signing key; the app sign-in uses it to sign the token request assertion |
| `-KeyLength 2048` | RSA key length of 2048 bits |
| `-NotAfter (Get-Date).AddYears(2)` | Expiration date in two years; the certificate must then be renewed and uploaded again |
| `Export-Certificate -Cert` | Certificate object to export |
| `-FilePath` | Destination file; as `.cer`, it contains only the public portion |

</details>

Upload the exported `.cer`\-file in the app registration under Certificates & secrets. The Scheduled Task account needs read access to the private key (`certlm.msc` → certificate → All Tasks → Manage Private Keys).

## 4\. Restrict access to individual mailboxes

Application permissions initially apply to all mailboxes in the tenant. Therefore, restrict the app using an Application Access Policy for a mail-enabled security group. Perform this step once in Exchange Online PowerShell. Microsoft now refers to RBAC for Applications as the replacement for Application Access Policies; there, the scope is defined through a Management Scope or an Administrative Unit, and permission is granted in Exchange Online rather than Entra ID.

```powershell
New-ApplicationAccessPolicy -AppId "<App-ID>" `
    -PolicyScopeGroupId "graph-mailboxes@example.com" `
    -AccessRight RestrictAccess `
    -Description "eCall Graph: nur Log-Postfach"

# Check effectiveness
Test-ApplicationAccessPolicy -AppId "<App-ID>" -Identity "ecall-logs@example.com"
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `New-ApplicationAccessPolicy -AppId` | Application (Client) ID of the app registration to which the policy applies |
| `-PolicyScopeGroupId` | Mail-enabled security group whose members define the scope |
| `-AccessRight RestrictAccess` | Restricts the app to the group’s mailboxes; the alternative `DenyAccess` would block precisely those mailboxes |
| `-Description` | Free text for documenting the policy |
| `Test-ApplicationAccessPolicy -Identity` | Checks whether the app may access a specific mailbox (`AccessCheckResult: Granted` or `Denied`) |

</details>

## 5\. Establish a connection

Sign-in uses the tenant ID, app ID, and certificate thumbprint, entirely without user interaction:

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
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Connect-MgGraph -TenantId` | Tenant against which the app signs in |
| `-ClientId` | Application (Client) ID of the app registration |
| `-CertificateThumbprint` | Selects the sign-in certificate by its thumbprint from the local certificate store; the combination of `-ClientId` and certificate provides an app-only sign-in without a user |
| `-NoWelcome` | Suppresses the welcome text after sign-in; useful for script output and logs |

</details>

## 6. Read messages and download ZIP attachments

The script can now process the inbox, save and extract ZIP attachments, and move processed messages to “Deleted Items.” The download uses the endpoint `/$value` and `Invoke-MgGraphRequest -OutputFilePath`. This writes the raw content directly to a file without holding a large attachment entirely in memory:

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

    # Move processed email to "Deleted Items"
    Move-MgUserMessage -UserId $Mailbox -MessageId $msg.Id -DestinationId "deleteditems" | Out-Null
}
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Add-Type -AssemblyName System.IO.Compression.FileSystem` | Loads the .NET assembly containing the `ZipFile` class for extraction |
| `Get-MgUserMessage -UserId` | Mailbox whose messages are read; specifying it is required for app-only sign-in |
| `-Top 100` | Limits the query to a maximum of 100 messages per call |
| `-Property id, subject, hasAttachments` | Requests only the required fields; this reduces the response size and speeds up the call |
| `Get-MgUserMessageAttachment -MessageId` | Message whose attachments are listed |
| `Invoke-MgGraphRequest -Method GET` | HTTP method for the direct call to the Graph API |
| `-Uri` | Endpoint called; the appended `/$value` returns the attachment’s raw file content rather than a JSON object |
| `-OutputFilePath` | Writes the response directly to the destination file without retaining the entire attachment in memory |
| `Move-MgUserMessage -DestinationId "deleteditems"` | Moves the processed message to the destination folder; `deleteditems` is the well-known folder name for “Deleted Items” |

</details>

For more than 100 emails, work with `Get-MgUserMessage -All` or paging; one batch is usually sufficient for a monthly run.

## 7\. Send report email through Graph

`Send-MailMessage` is also deprecated. Using the same app registration (permission `Mail.Send`), the email is sent directly through Graph, here with a file as a base64-encoded attachment:

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
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-UserId` | Mailbox on whose behalf the email is sent; it must be covered by the scope of the Application Access Policy |
| `-BodyParameter` | The complete message as a hashtable in the Graph schema: `message` with subject, body, recipients, and attachments, plus `saveToSentItems` for saving it in “Sent Items” |

</details>

## 8\. Run unattended

As a scheduled task, the script runs without sign-in because the certificate is in the account’s store:

```powershell
$action  = New-ScheduledTaskAction -Execute "powershell.exe" `
    -Argument '-NoProfile -ExecutionPolicy Bypass -File "D:\Scripts\graph-import.ps1"'
$trigger = New-ScheduledTaskTrigger -Daily -At 06:00
Register-ScheduledTask -TaskName "eCall-Graph-Import" -Action $action -Trigger $trigger `
    -User "DOMAIN\svc-ecall" -Password (Read-Host "Passwort")
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `New-ScheduledTaskAction -Execute` | Program to run, here `powershell.exe` |
| `-Argument` | Command line for the program: `-NoProfile` skips profile scripts, `-ExecutionPolicy Bypass` bypasses the script policy for this invocation, and `-File` specifies the script |
| `New-ScheduledTaskTrigger -Daily -At 06:00` | Daily trigger at 6:00 AM |
| `Register-ScheduledTask -TaskName` | Task name in Task Scheduler |
| `-Action` / `-Trigger` | Links the previously created action and trigger to the task |
| `-User` | Account under which the task runs; the private key must be readable in its certificate store |
| `-Password (Read-Host "Passwort")` | Prompts interactively for the account password so the task can start even without a signed-in user; this keeps it out of the script and history file |

</details>

The complete example with logging and error handling is available on GitHub: [pfstr/ecall-log-analyzer](https://github.com/pfstr/ecall-log-analyzer).

## Change log

- **October 25, 2026:** Corrected: EWS in Exchange Online has been gradually retired since October 2026 and will not be fully disabled until April 2027. Added: RBAC for Applications as Microsoft’s replacement for Application Access Policies, with a new source and current timeline.

## Sources

1.  [Microsoft – “Retirement of Exchange Web Services in Exchange Online”](https://techcommunity.microsoft.com/blog/exchange/retirement-of-exchange-web-services-in-exchange-online/3924440): 2023 announcement with the original October 2026 deadline for the end of EWS in Exchange Online.
    
2.  [Microsoft Learn – “Get access without a user (App-only)”](https://learn.microsoft.com/en-us/graph/auth-v2-service): app-only authentication to Microsoft Graph using a certificate.
    
3.  [Microsoft Learn – “Role Based Access Control for Applications in Exchange Online”](https://learn.microsoft.com/en-us/exchange/permissions-exo/application-rbac): restricting an app to individual mailboxes; RBAC for Applications replaces Application Access Policies, including migration steps.
    
4.  [Microsoft Learn – “Deprecation of Exchange Web Services in Exchange Online”](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/deprecation-of-ews-exchange-online): current timeline, gradual retirement beginning in October 2026, and full deactivation in April 2027.
