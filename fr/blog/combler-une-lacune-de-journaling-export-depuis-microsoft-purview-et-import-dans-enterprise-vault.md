---
title: "Combler une lacune de journalisation : export depuis Microsoft Purview et import dans Enterprise Vault"
navTitle: "Lacune de journalisation"
description: "Si les rapports de journalisation d’Exchange Online sont interrompus pendant quelques jours, ils ne peuvent pas être générés a posteriori. Leur contenu se trouve toutefois encore dans les boîtes aux lettres. Découvrez comment l’exporter au format PST via la nouvelle eDiscovery de Purview, pourquoi le PST Migrator d’Enterprise Vault n’est généralement pas adapté et comment effectuer l’import ultérieur via la boîte aux lettres de journalisation sans évincer le flux de journalisation en cours, avec deux scripts génériques pour l’importation, le déplacement et la journalisation."
date: "2026-09-22"
kategorie: "Archivage et journalisation"
timeToRead: "18 min de lecture"
themen:
  - archivierung-journaling
  - microsoft-365-exchange
  - exchange-onprem-hybrid
produkte:
  - "exchange-online"
  - "exchange-on-premises"
protokolle:
  - "backup-dr"
  - "powershell"
  - "troubleshooting"
slug: "combler-une-lacune-de-journaling-export-depuis-microsoft-purview-et-import-dans-enterprise-vault"
translationId: "article-5b24118ac7567d96"
translationOf: journaling-luecke-purview-export-enterprise-vault-import
translationSourceHash: b22e076002b1662f409633c5b0b15584b7ef1d41cbe4d86f768096e0973f9d7c
translationModel: gpt-5.6-terra
translatedAt: 2026-10-27T09:38:09.743Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/fr/blog/combler-une-lacune-de-journaling-export-depuis-microsoft-purview-et-import-dans-enterprise-vault
---

# Combler une lacune de journalisation : export depuis Microsoft Purview et import dans Enterprise Vault

La journalisation est un processus de transport. Exchange génère le rapport de journalisation au moment où le message passe par le transport et le remet à la boîte aux lettres de journalisation. Si cette remise échoue, par exemple parce qu’un connecteur ou un objet destinataire est mal configuré, aucune fonction ne permet de récupérer ultérieurement le rapport. Ce qui a transité durant cette période manque dans l’archive.

Le contenu des messages reste toutefois dans les boîtes aux lettres des personnes concernées et, avec une stratégie de conservation sans expiration, également lorsque les utilisateurs ont supprimé les messages. Il n’existe donc qu’une seule approche praticable pour régulariser la situation : exporter la période depuis toutes les boîtes aux lettres et importer les copies dans l’archive de journalisation. La procédure suivante utilise la nouvelle eDiscovery de Microsoft Purview pour l’exportation et Veritas Enterprise Vault 15 pour l’importation ; les mesures proviennent d’un import ultérieur de plusieurs centaines de milliers de messages, lors duquel les deux scripts ont été créés.

## Ce qu’un export peut remplacer et ce qu’il ne peut pas remplacer

Un rapport de journalisation se compose de l’enveloppe et du contenu. L’enveloppe contient les destinataires effectivement remis par le transport, y compris les copies cachées et les membres résolus des listes de distribution. Une copie de boîte aux lettres ne contient pas ces informations. Après l’import ultérieur, il n’est plus possible de déterminer qui a reçu un message en copie cachée. En revanche, le contenu lui-même peut être entièrement reconstitué, à condition que deux conditions soient remplies.

Premièrement, une conservation doit être appliquée aux boîtes aux lettres afin que les messages supprimés se trouvent encore dans les éléments récupérables. Vérifiez cela dans PowerShell Sécurité et conformité :

```powershell
Connect-IPPSSession
Get-RetentionCompliancePolicy -DistributionDetail |
  Format-List Name, Enabled, Mode, ExchangeLocation, ExchangeLocationException
Get-RetentionComplianceRule |
  Format-List Name, Policy, RetentionComplianceAction, RetentionDuration
```

Une stratégie avec `RetentionComplianceAction: Keep` et `RetentionDuration: Unlimited` via `ExchangeLocation: All` protège intégralement le contenu. La liste d’exclusion dans `ExchangeLocationException` désigne les boîtes aux lettres auxquelles cela ne s’applique pas. Pour celles-ci, seuls les éléments encore présents peuvent être reconstitués.

Deuxièmement, un export compte les copies de boîtes aux lettres, et non les messages. Un message envoyé à quinze destinataires internes apparaît quinze fois dans le résultat. Dans le cas décrit, 98'131 copies pour une seule journée correspondaient à environ 6'800 messages uniques, déterminés à partir du suivi des messages. La manière de traiter ces copies relève de la conformité, non de l’informatique (voir la section sur la déduplication).

## Déterminer la période

Le début de la lacune se trouve dans la boîte aux lettres de journalisation elle-même, sur le serveur Exchange qui l’héberge. L’heure du dernier rapport remis marque le début :

```powershell
Get-MailboxFolderStatistics "journal@example.com" -IncludeOldestAndNewestItems |
  Where-Object { $_.ItemsInFolder -gt 0 } |
  Format-Table Name, ItemsInFolder, OldestItemReceivedDate, NewestItemReceivedDate -AutoSize
```

Il en va de même pour la boîte aux lettres indiquée dans `JournalingReportNdrTo` de la configuration de transport. Elle collecte les rapports de journalisation non distribuables, et son entrée la plus récente confirme l’heure. La fin de la lacune correspond au moment où la règle de journalisation a été redirigée vers la cible réparée. Entre les deux se trouve la période d’exportation, en UTC.

