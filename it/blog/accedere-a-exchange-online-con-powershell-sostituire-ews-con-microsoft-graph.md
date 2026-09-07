---
title: "Accedere a Exchange Online con PowerShell: sostituire EWS con Microsoft Graph"
navTitle: "Graph in PowerShell"
description: "EWS termina in Exchange Online il 1° ottobre 2026. Ecco come registrare un'app, autenticare uno script PowerShell con un certificato, limitare l'accesso a singole cassette postali ed elaborare messaggi e allegati tramite Microsoft Graph."
date: "2026-07-11"
kategorie: "Totemomail"
timeToRead: "5 min di lettura"
themen:
  - microsoft-365-exchange
slug: "accedere-a-exchange-online-con-powershell-sostituire-ews-con-microsoft-graph"
translationOf: "microsoft-graph-powershell-postfach-anbindung"
translationId: article-4c6a02c79b7bf0fe
translationReview: automatic
translationSourceHash: 66e214f25e8088562270157199f9ff55fcd9828362abf145170a12f287cd0f6c
translatedAt: 2026-09-05T07:55:45.331Z
url: https://rafaelpfister.ch/it/blog/accedere-a-exchange-online-con-powershell-sostituire-ews-con-microsoft-graph
translationModel: gpt-5.6-terra
---

# Accedere a Exchange Online con PowerShell: sostituire EWS con Microsoft Graph

Microsoft dismetterà Exchange Web Services (EWS) in Exchange Online il **1° ottobre 2026**. Gli script che recuperano messaggi o allegati da una cassetta postale devono quindi passare a Microsoft Graph.

L'esempio di questo articolo viene eseguito senza accesso utente: scarica allegati ZIP da una cassetta postale, li estrae, sposta i messaggi elaborati e infine invia un report. A questo scopo sono necessari una registrazione dell'app, un certificato e due autorizzazioni Graph. Valori come `example.com`, ID tenant e ID app sono segnaposto.

## 1. Moduli PowerShell necessari

Sono sufficienti tre moduli dell'SDK Microsoft Graph, non l'intero meta-modulo `Microsoft.Graph`:

```powershell
Install-Module Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions -Scope AllUsers
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions` | I tre sottomoduli necessari come argomento posizionale `-Name`: autenticazione, cmdlet di posta e azioni come `Send-MgUserMail` |
| `-Scope AllUsers` | Installa i moduli a livello di computer in `Program Files`; necessario affinché siano disponibili anche per l'account di servizio configurato successivamente per l'attività pianificata (richiede diritti di amministratore) |

</details>

## 2\. Registrazione dell'app in Entra ID

Uno script non presidiato effettua l'accesso come applicazione con autorizzazioni proprie. Nel [Centro di amministrazione di Entra](https://entra.microsoft.com) creare una nuova registrazione in **App registrations** e assegnare queste due autorizzazioni in **API permissions → Microsoft Graph → Application permissions**:

-   `Mail.ReadWrite`: leggere le e-mail e spostarle dopo l'elaborazione
    
-   `Mail.Send`: inviare l'e-mail di report
    
Concedere quindi il consenso dell'amministratore e annotare l'ID tenant e l'Application (Client) ID.

## 3\. Certificato anziché Client Secret

Un'autenticazione app-only funziona con Client Secret o certificato. Per le attività pianificate, un certificato è la scelta migliore: la chiave privata rimane nell'archivio certificati e nello script non compare alcuna password. Creare il certificato sul server che esegue lo script ed esportare solo la parte pubblica:

```powershell
$cert = New-SelfSignedCertificate -Subject "CN=eCall-Graph" `
    -CertStoreLocation "Cert:\LocalMachine\My" `
    -KeyExportPolicy NonExportable -KeySpec Signature `
    -KeyLength 2048 -NotAfter (Get-Date).AddYears(2)

