---
title: "EXO-Migration ohne Remote Move"
navTitle: "EXO-Migration ohne Remote Move"
description: "Wie lokale Exchange-Mailboxen kontrolliert als neue, leere Exchange-Online-Mailboxen bereitgestellt werden: PST-Sicherung, CSV-Freigabe, RemoteMailbox, Synchronisation, Validierung und Rollback."
date: "2026-08-07"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "12 Min. Lesezeit"
themen:
  - "exchange-onprem-hybrid"
  - "microsoft-365-exchange"
  - "active-directory-entra"
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange"
protokolle:
  - "powershell"
  - "migration"
related:
  - "typische-ursachen-fuer-mail-loops-und-deren-behebung"
  - "totemomail-m365"
  - "ghost-sender-exchange-online-nebeneingang"
slug: "exchange-mailboxen-ohne-remote-move-cloud-neu-erstellen"
translationId: "article-8f3c1b7a62d94e50"
url: "https://rafaelpfister.ch/blog/exchange-mailboxen-ohne-remote-move-cloud-neu-erstellen"
draft: false
---

# EXO-Migration ohne Remote Move

Ein Hybrid Remote Move ist der normale Weg, um ein lokales Exchange-Postfach mitsamt Inhalt nach Exchange Online zu verschieben. Nicht jede Organisation erlaubt diesen Migrationspfad. Wenn eine Sicherheitsrichtlinie Remote Moves ausschliesst, kann ein bewusst anderer Ansatz vertretbar sein: Das lokale Postfach wird als PST gesichert, vom synchronisierten AD-Benutzer getrennt und für denselben Benutzer wird eine neue, leere Mailbox in Exchange Online bereitgestellt.

Dieser Weg ist **keine Mailbox-Migration**. Er überträgt weder Nachrichten noch Kalender, Regeln oder Berechtigungen in die Cloud. Die PST dient ausschliesslich als Sicherung und wird in diesem Szenario nicht importiert. Der Ablauf eignet sich daher nur, wenn eine leere Ziel-Mailbox fachlich akzeptiert und der Verlust der aktiven Postfachkonfiguration ausdrücklich freigegeben ist.

## Zielzustand und harte Voraussetzungen

Nach dem Cutover bleibt derselbe AD-Benutzer bestehen. Lokal wird er jedoch nicht mehr als `UserMailbox`, `SharedMailbox`, `RoomMailbox` oder `EquipmentMailbox`, sondern als `RemoteMailbox` verwaltet. Nach der Synchronisation repräsentiert dieses Objekt die neue Mailbox in Exchange Online.

Der gewünschte Zustand sieht so aus:

1. Das lokale Postfach ist vollständig als PST gesichert.
2. Das lokale Postfach ist getrennt, aber innerhalb der konfigurierten Retention noch nicht endgültig gelöscht.
3. Der bestehende AD-Benutzer ist als RemoteMailbox aktiviert.
4. Primäre Adresse, Aliase und der alte `LegacyExchangeDN` bleiben erhalten.
5. Entra Connect hat die Änderungen synchronisiert.
6. Für Benutzerpostfächer ist ein Exchange-Online-Serviceplan zugewiesen.
7. Exchange Online zeigt ein echtes Cloud-Postfach und der Mailflow endet dort.

Vor dem Start müssen ausserdem geklärt sein:

- Die PST-Freigabe ist per UNC erreichbar. Die Gruppe `Exchange Trusted Subsystem` besitzt dort Lese- und Schreibrechte.
- Das ausführende Konto hat die Management-Rolle `Mailbox Import Export`.
- Die PST ist nur die vereinbarte Sicherung; es ist kein späterer Import eingeplant.
- Aufbewahrung, Litigation Hold, eDiscovery und regulatorische Vorgaben sind separat geprüft.
- Stellvertretungen, Send-As, Send-on-Behalf, Weiterleitungen, Inbox-Regeln, mobile Geräte und Applikationszugriffe sind inventarisiert.
- Während Export und Umschaltung werden eingehende Nachrichten am vorgelagerten Gateway kontrolliert zurückgehalten. Benutzer und Anwendungen dürfen nicht mehr in das Quellpostfach schreiben.
- Die Retention der lokalen Mailbox-Datenbank deckt das Rollback-Fenster ab.

## Warum eine CSV-Freigabeliste unverzichtbar ist

Eine direkte Pipeline wie `Get-Mailbox | Disable-Mailbox` ist für diesen Vorgang zu riskant. Sie würde auch System-, Discovery- oder anderweitig nicht freigegebene Postfächer erfassen können. Der folgende Ablauf arbeitet deshalb mit zwei expliziten Freigaben:

- `Action=CUTOVER` bestimmt, welche Zeile tatsächlich umgestellt werden darf.
- `PstVerified=YES` bestätigt, dass die Exportdatei technisch und organisatorisch geprüft wurde.

Zuerst wird nur das Inventar erzeugt:

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
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `Get-Mailbox -ResultSize Unlimited` | Hebt die Standardgrenze von 1000 Ergebnissen auf; ohne diesen Parameter fehlen in grossen Umgebungen Postfächer im Inventar |
| `-RecipientTypeDetails UserMailbox,SharedMailbox,RoomMailbox,EquipmentMailbox` | Beschränkt die Abfrage auf die vier umzustellenden Postfachtypen; System- und Discovery-Postfächer bleiben aussen vor |
| `Sort-Object PrimarySmtpAddress` | Sortiert die Ausgabe nach der primären SMTP-Adresse, damit die CSV-Datei bei der fachlichen Durchsicht stabil geordnet ist |
| `Export-Csv -Path` | Zielpfad der CSV-Datei |
| `-NoTypeInformation` | Unterdrückt die Typ-Kopfzeile `#TYPE ...`, die ältere PowerShell-Versionen sonst als erste Zeile schreiben |
| `-Encoding UTF8` | Schreibt die Datei UTF-8-kodiert, damit Umlaute in Anzeigenamen korrekt erhalten bleiben |

</details>

Die Datei wird anschliessend fachlich bereinigt. Nur tatsächlich freigegebene Postfächer erhalten `Action=CUTOVER`. Systemmailboxen und Sonderobjekte gehören nicht in diese Liste.

## Phase 1: Primärpostfach und Archiv als PST sichern

`New-MailboxExportRequest` schreibt nur auf einen UNC-Pfad. Für jedes Postfach wird ein eindeutiger Dateiname erzeugt. Ein aktives Online-Archiv des lokalen Exchange wird separat exportiert:

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
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `Import-Csv $CsvPath` | Liest die Freigabeliste ein; jede Zeile wird zu einem Objekt mit den CSV-Spalten als Eigenschaften |
| `Where-Object Action -eq "CUTOVER"` | Verarbeitet nur explizit freigegebene Zeilen |
| `New-MailboxExportRequest -Mailbox` | Quellpostfach des Exports (hier die Identity aus der CSV-Zeile) |
| `-FilePath` | Zielpfad der PST-Datei; muss ein UNC-Pfad sein, lokale Pfade lehnt das Cmdlet ab |
| `-Name` | Eindeutiger Name des Requests; erlaubt später die gezielte Zuordnung von Primär- und Archiv-Export |
| `-BatchName` | Fasst alle Requests eines Laufs unter einem Batch-Namen zusammen; Grundlage für Statusabfrage und Aufräumen |
| `-IsArchive` | Exportiert das Online-Archiv statt des Primärpostfachs; deshalb der zweite Request pro Postfach mit aktivem Archiv |

</details>

Der Export ist erst freigegeben, wenn **jeder** Request den Status `Completed` hat:

```powershell
Get-MailboxExportRequest -BatchName $BatchName |
    Get-MailboxExportRequestStatistics -IncludeReport |
    Format-Table DisplayName,Status,PercentComplete,FailureCode,Message -AutoSize

Get-MailboxExportRequest -BatchName $BatchName |
    Where-Object Status -ne "Completed"
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `Get-MailboxExportRequest -BatchName` | Listet alle Export-Requests des angegebenen Batches |
| `Get-MailboxExportRequestStatistics -IncludeReport` | Ergänzt die Statistik um den detaillierten Verlaufsbericht, in dem Fehlerursachen einzelner Requests stehen |
| `Format-Table ... -AutoSize` | Tabellarische Anzeige der genannten Eigenschaften; `-AutoSize` passt die Spaltenbreiten an den Inhalt an |
| `Where-Object Status -ne "Completed"` | Filtert auf alle noch nicht abgeschlossenen oder fehlgeschlagenen Requests; die Ausgabe muss leer sein, bevor es weitergeht |

</details>

Zusätzlich sind Existenz, Grösse, Lesbarkeit, Backup-Übernahme und Zugriffsschutz der Dateien zu prüfen. Erst danach wird für die entsprechende CSV-Zeile `PstVerified=YES` gesetzt.

## Phase 2: Postfachdaten und Exchange-Attribute sichern

Vor der ersten Änderung wird pro Postfach ein maschinenlesbarer Snapshot erstellt. Er ist wichtiger als ein Screenshot, weil Aliase, GUIDs und der `LegacyExchangeDN` später exakt rekonstruiert werden können:

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
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `New-Item -ItemType Directory -Path ... -Force` | Legt das Snapshot-Verzeichnis an; `-Force` unterdrückt den Fehler, falls es bereits existiert |
| `Get-Mailbox -Identity` | Holt das aktuelle Mailbox-Objekt zur jeweiligen CSV-Zeile |
| `Select-Object Identity,...,ServerName` | Reduziert das Objekt auf die Attribute, die für eine spätere Rekonstruktion nötig sind (GUIDs, Adressen, `LegacyExchangeDN`, Datenbank) |
| `Export-Clixml` | Serialisiert das Objekt als typerhaltendes CLIXML; im Gegensatz zu CSV bleiben Mehrfachwerte wie `EmailAddresses` vollständig erhalten und sind per `Import-Clixml` wieder einlesbar |

</details>

Delegationen und Weiterleitungen benötigen eigene Exporte. Mindestens diese Informationen sollten separat gesichert werden:

```powershell
$mailbox = Get-Mailbox -Identity user01@contoso.com

