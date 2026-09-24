---
title: "Closing a journaling gap: Exporting from Microsoft Purview and importing into Enterprise Vault"
navTitle: "Journaling gap"
description: "If journal reports from Exchange Online fail for several days, they cannot be generated retroactively. However, the content still exists in the mailboxes. Learn how to export it as PST through the new Purview eDiscovery, why Enterprise Vault's PST Migrator is usually not an option, and how to reimport it through the journal mailbox without displacing the active journal stream, using two generic scripts for importing, moving, and logging."
date: "2026-09-22"
kategorie: "Archiving and Journaling"
timeToRead: "18 min read"
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
slug: "closing-a-journaling-gap-exporting-from-microsoft-purview-and-importing-into-enterprise-vault"
translationId: "article-5b24118ac7567d96"
aiPrompt: |
  Du bist mein Exchange- und Archivierungsassistent. Für einen Zeitraum sind keine Journalberichte im Enterprise-Vault-Journalarchiv angekommen. Hilf mir Schritt für Schritt: Zeitraum der Lücke aus den Postfachstatistiken bestimmen, Aufbewahrungsrichtlinien prüfen, die Suche in der neuen Purview-eDiscovery mit korrekt geklammerter KQL-Abfrage anlegen, den Export als PST je Postfach ohne Ordnerstruktur konfigurieren, die PST-Dateien per New-MailboxImportRequest in das Journalpostfach importieren, die importierten Unterordner per EWS in den Posteingang leeren und den Nachimport so drosseln, dass der laufende Journalstrom Vorrang behält. Weise mich auf die Grenzen hin: fehlende Umschlagempfänger und Mehrfachkopien ohne Entdopplung.
translationOf: journaling-luecke-purview-export-enterprise-vault-import
url: https://rafaelpfister.ch/en/blog/closing-a-journaling-gap-exporting-from-microsoft-purview-and-importing-into-enterprise-vault
translationSourceHash: 532fe3454abb44c445145c6fd3e7424685e8bc5a28dfa18d25cff8a7f5516094
translationModel: gpt-5.6-terra
translatedAt: 2026-09-24T08:37:39.979Z
translationReview: automatic
---

# Closing a journaling gap: Exporting from Microsoft Purview and importing into Enterprise Vault

Journaling is a transport operation. Exchange generates the journal report when the message passes through transport and delivers it to the journal mailbox. If that delivery fails, for example because a connector or recipient object is configured incorrectly, there is no function that retrieves the report later. Anything that passed through transport during that period is missing from the archive.

However, the message content remains in the mailboxes of those involved, and under a retention policy without expiry, it remains there even if users deleted the messages. This leaves only one viable way to catch up: export the period from all mailboxes and import the copies into the journal archive. The following process uses the new eDiscovery in Microsoft Purview on the export side and Veritas Enterprise Vault 15 on the import side; the measurements come from a reimport of several hundred thousand messages, during which the two scripts were developed.

## What an export can replace—and what it cannot

A journal report consists of the envelope and the content. The envelope contains the recipients that transport actually delivered to, including Bcc recipients and resolved distribution list members. A mailbox copy does not contain this information. After reimporting, it is no longer possible to determine who received a message as a Bcc recipient. The content itself, however, can be reconstructed completely if two conditions are met.

First, retention must apply to the mailboxes so that deleted messages remain in Recoverable Items. Check this in Security & Compliance PowerShell:

```powershell
Connect-IPPSSession
Get-RetentionCompliancePolicy -DistributionDetail |
  Format-List Name, Enabled, Mode, ExchangeLocation, ExchangeLocationException
Get-RetentionComplianceRule |
  Format-List Name, Policy, RetentionComplianceAction, RetentionDuration
```

A policy with `RetentionComplianceAction: Keep` and `RetentionDuration: Unlimited` through `ExchangeLocation: All` fully preserves the content. The exception list in `ExchangeLocationException` identifies the mailboxes to which this does not apply. For those, only what is still present can be reconstructed.

Second, an export counts mailbox copies, not messages. A message sent to fifteen internal recipients appears fifteen times in the result. In the case described, 98,131 copies for a single day corresponded to approximately 6,800 unique messages, determined from message tracking. Compliance—not IT—decides how to handle these copies (see the section on deduplication).

