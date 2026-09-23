---
title: "Journaling-Lücke schliessen: Export aus Microsoft Purview und Import in Enterprise Vault"
navTitle: "Journaling-Lücke"
description: "Fallen Journalberichte aus Exchange Online für einige Tage aus, lassen sie sich nicht nachträglich erzeugen. Der Inhalt liegt aber noch in den Postfächern. Wie Sie ihn über die neue Purview-eDiscovery als PST exportieren, warum der PST-Migrator von Enterprise Vault dafür meist nicht in Frage kommt und wie der Nachimport über das Journalpostfach läuft, ohne den laufenden Journalstrom zu verdrängen, mit zwei generischen Skripten für Import, Verschieben und Protokoll."
date: "2026-09-22"
kategorie: "Archivierung und Journaling"
timeToRead: "18 Min. Lesezeit"
themen:
  - "archivierung-journaling"
  - "microsoft-365-exchange"
  - "exchange-onprem-hybrid"
produkte:
  - "exchange-online"
  - "exchange-on-premises"
protokolle:
  - "backup-dr"
  - "powershell"
  - "troubleshooting"
slug: "journaling-luecke-purview-export-enterprise-vault-import"
translationId: "article-5b24118ac7567d96"
url: "https://rafaelpfister.ch/blog/journaling-luecke-purview-export-enterprise-vault-import"
aiPrompt: |
  Du bist mein Exchange- und Archivierungsassistent. Für einen Zeitraum sind keine Journalberichte im Enterprise-Vault-Journalarchiv angekommen. Hilf mir Schritt für Schritt: Zeitraum der Lücke aus den Postfachstatistiken bestimmen, Aufbewahrungsrichtlinien prüfen, die Suche in der neuen Purview-eDiscovery mit korrekt geklammerter KQL-Abfrage anlegen, den Export als PST je Postfach ohne Ordnerstruktur konfigurieren, die PST-Dateien per New-MailboxImportRequest in das Journalpostfach importieren, die importierten Unterordner per EWS in den Posteingang leeren und den Nachimport so drosseln, dass der laufende Journalstrom Vorrang behält. Weise mich auf die Grenzen hin: fehlende Umschlagempfänger und Mehrfachkopien ohne Entdopplung.
---
# Journaling-Lücke schliessen: Export aus Microsoft Purview und Import in Enterprise Vault

Journaling ist ein Transportvorgang. Exchange erzeugt den Journalbericht in dem Moment, in dem die Nachricht den Transport durchläuft, und stellt ihn dem Journalpostfach zu. Scheitert diese Zustellung, etwa weil ein Konnektor oder ein Empfängerobjekt falsch konfiguriert ist, gibt es keine Funktion, die den Bericht später nachholt. Was in diesem Zeitraum durch den Transport ging, fehlt im Archiv.

Der Inhalt der Nachrichten liegt aber weiterhin in den Postfächern der Beteiligten, und bei einer Aufbewahrungsrichtlinie ohne Ablauf auch dann, wenn Benutzer die Nachrichten gelöscht haben. Daraus ergibt sich der einzige gangbare Weg zur Nachführung: den Zeitraum aus allen Postfächern exportieren und die Kopien in das Journalarchiv einspielen. Der folgende Ablauf verwendet die neue eDiscovery in Microsoft Purview auf der Exportseite und Veritas Enterprise Vault 15 auf der Importseite; die Messwerte stammen aus einem Nachimport von mehreren hunderttausend Nachrichten, die zwei Skripte sind dabei entstanden.

## Was ein Export ersetzen kann und was nicht

Ein Journalbericht besteht aus dem Umschlag und dem Inhalt. Der Umschlag enthält die Empfänger, die der Transport tatsächlich beliefert hat: auch Blindkopien und die aufgelösten Mitglieder von Verteilerlisten. Eine Postfachkopie enthält diese Angaben nicht. Wer eine Nachricht als Blindkopie erhalten hat, ist nach dem Nachimport nicht mehr feststellbar. Der Inhalt selbst dagegen ist vollständig rekonstruierbar, sofern zwei Bedingungen erfüllt sind.

Erstens muss eine Aufbewahrung über den Postfächern liegen, damit auch gelöschte Nachrichten noch in den wiederherstellbaren Elementen liegen. Das prüfen Sie in der Security-&-Compliance-PowerShell:

```powershell
Connect-IPPSSession
Get-RetentionCompliancePolicy -DistributionDetail |
  Format-List Name, Enabled, Mode, ExchangeLocation, ExchangeLocationException
Get-RetentionComplianceRule |
  Format-List Name, Policy, RetentionComplianceAction, RetentionDuration
```

Eine Richtlinie mit `RetentionComplianceAction: Keep` und `RetentionDuration: Unlimited` über `ExchangeLocation: All` sichert den Inhalt vollständig. Die Ausnahmeliste in `ExchangeLocationException` benennt die Postfächer, für die das nicht gilt. Für diese ist nur rekonstruierbar, was dort noch vorhanden ist.

Zweitens zählt ein Export Postfachkopien, keine Nachrichten. Eine Nachricht an fünfzehn interne Empfänger liegt fünfzehnmal im Ergebnis. Im beschriebenen Fall standen für einen einzelnen Tag 98'131 Kopien rund 6'800 eindeutigen Nachrichten gegenüber, ermittelt aus der Nachrichtenverfolgung. Wie mit diesen Kopien umzugehen ist, entscheidet Compliance, nicht die IT (siehe den Abschnitt zur Entdopplung).

## Den Zeitraum bestimmen

Der Beginn der Lücke steht im Journalpostfach selbst, auf dem Exchange-Server, der es hält. Der Zeitpunkt des letzten zugestellten Berichts ist der Beginn:

```powershell
Get-MailboxFolderStatistics "journal@example.com" -IncludeOldestAndNewestItems |
  Where-Object { $_.ItemsInFolder -gt 0 } |
  Format-Table Name, ItemsInFolder, OldestItemReceivedDate, NewestItemReceivedDate -AutoSize
```

Dasselbe gilt für das Postfach, das in `JournalingReportNdrTo` der Transportkonfiguration steht. Es sammelt unzustellbare Journalberichte, und sein jüngster Eintrag bestätigt den Zeitpunkt. Das Ende der Lücke ist der Zeitpunkt, an dem die Journalregel auf das reparierte Ziel umgestellt wurde. Zwischen beiden liegt der Exportzeitraum, in UTC.

Die Nachrichtenverfolgung liefert die Sollzahl für den Abgleich: eine Zeile je Empfänger und Nachricht, mit `MessageId`. Ein historischer Bericht deckt den ganzen Zeitraum ab:

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

Die eindeutigen `MessageId`-Werte aus diesem Bericht sind die Zahl, gegen die der Nachimport am Ende gemessen wird.

## Export aus der neuen eDiscovery

Microsoft hat die klassischen Oberflächen für Content Search und eDiscovery (Standard) am 31. August 2025 abgeschaltet. Was in vielen Anleitungen noch steht, insbesondere die Option zur Entdopplung beim Export, gibt es in der neuen eDiscovery nicht mehr. Der Ablauf im Purview-Portal:

1. Unter *eDiscovery* einen Fall anlegen oder öffnen. Für den Export braucht das ausführende Konto die Rolle *eDiscovery Manager* und eine Lizenz Microsoft 365 E3 oder E5.
2. Eine Suche anlegen. Datenquelle: alle Postfächer, oder für einen Probelauf ein einzelnes Postfach.
3. Als Abfrage KQL verwenden. Die Klammern sind entscheidend, weil `AND` stärker bindet als `OR`:

```text
kind:email AND ((received>=2026-09-10 AND received<2026-09-12)
             OR (sent>=2026-09-10 AND sent<2026-09-12))
```

Ohne `kind:email` zählt die Suche auch Kalendereinträge, Kontakte und Aufgaben mit. Im beschriebenen Fall machte das den Unterschied zwischen 473'725 und 98'131 Treffern für einen Tag. Ohne die äusseren Klammern gilt die zweite Datumsbedingung nur für `sent`, und das Ergebnis enthält Nachrichten aus dem gesamten Postfachbestand.

4. *Generate statistics* ausführen. Bei mandantenweiten Suchen beschleunigt das den Export erheblich, weil nur noch Postfächer mit Treffern verarbeitet werden. Standorte, die dabei mit Fehler enden, über *Retry failed locations* nachziehen, bevor exportiert wird; sonst fehlen sie im Ergebnis, und das fällt erst nach dem Download auf.
5. *Export* mit den folgenden Einstellungen.

