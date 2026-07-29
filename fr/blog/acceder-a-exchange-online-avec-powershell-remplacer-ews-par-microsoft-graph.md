---
title: "Accéder à Exchange Online avec PowerShell : remplacer EWS par Microsoft Graph"
navTitle: "Graph dans PowerShell"
description: "EWS prendra fin dans Exchange Online le 1er octobre 2026. Voici comment enregistrer une application, authentifier un script PowerShell par certificat, limiter l’accès à certaines boîtes aux lettres et traiter les messages et pièces jointes avec Microsoft Graph."
date: "2026-07-11"
kategorie: "Totemomail"
timeToRead: "5 min de lecture"
themen:
  - microsoft-365-exchange
slug: "acceder-a-exchange-online-avec-powershell-remplacer-ews-par-microsoft-graph"
translationOf: "microsoft-graph-powershell-postfach-anbindung"
url: "https://rafaelpfister.ch/fr/blog/acceder-a-exchange-online-avec-powershell-remplacer-ews-par-microsoft-graph"
translationId: article-4c6a02c79b7bf0fe
translationReview: automatic
translationSourceHash: bab6aabe691a64409231665e5d2dd2288fb409fa66b4a7f33cfcb79aeaa643b3
translatedAt: 2026-07-29T12:29:38.931Z
---

# Accéder à Exchange Online avec PowerShell : remplacer EWS par Microsoft Graph

Microsoft mettra fin à Exchange Web Services (EWS) dans Exchange Online le **1er octobre 2026**. Les scripts qui récupèrent des messages ou des pièces jointes depuis une boîte aux lettres doivent donc passer à Microsoft Graph.

L’exemple de cet article s’exécute sans connexion utilisateur : il télécharge les pièces jointes ZIP depuis une boîte aux lettres, les décompresse, déplace les messages traités puis envoie un rapport. Cela nécessite un enregistrement d’application, un certificat et deux autorisations Graph. Des valeurs telles que `example.com`, l’ID du tenant et l’ID de l’application sont des espaces réservés.

## 1. Modules PowerShell requis

Trois modules du SDK Microsoft Graph suffisent, et non le méta-module complet `Microsoft.Graph` :

```powershell
Install-Module Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions -Scope AllUsers
```

## 2\. Enregistrement de l’application dans Entra ID

