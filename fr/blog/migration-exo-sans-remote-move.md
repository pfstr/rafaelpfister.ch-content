---
title: "Migration EXO sans Remote Move"
navTitle: "Migration EXO sans Remote Move"
description: "Comment provisionner de manière contrôlée des boîtes aux lettres Exchange locales sous forme de nouvelles boîtes aux lettres Exchange Online vides : sauvegarde PST, approbation CSV, RemoteMailbox, synchronisation, validation et rollback."
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
translationSourceHash: dc64d2c419e3ac0f4dd730785b3cd7f37c3f23effd2317feb4d61a46fa33401a
translationModel: gpt-5.6-terra
translatedAt: 2026-08-31T10:19:46.261Z
translationReview: required
url: https://rafaelpfister.ch/fr/blog/migration-exo-sans-remote-move
---

# Migration EXO sans Remote Move

Un Hybrid Remote Move est la voie habituelle pour déplacer une boîte aux lettres Exchange locale avec son contenu vers Exchange Online. Toutes les organisations n'autorisent pas ce chemin de migration. Lorsqu'une politique de sécurité exclut les Remote Moves, une approche délibérément différente peut être justifiée : la boîte aux lettres locale est sauvegardée au format PST, dissociée de l'utilisateur AD synchronisé, puis une nouvelle boîte aux lettres vide est provisionnée dans Exchange Online pour ce même utilisateur.

Cette approche n'est **pas une migration de boîte aux lettres**. Elle ne transfère ni messages, ni calendriers, ni règles, ni autorisations vers le cloud. Le PST sert exclusivement de sauvegarde et n'est pas importé dans ce scénario. Cette procédure ne convient donc que si une boîte aux lettres cible vide est acceptable sur le plan métier et si la perte de la configuration active de la boîte aux lettres est explicitement approuvée.

## État cible et prérequis stricts

Après le cutover, le même utilisateur AD est conservé. Localement, il n'est toutefois plus géré en tant que `UserMailbox`, `SharedMailbox`, `RoomMailbox` ou `EquipmentMailbox`, mais en tant que `RemoteMailbox`. Après la synchronisation, cet objet représente la nouvelle boîte aux lettres dans Exchange Online.

L'état souhaité est le suivant :

1. La boîte aux lettres locale est entièrement sauvegardée au format PST.
2. La boîte aux lettres locale est dissociée, mais pas encore définitivement supprimée pendant la période de rétention configurée.
3. L'utilisateur AD existant est activé en tant que RemoteMailbox.
4. L'adresse principale, les alias et l'ancien `LegacyExchangeDN` sont conservés.
5. Entra Connect a synchronisé les modifications.
6. Un plan de service Exchange Online est attribué aux boîtes aux lettres utilisateur.
7. Exchange Online affiche une véritable boîte aux lettres cloud et le flux de messagerie y aboutit.

Les points suivants doivent également être clarifiés avant le démarrage :

- Le partage PST est accessible via UNC. Le groupe `Exchange Trusted Subsystem` y dispose des droits de lecture et d'écriture.
- Le compte exécutant dispose du rôle de gestion `Mailbox Import Export`.
- Le PST constitue uniquement la sauvegarde convenue ; aucune importation ultérieure n'est prévue.
- La conservation, Litigation Hold, eDiscovery et les exigences réglementaires sont vérifiées séparément.
- Les délégations, Send-As, Send-on-Behalf, redirections, règles de boîte de réception, appareils mobiles et accès applicatifs sont inventoriés.
- Pendant l'exportation et le basculement, les messages entrants sont retenus de manière contrôlée au niveau de la passerelle en amont. Les utilisateurs et les applications ne doivent plus écrire dans la boîte aux lettres source.
- La rétention de la base de données de boîtes aux lettres locale couvre la fenêtre de rollback.

## Pourquoi une liste d'approbation CSV est indispensable

Un pipeline direct tel que `Get-Mailbox | Disable-Mailbox` est trop risqué pour cette opération. Il pourrait également inclure des boîtes aux lettres système, Discovery ou d'autres boîtes non approuvées. Le processus suivant s'appuie donc sur deux approbations explicites :

