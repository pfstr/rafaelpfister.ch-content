---
title: "Migration EXO sans Remote Move"
navTitle: "Migration EXO sans Remote Move"
description: "Comment provisionner de manière contrôlée des boîtes aux lettres Exchange locales sous forme de nouvelles boîtes aux lettres Exchange Online vides : sauvegarde PST, validation CSV, RemoteMailbox, synchronisation, validation et rollback."
date: "2026-08-07"
kategorie: "Exchange OnPrem / Hybride"
timeToRead: "12 min de lecture"
themen:
  - exchange-onprem-hybrid
  - microsoft-365-exchange
  - active-directory-entra
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange"
protokolle:
  - "powershell"
  - "migration"
related:
  - typische-ursachen-fuer-mail-loops-und-deren-behebung
  - totemomail-m365
  - ghost-sender-exchange-online-nebeneingang
slug: "migration-exo-sans-remote-move"
translationId: "article-8f3c1b7a62d94e50"
draft: false
translationOf: exchange-mailboxen-ohne-remote-move-cloud-neu-erstellen
translationSourceHash: 8ef7108bef25cbfdabbdd58d6ed17a1cc34ba2da65eb7e685180b0976f99d62c
translationModel: gpt-5.6-terra
translatedAt: 2026-10-30T03:32:35.749Z
translationReview: required
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/fr/blog/migration-exo-sans-remote-move
---

# Migration EXO sans Remote Move

Un Hybrid Remote Move est la méthode habituelle pour déplacer une boîte aux lettres Exchange locale avec son contenu vers Exchange Online. Toutes les organisations n’autorisent pas ce chemin de migration. Lorsqu’une politique de sécurité exclut les Remote Moves, une approche délibérément différente peut être acceptable : la boîte aux lettres locale est sauvegardée au format PST, dissociée de l’utilisateur AD synchronisé, puis une nouvelle boîte aux lettres vide est provisionnée dans Exchange Online pour ce même utilisateur.

Cette approche **n’est pas une migration de boîte aux lettres**. Elle ne transfère ni les messages, ni les calendriers, ni les règles, ni les autorisations vers le cloud. Le PST sert exclusivement de sauvegarde et n’est pas importé dans ce scénario. Cette procédure convient donc uniquement si une boîte aux lettres cible vide est acceptable sur le plan métier et si la perte de la configuration active de la boîte aux lettres est explicitement approuvée.

## État cible et prérequis impératifs

Après le basculement, le même utilisateur AD est conservé. Localement, il n’est toutefois plus géré comme `UserMailbox`, `SharedMailbox`, `RoomMailbox` ou `EquipmentMailbox`, mais comme `RemoteMailbox`. Après la synchronisation, cet objet représente la nouvelle boîte aux lettres dans Exchange Online.

L’état souhaité est le suivant :

1. La boîte aux lettres locale est entièrement sauvegardée au format PST.
2. La boîte aux lettres locale est dissociée, mais n’est pas encore définitivement supprimée pendant la période de rétention configurée.
3. L’utilisateur AD existant est activé comme RemoteMailbox.
4. L’adresse principale, les alias et l’ancien `LegacyExchangeDN` sont conservés.
5. Entra Connect a synchronisé les modifications.
6. Un plan de service Exchange Online est attribué aux boîtes aux lettres utilisateur.
7. Exchange Online affiche une véritable boîte aux lettres cloud et le flux de messagerie y aboutit.

Avant de commencer, les points suivants doivent également être clarifiés :

- Le partage PST est accessible via UNC. Le groupe `Exchange Trusted Subsystem` y dispose de droits de lecture et d’écriture.
- Le compte exécutant dispose du rôle de gestion `Mailbox Import Export`.
- Le PST constitue uniquement la sauvegarde convenue ; aucune importation ultérieure n’est prévue.
- La rétention, Litigation Hold, eDiscovery et les exigences réglementaires ont été vérifiées séparément.
- Les délégations, Send-As, Send-on-Behalf, les transferts, les règles de boîte de réception, les appareils mobiles et les accès applicatifs sont inventoriés.
- Pendant l’exportation et la bascule, les messages entrants sont retenus de manière contrôlée au niveau de la passerelle en amont. Les utilisateurs et les applications ne doivent plus écrire dans la boîte aux lettres source.
- La rétention de la base de données de boîtes aux lettres locale couvre la fenêtre de rollback.

