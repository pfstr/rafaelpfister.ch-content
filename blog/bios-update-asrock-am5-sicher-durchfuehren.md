---
title: "BIOS-Update sicher durchführen: Anleitung am Beispiel ASRock AM5, inklusive BitLocker-Fallstrick"
navTitle: "BIOS-Update"
description: "Der komplette Ablauf eines BIOS-Updates am Beispiel eines ASRock-AM5-Boards: Version ermitteln, Download per Hash verifizieren, BitLocker korrekt pausieren, ins UEFI booten (auch wenn F2 ins Leere läuft), per Instant Flash aktualisieren und die Einstellungen nach dem Update sinnvoll setzen."
date: "2026-08-26"
kategorie: "PC & Hardware"
timeToRead: "8 min to read"
themen:
  - "pc-hardware"
produkte:
  - "windows-client"
protokolle:
  - "troubleshooting"
  - "releases"
slug: "bios-update-asrock-am5-sicher-durchfuehren"
url: "https://rafaelpfister.ch/blog/bios-update-asrock-am5-sicher-durchfuehren"
translationId: "article-82840b2d159b9367"
---

Ein BIOS-Update gehört zu den Wartungsarbeiten, die selten anstehen und deshalb bei jedem Mal wieder Fragen aufwerfen: Welche Version ist die richtige, wie kommt sie sicher auf das Board, und was ist vorher und nachher zu beachten? Diese Anleitung dokumentiert den kompletten Ablauf am Beispiel eines ASRock A620I Lightning WiFi (Sockel AM5) mit der herstellereigenen Methode Instant Flash. Die Schritte übertragen sich auf jedes moderne Mainboard, die Fallstricke (BitLocker, Fast Boot, Einstellungs-Reset) sind herstellerunabhängig.

## Wann ein BIOS-Update angezeigt ist

Drei Anlässe rechtfertigen den Eingriff. Erstens Sicherheitskorrekturen: Firmware-Lücken lassen sich nur über ein BIOS-Update schliessen, und die Changelogs der Hersteller benennen sie meist nur knapp. Zweitens Kompatibilität: Unterstützung für neue CPU-Generationen und verbesserte Speicherkompatibilität kommen ausschliesslich über neue Firmware-Stände, bei AM5 über die AGESA-Referenzfirmware von AMD, die die Board-Hersteller in ihre BIOS-Versionen einbetten. Drittens Stabilität: Startet ein System spontan neu und protokolliert das Ereignisprotokoll dazu nur Kernel-Power 41 mit `BugcheckCode=0`, ist der Absturz auf Hardware- oder Firmware-Ebene passiert, ohne Beteiligung von Windows; typische Ursachen sind instabile Spannungen und Speichertraining, und genau diese Ebene pflegen die AGESA-Releases. Einträge wie "Improve memory compatibility and system stability" oder ein überarbeitetes EXPO-Handling in den Changelogs sind der Hinweis, dass ein Update solche Probleme adressiert. Läuft ein System dagegen stabil und ist von den behobenen Lücken nicht betroffen, ist Abwarten legitim; ein BIOS-Update ohne Anlass ist ein Risiko ohne Gegenwert.

## Schritt 1: Ist-Zustand ermitteln

Bevor Sie etwas herunterladen, brauchen Sie zwei Angaben: das exakte Board-Modell und die installierte BIOS-Version. Beides liefert PowerShell ohne Neustart:

```powershell
Get-CimInstance Win32_ComputerSystem | Select-Object Manufacturer, Model
Get-CimInstance Win32_BIOS | Select-Object SMBIOSBIOSVersion, ReleaseDate
```

Notieren Sie sich die Version. Sie brauchen sie später für die Erfolgskontrolle, und beim Lesen der Changelogs wollen Sie wissen, welche Versionen Sie überspringen.

## Schritt 2: BIOS herunterladen und Prüfsumme verifizieren