- `Action=CUTOVER` détermine quelle ligne peut réellement être basculée.
- `PstVerified=YES` confirme que le fichier d'exportation a été vérifié sur les plans technique et organisationnel.

Dans un premier temps, seul l'inventaire est généré :

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
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `Get-Mailbox -ResultSize Unlimited` | Supprime la limite standard de 1 000 résultats ; sans ce paramètre, des boîtes aux lettres manquent dans l'inventaire des grands environnements |
| `-RecipientTypeDetails UserMailbox,SharedMailbox,RoomMailbox,EquipmentMailbox` | Limite la requête aux quatre types de boîtes aux lettres à basculer ; les boîtes aux lettres système et Discovery sont exclues |
| `Sort-Object PrimarySmtpAddress` | Trie la sortie selon l'adresse SMTP principale afin que le fichier CSV reste ordonné de façon stable lors de la revue métier |
| `Export-Csv -Path` | Chemin cible du fichier CSV |
| `-NoTypeInformation` | Supprime l'en-tête de type `#TYPE ...`, que les anciennes versions de PowerShell écrivent sinon comme première ligne |
| `-Encoding UTF8` | Écrit le fichier en UTF-8 afin de préserver correctement les caractères accentués dans les noms d'affichage |

</details>

Le fichier est ensuite nettoyé sur le plan métier. Seules les boîtes aux lettres effectivement approuvées reçoivent `Action=CUTOVER`. Les boîtes aux lettres système et les objets spéciaux ne doivent pas figurer dans cette liste.

## Phase 1 : sauvegarder la boîte aux lettres principale et l'archive au format PST

`New-MailboxExportRequest` n'écrit que vers un chemin UNC. Un nom de fichier unique est généré pour chaque boîte aux lettres. Une archive en ligne active de l'Exchange local est exportée séparément :

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
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `Import-Csv $CsvPath` | Lit la liste d'approbation ; chaque ligne devient un objet dont les propriétés correspondent aux colonnes CSV |
| `Where-Object Action -eq "CUTOVER"` | Traite uniquement les lignes explicitement approuvées |
| `New-MailboxExportRequest -Mailbox` | Boîte aux lettres source de l'exportation, ici l'identité de la ligne CSV |
| `-FilePath` | Chemin cible du fichier PST ; il doit s'agir d'un chemin UNC, les chemins locaux sont refusés par le cmdlet |
| `-Name` | Nom unique de la demande ; permet ensuite d'associer précisément les exportations principale et d'archive |
| `-BatchName` | Regroupe toutes les demandes d'une exécution sous un nom de lot ; sert de base à la consultation du statut et au nettoyage |
| `-IsArchive` | Exporte l'archive en ligne au lieu de la boîte aux lettres principale ; d'où la seconde demande pour chaque boîte aux lettres avec archive active |

</details>

L'exportation n'est approuvée que lorsque **chaque** demande a le statut `Completed` :

```powershell
Get-MailboxExportRequest -BatchName $BatchName |
    Get-MailboxExportRequestStatistics -IncludeReport |
    Format-Table DisplayName,Status,PercentComplete,FailureCode,Message -AutoSize

Get-MailboxExportRequest -BatchName $BatchName |
    Where-Object Status -ne "Completed"
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `Get-MailboxExportRequest -BatchName` | Liste toutes les demandes d'exportation du lot indiqué |
| `Get-MailboxExportRequestStatistics -IncludeReport` | Complète les statistiques par le rapport d'historique détaillé, qui indique les causes d'erreur des demandes individuelles |
| `Format-Table ... -AutoSize` | Affichage tabulaire des propriétés mentionnées ; `-AutoSize` adapte la largeur des colonnes au contenu |
| `Where-Object Status -ne "Completed"` | Filtre toutes les demandes qui ne sont pas encore terminées ou qui ont échoué ; la sortie doit être vide avant de poursuivre |

</details>

Il convient également de vérifier l'existence, la taille, la lisibilité, la prise en charge par la sauvegarde et la protection d'accès des fichiers. Ce n'est qu'ensuite que `PstVerified=YES` est défini pour la ligne CSV correspondante.

## Phase 2 : sauvegarder les données de boîte aux lettres et les attributs Exchange

Avant la première modification, un snapshot lisible par machine est créé pour chaque boîte aux lettres. Il est plus important qu'une capture d'écran, car les alias, GUID et `LegacyExchangeDN` peuvent ensuite être reconstruits exactement :

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
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `New-Item -ItemType Directory -Path ... -Force` | Crée le répertoire des snapshots ; `-Force` supprime l'erreur s'il existe déjà |
| `Get-Mailbox -Identity` | Récupère l'objet de boîte aux lettres actuel pour chaque ligne CSV |
| `Select-Object Identity,...,ServerName` | Réduit l'objet aux attributs nécessaires à une reconstruction ultérieure (GUID, adresses, `LegacyExchangeDN`, base de données) |
| `Export-Clixml` | Sérialise l'objet sous forme de CLIXML conservant les types ; contrairement au CSV, les valeurs multiples telles que `EmailAddresses` sont intégralement conservées et peuvent être relues avec `Import-Clixml` |

</details>

Les délégations et les redirections nécessitent des exportations distinctes. Au minimum, les informations suivantes doivent être sauvegardées séparément :

```powershell
$mailbox = Get-Mailbox -Identity user01@contoso.com