Export-Certificate -Cert $cert -FilePath .\eCall-Graph.cer
$cert.Thumbprint   # -> im Skript als Thumbprint verwenden
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `New-SelfSignedCertificate -Subject` | Nome del soggetto del certificato; serve solo per identificarlo nell'archivio certificati |
| `-CertStoreLocation "Cert:\LocalMachine\My"` | Archivia il certificato nell'archivio del computer, non in quello dell'utente; è quindi disponibile indipendentemente dall'utente connesso |
| `-KeyExportPolicy NonExportable` | Impedisce l'esportazione della chiave privata; non lascia il server |
| `-KeySpec Signature` | Crea una chiave di firma; l'autenticazione dell'app firma con essa l'asserzione della richiesta di token |
| `-KeyLength 2048` | Lunghezza della chiave RSA di 2048 bit |
| `-NotAfter (Get-Date).AddYears(2)` | Scadenza tra due anni; dopo sarà necessario rinnovare e caricare nuovamente il certificato |
| `Export-Certificate -Cert` | Oggetto certificato da esportare |
| `-FilePath` | File di destinazione; contiene come `.cer` solo la parte pubblica |

</details>

Caricare il file `.cer`\-esportato nella registrazione dell'app, in Certificates & secrets. L'account dell'attività pianificata necessita dell'autorizzazione di lettura sulla chiave privata (`certlm.msc` → certificato → All Tasks → Manage Private Keys).

## 4\. Limitare l'accesso a singole cassette postali

Inizialmente le Application Permissions sono valide per tutte le cassette postali del tenant. Limitare quindi l'app a un gruppo di sicurezza abilitato alla posta tramite un'Application Access Policy. Questo passaggio viene eseguito una sola volta in PowerShell di Exchange Online:

```powershell
New-ApplicationAccessPolicy -AppId "<App-ID>" `
    -PolicyScopeGroupId "graph-mailboxes@example.com" `
    -AccessRight RestrictAccess `
    -Description "eCall Graph: nur Log-Postfach"

# Verificare l'efficacia
Test-ApplicationAccessPolicy -AppId "<App-ID>" -Identity "ecall-logs@example.com"
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `New-ApplicationAccessPolicy -AppId` | Application (Client) ID della registrazione dell'app a cui si applica il criterio |
| `-PolicyScopeGroupId` | Gruppo di sicurezza abilitato alla posta i cui membri definiscono l'ambito |
| `-AccessRight RestrictAccess` | Limita l'app alle cassette postali del gruppo; l'alternativa `DenyAccess` bloccherebbe proprio queste cassette postali |
| `-Description` | Testo libero per documentare il criterio |
| `Test-ApplicationAccessPolicy -Identity` | Verifica per una specifica cassetta postale se l'app può accedervi (`AccessCheckResult: Granted` oppure `Denied`) |

</details>

## 5\. Stabilire la connessione

L'autenticazione utilizza ID tenant, ID app e impronta digitale del certificato, senza alcuna interazione utente:

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
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `Connect-MgGraph -TenantId` | Tenant presso cui l'app effettua l'accesso |
| `-ClientId` | Application (Client) ID della registrazione dell'app |
| `-CertificateThumbprint` | Seleziona il certificato di autenticazione dall'archivio certificati locale tramite la relativa impronta digitale; la combinazione di `-ClientId` e certificato consente un'autenticazione app-only senza utente |
| `-NoWelcome` | Sopprime il messaggio di benvenuto dopo l'accesso; utile per output degli script e log |

</details>

## 6. Leggere i messaggi e scaricare gli allegati ZIP

Ora lo script può scorrere la posta in arrivo, salvare ed estrarre gli allegati ZIP e spostare i messaggi elaborati in «Posta eliminata». Il download avviene tramite l'endpoint `/$value` e `Invoke-MgGraphRequest -OutputFilePath`. In questo modo il contenuto grezzo viene salvato direttamente in un file, senza mantenere completamente in memoria un allegato di grandi dimensioni:

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

    # spostare l'e-mail elaborata in "Posta eliminata"
    Move-MgUserMessage -UserId $Mailbox -MessageId $msg.Id -DestinationId "deleteditems" | Out-Null
}
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `Add-Type -AssemblyName System.IO.Compression.FileSystem` | Carica l'assembly .NET con la classe `ZipFile` per l'estrazione |
| `Get-MgUserMessage -UserId` | Cassetta postale di cui vengono letti i messaggi; con l'autenticazione app-only l'indicazione è obbligatoria |
| `-Top 100` | Limita la query a un massimo di 100 messaggi per chiamata |
| `-Property id, subject, hasAttachments` | Richiede solo i campi necessari; riduce la risposta e accelera la chiamata |
| `Get-MgUserMessageAttachment -MessageId` | Messaggio di cui vengono elencati gli allegati |
| `Invoke-MgGraphRequest -Method GET` | Metodo HTTP della chiamata diretta all'API Graph |
| `-Uri` | Endpoint richiamato; il `/$value` aggiunto restituisce il contenuto file grezzo dell'allegato anziché un oggetto JSON |
| `-OutputFilePath` | Scrive la risposta direttamente nel file di destinazione, senza mantenere l'allegato interamente in memoria |
| `Move-MgUserMessage -DestinationId "deleteditems"` | Sposta il messaggio elaborato nella cartella di destinazione; `deleteditems` è il nome di cartella noto per «Posta eliminata» |

