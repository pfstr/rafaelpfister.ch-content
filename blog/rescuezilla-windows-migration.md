---
title: "Rescuezilla: Windows auf eine neue SSD migrieren, mit Vorstellung des Tools"
navTitle: "Rescuezilla-Migration"
description: "Rescuezilla ist eine kostenlose, Clonezilla-kompatible Imaging-Lösung mit grafischer Oberfläche. Der Artikel stellt das Tool vor und zeigt den kompletten Umzug einer Windows-Installation auf eine neue SSD: Vorbereitung in Windows, Boot-Stick, Backup, Prüfung, Restore und Nacharbeiten."
date: "2026-09-29"
kategorie: "PC & Hardware"
timeToRead: "10 min to read"
themen:
  - "pc-hardware"
produkte:
  - "windows-client"
protokolle:
  - "backup-dr"
  - "migration"
slug: "rescuezilla-windows-migration"
url: "https://rafaelpfister.ch/blog/rescuezilla-windows-migration"
translationId: "article-d9438c5774d0167c"
---

Wer eine Windows-Installation auf eine neue SSD oder in einen neuen Rechner übernehmen will, ohne Windows neu zu installieren, braucht ein Werkzeug, das den ganzen Datenträger mit allen Partitionen sichert und wiederherstellt. Rescuezilla ist ein solches Werkzeug: kostenlos, quelloffen und mit einer grafischen Oberfläche, die auch ohne Linux-Kenntnisse bedienbar ist. Dieser Artikel stellt das Tool vor und beschreibt die Migration auf eine gleich grosse oder grössere SSD. Für eine kleinere Ziel-SSD gibt es eine eigene Anleitung: [Windows mit Rescuezilla auf eine kleinere SSD umziehen](/blog/windows-kleinere-ssd-rescuezilla).

## Was Rescuezilla ist

Rescuezilla ist ein Live-System auf Basis von Ubuntu, das von einem USB-Stick startet. Es läuft unabhängig vom installierten Betriebssystem und sichert deshalb auch Volumes, die Windows im laufenden Betrieb sperrt. Das Projekt entstand 2019 als Fork von Redo Backup and Recovery, das zu diesem Zeitpunkt seit sieben Jahren nicht mehr gepflegt wurde. Seit Version 2.0 (2020) schreibt Rescuezilla Images im Format von Clonezilla: Ein mit Rescuezilla erstelltes Backup lässt sich mit Clonezilla zurückspielen und umgekehrt. Die Lizenz ist GPL-3.0. Aktuell ist Version 2.6.2 vom Mai 2026 auf Basis von Ubuntu 26.04 LTS mit Partclone 0.3.47.

Die eigentliche Arbeit erledigt Partclone. Es kennt die gängigen Dateisysteme (NTFS, FAT, ext4 und weitere) und liest nur die belegten Blöcke. Eine 1-TB-Partition mit 350 GB Daten ergibt deshalb ein Image von rund 215 GB (gzip-komprimiert), nicht von 1 TB. Partitionen ohne erkanntes Dateisystem, etwa die Microsoft-reservierte Partition, sichert Rescuezilla blockweise mit `dd`; diese Dateien tragen im Image die Endung `.dd-ptcl-img`.

<details class="options-details">
<summary>Funktionen im Überblick</summary>

| Funktion | Zweck |
|---|---|
| Backup | sichert ausgewählte Partitionen einer Disk samt Partitionstabelle als Image auf ein lokales Laufwerk oder eine Netzwerkfreigabe (SMB, SSH) |
| Restore | spielt ein Image auf eine Disk zurück, wahlweise mit Überschreiben der Partitionstabelle |
| Verify Image | prüft, ob ein vorhandenes Image vollständig und lesbar ist |
| Clone | kopiert eine Disk direkt auf eine zweite, ohne Zwischenspeicher |
| Image Explorer (beta) | bindet ein Image schreibgeschützt ein, um einzelne Dateien herauszuholen |
| VM-Images | liest neben Clonezilla-Images auch VDI, VMDK, VHDX, QCOW2 und Raw-Images |
| Zusatzwerkzeuge | GParted, Dateimanager, Webbrowser und Tools zur Wiederherstellung gelöschter Dateien auf dem Live-Desktop |
| CLI | experimentelle Kommandozeile (seit 2.5) für Backup, Verify, Restore und Clone |

</details>

Eine Einschränkung ist für Migrationen wichtig: Rescuezilla verkleinert keine Partitionen. Die Zieldisk muss mindestens bis zum Ende der letzten gesicherten Partition reichen. Ist sie kleiner, ist eine Vorarbeit mit GParted nötig, die in der oben verlinkten Anleitung beschrieben ist.