## Pourquoi une liste de validation CSV est indispensable

Un pipeline direct tel que `Get-Mailbox | Disable-Mailbox` est trop risqué pour cette opération. Il pourrait également inclure des boîtes aux lettres système, Discovery ou d’autres boîtes non approuvées. La procédure suivante s’appuie donc sur deux validations explicites :

- `Action=CUTOVER` définit quelle ligne peut réellement être basculée.
- `PstVerified=YES` confirme que le fichier d’exportation a été vérifié sur les plans technique et organisationnel.

Seul l’inventaire est d’abord généré :

```powershell
$CsvPath = "C:\Migration\mailboxes.csv"
$RemoteRoutingDomain = "contoso.mail.onmicrosoft.com"

Get-Mailbox -ResultSize Unlimited -RecipientTypeDetails `
    UserMailbox,SharedMailbox,RoomMailbox,EquipmentMailbox |
    Sort-Object PrimarySmtpAddress |
    ForEach-Object {
        [pscustomobject]@{
            Identity             = $_.Identity
            DisplayName          = $_.DisplayName
            PrimarySmtpAddress   = $_.PrimarySmtpAddress.ToString()
            Alias                = $_.Alias
            SourceType           = $_.RecipientTypeDetails.ToString()
            ArchiveStatus        = $_.ArchiveStatus.ToString()
            ServerName           = $_.ServerName
            Database             = $_.Database.ToString()
            RemoteRoutingAddress = "$($_.Alias)@$RemoteRoutingDomain"
            Action               = "REVIEW"
            PstVerified          = "NO"
        }
    } |
    Export-Csv -Path $CsvPath -NoTypeInformation -Encoding UTF8
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `Get-Mailbox -ResultSize Unlimited` | Supprime la limite par défaut de 1 000 résultats ; sans ce paramètre, des boîtes aux lettres manquent dans l’inventaire des grands environnements |
| `-RecipientTypeDetails UserMailbox,SharedMailbox,RoomMailbox,EquipmentMailbox` | Limite la requête aux quatre types de boîtes aux lettres à basculer ; les boîtes aux lettres système et Discovery sont exclues |
| `Sort-Object PrimarySmtpAddress` | Trie la sortie par adresse SMTP principale afin que le fichier CSV reste ordonné de manière stable lors de la vérification métier |
| `Export-Csv -Path` | Chemin de destination du fichier CSV |
| `-NoTypeInformation` | Supprime la ligne d’en-tête de type `#TYPE ...`, que les anciennes versions de PowerShell écrivent sinon en première ligne |
| `-Encoding UTF8` | Écrit le fichier avec un encodage UTF-8 afin que les caractères accentués dans les noms d’affichage soient correctement conservés |

</details>

Le fichier est ensuite nettoyé sur le plan métier. Seules les boîtes aux lettres effectivement approuvées reçoivent `Action=CUTOVER`. Les boîtes aux lettres système et les objets spéciaux ne doivent pas figurer dans cette liste.

## Phase 1 : sauvegarder la boîte aux lettres principale et l’archive au format PST

`New-MailboxExportRequest` écrit uniquement vers un chemin UNC. Un nom de fichier unique est généré pour chaque boîte aux lettres. Une archive en ligne active de l’Exchange local est exportée séparément :