$mailbox | Format-List ForwardingAddress,ForwardingSmtpAddress,DeliverToMailboxAndForward
Get-MailboxPermission -Identity $mailbox.Identity
Get-ADPermission -Identity $mailbox.DistinguishedName
Get-InboxRule -Mailbox $mailbox.Identity
Get-CalendarProcessing -Identity $mailbox.Identity -ErrorAction SilentlyContinue
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `Format-List ForwardingAddress,ForwardingSmtpAddress,DeliverToMailboxAndForward` | Zeigt die drei Weiterleitungsattribute des Postfachs in Listendarstellung |
| `Get-MailboxPermission -Identity` | Listet Postfachberechtigungen wie Full Access |
| `Get-ADPermission -Identity` | Listet AD-Berechtigungen auf dem Benutzerobjekt, darunter Send-As; erwartet hier den Distinguished Name |
| `Get-InboxRule -Mailbox` | Listet die serverseitigen Posteingangsregeln des Postfachs |
| `Get-CalendarProcessing -Identity` | Zeigt die Buchungskonfiguration; relevant für Raum- und Gerätepostfächer |
| `-ErrorAction SilentlyContinue` | Unterdrückt den Fehler bei Postfachtypen ohne Buchungskonfiguration, damit die Sicherung nicht abbricht |

</details>

Diese Konfigurationen wechseln nicht automatisch auf die neue Cloud-Mailbox.

## Phase 3: Lokale Mailbox trennen und RemoteMailbox aktivieren

Der eigentliche Cutover ist kurz, aber folgenreich. `Disable-Mailbox` entfernt die Exchange-Attribute vom AD-Benutzer und trennt das lokale Postfach. Die Mailbox-Daten bleiben bis zum Ablauf der Datenbank-Retention als disconnected mailbox erhalten. Direkt danach aktiviert `Enable-RemoteMailbox` denselben AD-Benutzer für Exchange Online.

Das folgende Skript verarbeitet ausschliesslich doppelt freigegebene Zeilen. Es bewahrt die primäre SMTP-Adresse, alle bestehenden Proxy-Adressen und den alten `LegacyExchangeDN` als X500-Adresse. Der X500-Eintrag verhindert NDRs bei Antworten auf ältere Nachrichten oder bei alten Outlook-Autocomplete-Einträgen.

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
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `Get-Mailbox -Identity ... -ErrorAction Stop` | Holt das Quellpostfach; `-ErrorAction Stop` macht einen Lookup-Fehler zum abbrechenden Fehler, statt still weiterzulaufen |
| `Disable-Mailbox -Identity` | Entfernt die Exchange-Attribute vom AD-Benutzer und trennt das lokale Postfach; die Daten bleiben als disconnected mailbox in der Datenbank |
| `-Confirm:$false` | Unterdrückt die interaktive Rückfrage; die Freigabe erfolgt hier über die CSV-Liste, nicht am Prompt |
| `Enable-RemoteMailbox -Identity` | Aktiviert denselben AD-Benutzer als RemoteMailbox für Exchange Online |
| `-Alias` | Setzt den Exchange-Alias wieder auf den Wert aus der Freigabeliste |
| `-PrimarySmtpAddress` | Erhält die bisherige primäre SMTP-Adresse |
| `-RemoteRoutingAddress` | Zieladresse in der `mail.onmicrosoft.com`-Routing-Domain, über die der lokale Exchange die Cloud-Mailbox erreicht |
| `-ACLableSyncedObjectEnabled` | Kennzeichnet das Objekt als ACL-fähig, damit Berechtigungen wie Full Access nach der Synchronisation in Exchange Online auswertbar bleiben |
| `-Shared` / `-Room` / `-Equipment` | Erzeugt statt eines Benutzerpostfachs den jeweiligen Spezialtyp; das Skript setzt genau einen Schalter passend zum Quelltyp |
| `Set-RemoteMailbox -EmailAddressPolicyEnabled $false` | Nimmt das Objekt von der E-Mail-Adressrichtlinie aus, damit diese die manuell gesetzten Adressen nicht überschreibt |
| `-EmailAddresses` | Setzt die komplette, deduplizierte Adressliste inklusive alter Proxy-Adressen, Routing-Adresse und X500-Eintrag |
| `Get-RemoteMailbox -Identity` | Kontrollabfrage des Ergebnisses direkt nach der Umstellung |

