---
title: "Combler une lacune de journaling : export depuis Microsoft Purview et import dans Enterprise Vault"
navTitle: "Lacune de journaling"
description: "Si les rapports de journalisation d’Exchange Online cessent d’être produits pendant quelques jours, il n’est pas possible de les générer a posteriori. Leur contenu reste toutefois dans les boîtes aux lettres. Découvrez comment l’exporter au format PST via la nouvelle eDiscovery de Purview, pourquoi le PST Migrator d’Enterprise Vault n’est généralement pas adapté, et comment effectuer la réimportation via la boîte aux lettres de journalisation sans évincer le flux de journalisation en cours, avec deux scripts génériques d’importation, de déplacement et de journalisation."
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
aiPrompt: |
  Du bist mein Exchange- und Archivierungsassistent. Für einen Zeitraum sind keine Journalberichte im Enterprise-Vault-Journalarchiv angekommen. Hilf mir Schritt für Schritt: Zeitraum der Lücke aus den Postfachstatistiken bestimmen, Aufbewahrungsrichtlinien prüfen, die Suche in der neuen Purview-eDiscovery mit korrekt geklammerter KQL-Abfrage anlegen, den Export als PST je Postfach ohne Ordnerstruktur konfigurieren, die PST-Dateien per New-MailboxImportRequest in das Journalpostfach importieren, die importierten Unterordner per EWS in den Posteingang leeren und den Nachimport so drosseln, dass der laufende Journalstrom Vorrang behält. Weise mich auf die Grenzen hin: fehlende Umschlagempfänger und Mehrfachkopien ohne Entdopplung.
translationOf: journaling-luecke-purview-export-enterprise-vault-import
url: https://rafaelpfister.ch/fr/blog/combler-une-lacune-de-journaling-export-depuis-microsoft-purview-et-import-dans-enterprise-vault
translationSourceHash: 532fe3454abb44c445145c6fd3e7424685e8bc5a28dfa18d25cff8a7f5516094
translationModel: gpt-5.6-terra
translatedAt: 2026-09-24T08:38:46.529Z
translationReview: automatic
---

# Combler une lacune de journaling : export depuis Microsoft Purview et import dans Enterprise Vault

La journalisation est un processus de transport. Exchange génère le rapport de journalisation au moment où le message passe par le transport et le remet à la boîte aux lettres de journalisation. Si cette remise échoue, par exemple parce qu’un connecteur ou un objet destinataire est mal configuré, aucune fonction ne permet de récupérer le rapport ultérieurement. Ce qui est passé par le transport durant cette période manque dans l’archive.

Le contenu des messages reste toutefois dans les boîtes aux lettres des personnes concernées et, avec une stratégie de rétention sans expiration, même lorsque les utilisateurs ont supprimé les messages. Il n’existe donc qu’une seule manière praticable de régulariser la situation : exporter la période depuis toutes les boîtes aux lettres et importer les copies dans l’archive de journalisation. Le processus suivant utilise la nouvelle eDiscovery de Microsoft Purview pour l’exportation et Veritas Enterprise Vault 15 pour l’importation ; les mesures proviennent d’une réimportation de plusieurs centaines de milliers de messages, au cours de laquelle les deux scripts ont été créés.

## Ce qu’un export peut remplacer, et ce qu’il ne peut pas remplacer

Un rapport de journalisation se compose de l’enveloppe et du contenu. L’enveloppe contient les destinataires effectivement livrés par le transport : y compris les destinataires en copie cachée et les membres résolus des listes de distribution. Une copie de boîte aux lettres ne contient pas ces informations. Il n’est plus possible de déterminer, après la réimportation, qui a reçu un message en copie cachée. En revanche, le contenu lui-même peut être entièrement reconstruit, à condition que deux conditions soient remplies.

Premièrement, une rétention doit s’appliquer aux boîtes aux lettres afin que les messages supprimés se trouvent encore dans les éléments récupérables. Vérifiez cela dans PowerShell Sécurité et conformité :

