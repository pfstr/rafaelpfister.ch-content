---
title: "Proton Drive CLI: Proton Drive aus Skripten und vom Server bedienen"
navTitle: "Proton Drive CLI"
description: "Seit Juni 2026 bietet Proton ein offizielles Kommandozeilenwerkzeug für Proton Drive an. Der Artikel beschreibt Befehle, Anmeldung auf Servern ohne Desktop, Konfliktstrategien für Skripte und die Grenzen gegenüber Rclone."
date: "2026-10-01"
kategorie: "Proton Drive"
timeToRead: "9 Min. Lesezeit"
themen:
  - "proton-drive"
produkte:
  - "proton-drive"
protokolle:
  - "storage"
  - "backup-dr"
related:
  - "proton-drive-linux-status"
  - "rclone-mount-in-docker-container"
slug: "proton-drive-cli"
translationId: "article-97376b998fdaec4c"
url: "https://rafaelpfister.ch/blog/proton-drive-cli"
---

Am 9. Juni 2026 hat Proton die **Proton Drive CLI** veröffentlicht, ein offizielles Kommandozeilenwerkzeug für Windows, macOS und Linux. Es basiert auf demselben SDK wie die offiziellen Drive-Apps, verschlüsselt Ende-zu-Ende und ist als einzelne ausführbare Datei `proton-drive` verfügbar. Der Quellcode liegt im öffentlichen SDK-Repository unter `cli/`.

Die CLI ist für einzelne, zeitlich bestimmte Vorgänge gedacht: Dateien nach einem Build hochladen, einen Ordner per Zeitplan sichern, Freigaben prüfen oder entziehen. Sie synchronisiert nicht im Hintergrund und hängt kein Dateisystem ein. Aktuell ist Version **0.8.0 vom 13. August 2026**; die Versionsnummer zeigt, dass sich Befehle und Optionen noch ändern können (Version 0.8.0 hat die Konfliktstrategien mit einer inkompatiblen Änderung neu benannt).

Die Einordnung in die übrigen Linux-Optionen (Rclone, angekündigter Desktop-Client) steht im Statusartikel [Proton Drive unter Linux](/blog/proton-drive-linux-status).

## Befehlsübersicht

Die Befehle sind in Gruppen organisiert: `proton-drive <gruppe> <befehl> [optionen] [argumente]`. Gruppennamen lassen sich abkürzen, solange sie eindeutig bleiben; für `filesystem` gibt es zusätzlich den Alias `fs`. Ohne Argumente startet eine interaktive Shell. Die vollständige Hilfe liefert `proton-drive help` bzw. `proton-drive <gruppe> <befehl> --help`.

<details class="options-details">
<summary>Optionen im Überblick</summary>

| Gruppe / Befehl | Wirkung |
|---|---|
| `auth login` / `auth logout` | Anmeldung über den Browser; Abmeldung löscht lokale Anmeldedaten und Caches |
| `filesystem list <pfad>` | Inhalt eines Ordners auflisten; `/` zeigt die Wurzelbereiche |
| `filesystem info` / `size` | Metadaten eines Elements bzw. Grösse eines Ordners inkl. Papierkorb-Inhalten |
| `filesystem upload` / `download` | Dateien und Ordner hoch- bzw. herunterladen |
| `filesystem create-folder`, `rename`, `copy`, `move` | Ordner anlegen, umbenennen, kopieren, verschieben |
| `filesystem trash` / `restore` | In den Papierkorb verschieben bzw. wiederherstellen |
| `filesystem delete` / `empty-trash` | Endgültig löschen bzw. `/trash` leeren |
| `sharing status <pfad>` | Mitglieder, offene Einladungen und Link-Einstellungen anzeigen |
| `sharing invite` / `remove` | Personen per E-Mail einladen bzw. Zugriff entziehen |
| `sharing set-url` / `remove-url` | Öffentlichen Link anlegen, ändern oder entfernen |
| `sharing leave` / `report` | Eine mit Ihnen geteilte Freigabe verlassen bzw. als Missbrauch melden |
| `invitation list` / `accept` / `reject` | Eingegangene Einladungen verwalten |
| `album …`, `photo timeline`, `photo upload`, `photo download` | Proton Photos: Alben und Zeitleiste |
| `version` | Versionen von CLI und SDK anzeigen |
| `--json` (`-j`) | Maschinenlesbare JSON-Ausgabe, bei jedem Befehl |
| `--verbose` (`-v`) | Protokollausgabe direkt auf der Konsole |
| `--help` (`-h`) | Hilfe zum jeweiligen Befehl |

</details>