```powershell
$CsvPath = "C:\Migration\mailboxes.csv"
$PstShare = "\\fileserver\exchange-pst$"
$BatchName = "EXO-NewMailbox-20260807"

$targets = Import-Csv $CsvPath | Where-Object Action -eq "CUTOVER"

foreach ($row in $targets) {
    $safeName = ($row.PrimarySmtpAddress -replace '[^a-zA-Z0-9@._-]', '_')
    $primaryPath = "$PstShare\$safeName.pst"

    New-MailboxExportRequest `
        -Mailbox $row.Identity `
        -FilePath $primaryPath `
        -Name "Primary-$safeName" `
        -BatchName $BatchName

    if ($row.ArchiveStatus -eq "Active") {
        New-MailboxExportRequest `
            -Mailbox $row.Identity `
            -IsArchive `
            -FilePath "$PstShare\$safeName-archive.pst" `
            -Name "Archive-$safeName" `
            -BatchName $BatchName
    }
}
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `Import-Csv $CsvPath` | Lit la liste de validation ; chaque ligne devient un objet dont les colonnes CSV sont des propriétés |
| `Where-Object Action -eq "CUTOVER"` | Ne traite que les lignes explicitement approuvées |
| `New-MailboxExportRequest -Mailbox` | Boîte aux lettres source de l’exportation (ici l’identité de la ligne CSV) |
| `-FilePath` | Chemin de destination du fichier PST ; il doit s’agir d’un chemin UNC, les chemins locaux sont refusés par le cmdlet |
| `-Name` | Nom unique de la demande ; permet ensuite d’associer précisément l’exportation principale et l’exportation d’archive |
| `-BatchName` | Regroupe toutes les demandes d’une exécution sous un nom de lot ; sert de base aux requêtes d’état et au nettoyage |
| `-IsArchive` | Exporte l’archive en ligne au lieu de la boîte aux lettres principale ; d’où la seconde demande par boîte aux lettres avec archive active |

</details>

L’exportation n’est validée que lorsque **chaque** demande a le statut `Completed` :

```powershell
Get-MailboxExportRequest -BatchName $BatchName |
    Get-MailboxExportRequestStatistics -IncludeReport |
    Format-Table DisplayName,Status,PercentComplete,FailureCode,Message -AutoSize

Get-MailboxExportRequest -BatchName $BatchName |
    Where-Object Status -ne "Completed"
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `Get-MailboxExportRequest -BatchName` | Répertorie toutes les demandes d’exportation du lot indiqué |
| `Get-MailboxExportRequestStatistics -IncludeReport` | Complète les statistiques avec le rapport d’historique détaillé, qui contient les causes d’erreur des demandes individuelles |
| `Format-Table ... -AutoSize` | Affichage sous forme de tableau des propriétés indiquées ; `-AutoSize` adapte la largeur des colonnes au contenu |
| `Where-Object Status -ne "Completed"` | Filtre toutes les demandes qui ne sont pas encore terminées ou qui ont échoué ; la sortie doit être vide avant de poursuivre |

</details>

En outre, l’existence, la taille, la lisibilité, la prise en charge par la sauvegarde et la protection d’accès des fichiers doivent être vérifiées. Ce n’est qu’ensuite que `PstVerified=YES` est défini pour la ligne CSV correspondante.

## Phase 2 : sauvegarder les données de boîte aux lettres et les attributs Exchange

Avant la première modification, un instantané lisible par machine est créé pour chaque boîte aux lettres. Il est plus important qu’une capture d’écran, car les alias, GUID et le `LegacyExchangeDN` peuvent être reconstruits avec exactitude ultérieurement :

```powershell
$SnapshotPath = "C:\Migration\Snapshots"
New-Item -ItemType Directory -Path $SnapshotPath -Force | Out-Null