```powershell
Connect-IPPSSession
Get-RetentionCompliancePolicy -DistributionDetail |
  Format-List Name, Enabled, Mode, ExchangeLocation, ExchangeLocationException
Get-RetentionComplianceRule |
  Format-List Name, Policy, RetentionComplianceAction, RetentionDuration
```

Une stratégie avec `RetentionComplianceAction: Keep` et `RetentionDuration: Unlimited` sur `ExchangeLocation: All` sécurise entièrement le contenu. La liste d’exceptions dans `ExchangeLocationException` désigne les boîtes aux lettres auxquelles cela ne s’applique pas. Pour elles, seul ce qui est encore présent peut être reconstruit.

Deuxièmement, un export compte des copies de boîtes aux lettres, pas des messages. Un message envoyé à quinze destinataires internes apparaît quinze fois dans le résultat. Dans le cas décrit, pour une seule journée, 98'131 copies correspondaient à environ 6'800 messages uniques, déterminés à partir du suivi des messages. La manière de traiter ces copies relève de la conformité, et non de l’informatique (voir la section sur la déduplication).

## Déterminer la période

Le début de la lacune figure dans la boîte aux lettres de journalisation elle-même, sur le serveur Exchange qui l’héberge. L’heure du dernier rapport remis marque le début :

```powershell
Get-MailboxFolderStatistics "journal@example.com" -IncludeOldestAndNewestItems |
  Where-Object { $_.ItemsInFolder -gt 0 } |
  Format-Table Name, ItemsInFolder, OldestItemReceivedDate, NewestItemReceivedDate -AutoSize
```

Il en va de même pour la boîte aux lettres figurant dans `JournalingReportNdrTo` de la configuration du transport. Elle collecte les rapports de journalisation non distribuables, et son entrée la plus récente confirme le moment. La fin de la lacune correspond au moment où la règle de journalisation a été redirigée vers la destination réparée. La période d’exportation se situe entre ces deux moments, en UTC.

Le suivi des messages fournit le chiffre attendu pour le rapprochement : une ligne par destinataire et par message, avec `MessageId`. Un rapport historique couvre toute la période :

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

Les valeurs uniques de `MessageId` de ce rapport constituent le nombre auquel la réimportation est comparée à la fin.

## Exporter depuis la nouvelle eDiscovery

Microsoft a désactivé les interfaces classiques de Content Search et eDiscovery (Standard) le 31 août 2025. Ce qui figure encore dans de nombreux guides, notamment l’option de déduplication à l’exportation, n’existe plus dans la nouvelle eDiscovery. Procédure dans le portail Purview :

1. Sous *eDiscovery*, créez ou ouvrez un dossier. Pour l’exportation, le compte exécutant doit disposer du rôle *eDiscovery Manager* et d’une licence Microsoft 365 E3 ou E5.
2. Créez une recherche. Source de données : toutes les boîtes aux lettres, ou une seule boîte aux lettres pour un essai.
3. Utilisez KQL comme requête. Les parenthèses sont essentielles, car `AND` a une priorité supérieure à `OR` :

```text
kind:email AND ((received>=2026-09-10 AND received<2026-09-12)
             OR (sent>=2026-09-10 AND sent<2026-09-12))
```

Sans `kind:email` , la recherche compte également les entrées de calendrier, les contacts et les tâches. Dans le cas décrit, cela représentait la différence entre 473'725 et 98'131 résultats pour une journée. Sans les parenthèses extérieures, la seconde condition de date ne s’applique qu’à `sent`, et le résultat contient des messages de l’ensemble du contenu de la boîte aux lettres.

4. Exécutez *Generate statistics*. Pour les recherches à l’échelle du tenant, cela accélère considérablement l’exportation, car seules les boîtes aux lettres ayant des résultats sont traitées. Avant l’exportation, relancez les emplacements terminés en erreur via *Retry failed locations* ; sinon ils manqueront dans le résultat, ce qui ne devient visible qu’après le téléchargement.
5. Effectuez l’*Export* avec les paramètres suivants.

