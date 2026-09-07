---
title: "Accéder à Exchange Online avec PowerShell : remplacer EWS par Microsoft Graph"
navTitle: "Graph dans PowerShell"
description: "EWS prendra fin dans Exchange Online le 1er octobre 2026. Voici comment enregistrer une application, authentifier un script PowerShell par certificat, limiter l’accès à certaines boîtes aux lettres et traiter les messages et pièces jointes via Microsoft Graph."
date: "2026-07-11"
kategorie: "Totemomail"
timeToRead: "5 min de lecture"
themen:
  - microsoft-365-exchange
slug: "acceder-a-exchange-online-avec-powershell-remplacer-ews-par-microsoft-graph"
translationOf: "microsoft-graph-powershell-postfach-anbindung"
translationId: article-4c6a02c79b7bf0fe
translationReview: required
translationSourceHash: 66e214f25e8088562270157199f9ff55fcd9828362abf145170a12f287cd0f6c
translatedAt: 2026-09-05T07:54:53.990Z
url: https://rafaelpfister.ch/fr/blog/acceder-a-exchange-online-avec-powershell-remplacer-ews-par-microsoft-graph
translationModel: gpt-5.6-terra
---

# Accéder à Exchange Online avec PowerShell : remplacer EWS par Microsoft Graph

Microsoft mettra fin à Exchange Web Services (EWS) dans Exchange Online le **1er octobre 2026**. Les scripts qui récupèrent des messages ou des pièces jointes depuis une boîte aux lettres doivent donc migrer vers Microsoft Graph.

L’exemple présenté dans cet article fonctionne sans connexion utilisateur : il télécharge les pièces jointes ZIP depuis une boîte aux lettres, les extrait, déplace les messages traités puis envoie un rapport. Il nécessite pour cela un enregistrement d’application, un certificat et deux autorisations Graph. Des valeurs comme `example.com`, l’ID du tenant et l’ID de l’application sont des espaces réservés.

## 1. Modules PowerShell requis

Trois modules du SDK Microsoft Graph suffisent, sans installer le méta-module complet `Microsoft.Graph` :