</details>

Con più di 100 e-mail, lavorare con `Get-MgUserMessage -All` o con la paginazione; per un'esecuzione mensile di solito è sufficiente un batch.

## 7\. Inviare l'e-mail di report tramite Graph

Anche `Send-MailMessage` è obsoleto. Con la stessa registrazione dell'app (autorizzazione `Mail.Send`) l'e-mail viene inviata direttamente tramite Graph, qui con un file come allegato codificato in base64:

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
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-UserId` | Cassetta postale a nome della quale viene inviata l'e-mail; deve rientrare nell'ambito dell'Application Access Policy |
| `-BodyParameter` | L'intero messaggio come hashtable nello schema Graph: `message` con oggetto, corpo, destinatari e allegati, nonché `saveToSentItems` per il salvataggio in «Posta inviata» |

</details>

## 8\. Esecuzione non presidiata

Come attività pianificata, lo script viene eseguito senza accesso perché il certificato si trova nello store dell'account:

```powershell
$action  = New-ScheduledTaskAction -Execute "powershell.exe" `
    -Argument '-NoProfile -ExecutionPolicy Bypass -File "D:\Scripts\graph-import.ps1"'
$trigger = New-ScheduledTaskTrigger -Daily -At 06:00
Register-ScheduledTask -TaskName "eCall-Graph-Import" -Action $action -Trigger $trigger `
    -User "DOMAIN\svc-ecall" -Password (Read-Host "Passwort")
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `New-ScheduledTaskAction -Execute` | Programma da eseguire, qui `powershell.exe` |
| `-Argument` | Riga di comando per il programma: `-NoProfile` salta gli script di profilo, `-ExecutionPolicy Bypass` aggira il criterio script per questa chiamata, `-File` indica lo script |
| `New-ScheduledTaskTrigger -Daily -At 06:00` | Trigger giornaliero alle 06:00 |
| `Register-ScheduledTask -TaskName` | Nome dell'attività nell'Utilità di pianificazione |
| `-Action` / `-Trigger` | Collega all'attività l'azione e il trigger creati in precedenza |
| `-User` | Account con cui viene eseguita l'attività; la chiave privata deve essere leggibile nel suo archivio certificati |
| `-Password (Read-Host "Passwort")` | Richiede interattivamente la password dell'account affinché l'attività possa avviarsi anche senza un utente connesso; in questo modo non finisce nello script né nel file della cronologia |

</details>

L'esempio completo con registrazione e gestione degli errori è disponibile su GitHub: [pfstr/eCall-Log-Analyzer](https://github.com/pfstr/eCall-Log-Analyzer).

## Fonti

1.  [Microsoft – «Retirement of Exchange Web Services in Exchange Online»](https://techcommunity.microsoft.com/blog/exchange/retirement-of-exchange-web-services-in-exchange-online/3924440): annuncio e data di riferimento (1° ottobre 2026) per la fine di EWS in Exchange Online.
    
2.  [Microsoft Learn – «Get access without a user (App-only)»](https://learn.microsoft.com/en-us/graph/auth-v2-service): autenticazione app-only a Microsoft Graph con certificato.
    
3.  [Microsoft Learn – «Limiting application permissions to specific mailboxes»](https://learn.microsoft.com/en-us/graph/auth-limit-mailbox-access): Application Access Policy per limitare l'app a singole cassette postali.