Import-Csv "C:\Migration\mailboxes.csv" |
    Where-Object { $_.Action -eq "CUTOVER" -and $_.PstVerified -eq "YES" } |
    ForEach-Object {
        $mailbox = Get-Mailbox -Identity $_.Identity
        $safeName = ($_.PrimarySmtpAddress -replace '[^a-zA-Z0-9@._-]', '_')

        $mailbox |
            Select-Object Identity,DistinguishedName,ExchangeGuid,ArchiveGuid,
                RecipientTypeDetails,PrimarySmtpAddress,EmailAddresses,
                LegacyExchangeDN,Alias,Database,ServerName |
            Export-Clixml "$SnapshotPath\$safeName.xml"
    }
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `New-Item -ItemType Directory -Path ... -Force` | Crée le répertoire des instantanés ; `-Force` supprime l’erreur s’il existe déjà |
| `Get-Mailbox -Identity` | Récupère l’objet de boîte aux lettres actuel pour chaque ligne CSV |
| `Select-Object Identity,...,ServerName` | Réduit l’objet aux attributs nécessaires à une reconstruction ultérieure (GUID, adresses, `LegacyExchangeDN`, base de données) |
| `Export-Clixml` | Sérialise l’objet au format CLIXML en conservant les types ; contrairement au CSV, les valeurs multiples telles que `EmailAddresses` sont entièrement conservées et peuvent être relues avec `Import-Clixml` |

</details>

Les délégations et les transferts nécessitent des exportations distinctes. Au minimum, les informations suivantes doivent être sauvegardées séparément :

```powershell
$mailbox = Get-Mailbox -Identity user01@contoso.com

$mailbox | Format-List ForwardingAddress,ForwardingSmtpAddress,DeliverToMailboxAndForward
Get-MailboxPermission -Identity $mailbox.Identity
Get-ADPermission -Identity $mailbox.DistinguishedName
Get-InboxRule -Mailbox $mailbox.Identity
Get-CalendarProcessing -Identity $mailbox.Identity -ErrorAction SilentlyContinue
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `Format-List ForwardingAddress,ForwardingSmtpAddress,DeliverToMailboxAndForward` | Affiche les trois attributs de transfert de la boîte aux lettres sous forme de liste |
| `Get-MailboxPermission -Identity` | Répertorie les autorisations de boîte aux lettres telles que Full Access |
| `Get-ADPermission -Identity` | Répertorie les autorisations AD sur l’objet utilisateur, notamment Send-As ; attend ici le Distinguished Name |
| `Get-InboxRule -Mailbox` | Répertorie les règles de boîte de réception côté serveur |
| `Get-CalendarProcessing -Identity` | Affiche la configuration de réservation ; pertinente pour les boîtes aux lettres de salle et d’équipement |
| `-ErrorAction SilentlyContinue` | Supprime l’erreur pour les types de boîte aux lettres sans configuration de réservation, afin que la sauvegarde ne s’interrompe pas |

</details>

Ces configurations ne sont pas transférées automatiquement vers la nouvelle boîte aux lettres cloud.

## Phase 3 : dissocier la boîte aux lettres locale et activer RemoteMailbox

Le basculement effectif est bref, mais lourd de conséquences. `Disable-Mailbox` supprime les attributs Exchange de l’utilisateur AD et dissocie la boîte aux lettres locale. Les données de boîte aux lettres restent disponibles en tant que disconnected mailbox jusqu’à l’expiration de la rétention de la base de données. Juste après, `Enable-RemoteMailbox` active le même utilisateur AD pour Exchange Online.

Le script suivant ne traite que les lignes doublement approuvées. Il conserve l’adresse SMTP principale, toutes les adresses proxy existantes et l’ancien `LegacyExchangeDN` comme adresse X500. L’entrée X500 évite les NDR lors de réponses à des messages anciens ou avec d’anciennes entrées d’auto-complétion Outlook.

```powershell
$CsvPath = "C:\Migration\mailboxes.csv"
$PstShare = "\\fileserver\exchange-pst$"

$targets = Import-Csv $CsvPath |
    Where-Object { $_.Action -eq "CUTOVER" -and $_.PstVerified -eq "YES" }

