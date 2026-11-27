---
title: "Exchange-EDB-Restore mit Ontrack PowerControls"
navTitle: "EDB-Restore mit PowerControls"
description: "Einzelne Mails, Ordner oder ganze Postfächer aus einer Exchange-EDB wiederherstellen, ohne die Datenbank zu mounten: Dateien bereitstellen, EDB öffnen, Ziel verbinden (Exchange, Exchange Online, PST), wiederherstellen und exportieren."
date: "2026-11-27"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "12 Min. Lesezeit"
themen:
  - "exchange-onprem-hybrid"
  - "microsoft-365-exchange"
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange-on-premises"
  - "exchange-online"
protokolle:
  - "backup-dr"
  - "powershell"
slug: "ontrack-powercontrols-exchange-edb-restore"
translationId: "article-ed21bfa955fa52d2"
url: "https://rafaelpfister.ch/blog/ontrack-powercontrols-exchange-edb-restore"
draft: false
---

Kurzfazit: Ontrack PowerControls öffnet eine Exchange-Datenbank (`.edb`) direkt als Datei, ohne Exchange-Server und ohne Mount. Postfächer, Ordner und einzelne Elemente lassen sich durchsuchen und per Copy-Paste in ein Exchange-Postfach, ein Exchange-Online-Postfach oder eine PST-Datei zurückschreiben. Die Anleitung bezieht sich auf PowerControls 9.6.4 (Oktober 2025), die erste Version mit Support für Exchange Server SE, Outlook 2024 und Windows Server 2025.

## Wann PowerControls statt Recovery Database

Exchange bringt mit der Recovery Database (RDB) ein eigenes Verfahren mit: Datenbank aus dem Backup zurückspielen, als RDB mounten und mit `New-MailboxRestoreRequest` in ein Produktivpostfach kopieren. Das setzt einen Exchange-Server mit passender Version, genügend Speicherplatz auf dem Mailbox-Server und eine konsistente Datenbank voraus.

PowerControls ist in diesen Fällen die bessere Wahl:

- **Kein passender Exchange mehr vorhanden:** Die Datenbank stammt von einer abgelösten Version, etwa aus einem Archiv-Backup von Exchange 2010, und die Umgebung läuft inzwischen auf Exchange Server SE oder Exchange Online.
- **Granulare Suche:** Gesucht wird eine bestimmte Mail über mehrere Postfächer oder mehrere Backup-Stände hinweg, mit Filter auf Betreff, Inhalt, Anhang und Datum.
- **Export statt Restore:** Ergebnisse sollen als PST oder MSG an Juristen, Revision oder den Benutzer gehen, nicht zurück ins Postfach.
- **Restore nach Exchange Online:** Die EDB liegt nur noch aus der On-Premises-Zeit vor, das Postfach ist bereits migriert.

PowerControls verändert die `.edb` und die zugehörigen Logdateien nicht. Es legt beim ersten Öffnen lediglich Index-Dateien (`.idx`, `.dat`) im temporären Pfad an.

## Voraussetzungen

| Punkt | Anforderung |
|---|---|
| Version | PowerControls 9.6.4 für Exchange Server SE als Quelle oder Ziel |
| Restore-Rechner | Windows-Client oder Windows Server mit Outlook (Outlook 2016 und neuer verbinden nur noch über Autodiscover) |
| Lizenz | Die Trial-Version läuft 30 Tage. Sie öffnet Quellen und Ziele, sucht, prüft und erstellt Reports, kopiert und exportiert aber nicht |
| Rechte Exchange On-Premises | Full Access auf jedes Zielpostfach, in das wiederhergestellt wird |
| Rechte Postfacherstellung | AD-Rechte zum Anlegen von Postfächern und die Exchange Management Tools in derselben Version und demselben CU wie der Zielserver |
| Rechte Exchange Online | App-Registrierung in Entra ID mit `full_access_as_app` (EWS), siehe Abschnitt Exchange Online |
| Speicherplatz | Platz für die EDB-Kopie plus temporäre Index-Dateien in der Grössenordnung der Logdateien |