</details>

Das Skript ist absichtlich kein vollautomatisches Migrationswerkzeug. Es beendet den Lauf beim ersten Widerspruch, damit ein Administrator Ursache und Zustand beurteilen kann. Vor einem produktiven Batch sollte der Code mit wenigen Testpostfächern und den eingesetzten Exchange-Versionen validiert werden.

## Phase 4: Synchronisieren, lizenzieren und verifizieren

Nach der lokalen Änderung wird auf dem Entra-Connect-Server ein Delta-Zyklus gestartet:

```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-PolicyType Delta` | Synchronisiert nur die seit dem letzten Zyklus geänderten Objekte; die Alternative `Initial` wäre ein vollständiger, deutlich längerer Durchlauf |

</details>

Für Benutzerpostfächer muss anschliessend ein gültiger Exchange-Online-Serviceplan zugewiesen sein, beispielsweise über gruppenbasierte Lizenzierung. Shared-, Raum- und Gerätepostfächer sind nach den aktuellen Microsoft-Lizenzbedingungen und den benötigten Funktionen zu beurteilen.

Die Provisionierung ist asynchron. Microsoft nennt für normale Änderungen meist weniger als 30 Minuten, in einzelnen Fällen jedoch bis zu 24 Stunden. Während dieser Zeit sollte der vorgelagerte Mailflow Nachrichten kontrolliert zurückhalten, statt sie an ein noch nicht bereites Ziel zuzustellen.

Die lokale Kontrolle muss nun eine RemoteMailbox zeigen:

```powershell
Get-RemoteMailbox -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,RemoteRoutingAddress,EmailAddresses

Get-Mailbox -Identity user01@contoso.com -ErrorAction SilentlyContinue
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `Get-RemoteMailbox -Identity` | Muss das umgestellte Objekt als RemoteMailbox liefern |
| `Format-List RecipientTypeDetails,...` | Zeigt Typ, Adressen und Routing-Adresse zur Kontrolle in Listendarstellung |
| `Get-Mailbox -Identity ... -ErrorAction SilentlyContinue` | Gegenprobe: Der Aufruf darf nichts mehr liefern, denn lokal existiert kein verbundenes Postfach mehr; `-ErrorAction SilentlyContinue` unterdrückt die dabei erwartete Fehlermeldung |

</details>

In Exchange Online wird geprüft, ob aus dem bisherigen MailUser ein echtes Postfach geworden ist:

```powershell
Get-EXORecipient -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,EmailAddresses

Get-EXOMailbox -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,ExchangeGuid
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `Get-EXORecipient -Identity` | Zeigt den Empfängertyp in Exchange Online; erwartet wird `UserMailbox` beziehungsweise der Spezialtyp, nicht mehr `MailUser` |
| `Get-EXOMailbox -Identity` | Liefert nur echte Cloud-Postfächer; ein Treffer belegt die abgeschlossene Provisionierung |
| `Format-List ...,ExchangeGuid` | Listet die Kontrollattribute; die `ExchangeGuid` identifiziert die neue Cloud-Mailbox eindeutig |

</details>

Der Batch gilt erst als abgeschlossen, wenn zusätzlich folgende Tests erfolgreich sind:

- Zustellung von extern und intern
- Versand nach extern und intern
- Antwort auf eine alte Nachricht, um die X500-Adresse zu prüfen
- Anmeldung mit Outlook und Outlook im Web
- Stellvertretungen und Send-As
- Weiterleitungen und Transportregeln
- Raum- und Gerätebuchungen
- Anwendungen, Scanner und SMTP-Relays
- Message Trace mit Zustellung an die neue Cloud-Mailbox