## Image oder Klon

Rescuezilla bietet zwei Wege für eine Migration. Beim **Klonen** sind Quell- und Zieldisk gleichzeitig angeschlossen, und Rescuezilla kopiert direkt. Das spart Zeit und einen dritten Datenträger, setzt aber voraus, dass beide Disks gleichzeitig im Rechner oder an einem Adapter hängen. Beim **Imaging** entsteht zuerst ein Image auf einem externen Laufwerk, das danach auf die neue Disk zurückgespielt wird. Das dauert länger, hat aber einen Vorteil: Das Image bleibt als vollständige Sicherung des alten Stands erhalten, auch wenn beim Restore etwas schiefgeht. Für eine Migration ist das Image deshalb die sicherere Wahl, und diese Variante beschreibt der Artikel.

## Voraussetzungen

1.  **USB-Stick** für Rescuezilla. Sein Inhalt wird beim Schreiben des Boot-Images gelöscht.

2.  **Externes Laufwerk** für das Image, mit freiem Speicherplatz etwa in der Grösse der belegten Daten. exFAT und NTFS funktionieren beide.

3.  **Zieldisk**, mindestens so gross wie die Quelldisk. Ist sie kleiner, zuerst die Anleitung für kleinere SSDs durcharbeiten.

4.  **BitLocker-Wiederherstellungsschlüssel**, falls C: verschlüsselt ist. Abrufbar unter aka.ms/myrecoverykey oder im Entra-Portal bei verwalteten Geräten.

## Schritt 1: Windows vorbereiten

Drei Punkte prüfen Sie vor dem Backup in einer PowerShell mit Administratorrechten.

**BitLocker.** Partclone kann ein BitLocker-Volume nicht als NTFS lesen. Rescuezilla sichert es dann wie jede Partition ohne erkanntes Dateisystem blockweise, das Image wird so gross wie die ganze Partition. Für eine Migration empfiehlt es sich, BitLocker vorher auszuschalten und nach dem Umzug wieder einzuschalten:

```powershell
manage-bde -status C:
manage-bde -off C:
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-status` | zeigt Verschlüsselungsgrad und Schutzstatus des Volumes |
| `-off` | entschlüsselt das Volume vollständig; läuft im Hintergrund weiter |
| `C:` | Positionsargument: das betroffene Volume |

</details>

Warten Sie, bis `manage-bde -status C:` den Wert `Fully Decrypted` zeigt. Bei Geräten, die über Intune verwaltet werden, kann eine Richtlinie BitLocker wieder einschalten; prüfen Sie den Status deshalb direkt vor dem Neustart noch einmal.

**Ruhezustand und Schnellstart.** Mit aktivem Schnellstart fährt Windows nicht vollständig herunter, und NTFS gilt als noch in Gebrauch. Partclone bricht dann ab. Ein Befehl deaktiviert beides:

```powershell
powercfg /h off
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `/h off` | Kurzform von `/hibernate off`: deaktiviert Ruhezustand und Schnellstart, `hiberfil.sys` wird entfernt |

</details>

**Zustand des Dateisystems.** Ist das Dirty-Bit gesetzt, bricht Partclone mit der Meldung ab, das Volume sei „scheduled for a check or it was shutdown uncleanly“. Vorab prüfen:

```powershell
fsutil dirty query C:
```

Meldet der Befehl `is Dirty`, planen Sie mit `chkdsk C: /f` eine Prüfung für den nächsten Start ein, starten neu und prüfen erneut.

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `dirty query C:` | fsutil: fragt ab, ob das Dirty-Bit des Volumes gesetzt ist |
| `/f` | chkdsk: behebt Fehler; auf dem Systemvolume wird die Prüfung für den nächsten Start eingeplant |

</details>

## Schritt 2: Boot-Stick erstellen und starten

Laden Sie das ISO-Image von der GitHub-Release-Seite des Projekts. Die Standardvariante trägt den Ubuntu-Codenamen im Dateinamen, bei 2.6.2 `rescuezilla-2.6.2-64bit.resolute.iso`. Schreiben Sie es mit einem Image-Writer wie balenaEtcher auf den USB-Stick; die Projektseite empfiehlt dieses Programm für Windows, macOS und Linux.

Den Start vom Stick lenkt Windows selbst ein:

```powershell
shutdown /r /o /t 0
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `/r` | Neustart statt Herunterfahren |
| `/o` | startet in die erweiterten Startoptionen; nur zusammen mit `/r` |
| `/t 0` | Wartezeit in Sekunden bis zur Ausführung |