$mailbox | Format-List ForwardingAddress,ForwardingSmtpAddress,DeliverToMailboxAndForward
Get-MailboxPermission -Identity $mailbox.Identity
Get-ADPermission -Identity $mailbox.DistinguishedName
Get-InboxRule -Mailbox $mailbox.Identity
Get-CalendarProcessing -Identity $mailbox.Identity -ErrorAction SilentlyContinue
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `Format-List ForwardingAddress,ForwardingSmtpAddress,DeliverToMailboxAndForward` | Affiche les trois attributs de redirection de la boîte aux lettres sous forme de liste |
| `Get-MailboxPermission -Identity` | Liste les autorisations de boîte aux lettres telles que Full Access |
| `Get-ADPermission -Identity` | Liste les autorisations AD sur l'objet utilisateur, dont Send-As ; attend ici le Distinguished Name |
| `Get-InboxRule -Mailbox` | Liste les règles de boîte de réception côté serveur |
| `Get-CalendarProcessing -Identity` | Affiche la configuration de réservation ; pertinente pour les boîtes aux lettres de salle et d'équipement |
| `-ErrorAction SilentlyContinue` | Supprime l'erreur pour les types de boîtes aux lettres sans configuration de réservation afin que la sauvegarde ne s'interrompe pas |

</details>

Ces configurations ne sont pas automatiquement transférées vers la nouvelle boîte aux lettres cloud.

## Phase 3 : dissocier la boîte aux lettres locale et activer RemoteMailbox

Le cutover proprement dit est court, mais lourd de conséquences. `Disable-Mailbox` supprime les attributs Exchange de l'utilisateur AD et dissocie la boîte aux lettres locale. Les données de boîte aux lettres sont conservées en tant que disconnected mailbox jusqu'à l'expiration de la rétention de la base de données. Juste après, `Enable-RemoteMailbox` active le même utilisateur AD pour Exchange Online.

Le script suivant traite exclusivement les lignes doublement approuvées. Il conserve l'adresse SMTP principale, toutes les adresses proxy existantes et l'ancien `LegacyExchangeDN` comme adresse X500. L'entrée X500 empêche les NDR lors de réponses à des messages anciens ou avec d'anciennes entrées de saisie semi-automatique Outlook.

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
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `Get-Mailbox -Identity ... -ErrorAction Stop` | Récupère la boîte aux lettres source ; `-ErrorAction Stop` transforme une erreur de recherche en erreur bloquante au lieu de poursuivre silencieusement |
| `Disable-Mailbox -Identity` | Supprime les attributs Exchange de l'utilisateur AD et dissocie la boîte aux lettres locale ; les données restent dans la base de données en tant que disconnected mailbox |
| `-Confirm:$false` | Supprime la demande de confirmation interactive ; l'approbation est effectuée ici via la liste CSV, et non à l'invite |
| `Enable-RemoteMailbox -Identity` | Active le même utilisateur AD comme RemoteMailbox pour Exchange Online |
| `-Alias` | Rétablit l'alias Exchange à la valeur de la liste d'approbation |
| `-PrimarySmtpAddress` | Conserve l'adresse SMTP principale existante |
| `-RemoteRoutingAddress` | Adresse cible dans le domaine de routage `mail.onmicrosoft.com`, via lequel l'Exchange local atteint la boîte aux lettres cloud |
| `-ACLableSyncedObjectEnabled` | Marque l'objet comme compatible ACL afin que des autorisations telles que Full Access restent exploitables dans Exchange Online après la synchronisation |
| `-Shared` / `-Room` / `-Equipment` | Crée le type spécial concerné au lieu d'une boîte aux lettres utilisateur ; le script définit exactement un commutateur selon le type source |
| `Set-RemoteMailbox -EmailAddressPolicyEnabled $false` | Exclut l'objet de la stratégie d'adresses e-mail afin qu'elle ne remplace pas les adresses définies manuellement |
| `-EmailAddresses` | Définit la liste complète d'adresses dédupliquée, y compris les anciennes adresses proxy, l'adresse de routage et l'entrée X500 |
| `Get-RemoteMailbox -Identity` | Requête de contrôle du résultat immédiatement après le basculement |

