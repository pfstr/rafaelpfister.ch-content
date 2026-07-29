---
title: "Få tilgang til Exchange Online med PowerShell: erstatt EWS med Microsoft Graph"
navTitle: "Graph i PowerShell"
description: "EWS avvikles i Exchange Online 1. oktober 2026. Slik registrerer du en app, autentiserer et PowerShell-skript med sertifikat, begrenser tilgangen til enkelte postbokser og behandler meldinger og vedlegg via Microsoft Graph."
date: "2026-07-11"
kategorie: "Totemomail"
timeToRead: "5 min lesetid"
themen:
  - microsoft-365-exchange
slug: "fa-tilgang-til-exchange-online-med-powershell-erstatt-ews-med-microsoft-graph"
translationOf: "microsoft-graph-powershell-postfach-anbindung"
url: "https://rafaelpfister.ch/no/blog/fa-tilgang-til-exchange-online-med-powershell-erstatt-ews-med-microsoft-graph"
translationId: article-4c6a02c79b7bf0fe
translationReview: automatic
translationSourceHash: bab6aabe691a64409231665e5d2dd2288fb409fa66b4a7f33cfcb79aeaa643b3
translatedAt: 2026-07-29T12:29:38.967Z
---

# Få tilgang til Exchange Online med PowerShell: erstatt EWS med Microsoft Graph

Microsoft avvikler Exchange Web Services (EWS) i Exchange Online **1. oktober 2026**. Skript som henter meldinger eller vedlegg fra en postboks, må derfor bytte til Microsoft Graph.

Eksempelet i dette innlegget kjører uten brukerinnlogging: Det laster ned ZIP-vedlegg fra en postboks, pakker dem ut, flytter behandlede meldinger og sender deretter en rapport. Dette krever en appregistrering, et sertifikat og to Graph-tillatelser. Verdier som `example.com`, tenant-ID og app-ID er plassholdere.

## 1. Nødvendige PowerShell-moduler

Tre moduler fra Microsoft Graph SDK er tilstrekkelig, ikke hele metapakken `Microsoft.Graph`:

```powershell
Install-Module Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions -Scope AllUsers
```

## 2\. Appregistrering i Entra ID

Et uovervåket skript logger på som en applikasjon med egne rettigheter. Opprett en ny registrering i [Entra Admin Center](https://entra.microsoft.com) under **App registrations**, og tildel disse to rettighetene under **API permissions → Microsoft Graph → Application permissions**:

-   `Mail.ReadWrite`: lese e-post og flytte den etter behandling
    
-   `Mail.Send`: sende rapport-e-posten
    

Gi deretter administratortilslutning, og noter tenant-ID samt Application (Client) ID.

## 3\. Sertifikat i stedet for klienthemmelighet

En app-only-innlogging fungerer med klienthemmelighet eller sertifikat. For planlagte oppgaver er et sertifikat et bedre valg: Den private nøkkelen forblir i sertifikatlageret, og skriptet inneholder ikke noe passord. Opprett sertifikatet på serveren som skal kjøre oppgaven, og eksporter kun den offentlige delen:

```powershell
$cert = New-SelfSignedCertificate -Subject "CN=eCall-Graph" `
    -CertStoreLocation "Cert:\LocalMachine\My" `
    -KeyExportPolicy NonExportable -KeySpec Signature `
    -KeyLength 2048 -NotAfter (Get-Date).AddYears(2)

Export-Certificate -Cert $cert -FilePath .\eCall-Graph.cer
$cert.Thumbprint   # -> bruk som Thumbprint i skriptet
```

Last opp den eksporterte `.cer`\-filen i appregistreringen under Certificates & secrets. Kontoen for den planlagte oppgaven trenger lesetilgang til den private nøkkelen (`certlm.msc` → sertifikat → All Tasks → Manage Private Keys).

## 4\. Begrens tilgang til enkelte postbokser

Application Permissions gjelder i utgangspunktet alle postbokser i tenant-en. Begrens derfor appen med en Application Access Policy til en e-postaktivert sikkerhetsgruppe. Dette trinnet utføres én gang i Exchange Online PowerShell:

```powershell
New-ApplicationAccessPolicy -AppId "<App-ID>" `
    -PolicyScopeGroupId "graph-mailboxes@example.com" `
    -AccessRight RestrictAccess `
    -Description "eCall Graph: kun loggpostboks"

# Kontroller at den virker
Test-ApplicationAccessPolicy -AppId "<App-ID>" -Identity "ecall-logs@example.com"
```

## 5\. Opprett tilkobling

Innloggingen bruker tenant-ID, app-ID og sertifikatets fingeravtrykk, helt uten brukerinteraksjon:

```powershell
$TenantId   = "00000000-0000-0000-0000-000000000000"
$ClientId   = "00000000-0000-0000-0000-000000000000"
$Thumbprint = "0000000000000000000000000000000000000000"
$Mailbox    = "ecall-logs@example.com"

Import-Module Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions
Connect-MgGraph -TenantId $TenantId -ClientId $ClientId `
    -CertificateThumbprint $Thumbprint -NoWelcome
```

## 6. Les meldinger og last ned ZIP-vedlegg

Skriptet kan nå gå gjennom innboksen, lagre og pakke ut ZIP-vedlegg samt flytte behandlede meldinger til «Slettede elementer». Nedlastingen skjer via endepunktet `/$value` og `Invoke-MgGraphRequest -OutputFilePath`. Dermed lagres råinnholdet direkte i en fil uten å holde et stort vedlegg fullt ut i minnet:

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

Ved mer enn 100 e-poster må du arbeide med `Get-MgUserMessage -All` eller paginering; for en månedlig kjøring er vanligvis én batch nok.

## 7\. Send rapport-e-post via Graph

Også `Send-MailMessage` er utdatert. Med den samme appregistreringen (rettighet `Mail.Send`) sendes e-posten direkte via Graph, her med en fil som base64-kodet vedlegg:

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

## 8\. Kjør uovervåket

Som en planlagt oppgave kjører skriptet uten innlogging, fordi sertifikatet ligger i kontoens sertifikatlager:

```powershell
$action  = New-ScheduledTaskAction -Execute "powershell.exe" `
    -Argument '-NoProfile -ExecutionPolicy Bypass -File "D:\Scripts\graph-import.ps1"'
$trigger = New-ScheduledTaskTrigger -Daily -At 06:00
Register-ScheduledTask -TaskName "eCall-Graph-Import" -Action $action -Trigger $trigger `
    -User "DOMAIN\svc-ecall" -Password (Read-Host "Passord")
```

Det fullstendige eksempelet med logging og feilbehandling finnes på GitHub: [pfstr/eCall-Log-Analyzer](https://github.com/pfstr/eCall-Log-Analyzer).

## Kilder

1.  [Microsoft – «Retirement of Exchange Web Services in Exchange Online»](https://techcommunity.microsoft.com/blog/exchange/retirement-of-exchange-web-services-in-exchange-online/3924440): Kunngjøring og frist (1. oktober 2026) for slutten på EWS i Exchange Online.
    
2.  [Microsoft Learn – «Get access without a user (App-only)»](https://learn.microsoft.com/en-us/graph/auth-v2-service): App-only-autentisering mot Microsoft Graph med sertifikat.
    
3.  [Microsoft Learn – «Limiting application permissions to specific mailboxes»](https://learn.microsoft.com/en-us/graph/auth-limit-mailbox-access): Application Access Policy for å begrense appen til bestemte postbokser.