Le suivi des messages fournit le nombre de référence pour le rapprochement : une ligne par destinataire et par message, avec `MessageId`. Un rapport historique couvre toute la période :

```powershell
Connect-ExchangeOnline
$bericht = @{
  ReportTitle   = "Journal-Luecke"
  ReportType    = "MessageTrace"
  StartDate     = "2026-09-10"
  EndDate       = "2026-09-12"
  NotifyAddress = "admin@example.com"
}
Start-HistoricalSearch @bericht
```

Les valeurs `MessageId` uniques de ce rapport constituent le nombre auquel l’import ultérieur est comparé à la fin.

## Export depuis la nouvelle eDiscovery

Microsoft a désactivé les interfaces classiques de Content Search et eDiscovery (Standard) le 31 août 2025. Ce qui figure encore dans de nombreux guides, notamment l’option de déduplication à l’exportation, n’existe plus dans la nouvelle eDiscovery. Procédure dans le portail Purview :

1. Sous *eDiscovery*, créez ou ouvrez un dossier. Pour l’exportation, le compte exécutant doit disposer du rôle *eDiscovery Manager* ainsi que d’une licence Microsoft 365 E3 ou E5.
2. Créez une recherche. Source de données : toutes les boîtes aux lettres ou, pour un test, une seule boîte aux lettres.
3. Utilisez KQL comme requête. Les parenthèses sont essentielles, car `AND` a une priorité supérieure à `OR` :

```text
kind:email AND ((received>=2026-09-10 AND received<2026-09-12)
             OR (sent>=2026-09-10 AND sent<2026-09-12))
```

Sans `kind:email` , la recherche compte aussi les entrées de calendrier, les contacts et les tâches. Dans le cas décrit, cela faisait la différence entre 473'725 et 98'131 résultats pour une journée. Sans les parenthèses extérieures, la seconde condition de date s’applique seulement à `sent`, et le résultat contient des messages provenant de l’ensemble des boîtes aux lettres.

4. Exécutez *Generate statistics*. Pour les recherches à l’échelle du locataire, cela accélère considérablement l’exportation, car seules les boîtes aux lettres comportant des résultats sont alors traitées. Relancez les emplacements qui échouent via *Retry failed locations* avant l’exportation ; sinon, ils manqueront dans le résultat, et cela ne deviendra visible qu’après le téléchargement.
5. Lancez *Export* avec les paramètres suivants.

| Option | Effet |
|---|---|
| *Export type: Export items with items report* | Messages plus `items.csv`; le rapport seul ne contient aucun contenu |
| *Export format: Create PSTs for messages* | Un PST par boîte aux lettres au lieu de fichiers `.msg` individuels |
| *Maximum PST package size* | À partir de cette taille, un PST est scindé en parties (`.001.pst`, `.002.pst`) |
| *Maximum .zip package size* | Taille des paquets de téléchargement ; elle doit être au moins égale à celle du PST |
| *Organize data from different locations into separate folders or PSTs* | **Activer** : un fichier par boîte aux lettres, nommé `<smtp-adresse>.001.pst` |
| *Include folder and path of the source* | **Désactiver** : tous les messages sont placés dans le dossier `Items` du PST au lieu de l’arborescence de la boîte aux lettres |
| *Give each item a friendly name* | Sans effet lors d’un export PST |

Le paramètre *Include folder and path of the source* détermine l’importation ultérieure. Avec l’arborescence des dossiers, l’import crée dans la boîte aux lettres de journalisation les dossiers de chaque boîte aux lettres, dans la langue de chaque utilisateur : `Posteingang`, `Posta in arrivo`, `Boîte de réception`. Sans arborescence, tout se trouve dans un seul dossier `Items`, et l’import ultérieur dispose d’un emplacement unique à partir duquel poursuivre.

Le téléchargement fournit des paquets Zip. Microsoft recommande 7-Zip ou un outil comparable plutôt que l’Explorateur Windows. Les paquets expirent 14 jours après leur création. Le fichier `items.csv` du rapport de processus de l’exportation (sous *Process manager*, Export, Rapports) constitue la preuve de ce que l’export contient et de ce qui a été ignoré ; il fait partie du dossier.

### Déduplication

La Content Search classique proposait la case à cocher *Enable de-duplication* : les messages ayant le même `InternetMessageId`, `ConversationTopic` et `BodyTagInfo` n’étaient exportés qu’une seule fois, les autres emplacements étant indiqués dans `Results.csv`. La nouvelle eDiscovery ne propose pas cette fonctionnalité pour l’export direct depuis une recherche. La seule méthode documentée passe par un *Review Set* : chargez-y les résultats de recherche, exécutez Analytics, utilisez le filtre généré automatiquement *For Review*, qui exclut les doublons, puis exportez depuis le Review Set. Cela requiert eDiscovery Premium et donc une licence E5 pour l’utilisateur exécutant. Dans Microsoft Q&A, plusieurs utilisateurs signalent des doublons malgré cette procédure ; un essai sur une journée est recommandé avant l’export complet.

Sans déduplication, Enterprise Vault déduplique au niveau du stockage — les pièces jointes et corps de messages identiques ne sont stockés qu’une seule fois — mais pas au niveau des éléments. Chaque copie de boîte aux lettres devient une entrée d’archive distincte, un résultat de recherche distinct et un compteur distinct. Le décompte pour la preuve ne peut alors pas être déduit de l’archive, mais uniquement de `items.csv` comparé au suivi des messages. La décision d’accepter les copies ou de réexporter au moyen d’un Review Set se prend avant l’importation, non après.