Die vollständige Liste der unterstützten Betriebssysteme, Outlook-Versionen und Exchange-Versionen steht im Readme des Installationspakets.

### Version vor dem Öffnen prüfen

Prüfen Sie unter **Help > About**, welche PowerControls-Version installiert ist, und vergleichen Sie sie mit dem Build der Exchange-Server, von denen die Datenbank stammt:

```powershell
Get-ExchangeServer | Format-Table Name, AdminDisplayVersion -AutoSize
```

| Build | Version | PowerControls |
|---|---|---|
| `15.2.1544` und tiefer | Exchange 2019 CU14 und älter | ab 9.2 |
| `15.2.1748` | Exchange 2019 CU15 (Februar 2025) | neuer als 9.6.2 |
| `15.2.2562` und höher | Exchange Server SE | 9.6.4 |

Zeigt PowerControls nach dem Öffnen die Ordnerstruktur an, aber in keinem Ordner Elemente, ist eine zu alte PowerControls-Version die erste Ursache, die Sie ausschliessen sollten.

### Update installieren

PowerControls hat keine eingebaute Update-Funktion. Eine neue Version wird über die bestehende installiert:

1. Das Installationspaket beim Ontrack-Support mit der Lizenznummer aus **Help > About > License Info** anfordern. Alternativ kommt es über das Trial-Formular auf der Ontrack-Website per Mail. Trial- und Vollversion sind dasselbe Paket, den Unterschied macht die Lizenzdatei.
2. Offene Quellen und Ziele mit **Close Store** schliessen und PowerControls beenden. Die Lizenzdatei sichern.
3. Setup als Administrator ausführen.
4. Unter **Help > About** Version und Lizenz prüfen. Läuft PowerControls danach als Free Trial, die Lizenzdatei erneut einspielen.
5. Die alten `.idx`- und `.dat`-Dateien im temporären Pfad löschen, damit die neue Version den Index neu aufbaut.

Setzen Sie den Ontrack Administrative Server (OAS) für Rechteverwaltung oder Auditing ein, muss der OAS vor dem Update auf 9.6.4 zuerst auf 9.6.3 aktualisiert werden.

## Datenbankdateien bereitstellen

Zuerst müssen die Datenbankdateien als Dateien vorliegen. Bei einem Image- oder VSS-Backup (Veeam, Rubrik, Windows Server Backup, Datto) stellen Sie die Dateien per File-Level-Restore in ein Arbeitsverzeichnis zurück oder hängen den Snapshot als Volume am Restore-Rechner ein. Ein eingehängter Snapshot ist meist nur lesbar. Das genügt, weil PowerControls die EDB und die Logs nicht verändert; der temporäre Pfad muss dann auf einem beschreibbaren Laufwerk liegen. Bei klassischen Tape- und Agent-Backups (Backup Exec, NetBackup, Commvault, Data Protector, NetWorker, IBM TSM, Arcserve) extrahiert der separat lizenzierte Ontrack ExtractWizard die EDB direkt aus dem Backup-Satz.

Legen Sie die Dateien so ab:

```text
D:\Restore\DB01\DB01.edb
D:\Restore\DB01\Logs\E00.chk
D:\Restore\DB01\Logs\E00*.log
D:\Restore\Temp\
```

- `.edb` in einem Verzeichnis, `.log` und `.chk` in einem zweiten Verzeichnis. Die beiden Verzeichnisse dürfen verschieden sein.
- Die Logdateien sind nicht zwingend, aber ohne sie fehlen Transaktionen, die zum Zeitpunkt des Backups noch nicht in die EDB geschrieben waren. Je nach Zustand der Datenbank meldet PowerControls dann Korruptionsfehler.
- Den temporären Pfad auf ein Volume mit ausreichend freiem Speicherplatz legen, nicht auf das Systemlaufwerk. Ab Exchange 2007 ist jede Logdatei 1 MB gross; 3000 Logs benötigen also rund 3 GB im temporären Pfad.