foreach ($row in $targets) {
    $safeName = ($row.PrimarySmtpAddress -replace '[^a-zA-Z0-9@._-]', '_')
    $pstPath = "$PstShare\$safeName.pst"

    if (-not (Test-Path $pstPath)) {
        throw "PST fehlt: $pstPath"
    }

    if ((Get-Item $pstPath).Length -eq 0) {
        throw "PST ist leer: $pstPath"
    }

    $mailbox = Get-Mailbox -Identity $row.Identity -ErrorAction Stop
    $primary = $mailbox.PrimarySmtpAddress.ToString()
    $legacyDn = $mailbox.LegacyExchangeDN

    if ($primary -ine $row.PrimarySmtpAddress) {
        throw "Primaere SMTP-Adresse stimmt nicht mit der Freigabeliste ueberein: $($row.Identity)"
    }

    $preservedAddresses = @($mailbox.EmailAddresses | ForEach-Object ToString)

    Disable-Mailbox -Identity $row.Identity -Confirm:$false

    $enableParams = @{
        Identity             = $row.Identity
        Alias                = $row.Alias
        PrimarySmtpAddress   = $primary
        RemoteRoutingAddress = $row.RemoteRoutingAddress
        ACLableSyncedObjectEnabled = $true
    }

    switch ($row.SourceType) {
        "SharedMailbox"    { $enableParams.Shared = $true }
        "RoomMailbox"      { $enableParams.Room = $true }
        "EquipmentMailbox" { $enableParams.Equipment = $true }
        "UserMailbox"      { }
        default             { throw "Nicht unterstuetzter Mailbox-Typ: $($row.SourceType)" }
    }

    Enable-RemoteMailbox @enableParams

    $orderedAddresses = @(
        "SMTP:$primary"
        $preservedAddresses |
            Where-Object { $_ -notmatch '^SMTP:' -or $_.Substring(5) -ine $primary }
        "smtp:$($row.RemoteRoutingAddress)"
        "X500:$legacyDn"
    )

    $seen = [System.Collections.Generic.HashSet[string]]::new(
        [System.StringComparer]::OrdinalIgnoreCase
    )
    $finalAddresses = foreach ($address in $orderedAddresses) {
        if ($seen.Add($address)) { $address }
    }

    Set-RemoteMailbox -Identity $row.Identity -EmailAddressPolicyEnabled $false
    Set-RemoteMailbox -Identity $row.Identity -EmailAddresses $finalAddresses
    Set-RemoteMailbox -Identity $row.Identity -PrimarySmtpAddress $primary

    Get-RemoteMailbox -Identity $row.Identity |
        Format-List DisplayName,RecipientTypeDetails,PrimarySmtpAddress,
            RemoteRoutingAddress,EmailAddresses
}
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `Get-Mailbox -Identity ... -ErrorAction Stop` | Récupère la boîte aux lettres source ; `-ErrorAction Stop` transforme une erreur de recherche en erreur bloquante au lieu de poursuivre silencieusement |
| `Disable-Mailbox -Identity` | Supprime les attributs Exchange de l’utilisateur AD et dissocie la boîte aux lettres locale ; les données restent en tant que disconnected mailbox dans la base de données |
| `-Confirm:$false` | Supprime la confirmation interactive ; l’approbation est effectuée ici par la liste CSV, et non via l’invite |
| `Enable-RemoteMailbox -Identity` | Active le même utilisateur AD comme RemoteMailbox pour Exchange Online |
| `-Alias` | Rétablit l’alias Exchange à la valeur de la liste de validation |
| `-PrimarySmtpAddress` | Conserve l’adresse SMTP principale existante |
| `-RemoteRoutingAddress` | Adresse cible dans le domaine de routage `mail.onmicrosoft.com`, par lequel l’Exchange local atteint la boîte aux lettres cloud |
| `-ACLableSyncedObjectEnabled` | Marque l’objet comme compatible ACL afin que les autorisations telles que Full Access restent exploitables dans Exchange Online après la synchronisation |
| `-Shared` / `-Room` / `-Equipment` | Crée le type spécial correspondant au lieu d’une boîte aux lettres utilisateur ; le script définit exactement un commutateur selon le type source |
| `Set-RemoteMailbox -EmailAddressPolicyEnabled $false` | Exclut l’objet de la stratégie d’adresses e-mail afin qu’elle ne remplace pas les adresses définies manuellement |
| `-EmailAddresses` | Définit la liste complète d’adresses dédupliquée, y compris les anciennes adresses proxy, l’adresse de routage et l’entrée X500 |
| `Get-RemoteMailbox -Identity` | Requête de contrôle du résultat directement après la bascule |

