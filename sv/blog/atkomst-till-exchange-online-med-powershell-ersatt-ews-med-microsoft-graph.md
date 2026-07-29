---
title: "Åtkomst till Exchange Online med PowerShell: ersätt EWS med Microsoft Graph"
navTitle: "Graph i PowerShell"
description: "EWS upphör i Exchange Online den 1 oktober 2026. Så registrerar du en app, autentiserar ett PowerShell-skript med certifikat, begränsar åtkomsten till enskilda postlådor och bearbetar meddelanden och bilagor via Microsoft Graph."
date: "2026-07-11"
kategorie: "Totemomail"
timeToRead: "5 min lästid"
themen:
  - microsoft-365-exchange
slug: "atkomst-till-exchange-online-med-powershell-ersatt-ews-med-microsoft-graph"
translationOf: "microsoft-graph-powershell-postfach-anbindung"
url: "https://rafaelpfister.ch/sv/blog/atkomst-till-exchange-online-med-powershell-ersatt-ews-med-microsoft-graph"
translationId: article-4c6a02c79b7bf0fe
translationReview: automatic
translationSourceHash: bab6aabe691a64409231665e5d2dd2288fb409fa66b4a7f33cfcb79aeaa643b3
translatedAt: 2026-07-29T12:29:38.957Z
---

# Åtkomst till Exchange Online med PowerShell: ersätt EWS med Microsoft Graph

Microsoft avvecklar Exchange Web Services (EWS) i Exchange Online den **1 oktober 2026**. Skript som hämtar meddelanden eller bilagor från en postlåda måste därför migrera till Microsoft Graph.

Exemplet i det här inlägget körs utan användarinloggning: det laddar ned ZIP-bilagor från en postlåda, packar upp dem, flyttar bearbetade meddelanden och skickar därefter en rapport. Det kräver en appregistrering, ett certifikat och två Graph-behörigheter. Värden som `example.com`, klient-ID och app-ID är platshållare.

## 1. Nödvändiga PowerShell-moduler

Tre moduler från Microsoft Graph SDK räcker, inte hela metapaketet `Microsoft.Graph`:

```powershell
Install-Module Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions -Scope AllUsers
```

## 2\. Appregistrering i Entra ID

Ett obevakat skript autentiserar sig som en applikation med egna behörigheter. Skapa en ny registrering i [Entra Admin Center](https://entra.microsoft.com) under **App registrations** och tilldela följande två behörigheter under **API permissions → Microsoft Graph → Application permissions**:

-   `Mail.ReadWrite`: läsa e-post och flytta den efter bearbetning
    
-   `Mail.Send`: skicka rapportmeddelandet
    

Ge sedan administratörsgodkännande och anteckna klient-ID samt Application (Client) ID.

## 3\. Certifikat i stället för klienthemlighet

En app-only-autentisering fungerar med klienthemlighet eller certifikat. För schemalagda uppgifter är ett certifikat det bättre valet: den privata nyckeln stannar i certifikatarkivet och skriptet innehåller inget lösenord. Skapa certifikatet på servern som kör skriptet och exportera endast den offentliga delen:

```powershell
$cert = New-SelfSignedCertificate -Subject "CN=eCall-Graph" `
    -CertStoreLocation "Cert:\LocalMachine\My" `
    -KeyExportPolicy NonExportable -KeySpec Signature `
    -KeyLength 2048 -NotAfter (Get-Date).AddYears(2)

Export-Certificate -Cert $cert -FilePath .\eCall-Graph.cer
$cert.Thumbprint   # -> använd som tumavtryck i skriptet
```

Ladda upp den exporterade `.cer`\-filen i appregistreringen under Certificates & secrets. Kontot för den schemalagda uppgiften behöver läsbehörighet till den privata nyckeln (`certlm.msc` → certifikat → All Tasks → Manage Private Keys).

## 4\. Begränsa åtkomsten till enskilda postlådor

Application Permissions gäller först för alla postlådor i klientorganisationen. Begränsa därför appen med en Application Access Policy till en e-postaktiverad säkerhetsgrupp. Detta steg utförs en gång i Exchange Online PowerShell:

```powershell
New-ApplicationAccessPolicy -AppId "<App-ID>" `
    -PolicyScopeGroupId "graph-mailboxes@example.com" `
    -AccessRight RestrictAccess `
    -Description "eCall Graph: endast loggpostlåda"

# Kontrollera att den fungerar
Test-ApplicationAccessPolicy -AppId "<App-ID>" -Identity "ecall-logs@example.com"
```

## 5\. Upprätta anslutning

Autentiseringen använder klient-ID, app-ID och certifikatets tumavtryck, helt utan användarinteraktion:

```powershell
$TenantId   = "00000000-0000-0000-0000-000000000000"
$ClientId   = "00000000-0000-0000-0000-000000000000"
$Thumbprint = "0000000000000000000000000000000000000000"
$Mailbox    = "ecall-logs@example.com"

Import-Module Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions
Connect-MgGraph -TenantId $TenantId -ClientId $ClientId `
    -CertificateThumbprint $Thumbprint -NoWelcome
```

## 6. Läs meddelanden och hämta ZIP-bilagor

Nu kan skriptet gå igenom inkorgen, spara och packa upp ZIP-bilagor samt flytta bearbetade meddelanden till «Borttagna objekt». Nedladdningen sker via slutpunkten `/$value` och `Invoke-MgGraphRequest -OutputFilePath`. Därmed hamnar råinnehållet direkt i en fil, utan att en stor bilaga måste hållas helt i arbetsminnet:

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

Vid fler än 100 e-postmeddelanden, arbeta med `Get-MgUserMessage -All` eller sidindelning; för en månadskörning räcker oftast en batch.

## 7\. Skicka rapportmeddelande via Graph

Även `Send-MailMessage` är föråldrat. Med samma appregistrering (behörighet `Mail.Send`) skickas e-postmeddelandet direkt via Graph, här med en fil som base64-kodad bilaga:

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

## 8\. Kör obevakat

Som en schemalagd uppgift körs skriptet utan inloggning, eftersom certifikatet finns i kontots arkiv:

```powershell
$action  = New-ScheduledTaskAction -Execute "powershell.exe" `
    -Argument '-NoProfile -ExecutionPolicy Bypass -File "D:\Scripts\graph-import.ps1"'
$trigger = New-ScheduledTaskTrigger -Daily -At 06:00
Register-ScheduledTask -TaskName "eCall-Graph-Import" -Action $action -Trigger $trigger `
    -User "DOMAIN\svc-ecall" -Password (Read-Host "Lösenord")
```

Det fullständiga exemplet med loggning och felhantering finns på GitHub: [pfstr/eCall-Log-Analyzer](https://github.com/pfstr/eCall-Log-Analyzer).

## Källor

1.  [Microsoft – «Retirement of Exchange Web Services in Exchange Online»](https://techcommunity.microsoft.com/blog/exchange/retirement-of-exchange-web-services-in-exchange-online/3924440): Tillkännagivande och brytdatum (1 oktober 2026) för slutet på EWS i Exchange Online.
    
2.  [Microsoft Learn – «Get access without a user (App-only)»](https://learn.microsoft.com/en-us/graph/auth-v2-service): App-only-autentisering mot Microsoft Graph med certifikat.
    
3.  [Microsoft Learn – «Limiting application permissions to specific mailboxes»](https://learn.microsoft.com/en-us/graph/auth-limit-mailbox-access): Application Access Policy för att begränsa appen till enskilda postlådor.