Stammt die Datenbank aus einer DAG, enthält das Log-Verzeichnis neben den aktuellen Logs weitere Unterordner. Für PowerControls zählt nur der Hauptordner mit `E00.chk` und den `E00*.log`-Dateien:

| Inhalt | Bedeutung | Für PowerControls |
|---|---|---|
| `E00.chk`, `E00*.log` im Hauptordner | Aktuelle Checkpoint- und Logdateien | Diesen Ordner auswählen |
| `*.jrs` im Hauptordner | Reserve-Logs der Datenbank-Engine | Werden nicht benötigt, stören nicht |
| `IgnoredLogs` | Vom Replikationsdienst verworfene Logs einer passiven Kopie | Nicht auswählen |
| `<Zeitstempel>E00OutOfDate` | Alte Logs nach einem Reseed oder einem divergierten Log-Stream | Nicht auswählen |
| `incseedInspect` | Arbeitsordner des inkrementellen Reseeds | Nicht auswählen |

Ob der Hauptordner die erwarteten Dateien enthält, zeigt eine Zählung nach Dateiendung:

```powershell
Get-ChildItem "D:\Restore\DB01\Logs" -File |
  Group-Object Extension | Select-Object Name, Count
```

Den Zustand der Datenbank können Sie vorab mit `eseutil` prüfen. `eseutil.exe` liegt im `Bin`-Verzeichnis jeder Exchange-Installation:

