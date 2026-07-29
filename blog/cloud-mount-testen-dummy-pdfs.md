---
title: "Cold Storage mit Rclone testen: ein praxistauglicher Testplan"
navTitle: "Rclone testen"
description: "Bevor ein Dienst seine Dateien über einen Rclone-Mount aus der Cloud liest, solltest du mehr als den Verzeichniszugriff prüfen. Dieser Testplan deckt Cold Reads, Warm Reads, Schreibvorgänge, Cache-Verhalten, Dateiintegrität und Ausfälle ab."
date: "2026-07-26"
kategorie: "Rclone"
timeToRead: "11 Min. Lesezeit"
themen:
  - "rclone"
related:
  - "rclone-mount-in-docker-container"
  - "paperless-dokumente-clouddienst-auslagern"
slug: "cloud-mount-testen-dummy-pdfs"
translationId: "article-8592f808b2e93cd4"
url: "https://rafaelpfister.ch/blog/cloud-mount-testen-dummy-pdfs"
---

Ein Rclone-Mount ist schnell eingerichtet. Das Remote erscheint als Verzeichnis, `ls` zeigt Dateien und der erste Funktionstest ist bestanden. Für den produktiven Betrieb sagt das noch wenig aus.

Sobald ein Dienst auf den Mount zugreift, kommen weitere Fragen dazu: Wie lange dauert der erste Zugriff auf eine Datei? Welche Zugriffe bedient der lokale Cache? Was geschieht mit einer noch nicht hochgeladenen Datei, wenn Rclone abstürzt? Sieht ein laufender Container den neu aufgebauten Mount wieder? Und wie reagiert der Dienst, wenn die Cloud vorübergehend nicht erreichbar ist?

Dieser Artikel liefert dafür einen allgemeinen Testplan. Du kannst ihn für ein Dokumentenarchiv, einen Medienserver, eine Fotoverwaltung oder jeden anderen Dienst verwenden, der selten benötigte Dateien über Rclone aus einem Cold Storage bezieht.

## Zuerst festlegen, was du erreichen willst

Cold Storage bedeutet nicht automatisch dasselbe für jede Anwendung. Ein Medienserver liest grosse Dateien meist sequenziell. Eine Fotoverwaltung lädt viele kleine Vorschaudaten und springt an verschiedene Stellen. Ein Dokumentenarchiv öffnet vergleichsweise kleine Dateien, dafür aber oft nur einmal.

Notiere vor dem Test die wichtigsten Eigenschaften deines echten Bestands:

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

Passe Anzahl und Grössen an deinen echten Bestand an. Teste nicht nur Durchschnittsdateien. Einige sehr kleine Dateien zeigen, wie teuer Metadatenzugriffe sind; einige grosse Dateien machen Durchsatz, Read-Ahead und Cache-Verhalten sichtbar.

Ergänze ausserdem Dateinamen, die in der Praxis Probleme verursachen können:

```bash
mkdir -p "testdata/Sonderfälle/Unterordner"
printf 'Leerzeichen\n' > "testdata/Sonderfälle/Datei mit Leerzeichen.txt"
printf 'Umlaute\n' > "testdata/Sonderfälle/Grösse und Änderung.txt"
printf 'Grossschreibung\n' > "testdata/Sonderfälle/Test.txt"
printf 'Kleinschreibung\n' > "testdata/Sonderfälle/test.txt"
```

Der letzte Test ist besonders wichtig, wenn lokales Dateisystem und Cloud-Backend Gross- und Kleinschreibung unterschiedlich behandeln.

Wenn dein Dienst nur bestimmte Formate akzeptiert, reichen beliebige Binärdateien nicht. Erzeuge dann zusätzlich synthetische Dateien in genau diesen Formaten. Bei Paperless-ngx waren das PDFs mit echter Textebene, damit der Test nicht versehentlich die OCR-Leistung statt des Speicherpfads misst. Bei einer Fotoverwaltung gehören unterschiedliche Bildgrössen und Formate in den Bestand, bei einem Medienserver kurze Dateien mit verschiedenen Codecs.

## Eine Referenzmessung ohne Mount

Bevor FUSE und VFS-Cache ins Spiel kommen, solltest du das Backend direkt messen. Kopiere den Bestand mit Rclone ins Test-Remote und speichere ein ausführliches Protokoll:

```bash
rclone copy ./testdata remote:cold-storage-test \
  --progress \
  --stats 5s \
  --log-file rclone-copy.log \
  --log-level INFO
```

Prüfe danach, ob Quelle und Ziel übereinstimmen:

```bash
rclone check ./testdata remote:cold-storage-test \
  --one-way \
  --download
```

Mit `--download` liest Rclone die Daten tatsächlich und vergleicht sie, auch wenn das Backend keine passenden Hashes bereitstellt. Das dauert länger, liefert aber eine brauchbare Ausgangsbasis für den späteren Integritätstest.

Halte Upload-Zeit, Transferrate, Anzahl Retries und API-Fehler fest. Wenn bereits der direkte Zugriff instabil ist, kann der Mount das nicht reparieren.

## Den Test-Mount vom produktiven Cache trennen

Verwende für die Messung einen eigenen Mountpunkt und ein eigenes Cache-Verzeichnis:

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

Die Werte sind ein Beispiel und keine allgemeine Empfehlung. Entscheidend ist die Trennung: Ein leerer Test-Cache macht Cold Reads reproduzierbar, ohne dass du Dateien aus einem laufenden produktiven Cache löschen musst.

`--vfs-cache-mode full` ist für Anwendungen meist der aufschlussreichste Testmodus. Rclone puffert dabei Lese- und Schreibzugriffe lokal und kann Dateizugriffe besser abbilden, die bei einem reinen Objektspeicher nicht möglich wären. Die zusätzliche Kompatibilität kostet lokalen Speicherplatz.

## Immer aus Sicht des echten Dienstes prüfen

Ein Mount kann für deinen Benutzer funktionieren und für den Dienst trotzdem unbrauchbar sein. Häufige Ursachen sind eine andere Benutzer-ID, fehlendes `--allow-other`, Container-Grenzen oder eine falsche Mount-Propagation.

Führe deshalb mindestens einen vollständigen Lesezugriff mit derselben Identität aus, unter der später die Anwendung läuft:

```bash
sudo -u <service-user> sha256sum /mnt/rclone-test/pfad/zur/datei
```

Läuft der Dienst in Docker, gehört der Test in den Container:

```bash
docker exec --user <uid>:<gid> <app-container> \
  sha256sum /pfad/im/container/datei
```

Noch besser ist ein echter Anwendungstest. Öffne die Datei über die Weboberfläche oder API des Dienstes. Nur so bemerkst du, ob die Anwendung beispielsweise mehrere parallele Reads startet, an das Dateiende springt oder zusätzliche Metadaten erwartet.

## Cold Reads und Warm Reads getrennt messen

Bei `--vfs-cache-mode full` liegen zwischen Anwendung und Cloud drei Ebenen:

| Ebene | Was dort liegt |
|---|---|
| Remote | die vollständige Datei im Clouddienst |
| VFS-Cache | lokal gespeicherte Bereiche bereits gelesener Dateien |
| Linux-Page-Cache | kürzlich verwendete Daten im RAM |

Für einen Cold Read nimmst du eine Datei, deren Inhalt noch nie über den Test-Mount gelesen wurde. Beim direkt anschliessenden Warm Read liegt sie im VFS-Cache und meistens zusätzlich im RAM.

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

Miss nicht nur eine Datei. Verwende mindestens zehn bisher ungelesene Dateien verschiedener Grösse und notiere Median, langsamsten Wert und Dateigrösse. Ein einzelner Bestwert ist keine Entscheidungsgrundlage.

Ein Warm Read ist kein reiner Festplattentest, weil der Kernel Teile der Datei im RAM halten kann. Für die meisten Cold-Storage-Szenarien ist das kein Problem. Entscheidend ist, was ein Benutzer beim ersten und beim wiederholten Öffnen erlebt. Wenn du RAM und lokale Platte getrennt beurteilen willst, musst du den Page-Cache zusätzlich kontrollieren und nachweislich räumen.

## Nicht nur vollständige Lesezugriffe testen

`cat` liest eine Datei von Anfang bis Ende. Viele Anwendungen verhalten sich anders:

- Ein Videoplayer liest zunächst Header und Index, springt später an eine andere Position und lädt dann sequenziell weiter.
- Eine Bildverwaltung liest Metadaten und erzeugt anschliessend ein Vorschaubild.
- Ein Archivprogramm kann zuerst das Dateiende lesen.
- Mehrere Worker können gleichzeitig auf verschiedene Dateien zugreifen.