| Option | Wirkung |
|---|---|
| *Export type: Export items with items report* | Nachrichten plus `items.csv`; der reine Bericht enthält keine Inhalte |
| *Export format: Create PSTs for messages* | Eine PST je Postfach statt einzelner `.msg`-Dateien |
| *Maximum PST package size* | Ab dieser Grösse wird eine PST in Teile gesplittet (`.001.pst`, `.002.pst`) |
| *Maximum .zip package size* | Grösse der Download-Pakete; muss mindestens der PST-Grösse entsprechen |
| *Organize data from different locations into separate folders or PSTs* | **Einschalten**: eine Datei je Postfach, benannt `<smtp-adresse>.001.pst` |
| *Include folder and path of the source* | **Ausschalten**: alle Nachrichten landen im Ordner `Items` der PST statt in der Ordnerstruktur des Postfachs |
| *Give each item a friendly name* | Ohne Wirkung bei PST-Export |

Die Einstellung *Include folder and path of the source* entscheidet über den späteren Import. Mit Ordnerstruktur legt der Import im Journalpostfach die Ordner jedes Postfachs an, in der Sprache des jeweiligen Benutzers: `Posteingang`, `Posta in arrivo`, `Boîte de réception`. Ohne Ordnerstruktur liegt alles in einem Ordner `Items`, und der Nachimport hat einen einzigen Ort, von dem er weiterarbeitet.

Der Download liefert Zip-Pakete. Microsoft empfiehlt 7-Zip oder ein vergleichbares Werkzeug statt des Windows-Explorers. Die Pakete laufen 14 Tage nach der Erstellung ab. Die Datei `items.csv` aus dem Prozessbericht des Exports (unter *Process manager*, Export, Berichte) ist der Nachweis, was der Export enthält und was übersprungen wurde; sie gehört zum Bestand.

### Entdopplung

Die klassische Content Search kannte das Häkchen *Enable de-duplication*: Nachrichten mit gleicher `InternetMessageId`, `ConversationTopic` und `BodyTagInfo` wurden nur einmal exportiert, die übrigen Fundorte standen in `Results.csv`. Die neue eDiscovery bietet das beim direkten Export aus einer Suche nicht an. Der einzige dokumentierte Weg ist ein *Review Set*: Suchergebnisse hineinladen, Analytics laufen lassen, den automatisch erzeugten Filter *For Review* verwenden, der Duplikate ausschliesst, und aus dem Review Set exportieren. Das setzt eDiscovery Premium und damit eine E5-Lizenz für den ausführenden Benutzer voraus. In Microsoft Q&A berichten mehrere Nutzer von Duplikaten trotz dieses Ablaufs; ein Probelauf mit einem Tag ist vor dem Vollexport ratsam.

Ohne Entdopplung gilt: Enterprise Vault entdoppelt auf der Speicherebene (identische Anhänge und Nachrichtenkörper werden einmal abgelegt), aber nicht auf der Elementebene. Jede Postfachkopie wird ein eigener Archiveintrag, ein eigener Treffer in der Suche und ein eigener Zähler. Die Zählung für den Nachweis lässt sich dann nicht aus dem Archiv ableiten, sondern nur aus `items.csv` gegen die Nachrichtenverfolgung. Diese Entscheidung, Kopien akzeptieren oder über ein Review Set neu exportieren, fällt vor dem Import, nicht danach.

## Die Wege in Enterprise Vault

Enterprise Vault bringt für PST-Dateien den PST-Migrator mit, als Assistent in der Administrationskonsole (*Archives*, Rechtsklick, *Import PST*) und als skriptbare Variante über den Policy Manager EVPM. Für den Nachimport in ein Journalarchiv hat dieser Weg drei Einschränkungen.

Erstens die Lizenz. Der Migrator braucht das Feature `EVPSTM` (*Exchange PST Migrator*). In einer Site, die nur Journaling betreibt, ist es häufig nicht lizenziert, und der Assistent bricht auf der ersten Seite mit *Required license not installed* ab. Der Policy Manager läuft über denselben Migrator und meldet dasselbe.

Zweitens das Ziel. Der Assistent bietet Journalarchive nicht als Ziel an, nur Postfacharchive und Internet-Mail-Archive. Ein Import direkt in das Journalarchiv geht nur per EVPM mit `ArchiveName`, und er umgeht dabei die Verarbeitungsregeln des Journaling Tasks. Als Ziel bleibt ein eigenes Shared Archive im Vault Store des Journals, das die Suche gemeinsam mit dem Journalarchiv abdeckt.