```powershell
eseutil /mh "D:\Restore\DB01\DB01.edb"
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `/m` | File-Dump-Modus: liest Metadaten aus der Datei, ohne sie zu verändern |
| `h` | Gibt den Datenbank-Header aus, darunter `State` und `Log Required` |
| `"D:\…\DB01.edb"` | Pfad zur Datenbankdatei |

</details>

`State: Clean Shutdown` bedeutet, dass die Datenbank ohne Logdateien konsistent ist. Bei `Dirty Shutdown` zeigt `Log Required` den Bereich der Logdateien, die noch eingespielt werden müssen. Diese Logs gehören dann in das Log-Verzeichnis. PowerControls spielt sie beim Öffnen selbst ein. Ein `eseutil /r` oder `eseutil /p` auf der Datenbank ist dafür nicht nötig. Führen Sie `eseutil /p` nie auf der einzigen Kopie aus: Der Hard Repair entfernt beschädigte Seiten endgültig.

## EDB als Quelle öffnen

Beim Start öffnet PowerControls den Data Wizard. Alternativ starten Sie ihn über **File > Use Wizard** oder öffnen die Quelle direkt über **File > Open Source**.

1. Auf der Seite **Source Selection** unter **Select Source** den Eintrag **EDB File (\*.edb)** wählen.
2. Unter **EDB Path (\*.edb)** die Datenbankdatei auswählen.
3. Unter **Log File Path (\*.log, \*.pat, \*.chk)** das Log-Verzeichnis eintragen. PowerControls setzt dieses Feld nach der EDB-Auswahl automatisch auf das Verzeichnis der EDB, auch wenn die Logs woanders liegen. Füllen Sie das Feld deshalb erst nach der EDB aus und kontrollieren Sie es vor **Next**. Der Dialog **Browse For Folder** zeigt nur Ordner an, keine `.chk`- oder `.log`-Dateien; ausgewählt wird der Ordner, in dem sie liegen. Den Pfad können Sie auch direkt in das Feld einfügen.
4. Unter **Temporary File Path (\*.idx, \*.dat, \*.lkp)** das vorbereitete Temp-Verzeichnis eintragen.
5. Optional die Anzeige unter **Mailbox Criteria** auf bestimmte Postfächer einschränken. Bei Datenbanken mit vielen Postfächern beschleunigt das die weitere Arbeit.
6. **Next** wählen.

Auf der Seite **Now Processing Data File** verarbeitet PowerControls die Datenbank in drei Schritten:

1. Pre-Scan der Logdateien (Integritätsprüfung)
2. Einspielen der Logdateien
3. Hashing der EDB (Aufbau der Ordnerhierarchie)

Erscheint die Meldung **No log files found**, steht im Feld **Log File Path** meist noch der EDB-Ordner. Brechen Sie in diesem Fall mit **Cancel** ab und korrigieren Sie den Pfad. Mit **OK** öffnet PowerControls die Datenbank ohne Logs; bei einer Datenbank im Zustand `Dirty Shutdown` fehlen dann die letzten Transaktionen vor dem Backup. Bei `Clean Shutdown` ist **OK** unbedenklich.

Sind Logdateien beschädigt oder fehlen sie, bietet PowerControls ebenfalls an, ohne Logs weiterzuarbeiten. Das ist für eine erste Sichtung brauchbar. Für einen vollständigen Restore beschaffen Sie besser die fehlenden Logs aus dem Backup. Beim nächsten Öffnen derselben EDB verwendet PowerControls die bereits erzeugten Index-Dateien, der Vorgang ist dann deutlich schneller.

## Ziel verbinden

Nach der Quelle fragt der Wizard auf der Seite **Target Selection** nach dem Ziel. Später fügen Sie Ziele über **File > Open\Create Target** hinzu. Quell- und Zielbereich können mehrere Datenbanken, PST-Dateien und Postfächer gleichzeitig enthalten.

| Ziel | Einsatz |
|---|---|
| **Outlook Data File (\*.pst)** | Export für Benutzer, Revision oder Import in Exchange Online |
| **Single Mailbox (on-premise)** | Restore in ein einzelnes Postfach auf Exchange On-Premises |
| **All Mailboxes (on-premise)** | Restore mehrerer oder aller Postfächer einer Organisation, optional mit Postfacherstellung |
| **Specified Mailboxes (on-premise)** | wie oben, gefiltert nach Hosted Organization oder Address Book Policy |
| **Single Mailbox (Office 365)** | Restore in ein Exchange-Online-Postfach über EWS mit Modern Authentication |

### Exchange On-Premises

Für **Single Mailbox (on-premise)** geben Sie die vollständige E-Mail-Adresse des Zielpostfachs ein und lassen **Connect using Autodiscover** aktiviert. Ab Exchange 2013 ist kein direkter RPC-Zugriff mehr möglich: PowerControls verbindet sich über Outlook Anywhere beziehungsweise MAPI over HTTP mit den Einstellungen aus Autodiscover. Ist der Restore-Rechner nicht Mitglied der Domäne, fragt PowerControls nach Benutzername, Passwort und Domäne eines Kontos mit Vollzugriff auf das Zielpostfach.

Für **All Mailboxes (on-premise)** geben Sie einen Domain Controller und einen beliebigen Exchange-Server ab Version 2013 an oder lassen den Servernamen leer. PowerControls öffnet alle Postfächer, die im AD des angegebenen DCs stehen und für das angemeldete Konto zugänglich sind.

Das Restore-Konto benötigt Full Access auf jedes Zielpostfach. Für einen zeitlich begrenzten Restore können Sie die Berechtigung gezielt setzen und danach wieder entfernen:

```powershell
$recht = @{
  Identity     = "anna.muster@example.com"
  User         = "EXAMPLE\svc-restore"
  AccessRights = "FullAccess"
  AutoMapping  = $false
}
Add-MailboxPermission @recht
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-Identity` | Zielpostfach, in das wiederhergestellt wird |
| `-User` | Konto, unter dem PowerControls läuft |
| `-AccessRights FullAccess` | Vollzugriff auf das Postfach, von PowerControls für Copy-Vorgänge verlangt |
| `-AutoMapping $false` | Verhindert, dass Outlook das Postfach beim Restore-Konto automatisch einbindet |

</details>

Nach dem Restore entfernen Sie die Berechtigung mit `Remove-MailboxPermission` und denselben Werten für `Identity`, `User` und `AccessRights`.

### Exchange Online

Für **Single Mailbox (Office 365)** verbindet sich PowerControls über Exchange Web Services (EWS) mit OAuth. Dafür ist eine App-Registrierung in Entra ID nötig:

1. Im Entra Admin Center unter **App registrations** eine neue Registrierung anlegen, zum Beispiel `PowerControls-Restore`.
2. Unter **Authentication** die Plattform **Mobile and desktop applications** mit der Redirect-URI `https://login.microsoftonline.com/common/oauth2/nativeclient` hinzufügen.
3. Unter **API permissions** die Application Permission **Office 365 Exchange Online > full_access_as_app** hinzufügen und Admin Consent erteilen.
4. Unter **Certificates & secrets** ein Client Secret mit kurzer Laufzeit erstellen.
5. **Application (client) ID**, **Directory (tenant) ID** und den Wert des Secrets in PowerControls unter **Client ID**, **Tenant ID** und **Client Secret** eintragen.