| Option | Effet |
|---|---|
| *Export type: Export items with items report* | Messages plus `items.csv`; le rapport seul ne contient aucun contenu |
| *Export format: Create PSTs for messages* | Un fichier PST par boîte aux lettres au lieu de fichiers `.msg` individuels |
| *Maximum PST package size* | À partir de cette taille, un PST est scindé en parties (`.001.pst`, `.002.pst`) |
| *Maximum .zip package size* | Taille des paquets de téléchargement ; doit être au moins égale à la taille du PST |
| *Organize data from different locations into separate folders or PSTs* | **Activer** : un fichier par boîte aux lettres, nommé `<smtp-adresse>.001.pst` |
| *Include folder and path of the source* | **Désactiver** : tous les messages sont placés dans le dossier `Items` du PST, au lieu de l’arborescence de la boîte aux lettres |
| *Give each item a friendly name* | Sans effet lors de l’exportation PST |

Le paramètre *Include folder and path of the source* détermine l’importation ultérieure. Avec l’arborescence des dossiers, l’importation crée dans la boîte aux lettres de journalisation les dossiers de chaque boîte aux lettres, dans la langue de l’utilisateur concerné : `Posteingang`, `Posta in arrivo`, `Boîte de réception`. Sans arborescence, tout se trouve dans un dossier `Items`, et la réimportation dispose d’un emplacement unique à partir duquel poursuivre.

Le téléchargement fournit des paquets Zip. Microsoft recommande 7-Zip ou un outil comparable plutôt que l’Explorateur Windows. Les paquets expirent 14 jours après leur création. Le fichier `items.csv` du rapport de processus de l’exportation (sous *Process manager*, Export, Rapports) documente ce que l’exportation contient et ce qui a été ignoré ; il fait partie du dossier.

### Déduplication

La Content Search classique comportait la case à cocher *Enable de-duplication* : les messages ayant le même `InternetMessageId`, `ConversationTopic` et `BodyTagInfo` n’étaient exportés qu’une seule fois, les autres emplacements figurant dans `Results.csv`. La nouvelle eDiscovery ne propose plus cette possibilité lors de l’exportation directe d’une recherche. La seule méthode documentée consiste à utiliser un *Review Set* : y charger les résultats de recherche, exécuter Analytics, appliquer le filtre généré automatiquement *For Review*, qui exclut les doublons, puis exporter depuis le Review Set. Cela nécessite eDiscovery Premium et donc une licence E5 pour l’utilisateur exécutant. Sur Microsoft Q&A, plusieurs utilisateurs signalent des doublons malgré cette procédure ; un essai sur une journée est recommandé avant l’exportation complète.

Sans déduplication, Enterprise Vault déduplique au niveau du stockage (les pièces jointes et les corps de messages identiques sont stockés une seule fois), mais pas au niveau des éléments. Chaque copie de boîte aux lettres devient une entrée d’archive distincte, un résultat de recherche distinct et un compteur distinct. Le décompte servant de preuve ne peut alors pas être déduit de l’archive, mais uniquement de `items.csv` comparé au suivi des messages. La décision d’accepter les copies ou de réexporter via un Review Set doit être prise avant l’importation, et non après.

## Les voies d’accès à Enterprise Vault

Enterprise Vault fournit le PST Migrator pour les fichiers PST, sous forme d’assistant dans la console d’administration (*Archives*, clic droit, *Import PST*) et de variante scriptable via le Policy Manager EVPM. Cette méthode présente trois limitations pour une réimportation dans une archive de journalisation.

Premièrement, la licence. Le Migrator requiert la fonctionnalité `EVPSTM` (*Exchange PST Migrator*). Dans un site ne pratiquant que la journalisation, elle n’est souvent pas concédée sous licence, et l’assistant s’arrête dès la première page avec *Required license not installed*. Le Policy Manager utilise le même Migrator et affiche le même message.

Deuxièmement, la cible. L’assistant ne propose pas les archives de journalisation comme cible, uniquement les archives de boîtes aux lettres et les archives de messagerie Internet. Une importation directe dans l’archive de journalisation n’est possible que via EVPM avec `ArchiveName`, et contourne alors les règles de traitement du Journaling Task. Il reste comme cible une Shared Archive dédiée dans le Vault Store de la journalisation, que la recherche couvre conjointement avec l’archive de journalisation.