## Les voies vers Enterprise Vault

Enterprise Vault propose le PST Migrator pour les fichiers PST, sous forme d’assistant dans la console d’administration (*Archives*, clic droit, *Import PST*) et sous forme scriptable via le Policy Manager EVPM. Pour l’import ultérieur dans une archive de journalisation, cette méthode présente trois limites.

Premièrement, la licence. Le Migrator requiert la fonctionnalité `EVPSTM` (*Exchange PST Migrator*). Dans un site qui ne fait que de la journalisation, elle n’est souvent pas sous licence, et l’assistant s’arrête dès la première page avec *Required license not installed*. Le Policy Manager utilise le même Migrator et affiche le même message.

Deuxièmement, la cible. L’assistant ne propose pas les archives de journalisation comme destination, seulement les archives de boîtes aux lettres et les archives Internet Mail. Un import direct dans l’archive de journalisation n’est possible que via EVPM avec `ArchiveName`, ce qui contourne les règles de traitement du Journaling Task. Il reste comme destination une Shared Archive distincte dans le Vault Store de la journalisation, que la recherche couvre conjointement avec l’archive de journalisation.

Troisièmement, la traçabilité : le Migrator écrit dans le fichier PST — il marque les éléments migrés —, modifiant ainsi le fichier, et les rapports de migration de chaque fichier doivent être rapprochés des chiffres d’exportation.

La seconde méthode ne nécessite pas de licence supplémentaire : l’Exchange Journaling Task d’Enterprise Vault n’archive pas seulement les rapports de journalisation. S’il reconnaît un élément comme rapport de journalisation, il l’ouvre et récupère les données d’enveloppe. Il archive directement un élément ordinaire, comme il l’a toujours fait à l’époque de la journalisation Exchange simple, sans enveloppe. En important les fichiers PST dans la boîte aux lettres de journalisation, les messages sont écrits par le Task régulier dans l’archive de journalisation régulière, avec la même catégorie de conservation, la même indexation et sans modification de l’architecture d’archivage.

Cela peut être prouvé avec un seul message de test envoyé à la boîte aux lettres de journalisation : s’il ne se trouve peu après dans aucun dossier, pas même dans *Invalid Journal Report*, et que la recherche le trouve dans l’archive de journalisation, le Task archive les messages ordinaires.

Cette méthode a deux propriétés qui déterminent la procédure. Le Task traite exclusivement le niveau supérieur de la boîte de réception, et non les sous-dossiers. Cela se vérifie dans la boîte aux lettres de journalisation : le dossier de recherche *Initial Trawl With Pending* sous *Enterprise Vault Search Folders* compte exactement les éléments de la boîte de réception ; les sous-dossiers ne sont pas inclus. Et l’importation via Exchange crée toujours des dossiers ; la manière dont cela se produit est expliquée dans la section suivante.

## Importer dans la boîte aux lettres de journalisation

`New-MailboxImportRequest` importe un fichier PST côté serveur via le Mailbox Replication Service. Le fichier doit se trouver sur un partage auquel *Exchange Trusted Subsystem* dispose d’un accès complet, et le compte exécutant doit avoir le rôle *Mailbox Import Export*, qui n’est attribué à personne par défaut :

```powershell
Get-ManagementRoleAssignment -Role "Mailbox Import Export" |
  Format-Table RoleAssigneeName, RoleAssigneeType -AutoSize
New-ManagementRoleAssignment -Role "Mailbox Import Export" -User "admin@example.com"
```

L’importation elle-même :

```powershell
$import = @{
  Mailbox          = "journal@example.com"
  FilePath         = "\\EVSERVER01\pstimport$\unzipped\user@example.com.001.pst"
  TargetRootFolder = "Inbox"
  Name             = "NI-user_example.com.001"
  BatchName        = "Nachimport"
}
New-MailboxImportRequest @import
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-Mailbox` | Boîte aux lettres cible, ici la boîte aux lettres de journalisation |
| `-FilePath` | Chemin UNC du fichier PST ; les chemins locaux sont refusés |
| `-TargetRootFolder` | Dossier dans lequel le contenu du PST est placé ; sans indication, les dossiers PST sont mappés sur des dossiers de boîte aux lettres du même nom |
| `-SourceRootFolder` | Dossier du PST à partir duquel l’importation est effectuée ; en pratique, le dossier lui-même est tout de même créé (voir texte) |
| `-Name` | Nom unique du travail ; dérivé du nom de fichier pour des milliers de fichiers |
| `-BatchName` | Regroupement pour les requêtes et statistiques |

</details>

Trois observations issues du test déterminent la suite de la procédure. Sans `TargetRootFolder` et avec un PST structuré (arborescence exportée), MRS crée la racine du PST comme dossier distinct à côté de la boîte de réception, dans le cas d’une boîte aux lettres allemande `Oberste-Ebene-des-Informationsspeichers` avec les sous-dossiers `Posteingang`, `Gesendete-Elemente` et ainsi de suite, ainsi que `Recoverable-Items` avec `Deletions`. Avec `TargetRootFolder "Inbox"` et un PST plat, tous les messages arrivent dans `/Inbox/Items`. Et la combinaison de `SourceRootFolder "Items"` et `TargetRootFolder "Inbox"` a également donné `/Inbox/Items` lors du test ; le niveau n’a pas été supprimé.