`full_access_as_app` erlaubt der App den Zugriff auf jedes Postfach im Tenant. Schränken Sie den Zugriff mit einer Application Access Policy auf eine Mail-aktivierte Sicherheitsgruppe ein, die nur die Zielpostfächer enthält:

```powershell
$policy = @{
  AppId              = "<Application-ID>"
  PolicyScopeGroupId = "restore-ziele@example.com"
  AccessRight        = "RestrictAccess"
  Description        = "PowerControls Restore"
}
New-ApplicationAccessPolicy @policy
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-AppId` | Application (client) ID der App-Registrierung |
| `-PolicyScopeGroupId` | Mail-aktivierte Sicherheitsgruppe mit den erlaubten Zielpostfächern |
| `-AccessRight RestrictAccess` | Die App darf nur auf Postfächer in dieser Gruppe zugreifen |
| `-Description` | Freitext zur Dokumentation |

</details>

Mit `Test-ApplicationAccessPolicy -AppId "<Application-ID>" -Identity "anna.muster@example.com"` prüfen Sie, ob die Richtlinie greift. Nach Abschluss des Restores löschen Sie das Client Secret oder die ganze App-Registrierung.

#### EWS-Abschaltung in Exchange Online beachten

Microsoft schaltet EWS in Exchange Online stufenweise ab. Seit dem 10. Oktober 2026 gilt in Tenants mit `EWSEnabled = True`: Nur Apps, deren App-ID in `EwsAllowedAppIDs` steht, dürfen EWS verwenden. Fehlt die App-ID von PowerControls in dieser Liste, schlägt die Verbindung zum Office-365-Ziel fehl. Die Änderung wirkt erst nach bis zu 24 Stunden.

Die Liste lässt sich nicht inkrementell ergänzen. `Set-OrganizationConfig` überschreibt den gesamten Wert. Lesen Sie die bestehende Liste deshalb zuerst aus und schreiben Sie sie ergänzt zurück:

```powershell
$aktuell = Get-OrganizationConfig -RetrieveEwsOperationAccessPolicy |
  Select-Object -ExpandProperty EwsAllowedAppIDs
$neu = "<Application-ID>"
$liste = @($aktuell, $neu) | Where-Object { $_ }
Set-OrganizationConfig -EwsAllowedAppIDs ($liste -join ",")
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-RetrieveEwsOperationAccessPolicy` | Liest die EWS-Zugriffseinstellungen inklusive `EwsAllowedAppIDs` mit aus |
| `-ExpandProperty EwsAllowedAppIDs` | Gibt die bestehende Liste als Wert statt als Objekt zurück |
| `Where-Object { $_ }` | Entfernt einen leeren Eintrag, falls noch keine Liste existiert |
| `-EwsAllowedAppIDs` | Setzt die vollständige, kommagetrennte Liste der erlaubten App-IDs |

