---
title: "Outlook New nach Windows-Update zurückgesetzt: Konten weg, Ursache im Log"
navTitle: "Outlook-Reset"
description: "Das neue Outlook für Windows startet nach einem Update-Neustart wie frisch installiert, alle Konten und Einstellungen fehlen. Im Olk-Log ist die Ursache nachvollziehbar: Eine serverseitig aktivierte Prüfung (Flight nh-userDataSentinel) vermisst eine Markierungsdatei und löscht die lokalen Daten ohne Rückfrage. Der Artikel zeigt die Log-Einträge, wie Sie den Fall auf eigenen Geräten nachweisen und was nach dem Reset zu tun ist."
date: "2026-11-11"
kategorie: "Outlook"
timeToRead: "6 Min. Lesezeit"
themen:
  - "outlook"
  - "windows-client"
produkte:
  - "outlook"
  - "windows-client"
protokolle:
  - "troubleshooting"
slug: "outlook-new-reset-nach-update"
translationId: "article-c1340c0bc1862647"
url: "https://rafaelpfister.ch/blog/outlook-new-reset-nach-update"
---
# Outlook New nach Windows-Update zurückgesetzt: Konten weg, Ursache im Log

Nach dem Neustart für die Windows-Updates vom November 2026 startete das neue Outlook für Windows auf einem meiner Geräte wie nach einer Neuinstallation. Das Geschäftskonto, das als Hauptkonto eingebunden war, fehlte ebenso wie die übrigen Konten, die Signaturen und die persönlichen Einstellungen. Eine Fehlermeldung gab es nicht, Outlook zeigte direkt den Assistenten zum Hinzufügen eines Kontos.

Die Ursache steht im lokalen Protokoll von Outlook: Die App hat ihre lokalen Daten selbst gelöscht, weil eine neu aktivierte Prüfung den Zustand als "Reset erforderlich" bewertet hat. Hinweise auf einen Eingriff durch Windows Update, einen Benutzer oder einen Angreifer gibt es keine.

## Ablauf

| Zeit (Lokalzeit) | Ereignis | Quelle |
|---|---|---|
| 18:26 | Letzter Eintrag der laufenden Outlook-Sitzung, kein geordnetes Beenden | `Olk\logs\olk_*.log` |
| 18:27 und 18:28 | Zwei Neustarts durch Windows Update (`MoNotificationUx.exe`, danach `TrustedInstaller.exe`) | System-Log, Ereignis 1074 |
| 18:41:55 | Outlook startet, Prüfung `nh-userDataSentinel` ist aktiv und meldet `ResetRequired` | neues `olk_*.log` |
| 18:41:56 | Outlook löscht seine lokalen Daten und startet als Neuinstallation | neues `olk_*.log` |

Die Zeitstempel im Olk-Log und in den Dateinamen der Logs sind in UTC. In der Tabelle sind sie in Lokalzeit (MEZ) umgerechnet.

## Die Einträge im Log

Die ersten Zeilen der neuen Sitzung zeigen die Prüfung und das Ergebnis:

```text
Checking flight enabled status  flightName nh-userDataSentinel  isEnabled true
Reset sentinel check completed.  installState ResetRequired
  settingsFound true  resetSentinelFlagFound true
Reset sentinel check triggered user data reset.
Prompted for clear local state  clearLocalStateName Silent
```

Danach folgen die Löschvorgänge, jeweils mit `succeeded true`:

| Pfad unter `%LOCALAPPDATA%\Microsoft\Olk` | Inhalt |
|---|---|
| `EBWebView` | WebView2-Profil mit Anmeldestatus, Kontenliste und Offline-Daten |
| `UserSettings.json` | Einstellungen der App |
| `Attachments` | lokal zwischengespeicherte Anhänge |
| `pst_index_v2` | Index eingebundener PST-Dateien |
| `cache\Notification`, `cache\*.db` | Benachrichtigungs- und Hintergrund-Cache |
| `updated.txt`, `xpdApi.log`, `Feedback` | Status- und Hilfsdateien |

Einige Zeilen später steht `Last update date file missing, creating it. Barring clearing local state, this is likely a fresh install`. Ab diesem Punkt verhält sich Outlook wie eine Neuinstallation.

## Warum die Prüfung angeschlagen hat

`nh-userDataSentinel` ist ein Flight, also eine Funktion, die Microsoft serverseitig für einzelne Installationen einschaltet. In allen älteren Logs auf dem Gerät steht für diesen Flight `Flight status not set; using default (false)`, zuletzt um 13:23 UTC am selben Tag. Beim Start um 17:41 UTC war er erstmals aktiv.

Die Prüfung sucht im Paketordner der App eine Markierungsdatei:

```text
%LOCALAPPDATA%\Packages\Microsoft.OutlookForWindows_8wekyb3d8bbwe\LocalState\OlkSentinel
```

Findet Outlook Benutzereinstellungen im Olk-Ordner (`settingsFound true`), aber keine passende Markierung im Paketordner, wertet es das als Hinweis, dass die App zurückgesetzt oder neu installiert wurde, und löscht die Benutzerdaten. Auf dem betroffenen Gerät wurde `OlkSentinel` um 18:41:56 neu angelegt, die übrigen Dateien in `LocalState` stammen vom September 2026. Der Paketordner selbst wurde also nicht zurückgesetzt, die Markierungsdatei gab es vorher schlicht nicht, weil die Prüfung neu war. Beim ersten Start mit aktivem Flight war das Ergebnis deshalb ein Reset, obwohl keiner nötig war.