Dans les trois cas, les messages se trouvent hors de portée du Journaling Task. La dernière étape, consistant à les déplacer à plat vers la boîte de réception, ne peut pas être effectuée avec les outils Exchange intégrés ; elle passe par EWS. L’EWS Managed API se trouve sous la forme `Microsoft.Exchange.WebServices.dll` dans le répertoire d’installation d’Enterprise Vault, et le Vault Service Account dispose d’un accès complet à la boîte aux lettres de journalisation ; aucune autorisation supplémentaire n’est donc nécessaire. Le script s’exécute sur le serveur EV dans Windows PowerShell 5.1, car la bibliothèque repose sur le .NET Framework.

Cette étape de déplacement constitue en même temps le régulateur de toute la procédure : elle décide du nombre de messages que le Task voit à la fois.

## Le test avec une boîte aux lettres

Avant de traiter des milliers de fichiers, il est possible de vérifier l’ensemble du processus avec une seule boîte aux lettres. Il est judicieux d’utiliser la boîte aux lettres de la personne ayant exécuté l’exportation : elle a le mandat, il s’agit de ses propres données et le fichier est petit. Procédure :

1. Copiez le fichier PST dans un dossier séparé et enregistrez le hachage SHA-256 de l’original dans un fichier CSV.
2. Importez avec `TargetRootFolder "Inbox"`, puis obtenez la statistique avec `Get-MailboxImportRequestStatistics` : `ItemsTransferred` est le nombre de référence.
3. Consultez les statistiques des dossiers de la boîte aux lettres de journalisation : où se trouvent les éléments ?
4. Déplacez les éléments de `/Inbox/Items` vers la boîte de réception via EWS.
5. Comptez les anciens éléments de la boîte de réception toutes les quelques minutes : les éléments ayant `DateTimeReceived` avant le début du flux en cours sont les éléments importés. Si leur nombre baisse, le Task archive.
6. Effectuez le rapprochement : recherchez dans l’archive de journalisation (dans Discovery Accelerator ou la recherche EV) par période et propriétaire de boîte aux lettres, puis comparez le nombre de résultats à `ItemsTransferred`.

Dans le cas décrit, 128 éléments ont été importés et 126 archivés. Les deux restants étaient un brouillon et un élément sans expéditeur ; le Task laisse les éléments sans expéditeur en place. Les brouillons n’ont jamais traversé le transport et ne doivent pas figurer dans un journal ; le script de déplacement les exclut donc au moyen de l’indicateur `IsDraft`, indépendamment de la langue. Les éléments des dossiers *Versions* et *Conflits* — anciennes versions conservées et résidus de synchronisation — ne sont pas déplacés non plus.

Le journal des événements `Veritas Enterprise Vault` indique après le test ce que le Task a rejeté. L’événement 3071 (*could not be archived as it may be corrupt*) et l’événement 3288 (*no longer archive pending*) se répètent à chaque passage pour les mêmes éléments ; ces éléments doivent être déplacés vers un dossier hors de la boîte de réception afin que le Task ne tente pas de les traiter à nouveau à chaque passage.

## Script 1 : importation par lots

Le premier script s’exécute dans l’Exchange Management Shell sur un serveur Exchange. Il crée les demandes d’importation par lots, attend, consigne pour chaque fichier l’état et `ItemsTransferred` dans un fichier CSV, puis supprime les demandes terminées. Les exécutions répétées ignorent les fichiers déjà présents dans le journal. Le préfixe `NI` signifie import ultérieur.