## Determining the period

The start of the gap is in the journal mailbox itself, on the Exchange server that hosts it. The time of the last delivered report is the start:

```powershell
Get-MailboxFolderStatistics "journal@example.com" -IncludeOldestAndNewestItems |
  Where-Object { $_.ItemsInFolder -gt 0 } |
  Format-Table Name, ItemsInFolder, OldestItemReceivedDate, NewestItemReceivedDate -AutoSize
```

The same applies to the mailbox configured in `JournalingReportNdrTo` of the transport configuration. It collects undeliverable journal reports, and its most recent entry confirms the time. The end of the gap is the time when the journaling rule was switched to the repaired destination. The export period, in UTC, lies between the two.

Message tracking provides the expected count for reconciliation: one row per recipient and message, with `MessageId`. A historical report covers the entire period:

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

The unique `MessageId` values from this report are the number against which the reimport is measured at the end.

## Exporting from the new eDiscovery

Microsoft retired the classic Content Search and eDiscovery (Standard) interfaces on August 31, 2025. What many guides still describe, particularly the option to deduplicate on export, no longer exists in the new eDiscovery. The process in the Purview portal:

1. Under *eDiscovery*, create or open a case. The account performing the export needs the *eDiscovery Manager* role and a Microsoft 365 E3 or E5 license.
2. Create a search. Data source: all mailboxes, or a single mailbox for a test run.
3. Use KQL as the query. The parentheses are essential because `AND` has higher precedence than `OR`:

```text
kind:email AND ((received>=2026-09-10 AND received<2026-09-12)
             OR (sent>=2026-09-10 AND sent<2026-09-12))
```

Without `kind:email` the search also counts calendar entries, contacts, and tasks. In the case described, this made the difference between 473,725 and 98,131 hits for one day. Without the outer parentheses, the second date condition applies only to `sent`, and the result includes messages from the entire mailbox inventory.

4. Run *Generate statistics*. For tenant-wide searches, this substantially speeds up export because only mailboxes with hits are processed. Before exporting, retry locations that finish with errors using *Retry failed locations*; otherwise, they are missing from the result, which only becomes apparent after the download.
5. Select *Export* with the following settings.

| Option | Effect |
|---|---|
| *Export type: Export items with items report* | Messages plus `items.csv`; the report alone contains no content |
| *Export format: Create PSTs for messages* | One PST per mailbox instead of individual `.msg` files |
| *Maximum PST package size* | At this size, a PST is split into parts (`.001.pst`, `.002.pst`) |
| *Maximum .zip package size* | Size of the download packages; must be at least the PST size |
| *Organize data from different locations into separate folders or PSTs* | **Enable**: one file per mailbox, named `<smtp-adresse>.001.pst` |
| *Include folder and path of the source* | **Disable**: all messages go into the `Items` folder of the PST instead of the mailbox folder structure |
| *Give each item a friendly name* | Has no effect for PST export |

The *Include folder and path of the source* setting determines the later import. With a folder structure, the import creates each mailbox's folders in the journal mailbox, in the respective user's language: `Posteingang`, `Posta in arrivo`, `Boîte de réception`. Without a folder structure, everything is in a single `Items` folder, and the reimport has one location from which to continue.

The download provides Zip packages. Microsoft recommends 7-Zip or a comparable tool instead of Windows Explorer. Packages expire 14 days after creation. The `items.csv` file from the export process report (under *Process manager*, Export, Reports) documents what the export contains and what was skipped; it belongs in the records.

### Deduplication

Classic Content Search offered the *Enable de-duplication* checkbox: messages with the same `InternetMessageId`, `ConversationTopic` and `BodyTagInfo` were exported only once, while the remaining hit locations appeared in `Results.csv`. The new eDiscovery does not offer this for direct export from a search. The only documented approach is a *Review Set*: load the search results into it, run Analytics, use the automatically generated *For Review* filter that excludes duplicates, and export from the Review Set. This requires eDiscovery Premium and therefore an E5 license for the performing user. Several users report duplicates despite this process in Microsoft Q&A; a one-day test run is advisable before the full export.

