---
title: "Accedere a Exchange Online con PowerShell: sostituire EWS con Microsoft Graph"
navTitle: "Graph in PowerShell"
description: "EWS terminerà in Exchange Online il 1° ottobre 2026. Ecco come registrare un'app, autenticare uno script PowerShell con un certificato, limitare l'accesso a singole cassette postali ed elaborare messaggi e allegati tramite Microsoft Graph."
date: "2026-07-11"
kategorie: "Totemomail"
timeToRead: "5 min di lettura"
themen:
  - "microsoft-365-exchange"
slug: "accedere-a-exchange-online-con-powershell-sostituire-ews-con-microsoft-graph"
translationOf: "microsoft-graph-powershell-postfach-anbindung"
url: "https://rafaelpfister.ch/it/blog/accedere-a-exchange-online-con-powershell-sostituire-ews-con-microsoft-graph"
---

# Accedere a Exchange Online con PowerShell: sostituire EWS con Microsoft Graph

Microsoft ritirerà Exchange Web Services (EWS) in Exchange Online il **1° ottobre 2026**. Gli script che recuperano messaggi o allegati da una cassetta postale devono quindi passare a Microsoft Graph.

L'esempio di questo articolo viene eseguito senza accesso utente: scarica allegati ZIP da una cassetta postale, li estrae, sposta i messaggi elaborati e invia quindi un rapporto. A tale scopo sono necessari una registrazione dell'app, un certificato e due autorizzazioni Graph. Valori come `example.com`, l'ID tenant e l'ID app sono segnaposto.

## 1. Moduli PowerShell necessari

Sono sufficienti tre moduli dell'SDK Microsoft Graph, non l'intero meta-modulo `Microsoft.Graph`:

```powershell
Install-Module Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions -Scope AllUsers
```

## 2\. Registrazione dell'app in Entra ID

Uno script non presidiato si autentica come applicazione con autorizzazioni proprie. Create una nuova registrazione nel [Centro di amministrazione Entra](https://entra.microsoft.com) in **App registrations** e assegnate queste due autorizzazioni in **API permissions → Microsoft Graph → Application permissions**:

-   `Mail.ReadWrite`: leggere le email e spostarle dopo l'elaborazione
    
-   `Mail.Send`: inviare l'email del rapporto
    
Concedete quindi il consenso dell'amministratore e annotate l'ID tenant e l'Application (Client) ID.

## 3\. Certificato anziché client secret

Un'autenticazione solo app funziona con un client secret o un certificato. Per le attività pianificate, un certificato è la scelta migliore: la chiave privata rimane nell'archivio certificati e nello script non è presente alcuna password. Create il certificato sul server che esegue lo script ed esportate solo la parte pubblica:

```powershell
$cert = New-SelfSignedCertificate -Subject "CN=eCall-Graph" `
    -CertStoreLocation "Cert:\LocalMachine\My" `
    -KeyExportPolicy NonExportable -KeySpec Signature `
    -KeyLength 2048 -NotAfter (Get-Date).AddYears(2)

Export-Certificate -Cert $cert -FilePath .\eCall-Graph.cer
$cert.Thumbprint   # -> usare come impronta digitale nello script
```

Caricate il file `.cer`\- nella registrazione dell'app, in Certificates & secrets. L'account dell'attività pianificata necessita dell'autorizzazione di lettura per la chiave privata (`certlm.msc` → certificato → All Tasks → Manage Private Keys).

## 4\. Limitare l'accesso a singole cassette postali

Inizialmente, le autorizzazioni applicative valgono per tutte le cassette postali del tenant. Limitate pertanto l'app a un gruppo di sicurezza abilitato alla posta mediante una Application Access Policy. Questo passaggio viene eseguito una sola volta in Exchange Online PowerShell:

```powershell
New-ApplicationAccessPolicy -AppId "<ID-app>" `
    -PolicyScopeGroupId "graph-mailboxes@example.com" `
    -AccessRight RestrictAccess `
    -Description "eCall Graph: solo cassetta postale dei log"

# Verificare l'efficacia
Test-ApplicationAccessPolicy -AppId "<ID-app>" -Identity "ecall-logs@example.com"
```

## 5\. Stabilire la connessione

L'autenticazione utilizza l'ID tenant, l'ID app e l'impronta digitale del certificato, senza alcuna interazione utente:

```powershell
$TenantId   = "00000000-0000-0000-0000-000000000000"
$ClientId   = "00000000-0000-0000-0000-000000000000"
$Thumbprint = "0000000000000000000000000000000000000000"
$Mailbox    = "ecall-logs@example.com"

Import-Module Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions
Connect-MgGraph -TenantId $TenantId -ClientId $ClientId `
    -CertificateThumbprint $Thumbprint -NoWelcome
```

## 6. Leggere i messaggi e scaricare gli allegati ZIP

Ora lo script può scorrere la posta in arrivo, salvare ed estrarre gli allegati ZIP e spostare i messaggi elaborati in «Elementi eliminati». Il download avviene tramite l'endpoint `/$value` e `Invoke-MgGraphRequest -OutputFilePath`. In questo modo il contenuto grezzo viene scritto direttamente in un file, senza mantenere interamente in memoria un allegato di grandi dimensioni:

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

Con più di 100 email, lavorate con `Get-MgUserMessage -All` o con la paginazione; per un'esecuzione mensile di solito è sufficiente un batch.

## 7\. Inviare l'email del rapporto tramite Graph

Anche `Send-MailMessage` è obsoleto. Tramite la stessa registrazione dell'app (autorizzazione `Mail.Send`) l'email viene inviata direttamente tramite Graph, qui con un file come allegato codificato in base64:

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

## 8\. Esecuzione non presidiata

Come attività pianificata, lo script viene eseguito senza autenticazione perché il certificato si trova nell'archivio dell'account:

```powershell
$action  = New-ScheduledTaskAction -Execute "powershell.exe" `
    -Argument '-NoProfile -ExecutionPolicy Bypass -File "D:\Scripts\graph-import.ps1"'
$trigger = New-ScheduledTaskTrigger -Daily -At 06:00
Register-ScheduledTask -TaskName "eCall-Graph-Import" -Action $action -Trigger $trigger `
    -User "DOMAIN\svc-ecall" -Password (Read-Host "Password")
```

L'esempio completo con registrazione e gestione degli errori è disponibile su GitHub: [pfstr/eCall-Log-Analyzer](https://github.com/pfstr/eCall-Log-Analyzer).

## Fonti

1.  [Microsoft – «Ritiro di Exchange Web Services in Exchange Online»](https://techcommunity.microsoft.com/blog/exchange/retirement-of-exchange-web-services-in-exchange-online/3924440): annuncio e data di riferimento (1° ottobre 2026) per la fine di EWS in Exchange Online.
    
2.  [Microsoft Learn – «Ottenere l'accesso senza un utente (solo app)»](https://learn.microsoft.com/en-us/graph/auth-v2-service): autenticazione solo app a Microsoft Graph con certificato.
    
3.  [Microsoft Learn – «Limitare le autorizzazioni applicative a cassette postali specifiche»](https://learn.microsoft.com/en-us/graph/auth-limit-mailbox-access): Application Access Policy per limitare l'app a singole cassette postali.