```powershell
$NIPostfach  = "journal@example.com"
$NIQuellen   = @("\\EVSERVER01\pstimport$\unzipped", "\\EVSERVER01\pstimport$\unzipped2")
$NIProtokoll = "\\EVSERVER01\pstimport$\protokoll\import-protokoll.csv"

function Schreibe-NIImport($Ordner, $Datei, $Request, $Status, $Items, $Meldung) {
  [pscustomobject]@{
    Zeit = Get-Date -Format "yyyy-MM-dd HH:mm:ss"; Ordner = $Ordner; Datei = $Datei
    Request = $Request; Status = $Status; Items = $Items; Meldung = $Meldung
  } | Export-Csv -Path $NIProtokoll -Append -NoTypeInformation -Encoding UTF8
}

function Get-NIDateien {
  foreach ($q in $NIQuellen) {
    $o = Split-Path $q -Leaf
    Get-ChildItem $q -Filter *.pst | Sort-Object Name | ForEach-Object {
      [pscustomobject]@{
        Ordner = $o; Datei = $_.Name; Pfad = $_.FullName
        MB = [math]::Round($_.Length / 1MB, 1)
        Request = "NI-" + $o + "-" + ($_.Name -replace "@", "_" -replace "\.pst$", "")
      }
    }
  }
}

function Get-NIErledigt {
  if (Test-Path $NIProtokoll) {
    Import-Csv $NIProtokoll |
      Where-Object { $_.Status -in @("angelegt", "Completed", "CompletedWithWarning") } |
      Select-Object -ExpandProperty Request -Unique
  }
}

function Start-NICharge([int]$Anzahl = 100) {
  $erledigt = @(Get-NIErledigt)
  $alle  = @(Get-NIDateien)
  $offen = @($alle | Where-Object { $erledigt -notcontains $_.Request } | Select-Object -First $Anzahl)
  foreach ($d in $offen) {
    $auftrag = @{
      Mailbox          = $NIPostfach
      FilePath         = $d.Pfad
      TargetRootFolder = "Inbox"
      Name             = $d.Request
      BatchName        = "Nachimport-" + $d.Ordner
      ErrorAction      = "Stop"
    }
    try {
      $null = New-MailboxImportRequest @auftrag
      Schreibe-NIImport $d.Ordner $d.Datei $d.Request "angelegt" "" ""
    } catch {
      Schreibe-NIImport $d.Ordner $d.Datei $d.Request "Fehler-Anlage" "" $_.Exception.Message
    }
  }
  "angelegt: $($offen.Count)   verbleibend: $($alle.Count - $erledigt.Count - $offen.Count)"
}

function Wait-NICharge {
  do {
    $alle = @(Get-MailboxImportRequest -Mailbox $NIPostfach | Where-Object { $_.Name -like "NI-*" })
    $laufend = @($alle | Where-Object { $_.Status -notin @("Completed", "Failed", "CompletedWithWarning") })
    "$(Get-Date -Format HH:mm:ss)  laufend: $($laufend.Count)   fertig: $($alle.Count - $laufend.Count)"
    if ($laufend.Count -gt 0) { Start-Sleep -Seconds 60 }
  } while ($laufend.Count -gt 0)
  foreach ($r in $alle) {
    $s = $r | Get-MailboxImportRequestStatistics
    if (-not $s) { Schreibe-NIImport "" "" $r.Name "Statistik-Fehler" "" ""; continue }
    $ordner = $r.BatchName -replace "^Nachimport-", ""
    $datei  = if ($s.FilePath) { Split-Path $s.FilePath -Leaf } else { "" }
    Schreibe-NIImport $ordner $datei $r.Name $s.Status $s.ItemsTransferred $s.Message
    if ($s.Status -in @("Completed", "CompletedWithWarning")) {
      $r | Remove-MailboxImportRequest -Confirm:$false
    }
  }
  Get-MailboxStatistics $NIPostfach | Format-List ItemCount, TotalItemSize
}

function Resume-NICharge([int]$Anzahl = 200, [int]$MaxLager = 20000) {
  $stat  = Get-MailboxFolderStatistics $NIPostfach
  $lager = ($stat | Where-Object { $_.FolderPath -like "/Inbox/*" } | Measure-Object ItemsInFolder -Sum).Sum
  "in Unterordnern wartend: $lager"
  if ($lager -gt $MaxLager) { "Lager voll, nichts freigegeben"; return }
  Get-MailboxImportRequest -Mailbox $NIPostfach -Status Suspended |
    Select-Object -First $Anzahl |
    Resume-MailboxImportRequest -Confirm:$false
  "freigegeben: $Anzahl"
}

function Get-NIImportStand {
  $p = Import-Csv $NIProtokoll
  $p | Group-Object Status | Select-Object Name, Count | Format-Table -AutoSize
  "Elemente importiert: " + (($p | Where-Object { $_.Status -like "Completed*" } | Measure-Object Items -Sum).Sum)
  "Dateien gesamt: " + @(Get-NIDateien).Count
}
```

<details class="options-details">
<summary>Explication des fonctions</summary>

| Fonction | Effet |
|---|---|
| `Get-NIDateien` | Liste tous les fichiers PST des dossiers source avec un nom de demande dérivé ; les noms de fichiers identiques dans des dossiers différents — par exemple un export par jour — restent distinguables |
| `Start-NICharge -Anzahl` | Crée jusqu’à N demandes d’importation pour les fichiers qui ne sont pas encore consignés |
| `Wait-NICharge` | Attend qu’aucune demande ne soit plus en cours, écrit l’état et `ItemsTransferred` dans le journal, supprime les demandes terminées ; les demandes échouées restent en place pour analyse. Les statistiques et la suppression passent par le pipeline, car `-Identity` dans un shell distant n’accepte pas l’objet Identity désérialisé |
| `Resume-NICharge -Anzahl -MaxLager` | Ne libère les demandes suspendues que si moins de `MaxLager` messages attendent dans les sous-dossiers de la boîte de réception |
| `Get-NIImportStand` | Récapitulatif par état, somme des éléments importés, nombre total de fichiers |

</details>

Vous devez connaître deux comportements de MRS. Par défaut, Exchange autorise dix demandes simultanées vers la même boîte aux lettres cible ; toutes les suivantes restent dans la file d’attente avec un motif tel que `StalledDueToTarget_MailboxCapacityExceeded` ou `StalledDueToTarget_MdbReplication`, puis avancent. Il s’agit de la gestion de charge, non d’un problème de capacité, même si le nom le laisse entendre. Et `Get-MailboxImportRequest` affiche l’état avec retard ; la statistique est plus actuelle.

Le débit était d’environ six fichiers PST par minute pour une taille moyenne de quelques mégaoctets. MRS est donc nettement plus rapide qu’Enterprise Vault. Tout ce qui est importé reste dans la boîte aux lettres de journalisation jusqu’à ce que le Task l’archive et le supprime. `Resume-NICharge` lie donc la libération de nouvelles demandes à la quantité qui n’a pas encore été déplacée, afin que la boîte aux lettres ne grossisse pas jusqu’à la taille de l’ensemble de l’exportation.

## Script 2 : déplacement et cadence

Le second script s’exécute sous le Vault Service Account sur le serveur EV dans Windows PowerShell 5.1. Il déplace à plat les messages de tous les sous-dossiers de la boîte de réception vers celle-ci, exclut les brouillons ainsi que les dossiers *Versions* et *Conflits*, et n’alimente que lorsque la boîte de réception reste sous un seuil.