Drittens die Nachvollziehbarkeit: Der Migrator schreibt in die PST-Datei (er markiert migrierte Elemente), der Bestand verändert sich also, und die Migrationsberichte je Datei müssen mit den Exportzahlen abgeglichen werden.

Der zweite Weg braucht keine zusätzliche Lizenz: Der Exchange Journaling Task von Enterprise Vault archiviert nicht nur Journalberichte. Erkennt er ein Element als Journalbericht, packt er es aus und übernimmt die Umschlagdaten. Ein gewöhnliches Element archiviert er direkt, so wie er es zu Zeiten des einfachen Exchange-Journalings ohne Umschlag immer getan hat. Wer die PST-Dateien in das Journalpostfach importiert, bekommt die Nachrichten vom regulären Task ins reguläre Journalarchiv geschrieben, mit derselben Aufbewahrungskategorie, derselben Indexierung und ohne Änderung an der Archivierungslandschaft.

Beweisen lässt sich das mit einer einzelnen Testnachricht an das Journalpostfach: Steht sie kurz darauf in keinem Ordner mehr, auch nicht in *Invalid Journal Report*, und findet die Suche sie im Journalarchiv, archiviert der Task gewöhnliche Nachrichten.

Dieser Weg hat zwei Eigenschaften, die den Ablauf bestimmen. Der Task verarbeitet ausschliesslich die oberste Ebene des Posteingangs, keine Unterordner. Das lässt sich am Journalpostfach ablesen: Der Suchordner *Initial Trawl With Pending* unter *Enterprise Vault Search Folders* zählt genau die Elemente des Posteingangs, Unterordner sind nicht enthalten. Und der Import über Exchange legt immer Ordner an; wie, steht im nächsten Abschnitt.

## Import in das Journalpostfach

`New-MailboxImportRequest` importiert eine PST-Datei serverseitig über den Mailbox Replication Service. Die Datei muss auf einer Freigabe liegen, auf die *Exchange Trusted Subsystem* Vollzugriff hat, und das ausführende Konto braucht die Rolle *Mailbox Import Export*, die standardmässig niemandem zugewiesen ist:

```powershell
Get-ManagementRoleAssignment -Role "Mailbox Import Export" |
  Format-Table RoleAssigneeName, RoleAssigneeType -AutoSize
New-ManagementRoleAssignment -Role "Mailbox Import Export" -User "admin@example.com"
```

