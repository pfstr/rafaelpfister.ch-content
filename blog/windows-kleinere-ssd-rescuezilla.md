---
title: "Windows mit Rescuezilla auf eine kleinere SSD umziehen: von 1 TB auf 512 GB"
navTitle: "Umzug kleinere SSD"
description: "Rescuezilla stellt ein Image nur auf eine Disk zurück, die mindestens bis zum Ende der letzten Partition reicht. Die Anleitung zeigt am Beispiel einer 1-TB-NVMe, wie Sie C: mit GParted verkleinern, die Recovery-Partition verschieben, das Dirty-Flag entfernen und das neue Image auf eine 512-GB-SSD zurückspielen."
date: "2026-09-29"
kategorie: "PC & Hardware"
timeToRead: "9 min to read"
themen:
  - "pc-hardware"
produkte:
  - "windows-client"
protokolle:
  - "backup-dr"
  - "migration"
slug: "windows-kleinere-ssd-rescuezilla"
url: "https://rafaelpfister.ch/blog/windows-kleinere-ssd-rescuezilla"
translationId: "article-6947213279383f27"
---

Rescuezilla ist ein kostenloses Live-System zum Sichern und Wiederherstellen ganzer Datenträger, kompatibel zum Image-Format von Clonezilla (Vorstellung des Tools und Grundablauf: [Rescuezilla: Windows auf eine neue SSD migrieren](/blog/rescuezilla-windows-migration)). Solange die Zieldisk gleich gross oder grösser ist, reichen Backup und Restore. Ist sie kleiner, bricht der Restore ab, obwohl die belegten Daten darauf Platz hätten. Dieser Artikel dokumentiert den Umzug eines Windows-11-Systems von einer 1-TB-NVMe (C: mit 930 GB, davon rund 350 GB belegt) auf eine 512-GB-NVMe, inklusive der Fehlermeldungen, die unterwegs auftreten.

## Warum der Restore auf die kleinere Disk scheitert

Rescuezilla sichert jede Partition einzeln mit Partclone und speichert zusätzlich die Partitionstabelle. Beim Restore schreibt es diese Tabelle unverändert auf die Zieldisk. Eine Partition verkleinern kann Rescuezilla nicht. Vor dem Start prüft es deshalb, ob die letzte Partition vollständig auf der Zieldisk liegt, und bricht sonst mit dieser Meldung ab:

```text
The source partition table's final partition (/dev/nvme0n1p4:
1000203091968 bytes) must refer to a region completely within
the destination disk (512110190592 bytes).
```

Eine typische Windows-Installation hat vier Partitionen: die EFI-Systempartition (200 MB), die Microsoft-reservierte Partition (16 MB), C: und die Wiederherstellungspartition mit Windows RE (hier 904 MiB). Windows legt die Wiederherstellungspartition ans Ende der Disk. Es reicht also nicht, C: zu verkleinern: Auch die Wiederherstellungspartition muss nach vorne, direkt hinter C:.

Der Ablauf, den das Rescuezilla-Wiki für diesen Fall beschreibt, lautet: Quelldisk mit GParted verkleinern, neues Image ziehen, dieses Image zurückspielen. Ein vorher gezogenes Image der unveränderten Disk bleibt als Sicherung liegen, bis das neue System läuft.

## Schritt 1: Zielgrösse berechnen

Eine als 512 GB verkaufte SSD hat 512'110'190'592 Bytes, das sind 476,9 GiB (Rescuezilla zeigt auch diesen Wert an). Davon gehen EFI, MSR und Wiederherstellungspartition ab, zusammen gut 1,1 GiB. Für C: bleiben damit knapp 475 GiB. Mit etwas Reserve ist **470 GiB = 481'280 MiB** ein sinnvoller Zielwert. Die rund 6 GB, die auf der neuen SSD am Ende frei bleiben, fallen nicht ins Gewicht.

Die belegten Daten müssen unter diesem Wert liegen. Wie weit Windows ein Volume verkleinern könnte, zeigt dieser Befehl in einer PowerShell mit Administratorrechten:

```powershell
$s = Get-PartitionSupportedSize -DriveLetter C
"{0:N1} GB" -f ($s.SizeMin / 1GB)
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-DriveLetter C` | Volume, dessen mögliche Grössen abgefragt werden |
| `SizeMin` | kleinste Grösse, auf die Windows das Volume selbst verkleinern könnte |
| `"{0:N1} GB" -f` | formatiert den Bytewert als Gigabyte mit einer Nachkommastelle |

</details>

Liegt `SizeMin` deutlich über der belegten Datenmenge, blockieren unverschiebbare Dateien (Auslagerungsdatei, Wiederherstellungspunkte, MFT) die Verkleinerung mit Windows-Bordmitteln. GParted verschiebt diese Daten beim Verkleinern mit, die Grenze gilt dort nicht.

## Schritt 2: BitLocker und Ruhezustand ausschalten

GParted kann ein BitLocker-verschlüsseltes Volume weder lesen noch verkleinern, und Partclone kann es nicht als NTFS sichern. Prüfen Sie den Status in einer PowerShell mit Administratorrechten:

```powershell
manage-bde -status C:
```

Steht dort nicht `Fully Decrypted`, schalten Sie BitLocker ab und warten, bis die Entschlüsselung abgeschlossen ist:

```powershell
manage-bde -off C:
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-status` | zeigt Verschlüsselungsgrad, Methode und Schutzstatus des Volumes |
| `-off` | entschlüsselt das Volume vollständig und entfernt die Schlüsselschutzvorrichtungen |
| `C:` | Positionsargument: das betroffene Volume |

</details>

Die Entschlüsselung läuft im Hintergrund und kann je nach Datenmenge eine Stunde oder länger dauern. Bei Geräten, die über Intune verwaltet werden, kann eine Richtlinie BitLocker nach kurzer Zeit wieder einschalten. Prüfen Sie den Status deshalb unmittelbar vor dem Start von GParted noch einmal.

Zusätzlich muss der Ruhezustand aus sein. Mit aktivem Schnellstart fährt Windows nicht vollständig herunter, sondern speichert den Kernel-Zustand in `hiberfil.sys`. NTFS gilt dann als noch in Gebrauch, und GParted verweigert jede Änderung. Ein Befehl schaltet Ruhezustand und Schnellstart gemeinsam ab und löscht `hiberfil.sys`:

```powershell
powercfg /h off
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `/h off` | Kurzform von `/hibernate off`: deaktiviert Ruhezustand und Schnellstart, `hiberfil.sys` wird entfernt |

</details>

## Schritt 3: Rescuezilla booten

Windows kann den nächsten Start direkt in das Startmenü mit den erweiterten Optionen lenken. Dort wählen Sie „Gerät verwenden“ und den USB-Stick mit Rescuezilla:

```powershell
shutdown /r /o /t 0
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `/r` | Neustart statt Herunterfahren |
| `/o` | startet in die erweiterten Startoptionen (Windows RE); nur zusammen mit `/r` |
| `/t 0` | Wartezeit in Sekunden bis zur Ausführung |

</details>

Alternativ öffnet sich beim Einschalten das Bootmenü der Firmware, bei ASRock-Boards mit F11. Fehlt „Gerät verwenden“, führt `shutdown /r /fw /t 0` direkt ins UEFI-Setup, wo sich das Startlaufwerk für den nächsten Start wählen lässt.

## Schritt 4: C: verkleinern und die Wiederherstellungspartition verschieben

Auf dem Rescuezilla-Desktop starten Sie **Partition Editor** (GParted), nicht Rescuezilla selbst.

1.  Wählen Sie oben rechts die Quelldisk. Achten Sie auf die Grösse (hier 931.51 GiB), damit Sie nicht versehentlich die externe Sicherungsdisk oder eine zweite interne Disk bearbeiten.

2.  Rechtsklick auf die NTFS-Partition mit C: → **Resize/Move**. Im Feld **New size (MiB)** den Zielwert eintragen, hier `481280`, und mit der Tab-Taste bestätigen. **Free space preceding** bleibt unverändert. Mit **Resize/Move** übernehmen.

3.  Unter C: steht jetzt eine Zeile `unallocated`, darunter die Wiederherstellungspartition (NTFS, rund 900 MiB, Flags `hidden, diag`). Rechtsklick auf die Wiederherstellungspartition → **Resize/Move**.