```powershell
Add-Type -Path "C:\Program Files (x86)\Enterprise Vault\Microsoft.Exchange.WebServices.dll"
$NIProtokoll = "E:\pstimport\protokoll\verschieben-protokoll.csv"
$ews = New-Object Microsoft.Exchange.WebServices.Data.ExchangeService(
  [Microsoft.Exchange.WebServices.Data.ExchangeVersion]::Exchange2013_SP1)
$ews.UseDefaultCredentials = $true
$ews.Url = New-Object Uri("https://mailserver01.example.com/EWS/Exchange.asmx")
$mb = New-Object Microsoft.Exchange.WebServices.Data.Mailbox("journal@example.com")

function Schreibe-NIMove($Schritt, $Wert, $Meldung) {
  [pscustomobject]@{
    Zeit = Get-Date -Format "yyyy-MM-dd HH:mm:ss"; Schritt = $Schritt; Wert = $Wert; Meldung = $Meldung
  } | Export-Csv -Path $NIProtokoll -Append -NoTypeInformation -Encoding UTF8
}

function Get-NIInbox {
  [Microsoft.Exchange.WebServices.Data.Folder]::Bind($ews,
    (New-Object Microsoft.Exchange.WebServices.Data.FolderId(
      [Microsoft.Exchange.WebServices.Data.WellKnownFolderName]::Inbox, $mb)))
}

function Get-NIRueckstand {
  $inbox = Get-NIInbox
  $f = New-Object Microsoft.Exchange.WebServices.Data.SearchFilter+IsLessThan(
    [Microsoft.Exchange.WebServices.Data.ItemSchema]::DateTimeReceived, (Get-Date).AddHours(-2))
  $r = $inbox.FindItems($f, (New-Object Microsoft.Exchange.WebServices.Data.ItemView(1)))
  [pscustomobject]@{ Zeit = Get-Date -Format "HH:mm:ss"; Posteingang = $inbox.TotalCount; Alt = $r.TotalCount }
}

function Get-NIUnterordner {
  $inbox = Get-NIInbox
  $liste = New-Object System.Collections.ArrayList
  $offset = 0
  do {
    $fv = New-Object Microsoft.Exchange.WebServices.Data.FolderView(500, $offset)
    $fv.Traversal = [Microsoft.Exchange.WebServices.Data.FolderTraversal]::Deep
    $res = $inbox.FindFolders($fv)
    foreach ($f in $res.Folders) { $null = $liste.Add($f) }
    $offset += 500
  } while ($res.MoreAvailable)
  $ausgeschlossen = @("Versions", "Konflikte", "Conflicts", "Conflitti", "Conflits")
  $liste | Where-Object { $_.TotalCount -gt 0 -and $_.DisplayName -notin $ausgeschlossen }
}

function Move-NIPortion([int]$Max = 2000) {
  $inbox = Get-NIInbox
  $keinEntwurf = New-Object Microsoft.Exchange.WebServices.Data.SearchFilter+IsEqualTo(
    [Microsoft.Exchange.WebServices.Data.ItemSchema]::IsDraft, $false)
  $n = 0
  foreach ($ordner in Get-NIUnterordner) {
    $runden = 0
    do {
      $seite = $ordner.FindItems($keinEntwurf, (New-Object Microsoft.Exchange.WebServices.Data.ItemView(100)))
      if ($seite.Items.Count -gt 0) {
        $ids = New-Object 'System.Collections.Generic.List[Microsoft.Exchange.WebServices.Data.ItemId]'
        foreach ($i in $seite.Items) { $ids.Add($i.Id) }
        $antwort = $ews.MoveItems($ids, $inbox.Id)
        $fehler = @($antwort | Where-Object { $_.Result -ne "Success" }).Count
        if ($fehler -gt 0) { Schreibe-NIMove "Fehler" $fehler $ordner.DisplayName; break }
        $n += $seite.Items.Count
      }
      $runden++
    } while ($seite.Items.Count -gt 0 -and $n -lt $Max -and $runden -lt 200)
    if ($n -ge $Max) { break }
  }
  Schreibe-NIMove "Verschoben" $n ""
  return $n
}

function Start-NIDauerlauf([int]$Portion = 2000, [int]$MaxPosteingang = 3000) {
  $stop = "E:\pstimport\protokoll\STOP.txt"
  while (-not (Test-Path $stop)) {
    $s = Get-NIRueckstand
    if ($s.Posteingang -gt $MaxPosteingang) {
      "$($s.Zeit)  Posteingang $($s.Posteingang)  alt $($s.Alt)  warte auf EV"
      Schreibe-NIMove "Rueckstand" $s.Alt ("Posteingang " + $s.Posteingang)
      Start-Sleep -Seconds 300
      continue
    }
    $n = Move-NIPortion -Max $Portion
    "$(Get-Date -Format HH:mm:ss)  verschoben $n  (Posteingang vorher $($s.Posteingang))"
    if ($n -eq 0) { Start-Sleep -Seconds 600 }
  }
  "STOP-Datei gefunden, Dauerlauf beendet"
}

function Get-NIMoveStand {
  $p = Import-Csv $NIProtokoll
  "verschoben gesamt: " + (($p | Where-Object { $_.Schritt -eq "Verschoben" } | Measure-Object Wert -Sum).Sum)
  "Fehler: " + @($p | Where-Object { $_.Schritt -eq "Fehler" }).Count
  Get-NIRueckstand
}
```

<details class="options-details">
<summary>Explication des fonctions</summary>