Without deduplication, the following applies: Enterprise Vault deduplicates at the storage level (identical attachments and message bodies are stored once), but not at the item level. Each mailbox copy becomes its own archive entry, its own search result, and its own count. The count for verification can then not be derived from the archive, but only from `items.csv` against message tracking. The decision to accept copies or re-export through a Review Set is made before import, not afterward.

## Paths into Enterprise Vault

Enterprise Vault includes the PST Migrator for PST files, as a wizard in the Administration Console (*Archives*, right-click, *Import PST*) and as a scriptable option through Policy Manager EVPM. For reimporting into a journal archive, this path has three limitations.

First, licensing. The Migrator requires the `EVPSTM` feature (*Exchange PST Migrator*). At a site that only performs journaling, it is often not licensed, and the wizard stops on the first page with *Required license not installed*. Policy Manager uses the same Migrator and reports the same issue.

Second, the destination. The wizard does not offer journal archives as a destination, only mailbox archives and Internet Mail Archives. Importing directly into the journal archive is possible only through EVPM with `ArchiveName`, and it bypasses the Journaling Task processing rules. The remaining destination is a dedicated Shared Archive in the journal's Vault Store, which search covers together with the journal archive.

Third, traceability: the Migrator writes to the PST file (it marks migrated items), so the inventory changes, and the migration reports for each file must be reconciled with the export counts.

The second path requires no additional license: the Enterprise Vault Exchange Journaling Task archives more than just journal reports. If it recognizes an item as a journal report, it unwraps it and takes over the envelope data. It archives an ordinary item directly, just as it always did in the days of simple Exchange journaling without an envelope. Anyone who imports the PST files into the journal mailbox has the messages written by the regular task into the regular journal archive, with the same retention category, the same indexing, and no changes to the archiving environment.

This can be proven with a single test message sent to the journal mailbox: if it shortly afterward is no longer in any folder, including *Invalid Journal Report*, and search finds it in the journal archive, the task archives ordinary messages.

This approach has two characteristics that determine the process. The task processes only the top level of the Inbox, not subfolders. This can be seen in the journal mailbox: the *Initial Trawl With Pending* search folder under *Enterprise Vault Search Folders* counts exactly the Inbox items; subfolders are not included. And importing through Exchange always creates folders; the next section explains how.

## Importing into the journal mailbox

`New-MailboxImportRequest` imports a PST file server-side through the Mailbox Replication Service. The file must be on a share to which *Exchange Trusted Subsystem* has Full Control, and the performing account needs the *Mailbox Import Export* role, which is not assigned to anyone by default:

```powershell
Get-ManagementRoleAssignment -Role "Mailbox Import Export" |
  Format-Table RoleAssigneeName, RoleAssigneeType -AutoSize
New-ManagementRoleAssignment -Role "Mailbox Import Export" -User "admin@example.com"
```