</details>

Gemäss Message Center MC1485116 soll die Abschaltung bis Anfang Juli 2027 abgeschlossen sein. Danach ist der direkte Restore nach Exchange Online über EWS nicht mehr möglich, sofern Ontrack bis dahin keine Graph-basierte Verbindung nachliefert. Die Alternative bleibt der Weg über PST: in PowerControls in eine PST exportieren und die PST per Purview-Import oder über Outlook in das Postfach einlesen.

## Elemente suchen

Für einzelne Mails ist die Suche schneller als das Durchblättern der Ordner. Markieren Sie im Quellbereich die Datenbank, ein Postfach oder einen Ordner und öffnen Sie die Suche über **Search** in der Symbolleiste. Das Fenster **Find in Source** bietet:

- **Search for words** und **Exclude words** für Suchbegriffe und Ausschlüsse
- **Search In** mit **Message Subject**, **Message Body**, **Attachment File Name(s)** und **Attachment Text**
- Datumsfilter auf **Received**, **Sent** oder **Due** mit **After** und **Before**

Gefundene Elemente kopieren Sie direkt aus der Ergebnisliste. Unter **Edit > Preferences > General** unterdrückt **Check for duplicate messages during search** doppelte Treffer aus mehreren Quellen. Die Option verlangsamt die Suche.

## Wiederherstellen

Der Restore selbst ist ein Copy-Paste vom Quell- in den Zielbereich. Entscheidend ist die Wahl zwischen **Paste** und **Paste Special**:

| Kopiert wird | Einfügen mit | Ergebnis |
|---|---|---|
| Einzelne Mails | **Paste** (`Ctrl+V`) | Mails werden ohne Unterordner in den markierten Zielordner eingefügt |
| Einzelne Mails | **Paste Special** (`Ctrl+Y`) | Die ursprüngliche Ordnerstruktur wird im Ziel nachgebildet |
| Ordner, Postfach, ganze EDB | **Paste** oder Drag-and-drop | Ordnerstruktur bleibt immer erhalten |

Für Treffer aus **Find in Source** verwenden Sie immer **Paste Special**: Die Ergebnisse stammen meist aus verschiedenen Ordnern, und ohne Paste Special geht die Zuordnung verloren.

Typische Szenarien:

- **Gelöschten Ordner eines Benutzers zurückholen:** Ordner im Quellbereich kopieren, im Zielbereich auf das Postfach oder einen Unterordner wie `Restore 2026-11-27` einfügen.
- **Ganzes Postfach wiederherstellen:** Postfach im Quellbereich kopieren und auf das Zielpostfach einfügen. PowerControls legt einen Ordner mit dem Postfachnamen an, falls er nicht existiert.
- **Komplette Datenbank zurückschreiben:** EDB auf den Exchange-Root-Knoten des Ziels ziehen. PowerControls ordnet die Postfächer anhand der Namen zu. Mit **Allow mailbox creation during copy** unter **Edit > Preferences > Target Options** legt es fehlende Postfächer an, ohne diese Option überspringt es sie.

Ein eigener Restore-Ordner im Zielpostfach ist empfehlenswert: Der Benutzer sieht klar, welche Elemente aus dem Backup stammen, und Duplikate im Posteingang bleiben aus. Wird der Ordner `Recoverable Items` kopiert, benennt PowerControls ihn mit dem Datum um, etwa `Recoverable Items (27.11.2026)`.