Teste diese Abläufe mit der tatsächlichen Anwendung. Beobachte parallel das Rclone-Protokoll und den Cache. Bei grossen Dateien ist interessant, wie viel Rclone wirklich lokal ablegt und ob `--vfs-read-ahead` zum Zugriffsmuster passt.

Ein Rclone-Mount ist ausserdem kein sinnvoller Speicherort für Datenbanken oder andere Dateien, die verlässliches Locking und häufige Änderungen innerhalb derselben Datei benötigen. Der VFS-Layer gleicht Unterschiede zwischen Dateisystem und Objektspeicher aus, macht aus dem Backend aber kein lokales Dateisystem.

## Den Schreibpfad separat abnehmen

Wenn dein Dienst nur liest, mounte das Remote nach Möglichkeit schreibgeschützt. Muss er schreiben, teste Erstellen, Überschreiben, Umbenennen und Löschen einzeln.

Eine geschriebene Datei erscheint nicht zwingend sofort im Remote. Bei aktivem VFS-Cache beginnt der Upload erst, nachdem die Datei geschlossen wurde und `--vfs-write-back` abgelaufen ist. Prüfe deshalb beide Zustände:

1. Die Anwendung hat die Datei erfolgreich geschlossen.
2. Die Datei ist anschliessend über einen direkten Rclone-Zugriff im Remote lesbar.

```bash
printf 'writeback-test\n' > /mnt/rclone-test/writeback-test.txt

# Nach Ablauf von --vfs-write-back:
rclone cat remote:cold-storage-test/writeback-test.txt
```

Wiederhole den Test mit einer grossen Datei und beende Rclone während des noch laufenden Uploads. Starte danach mit demselben Cache-Verzeichnis neu und kontrolliere, ob der Upload fortgesetzt wird. Genau dieses Zeitfenster entscheidet darüber, wie viele Daten bei einem Serverausfall gefährdet sind.

Teste auch Umbenennen und Löschen. Viele Cloud-Backends bilden diese Operationen anders ab als ein lokales Dateisystem. Relevant ist nicht nur, ob der Befehl erfolgreich endet, sondern wann die Änderung bei einem direkten Zugriff auf das Remote und bei weiteren Clients sichtbar wird.

## Änderungen ausserhalb des Mounts testen

Dateien können über die Weboberfläche des Anbieters, einen zweiten Rclone-Prozess oder einen anderen Server verändert werden. Der Mount sieht solche Änderungen nicht immer sofort, weil Verzeichnisinformationen zwischengespeichert werden.

Lege deshalb mit einem zweiten Rclone-Aufruf eine Datei direkt im Remote an:

```bash
printf 'external-change\n' > external-change.txt
rclone copyto external-change.txt \
  remote:cold-storage-test/external-change.txt
```

Miss, wann die Datei im Mount erscheint. Wiederhole den Test für Änderung und Löschung. Das Ergebnis hängt vom Backend, dessen Polling-Unterstützung sowie `--poll-interval` und `--dir-cache-time` ab. Wenn die Anwendung aktuelle Änderungen sofort sehen muss, gehört dieses Verhalten ausdrücklich in die Abnahmekriterien.

Bei aktivierter Remote-Control-Schnittstelle kannst du den Verzeichnis-Cache gezielt verwerfen:

```bash
rclone rc vfs/forget
```

Das ist nützlich für einen manuellen Test, ersetzt aber keine passende Betriebsstrategie.

## Den Cache unter Druck setzen

Ein fast leerer Cache ist der einfachste Fall. Setze `--vfs-cache-max-size` in einer zweiten Testrunde bewusst klein und lies mehr Daten, als hineinpassen.

```bash
du -sh /var/cache/rclone-test/vfs
du -sh --apparent-size /var/cache/rclone-test/vfs
find /var/cache/rclone-test/vfs -type f | wc -l
```

Die beiden Grössen können stark voneinander abweichen. Im Full-Modus verwendet Rclone Sparse Files: Eine Datei zeigt ihre vollständige logische Grösse, obwohl nur die gelesenen Bereiche lokalen Platz belegen.

Das Cache-Limit ist zudem weich. Rclone prüft es im Rhythmus von `--vfs-cache-poll-interval`, und geöffnete Dateien können nicht entfernt werden. Der Cache darf das Limit deshalb kurzzeitig überschreiten. Er sollte nach dem Schliessen der Dateien und dem nächsten Aufräumlauf aber wieder sinken.