Der Import selbst:

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
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-Mailbox` | Zielpostfach, hier das Journalpostfach |
| `-FilePath` | UNC-Pfad der PST-Datei; lokale Pfade werden abgelehnt |
| `-TargetRootFolder` | Ordner, unter dem der PST-Inhalt abgelegt wird; ohne Angabe werden PST-Ordner auf gleichnamige Postfachordner abgebildet |
| `-SourceRootFolder` | Ordner in der PST, ab dem importiert wird; in der Praxis wird der Ordner selbst trotzdem angelegt (siehe Text) |
| `-Name` | Eindeutiger Name des Auftrags; bei tausenden Dateien aus dem Dateinamen abgeleitet |
| `-BatchName` | Gruppierung für Abfragen und Statistiken |

</details>

Drei Beobachtungen aus dem Probelauf bestimmen den weiteren Ablauf. Ohne `TargetRootFolder` und mit einer strukturierten PST (Ordnerstruktur exportiert) legt MRS die Wurzel der PST als eigenen Ordner neben dem Posteingang an, im Fall eines deutschen Postfachs `Oberste-Ebene-des-Informationsspeichers` mit den Unterordnern `Posteingang`, `Gesendete-Elemente` und so weiter, dazu `Recoverable-Items` mit `Deletions`. Mit `TargetRootFolder "Inbox"` und einer flachen PST landen alle Nachrichten in `/Inbox/Items`. Und `SourceRootFolder "Items"` in Kombination mit `TargetRootFolder "Inbox"` ergab im Test ebenfalls `/Inbox/Items`, die Ebene wurde nicht abgestreift.

In allen drei Fällen liegen die Nachrichten ausserhalb der Sicht des Journaling Tasks. Der letzte Schritt, sie flach in den Posteingang zu verschieben, lässt sich mit Exchange-Bordmitteln nicht erledigen; er läuft über EWS. Die EWS Managed API liegt als `Microsoft.Exchange.WebServices.dll` im Installationsverzeichnis von Enterprise Vault, und das Vault Service Account hat Vollzugriff auf das Journalpostfach, sodass keine zusätzliche Berechtigung nötig ist. Das Skript läuft auf dem EV-Server in Windows PowerShell 5.1, weil die Bibliothek auf dem .NET Framework aufbaut.

Dieser Verschiebeschritt ist gleichzeitig die Drossel des ganzen Verfahrens: Er entscheidet, wie viele Nachrichten der Task auf einmal sieht.

## Der Probelauf mit einem Postfach

Bevor tausende Dateien laufen, lässt sich der ganze Weg mit einem einzelnen Postfach prüfen. Sinnvoll ist das Postfach der Person, die den Export ausgeführt hat: Sie hat das Mandat, es sind ihre eigenen Daten, und die Datei ist klein. Der Ablauf:

1. Kopie der PST-Datei in einen eigenen Ordner, SHA-256-Hash des Originals in eine CSV-Datei.
2. Import mit `TargetRootFolder "Inbox"`, Statistik mit `Get-MailboxImportRequestStatistics`: `ItemsTransferred` ist die Sollzahl.
3. Ordnerstatistik des Journalpostfachs: Wo liegen die Elemente?
4. Verschieben aus `/Inbox/Items` in den Posteingang per EWS.
5. Zählung der alten Elemente im Posteingang alle paar Minuten: Elemente mit `DateTimeReceived` vor dem Beginn des laufenden Stroms sind die importierten. Fällt die Zahl, archiviert der Task.
6. Abgleich: Suche über das Journalarchiv (in Discovery Accelerator oder der EV-Suche) mit Zeitraum und Postfachinhaber, Trefferzahl gegen `ItemsTransferred`.

Im beschriebenen Fall wurden 128 Elemente importiert und 126 archiviert. Die zwei verbliebenen waren ein Entwurf und ein Element ohne Absender; der Task lässt Elemente ohne Absender liegen. Entwürfe haben den Transport nie durchlaufen und gehören in kein Journal, deshalb schliesst das Verschiebeskript sie über das Kennzeichen `IsDraft` aus, sprachunabhängig. Elemente aus den Ordnern *Versions* und *Konflikte* (frühere Fassungen unter Aufbewahrung, Synchronisierungsreste) werden ebenfalls nicht verschoben.

Das Ereignisprotokoll `Veritas Enterprise Vault` zeigt nach dem Probelauf, was der Task abgelehnt hat. Ereignis 3071 (*could not be archived as it may be corrupt*) und 3288 (*no longer archive pending*) wiederholen sich bei jedem Durchgang für dieselben Elemente; solche Elemente sollten Sie in einen Ordner ausserhalb des Posteingangs verschieben, damit der Task sie nicht bei jedem Durchgang erneut versucht.

## Skript 1: Import in Chargen

Das erste Skript läuft in der Exchange Management Shell auf einem Exchange-Server. Es legt Importaufträge in Chargen an, wartet, protokolliert je Datei den Status und `ItemsTransferred` in eine CSV-Datei und entfernt abgeschlossene Aufträge. Wiederholte Aufrufe überspringen Dateien, die bereits im Protokoll stehen. Das Präfix `NI` steht für Nachimport.

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
<summary>Funktionen erklärt</summary>

| Funktion | Wirkung |
|---|---|
| `Get-NIDateien` | Liste aller PST-Dateien aus den Quellordnern mit abgeleitetem Auftragsnamen; gleiche Dateinamen in verschiedenen Ordnern (etwa je ein Export pro Tag) bleiben unterscheidbar |
| `Start-NICharge -Anzahl` | Legt bis zu N Importaufträge für noch nicht protokollierte Dateien an |
| `Wait-NICharge` | Wartet, bis kein Auftrag mehr läuft, schreibt Status und `ItemsTransferred` ins Protokoll, entfernt abgeschlossene Aufträge; fehlgeschlagene bleiben zur Analyse stehen. Statistik und Entfernen laufen über die Pipeline, weil `-Identity` in einer ferngesteuerten Shell das deserialisierte Identity-Objekt nicht annimmt |
| `Resume-NICharge -Anzahl -MaxLager` | Gibt angehaltene Aufträge nur frei, wenn in den Unterordnern des Posteingangs weniger als `MaxLager` Nachrichten warten |
| `Get-NIImportStand` | Zusammenfassung nach Status, Summe der importierten Elemente, Gesamtzahl der Dateien |

</details>

Zwei Verhaltensweisen von MRS sollten Sie kennen. Exchange lässt standardmässig zehn gleichzeitige Aufträge auf dasselbe Zielpostfach zu; alle weiteren stehen mit einem Grund wie `StalledDueToTarget_MailboxCapacityExceeded` oder `StalledDueToTarget_MdbReplication` in der Warteschlange und rücken nach. Das ist die Arbeitslaststeuerung, kein Kapazitätsproblem, auch wenn der Name das nahelegt. Und `Get-MailboxImportRequest` zeigt den Status mit Verzögerung; die Statistik ist aktueller.

Der Durchsatz lag bei rund sechs PST-Dateien pro Minute bei einer Durchschnittsgrösse von wenigen Megabyte. MRS ist damit deutlich schneller als Enterprise Vault. Alles, was importiert ist, liegt im Journalpostfach, bis der Task es archiviert und löscht. `Resume-NICharge` koppelt deshalb die Freigabe weiterer Aufträge an die Menge, die noch nicht verschoben ist, damit das Postfach nicht auf die Grösse des gesamten Exports anwächst.

## Skript 2: Verschieben und Takt

Das zweite Skript läuft als Vault Service Account auf dem EV-Server in Windows PowerShell 5.1. Es verschiebt Nachrichten aus allen Unterordnern des Posteingangs flach in den Posteingang, lässt Entwürfe und die Ordner *Versions* und *Konflikte* aus, und legt nur dann nach, wenn der Posteingang unter einer Schwelle liegt.

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
<summary>Funktionen erklärt</summary>

| Funktion | Wirkung |
|---|---|
| `Get-NIRueckstand` | Anzahl der Elemente im Posteingang und Anzahl der Elemente, die älter als zwei Stunden sind; letztere sind die nachimportierten, weil der laufende Journalstrom nur aktuelle Empfangszeiten trägt |
| `Get-NIUnterordner` | Alle Unterordner des Posteingangs mit Inhalt, ohne *Versions* und *Konflikte* |
| `Move-NIPortion -Max` | Verschiebt bis zu N Nachrichten ohne Entwurfskennzeichen in den Posteingang, seitenweise zu 100 über `MoveItems` mit einer typisierten `ItemId`-Liste; ein PowerShell-Array passt nicht auf die Signatur |
| `Start-NIDauerlauf -Portion -MaxPosteingang` | Endlosschleife: alle fünf Minuten den Posteingang messen, nur nachlegen, wenn er unter der Schwelle liegt; beendet sich, sobald die Datei `STOP.txt` existiert |
| `Get-NIMoveStand` | Summe der verschobenen Elemente, Fehlerzahl, aktueller Rückstand |

</details>

Die Schwelle `MaxPosteingang` ist die wichtigste Zahl des Verfahrens. Sie muss über dem normalen Arbeitsvorrat des Tasks liegen (im beschriebenen Fall rund 1'500 Berichte bei 100 Nachrichten pro Minute und etwa 20 Minuten Verzug) und so tief, dass der Task den laufenden Strom nie liegen lässt. Warum das nötig ist, zeigt der nächste Abschnitt.

## Messwerte und die Drossel

Mit den Standardeinstellungen des Journaling Tasks (5 gleichzeitige Verbindungen zum Exchange-Server, 1'000 Elemente je Durchgang) archivierte Enterprise Vault die nachimportierten Nachrichten in der ersten Nacht mit rund 5'600 pro Stunde. Am Morgen zeigte die Ordnerstatistik, was das gekostet hatte: Der Posteingang stand bei 16'500 statt 1'500. Der laufende Journalstrom, rund 6'000 Berichte pro Stunde, war um mehr als zwei Stunden in Verzug. Der Task hatte die alten Nachrichten mitverarbeitet und den aktuellen Berichten den Vorrang genommen.

Die Kapazität des Tasks ist die Summe beider Ströme. Der Nachimport darf nur den Teil bekommen, der nach dem Tagesgeschäft übrig ist. Deshalb misst der Dauerlauf den Posteingang selbst, nicht die Zahl der alten Elemente, und legt nur nach, wenn der Strom aktuell ist.

Wo die Grenze des Tasks liegt, zeigen drei Messungen. Die Warteschlangen zwischen Task und Storage Service sind MSMQ-Warteschlangen; sind die des Storage Service leer, während die des Tasks gefüllt sind, bremst der Task beim Auslesen aus Exchange, nicht der Speicher:

```powershell
Get-Counter -Counter '\MSMQ Queue(*)\Messages in Queue' -SampleInterval 5 -MaxSamples 6 |
  ForEach-Object {
    $t = $_.Timestamp.ToString("HH:mm:ss")
    $_.CounterSamples |
      Where-Object { $_.InstanceName -like "*enterprise vault*" -and $_.CookedValue -gt 0 } |
      ForEach-Object { "$t  $($_.InstanceName)  $($_.CookedValue)" }
  }