| Fonction | Effet |
|---|---|
| `Get-NIRueckstand` | Nombre d’éléments dans la boîte de réception et nombre d’éléments datant de plus de deux heures ; ces derniers sont les éléments importés ultérieurement, car le flux de journalisation en cours ne contient que des heures de réception récentes |
| `Get-NIUnterordner` | Tous les sous-dossiers non vides de la boîte de réception, à l’exception de *Versions* et *Conflits* |
| `Move-NIPortion -Max` | Déplace jusqu’à N messages sans indicateur de brouillon vers la boîte de réception, par pages de 100 via `MoveItems` avec une liste `ItemId` typée ; un tableau PowerShell ne correspond pas à la signature |
| `Start-NIDauerlauf -Portion -MaxPosteingang` | Boucle infinie : mesure la boîte de réception toutes les cinq minutes et n’alimente que si elle est sous le seuil ; se termine dès que le fichier `STOP.txt` existe |
| `Get-NIMoveStand` | Somme des éléments déplacés, nombre d’erreurs, retard actuel |

</details>

Le seuil `MaxPosteingang` est le chiffre le plus important de la procédure. Il doit être supérieur au volume de travail normal du Task — dans le cas décrit, environ 1'500 rapports à 100 messages par minute et quelque 20 minutes de retard — et suffisamment bas pour que le Task ne laisse jamais le flux en cours en attente. La section suivante explique pourquoi cela est nécessaire.

## Mesures et régulation

Avec les paramètres standard du Journaling Task — 5 connexions simultanées au serveur Exchange, 1'000 éléments par passage — Enterprise Vault a archivé les messages importés ultérieurement durant la première nuit à raison d’environ 5'600 par heure. Le matin, les statistiques des dossiers ont montré le coût : la boîte de réception était à 16'500 au lieu de 1'500. Le flux de journalisation en cours, environ 6'000 rapports par heure, accusait plus de deux heures de retard. Le Task avait traité les anciens messages et fait passer les rapports actuels au second plan.

La capacité du Task est la somme des deux flux. L’import ultérieur ne peut recevoir que la part restant après l’activité quotidienne. C’est pourquoi l’exécution continue mesure la boîte de réception elle-même, non le nombre d’anciens éléments, et n’alimente que lorsque le flux est à jour.

Trois mesures indiquent où se trouve la limite du Task. Les files d’attente entre le Task et le Storage Service sont des files d’attente MSMQ ; si celles du Storage Service sont vides alors que celles du Task sont remplies, le Task est limité par la lecture depuis Exchange, non par le stockage :

```powershell
Get-Counter -Counter '\MSMQ Queue(*)\Messages in Queue' -SampleInterval 5 -MaxSamples 6 |
  ForEach-Object {
    $t = $_.Timestamp.ToString("HH:mm:ss")
    $_.CounterSamples |
      Where-Object { $_.InstanceName -like "*enterprise vault*" -and $_.CookedValue -gt 0 } |
      ForEach-Object { "$t  $($_.InstanceName)  $($_.CookedValue)" }
  }
```

Côté Exchange, la latence RPC compte par type de client, non par nombre de requêtes ; des valeurs inférieures à 10 ms indiquent que le serveur dispose de réserves :

```powershell
$zaehler = @('\MSExchangeIS Client Type(*)\RPC Average Latency', '\Processor(_Total)\% Processor Time')
Get-Counter -ComputerName MAILSERVER01 -Counter $zaehler -SampleInterval 5 -MaxSamples 6 |
  ForEach-Object {
    $t = $_.Timestamp.ToString("HH:mm:ss")
    $_.CounterSamples | Where-Object { $_.CookedValue -gt 0 } |
      ForEach-Object { "$t  $($_.InstanceName)  $($_.Path.Split('\')[-1])  $([math]::Round($_.CookedValue,1))" }
  }
```

Et la stratégie de limitation du Vault Service Account : Veritas exige une stratégie avec `RcaMaxConcurrency: Unlimited`; avec la stratégie standard, Exchange limite les connexions MAPI simultanées par compte et les connexions Task supplémentaires n’aboutissent pas :

```powershell
Get-ThrottlingPolicyAssociation -Identity "svc-ev" | Format-List ThrottlingPolicyId
Get-ThrottlingPolicy | Format-Table Name, IsServiceAccount, RcaMaxConcurrency, EwsMaxConcurrency -AutoSize
```

Si le Task est le facteur limitant, le paramètre approprié se trouve dans les propriétés du Task (*Enterprise Vault Servers*, serveur, *Tasks*, Journaling Task, onglet *Settings*) :

| Paramètre | Valeur par défaut | Effet |
|---|---|---|
| *Number of concurrent connections to Exchange Server* | 5 | Threads lisant la boîte aux lettres en parallèle ; effet linéaire tant que la latence Exchange et les files d’attente du stockage restent normales |
| *Maximum number of items per target per pass* | 1000 | Éléments par passage ; avec un grand retard, moins de reprises |

Les modifications prennent effet après le redémarrage du Task (clic droit, *Stop*, puis *Start*). Il est judicieux de doubler la valeur, puis de mesurer à l’aide des horodatages du journal de déplacement avant de passer au niveau suivant. Il n’est pas possible d’utiliser un second Journaling Task sur la même boîte aux lettres ; une boîte aux lettres de journalisation est affectée à un seul Task. Une véritable parallélisation nécessite une seconde boîte aux lettres de journalisation et son propre Task sur un second serveur EV, dans l’archive duquel la seconde moitié des fichiers est importée. Cela modifie l’architecture d’archivage et doit être coordonné avec l’exploitation.