## Rollback und Aufräumen

Das lokale Quellpostfach darf während der Validierungsphase nicht mit `Remove-StoreMailbox` gelöscht werden. Solange es innerhalb der Mailbox-Retention als disconnected mailbox vorhanden ist, besteht noch eine technische Rückfallmöglichkeit. Ein Rollback erfordert jedoch eine kontrollierte Umkehr der RemoteMailbox-Attribute und das erneute Verbinden der lokalen Mailbox; gleichzeitig muss verhindert werden, dass zwei aktive Zustellziele entstehen.

Vor einem Rollback sind deshalb Mailflow, Synchronisationszustand und bereits in der Cloud eingegangene Nachrichten zu sichern. Ein Rückwechsel ist kein einfacher Einzeiler und gehört als getestetes Runbook zum Change.

Nach erfolgreicher Abnahme werden Export-Requests bereinigt, PST-Dateien gemäss Schutz- und Aufbewahrungskonzept archiviert und temporäre Berechtigungen auf der Exportfreigabe entfernt:

```powershell
Get-MailboxExportRequest -BatchName $BatchName |
    Remove-MailboxExportRequest -Confirm:$false
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `Get-MailboxExportRequest -BatchName` | Wählt genau die Export-Requests des abgeschlossenen Batches aus |
| `Remove-MailboxExportRequest -Confirm:$false` | Entfernt die Requests ohne Rückfrage; die PST-Dateien selbst bleiben davon unberührt |

</details>

Die disconnected mailboxes sollten erst nach Ablauf des vereinbarten Rollback-Fensters und gemäss Retention-Konzept endgültig bereinigt werden.

## Fazit

Wenn Hybrid Remote Moves nicht erlaubt sind und keine Postfachdaten in Exchange Online übernommen werden müssen, lässt sich ein bestehender synchronisierter AD-Benutzer kontrolliert von einer lokalen Mailbox auf eine neue Cloud-Mailbox umstellen. Der kritische Teil ist nicht `Enable-RemoteMailbox`, sondern die Prozesskontrolle darum herum: vollständige Inventarisierung, verifizierte PST-Sicherung, explizite Freigaben, Erhalt der Proxy- und X500-Adressen, ein kontrollierter Mailflow sowie ein echtes Rollback-Fenster.

## Quellen

1. [Microsoft Learn – Enable-RemoteMailbox](https://learn.microsoft.com/en-us/powershell/module/exchange/enable-remotemailbox?view=exchange-ps): Aktiviert einen bestehenden lokalen AD-Benutzer für eine Mailbox im cloudbasierten Dienst und dokumentiert die Schalter für Benutzer-, Shared-, Raum- und Gerätepostfächer.

2. [Microsoft Learn – New-MailboxExportRequest](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-mailboxexportrequest?view=exchange-ps): Referenz für den Export primärer und archivierter lokaler Postfächer in PST-Dateien.

3. [Microsoft Learn – Mailbox import and export requests](https://learn.microsoft.com/en-us/exchange/mailbox-import-and-export-requests-exchange-2013-help): Voraussetzungen für Exportfreigabe, Berechtigungen und Exchange Trusted Subsystem.

4. [Microsoft Learn – Disable or delete a mailbox in Exchange Server](https://learn.microsoft.com/en-us/exchange/recipients/disconnected-mailboxes/disable-or-delete-mailboxes): Verhalten von `Disable-Mailbox`, Entfernung der Exchange-Attribute und Aufbewahrung des getrennten Postfachs.

5. [Microsoft Learn – Disconnected mailboxes](https://learn.microsoft.com/en-us/exchange/recipients/disconnected-mailboxes/disconnected-mailboxes): Wiederverbinden, Wiederherstellen und endgültiges Löschen getrennter Mailboxen.

6. [Microsoft Learn – Delays in provisioning of a user or mailbox](https://learn.microsoft.com/en-us/troubleshoot/exchange/user-and-shared-mailboxes/delays-provision-mailbox-sync-changes): Typische Dauer und Fehlersuche bei der Provisionierung in Exchange Online.

7. [Microsoft Learn – Move mailboxes between on-premises and Exchange Online organizations](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/move-mailboxes): Der reguläre Hybrid-Remote-Move als Referenz und Abgrenzung zum hier beschriebenen Neuaufbau.