The import itself:

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
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-Mailbox` | Target mailbox, here the journal mailbox |
| `-FilePath` | UNC path of the PST file; local paths are rejected |
| `-TargetRootFolder` | Folder under which the PST contents are stored; if omitted, PST folders are mapped to mailbox folders with the same names |
| `-SourceRootFolder` | PST folder from which to import; in practice, the folder itself is still created (see text) |
| `-Name` | Unique request name; derived from the file name for thousands of files |
| `-BatchName` | Grouping for queries and statistics |

</details>

Three observations from the test run determine the subsequent process. Without `TargetRootFolder` and with a structured PST (folder structure exported), MRS creates the PST root as its own folder next to the Inbox, in the case of a German mailbox `Oberste-Ebene-des-Informationsspeichers` with the subfolders `Posteingang`, `Gesendete-Elemente` and so on, plus `Recoverable-Items` with `Deletions`. With `TargetRootFolder "Inbox"` and a flat PST, all messages end up in `/Inbox/Items`. And `SourceRootFolder "Items"` combined with `TargetRootFolder "Inbox"` also resulted in `/Inbox/Items` in the test; the level was not stripped.

In all three cases, the messages are outside the Journaling Task's view. The final step of moving them flat into the Inbox cannot be accomplished using built-in Exchange tools; it uses EWS. The EWS Managed API is available as `Microsoft.Exchange.WebServices.dll` in the Enterprise Vault installation directory, and the Vault Service Account has Full Access to the journal mailbox, so no additional permission is required. The script runs on the EV server in Windows PowerShell 5.1 because the library is based on .NET Framework.

This move step is also the throttle for the entire process: it determines how many messages the task sees at once.

## The test run with one mailbox

Before running thousands of files, the entire path can be tested with a single mailbox. The mailbox of the person who performed the export is a sensible choice: they have the authorization, the data is their own, and the file is small. The process:

1. Copy the PST file into a separate folder and store the SHA-256 hash of the original in a CSV file.
2. Import with `TargetRootFolder "Inbox"`, obtain statistics with `Get-MailboxImportRequestStatistics`: `ItemsTransferred` is the expected count.
3. Retrieve folder statistics for the journal mailbox: where are the items?
4. Move from `/Inbox/Items` into the Inbox through EWS.
5. Count old items in the Inbox every few minutes: items with `DateTimeReceived` before the start of the active stream are the imported items. If the number decreases, the task is archiving.
6. Reconcile: search the journal archive (in Discovery Accelerator or EV Search) by time period and mailbox owner, and compare the hit count with `ItemsTransferred`.

In the described case, 128 items were imported and 126 archived. The two remaining items were a draft and an item without a sender; the task leaves items without a sender in place. Drafts never passed through transport and do not belong in a journal, which is why the move script excludes them using the `IsDraft` flag, language-independently. Items from the *Versions* and *Conflicts* folders (earlier versions under retention, synchronization remnants) are likewise not moved.

The `Veritas Enterprise Vault` event log shows what the task rejected after the test run. Event 3071 (*could not be archived as it may be corrupt*) and 3288 (*no longer archive pending*) recur for the same items on every pass; move such items to a folder outside the Inbox so the task does not retry them on every pass.

## Script 1: Batch import

The first script runs in the Exchange Management Shell on an Exchange server. It creates import requests in batches, waits, logs the status and `ItemsTransferred` for each file to a CSV file, and removes completed requests. Repeated runs skip files already listed in the log. The `NI` prefix stands for reimport.

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
<summary>Functions explained</summary>

| Function | Effect |
|---|---|
| `Get-NIDateien` | Lists all PST files from the source folders with a derived request name; identical file names in different folders (such as one export per day) remain distinguishable |
| `Start-NICharge -Anzahl` | Creates up to N import requests for files not yet logged |
| `Wait-NICharge` | Waits until no request is running, writes status and `ItemsTransferred` to the log, removes completed requests; failed requests remain for analysis. Statistics and removal run through the pipeline because `-Identity` in a remote shell does not accept the deserialized Identity object |
| `Resume-NICharge -Anzahl -MaxLager` | Releases suspended requests only if fewer than `MaxLager` messages are waiting in the Inbox subfolders |
| `Get-NIImportStand` | Summary by status, total imported items, total number of files |

</details>

You should know two MRS behaviors. By default, Exchange allows ten simultaneous requests to the same target mailbox; all additional requests wait in the queue with a reason such as `StalledDueToTarget_MailboxCapacityExceeded` or `StalledDueToTarget_MdbReplication` and advance later. This is workload management, not a capacity issue, even if the name suggests otherwise. And `Get-MailboxImportRequest` displays status with a delay; statistics are more current.

Throughput was about six PST files per minute at an average size of a few megabytes. MRS is therefore substantially faster than Enterprise Vault. Everything that has been imported remains in the journal mailbox until the task archives and deletes it. `Resume-NICharge` therefore ties the release of additional requests to the amount not yet moved, so that the mailbox does not grow to the size of the entire export.

## Script 2: Moving and pacing

The second script runs as the Vault Service Account on the EV server in Windows PowerShell 5.1. It moves messages from all Inbox subfolders flat into the Inbox, excludes drafts and the *Versions* and *Conflicts* folders, and adds more only when the Inbox is below a threshold.

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
<summary>Functions explained</summary>

| Function | Effect |
|---|---|
| `Get-NIRueckstand` | Number of Inbox items and number of items older than two hours; the latter are the reimported items because the active journal stream carries only current receipt times |
| `Get-NIUnterordner` | All nonempty Inbox subfolders, excluding *Versions* and *Conflicts* |
| `Move-NIPortion -Max` | Moves up to N messages without the draft flag into the Inbox, 100 at a time through `MoveItems` using a typed `ItemId` list; a PowerShell array does not match the signature |
| `Start-NIDauerlauf -Portion -MaxPosteingang` | Infinite loop: measures the Inbox every five minutes and adds more only when it is below the threshold; exits as soon as the `STOP.txt` file exists |
| `Get-NIMoveStand` | Total moved items, error count, current backlog |

</details>

The `MaxPosteingang` threshold is the most important number in the process. It must exceed the task's normal work backlog (in the case described, about 1,500 reports at 100 messages per minute and roughly 20 minutes of delay) and be low enough that the task never leaves the active stream behind. The next section explains why this is necessary.

## Measurements and the throttle

With the Journaling Task default settings (5 simultaneous connections to the Exchange server, 1,000 items per pass), Enterprise Vault archived the reimported messages during the first night at approximately 5,600 per hour. In the morning, folder statistics showed the cost: the Inbox was at 16,500 instead of 1,500. The active journal stream, about 6,000 reports per hour, was more than two hours behind. The task had processed the old messages alongside the current reports instead of prioritizing the latter.

The task's capacity is the sum of both streams. The reimport may receive only the portion remaining after day-to-day operations. The continuous run therefore measures the Inbox itself, not the number of old items, and adds more only when the stream is current.

Three measurements show where the task's limit lies. The queues between the task and Storage Service are MSMQ queues; if the Storage Service queues are empty while those of the task are full, the task is limited while reading from Exchange, not by storage:

```powershell
Get-Counter -Counter '\MSMQ Queue(*)\Messages in Queue' -SampleInterval 5 -MaxSamples 6 |
  ForEach-Object {
    $t = $_.Timestamp.ToString("HH:mm:ss")
    $_.CounterSamples |
      Where-Object { $_.InstanceName -like "*enterprise vault*" -and $_.CookedValue -gt 0 } |
      ForEach-Object { "$t  $($_.InstanceName)  $($_.CookedValue)" }
  }