```powershell
Install-Module Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions -Scope AllUsers
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `Microsoft.Graph.Authentication, Microsoft.Graph.Mail, Microsoft.Graph.Users.Actions` | Les trois sous-modules nécessaires comme argument positionnel `-Name` : authentification, cmdlets de messagerie et actions telles que `Send-MgUserMail` |
| `-Scope AllUsers` | Installe les modules pour l’ensemble de la machine sous `Program Files`; nécessaire afin qu’ils soient également disponibles pour le compte de service de la tâche planifiée configuré ultérieurement (droits administrateur requis) |

</details>

## 2\. Enregistrement de l’application dans Entra ID

Un script non supervisé s’authentifie en tant qu’application avec ses propres droits. Dans le [centre d’administration Entra](https://entra.microsoft.com), créez un nouvel enregistrement sous **App registrations** et attribuez les deux autorisations suivantes sous **API permissions → Microsoft Graph → Application permissions** :

-   `Mail.ReadWrite` : lire les e-mails et les déplacer après traitement
    
-   `Mail.Send` : envoyer l’e-mail de rapport
    
Accordez ensuite le consentement administrateur et notez l’ID du tenant ainsi que l’Application (Client) ID.

## 3\. Certificat plutôt que secret client

Une authentification App-Only fonctionne avec un secret client ou un certificat. Pour les tâches planifiées, un certificat est le meilleur choix : la clé privée reste dans le magasin de certificats et aucun mot de passe ne figure dans le script. Créez le certificat sur le serveur qui exécutera le script et n’exportez que la partie publique :

```powershell
$cert = New-SelfSignedCertificate -Subject "CN=eCall-Graph" `
    -CertStoreLocation "Cert:\LocalMachine\My" `
    -KeyExportPolicy NonExportable -KeySpec Signature `
    -KeyLength 2048 -NotAfter (Get-Date).AddYears(2)

Export-Certificate -Cert $cert -FilePath .\eCall-Graph.cer
$cert.Thumbprint   # -> im Skript als Thumbprint verwenden
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `New-SelfSignedCertificate -Subject` | Nom du sujet du certificat ; sert uniquement à l’identifier dans le magasin de certificats |
| `-CertStoreLocation "Cert:\LocalMachine\My"` | Enregistre le certificat dans le magasin de l’ordinateur, et non dans celui de l’utilisateur ; il est ainsi disponible indépendamment de l’utilisateur connecté |
| `-KeyExportPolicy NonExportable` | Empêche l’exportation de la clé privée ; elle ne quitte pas le serveur |
| `-KeySpec Signature` | Crée une clé de signature ; l’authentification de l’application l’utilise pour signer l’assertion de demande de jeton |
| `-KeyLength 2048` | Longueur de clé RSA de 2048 bits |
| `-NotAfter (Get-Date).AddYears(2)` | Expiration dans deux ans ; le certificat devra alors être renouvelé et téléversé à nouveau |
| `Export-Certificate -Cert` | Objet certificat à exporter |
| `-FilePath` | Fichier cible ; il contient, sous la forme `.cer`, uniquement la partie publique |

</details>

Téléversez le fichier `.cer`\-exporté dans l’enregistrement de l’application, sous Certificates & secrets. Le compte de la tâche planifiée doit disposer d’un droit de lecture sur la clé privée (`certlm.msc` → certificat → All Tasks → Manage Private Keys).

## 4\. Limiter l’accès à certaines boîtes aux lettres

Les autorisations d’application s’appliquent initialement à toutes les boîtes aux lettres du tenant. Limitez donc l’application, à l’aide d’une Application Access Policy, à un groupe de sécurité à extension messagerie. Cette étape est exécutée une seule fois dans Exchange Online PowerShell :

```powershell
New-ApplicationAccessPolicy -AppId "<App-ID>" `
    -PolicyScopeGroupId "graph-mailboxes@example.com" `
    -AccessRight RestrictAccess `
    -Description "eCall Graph: nur Log-Postfach"

# Vérifier l’efficacité
Test-ApplicationAccessPolicy -AppId "<App-ID>" -Identity "ecall-logs@example.com"
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `New-ApplicationAccessPolicy -AppId` | Application (Client) ID de l’enregistrement d’application auquel s’applique la stratégie |
| `-PolicyScopeGroupId` | Groupe de sécurité à extension messagerie dont les membres définissent le périmètre |
| `-AccessRight RestrictAccess` | Limite l’application aux boîtes aux lettres du groupe ; l’alternative `DenyAccess` bloquerait précisément ces boîtes aux lettres |
| `-Description` | Texte libre destiné à documenter la stratégie |
| `Test-ApplicationAccessPolicy -Identity` | Vérifie, pour une boîte aux lettres précise, si l’application peut y accéder (`AccessCheckResult: Granted` ou `Denied`) |

</details>

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

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `Connect-MgGraph -TenantId` | Tenant auprès duquel l’application s’authentifie |
| `-ClientId` | Application (Client) ID de l’enregistrement d’application |
| `-CertificateThumbprint` | Sélectionne le certificat d’authentification dans le magasin local de certificats via son empreinte ; la combinaison de `-ClientId` et du certificat permet une authentification App-Only sans utilisateur |
| `-NoWelcome` | Supprime le message de bienvenue après l’authentification ; utile pour les sorties de scripts et les journaux |

</details>

## 6. Lire les messages et télécharger les pièces jointes ZIP

Le script peut à présent parcourir la boîte de réception, enregistrer et extraire les pièces jointes ZIP, puis déplacer les messages traités vers « Éléments supprimés ». Le téléchargement s’effectue via le point de terminaison `/$value` et `Invoke-MgGraphRequest -OutputFilePath`. Le contenu brut est ainsi écrit directement dans un fichier, sans conserver une pièce jointe volumineuse entièrement en mémoire :

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

    # déplacer l’e-mail traité vers « Éléments supprimés »
    Move-MgUserMessage -UserId $Mailbox -MessageId $msg.Id -DestinationId "deleteditems" | Out-Null
}
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `Add-Type -AssemblyName System.IO.Compression.FileSystem` | Charge l’assembly .NET avec la classe `ZipFile` pour l’extraction |
| `Get-MgUserMessage -UserId` | Boîte aux lettres dont les messages sont lus ; cette indication est obligatoire avec l’authentification App-Only |
| `-Top 100` | Limite la requête à 100 messages au maximum par appel |
| `-Property id, subject, hasAttachments` | Demande uniquement les champs nécessaires ; cela réduit la réponse et accélère l’appel |
| `Get-MgUserMessageAttachment -MessageId` | Message dont les pièces jointes sont listées |
| `Invoke-MgGraphRequest -Method GET` | Méthode HTTP de l’appel direct à l’API Graph |
| `-Uri` | Point de terminaison appelé ; le `/$value` ajouté renvoie le contenu brut du fichier joint au lieu d’un objet JSON |
| `-OutputFilePath` | Écrit la réponse directement dans le fichier cible sans conserver la pièce jointe complète en mémoire |
| `Move-MgUserMessage -DestinationId "deleteditems"` | Déplace le message traité vers le dossier cible ; `deleteditems` est le nom de dossier connu pour « Éléments supprimés » |

</details>

Pour plus de 100 e-mails, utilisez `Get-MgUserMessage -All` ou la pagination ; pour une exécution mensuelle, un lot suffit généralement.

## 7\. Envoyer l’e-mail de rapport via Graph

`Send-MailMessage` est également obsolète. Avec le même enregistrement d’application (autorisation `Mail.Send`), l’e-mail est envoyé directement via Graph, ici avec un fichier joint encodé en base64 :

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
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-UserId` | Boîte aux lettres au nom de laquelle l’e-mail est envoyé ; elle doit être couverte par le périmètre de l’Application Access Policy |
| `-BodyParameter` | Le message complet sous forme de table de hachage dans le schéma Graph : `message` avec objet, corps, destinataires et pièces jointes, ainsi que `saveToSentItems` pour l’enregistrement dans « Éléments envoyés » |

