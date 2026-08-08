---
title: "Migration EXO sans Remote Move"
navTitle: "Migration EXO sans Remote Move"
description: "Comment provisionner de manière contrôlée des boîtes aux lettres Exchange locales comme nouvelles boîtes aux lettres Exchange Online vides : sauvegarde PST, validation CSV, RemoteMailbox, synchronisation, validation et rollback."
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
url: https://rafaelpfister.ch/fr/blog/migration-exo-sans-remote-move
translationSourceHash: 861f11b6e2f1e316ca773f049637fa2ac6ed5efdab5ec74d8c28178f3ea7e98c
translationModel: gpt-5.6-terra
translatedAt: 2026-08-08T13:47:48.520Z
translationReview: automatic
---

# Migration EXO sans Remote Move

Un Hybrid Remote Move est la méthode habituelle pour déplacer une boîte aux lettres Exchange locale, y compris son contenu, vers Exchange Online. Toutes les organisations n’autorisent pas cette voie de migration. Lorsqu’une politique de sécurité exclut les Remote Moves, une approche délibérément différente peut être acceptable : la boîte aux lettres locale est sauvegardée au format PST, dissociée de l’utilisateur AD synchronisé, puis une nouvelle boîte aux lettres vide est provisionnée dans Exchange Online pour ce même utilisateur.

Cette méthode **n’est pas une migration de boîte aux lettres**. Elle ne transfère ni les messages, ni les calendriers, ni les règles, ni les autorisations vers le cloud. Le PST sert exclusivement de sauvegarde et n’est pas importé dans ce scénario. Cette procédure ne convient donc que lorsqu’une boîte aux lettres cible vide est acceptable sur le plan fonctionnel et que la perte de la configuration active de la boîte aux lettres est explicitement approuvée.

## État cible et prérequis impératifs

Après le cutover, le même utilisateur AD reste en place. Localement, il n’est toutefois plus géré comme `UserMailbox`, `SharedMailbox`, `RoomMailbox` ou `EquipmentMailbox`, mais comme `RemoteMailbox`. Après la synchronisation, cet objet représente la nouvelle boîte aux lettres dans Exchange Online.

L’état souhaité est le suivant :

1. La boîte aux lettres locale est entièrement sauvegardée au format PST.
2. La boîte aux lettres locale est dissociée, mais pas encore définitivement supprimée pendant la période de rétention configurée.
3. L’utilisateur AD existant est activé en tant que RemoteMailbox.
4. L’adresse principale, les alias et l’ancien `LegacyExchangeDN` sont conservés.
5. Entra Connect a synchronisé les modifications.
6. Un plan de service Exchange Online est attribué aux boîtes aux lettres utilisateur.
7. Exchange Online affiche une véritable boîte aux lettres cloud et le flux de messagerie y aboutit.

Les points suivants doivent également être clarifiés avant le démarrage :

- Le partage PST est accessible via UNC. Le groupe `Exchange Trusted Subsystem` y dispose des droits de lecture et d’écriture.
- Le compte exécutant dispose du rôle de gestion `Mailbox Import Export`.
- Le PST constitue uniquement la sauvegarde convenue ; aucune importation ultérieure n’est prévue.
- La rétention, Litigation Hold, eDiscovery et les exigences réglementaires ont été vérifiées séparément.
- Les délégations, Send-As, Send-on-Behalf, transferts, règles de boîte de réception, appareils mobiles et accès applicatifs sont inventoriés.
- Pendant l’exportation et le basculement, les messages entrants sont retenus de manière contrôlée au niveau de la passerelle en amont. Les utilisateurs et les applications ne doivent plus écrire dans la boîte aux lettres source.
- La rétention de la base de données de boîtes aux lettres locale couvre la fenêtre de rollback.

## Pourquoi une liste d’approbation CSV est indispensable

Un pipeline direct tel que `Get-Mailbox | Disable-Mailbox` est trop risqué pour cette opération. Il pourrait également inclure des boîtes aux lettres système, de découverte ou d’autres boîtes non approuvées. La procédure suivante repose donc sur deux validations explicites :

- `Action=CUTOVER` détermine quelle ligne peut effectivement être basculée.
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

Le fichier est ensuite nettoyé sur le plan fonctionnel. Seules les boîtes aux lettres effectivement approuvées reçoivent `Action=CUTOVER`. Les boîtes aux lettres système et les objets spéciaux ne doivent pas figurer dans cette liste.

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