Laden Sie das BIOS ausschliesslich von der Produktseite des Herstellers, nie von Drittanbieter-Portalen. ASRock veröffentlicht zu jeder Version die SHA256-Prüfsumme; nach dem Download vergleichen Sie diese, bevor die Datei auch nur in die Nähe eines USB-Sticks kommt:

```powershell
Get-FileHash .\A620I_Lightning_WiFi_4.43.zip -Algorithm SHA256
```

Stimmt der Wert nicht mit der Herstellerangabe überein, ist der Download beschädigt oder manipuliert: nicht flashen. Nach dem Entpacken bleibt eine einzelne ROM-Datei übrig, im Beispiel `A62IRW_4.43.ROM` mit 32 MB.

## Schritt 3: USB-Stick vorbereiten

Der Flash-Mechanismus im UEFI (bei ASRock "Instant Flash", bei anderen Herstellern Q-Flash, EZ Flash oder M-Flash) liest den Stick direkt aus der Firmware heraus. Das bedeutet: nur FAT32 wird zuverlässig erkannt, NTFS und exFAT nicht. Fast jeder fertig gekaufte Stick ist bereits FAT32; kontrollieren können Sie es so:

```powershell
Get-CimInstance Win32_LogicalDisk -Filter "DriveType=2" |
  Select-Object DeviceID, FileSystem, VolumeName
```

Kopieren Sie die ROM-Datei ins Wurzelverzeichnis des Sticks. Ein Neuformatieren ist nur nötig, wenn das Dateisystem nicht passt. Die Stick-Grösse ist unkritisch, die Datei ist kleiner als jede heute übliche Kapazität.

Ein Hinweis zur Methodenwahl: Viele Boards bieten zusätzlich einen BIOS-Flashback-Knopf, der ohne CPU und ohne funktionierendes System flasht. Das ist der Rettungsweg für ein Board, das nicht mehr startet. Für ein laufendes System ist Instant Flash im UEFI der richtige und einfachere Weg. Windows-basierte Flash-Tools sind bei aktuellen Plattformen weder nötig noch empfehlenswert.

## Schritt 4: BitLocker pausieren, sonst droht die Schlüsselabfrage

Das ist der Fallstrick, der in vielen Anleitungen fehlt. Ist die Systemplatte mit BitLocker verschlüsselt (bei Windows 11 mit Microsoft-Konto häufig automatisch aktiv), bindet BitLocker den Schlüssel an die Messwerte des TPM. Ein BIOS-Update verändert diese Messwerte, und beim nächsten Start verlangt Windows den 48-stelligen Wiederherstellungsschlüssel. Wer den nicht griffbereit hat, steht vor einem unzugänglichen System.

BitLocker bringt für dieses Szenario einen eigenen Mechanismus mit. In einer PowerShell mit Administratorrechten:

```powershell
Suspend-BitLocker -MountPoint C: -RebootCount 2
```

Der Parameter `RebootCount 2` deckt beide anstehenden Neustarts ab (einmal ins UEFI, einmal nach dem Flash); danach reaktiviert sich der Schutz von selbst und versiegelt den Schlüssel gegen die neuen Messwerte. Prüfen Sie unabhängig davon vorher, dass der Wiederherstellungsschlüssel auffindbar ist, etwa im Microsoft-Konto unter aka.ms/myrecoverykey oder per `manage-bde -protectors -get C:`.

## Schritt 5: Ins UEFI kommen, auch wenn F2 nicht reagiert

Der klassische Weg über F2 oder Entf beim Einschalten scheitert auf modernen Systemen oft: Mit aktiviertem Fast Boot initialisiert die Firmware die USB-Tastatur erst nach dem POST, der Tastendruck kommt nicht an. Sie sind auf die Taste aber nicht angewiesen: Windows kann den nächsten Neustart direkt ins UEFI-Setup lenken. In einer PowerShell mit Administratorrechten:

```powershell
shutdown /r /fw /t 5
```

