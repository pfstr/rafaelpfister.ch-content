---
title: "Cold Storage mit Rclone testen: ein praxistauglicher Testplan"
navTitle: "Rclone testen"
description: "Bevor ein Dienst seine Dateien über einen Rclone-Mount aus der Cloud liest, sollten Sie mehr als den Verzeichniszugriff prüfen. Dieser Testplan deckt Cold Reads, Warm Reads, Schreibvorgänge, Cache-Verhalten, Dateiintegrität und Ausfälle ab."
date: "2026-07-26"
kategorie: "Rclone"
timeToRead: "11 Min. Lesezeit"
themen:
  - "rclone"
  - "testing"
produkte:
  - "rclone"
protokolle:
  - "testing"
  - "backup-dr"
  - "storage"
  - "troubleshooting"
related:
  - "rclone-mount-in-docker-container"
  - "paperless-dokumente-clouddienst-auslagern"
slug: "cloud-mount-testen-dummy-pdfs"
translationId: "article-8592f808b2e93cd4"
url: "https://rafaelpfister.ch/blog/cloud-mount-testen-dummy-pdfs"
---

Ein Rclone-Mount ist schnell eingerichtet. Das Remote erscheint als Verzeichnis, `ls` zeigt Dateien und der erste Funktionstest ist bestanden. Für den produktiven Betrieb sagt das noch wenig aus.

Sobald ein Dienst auf den Mount zugreift, kommen weitere Fragen dazu: Wie lange dauert der erste Zugriff auf eine Datei? Welche Zugriffe bedient der lokale Cache? Was geschieht mit einer noch nicht hochgeladenen Datei, wenn Rclone abstürzt? Sieht ein laufender Container den neu aufgebauten Mount wieder? Und wie reagiert der Dienst, wenn die Cloud vorübergehend nicht erreichbar ist?

Dieser Artikel liefert dafür einen allgemeinen Testplan. Sie können ihn für ein Dokumentenarchiv, einen Medienserver, eine Fotoverwaltung oder jeden anderen Dienst verwenden, der selten benötigte Dateien über Rclone aus einem Cold Storage bezieht.

## Die wichtigsten Optionen von rclone

Zur Orientierung vorab die Rclone-Optionen, die in diesem Testplan vorkommen, sinngemäss aus der Dokumentation übersetzt:

<details class="options-details">
<summary>Optionen im Überblick</summary>

| Option | Bedeutung |
|---|---|
| `--seed n` | Startwert des Zufallsgenerators bei `rclone test makefiles`; gleicher Seed ergibt denselben Dateibaum |
| `--files n` | Anzahl zu erzeugender Testdateien |
| `--files-per-directory n` | Durchschnittliche Anzahl Dateien pro Verzeichnis |
| `--min-file-size grösse` | Kleinste erzeugte Dateigrösse (Suffixe wie K, M, G erlaubt) |
| `--max-file-size grösse` | Grösste erzeugte Dateigrösse |
| `--progress` | Laufende Fortschrittsanzeige während der Übertragung |
| `--stats dauer` | Intervall, in dem Transferstatistiken ausgegeben werden |
| `--log-file datei` | Schreibt das Protokoll in die angegebene Datei |
| `--log-level stufe` | Detailgrad des Protokolls: DEBUG, INFO, NOTICE oder ERROR |
| `--one-way` | Prüft bei `rclone check` nur, ob alle Quelldateien im Ziel vorhanden und identisch sind; zusätzliche Dateien im Ziel gelten nicht als Fehler |
| `--download` | Lädt die Daten beim Vergleich tatsächlich herunter, statt nur Hashes zu vergleichen |
| `--vfs-cache-mode modus` | Cache-Strategie des VFS-Layers; `full` puffert Lese- und Schreibzugriffe lokal |
| `--cache-dir verzeichnis` | Verzeichnis für den lokalen Cache |
| `--vfs-cache-max-size grösse` | Weiches Limit für die Gesamtgrösse des VFS-Caches |
| `--vfs-cache-poll-interval dauer` | Intervall, in dem Rclone den Cache prüft und bereinigt |
| `--vfs-write-back dauer` | Wartezeit nach dem Schliessen einer Datei, bevor der Upload ins Remote beginnt |
| `--vfs-read-ahead grösse` | Zusätzliche Datenmenge, die bei `full` über die angeforderte Position hinaus vorausgelesen wird |
| `--poll-interval dauer` | Intervall, in dem Rclone das Remote auf Änderungen abfragt (nur bei Backends mit Polling-Unterstützung) |
| `--dir-cache-time dauer` | Gültigkeitsdauer zwischengespeicherter Verzeichnislisten |
| `--allow-other` | Erlaubt anderen Benutzern als dem mountenden den Zugriff auf den FUSE-Mount |