Der Neustart durch Windows Update ist nach den Logs nur der Zeitpunkt, nicht die Ursache: Der Flight wird beim Start ausgewertet, und der erste Start nach dem Update war der erste mit aktivem Flight. Die Version der App (1.2026.1026.100) war vor und nach dem Neustart dieselbe.

## Den Fall auf eigenen Geräten nachweisen

Die folgende Abfrage durchsucht alle Olk-Logs nach den Einträgen der Prüfung:

```powershell
$logs = "$env:LOCALAPPDATA\Microsoft\Olk\logs"
Get-ChildItem $logs -Filter 'olk_*.log' |
  Select-String -Pattern 'Reset sentinel|Delete directory tree' |
  Select-Object Filename, LineNumber, Line
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-Filter 'olk_*.log'` | nur die Sitzungslogs der App, ohne ZIP-Archive und Push-Logs |
| `-Pattern` | regulärer Ausdruck; trifft die Prüfung und jeden Löschvorgang |
| `Select-Object Filename, LineNumber, Line` | zeigt pro Treffer Logdatei, Zeilennummer und Text |

</details>

Ältere Sitzungen packt Outlook in ZIP-Archive im selben Ordner. Liegt der Reset einige Tage zurück, steht der Eintrag möglicherweise nur noch im Archiv und muss vorher entpackt werden.

Als zweites Indiz eignen sich die Erstellungszeiten: Haben `EBWebView` und `UserSettings.json` im Olk-Ordner sowie `OlkSentinel` im Paketordner denselben Zeitstempel, ist der Reset zu diesem Zeitpunkt erfolgt.

```powershell
$olk = "$env:LOCALAPPDATA\Microsoft\Olk"
$pkg = "$env:LOCALAPPDATA\Packages\Microsoft.OutlookForWindows_8wekyb3d8bbwe"
Get-Item "$olk\EBWebView", "$olk\UserSettings.json", "$pkg\LocalState\OlkSentinel" |
  Select-Object FullName, CreationTime
```

## Was verloren ist und was nicht

Gelöscht wurden nur lokale Daten der App. Postfächer, Kalender und Kontakte liegen bei Microsoft 365 und Outlook.com auf dem Server und erscheinen nach dem erneuten Anmelden wieder. Die Anmeldung bei Windows (Web Account Manager) bleibt bestehen; Microsoft-Konten, die in Windows hinterlegt sind, bietet Outlook beim Hinzufügen direkt an.

Neu einrichten müssen Sie:

- alle Konten, in der gewünschten Reihenfolge (das zuerst hinzugefügte Konto ist das Hauptkonto, siehe unten)
- lokal gespeicherte Signaturen, falls sie nicht über die Cloud-Synchronisation des Kontos zurückkommen
- eingebundene PST-Dateien
- Einstellungen der App wie Ansicht, Benachrichtigungen und Startverhalten

Beim Hauptkonto ist die Reihenfolge entscheidend, weil das neue Outlook einige Funktionen nur im primären Konto anbietet, etwa S/MIME (siehe [S/MIME-Signatur im sekundären Konto](/blog/outlook-smime-sekundaeres-konto-anhaenge-fehlen)). Fügen Sie das Konto, das Hauptkonto sein soll, deshalb zuerst hinzu. Hat Outlook nach dem Reset bereits automatisch ein anderes Konto übernommen, entfernen Sie dieses, fügen das gewünschte Hauptkonto hinzu und binden die übrigen Konten danach wieder ein.

## Für Administratoren

Der Reset lässt sich nach dem Auftreten nicht rückgängig machen; eine Sicherung des Ordners `%LOCALAPPDATA%\Microsoft\Olk` vor dem Start ist die einzige Möglichkeit, den alten Stand zurückzuholen. In Umgebungen mit Profil-Containern (FSLogix) und nicht persistenten VDI-Sitzungen sollten Sie prüfen, ob sowohl der Olk-Ordner als auch der Paketordner `Microsoft.OutlookForWindows_8wekyb3d8bbwe` im Profil erhalten bleiben. Wird nur einer der beiden Ordner mitgenommen, entspricht das genau dem Zustand, den die Prüfung als Neuinstallation wertet.

Meldungen von Anwendern, das neue Outlook habe "alle Konten vergessen", lassen sich mit der Log-Abfrage oben in wenigen Minuten einordnen. Steht dort `Reset sentinel check triggered user data reset`, liegt kein Kontoproblem und kein Sicherheitsvorfall vor.

## Quellen

1.  [Microsoft Learn: Deploy the new Outlook for Windows on VDI](https://learn.microsoft.com/en-us/microsoft-365-apps/outlook/get-started/virtualized-desktop-infrastructure): Speicherorte der lokalen Daten des neuen Outlook und Hinweise zu Profil-Containern.

2.  [Microsoft Learn: Control installing and using the new Outlook](https://learn.microsoft.com/en-us/microsoft-365-apps/outlook/get-started/control-install): Verteilung und Steuerung der App in verwalteten Umgebungen.

3.  [Microsoft Q&A: Outlook (new) won't open, makes me attempt to sign in](https://learn.microsoft.com/en-us/answers/questions/5863503/outlook-new-wont-open-makes-me-attempt-to-sign-in): Anwenderberichte zu verlorenen Konten nach dem Zurücksetzen der lokalen Daten.