L’exportation n’est approuvée que lorsque **chaque** requête présente le statut `Completed` :

```powershell
Get-MailboxExportRequest -BatchName $BatchName |
    Get-MailboxExportRequestStatistics -IncludeReport |
    Format-Table DisplayName,Status,PercentComplete,FailureCode,Message -AutoSize

Get-MailboxExportRequest -BatchName $BatchName |
    Where-Object Status -ne "Completed"
```

L’existence, la taille, la lisibilité, la prise en charge par la sauvegarde et la protection d’accès des fichiers doivent également être vérifiées. Ce n’est qu’ensuite que `PstVerified=YES` est défini pour la ligne CSV correspondante.

## Phase 2 : sauvegarder les données de boîte aux lettres et les attributs Exchange

Avant la première modification, un snapshot lisible par machine est créé pour chaque boîte aux lettres. Il est plus important qu’une capture d’écran, car les alias, GUID et le `LegacyExchangeDN` peuvent ensuite être reconstruits avec précision :

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

Les délégations et les transferts nécessitent des exportations distinctes. Au minimum, les informations suivantes doivent être sauvegardées séparément :

```powershell
$mailbox = Get-Mailbox -Identity user01@contoso.com

$mailbox | Format-List ForwardingAddress,ForwardingSmtpAddress,DeliverToMailboxAndForward
Get-MailboxPermission -Identity $mailbox.Identity
Get-ADPermission -Identity $mailbox.DistinguishedName
Get-InboxRule -Mailbox $mailbox.Identity
Get-CalendarProcessing -Identity $mailbox.Identity -ErrorAction SilentlyContinue
```

Ces configurations ne sont pas automatiquement transférées vers la nouvelle boîte aux lettres cloud.

## Phase 3 : dissocier la boîte aux lettres locale et activer RemoteMailbox

Le cutover proprement dit est court, mais lourd de conséquences. `Disable-Mailbox` supprime les attributs Exchange de l’utilisateur AD et dissocie la boîte aux lettres locale. Les données de la boîte aux lettres sont conservées comme boîte aux lettres déconnectée jusqu’à l’expiration de la rétention de la base de données. Juste après, `Enable-RemoteMailbox` active le même utilisateur AD pour Exchange Online.

Le script suivant traite exclusivement les lignes doublement approuvées. Il conserve l’adresse SMTP principale, toutes les adresses proxy existantes et l’ancien `LegacyExchangeDN` en tant qu’adresse X500. L’entrée X500 empêche les NDR lors de réponses à d’anciens messages ou en présence d’anciennes entrées de saisie semi-automatique Outlook.

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

Le script n’est volontairement pas un moteur de migration entièrement autonome. Il arrête l’exécution à la première incohérence afin qu’un administrateur puisse évaluer la cause et l’état. Avant un lot de production, le code doit être validé avec quelques boîtes aux lettres de test et les versions Exchange utilisées.

## Phase 4 : synchroniser, attribuer les licences et vérifier

Après la modification locale, un cycle delta est lancé sur le serveur Entra Connect :

```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```

Pour les boîtes aux lettres utilisateur, un plan de service Exchange Online valide doit ensuite être attribué, par exemple via l’attribution de licences basée sur les groupes. Les boîtes aux lettres partagées, de salle et d’équipement doivent être évaluées selon les conditions de licence Microsoft actuelles et les fonctions nécessaires.

Le provisionnement est asynchrone. Microsoft indique généralement moins de 30 minutes pour les modifications normales, mais jusqu’à 24 heures dans certains cas. Pendant ce temps, le flux de messagerie en amont doit retenir les messages de manière contrôlée plutôt que de les remettre à une destination qui n’est pas encore prête.

Le contrôle local doit désormais afficher une RemoteMailbox :

```powershell
Get-RemoteMailbox -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,RemoteRoutingAddress,EmailAddresses

Get-Mailbox -Identity user01@contoso.com -ErrorAction SilentlyContinue
```

Dans Exchange Online, il convient de vérifier si l’ancien MailUser est devenu une véritable boîte aux lettres :

```powershell
Get-EXORecipient -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,EmailAddresses

Get-EXOMailbox -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,ExchangeGuid
```