Protokolliere Spitzenwert, Wert nach der Bereinigung und die dafür benötigte Zeit. So lässt sich der nötige lokale Speicher vernünftig dimensionieren.

## Zwei unterschiedliche Ausfälle simulieren

Eine nicht erreichbare Cloud und ein abgestürzter Rclone-Prozess sind zwei verschiedene Fehler:

| Ausfall | Was du damit prüfst |
|---|---|
| Backend oder Netzwerk nicht erreichbar, Rclone läuft weiter | Verhalten bei Retries, Timeouts und bereits gecachten Dateien |
| Rclone-Prozess beendet | Verhalten des FUSE-Mounts und Wiederherstellung des Mountpunkts |

Simuliere beides nur in der Testumgebung. Einen Rclone-Container kannst du für den zweiten Fall hart beenden:

```bash
docker kill --signal KILL <rclone-container>
```

Prüfe während des Ausfalls die Anwendung und nicht nur den Mountpunkt:

- Welche Funktionen bleiben verfügbar?
- Wie lange wartet ein Zugriff, bevor ein Fehler erscheint?
- Sind bereits vollständig gecachte Dateien noch erreichbar?
- Stoppt die Anwendung neue Schreibvorgänge?
- Entsteht eine verständliche Fehlermeldung oder nur ein hängender Prozess?
- Löst dein Monitoring aus?

Ein Schreibdienst darf bei fehlendem Mount nicht unbemerkt in das darunterliegende lokale Verzeichnis schreiben. Nach der Rückkehr des Mounts würden diese Dateien verdeckt. Ein einfacher Schutz vor jedem Schreibjob ist:

```bash
mountpoint -q /mnt/rclone-test || exit 1
```

Nach dem Neustart von Rclone prüfst du den Mount auf dem Host und aus jedem konsumierenden Container. Ein neu aufgebauter Mount erreicht einen bereits laufenden Container nur mit passender Mount-Propagation. Für Docker ist meistens `rslave` auf der konsumierenden Seite nötig. Die Details stehen im Artikel [Rclone-Mounts in Docker zuverlässig betreiben](/blog/rclone-mount-in-docker-container).

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

Definiere für jeden Punkt vorab einen Grenzwert. Dann endet der Test mit einer Entscheidung und nicht nur mit einer Sammlung interessanter Zahlen.

## Wann der Aufbau bereit ist

Ein Cold-Storage-Mount ist einsatzbereit, wenn du diese Fragen mit Ja beantworten kannst:

- Sind Cold Reads für den vorgesehenen Dienst schnell genug?
- Beschleunigt der Cache wiederholte Zugriffe wie erwartet?
- Bleibt der lokale Platzbedarf auch unter Last kontrollierbar?
- Stimmen alle Dateien nach einem vollständigen Download?
- Funktionieren alle benötigten Dateioperationen mit dem gewählten Backend?
- Verhält sich die Anwendung bei einem Cloud-Ausfall kontrolliert?
- Werden Schreibvorgänge bei fehlendem Mount sicher gestoppt?
- Erreicht ein neu aufgebauter Mount alle laufenden Verbraucher?
- Zeigt das Monitoring den Ausfall, bevor ein Benutzer ihn meldet?

Wenn eine Antwort fehlt, weisst du immerhin genau, woran du weiterarbeiten musst. Das ist wesentlich hilfreicher als ein Mount, der beim ersten `ls` gut aussah und erst im Betrieb seine Grenzen zeigt.

## Quellen

1.  [Rclone test makefiles](https://rclone.org/commands/rclone_test_makefiles/): reproduzierbare Testdateien und Verzeichnisstrukturen mit konfigurierbaren Grössen.

2.  [Rclone mount](https://rclone.org/commands/rclone_mount/): VFS-Cache-Modi, Writeback, Sparse Files, Cache-Limits und Verzeichnis-Cache.

3.  [Rclone check](https://rclone.org/commands/rclone_check/): Vergleich von Quelle und Ziel, einschliesslich vollständiger Prüfung mit `--download`.

4.  [Rclone Remote Control](https://rclone.org/rc/): gezieltes Verwerfen des VFS-Verzeichnis-Caches mit `vfs/forget`.