Un script sans surveillance s’authentifie en tant qu’application avec ses propres droits. Créez un nouvel enregistrement dans le [Centre d’administration Entra](https://entra.microsoft.com) sous **App registrations**, puis attribuez ces deux droits sous **API permissions → Microsoft Graph → Application permissions** :

-   `Mail.ReadWrite` : lire les e-mails et les déplacer après traitement
    
-   `Mail.Send` : envoyer l’e-mail de rapport
    

Accordez ensuite le consentement administrateur et notez l’ID du tenant ainsi que l’ID d’application (client).

## 3\. Certificat plutôt que secret client

Une authentification App-Only fonctionne avec un secret client ou un certificat. Pour les tâches planifiées, un certificat est le meilleur choix : la clé privée reste dans le magasin de certificats et aucun mot de passe ne figure dans le script. Créez le certificat sur le serveur qui exécutera le script et n’exportez que la partie publique :

```powershell
$cert = New-SelfSignedCertificate -Subject "CN=eCall-Graph" `
    -CertStoreLocation "Cert:\LocalMachine\My" `
    -KeyExportPolicy NonExportable -KeySpec Signature `
    -KeyLength 2048 -NotAfter (Get-Date).AddYears(2)

Export-Certificate -Cert $cert -FilePath .\eCall-Graph.cer
$cert.Thumbprint   # -> à utiliser comme empreinte dans le script
```

Téléversez le fichier `.cer`\- dans l’enregistrement d’application, sous Certificates & secrets. Le compte de la tâche planifiée a besoin d’un droit de lecture sur la clé privée (`certlm.msc` → certificat → All Tasks → Manage Private Keys).

## 4\. Limiter l’accès à certaines boîtes aux lettres

Les autorisations d’application s’appliquent initialement à toutes les boîtes aux lettres du tenant. Limitez donc l’application à un groupe de sécurité à extension messagerie à l’aide d’une Application Access Policy. Cette étape est exécutée une seule fois dans Exchange Online PowerShell :

```powershell
New-ApplicationAccessPolicy -AppId "<ID-application>" `
    -PolicyScopeGroupId "graph-mailboxes@example.com" `
    -AccessRight RestrictAccess `
    -Description "eCall Graph : uniquement la boîte aux lettres des journaux"

# Vérifier l’application de la stratégie
Test-ApplicationAccessPolicy -AppId "<ID-application>" -Identity "ecall-logs@example.com"
```

## 5\. Établir la connexion

L’authentification utilise l’ID du tenant, l’ID de l’application et l’empreinte du certificat, sans aucune interaction utilisateur :

```powershell
$TenantId   = "00000000-0000-0000-0000-000000000000"
$ClientId   = "00000000-0000-0000-0000-000000000000"
$Thumbprint = "0000000000000000000000000000000000000000"
$Mailbox    = "ecall-logs@example.com"

Import-Module Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions
Connect-MgGraph -TenantId $TenantId -ClientId $ClientId `
    -CertificateThumbprint $Thumbprint -NoWelcome
```

## 6. Lire les messages et télécharger les pièces jointes ZIP

Le script peut maintenant parcourir la boîte de réception, enregistrer et décompresser les pièces jointes ZIP, puis déplacer les messages traités vers « Éléments supprimés ». Le téléchargement passe par le point de terminaison `/$value` et `Invoke-MgGraphRequest -OutputFilePath`. Le contenu brut est ainsi écrit directement dans un fichier, sans conserver entièrement une pièce jointe volumineuse en mémoire :

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

En présence de plus de 100 e-mails, utilisez `Get-MgUserMessage -All` ou la pagination ; un lot suffit généralement pour une exécution mensuelle.

## 7\. Envoyer l’e-mail de rapport via Graph

`Send-MailMessage` est également obsolète. Avec le même enregistrement d’application (droit `Mail.Send`), l’e-mail est envoyé directement via Graph, ici avec un fichier joint encodé en base64 :

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

## 8\. Exécuter sans surveillance

En tant que tâche planifiée, le script s’exécute sans connexion, car le certificat se trouve dans le magasin du compte :

```powershell
$action  = New-ScheduledTaskAction -Execute "powershell.exe" `
    -Argument '-NoProfile -ExecutionPolicy Bypass -File "D:\Scripts\graph-import.ps1"'
$trigger = New-ScheduledTaskTrigger -Daily -At 06:00
Register-ScheduledTask -TaskName "eCall-Graph-Import" -Action $action -Trigger $trigger `
    -User "DOMAIN\svc-ecall" -Password (Read-Host "Mot de passe")
```

L’exemple complet avec journalisation et gestion des erreurs est disponible sur GitHub : [pfstr/eCall-Log-Analyzer](https://github.com/pfstr/eCall-Log-Analyzer).

## Sources

1.  [Microsoft – « Retrait d’Exchange Web Services dans Exchange Online »](https://techcommunity.microsoft.com/blog/exchange/retirement-of-exchange-web-services-in-exchange-online/3924440) : annonce et date limite (1er octobre 2026) de la fin d’EWS dans Exchange Online.
    
2.  [Microsoft Learn – « Obtenir un accès sans utilisateur (App-only) »](https://learn.microsoft.com/en-us/graph/auth-v2-service) : authentification App-Only auprès de Microsoft Graph avec un certificat.
    
3.  [Microsoft Learn – « Limiter les autorisations d’application à certaines boîtes aux lettres »](https://learn.microsoft.com/en-us/graph/auth-limit-mailbox-access) : Application Access Policy permettant de limiter l’application à certaines boîtes aux lettres.