## Ce qui ressort en périphérie

Un import ultérieur de cette ampleur met au jour des éléments que personne n’avait examinés auparavant. Deux exemples du cas décrit, car ils sont susceptibles de se reproduire.

La surveillance signalait sur le serveur Exchange un nombre de *RPC Requests/sec* supérieur au seuil, avec des valeurs prédéfinies de 60 et 70 requêtes par seconde dans le modèle de vérification. La charge de base du serveur dépassait déjà le seuil critique sans importation. Les requêtes par seconde constituent une mesure de charge ; la mesure de santé est la latence, qui était inférieure à une milliseconde. Le seuil doit être redéfini à partir de la charge de base dans l’historique de surveillance, par exemple au double et au triple du pic journalier typique, tandis que le seuil de latence reste inchangé.


Depuis des années, quatre éléments se trouvaient dans la boîte de réception de la boîte aux lettres de journalisation ; le Task les tentait à nouveau à chaque passage et les rejetait avec l’événement 3071. Ils n’étaient apparus dans aucune évaluation parce que personne ne lisait régulièrement le journal des événements `Veritas Enterprise Vault`. Il en allait de même pour la boîte aux lettres de capture de `JournalingReportNdrTo` : 117'000 rapports de journalisation non distribuables depuis 2020, indiquant que l’archivage avait déjà perdu des rapports à plusieurs reprises avant la lacune actuelle.

## Preuve et clôture

À la fin de l’import ultérieur, il y a trois fichiers et une recherche. `items.csv` de l’export Purview prouve ce qui a été exporté. `import-protokoll.csv` contient, pour chaque fichier, ce qu’Exchange a accepté. `verschieben-protokoll.csv` contient ce qui a été remis au Task et à quel moment. La recherche dans l’archive de journalisation sur la période de la lacune, répartie par jour, fournit le nombre présent dans l’archive. Les écarts entre ces nombres s’expliquent : brouillons, éléments sans expéditeur, dossiers *Versions* et *Conflits*, boîtes aux lettres hors conservation. Cette explication constitue précisément le message destiné à la conformité, avec les deux limites qu’aucun import ultérieur ne supprime : les destinataires d’enveloppe manquants et, sans déduplication, les copies au lieu des messages.

Procédez ensuite au nettoyage. Supprimez de la boîte aux lettres de journalisation les sous-dossiers vides et les brouillons restants, supprimez le partage contenant les fichiers PST, et effacez les fichiers PST une fois le rapprochement terminé. Retirez l’attribution du rôle *Mailbox Import Export* ; les paramètres du Task peuvent rester si l’exploitation connaît les mesures. Enfin, l’incident lui-même doit donner lieu à une surveillance de la remise de journalisation, car la lacune a été découverte grâce à un signalement fortuit, non par la surveillance.

## Historique des modifications

- **25 octobre 2026 :** remplacement de la source communautaire concernant l’importation PST dans des archives de journalisation par un article de support Enterprise Vault confirmant l’absence de prise en charge et citant Shared Archive ou la copie dans la boîte aux lettres de journalisation comme alternatives.

## Sources

1.  [Microsoft Learn: Export search results in eDiscovery](https://learn.microsoft.com/en-us/purview/edisc-search-export): liste complète des options d’exportation de la nouvelle eDiscovery, tailles de paquets, comportement de *Organize data* et *Include folder and path*, expiration des paquets après 14 jours.

2.  [Microsoft Learn: Deduplication in eDiscovery search results](https://learn.microsoft.com/en-us/purview/ediscovery-de-duplication-in-search-results): propriétés de comparaison de la déduplication classique et indication de l’arrêt des interfaces classiques le 31 août 2025.

3.  [Microsoft Learn: Export items from a review set in eDiscovery](https://learn.microsoft.com/en-us/purview/edisc-review-set-export): procédure d’exportation via un Review Set, qui est la seule à exclure les doublons dans la nouvelle eDiscovery.

4.  [Microsoft Q&A: eDiscovery cases, deduplication on export](https://learn.microsoft.com/en-us/answers/questions/2201291/ediscovery-cases-deduplication-on-export): confirmation que l’export direct ne propose plus de déduplication, avec des retours d’expérience sur la méthode Review Set.

5.  [Microsoft Learn: New-MailboxImportRequest](https://learn.microsoft.com/en-us/powershell/module/exchange/new-mailboximportrequest): paramètres de la demande d’importation, conditions préalables concernant le partage et le rôle *Mailbox Import Export*.

6.  [Microsoft Learn: Mailboxes are stalled during a migration](https://learn.microsoft.com/en-us/troubleshoot/exchange/migration/mailboxes-stalled-during-migration): gestion de charge avec dix demandes simultanées par cible et états `StalledDueToTarget` comme comportement attendu.

7.  [Veritas: Enterprise Vault PST Migration, wizard-assisted migration](https://www.veritas.com/support/en_US/doc/95955885-161896939-0/v11744603-161896939): procédure du PST Migrator, exigences d’accès du Storage Service et types d’archives cibles autorisés.

8.  [Enterprise Vault Support: Unable to migrate a Personal Store (PST) into a Journal Archive](https://supportenterprisevault.cloud.com/external/article/100029869/unable-to-migrate-a-personal-store-pst-i.html): l’importation PST dans une archive de journalisation n’est pas prise en charge ; l’article cite comme alternatives une Shared Archive ou la copie des messages dans la boîte de réception de la boîte aux lettres de journalisation.