4.  Im Feld **Free space preceding (MiB)** den Wert löschen, `0` eintragen und mit Tab bestätigen. **New size** bleibt gleich, **Free space following** springt auf den gesamten freien Bereich. Steht nach dem Tab `1` statt `0`, ist das die Ausrichtung auf ganze MiB und in Ordnung. Mit **Resize/Move** übernehmen.

5.  Eine Warnung, dass das Verschieben den Start verhindern könnte, mit OK bestätigen. Sie betrifft Partitionen mit Bootloader; Windows startet von der EFI-Systempartition, die unverändert bleibt.

6.  Die Reihenfolge ist jetzt: EFI, MSR, C:, Wiederherstellungspartition, `unallocated`. Bis hier ist nur vorgemerkt. Erst ein Klick auf das grüne Häkchen (**Apply All Operations**) führt die Änderungen aus.

Windows RE findet seine Partition nach dem Verschieben wieder, weil sie ihre Partitionsnummer behält. `reagentc /info` zeigt danach weiterhin `Enabled` mit dem Pfad `harddisk0\partition4\Recovery\WindowsRE`.

Die Rescuezilla-Anleitung verkleinert nur die letzte Partition. Das reicht, wenn C: die letzte Partition ist. Bei einer Standardinstallation von Windows 10 oder 11 liegt die Wiederherstellungspartition dahinter, dort ist das Verschieben zwingend.

## Schritt 5: Das Dirty-Flag entfernen

Das nächste Backup bricht nach dem Verkleinern bei C: mit dieser Meldung ab:

```text
ntfsclone-ng.c: NTFS Volume '/dev/nvme0n1p3' is scheduled for a check
or it was shutdown uncleanly. Please boot Windows or fix it by fsck.
```

Die Ursache ist gewollt: `ntfsresize`, das GParted für NTFS verwendet, markiert das Dateisystem vor jeder Grössenänderung zur Prüfung und lässt diese Markierung stehen. Laut Manpage soll Windows beim nächsten Start `chkdsk` ausführen. Im beschriebenen Fall war das Flag nach einem normalen Windows-Start aber weiterhin gesetzt, und `Get-Volume C` meldete `Full Repair Needed`. Den Zustand prüfen Sie in einer PowerShell mit Administratorrechten:

```powershell
fsutil dirty query C:
```

Meldet der Befehl `Volume - C: is Dirty`, lassen Sie die Prüfung beim nächsten Start ausführen:

```powershell
chkdsk C: /f
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `dirty query C:` | fsutil: fragt ab, ob das Dirty-Bit des Volumes gesetzt ist |
| `C:` | chkdsk: das zu prüfende Volume |
| `/f` | behebt gefundene Fehler; auf dem Systemvolume wird die Prüfung für den nächsten Start eingeplant |

</details>

Die Frage, ob die Prüfung beim nächsten Neustart ausgeführt werden soll, mit J beantworten und Windows normal neu starten. Danach muss `fsutil dirty query C:` die Meldung `is NOT Dirty` zeigen, und `Get-Volume C` meldet wieder `Healthy`. Erst dann starten Sie Rescuezilla erneut.

## Schritt 6: Neues Image ziehen und prüfen

Mit Rescuezilla ziehen Sie jetzt ein neues Image der verkleinerten Disk. Die Imagegrösse bleibt praktisch gleich wie beim ersten Backup (hier rund 214 GB), weil Partclone nur belegte Blöcke sichert. Rescuezilla legt jedes Backup in einem eigenen Ordner mit Zeitstempel ab, etwa `2026-09-29-1343-img-rescuezilla`.

In der Liste zur Imageauswahl stehen danach alle Images nebeneinander. Die Spalte **Partitions** zeigt die Grössen: Das Image vor dem Verkleinern enthält `ntfs 930.4GB`, das neue `ntfs 470GB`. Ein abgebrochener Backup-Versuch erscheint mit gelbem Warndreieck; er enthält die alte Partitionstabelle und löst beim Restore genau die Fehlermeldung aus dem ersten Abschnitt aus. Löschen Sie solche Ordner, damit sie nicht versehentlich gewählt werden.

Ob ein Image auf die Zieldisk passt, lässt sich vorab auch in der Datei `<disk>-pt.parted` im Imageordner ablesen. Sie enthält die gesicherte Partitionstabelle in Sektoren zu 512 Bytes:

```text
Number  Start       End         Size        File system  Name
 1      2048s       411647s     409600s     fat32        EFI system partition
 2      411648s     444415s     32768s                   Microsoft reserved partition
 3      444416s     986105855s  985661440s  ntfs         Basic data partition
 4      986105856s  987957247s  1851392s    ntfs