Meldet der Befehl den Fehler 203 ("The system could not find the environment option that was entered"), fehlen fast immer die Administratorrechte: Ohne Elevation darf der Prozess die nötige Firmware-Variable nicht setzen, und die Fehlermeldung benennt diese Ursache nicht. Ein zweiter möglicher Weg ohne Firmware-Variable führt über die Wiederherstellungsumgebung: `shutdown /r /o`, dann Problembehandlung, Erweiterte Optionen, UEFI-Firmwareeinstellungen.

## Schritt 6: Flashen mit Instant Flash

Im UEFI finden Sie Instant Flash im Tool-Menü. Das Werkzeug listet alle ROM-Dateien auf dem Stick auf; nach der Auswahl prüft es die Datei, flasht und startet selbständig neu. Während der wenigen Minuten gilt die einzige harte Regel des ganzen Vorgangs: Stromversorgung nicht unterbrechen und den Rechner nicht ausschalten. Ein abgebrochener Flash ist der einzige Schritt dieser Anleitung, der das Board tatsächlich startunfähig machen kann (und dann den erwähnten Flashback-Rettungsweg braucht).

## Schritt 7: Nacharbeiten, denn das Update setzt alles zurück

Nach dem Flash stehen sämtliche BIOS-Einstellungen auf Werkseinstellung. Das ist so vorgesehen und bietet eine diagnostische Chance: Das RAM läuft jetzt ohne EXPO-Profil auf der JEDEC-Grundgeschwindigkeit. Wenn Sie wegen Stabilitätsproblemen geflasht haben, lassen Sie es bewusst ein bis zwei Wochen so. Bleiben die Abstürze aus, war das Speicherprofil beteiligt, und Sie können EXPO mit der neuen Firmware gezielt erneut testen. Der Alltagsunterschied zwischen 4800 und 6000 MT/s ist ausserhalb von Benchmarks kaum spürbar; ein stabiler Rechner ist jeden Benchmark-Punkt wert.

Zwei Einstellungen lohnen den Besuch im UEFI ohnehin: Wer die Neustarts im Leerlauf hatte, kann unter Advanced, AMD CBS die Option "Power Supply Idle Control" auf "Typical Current Idle" setzen; das entschärft eine bekannte Unverträglichkeit mancher Netzteile mit den tiefen Idle-Zuständen der Ryzen-CPUs. Und wer künftig wieder per F2 ins Setup will, kann Fast Boot abstellen.

Die Erfolgskontrolle zurück in Windows:

```powershell
Get-CimInstance Win32_BIOS | Select-Object SMBIOSBIOSVersion
Get-CimInstance Win32_PhysicalMemory |
  Select-Object PartNumber, ConfiguredClockSpeed
manage-bde -status C:
```

Die erste Zeile muss die neue Version zeigen, die zweite den erwarteten Speichertakt, und BitLocker muss wieder "Schutz aktiviert" melden. Damit ist das Update abgeschlossen und dokumentiert. Wurde wegen Stabilitätsproblemen geflasht, zeigt erst die Beobachtung über die folgenden Wochen, ob sie behoben sind, am einfachsten mit einem Blick auf neue Kernel-Power-41-Einträge im System-Ereignisprotokoll.

## Quellen

1.  [ASRock A620I Lightning WiFi, BIOS-Downloads](https://pg.asrock.com/mb/AMD/A620I%20Lightning%20WiFi/index.asp#BIOS): Versionsliste mit Changelogs, SHA256-Prüfsummen und den unterstützten Update-Methoden des Beispiel-Boards.

2.  [Microsoft Learn: Suspend-BitLocker](https://learn.microsoft.com/en-us/powershell/module/bitlocker/suspend-bitlocker): Referenz zum Pausieren des BitLocker-Schutzes inklusive des Parameters RebootCount.

3.  [Microsoft Learn: Advanced troubleshooting for Event ID 41](https://learn.microsoft.com/en-us/troubleshoot/windows-client/performance/event-id-41-restart): Einordnung von Kernel-Power 41 und der Bedeutung von BugcheckCode 0.

4.  [Microsoft Learn: shutdown-Befehl](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/shutdown): Dokumentation der Parameter /fw und /o für den Neustart ins UEFI beziehungsweise in die Wiederherstellungsumgebung.