</details>

In den erweiterten Startoptionen wählen Sie „Gerät verwenden“ und den USB-Stick. Alternativ öffnet sich beim Einschalten das Bootmenü der Firmware (je nach Hersteller F8, F11 oder F12). Rescuezilla startet mit aktiviertem Secure Boot; es muss dafür nicht abgeschaltet werden. Bleibt der Bildschirm nach der Sprachauswahl schwarz, hilft der Eintrag „Graphical Fallback Mode“ im Bootmenü des Sticks.

## Schritt 3: Backup erstellen

Auf dem Desktop startet Rescuezilla automatisch. Der Assistent führt in nummerierten Schritten durch das Backup:

1.  **Backup** wählen.

2.  *Step 1: Select Drive To Backup*: die Systemdisk wählen. Modell und Grösse helfen bei der Unterscheidung, die Gerätenamen (`nvme0n1`, `sda`) können sich zwischen zwei Starts unterscheiden.

3.  *Step 2: Select Partitions to Save*: alle Partitionen angehakt lassen. Für einen bootfähigen Restore braucht es EFI-Systempartition, Microsoft-reservierte Partition, C: und die Wiederherstellungspartition.

4.  *Step 3: Select Destination Drive*: das externe Laufwerk (Local) oder eine Netzwerkfreigabe (Network).

5.  *Step 4: Select Destination Folder*: den Zielordner auf dem Laufwerk. Rescuezilla legt darin einen Unterordner mit Zeitstempel an.

6.  *Step 5: Name Your Backup*: optional eine Beschreibung, sie erscheint später in der Imageliste.

7.  *Step 6: Customize Compression Settings*: Die Voreinstellung gzip ist für die meisten Fälle passend.

8.  *Step 7: Confirm Backup Configuration*: Angaben prüfen und starten.

Für 350 GB Daten auf einer NVMe-SSD dauerte das Backup auf eine externe USB-SSD rund 20 Minuten. Am Ende meldet Rescuezilla für jede Partition, ob sie erfolgreich gesichert wurde. Steht dort bei einer Partition ein Fehler, ist das Image unvollständig, auch wenn der Ordner existiert.

## Schritt 4: Image prüfen

Vor dem Restore prüfen Sie das Image. **Verify Image** im Hauptmenü liest alle Teile und prüft, ob sie vollständig und lesbar sind. In der Imageliste zeigt die Spalte **Partitions** die gesicherten Partitionen mit Grösse; ein gelbes Warndreieck markiert Images mit fehlenden Partitionen.

Der Imageordner lässt sich auch unter Windows ansehen. Die wichtigsten Dateien:

| Datei | Inhalt |
|---|---|
| `nvme0n1-pt.parted` | Partitionstabelle in Textform, mit Start und Ende jeder Partition in Sektoren |
| `nvme0n1-gpt-1st`, `nvme0n1-gpt-2nd` | binäre Kopien der primären und der Backup-GPT |
| `nvme0n1p3.ntfs-ptcl-img.gz.aa`, `.ab`, … | Partclone-Image von C:, gzip-komprimiert, in Teile zu 4 GB zerlegt |
| `nvme0n1p2.dd-ptcl-img.gz.aa` | blockweise Kopie einer Partition ohne erkanntes Dateisystem |
| `clonezilla-img` | Protokoll des Backups mit Ergebnis je Partition |
| `Info-*.txt` | Hardware- und SMART-Angaben des Quellsystems zum Zeitpunkt des Backups |

Aus `nvme0n1-pt.parted` lässt sich ablesen, ob das Image auf die Zieldisk passt: Endsektor der letzten Partition plus eins, mal 512 Bytes, muss kleiner sein als die Grösse der Zieldisk in Bytes.

## Schritt 5: Restore auf die neue SSD

Bauen Sie die neue SSD ein und starten Sie wieder Rescuezilla.

1.  **Restore** wählen.

2.  *Step 1: Select Image Location*: das externe Laufwerk mit dem Image.

3.  *Step 2: Select Backup Image*: das Image wählen, anhand von Datum und Partitionsgrössen.

4.  *Step 3: Select Drive To Restore*: die neue SSD. Prüfen Sie Modell und Grösse zweimal: Alle Daten auf diesem Laufwerk werden überschrieben.

5.  *Step 4: Select Partitions to Restore*: alle Partitionen angehakt lassen, ebenso **Overwrite partition table**.

6.  *Step 5: Confirm Restore Configuration*: prüfen und starten.

Der Restore dauert etwa so lange wie das Backup. Danach fahren Sie herunter und entfernen den Stick.