```

Das Ende der letzten Partition (987'957'247 + 1 Sektoren × 512 Bytes = 505,8 GB) liegt unter den 512,1 GB der Zieldisk. Das Image passt.

## Schritt 7: Auf die neue SSD zurückspielen

1.  In Rescuezilla **Restore** wählen, das Laufwerk mit den Images und dann das **neue** Image (ohne Warndreieck, mit der verkleinerten C:-Partition).

2.  Als Ziel die neue SSD wählen. Prüfen Sie Modell und Grösse zweimal: Der Restore überschreibt die Zieldisk vollständig.

3.  Alle Partitionen angehakt lassen, ebenso **Overwrite partition table**, und den Restore starten.

4.  Danach von der neuen SSD booten. Am einfachsten klemmen Sie die alte Disk vorher ab; sonst wählen Sie die neue SSD im Bootmenü der Firmware.

## Nacharbeiten

Auf dem neuen Laufwerk schalten Sie wieder ein, was für den Umzug deaktiviert wurde. Den Ruhezustand aktiviert `powercfg /h on`. BitLocker aktivieren Sie über Einstellungen → Datenschutz und Sicherheit → Geräteverschlüsselung oder mit `manage-bde -on C:`; prüfen Sie danach mit `manage-bde -protectors -get C:`, dass der Wiederherstellungsschlüssel gesichert ist. Wurde Secure Boot für den Start von Rescuezilla im UEFI deaktiviert, schalten Sie es wieder ein; solange es aus ist, protokolliert BitLocker bei jedem Start das Ereignis 810.

Das Image der unveränderten 1-TB-Disk können Sie löschen, sobald Windows von der neuen SSD sauber startet und die Daten vollständig sind.

## Quellen

1.  [Rescuezilla Wiki: Restoring to a smaller disk](https://github.com/rescuezilla/rescuezilla/wiki/HOWTO:-Restoring-to-a-smaller-disk.-Eg,-1000GB-HDD-to-500GB-SSD): offizieller Ablauf mit GParted, Grund für die Einschränkung beim Restore.

2.  [Rescuezilla](https://rescuezilla.com/): Projektseite mit Download des Live-Systems.

3.  [GParted Manual](https://gparted.org/display-doc.php?name=help-manual): Bedienung von Resize/Move, Ausrichtung auf MiB und Ausführen vorgemerkter Operationen.

4.  [ntfsresize(8), Ubuntu Manpage](https://manpages.ubuntu.com/manpages/noble/man8/ntfsresize.8.html): Werkzeug hinter der NTFS-Verkleinerung in GParted; beschreibt die absichtlich gesetzte Prüfmarkierung.

5.  [Microsoft Learn: chkdsk](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/chkdsk): Parameter `/f` und Einplanen der Prüfung auf dem Systemvolume.

6.  [Microsoft Learn: fsutil dirty](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/fsutil-dirty): Abfrage und Bedeutung des Dirty-Bits.

7.  [Microsoft Learn: manage-bde status](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-status): Statusabfrage von BitLocker.

8.  [Microsoft Learn: manage-bde off](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-off): Entschlüsseln eines Volumes.

9.  [Microsoft Learn: Powercfg command-line options](https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/powercfg-command-line-options): Option `/hibernate` und ihr Einfluss auf den Schnellstart.

10.  [Microsoft Learn: Get-PartitionSupportedSize](https://learn.microsoft.com/en-us/powershell/module/storage/get-partitionsupportedsize): minimale und maximale Partitionsgrösse aus Sicht von Windows.

11.  [Microsoft Learn: REAgentC command-line options](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/reagentc-command-line-options): Status und Speicherort von Windows RE prüfen.

12.  [Microsoft Learn: shutdown](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/shutdown): Parameter `/o` und `/fw` für den Start in die erweiterten Optionen beziehungsweise ins UEFI.