</details>

## 8\. Exécuter sans supervision

En tant que tâche planifiée, le script s’exécute sans connexion, car le certificat se trouve dans le magasin du compte :

```powershell
$action  = New-ScheduledTaskAction -Execute "powershell.exe" `
    -Argument '-NoProfile -ExecutionPolicy Bypass -File "D:\Scripts\graph-import.ps1"'
$trigger = New-ScheduledTaskTrigger -Daily -At 06:00
Register-ScheduledTask -TaskName "eCall-Graph-Import" -Action $action -Trigger $trigger `
    -User "DOMAIN\svc-ecall" -Password (Read-Host "Passwort")
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `New-ScheduledTaskAction -Execute` | Programme à exécuter, ici `powershell.exe` |
| `-Argument` | Ligne de commande du programme : `-NoProfile` ignore les scripts de profil, `-ExecutionPolicy Bypass` contourne la stratégie d’exécution pour cet appel et `-File` désigne le script |
| `New-ScheduledTaskTrigger -Daily -At 06:00` | Déclencheur quotidien à 06:00 |
| `Register-ScheduledTask -TaskName` | Nom de la tâche dans le planificateur de tâches |
| `-Action` / `-Trigger` | Associe à la tâche l’action et le déclencheur créés précédemment |
| `-User` | Compte sous lequel la tâche est exécutée ; la clé privée doit être lisible dans son magasin de certificats |
| `-Password (Read-Host "Passwort")` | Demande le mot de passe du compte de manière interactive afin que la tâche puisse démarrer même sans utilisateur connecté ; il ne se retrouve ainsi ni dans le script ni dans le fichier d’historique |

</details>

L’exemple complet avec journalisation et gestion des erreurs est disponible sur GitHub : [pfstr/eCall-Log-Analyzer](https://github.com/pfstr/eCall-Log-Analyzer).

## Sources

1.  [Microsoft – « Retrait d’Exchange Web Services dans Exchange Online »](https://techcommunity.microsoft.com/blog/exchange/retirement-of-exchange-web-services-in-exchange-online/3924440) : annonce et date butoir (1er octobre 2026) de la fin d’EWS dans Exchange Online.
    
2.  [Microsoft Learn – « Obtenir un accès sans utilisateur (App-only) »](https://learn.microsoft.com/en-us/graph/auth-v2-service) : authentification App-Only auprès de Microsoft Graph avec certificat.
    
3.  [Microsoft Learn – « Limiter les autorisations d’application à des boîtes aux lettres spécifiques »](https://learn.microsoft.com/en-us/graph/auth-limit-mailbox-access) : Application Access Policy pour restreindre l’application à certaines boîtes aux lettres.