Troisièmement, la traçabilité : le Migrator écrit dans le fichier PST (il marque les éléments migrés), ce qui modifie le jeu de données, et les rapports de migration par fichier doivent être rapprochés des chiffres d’exportation.

La seconde voie ne nécessite aucune licence supplémentaire : l’Exchange Journaling Task d’Enterprise Vault n’archive pas uniquement les rapports de journalisation. S’il reconnaît un élément comme rapport de journalisation, il le décompresse et reprend les données d’enveloppe. Il archive directement un élément ordinaire, comme il l’a toujours fait à l’époque de la journalisation Exchange simple sans enveloppe. En important les fichiers PST dans la boîte aux lettres de journalisation, les messages sont écrits par le task régulier dans l’archive de journalisation régulière, avec la même catégorie de rétention, la même indexation et sans modification de l’environnement d’archivage.

Vous pouvez le prouver avec un seul message de test adressé à la boîte aux lettres de journalisation : s’il ne se trouve peu après dans aucun dossier, pas même dans *Invalid Journal Report*, et que la recherche le trouve dans l’archive de journalisation, le task archive les messages ordinaires.

Cette voie présente deux caractéristiques qui déterminent le processus. Le task traite exclusivement le niveau supérieur de la boîte de réception, pas les sous-dossiers. Cela se voit dans la boîte aux lettres de journalisation : le dossier de recherche *Initial Trawl With Pending* sous *Enterprise Vault Search Folders* compte exactement les éléments de la boîte de réception, sans les sous-dossiers. Et l’importation via Exchange crée toujours des dossiers ; la manière dont elle le fait est expliquée dans la section suivante.

## Importer dans la boîte aux lettres de journalisation