Nach jedem Kopiervorgang öffnet sich das Fenster **Copy Progress** mit Anzahl Elemente, Fehlern und den Postfächern, zu denen eine Verbindung möglich war oder fehlschlug. Bei einem Restore auf den Server-Root-Knoten verarbeitet PowerControls ein Postfach nach dem anderen. Nur dieses Fenster zeigt, ob alle Postfächer übertragen wurden. Speichern Sie den Report über **Save** für die Dokumentation.

Mit **Check for duplicate messages during copy and export** in den allgemeinen Einstellungen überspringt PowerControls Mails, die im Zielordner bereits vorhanden sind. Das ist bei wiederholten Restores in denselben Ordner sinnvoll, verlangsamt aber den Kopiervorgang.

## In PST exportieren

Für Übergaben ausserhalb von Exchange ist der Export die passende Funktion. Rechtsklick auf Postfach oder Ordner, dann **Export**. Im Fenster **Export Mailbox** beziehungsweise **Export Folder** wählen Sie unter **Format** PST, MSG oder TXT, bei Bedarf mit **Export Subfolders**.

- Aktivieren Sie unter **Edit > Preferences > Target Options** die Option **Create Unicode PSTs**. Ältere PSTs im ANSI-Format sind auf 2 GB begrenzt.
- **Limit maximum PST file size** teilt grosse Exporte automatisch in mehrere Dateien (`_001.pst`, `_002.pst` und so weiter). Das ist für den späteren Import in Exchange Online hilfreich.
- Bei ANSI-PSTs legt PowerControls ab 16 383 Elementen pro Ordner automatisch Folgeordner wie `Inbox (1)` an.

Meldet Outlook beim Öffnen der exportierten PST «File access is denied», fehlt Outlook der Schreibzugriff auf die Datei. Outlook benötigt ihn auch zum reinen Lesen. Häufige Ursachen:

- **Die PST ist in PowerControls noch geöffnet.** PowerControls hält Ziel-PSTs über MAPI geöffnet, bis sie mit **Close Store** geschlossen werden oder das Programm beendet wird.
- **PowerControls lief unter einem anderen Konto.** Die Datei gehört dann diesem Konto. Mit `icacls` prüfen und dem Outlook-Benutzer Änderungsrechte geben.
- **Schreibschutz-Attribut.** Mit `attrib -R <Datei>` entfernen.