```

Auf der Exchange-Seite zählt die RPC-Latenz je Clienttyp, nicht die Anzahl der Anfragen; Werte unter 10 ms bedeuten, dass der Server Reserven hat:

```powershell
$zaehler = @('\MSExchangeIS Client Type(*)\RPC Average Latency', '\Processor(_Total)\% Processor Time')
Get-Counter -ComputerName MAILSERVER01 -Counter $zaehler -SampleInterval 5 -MaxSamples 6 |
  ForEach-Object {
    $t = $_.Timestamp.ToString("HH:mm:ss")
    $_.CounterSamples | Where-Object { $_.CookedValue -gt 0 } |
      ForEach-Object { "$t  $($_.InstanceName)  $($_.Path.Split('\')[-1])  $([math]::Round($_.CookedValue,1))" }
  }
```

Und die Drosselungsrichtlinie des Vault Service Accounts: Veritas verlangt eine Richtlinie mit `RcaMaxConcurrency: Unlimited`; mit der Standardrichtlinie begrenzt Exchange die gleichzeitigen MAPI-Verbindungen je Konto, und zusätzliche Task-Verbindungen kommen nicht an:

```powershell
Get-ThrottlingPolicyAssociation -Identity "svc-ev" | Format-List ThrottlingPolicyId
Get-ThrottlingPolicy | Format-Table Name, IsServiceAccount, RcaMaxConcurrency, EwsMaxConcurrency -AutoSize
```

Bremst der Task, liegt die passende Einstellung in den Task-Eigenschaften (*Enterprise Vault Servers*, Server, *Tasks*, Journaling Task, Registerkarte *Settings*):

| Einstellung | Standard | Wirkung |
|---|---|---|
| *Number of concurrent connections to Exchange Server* | 5 | Threads, die parallel aus dem Postfach lesen; wirkt linear, solange Exchange-Latenz und Storage-Warteschlangen unauffällig bleiben |
| *Maximum number of items per target per pass* | 1000 | Elemente je Durchgang; bei grossem Rückstand seltener neu ansetzen |

Änderungen gelten nach dem Neustart des Tasks (Rechtsklick, *Stop*, dann *Start*). Sinnvoll ist eine Verdoppelung mit anschliessender Messung über die Zeitstempel im Verschiebeprotokoll, dann die nächste Stufe. Ein zweiter Journaling Task auf dasselbe Postfach ist nicht möglich; ein Journalpostfach ist genau einem Task zugeordnet. Echte Parallelität erreichen Sie mit einem zweiten Journalpostfach und eigenem Task auf einem zweiten EV-Server, in dessen Archiv die zweite Hälfte der Dateien importiert wird. Das ist eine Änderung an der Archivierungslandschaft und gehört mit dem Betrieb abgestimmt.

## Was am Rand auffällt

Ein Nachimport dieser Grösse legt Dinge offen, die vorher niemand angesehen hat. Zwei davon aus dem beschriebenen Fall, weil sie sich wiederholen dürften.

Das Monitoring meldete auf dem Exchange-Server *RPC Requests/sec* über der Schwelle, mit Vorgabewerten von 60 und 70 Anfragen pro Sekunde aus der Check-Vorlage. Die Grundlast des Servers lag ohne Import bereits über der kritischen Schwelle. Anfragen pro Sekunde sind eine Lastzahl; die Gesundheitszahl ist die Latenz, und die lag bei unter einer Millisekunde. Die Schwelle gehört auf Basis der Grundlast aus der Monitoring-Historie neu gesetzt, etwa auf das Doppelte und Dreifache der typischen Tagesspitze, die Latenzschwelle bleibt.


Im Posteingang des Journalpostfachs lagen seit Jahren vier Elemente, die der Task bei jedem Durchgang erneut versuchte und mit Ereignis 3071 ablehnte. Sie waren in keiner Auswertung aufgefallen, weil niemand das Ereignisprotokoll `Veritas Enterprise Vault` regelmässig las. Dasselbe galt für das Auffangpostfach aus `JournalingReportNdrTo`: 117'000 unzustellbare Journalberichte seit 2020, ein Hinweis darauf, dass die Archivierung schon vor der aktuellen Lücke wiederholt Berichte verloren hatte.

## Nachweis und Abschluss

Am Ende des Nachimports stehen drei Dateien und eine Suche. `items.csv` aus dem Purview-Export belegt, was exportiert wurde. `import-protokoll.csv` enthält je Datei, was Exchange übernommen hat. `verschieben-protokoll.csv` enthält, was dem Task übergeben wurde und wann. Die Suche über das Journalarchiv mit dem Zeitraum der Lücke, aufgeteilt nach Tagen, liefert die Zahl im Archiv. Die Differenzen zwischen diesen Zahlen sind erklärbar: Entwürfe, Elemente ohne Absender, Ordner *Versions* und *Konflikte*, Postfächer ausserhalb der Aufbewahrung. Genau diese Erklärung ist die Aussage für Compliance, zusammen mit den beiden Grenzen, die kein Nachimport aufhebt: fehlende Umschlagempfänger und, ohne Entdopplung, Kopien statt Nachrichten.

Danach das Aufräumen. Die leeren Unterordner und die liegen gebliebenen Entwürfe werden aus dem Journalpostfach entfernt, die Freigabe mit den PST-Dateien wird entfernt, die PST-Dateien werden gelöscht, sobald der Abgleich abgeschlossen ist. Die Rollenzuweisung *Mailbox Import Export* wird zurückgenommen, die Task-Einstellungen bleiben, wenn der Betrieb die Messwerte kennt. Und der Vorfall selbst bekommt eine Überwachung der Journalzustellung, denn die Lücke fiel über eine Zufallsmeldung auf, nicht über ein Monitoring.

## Quellen

1.  [Microsoft Learn: Export search results in eDiscovery](https://learn.microsoft.com/en-us/purview/edisc-search-export): vollständige Liste der Exportoptionen der neuen eDiscovery, Paketgrössen, Verhalten von *Organize data* und *Include folder and path*, 14-Tage-Ablauf der Pakete.

2.  [Microsoft Learn: Deduplication in eDiscovery search results](https://learn.microsoft.com/en-us/purview/ediscovery-de-duplication-in-search-results): Vergleichseigenschaften der klassischen Entdopplung und der Hinweis auf die Abschaltung der klassischen Oberflächen am 31. August 2025.

3.  [Microsoft Learn: Export items from a review set in eDiscovery](https://learn.microsoft.com/en-us/purview/edisc-review-set-export): der Exportweg über ein Review Set, der in der neuen eDiscovery als einziger Duplikate ausschliesst.

4.  [Microsoft Q&A: eDiscovery cases, deduplication on export](https://learn.microsoft.com/en-us/answers/questions/2201291/ediscovery-cases-deduplication-on-export): Bestätigung, dass der direkte Export keine Entdopplung mehr bietet, mit Erfahrungsberichten zum Review-Set-Weg.

5.  [Microsoft Learn: New-MailboxImportRequest](https://learn.microsoft.com/en-us/powershell/module/exchange/new-mailboximportrequest): Parameter des Importauftrags, Voraussetzungen für Freigabe und Rolle *Mailbox Import Export*.

6.  [Microsoft Learn: Mailboxes are stalled during a migration](https://learn.microsoft.com/en-us/troubleshoot/exchange/migration/mailboxes-stalled-during-migration): Arbeitslaststeuerung mit zehn gleichzeitigen Aufträgen je Ziel und die `StalledDueToTarget`-Zustände als erwartetes Verhalten.

7.  [Veritas: Enterprise Vault PST Migration, wizard-assisted migration](https://www.veritas.com/support/en_US/doc/95955885-161896939-0/v11744603-161896939): Ablauf des PST-Migrators, Zugriffsanforderungen des Storage Service und die zulässigen Zielarchivtypen.

8.  [Veritas VOX: Import PST to a Journal Archive](https://vox.veritas.com/t5/Enterprise-Vault/Import-PST-to-a-Journal-Archive/td-p/272928): Erfahrungen aus der Community, warum der Assistent Journalarchive nicht anbietet und wie EVPM mit `ArchiveName` arbeitet.