</details>

Le script n'est délibérément pas un outil de migration entièrement automatisé. Il arrête l'exécution à la première incohérence afin qu'un administrateur puisse évaluer la cause et l'état. Avant un lot de production, le code doit être validé avec quelques boîtes aux lettres de test et les versions Exchange utilisées.

## Phase 4 : synchroniser, attribuer les licences et vérifier

Après la modification locale, un cycle delta est démarré sur le serveur Entra Connect :

```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-PolicyType Delta` | Synchronise uniquement les objets modifiés depuis le dernier cycle ; l'alternative `Initial` effectuerait un cycle complet, nettement plus long |

</details>

Pour les boîtes aux lettres utilisateur, un plan de service Exchange Online valide doit ensuite être attribué, par exemple via l'attribution de licences basée sur les groupes. Les boîtes aux lettres partagées, de salle et d'équipement doivent être évaluées selon les conditions de licence Microsoft en vigueur et les fonctionnalités requises.

Le provisionnement est asynchrone. Microsoft indique généralement moins de 30 minutes pour les modifications ordinaires, mais jusqu'à 24 heures dans certains cas. Pendant cette période, le flux de messagerie en amont devrait retenir les messages de manière contrôlée au lieu de les remettre à une destination qui n'est pas encore prête.

Le contrôle local doit désormais afficher une RemoteMailbox :

```powershell
Get-RemoteMailbox -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,RemoteRoutingAddress,EmailAddresses

Get-Mailbox -Identity user01@contoso.com -ErrorAction SilentlyContinue
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `Get-RemoteMailbox -Identity` | Doit retourner l'objet basculé comme RemoteMailbox |
| `Format-List RecipientTypeDetails,...` | Affiche le type, les adresses et l'adresse de routage sous forme de liste pour contrôle |
| `Get-Mailbox -Identity ... -ErrorAction SilentlyContinue` | Contre-vérification : l'appel ne doit plus rien retourner, car aucune boîte aux lettres connectée n'existe plus localement ; `-ErrorAction SilentlyContinue` supprime le message d'erreur attendu |

</details>

Dans Exchange Online, vérifiez que l'ancien MailUser est devenu une véritable boîte aux lettres :

```powershell
Get-EXORecipient -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,EmailAddresses

Get-EXOMailbox -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,ExchangeGuid
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `Get-EXORecipient -Identity` | Affiche le type de destinataire dans Exchange Online ; `UserMailbox` ou le type spécial est attendu, et non plus `MailUser` |
| `Get-EXOMailbox -Identity` | Ne retourne que les véritables boîtes aux lettres cloud ; un résultat prouve que le provisionnement est terminé |
| `Format-List ...,ExchangeGuid` | Liste les attributs de contrôle ; `ExchangeGuid` identifie sans ambiguïté la nouvelle boîte aux lettres cloud |

</details>

Le lot n'est considéré comme terminé que lorsque les tests suivants ont également réussi :

- Réception depuis l'extérieur et l'intérieur
- Envoi vers l'extérieur et l'intérieur
- Réponse à un ancien message afin de vérifier l'adresse X500
- Connexion avec Outlook et Outlook sur le web
- Délégations et Send-As
- Redirections et règles de transport
- Réservations de salles et d'équipements
- Applications, scanners et relais SMTP
- Message Trace avec remise à la nouvelle boîte aux lettres cloud