`New-MailboxImportRequest` importe un fichier PST côté serveur via le Mailbox Replication Service. Le fichier doit se trouver sur un partage auquel *Exchange Trusted Subsystem* a un contrôle total, et le compte exécutant doit disposer du rôle *Mailbox Import Export*, qui n’est attribué à personne par défaut :

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
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-Mailbox` | Boîte aux lettres cible, ici la boîte aux lettres de journalisation |
| `-FilePath` | Chemin UNC du fichier PST ; les chemins locaux sont refusés |
| `-TargetRootFolder` | Dossier sous lequel le contenu du PST est déposé ; sans indication, les dossiers PST sont mappés aux dossiers de boîte aux lettres portant le même nom |
| `-SourceRootFolder` | Dossier du PST à partir duquel importer ; en pratique, le dossier lui-même est tout de même créé (voir le texte) |
| `-Name` | Nom unique de la demande ; dérivé du nom de fichier pour des milliers de fichiers |
| `-BatchName` | Regroupement pour les requêtes et statistiques |

</details>

Trois observations issues de l’essai déterminent la suite du processus. Sans `TargetRootFolder` et avec un PST structuré (arborescence des dossiers exportée), MRS crée la racine du PST comme dossier distinct à côté de la boîte de réception, dans le cas d’une boîte aux lettres allemande `Oberste-Ebene-des-Informationsspeichers` avec les sous-dossiers `Posteingang`, `Gesendete-Elemente` et ainsi de suite, ainsi que `Recoverable-Items` avec `Deletions`. Avec `TargetRootFolder "Inbox"` et un PST plat, tous les messages arrivent dans `/Inbox/Items`. Et `SourceRootFolder "Items"` associé à `TargetRootFolder "Inbox"` a également produit `/Inbox/Items` lors du test ; le niveau n’a pas été supprimé.

Dans les trois cas, les messages sont hors de la visibilité du Journaling Task. La dernière étape, qui consiste à les déplacer à plat dans la boîte de réception, ne peut pas être réalisée avec les outils Exchange natifs ; elle passe par EWS. L’API EWS Managed se trouve sous la forme de `Microsoft.Exchange.WebServices.dll` dans le répertoire d’installation d’Enterprise Vault, et le Vault Service Account dispose d’un contrôle total sur la boîte aux lettres de journalisation, de sorte qu’aucune autorisation supplémentaire n’est nécessaire. Le script s’exécute sur le serveur EV avec Windows PowerShell 5.1, car la bibliothèque repose sur le .NET Framework.

Cette étape de déplacement est aussi le régulateur de l’ensemble de la procédure : elle détermine le nombre de messages que le task voit à la fois.

## L’essai avec une boîte aux lettres

Avant de traiter des milliers de fichiers, tout le processus peut être vérifié avec une seule boîte aux lettres. La boîte aux lettres de la personne ayant effectué l’exportation est un bon choix : elle dispose du mandat, ce sont ses propres données et le fichier est petit. Procédure :

1. Copiez le fichier PST dans un dossier dédié et enregistrez le hachage SHA-256 de l’original dans un fichier CSV.
2. Importez avec `TargetRootFolder "Inbox"`, puis établissez les statistiques avec `Get-MailboxImportRequestStatistics` : `ItemsTransferred` est le nombre attendu.
3. Consultez les statistiques des dossiers de la boîte aux lettres de journalisation : où se trouvent les éléments ?
4. Déplacez les éléments de `/Inbox/Items` vers la boîte de réception via EWS.
5. Comptez les anciens éléments de la boîte de réception toutes les quelques minutes : les éléments dont `DateTimeReceived` est antérieur au début du flux en cours sont les éléments importés. Si leur nombre diminue, le task archive.
6. Effectuez le rapprochement : recherchez dans l’archive de journalisation (dans Discovery Accelerator ou la recherche EV) avec la période et le propriétaire de la boîte aux lettres, puis comparez le nombre de résultats à `ItemsTransferred`.

Dans le cas décrit, 128 éléments ont été importés et 126 archivés. Les deux éléments restants étaient un brouillon et un élément sans expéditeur ; le task laisse les éléments sans expéditeur en place. Les brouillons ne sont jamais passés par le transport et n’ont pas leur place dans un journal ; le script de déplacement les exclut donc via l’indicateur `IsDraft`, indépendamment de la langue. Les éléments des dossiers *Versions* et *Conflits* (versions antérieures sous rétention, résidus de synchronisation) ne sont pas non plus déplacés.

Le journal des événements `Veritas Enterprise Vault` indique après l’essai ce que le task a rejeté. Les événements 3071 (*could not be archived as it may be corrupt*) et 3288 (*no longer archive pending*) se répètent à chaque passage pour les mêmes éléments ; vous devriez déplacer ces éléments dans un dossier hors de la boîte de réception afin que le task ne les réessaie pas à chaque passage.

## Script 1 : importation par lots

Le premier script s’exécute dans l’Exchange Management Shell sur un serveur Exchange. Il crée des demandes d’importation par lots, attend, journalise pour chaque fichier le statut et `ItemsTransferred` dans un fichier CSV, puis supprime les demandes terminées. Les exécutions répétées ignorent les fichiers déjà présents dans le journal. Le préfixe `NI` signifie réimportation.

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
<summary>Fonctions expliquées</summary>

| Fonction | Effet |
|---|---|
| `Get-NIDateien` | Liste de tous les fichiers PST des dossiers sources avec un nom de demande dérivé ; les noms de fichiers identiques dans des dossiers différents (par exemple un export par jour) restent distinguables |
| `Start-NICharge -Anzahl` | Crée jusqu’à N demandes d’importation pour les fichiers non encore journalisés |
| `Wait-NICharge` | Attend qu’aucune demande ne soit plus en cours, écrit le statut et `ItemsTransferred` dans le journal, supprime les demandes terminées ; les demandes en échec restent en place pour analyse. Les statistiques et la suppression passent par le pipeline, car `-Identity` n’accepte pas l’objet Identity désérialisé dans un shell distant |
| `Resume-NICharge -Anzahl -MaxLager` | Ne libère les demandes suspendues que lorsque moins de `MaxLager` messages attendent dans les sous-dossiers de la boîte de réception |
| `Get-NIImportStand` | Récapitulatif par statut, somme des éléments importés, nombre total de fichiers |

</details>

Vous devez connaître deux comportements de MRS. Par défaut, Exchange autorise dix demandes simultanées vers la même boîte aux lettres cible ; toutes les suivantes restent dans la file d’attente avec une raison telle que `StalledDueToTarget_MailboxCapacityExceeded` ou `StalledDueToTarget_MdbReplication`, puis avancent. Il s’agit de la gestion de la charge de travail, non d’un problème de capacité, même si le nom le suggère. Par ailleurs, `Get-MailboxImportRequest` affiche le statut avec un retard ; les statistiques sont plus récentes.

Le débit était d’environ six fichiers PST par minute, pour une taille moyenne de quelques mégaoctets. MRS est donc nettement plus rapide qu’Enterprise Vault. Tout ce qui est importé reste dans la boîte aux lettres de journalisation jusqu’à ce que le task l’archive et le supprime. `Resume-NICharge` lie donc la libération de demandes supplémentaires à la quantité qui n’a pas encore été déplacée, afin que la boîte aux lettres ne grossisse pas jusqu’à la taille de l’exportation complète.

## Script 2 : déplacement et cadence

Le second script s’exécute sous le Vault Service Account sur le serveur EV dans Windows PowerShell 5.1. Il déplace à plat les messages de tous les sous-dossiers de la boîte de réception vers celle-ci, ignore les brouillons ainsi que les dossiers *Versions* et *Conflits*, et n’ajoute des éléments que lorsque la boîte de réception est sous un seuil.

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
<summary>Fonctions expliquées</summary>

| Fonction | Effet |
|---|---|
| `Get-NIRueckstand` | Nombre d’éléments dans la boîte de réception et nombre d’éléments âgés de plus de deux heures ; ces derniers sont les éléments réimportés, car le flux de journalisation en cours ne contient que des heures de réception récentes |
| `Get-NIUnterordner` | Tous les sous-dossiers non vides de la boîte de réception, à l’exclusion de *Versions* et *Conflits* |
| `Move-NIPortion -Max` | Déplace jusqu’à N messages sans indicateur de brouillon vers la boîte de réception, par pages de 100 via `MoveItems` avec une liste `ItemId` typée ; un tableau PowerShell ne correspond pas à la signature |
| `Start-NIDauerlauf -Portion -MaxPosteingang` | Boucle infinie : mesure la boîte de réception toutes les cinq minutes et n’ajoute des éléments que lorsqu’elle est sous le seuil ; s’arrête dès que le fichier `STOP.txt` existe |
| `Get-NIMoveStand` | Somme des éléments déplacés, nombre d’erreurs, retard actuel |

</details>

Le seuil `MaxPosteingang` est le nombre le plus important du processus. Il doit être supérieur au stock de travail normal du task (dans le cas décrit, environ 1'500 rapports à 100 messages par minute et environ 20 minutes de retard) et suffisamment bas pour que le task ne laisse jamais le flux en cours en attente. La section suivante explique pourquoi cela est nécessaire.

## Mesures et régulation

Avec les paramètres par défaut du Journaling Task (5 connexions simultanées au serveur Exchange, 1'000 éléments par passage), Enterprise Vault a archivé les messages réimportés la première nuit à raison d’environ 5'600 par heure. Le matin, les statistiques des dossiers ont montré le coût : la boîte de réception était à 16'500 au lieu de 1'500. Le flux de journalisation en cours, d’environ 6'000 rapports par heure, accusait plus de deux heures de retard. Le task avait traité les anciens messages et privé les rapports actuels de priorité.

La capacité du task est la somme des deux flux. La réimportation ne doit recevoir que la part restante après l’activité quotidienne. C’est pourquoi l’exécution continue mesure la boîte de réception elle-même, et non le nombre d’anciens éléments, et n’ajoute des éléments que lorsque le flux est à jour.

Trois mesures indiquent où se situe la limite du task. Les files d’attente entre le task et le Storage Service sont des files d’attente MSMQ ; si celles du Storage Service sont vides alors que celles du task sont remplies, le task est ralenti lors de la lecture depuis Exchange, et non par le stockage :

```powershell
Get-Counter -Counter '\MSMQ Queue(*)\Messages in Queue' -SampleInterval 5 -MaxSamples 6 |
  ForEach-Object {
    $t = $_.Timestamp.ToString("HH:mm:ss")
    $_.CounterSamples |
      Where-Object { $_.InstanceName -like "*enterprise vault*" -and $_.CookedValue -gt 0 } |
      ForEach-Object { "$t  $($_.InstanceName)  $($_.CookedValue)" }
  }