Die Pfade in Proton Drive sind immer POSIX-Pfade, auch unter Windows. Die Wurzel `/` enthält virtuelle Bereiche: `/my-files` (eigene Dateien), `/devices` (gesicherte Computer), `/shared-by-me`, `/shared-with-me`, `/trash` sowie die Photos-Bereiche `/photos`, `/albums`, `/photos-shared-by-me`, `/photos-shared-with-me` und `/photos-trash`.

## Installation unter Linux

Proton stellt die Builds auf einer eigenen Downloadseite bereit, jeweils mit SHA-512-Prüfsumme. Für Linux gibt es fünf Varianten:

| Build | Einsatz |
|---|---|
| `linux/x64` | Standard für aktuelle x86-64-Systeme |
| `linux/x64-baseline` | x86-64 ohne AVX2, z. B. NAS-Geräte und ältere Server-CPUs |
| `linux/arm64` | ARM-Server und Einplatinenrechner mit glibc |
| `linux/x64-musl`, `linux/arm64-musl` | Distributionen mit musl statt glibc, z. B. Alpine Linux und darauf basierende Container-Images |

Bricht der Standard-Build beim Start mit `Illegal instruction` ab, fehlt der CPU die AVX2-Erweiterung; dann ist der `x64-baseline`-Build die richtige Wahl. Die Datei bringt die Bun-Laufzeit eingebettet mit und braucht keine weiteren Abhängigkeiten:

```bash
chmod +x proton-drive
sudo install -m 0755 proton-drive /usr/local/bin/proton-drive
proton-drive version
```

Ohne Administratorrechte genügt es, die Datei nach `~/.local/bin` zu kopieren, sofern dieses Verzeichnis im `PATH` liegt.

## Anmeldung, auch auf Servern ohne Desktop

`auth login` fragt kein Passwort auf der Kommandozeile ab. Die CLI versucht, einen Browser zu öffnen, und gibt zusätzlich die Anmelde-URL aus. Diese URL lässt sich **auf einem anderen Gerät** öffnen; das Terminal wartet, bis die Anmeldung dort abgeschlossen ist. Die Zwei-Faktor-Authentifizierung läuft dabei regulär im Browser. Damit funktioniert die Anmeldung auch per SSH auf einem Server ohne grafische Oberfläche.

```bash
proton-drive auth login
```

Nach erfolgreicher Anmeldung speichert die CLI die Session, nicht das Passwort. Wo sie liegt, bestimmt die Umgebungsvariable `PROTON_DRIVE_CREDENTIALS_STORE`:

| Wert | Speicherort |
|---|---|
| `keychain` (Standard) | Schlüsselspeicher des Betriebssystems: Windows Credential Manager, macOS Keychain, unter Linux libsecret (GNOME Keyring, KWallet) |
| `pass` | GPG-verschlüsselter Eintrag `ch.proton.drive/drive-sdk-cli/auth-session` im Passwortmanager [pass](https://www.passwordstore.org/) |
| `unsafe_file` | Klartextdatei `auth-session.json` im Datenverzeichnis; laut Proton nur für Tests |

Auf einem Server ohne Desktop-Sitzung fehlt in der Regel ein entsperrter libsecret-Schlüsselbund. Für diesen Fall gibt es seit Version 0.6.0 die Option `pass`. Der Benutzer, unter dem die Skripte laufen, braucht dafür einen initialisierten Passwort-Store und einen GPG-Schlüssel, den der `gpg-agent` ohne interaktive Eingabe entsperren kann. Die Session muss bei jedem Aufruf über dieselbe Variable gefunden werden, also auch in Cron-Jobs und systemd-Units gesetzt sein:

```bash
export PROTON_DRIVE_CREDENTIALS_STORE=pass
proton-drive auth login
```

Gegenüber Rclone ist das ein Fortschritt: Passwort und TOTP-Schlüssel liegen nicht auf dem Server, und `auth logout` beendet den Zugang. Die Session hat aber weiterhin den vollen Umfang des Kontos. Eine Beschränkung auf einzelne Ordner oder nur lesenden Zugriff gibt es nicht. Für automatisierte Abläufe bleibt deshalb ein eigenes Proton-Konto die sicherere Variante.

Cache, Anwendungsdaten und Protokolle liegen unter Linux in den XDG-Verzeichnissen (`~/.cache/proton-drive-cli`, `~/.local/share/proton-drive-cli`, `~/.local/state/proton-drive-cli`). Mit `PROTON_DRIVE_CACHE_DIR` lassen sich alle drei in ein einziges Verzeichnis legen, etwa für einen Container mit eingebundenem Volume. Die Protokolle schreibt die CLI standardmässig mit Level `DEBUG`; `PROTON_DRIVE_LOG_LEVEL=WARNING` reduziert die Menge.

## Hochladen und Herunterladen in Skripten

Interaktiv fragt die CLI bei jedem Namenskonflikt nach, was geschehen soll. In Skripten ist das nicht möglich: Mit `--json` ist die interaktive Rückfrage abgeschaltet. Legen Sie deshalb die Konfliktstrategie für Dateien und Ordner immer ausdrücklich fest.

```bash
proton-drive filesystem upload --json \
  --file-conflict-strategy create-new-revision \
  --folder-conflict-strategy merge \
  --skip-thumbnails \
  /srv/export/berichte /my-files/backup
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `--json` (`-j`) | Ergebnis als JSON ausgeben; schaltet interaktive Rückfragen ab |
| `--file-conflict-strategy` (`-f`) | Verhalten, wenn eine Datei gleichen Namens existiert: `create-new-revision` (neue Version der bestehenden Datei), `rename` (Suffix anhängen), `replace` (entfernte Datei in den Papierkorb, lokale hochladen), `skip` |
| `--folder-conflict-strategy` (`-d`) | Verhalten bei bestehendem Ordner: `merge` (Inhalte zusammenführen), `rename`, `replace`, `skip` |
| `--skip-thumbnails` (`-t`) | Keine Vorschaubilder erzeugen; spart Rechenzeit bei Bildern |
| `/srv/export/berichte` | Lokale Quelle; mehrere Quellen sind möglich |
| `/my-files/backup` | Zielordner in Proton Drive (letztes Argument) |

</details>

`create-new-revision` ist für Sicherungen die passende Wahl: Proton Drive behält die früheren Versionen einer Datei, und Dateien mit unverändertem Inhalt überspringt die CLI seit Version 0.7.0 automatisch. Die CLI gleicht aber nicht ab: Lokal gelöschte Dateien bleiben in Proton Drive erhalten. Wer einen Spiegel mit Löschungen braucht, ist weiterhin auf `rclone sync` angewiesen.

Das Herunterladen funktioniert spiegelbildlich. Die Strategien unterscheiden sich, weil hier die lokale Seite überschrieben wird:

```bash
proton-drive filesystem download --json \
  --file-conflict-strategy remove \
  --folder-conflict-strategy merge \
  /my-files/backup/berichte /srv/restore
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `--file-conflict-strategy` (`-f`) | `rename`, `remove` (lokale Datei löschen und entfernte Version herunterladen) oder `skip` |
| `--folder-conflict-strategy` (`-d`) | `merge`, `rename`, `remove` oder `skip` |
| `/my-files/backup/berichte` | Quelle in Proton Drive; mehrere Quellen sind möglich |
| `/srv/restore` | Lokaler Zielordner (letztes Argument) |

</details>

Proton Docs und Proton Sheets überspringt die CLI beim Herunterladen; sie lassen sich derzeit nicht als Dateien exportieren.

Ein regelmässiger Upload lässt sich mit einem systemd-Timer oder Cron einplanen. Die JSON-Ausgabe lässt sich anschliessend mit `jq` auswerten, etwa für eine Meldung an das Monitoring.

## Freigaben verwalten

Für Offboarding oder Audits ist die Freigabeverwaltung oft nützlicher als der Dateitransfer. `sharing status` zeigt für ein Element alle Mitglieder, offenen Einladungen und die Einstellungen eines öffentlichen Links:

```bash
proton-drive sharing status --json /my-files/projekte/kunde-a
```

Eine Einladung mit Leserechten:

```bash
proton-drive sharing invite \
  --user person@example.com \
  --role viewer \
  /my-files/projekte/kunde-a
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `--user` (`-u`) | E-Mail-Adresse der eingeladenen Person; mehrfach angebbar |
| `--role` (`-r`) | Rolle, Standard `viewer`; weitere Rollen gemäss `--help` (z. B. `editor`) |
| `--message` (`-m`) | Nachricht in der Einladungs-Mail; wird im **Klartext** versendet |
| `--include-node-name` (`-n`) | Namen des Elements in die Einladungs-Mail aufnehmen; ebenfalls Klartext |
| `/my-files/projekte/kunde-a` | Freizugebendes Element |

</details>

Ein öffentlicher Link mit Passwort und Ablaufdatum:

```bash
proton-drive sharing set-url \
  --role viewer \
  --password 'Linkpasswort' \
  --expiration 2026-12-31 \
  /my-files/projekte/kunde-a/bericht.pdf
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `--role` | `viewer` (Standard) oder `editor` |
| `--password` | Eigenes Passwort für den Link |
| `--expiration` | Ablaufdatum im ISO-Format (`JJJJ-MM-TT`) |
| `/my-files/…/bericht.pdf` | Element, für das der Link angelegt oder geändert wird |

</details>

Ein auf der Kommandozeile übergebenes Passwort steht in der Shell-History und ist während der Ausführung in der Prozessliste sichtbar. In Skripten sollte es deshalb aus einer Variablen oder einem Secret-Speicher kommen. `sharing remove-url` entfernt den Link wieder, ohne direkte Mitglieder zu betreffen; `sharing remove --user …` entzieht einzelnen Personen den Zugriff.

## In Arbeit: Takeout

Im SDK-Repository ist seit dem 10. September 2026 ein weiterer Befehl `takeout run` enthalten. Er exportiert eine Offline-Kopie des Kontos in einen lokalen Ordner, wahlweise mit `--include my-files`, `devices`, `photos` und `revisions` (alle früheren Dateiversionen). Zu jedem Ordner schreibt er eine `manifest.json`, die den Export beschreibt; am Konto ändert er nichts. In der veröffentlichten Version 0.8.0 ist der Befehl noch nicht enthalten. Sobald er erscheint, ist er der naheliegende Weg für eine vollständige lokale Sicherung des Proton-Drive-Inhalts.

## Grenzen im Vergleich mit Rclone

| Anforderung | Proton Drive CLI 0.8.0 | Rclone (`protondrive`-Backend) |
|---|---|---|
| Offiziell unterstützt | Ja, von Proton, Open Source | Nein, Reverse Engineering, Beta |
| Anmeldung | Browser, auch auf anderem Gerät; Session im Schlüsselspeicher oder `pass` | Passwort und TOTP-Schlüssel in der Konfigurationsdatei |
| Hochladen, Herunterladen | Ja, mit Konfliktstrategien und Versionierung | Ja |
| Spiegel mit Löschungen (`sync`) | Nein | Ja |
| Dateisystem einhängen (FUSE) | Nein | Ja |
| Freigaben, Einladungen, Links | Ja | Nein |
| Proton Photos | Ja | Nein |
| Zugang mit begrenztem Umfang | Nein | Nein |

Für Sicherungen, Build-Artefakte und Freigabeverwaltung ist die CLI die bessere Wahl, weil sie offiziell unterstützt wird und ohne gespeichertes Passwort auskommt. Für einen Mount, wie ihn etwa eine [Paperless-Dokumentablage](/blog/paperless-dokumente-clouddienst-auslagern) braucht, und für Spiegelungen mit Löschungen bleibt Rclone vorerst notwendig. Die grösste Lücke bleibt in beiden Fällen dieselbe: Proton bietet keinen Maschinenzugang, der sich auf einzelne Ordner oder Lesezugriff beschränken lässt.

## Quellen

1.  [Proton: Introducing Proton Drive CLI](https://proton.me/blog/proton-drive-cli): die Ankündigung vom 9. Juni 2026 mit Einsatzzweck und JSON-Ausgabe.

2.  [Proton Support: Using Proton Drive CLI](https://proton.me/support/drive-cli): Anleitung für Download, Anmeldung und Grundbefehle.

3.  [Proton Drive CLI: Downloads](https://proton.me/download/drive/cli/index.html): aktuelle Version 0.8.0, alle Plattform-Builds mit SHA-512-Prüfsummen.

4.  [ProtonDriveApps/sdk: cli/README.md](https://github.com/ProtonDriveApps/sdk/blob/main/cli/README.md): Umgebungsvariablen, Speicherorte, Credential-Stores und der Hinweis zum `x64-baseline`-Build.

5.  [ProtonDriveApps/sdk: cli/CHANGELOG.md](https://github.com/ProtonDriveApps/sdk/blob/main/cli/CHANGELOG.md): Versionsverlauf von 0.4.2 bis 0.8.0, u. a. `pass`-Unterstützung (0.6.0) und das Überspringen unveränderter Dateien (0.7.0).

6.  [ProtonDriveApps/sdk: cli/src/commands](https://github.com/ProtonDriveApps/sdk/tree/main/cli/src/commands): Quellcode der Befehle mit Optionen, Konfliktstrategien und dem noch unveröffentlichten Takeout-Befehl.

7.  [Proton for Business: Proton Drive CLI](https://proton.me/business/drive/cli): Protons Einsatzszenarien für Unternehmen, etwa Freigaben beim Austritt von Mitarbeitenden entziehen.

8.  [Rclone: Proton Drive](https://rclone.org/protondrive/): das Community-Backend mit Mount- und Sync-Funktion als Vergleich.