## Rollback et nettoyage

La boîte aux lettres source locale ne doit pas être supprimée avec `Remove-StoreMailbox` pendant la phase de validation. Tant qu'elle est présente comme disconnected mailbox pendant la rétention de boîte aux lettres, une possibilité technique de retour reste disponible. Un rollback exige toutefois une inversion contrôlée des attributs RemoteMailbox et la reconnexion de la boîte aux lettres locale ; il faut simultanément empêcher la création de deux destinations de remise actives.

Avant un rollback, le flux de messagerie, l'état de synchronisation et les messages déjà reçus dans le cloud doivent donc être sauvegardés. Un retour n'est pas une simple commande sur une ligne et doit faire partie du changement sous forme de runbook testé.

Après validation réussie, les demandes d'exportation sont nettoyées, les fichiers PST sont archivés conformément au concept de protection et de conservation, et les autorisations temporaires sur le partage d'exportation sont supprimées :

```powershell
Get-MailboxExportRequest -BatchName $BatchName |
    Remove-MailboxExportRequest -Confirm:$false
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `Get-MailboxExportRequest -BatchName` | Sélectionne exactement les demandes d'exportation du lot terminé |
| `Remove-MailboxExportRequest -Confirm:$false` | Supprime les demandes sans confirmation ; les fichiers PST eux-mêmes ne sont pas affectés |

</details>

Les disconnected mailboxes ne doivent être définitivement nettoyées qu'après l'expiration de la fenêtre de rollback convenue et conformément au concept de rétention.

## Conclusion

Lorsque les Hybrid Remote Moves ne sont pas autorisés et qu'aucune donnée de boîte aux lettres ne doit être reprise dans Exchange Online, un utilisateur AD synchronisé existant peut être basculé de manière contrôlée d'une boîte aux lettres locale vers une nouvelle boîte aux lettres cloud. La partie critique n'est pas `Enable-RemoteMailbox`, mais le contrôle du processus qui l'entoure : inventaire complet, sauvegarde PST vérifiée, approbations explicites, conservation des adresses proxy et X500, flux de messagerie contrôlé et véritable fenêtre de rollback.

## Sources

1. [Microsoft Learn – Enable-RemoteMailbox](https://learn.microsoft.com/en-us/powershell/module/exchange/enable-remotemailbox?view=exchange-ps): Active un utilisateur AD local existant pour une boîte aux lettres dans le service cloud et documente les commutateurs pour les boîtes aux lettres utilisateur, partagées, de salle et d'équipement.

2. [Microsoft Learn – New-MailboxExportRequest](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-mailboxexportrequest?view=exchange-ps): Référence pour l'exportation de boîtes aux lettres locales principales et archivées vers des fichiers PST.

3. [Microsoft Learn – Mailbox import and export requests](https://learn.microsoft.com/en-us/exchange/mailbox-import-and-export-requests-exchange-2013-help): Prérequis relatifs au partage d'exportation, aux autorisations et à Exchange Trusted Subsystem.

4. [Microsoft Learn – Disable or delete a mailbox in Exchange Server](https://learn.microsoft.com/en-us/exchange/recipients/disconnected-mailboxes/disable-or-delete-mailboxes): Comportement de `Disable-Mailbox`, suppression des attributs Exchange et conservation de la boîte aux lettres dissociée.

5. [Microsoft Learn – Disconnected mailboxes](https://learn.microsoft.com/en-us/exchange/recipients/disconnected-mailboxes/disconnected-mailboxes): Reconnexion, restauration et suppression définitive des boîtes aux lettres dissociées.

6. [Microsoft Learn – Delays in provisioning of a user or mailbox](https://learn.microsoft.com/en-us/troubleshoot/exchange/user-and-shared-mailboxes/delays-provision-mailbox-sync-changes): Durée typique et résolution des problèmes de provisionnement dans Exchange Online.

7. [Microsoft Learn – Move mailboxes between on-premises and Exchange Online organizations](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/move-mailboxes): Le Hybrid Remote Move standard comme référence et distinction par rapport à la recréation décrite ici.