</details>

Le script n’est volontairement pas un outil de migration entièrement automatisé. Il arrête l’exécution à la première incohérence afin qu’un administrateur puisse évaluer la cause et l’état. Avant un lot de production, le code doit être validé avec quelques boîtes aux lettres de test et les versions Exchange utilisées.

## Phase 4 : synchroniser, attribuer les licences et vérifier

Après la modification locale, un cycle delta est démarré sur le serveur Entra Connect :

```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-PolicyType Delta` | Synchronise uniquement les objets modifiés depuis le dernier cycle ; l’alternative `Initial` exécuterait une synchronisation complète, nettement plus longue |

</details>

Pour les boîtes aux lettres utilisateur, un plan de service Exchange Online valide doit ensuite être attribué, par exemple via l’attribution de licences basée sur les groupes. Les boîtes aux lettres partagées, de salle et d’équipement doivent être évaluées selon les conditions de licence Microsoft actuelles et les fonctionnalités requises.

Le provisionnement est asynchrone. Microsoft indique généralement moins de 30 minutes pour les modifications courantes, mais dans certains cas jusqu’à 24 heures. Pendant cette période, le flux de messagerie en amont doit retenir les messages de manière contrôlée au lieu de les remettre à une cible qui n’est pas encore prête.

Le contrôle local doit désormais afficher une RemoteMailbox :

```powershell
Get-RemoteMailbox -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,RemoteRoutingAddress,EmailAddresses

Get-Mailbox -Identity user01@contoso.com -ErrorAction SilentlyContinue
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `Get-RemoteMailbox -Identity` | Doit renvoyer l’objet basculé comme RemoteMailbox |
| `Format-List RecipientTypeDetails,...` | Affiche le type, les adresses et l’adresse de routage sous forme de liste à des fins de contrôle |
| `Get-Mailbox -Identity ... -ErrorAction SilentlyContinue` | Contre-vérification : l’appel ne doit plus rien renvoyer, car aucune boîte aux lettres connectée n’existe plus localement ; `-ErrorAction SilentlyContinue` supprime le message d’erreur attendu |

</details>

Dans Exchange Online, il faut vérifier que l’ancien MailUser est devenu une véritable boîte aux lettres :

```powershell
Get-EXORecipient -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,EmailAddresses

Get-EXOMailbox -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,ExchangeGuid
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `Get-EXORecipient -Identity` | Affiche le type de destinataire dans Exchange Online ; `UserMailbox` ou le type spécial est attendu, et non plus `MailUser` |
| `Get-EXOMailbox -Identity` | Ne renvoie que les véritables boîtes aux lettres cloud ; un résultat prouve que le provisionnement est terminé |
| `Format-List ...,ExchangeGuid` | Répertorie les attributs de contrôle ; le `ExchangeGuid` identifie sans ambiguïté la nouvelle boîte aux lettres cloud |

</details>

Le lot n’est considéré comme terminé que si les tests suivants réussissent également :

- Réception depuis l’externe et l’interne
- Envoi vers l’externe et l’interne
- Réponse à un ancien message afin de vérifier l’adresse X500
- Connexion avec Outlook et Outlook sur le web
- Délégations et Send-As
- Transferts et règles de transport
- Réservations de salles et d’équipements
- Applications, scanners et relais SMTP
- Message Trace avec remise à la nouvelle boîte aux lettres cloud

## Rollback et nettoyage