## Schritt 6: Vom neuen Laufwerk starten

Am einfachsten klemmen Sie die alte Disk vor dem ersten Start ab. Sind beide Disks angeschlossen, tragen sie dieselben Partitions- und Volume-IDs, und die Firmware wählt unter Umständen die alte zum Start. Wird die alte Disk weiterverwendet, formatieren Sie sie erst, nachdem das neue System geprüft ist.

Ist die neue SSD grösser als die alte, liegt der zusätzliche Speicherplatz als nicht zugeordneter Bereich am Ende. C: lässt sich in der Datenträgerverwaltung aber nicht direkt erweitern, weil die Wiederherstellungspartition zwischen C: und dem freien Bereich liegt. Mit GParted auf dem Rescuezilla-Stick verschieben Sie zuerst die Wiederherstellungspartition ans Ende (Resize/Move, **Free space following** auf 0) und vergrössern danach C: auf den gesamten freien Bereich. Windows RE findet seine Partition danach wieder, weil sie ihre Partitionsnummer behält; `reagentc /info` zeigt den Status.

Wechselt mit der Migration auch der Rechner, startet Windows in der Regel ohne Anpassungen und installiert fehlende Treiber nach. Die Aktivierung erfolgt über die digitale Lizenz, bei einem Mainboard-Wechsel eventuell erst nach Anmeldung mit dem Microsoft-Konto über die Aktivierungs-Problembehandlung.

## Nacharbeiten

Auf dem neuen Laufwerk schalten Sie wieder ein, was für die Migration deaktiviert wurde. Den Ruhezustand aktiviert `powercfg /h on`. BitLocker aktivieren Sie über Einstellungen → Datenschutz und Sicherheit → Geräteverschlüsselung oder mit `manage-bde -on C:`; kontrollieren Sie danach mit `manage-bde -protectors -get C:`, dass ein Wiederherstellungsschlüssel vorhanden und gesichert ist.

Das Image auf dem externen Laufwerk behalten Sie, bis Windows auf dem neuen Laufwerk einige Tage ohne Auffälligkeiten gelaufen ist und alle Daten geprüft sind. Danach kann es gelöscht oder als Sicherung des Auslieferungszustands archiviert werden.

## Quellen

1.  [Rescuezilla](https://rescuezilla.com/): Projektseite mit Funktionsübersicht, Download und FAQ.

2.  [Rescuezilla auf GitHub](https://github.com/rescuezilla/rescuezilla): Quellcode, Projektgeschichte und Funktionsliste im README.

3.  [Rescuezilla Releases](https://github.com/rescuezilla/rescuezilla/releases/latest): aktuelle Version mit Release Notes, Kompatibilitätsliste und ISO-Varianten.

4.  [Rescuezilla Changelog](https://raw.githubusercontent.com/rescuezilla/rescuezilla/master/CHANGELOG.md): Einführung von CLI, Verify Image und Clone sowie die Aktualisierung des Secure-Boot-Shims.

5.  [Rescuezilla Wiki: Restoring to a smaller disk](https://github.com/rescuezilla/rescuezilla/wiki/HOWTO:-Restoring-to-a-smaller-disk.-Eg,-1000GB-HDD-to-500GB-SSD): offizieller Ablauf für Zieldisks, die kleiner als das Original sind.

6.  [Partclone](https://partclone.org/): Werkzeug, das Rescuezilla und Clonezilla für die dateisystembewusste Sicherung verwenden.

7.  [GParted Manual](https://gparted.org/display-doc.php?name=help-manual): Verschieben und Vergrössern von Partitionen.

8.  [Microsoft Learn: manage-bde status](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-status): Statusabfrage von BitLocker.

9.  [Microsoft Learn: manage-bde off](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-off): Entschlüsseln eines Volumes.

10.  [Microsoft Learn: Powercfg command-line options](https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/powercfg-command-line-options): Option `/hibernate` und ihr Einfluss auf den Schnellstart.

11.  [Microsoft Learn: fsutil dirty](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/fsutil-dirty): Abfrage des Dirty-Bits.

12.  [Microsoft Learn: chkdsk](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/chkdsk): Prüfung und Reparatur von NTFS-Volumes.

13.  [Microsoft Learn: shutdown](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/shutdown): Parameter `/o` für den Start in die erweiterten Startoptionen.

14.  [Microsoft Support: Reactivating Windows after a hardware change](https://support.microsoft.com/en-us/windows/reactivating-windows-after-a-hardware-change-2c0e962a-f04c-145b-6ead-fb3fc72b6665): Aktivierung über die digitale Lizenz nach einem Mainboard-Wechsel.