Le lot n’est considéré comme terminé que lorsque les tests suivants ont également réussi :

- Réception depuis l’externe et l’interne
- Envoi vers l’externe et l’interne
- Réponse à un ancien message afin de vérifier l’adresse X500
- Connexion avec Outlook et Outlook sur le Web
- Délégations et Send-As
- Transferts et règles de transport
- Réservations de salles et d’équipements
- Applications, scanners et relais SMTP
- Message Trace avec remise à la nouvelle boîte aux lettres cloud

## Rollback et nettoyage

La boîte aux lettres source locale ne doit pas être supprimée avec `Remove-StoreMailbox` pendant la phase de validation. Tant qu’elle existe comme boîte aux lettres déconnectée dans la période de rétention de la boîte aux lettres, une possibilité de retour technique demeure. Un rollback exige toutefois une inversion contrôlée des attributs RemoteMailbox et la reconnexion de la boîte aux lettres locale ; il faut simultanément empêcher l’apparition de deux destinations de remise actives.

Avant un rollback, le flux de messagerie, l’état de synchronisation et les messages déjà reçus dans le cloud doivent donc être sauvegardés. Un retour en arrière n’est pas une simple commande sur une ligne et doit faire partie du changement sous la forme d’un runbook testé.

Après réception réussie, les requêtes d’exportation sont nettoyées, les fichiers PST sont archivés conformément au concept de protection et de rétention, et les autorisations temporaires sur le partage d’exportation sont supprimées :

```powershell
Get-MailboxExportRequest -BatchName $BatchName |
    Remove-MailboxExportRequest -Confirm:$false
```

Les boîtes aux lettres déconnectées ne doivent être définitivement nettoyées qu’après l’expiration de la fenêtre de rollback convenue et conformément au concept de rétention.

## Conclusion

Lorsque les Hybrid Remote Moves ne sont pas autorisés et qu’aucune donnée de boîte aux lettres ne doit être reprise dans Exchange Online, il est possible de basculer de manière contrôlée un utilisateur AD synchronisé existant d’une boîte aux lettres locale vers une nouvelle boîte aux lettres cloud. La partie critique n’est pas `Enable-RemoteMailbox`, mais le contrôle du processus qui l’entoure : inventaire complet, sauvegarde PST vérifiée, validations explicites, conservation des adresses proxy et X500, flux de messagerie contrôlé et véritable fenêtre de rollback.

## Sources

1. [Microsoft Learn – Enable-RemoteMailbox](https://learn.microsoft.com/en-us/powershell/module/exchange/enable-remotemailbox?view=exchange-ps): Active un utilisateur AD local existant pour une boîte aux lettres dans le service cloud et documente les paramètres pour les boîtes aux lettres utilisateur, partagées, de salle et d’équipement.

2. [Microsoft Learn – New-MailboxExportRequest](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-mailboxexportrequest?view=exchange-ps): Référence pour l’exportation de boîtes aux lettres locales principales et archivées vers des fichiers PST.

3. [Microsoft Learn – Mailbox import and export requests](https://learn.microsoft.com/en-us/exchange/mailbox-import-and-export-requests-exchange-2013-help): Prérequis concernant le partage d’exportation, les autorisations et Exchange Trusted Subsystem.

4. [Microsoft Learn – Disable or delete a mailbox in Exchange Server](https://learn.microsoft.com/en-us/exchange/recipients/disconnected-mailboxes/disable-or-delete-mailboxes): Comportement de `Disable-Mailbox`, suppression des attributs Exchange et rétention de la boîte aux lettres dissociée.

5. [Microsoft Learn – Disconnected mailboxes](https://learn.microsoft.com/en-us/exchange/recipients/disconnected-mailboxes/disconnected-mailboxes): Reconnexion, restauration et suppression définitive des boîtes aux lettres déconnectées.

6. [Microsoft Learn – Delays in provisioning of a user or mailbox](https://learn.microsoft.com/en-us/troubleshoot/exchange/user-and-shared-mailboxes/delays-provision-mailbox-sync-changes): Durée habituelle et dépannage du provisionnement dans Exchange Online.

7. [Microsoft Learn – Move mailboxes between on-premises and Exchange Online organizations](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/move-mailboxes): Le Hybrid Remote Move standard comme référence et distinction par rapport à la recréation décrite ici.