```powershell
icacls "D:\Restore\anna.pst"
icacls "D:\Restore" /grant "EXAMPLE\anna.muster:(OI)(CI)M" /T
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `icacls <Datei>` | Zeigt die Berechtigungen der Datei |
| `/grant "<Konto>:…"` | Erteilt dem Konto zusätzliche Rechte |
| `(OI)(CI)` | Vererbung auf Dateien (Object Inherit) und Unterordner (Container Inherit) |
| `M` | Ändern (Modify): lesen, schreiben, löschen |
| `/T` | Auf alle vorhandenen Dateien und Unterordner anwenden |

</details>

Ist das Ziel ohnehin ein Postfach, sparen Sie den Umweg über die PST: Ordner direkt aus der Quelle auf das Zielpostfach kopieren, wie im Abschnitt Wiederherstellen beschrieben.

## Kommandozeile

Wiederkehrende oder grosse Exporte lassen sich über die Kommandozeile ausführen. Das folgende Beispiel kopiert den Posteingang eines Postfachs inklusive Unterordner in eine PST und schreibt einen Report:

```powershell
$argumente = @(
  "-copy", "-r", "-o", "D:\Restore\report.txt"
  "D:\Restore\DB01\DB01.edb"
  "-l", "D:\Restore\DB01\Logs"
  "-t", "D:\Restore\Temp"
  "anna.muster\Inbox"
  "D:\Restore\anna.pst", '$ROOT'
)
powercontrols @argumente
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-copy` | Kopiervorgang (Alternativen: `-export`, `-list`, `-integ`) |
| `-r` | Unterordner einschliessen |
| `-o D:\Restore\report.txt` | Report-Datei; existiert sie, hängt PowerControls an |
| `D:\Restore\DB01\DB01.edb` | Quelldatenbank |
| `-l D:\Restore\DB01\Logs` | Verzeichnis der Logdateien |
| `-t D:\Restore\Temp` | Temporärer Pfad für Index-Dateien |
| `"anna.muster\Inbox"` | Quellpfad: Postfach (Anzeigename oder UPN) und Ordner |
| `D:\Restore\anna.pst` | Ziel: PST-Datei oder `Server;Postfach` |
| `$ROOT` | Zielordner: Wurzel des Ziels |

</details>

Mit `-d:received -a:<mm/dd/yyyy> -b:<mm/dd/yyyy>` schränken Sie den Vorgang auf einen Zeitraum ein, mit `-list` geben Sie vorab die Postfächer der Datenbank aus. Die vollständige Syntax zeigt `powercontrols -h`.

## Nach dem Restore

- Report aus **Copy Progress** oder der Kommandozeile ablegen.
- Full-Access-Berechtigungen und Application Access Policy entfernen, Client Secret löschen.
- Temporäre Index-Dateien und die EDB-Arbeitskopie löschen. Die Kopie enthält sämtliche Mails aller Postfächer der Datenbank.
- Bei Exchange Online die App-ID wieder aus `EwsAllowedAppIDs` entfernen, wenn PowerControls nicht regelmässig im Einsatz ist.

## Quellen

1.  [Ontrack PowerControls for Exchange User Guide (PDF)](https://download.ontrack.com/downloads/OPCEX_Manual.pdf): Handbuch mit Data Wizard, Zieltypen, Copy-Paste-Regeln, PST-Limits, Kommandozeile und EWS-Konfiguration für Office 365.

2.  [Ontrack: PowerControls Current Version](https://ontrack.com/en-us/powercontrols/versions): Aktuelle Version mit Zugang zum Installationspaket über das Trial-Formular.

3.  [Ontrack ExtractWizard Readme (PDF)](https://download.ontrack.com/downloads/OPCEW_Readme.pdf): Unterstützte Backup-Formate und Update-Reihenfolge mit Ontrack Administrative Server.

4.  [Ontrack: PowerControls Version Documentation](https://www.ontrack.com/en-us/blog/powercontrols-version): Versionshistorie; 9.6.4 vom 9. Oktober 2025 mit Exchange Server SE, Outlook 2024 und Windows Server 2025.

5.  [Ontrack: PowerControls for Exchange](https://www.ontrack.com/en-us/powercontrols/exchange): Produktseite mit unterstützten Backup-Formaten für den ExtractWizard und den PowerPacks.

6.  [Microsoft Learn: Recovery databases in Exchange Server](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/recovery-databases): Das Microsoft-eigene Verfahren mit Recovery Database und `New-MailboxRestoreRequest`.

7.  [Microsoft Learn: Eseutil (Technical Reference)](https://learn.microsoft.com/en-us/previous-versions/tn-archive/bb123872(v=exchg.65)): Archivierte Referenz zu `eseutil /mh`, `/r` und `/p`; die Schalter gelten unverändert für aktuelle Versionen.

8.  [Microsoft Learn: Limiting application permissions to specific Exchange Online mailboxes](https://learn.microsoft.com/en-us/graph/auth-limit-mailbox-access): Application Access Policy für Graph und EWS.

9.  [Exchange Team Blog: Introducing EWSAllowedAppIDs](https://techcommunity.microsoft.com/blog/exchange/introducing-ewsallowedappids-preparing-final-phase-ews-retirement/4529471): Allow-Liste für EWS-Apps, Syntax zum Ergänzen bestehender Einträge.

10.  [MC1485116: EWS enforcement update for EWSAllowedAppIDs](https://mc.merill.net/message/MC1485116): Pflicht zur Allow-Liste ab 10. Oktober 2026 und Zeitplan bis Juli 2027.