```

Côté Exchange, la latence RPC est comptée par type de client, et non le nombre de requêtes ; des valeurs inférieures à 10 ms signifient que le serveur dispose de réserves :

```powershell
$zaehler = @('\MSExchangeIS Client Type(*)\RPC Average Latency', '\Processor(_Total)\% Processor Time')
Get-Counter -ComputerName MAILSERVER01 -Counter $zaehler -SampleInterval 5 -MaxSamples 6 |
  ForEach-Object {
    $t = $_.Timestamp.ToString("HH:mm:ss")
    $_.CounterSamples | Where-Object { $_.CookedValue -gt 0 } |
      ForEach-Object { "$t  $($_.InstanceName)  $($_.Path.Split('\')[-1])  $([math]::Round($_.CookedValue,1))" }
  }
```

Et la stratégie de limitation du Vault Service Account : Veritas exige une stratégie avec `RcaMaxConcurrency: Unlimited`; avec la stratégie par défaut, Exchange limite les connexions MAPI simultanées par compte, et les connexions supplémentaires du task ne passent pas :

```powershell
Get-ThrottlingPolicyAssociation -Identity "svc-ev" | Format-List ThrottlingPolicyId
Get-ThrottlingPolicy | Format-Table Name, IsServiceAccount, RcaMaxConcurrency, EwsMaxConcurrency -AutoSize
```

Si le task ralentit, le paramètre approprié se trouve dans les propriétés du task (*Enterprise Vault Servers*, serveur, *Tasks*, Journaling Task, onglet *Settings*) :

| Paramètre | Valeur par défaut | Effet |
|---|---|---|
| *Number of concurrent connections to Exchange Server* | 5 | Threads qui lisent parallèlement dans la boîte aux lettres ; effet linéaire tant que la latence Exchange et les files d’attente de stockage restent normales |
| *Maximum number of items per target per pass* | 1000 | Éléments par passage ; redémarre moins fréquemment en cas de retard important |

Les modifications prennent effet après le redémarrage du task (clic droit, *Stop*, puis *Start*). Il est judicieux de doubler la valeur puis de mesurer à l’aide des horodatages du journal de déplacement, avant de passer au niveau suivant. Il n’est pas possible d’utiliser un second Journaling Task sur la même boîte aux lettres ; une boîte aux lettres de journalisation est attribuée à un seul task. Une véritable parallélisation nécessite une seconde boîte aux lettres de journalisation et son propre task sur un second serveur EV, dans l’archive duquel la seconde moitié des fichiers est importée. Il s’agit d’une modification de l’environnement d’archivage qui doit être coordonnée avec l’exploitation.

## Ce qui apparaît en périphérie

Une réimportation de cette ampleur révèle des choses que personne n’avait examinées auparavant. Voici deux exemples du cas décrit, car ils sont susceptibles de se reproduire.

La supervision signalait sur le serveur Exchange un *RPC Requests/sec* au-dessus du seuil, avec des valeurs prédéfinies de 60 et 70 requêtes par seconde dans le modèle de contrôle. La charge de base du serveur dépassait déjà le seuil critique sans importation. Les requêtes par seconde sont une mesure de charge ; la mesure de santé est la latence, qui était inférieure à une milliseconde. Le seuil doit être redéfini sur la base de la charge de base issue de l’historique de supervision, par exemple au double et au triple du pic habituel de la journée ; le seuil de latence reste inchangé.


Dans la boîte de réception de la boîte aux lettres de journalisation, quatre éléments étaient présents depuis des années, que le task réessayait à chaque passage et rejetait avec l’événement 3071. Ils n’étaient apparus dans aucune évaluation, car personne ne lisait régulièrement le journal des événements `Veritas Enterprise Vault`. Il en allait de même pour la boîte aux lettres de récupération de `JournalingReportNdrTo`: 117'000 rapports de journalisation non distribuables depuis 2020, signe que l’archivage avait déjà perdu à plusieurs reprises des rapports avant la lacune actuelle.

## Justificatifs et clôture

À la fin de la réimportation, il reste trois fichiers et une recherche. `items.csv` de l’exportation Purview atteste de ce qui a été exporté. `import-protokoll.csv` contient, pour chaque fichier, ce qu’Exchange a pris en charge. `verschieben-protokoll.csv` contient ce qui a été remis au task et à quel moment. La recherche dans l’archive de journalisation pour la période de la lacune, répartie par jours, fournit le nombre présent dans l’archive. Les écarts entre ces nombres s’expliquent par les brouillons, les éléments sans expéditeur, les dossiers *Versions* et *Conflits*, ainsi que les boîtes aux lettres hors rétention. Cette explication constitue précisément la déclaration destinée à la conformité, avec les deux limites qu’aucune réimportation ne supprime : les destinataires d’enveloppe manquants et, sans déduplication, les copies plutôt que les messages.

Ensuite vient le nettoyage. Les sous-dossiers vides et les brouillons restants sont supprimés de la boîte aux lettres de journalisation, le partage contenant les fichiers PST est supprimé, et les fichiers PST sont supprimés une fois le rapprochement terminé. L’attribution du rôle *Mailbox Import Export* est retirée ; les paramètres du task sont conservés lorsque l’exploitation connaît les mesures. Enfin, l’incident lui-même doit conduire à mettre en place une supervision de la remise des journaux, car la lacune a été découverte par une alerte fortuite, et non par la supervision.

## Sources

1.  [Microsoft Learn: Export search results in eDiscovery](https://learn.microsoft.com/en-us/purview/edisc-search-export): liste complète des options d’exportation de la nouvelle eDiscovery, tailles des paquets, comportement de *Organize data* et *Include folder and path*, expiration des paquets après 14 jours.

2.  [Microsoft Learn: Deduplication in eDiscovery search results](https://learn.microsoft.com/en-us/purview/ediscovery-de-duplication-in-search-results): propriétés de comparaison de la déduplication classique et indication de la désactivation des interfaces classiques le 31 août 2025.

3.  [Microsoft Learn: Export items from a review set in eDiscovery](https://learn.microsoft.com/en-us/purview/edisc-review-set-export): voie d’exportation via un Review Set, qui est la seule dans la nouvelle eDiscovery à exclure les doublons.

4.  [Microsoft Q&A: eDiscovery cases, deduplication on export](https://learn.microsoft.com/en-us/answers/questions/2201291/ediscovery-cases-deduplication-on-export): confirmation que l’exportation directe ne propose plus de déduplication, avec des retours d’expérience sur la voie Review Set.

5.  [Microsoft Learn: New-MailboxImportRequest](https://learn.microsoft.com/en-us/powershell/module/exchange/new-mailboximportrequest): paramètres de la demande d’importation, conditions préalables pour le partage et le rôle *Mailbox Import Export*.

6.  [Microsoft Learn: Mailboxes are stalled during a migration](https://learn.microsoft.com/en-us/troubleshoot/exchange/migration/mailboxes-stalled-during-migration): gestion de la charge de travail avec dix demandes simultanées par cible et états `StalledDueToTarget` comme comportement attendu.

7.  [Veritas: Enterprise Vault PST Migration, wizard-assisted migration](https://www.veritas.com/support/en_US/doc/95955885-161896939-0/v11744603-161896939): processus du PST Migrator, exigences d’accès du Storage Service et types d’archives cibles autorisés.

8.  [Veritas VOX: Import PST to a Journal Archive](https://vox.veritas.com/t5/Enterprise-Vault/Import-PST-to-a-Journal-Archive/td-p/272928): retours de la communauté expliquant pourquoi l’assistant ne propose pas les archives de journalisation et comment EVPM utilise `ArchiveName`.