```

On the Exchange side, RPC latency is counted by client type, not the number of requests; values below 10 ms mean the server has reserve capacity:

```powershell
$zaehler = @('\MSExchangeIS Client Type(*)\RPC Average Latency', '\Processor(_Total)\% Processor Time')
Get-Counter -ComputerName MAILSERVER01 -Counter $zaehler -SampleInterval 5 -MaxSamples 6 |
  ForEach-Object {
    $t = $_.Timestamp.ToString("HH:mm:ss")
    $_.CounterSamples | Where-Object { $_.CookedValue -gt 0 } |
      ForEach-Object { "$t  $($_.InstanceName)  $($_.Path.Split('\')[-1])  $([math]::Round($_.CookedValue,1))" }
  }
```

And the throttling policy of the Vault Service Account: Veritas requires a policy with `RcaMaxConcurrency: Unlimited`; with the default policy, Exchange limits simultaneous MAPI connections per account, and additional task connections do not get through:

```powershell
Get-ThrottlingPolicyAssociation -Identity "svc-ev" | Format-List ThrottlingPolicyId
Get-ThrottlingPolicy | Format-Table Name, IsServiceAccount, RcaMaxConcurrency, EwsMaxConcurrency -AutoSize
```

If the task is the bottleneck, the appropriate settings are in the task properties (*Enterprise Vault Servers*, server, *Tasks*, Journaling Task, *Settings* tab):

| Setting | Default | Effect |
|---|---|---|
| *Number of concurrent connections to Exchange Server* | 5 | Threads that read from the mailbox in parallel; has a linear effect as long as Exchange latency and Storage queues remain unremarkable |
| *Maximum number of items per target per pass* | 1000 | Items per pass; restarts less frequently when there is a large backlog |

Changes take effect after restarting the task (right-click, *Stop*, then *Start*). Doubling followed by measuring using timestamps in the move log, then proceeding to the next level, is sensible. A second Journaling Task for the same mailbox is not possible; a journal mailbox is assigned to exactly one task. True parallelism requires a second journal mailbox and its own task on a second EV server, into whose archive the second half of the files is imported. This changes the archiving environment and should be coordinated with operations.

## What stands out at the margins

A reimport of this size reveals things nobody had looked at before. Two from the described case, because they are likely to recur.

Monitoring reported *RPC Requests/sec* above the threshold on the Exchange server, with default values of 60 and 70 requests per second from the check template. The server's baseline load was already above the critical threshold without the import. Requests per second are a load metric; latency is the health metric, and it was below one millisecond. The threshold should be reset based on the baseline load from monitoring history, perhaps to twice and three times the typical daily peak, while the latency threshold remains.


The journal mailbox Inbox had contained four items for years that the task retried on every pass and rejected with event 3071. They had not appeared in any reporting because nobody regularly read the `Veritas Enterprise Vault` event log. The same applied to the catch-all mailbox from `JournalingReportNdrTo`: 117,000 undeliverable journal reports since 2020, an indication that archiving had repeatedly lost reports before the current gap as well.

## Verification and completion

At the end of the reimport, there are three files and one search. `items.csv` from the Purview export documents what was exported. `import-protokoll.csv` contains, for each file, what Exchange accepted. `verschieben-protokoll.csv` contains what was handed to the task and when. Searching the journal archive for the gap period, split by day, provides the count in the archive. The differences between these numbers are explainable: drafts, items without a sender, *Versions* and *Conflicts* folders, and mailboxes outside retention. This explanation is the statement for Compliance, together with the two limitations no reimport can overcome: missing envelope recipients and, without deduplication, copies rather than messages.

Then clean up. Remove the empty subfolders and remaining drafts from the journal mailbox, remove the share containing the PST files, and delete the PST files once reconciliation is complete. Remove the *Mailbox Import Export* role assignment; leave the task settings in place if operations knows the measurements. And add monitoring for journal delivery to the incident itself, because the gap was discovered through an incidental alert rather than monitoring.

## Sources

1.  [Microsoft Learn: Export search results in eDiscovery](https://learn.microsoft.com/en-us/purview/edisc-search-export): complete list of export options in the new eDiscovery, package sizes, behavior of *Organize data* and *Include folder and path*, and 14-day package expiration.

2.  [Microsoft Learn: Deduplication in eDiscovery search results](https://learn.microsoft.com/en-us/purview/ediscovery-de-duplication-in-search-results): comparison properties of classic deduplication and notice of the retirement of the classic interfaces on August 31, 2025.

3.  [Microsoft Learn: Export items from a review set in eDiscovery](https://learn.microsoft.com/en-us/purview/edisc-review-set-export): the export path through a Review Set, which is the only way in the new eDiscovery to exclude duplicates.

4.  [Microsoft Q&A: eDiscovery cases, deduplication on export](https://learn.microsoft.com/en-us/answers/questions/2201291/ediscovery-cases-deduplication-on-export): confirmation that direct export no longer offers deduplication, with user experience reports on the Review Set path.

5.  [Microsoft Learn: New-MailboxImportRequest](https://learn.microsoft.com/en-us/powershell/module/exchange/new-mailboximportrequest): import request parameters and requirements for the share and the *Mailbox Import Export* role.

6.  [Microsoft Learn: Mailboxes are stalled during a migration](https://learn.microsoft.com/en-us/troubleshoot/exchange/migration/mailboxes-stalled-during-migration): workload management with ten simultaneous requests per destination and the `StalledDueToTarget` states as expected behavior.

7.  [Veritas: Enterprise Vault PST Migration, wizard-assisted migration](https://www.veritas.com/support/en_US/doc/95955885-161896939-0/v11744603-161896939): PST Migrator process, Storage Service access requirements, and permitted destination archive types.

8.  [Veritas VOX: Import PST to a Journal Archive](https://vox.veritas.com/t5/Enterprise-Vault/Import-PST-to-a-Journal-Archive/td-p/272928): community experience explaining why the wizard does not offer journal archives and how EVPM works with `ArchiveName`.