</details>

Die vollständigen Listen stehen in der Rclone-Dokumentation, insbesondere unter [rclone mount](https://rclone.org/commands/rclone_mount/) und in der Übersicht der [globalen Flags](https://rclone.org/flags/).

## Zuerst festlegen, was Sie erreichen wollen

Cold Storage bedeutet nicht automatisch dasselbe für jede Anwendung. Ein Medienserver liest grosse Dateien meist sequenziell. Eine Fotoverwaltung lädt viele kleine Vorschaudaten und springt an verschiedene Stellen. Ein Dokumentenarchiv öffnet vergleichsweise kleine Dateien, dafür aber oft nur einmal.

Notieren Sie vor dem Test die wichtigsten Eigenschaften Ihres echten Bestands:

- typische Dateigrösse sowie grösste vorkommende Datei
- Anzahl Dateien pro Verzeichnis
- vollständiges Lesen oder zufällige Zugriffe auf einzelne Bereiche
- Verhältnis zwischen Lese- und Schreibzugriffen
- Zahl gleichzeitiger Benutzer oder Prozesse
- Änderungen, die ausserhalb des Mounts direkt im Remote stattfinden
- akzeptable Wartezeit für einen Cold Read
- maximal verfügbarer Platz für den lokalen Cache

Erst daraus entstehen sinnvolle Erfolgskriterien. Eine einzelne Datei in 1.2 Sekunden zu öffnen, kann für ein Archiv völlig in Ordnung und für eine interaktive Anwendung unbrauchbar sein.

## Einen reproduzierbaren Testbestand erzeugen

Rclone bringt dafür bereits ein passendes Werkzeug mit. `rclone test makefiles` erzeugt mit einem festen Seed jedes Mal denselben Dateibaum:

```bash
rclone test makefiles ./testdata \
  --seed 42 \
  --files 250 \
  --files-per-directory 25 \
  --min-file-size 16K \
  --max-file-size 32M
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `./testdata` | Zielverzeichnis, in dem der Testbaum entsteht |
| `--seed 42` | Fester Startwert des Zufallsgenerators; jeder Lauf erzeugt denselben Bestand |
| `--files 250` | 250 Testdateien insgesamt |
| `--files-per-directory 25` | Durchschnittlich 25 Dateien pro Verzeichnis |
| `--min-file-size 16K` | Kleinste Datei 16 KiB |
| `--max-file-size 32M` | Grösste Datei 32 MiB |

</details>

Passen Sie Anzahl und Grössen an Ihren echten Bestand an. Testen Sie nicht nur Durchschnittsdateien. Einige sehr kleine Dateien zeigen, wie teuer Metadatenzugriffe sind; einige grosse Dateien machen Durchsatz, Read-Ahead und Cache-Verhalten sichtbar.

Ergänzen Sie ausserdem Dateinamen, die in der Praxis Probleme verursachen können:

```bash
mkdir -p "testdata/Sonderfälle/Unterordner"
printf 'Leerzeichen\n' > "testdata/Sonderfälle/Datei mit Leerzeichen.txt"
printf 'Umlaute\n' > "testdata/Sonderfälle/Grösse und Änderung.txt"
printf 'Grossschreibung\n' > "testdata/Sonderfälle/Test.txt"
printf 'Kleinschreibung\n' > "testdata/Sonderfälle/test.txt"
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `mkdir -p` | Legt auch fehlende übergeordnete Verzeichnisse an und meldet keinen Fehler, wenn das Verzeichnis existiert |
| `printf '…\n' > datei` | Schreibt den angegebenen Text als Dateiinhalt; die Umleitung erzeugt die Datei mit dem problematischen Namen |

</details>

Der letzte Test ist besonders wichtig, wenn lokales Dateisystem und Cloud-Backend Gross- und Kleinschreibung unterschiedlich behandeln.

Wenn Ihr Dienst nur bestimmte Formate akzeptiert, reichen beliebige Binärdateien nicht. Erzeugen Sie dann zusätzlich synthetische Dateien in genau diesen Formaten. Bei Paperless-ngx waren das PDFs mit echter Textebene, damit der Test nicht versehentlich die OCR-Leistung statt des Speicherpfads misst. Bei einer Fotoverwaltung gehören unterschiedliche Bildgrössen und Formate in den Bestand, bei einem Medienserver kurze Dateien mit verschiedenen Codecs.

## Eine Referenzmessung ohne Mount

Bevor FUSE und VFS-Cache ins Spiel kommen, sollten Sie das Backend direkt messen. Kopieren Sie den Bestand mit Rclone ins Test-Remote und speichern Sie ein ausführliches Protokoll:

```bash
rclone copy ./testdata remote:cold-storage-test \
  --progress \
  --stats 5s \
  --log-file rclone-copy.log \
  --log-level INFO
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `./testdata` | Quelle der Kopie: der lokal erzeugte Testbestand |
| `remote:cold-storage-test` | Ziel: Pfad im konfigurierten Remote |
| `--progress` | Laufende Fortschrittsanzeige im Terminal |
| `--stats 5s` | Transferstatistik alle 5 Sekunden statt im Standardintervall |
| `--log-file rclone-copy.log` | Vollständiges Protokoll in eine Datei für die spätere Auswertung |
| `--log-level INFO` | Protokolliert Transfers, Retries und Fehler, ohne den DEBUG-Umfang |

</details>

Prüfen Sie danach, ob Quelle und Ziel übereinstimmen:

```bash
rclone check ./testdata remote:cold-storage-test \
  --one-way \
  --download
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `./testdata` | Referenz: der lokale Originalbestand |
| `remote:cold-storage-test` | Prüfling: der frisch hochgeladene Bestand im Remote |
| `--one-way` | Prüft nur, ob alle Quelldateien im Ziel vorhanden und identisch sind; zusätzliche Dateien im Ziel werden nicht bemängelt |
| `--download` | Lädt die Daten herunter und vergleicht die Inhalte, statt sich auf Hashes zu verlassen |

</details>

`--download` ist hier entscheidend, weil manche Backends keine passenden Hashes bereitstellen. Der Vergleich dauert länger, liefert aber eine brauchbare Ausgangsbasis für den späteren Integritätstest.

Halten Sie Upload-Zeit, Transferrate, Anzahl Retries und API-Fehler fest. Wenn bereits der direkte Zugriff instabil ist, kann der Mount das nicht reparieren.

## Den Test-Mount vom produktiven Cache trennen

Verwenden Sie für die Messung einen eigenen Mountpunkt und ein eigenes Cache-Verzeichnis:

```bash
rclone mount remote:cold-storage-test /mnt/rclone-test \
  --vfs-cache-mode full \
  --cache-dir /var/cache/rclone-test \
  --vfs-cache-max-size 10G \
  --vfs-cache-poll-interval 1m \
  --allow-other \
  --log-file /var/log/rclone-test.log \
  --log-level INFO
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `remote:cold-storage-test` | Zu mountendes Remote samt Pfad |
| `/mnt/rclone-test` | Mountpunkt auf dem Testsystem |
| `--vfs-cache-mode full` | Puffert Lese- und Schreibzugriffe vollständig im lokalen Cache |
| `--cache-dir /var/cache/rclone-test` | Eigenes Cache-Verzeichnis, getrennt vom produktiven Cache |
| `--vfs-cache-max-size 10G` | Weiches Limit von 10 GiB für den VFS-Cache |
| `--vfs-cache-poll-interval 1m` | Cache-Prüfung und Bereinigung im Minutentakt |
| `--allow-other` | Auch andere Benutzer als der mountende dürfen zugreifen; nötig für Dienste und Container |
| `--log-file /var/log/rclone-test.log` | Protokoll in eine Datei, um Zugriffe während der Tests nachzuvollziehen |
| `--log-level INFO` | Mittlerer Detailgrad des Protokolls |

</details>

Die Werte sind ein Beispiel und keine allgemeine Empfehlung. Entscheidend ist die Trennung: Ein leerer Test-Cache macht Cold Reads reproduzierbar, ohne dass Sie Dateien aus einem laufenden produktiven Cache löschen müssen.

`--vfs-cache-mode full` ist für Anwendungen meist der aufschlussreichste Testmodus. Rclone puffert dabei Lese- und Schreibzugriffe lokal und kann Dateizugriffe besser abbilden, die bei einem reinen Objektspeicher nicht möglich wären. Die zusätzliche Kompatibilität kostet lokalen Speicherplatz.

## Immer aus Sicht des echten Dienstes prüfen

Ein Mount kann für Ihren Benutzer funktionieren und für den Dienst trotzdem unbrauchbar sein. Häufige Ursachen sind eine andere Benutzer-ID, fehlendes `--allow-other`, Container-Grenzen oder eine falsche Mount-Propagation.

Führen Sie deshalb mindestens einen vollständigen Lesezugriff mit derselben Identität aus, unter der später die Anwendung läuft:

```bash
sudo -u <service-user> sha256sum /mnt/rclone-test/pfad/zur/datei
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-u <service-user>` | Führt den Befehl als der angegebene Benutzer aus, nicht als root |
| `/mnt/rclone-test/pfad/zur/datei` | Zu lesende Datei; `sha256sum` erzwingt einen vollständigen Lesezugriff |

</details>

Läuft der Dienst in Docker, gehört der Test in den Container:

```bash
docker exec --user <uid>:<gid> <app-container> \
  sha256sum /pfad/im/container/datei
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `--user <uid>:<gid>` | Führt den Befehl im Container mit dieser Benutzer- und Gruppen-ID aus, unabhängig vom Standardbenutzer des Images |
| `<app-container>` | Name oder ID des laufenden Anwendungscontainers |
| `sha256sum /pfad/im/container/datei` | Auszuführender Befehl; der Pfad ist der Mount aus Sicht des Containers |

</details>

Noch besser ist ein echter Anwendungstest. Öffnen Sie die Datei über die Weboberfläche oder API des Dienstes. Nur so bemerken Sie, ob die Anwendung beispielsweise mehrere parallele Reads startet, an das Dateiende springt oder zusätzliche Metadaten erwartet.

## Cold Reads und Warm Reads getrennt messen

Bei `--vfs-cache-mode full` liegen zwischen Anwendung und Cloud drei Ebenen:

| Ebene | Was dort liegt |
|---|---|
| Remote | die vollständige Datei im Clouddienst |
| VFS-Cache | lokal gespeicherte Bereiche bereits gelesener Dateien |
| Linux-Page-Cache | kürzlich verwendete Daten im RAM |

Für einen Cold Read nehmen Sie eine Datei, deren Inhalt noch nie über den Test-Mount gelesen wurde. Beim direkt anschliessenden Warm Read liegt sie im VFS-Cache und meistens zusätzlich im RAM.

```bash
measure_read() {
  file="$1"
  label="$2"
  start=$(date +%s%3N)
  cat "$file" > /dev/null
  end=$(date +%s%3N)
  printf '%s: %s ms\n' "$label" "$((end - start))"
}

measure_read "/mnt/rclone-test/grosse-datei.bin" "Cold Read"
measure_read "/mnt/rclone-test/grosse-datei.bin" "Warm Read"
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `date +%s%3N` | Zeitstempel in Millisekunden: Unix-Sekunden, direkt gefolgt von den ersten drei Stellen des Nanosekunden-Anteils (GNU date) |
| `cat "$file" > /dev/null` | Liest die Datei vollständig, verwirft die Ausgabe; gemessen wird nur die Lesezeit |
| `"$1"`, `"$2"` | Argumente der Shell-Funktion: Dateipfad und Beschriftung der Messzeile |

</details>

Messen Sie nicht nur eine Datei. Verwenden Sie mindestens zehn bisher ungelesene Dateien verschiedener Grösse und notieren Sie Median, langsamsten Wert und Dateigrösse. Ein einzelner Bestwert ist keine Entscheidungsgrundlage.

Ein Warm Read ist kein reiner Festplattentest, weil der Kernel Teile der Datei im RAM halten kann. Für die meisten Cold-Storage-Szenarien ist das kein Problem. Entscheidend ist, was ein Benutzer beim ersten und beim wiederholten Öffnen erlebt. Wenn Sie RAM und lokale Platte getrennt beurteilen wollen, müssen Sie den Page-Cache zusätzlich kontrollieren und nachweislich räumen.

## Nicht nur vollständige Lesezugriffe testen

`cat` liest eine Datei von Anfang bis Ende. Viele Anwendungen verhalten sich anders:

- Ein Videoplayer liest zunächst Header und Index, springt später an eine andere Position und lädt dann sequenziell weiter.
- Eine Bildverwaltung liest Metadaten und erzeugt anschliessend ein Vorschaubild.
- Ein Archivprogramm kann zuerst das Dateiende lesen.
- Mehrere Worker können gleichzeitig auf verschiedene Dateien zugreifen.

Testen Sie diese Abläufe mit der tatsächlichen Anwendung. Beobachten Sie parallel das Rclone-Protokoll und den Cache. Bei grossen Dateien ist interessant, wie viel Rclone wirklich lokal ablegt und ob `--vfs-read-ahead` zum Zugriffsmuster passt.

Ein Rclone-Mount ist ausserdem kein sinnvoller Speicherort für Datenbanken oder andere Dateien, die verlässliches Locking und häufige Änderungen innerhalb derselben Datei benötigen. Der VFS-Layer gleicht Unterschiede zwischen Dateisystem und Objektspeicher aus, macht aus dem Backend aber kein lokales Dateisystem.

## Den Schreibpfad separat abnehmen

Wenn Ihr Dienst nur liest, mounten Sie das Remote nach Möglichkeit schreibgeschützt. Muss er schreiben, testen Sie Erstellen, Überschreiben, Umbenennen und Löschen einzeln.

Eine geschriebene Datei erscheint nicht zwingend sofort im Remote. Bei aktivem VFS-Cache beginnt der Upload erst, nachdem die Datei geschlossen wurde und `--vfs-write-back` abgelaufen ist. Prüfen Sie deshalb beide Zustände:

1. Die Anwendung hat die Datei erfolgreich geschlossen.
2. Die Datei ist anschliessend über einen direkten Rclone-Zugriff im Remote lesbar.

```bash
printf 'writeback-test\n' > /mnt/rclone-test/writeback-test.txt

# Nach Ablauf von --vfs-write-back:
rclone cat remote:cold-storage-test/writeback-test.txt
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `/mnt/rclone-test/writeback-test.txt` | Zieldatei im Mount; die Umleitung schreibt über den VFS-Cache |
| `remote:cold-storage-test/writeback-test.txt` | Direkter Zugriff am Mount vorbei: `rclone cat` liest die Datei aus dem Remote und gibt sie auf stdout aus |

</details>

Wiederholen Sie den Test mit einer grossen Datei und beenden Sie Rclone während des noch laufenden Uploads. Starten Sie danach mit demselben Cache-Verzeichnis neu und kontrollieren Sie, ob der Upload fortgesetzt wird. Genau dieses Zeitfenster entscheidet darüber, wie viele Daten bei einem Serverausfall gefährdet sind.

Testen Sie auch Umbenennen und Löschen. Viele Cloud-Backends bilden diese Operationen anders ab als ein lokales Dateisystem. Relevant ist nicht nur, ob der Befehl erfolgreich endet, sondern wann die Änderung bei einem direkten Zugriff auf das Remote und bei weiteren Clients sichtbar wird.

## Änderungen ausserhalb des Mounts testen

Dateien können über die Weboberfläche des Anbieters, einen zweiten Rclone-Prozess oder einen anderen Server verändert werden. Der Mount sieht solche Änderungen nicht immer sofort, weil Verzeichnisinformationen zwischengespeichert werden.

Legen Sie deshalb mit einem zweiten Rclone-Aufruf eine Datei direkt im Remote an:

```bash
printf 'external-change\n' > external-change.txt
rclone copyto external-change.txt \
  remote:cold-storage-test/external-change.txt
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `external-change.txt` | Quelle: die lokal erzeugte Datei |
| `remote:cold-storage-test/external-change.txt` | Ziel mit exaktem Dateinamen; `copyto` kopiert eine einzelne Datei unter genau diesem Namen, statt wie `copy` in ein Verzeichnis |

</details>

Messen Sie, wann die Datei im Mount erscheint. Wiederholen Sie den Test für Änderung und Löschung. Das Ergebnis hängt vom Backend, dessen Polling-Unterstützung sowie `--poll-interval` und `--dir-cache-time` ab. Wenn die Anwendung aktuelle Änderungen sofort sehen muss, gehört dieses Verhalten ausdrücklich in die Abnahmekriterien.

Bei aktivierter Remote-Control-Schnittstelle können Sie den Verzeichnis-Cache gezielt verwerfen:

```bash
rclone rc vfs/forget
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `vfs/forget` | Auszuführender Remote-Control-Befehl: verwirft den zwischengespeicherten Verzeichnisbaum des VFS, sodass der nächste Zugriff neu beim Remote nachfragt |

</details>

Das ist nützlich für einen manuellen Test, ersetzt aber keine passende Betriebsstrategie.

## Den Cache unter Druck setzen

Ein fast leerer Cache ist der einfachste Fall. Setzen Sie `--vfs-cache-max-size` in einer zweiten Testrunde bewusst klein und lesen Sie mehr Daten, als hineinpassen.

```bash
du -sh /var/cache/rclone-test/vfs
du -sh --apparent-size /var/cache/rclone-test/vfs
find /var/cache/rclone-test/vfs -type f | wc -l
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `du -s` | Fasst den Platzverbrauch zu einer Summenzeile zusammen, statt jedes Unterverzeichnis aufzulisten |
| `du -h` | Ausgabe in menschenlesbaren Einheiten (K, M, G) |
| `du --apparent-size` | Zeigt die logische Dateigrösse statt des tatsächlich belegten Plattenplatzes |
| `find … -type f` | Berücksichtigt nur reguläre Dateien, keine Verzeichnisse |
| `wc -l` | Zählt die Zeilen der Ausgabe, hier also die Anzahl Cache-Dateien |

</details>

Die beiden Grössen können stark voneinander abweichen. Im Full-Modus verwendet Rclone Sparse Files: Eine Datei zeigt ihre vollständige logische Grösse, obwohl nur die gelesenen Bereiche lokalen Platz belegen.

Das Cache-Limit ist zudem weich. Rclone prüft es im Rhythmus von `--vfs-cache-poll-interval`, und geöffnete Dateien können nicht entfernt werden. Der Cache darf das Limit deshalb kurzzeitig überschreiten. Er sollte nach dem Schliessen der Dateien und dem nächsten Aufräumlauf aber wieder sinken.

Protokollieren Sie Spitzenwert, Wert nach der Bereinigung und die dafür benötigte Zeit. So lässt sich der nötige lokale Speicher vernünftig dimensionieren.

## Zwei unterschiedliche Ausfälle simulieren

Eine nicht erreichbare Cloud und ein abgestürzter Rclone-Prozess sind zwei verschiedene Fehler:

| Ausfall | Was Sie damit prüfen |
|---|---|
| Backend oder Netzwerk nicht erreichbar, Rclone läuft weiter | Verhalten bei Retries, Timeouts und bereits gecachten Dateien |
| Rclone-Prozess beendet | Verhalten des FUSE-Mounts und Wiederherstellung des Mountpunkts |

Simulieren Sie beides nur in der Testumgebung. Einen Rclone-Container können Sie für den zweiten Fall hart beenden:

```bash
docker kill --signal KILL <rclone-container>
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `--signal KILL` | Sendet SIGKILL statt des Standardsignals SIGTERM; der Prozess erhält keine Gelegenheit zum Aufräumen |
| `<rclone-container>` | Name oder ID des Rclone-Containers |

</details>

Prüfen Sie während des Ausfalls die Anwendung und nicht nur den Mountpunkt:

- Welche Funktionen bleiben verfügbar?
- Wie lange wartet ein Zugriff, bevor ein Fehler erscheint?
- Sind bereits vollständig gecachte Dateien noch erreichbar?
- Stoppt die Anwendung neue Schreibvorgänge?
- Entsteht eine verständliche Fehlermeldung oder nur ein hängender Prozess?
- Löst Ihr Monitoring aus?

Ein Schreibdienst darf bei fehlendem Mount nicht unbemerkt in das darunterliegende lokale Verzeichnis schreiben. Nach der Rückkehr des Mounts würden diese Dateien verdeckt. Ein einfacher Schutz vor jedem Schreibjob ist:

```bash
mountpoint -q /mnt/rclone-test || exit 1
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-q` | Keine Ausgabe; das Ergebnis steht nur im Exit-Code |
| `/mnt/rclone-test` | Zu prüfender Pfad; Exit-Code 0 nur, wenn dort tatsächlich ein Mount aktiv ist |
| `\|\| exit 1` | Bricht das Skript ab, wenn der Pfad kein Mountpunkt ist |

</details>

Nach dem Neustart von Rclone prüfen Sie den Mount auf dem Host und aus jedem konsumierenden Container. Ein neu aufgebauter Mount erreicht einen bereits laufenden Container nur mit passender Mount-Propagation. Für Docker ist meistens `rslave` auf der konsumierenden Seite nötig. Die Details stehen im Artikel [Rclone-Mounts in Docker zuverlässig betreiben](/blog/rclone-mount-in-docker-container).

## Ein konkretes Beispiel aus Paperless-ngx

Für meinen Paperless-Test erzeugte ich 40 PDFs mit insgesamt 13.9 MB. Ein bisher ungeöffnetes Dokument brauchte rund 1.8 Sekunden, ein direkt wiederholter Zugriff 19 bis 24 Millisekunden. Ein auf 4 MB begrenzter VFS-Cache stieg kurzzeitig auf 12.7 MiB und wurde beim nächsten Lauf wieder bereinigt.

Während das Remote nicht erreichbar war, funktionierten Dokumentliste, Volltextsuche und Vorschaubilder weiter, weil diese Daten lokal lagen. Nur das Original liess sich nicht öffnen. Nach dem Wiederaufbau des Mounts konnte der laufende Paperless-Container wieder auf die Dateien zugreifen, ohne selbst neu gestartet zu werden.

Diese Zahlen sind kein Benchmark für Rclone oder Proton Drive. Interessant ist das Verhalten: Hot Storage blieb lokal verfügbar, Cold Reads waren langsamer aber planbar, und der Dienst erholte sich nach dem Ausfall.

## Was ins Testprotokoll gehört

Ein später nachvollziehbares Ergebnis enthält mindestens:

- Rclone-Version und verwendetes Backend
- Betriebssystem, FUSE-Variante und Dateisystem des Cache-Verzeichnisses
- vollständiger Mount-Befehl ohne Zugangsdaten
- Zahl, Grössenverteilung und Struktur der Testdateien
- Cold-Read- und Warm-Read-Werte für mehrere Dateien
- Schreibdauer bis zur Sichtbarkeit im Remote
- Cache-Spitzenwert und Dauer der Bereinigung
- Ergebnis von `rclone check --download`
- Verhalten bei Backend-Ausfall und beendetem Rclone-Prozess
- Wiederherstellungszeit aus Sicht der Anwendung
- Retries, Timeouts, Drosselungen und Authentifizierungsfehler aus dem Log

Definieren Sie für jeden Punkt vorab einen Grenzwert. Dann endet der Test mit einer Entscheidung und nicht nur mit einer Sammlung interessanter Zahlen.

## Wann der Aufbau bereit ist

Ein Cold-Storage-Mount ist einsatzbereit, wenn Sie diese Fragen mit Ja beantworten können:

- Sind Cold Reads für den vorgesehenen Dienst schnell genug?
- Beschleunigt der Cache wiederholte Zugriffe wie erwartet?
- Bleibt der lokale Platzbedarf auch unter Last kontrollierbar?
- Stimmen alle Dateien nach einem vollständigen Download?
- Funktionieren alle benötigten Dateioperationen mit dem gewählten Backend?
- Verhält sich die Anwendung bei einem Cloud-Ausfall kontrolliert?
- Werden Schreibvorgänge bei fehlendem Mount sicher gestoppt?
- Erreicht ein neu aufgebauter Mount alle laufenden Verbraucher?
- Zeigt das Monitoring den Ausfall, bevor ein Benutzer ihn meldet?

Wenn eine Antwort fehlt, wissen Sie immerhin genau, woran Sie weiterarbeiten müssen. Das ist wesentlich hilfreicher als ein Mount, der beim ersten `ls` gut aussah und erst im Betrieb seine Grenzen zeigt.

## Quellen

1.  [Rclone test makefiles](https://rclone.org/commands/rclone_test_makefiles/): reproduzierbare Testdateien und Verzeichnisstrukturen mit konfigurierbaren Grössen.

2.  [Rclone mount](https://rclone.org/commands/rclone_mount/): VFS-Cache-Modi, Writeback, Sparse Files, Cache-Limits und Verzeichnis-Cache.

3.  [Rclone check](https://rclone.org/commands/rclone_check/): Vergleich von Quelle und Ziel, einschliesslich vollständiger Prüfung mit `--download`.

4.  [Rclone Remote Control](https://rclone.org/rc/): gezieltes Verwerfen des VFS-Verzeichnis-Caches mit `vfs/forget`.

5.  [Rclone Global Flags](https://rclone.org/flags/): vollständige Referenz der globalen Optionen, darunter Logging, Statistiken und die VFS-Parameter.