La boîte aux lettres source locale ne doit pas être supprimée avec `Remove-StoreMailbox` pendant la phase de validation. Tant qu’elle est présente comme disconnected mailbox dans la rétention de la boîte aux lettres, une possibilité technique de retour existe encore. Un rollback exige toutefois une inversion contrôlée des attributs RemoteMailbox et la reconnexion de la boîte aux lettres locale ; il faut simultanément empêcher l’apparition de deux destinations de remise actives.

Avant un rollback, le flux de messagerie, l’état de synchronisation et les messages déjà reçus dans le cloud doivent donc être sauvegardés. Un retour en arrière n’est pas une simple commande sur une ligne et doit faire partie du changement sous la forme d’un runbook testé.

Après validation réussie, les demandes d’exportation sont nettoyées, les fichiers PST sont archivés conformément au concept de protection et de rétention, et les autorisations temporaires sur le partage d’exportation sont supprimées :

```powershell
Get-MailboxExportRequest -BatchName $BatchName |
    Remove-MailboxExportRequest -Confirm:$false
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `Get-MailboxExportRequest -BatchName` | Sélectionne précisément les demandes d’exportation du lot terminé |
| `Remove-MailboxExportRequest -Confirm:$false` | Supprime les demandes sans confirmation ; les fichiers PST eux-mêmes ne sont pas concernés |

</details>

Les disconnected mailboxes ne doivent être définitivement nettoyées qu’après expiration de la fenêtre de rollback convenue et conformément au concept de rétention.

## Conclusion

Lorsque les Hybrid Remote Moves ne sont pas autorisés et qu’aucune donnée de boîte aux lettres ne doit être reprise dans Exchange Online, il est possible de basculer de manière contrôlée un utilisateur AD synchronisé existant d’une boîte aux lettres locale vers une nouvelle boîte aux lettres cloud. La partie critique n’est pas `Enable-RemoteMailbox`, mais le contrôle du processus qui l’entoure : inventaire complet, sauvegarde PST vérifiée, validations explicites, conservation des adresses proxy et X500, flux de messagerie contrôlé et véritable fenêtre de rollback.

## Historique des modifications

- **25 octobre 2026 :** lien source mis à jour vers la nouvelle adresse de Microsoft Learn.

## Sources

1. [Microsoft Learn – Enable-RemoteMailbox](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox?view=exchange-ps): active un utilisateur AD local existant pour une boîte aux lettres dans le service cloud et documente les commutateurs pour les boîtes aux lettres utilisateur, partagées, de salle et d’équipement.

2. [Microsoft Learn – New-MailboxExportRequest](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-mailboxexportrequest?view=exchange-ps): référence pour l’exportation des boîtes aux lettres locales principales et archivées vers des fichiers PST.

3. [Microsoft Learn – Mailbox import and export requests](https://learn.microsoft.com/en-us/exchange/mailbox-import-and-export-requests-exchange-2013-help): prérequis relatifs au partage d’exportation, aux autorisations et à Exchange Trusted Subsystem.

4. [Microsoft Learn – Disable or delete a mailbox in Exchange Server](https://learn.microsoft.com/en-us/exchange/recipients/disconnected-mailboxes/disable-or-delete-mailboxes): comportement de `Disable-Mailbox`, suppression des attributs Exchange et conservation de la boîte aux lettres dissociée.

5. [Microsoft Learn – Disconnected mailboxes](https://learn.microsoft.com/en-us/exchange/recipients/disconnected-mailboxes/disconnected-mailboxes): reconnexion, restauration et suppression définitive des boîtes aux lettres dissociées.

6. [Microsoft Learn – Delays in provisioning of a user or mailbox](https://learn.microsoft.com/en-us/troubleshoot/exchange/user-and-shared-mailboxes/delays-provision-mailbox-sync-changes): durée habituelle et dépannage du provisionnement dans Exchange Online.

7. [Microsoft Learn – Move mailboxes between on-premises and Exchange Online organizations](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/move-mailboxes): le Hybrid Remote Move standard comme référence et distinction par rapport à la recréation décrite ici.
