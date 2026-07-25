---
title: "Paperless-Dokumente nach Proton Drive auslagern: ein Proof of Concept"
navTitle: "Paperless & Proton Drive"
description: "Lässt sich die Dokumentenablage von Paperless-ngx per rclone in Proton Drive verlagern und nur über einen lokalen Cache bedienen? Ein Testaufbau mit gemessenen Zugriffszeiten, dem Verhalten bei Cloud-Ausfall und vier Gründen, warum das noch kein Produktivbetrieb ist."
date: "2026-07-25"
kategorie: "Paperless-ngx"
timeToRead: "11 min to read"
themen:
  - "paperless-ngx"
draft: true
slug: "paperless-dokumente-proton-drive-auslagern"
url: "https://rafaelpfister.ch/blog/paperless-dokumente-proton-drive-auslagern"
aiPrompt: |
  Du bist mein Infrastruktur-Assistent. Hilf mir zu bewerten, ob ich die Dokumentenablage eines selbst betriebenen Dienstes in einen Cloud-Speicher auslagern kann:
  1. Kläre zuerst, welche Zugriffsmuster die Anwendung auf das Dateisystem hat (atomare Umbenennungen, Locking, wahlfreie Schreibzugriffe, viele kleine Metadaten-Zugriffe).
  2. Trenne heisse Daten (Datenbank, Suchindex, Vorschaubilder) von kalten Daten (fertig verarbeitete Originaldateien).
  3. Prüfe, ob der Cloud-Speicher über rclone mit VFS-Cache eingebunden werden kann und wie sich die Sitzung unbeaufsichtigt erneuert.
  4. Miss Kalt- und Warmzugriffszeiten sowie den lokalen Speicherbedarf des Caches unter Last.
  5. Teste ausdrücklich den Ausfall des Mounts und ob die Anwendung sich ohne Neustart erholt.
  6. Bewerte am Ende ehrlich, ob der Aufwand den gewonnenen Speicherplatz rechtfertigt oder ob eine Vergrösserung der lokalen Platte die bessere Lösung ist.
---

Paperless-ngx legt seine Dokumente in einem lokalen Verzeichnis ab, und dieses Verzeichnis wächst. Naheliegende Frage: Lassen sich die fertig verarbeiteten Dateien nicht in einen Cloud-Speicher verschieben und bei Bedarf über einen Cache zurückholen? Genau das habe ich mit Proton Drive und rclone aufgebaut und vermessen. Das Ergebnis vorweg: Es funktioniert technisch besser als erwartet, ist aber aus vier konkreten Gründen kein Produktivbetrieb.

## Die Idee: heisse und kalte Daten trennen

Der Ansatz stammt aus dem klassischen hierarchischen Storage-Management. Nach dem Einlesen greift Paperless im Normalbetrieb nicht mehr auf die PDF-Dateien zu. Die Suche läuft gegen den Suchindex und den in der Datenbank gespeicherten Text, die Dokumentliste zeigt Vorschaubilder. Die Originaldatei wird erst gelesen, wenn jemand ein konkretes Dokument öffnet oder herunterlädt.

Daraus ergibt sich eine saubere Trennung:

- **Lokal bleiben** Datenbank, Suchindex und Vorschaubilder. Diese Daten sind klein, werden ständig gelesen und verlangen echtes Locking.
- **Ausgelagert wird** das Verzeichnis mit den Originaldateien. Es macht den Grossteil des Volumens aus und wird selten angefasst.
- **Ein lokaler Cache** hält die zuletzt geöffneten Dokumente vor, damit wiederholte Zugriffe schnell bleiben.

Wichtig ist dabei: Die Dateien lassen sich nicht einfach wegkopieren. Paperless speichert die Pfade in der Datenbank und öffnet sie direkt. Verschwindet eine Datei, meldet die Integritätsprüfung sie als fehlend. Der Pfad muss also weiterhin auflösen, und genau das leistet ein transparenter Mount.

## Der Testaufbau

Als Basis diente ein kleiner Heimserver mit Ubuntu 25.10, Docker und einer 29-GB-SSD. Um die produktive Instanz nicht zu berühren, lief der Test in einer vollständig getrennten Paperless-Instanz mit eigener Datenbank, eigenen Volumes und eigenem Port.

Statt echter Dokumente kamen 40 generierte PDF-Dateien zum Einsatz, zusammen 13.9 MB bei 45 bis 596 KB pro Datei. Das hat zwei Gründe: Die Zugangsdaten des Cloud-Kontos liegen bei diesem Aufbau auf dem Server, und für die Messung von Übertragungszeiten ist die Grössenverteilung entscheidend, nicht der Inhalt.

Auf der Cloud-Seite kam ein **dediziertes Proton-Konto** zum Einsatz, nicht das Hauptkonto. Dazu später mehr, denn dieser Punkt entscheidet über die Machbarkeit.

Eingebunden wurde mit rclone 1.74.4 und dessen Proton-Drive-Backend:

```bash
rclone mount proton:paperless-poc/originals /pfad/media/documents/originals \
  --vfs-cache-mode full --vfs-cache-max-size 4M --vfs-cache-max-age 1h \
  --dir-cache-time 1h --allow-other --cache-dir /pfad/cache/originals --daemon
```

`--vfs-cache-mode full` ist der entscheidende Schalter: Er sorgt dafür, dass gelesene Dateien lokal zwischengespeichert werden und wahlfreie Zugriffe überhaupt möglich sind. Das Cache-Limit habe ich absichtlich auf 4 MB gesetzt, also deutlich unter die 13.9 MB Gesamtvolumen, damit die Verdrängung messbar wird. Damit der Container im FUSE-Mount lesen darf, muss auf dem Host `user_allow_other` in `/etc/fuse.conf` eingetragen sein.

## Die Messwerte

| Vorgang | Ergebnis |
|---|---|
| Upload nach Proton Drive (13.9 MiB, 40 Dateien) | 7 s |
| Erstes Lesen einer 596-KB-Datei (Cache leer) | 1765 ms |
| Wiederholtes Lesen derselben Datei (aus dem Cache) | 19–24 ms |
| Alle 40 Dateien kalt lesen | 58 s (⌀ 1468 ms pro Datei) |
| Dokumentliste (40 Einträge) | 39 ms |
| Volltextsuche | 272 ms |
| Lokaler Speicherbedarf vorher | 14 MB Originaldateien |
| Lokaler Speicherbedarf nachher | 5.3 MB Cache, dazu 3.2 MB Vorschaubilder |

Der Unterschied zwischen kaltem und warmem Zugriff beträgt etwa den Faktor 80. Ein Dokument aus der Cloud zu öffnen dauert rund eineinhalb bis zwei Sekunden, ein bereits gelesenes praktisch keine Zeit. Für ein Archiv, in dem einzelne Dokumente nachgeschlagen werden, ist das durchaus benutzbar.

## Was zuverlässig funktioniert hat

**Paperless liest die Dateien transparent.** Der Container sieht alle 40 Dateien im Mount, und die Anwendung öffnet sie über ihre normalen Pfade, ohne dass eine Anpassung nötig war.

**Die Integritätsprüfung besteht.** Der mitgelieferte Sanity-Checker berechnet die Prüfsumme jeder Datei und vergleicht sie mit der Datenbank. Er lief vollständig durch und meldete keine Abweichungen. Das ist der stärkste Beleg dafür, dass die Auslagerung sauber ist: Nicht nur einzelne Zugriffe klappen, sondern jede einzelne Datei kommt bitgenau aus der Cloud zurück.

**Suche und Oberfläche brauchen die Cloud nicht.** Der erkannte Text liegt in der Datenbank, im Test knapp 244'000 Zeichen für ein einzelnes Dokument. Liste, Suche und Vorschaubilder funktionierten deshalb auch dann noch, als ich den Mount komplett entfernt habe. Nur das Öffnen der Originaldatei schlug fehl. Die Weboberfläche blieb erreichbar.

**Die Verdrängung im Cache arbeitet.** Nach dem Vollzugriff hielt der Cache 24 Dateien, ältere Einträge wurden entfernt. Der lokale Fussabdruck sank von 14 MB auf 5.3 MB.

## Wo es bricht

### 1. Zwei-Faktor-Authentifizierung verhindert den unbeaufsichtigten Betrieb

Der erste Versuch lief über ein Konto mit aktivierter Zwei-Faktor-Authentifizierung. Upload und erste Zugriffe funktionierten, danach brach alles ab:

```text
422 POST https://drive-api.proton.me/auth/v4/2fa:
Ungültige Anmeldedaten (Code=8002)
```

Die Sitzung hielt etwa 35 Minuten. Beim notwendigen erneuten Anmelden schickt rclone den beim Einrichten eingegebenen Einmalcode noch einmal, und der ist längst verbraucht. Symptomatisch dafür waren wechselnde Verzeichnislisten: Erst waren 40 Dateien sichtbar, dann 17, dann keine mehr.

Es gibt zwei Auswege, und beide haben einen Preis. Man kann den dauerhaften TOTP-Seed in der rclone-Konfiguration hinterlegen, dann erzeugt rclone die Codes selbst. Damit liegen aber Passwort und zweiter Faktor am selben Ort, womit die Zwei-Faktor-Authentifizierung für dieses Konto praktisch aufgehoben ist. Oder man verzichtet für ein dediziertes Konto bewusst darauf, so wie in diesem Test. Für das Hauptkonto sollte man weder das eine noch das andere tun.

### 2. Das Cache-Limit ist keine harte Grenze

Konfiguriert waren 4 MB. Während des Vollzugriffs wuchs der Cache laut Protokoll auf 12.7 MiB an, bevor der Aufräumlauf ihn auf 3.5 MiB zurückschnitt:

```text
vfs cache: cleaned: objects 9 (was 38) in use 1, total size 3.471Mi (was 12.698Mi)
```

Die Bereinigung läuft in einem Intervall von etwa einer Minute. Bei einem Zugriffsschub kann der Cache also kurzzeitig fast den gesamten ausgelagerten Bestand lokal halten. Wer den Speicherplatz knapp kalkuliert, muss diesen Spitzenwert einplanen, nicht das konfigurierte Limit.

### 3. Routineaufgaben ziehen den gesamten Bestand durchs Netz

Der Sanity-Checker prüft Prüfsummen und liest dafür jede Datei vollständig. Im Test dauerte das 58 Sekunden für 13.9 MB. Rechnet man das auf ein reales Archiv von 10 GB hoch, wird daraus ein Vorgang von mehreren Stunden mit entsprechendem Datenvolumen. Dasselbe gilt für den Dokumenten-Export, für ein erneutes Erzeugen der Vorschaubilder und für eine erneute Texterkennung.

Wer so einen Aufbau betreibt, muss diese geplanten Aufgaben deshalb bewusst abschalten oder stark strecken. Sonst arbeitet im Hintergrund regelmässig ein Prozess, der die Auslagerung vollständig rückgängig macht.

### 4. Nach einem Aussetzer erholt sich der Container nicht selbst

Das ist der gravierendste Punkt. Ich habe den Mount entfernt, um einen Cloud-Ausfall zu simulieren, und ihn danach wieder eingerichtet. Auf dem Host waren alle 40 Dateien sofort wieder da. Im Container nicht:

```text
ls: cannot access '/usr/src/paperless/media/documents/originals':
Transport endpoint is not connected
```

Der Container hält an dem abgestorbenen Mount fest. Erst ein Neustart des Containers, der in diesem Aufbau 95 Sekunden bis zum gesunden Zustand brauchte, stellte den Zugriff wieder her. Übertragen heisst das: Jede abgelaufene Sitzung, jeder Netzwerkaussetzer und jeder Neustart des Mounts erfordert einen manuellen Eingriff. Ohne eine Überwachung, die den Mount prüft und den Container automatisch neu startet, ist das nicht betriebstauglich.

## Zwei Nebenbefunde

**Exporte sind versionsgebunden.** Der Versuch, einen mit Version 2.20.3 erzeugten Export in eine 3.0.2-Instanz einzulesen, scheiterte mit `SavedView has no field named 'show_on_dashboard'`. Ein Export gehört also zur Version, mit der er erstellt wurde. Für ein Backup bedeutet das: nach einem Upgrade neu exportieren.

**Die Archivdateien fehlen in diesem Test.** Die generierten PDF-Dateien enthielten bereits eine Textebene, weshalb Paperless im Modus `skip` keine Archivversion erzeugt hat. Bei echten Scans ist dieses Verzeichnis oft grösser als das der Originale. Das ausgelagerte Volumen wäre im realen Betrieb also deutlich höher als hier gemessen, das Verhalten aber identisch.

## Fazit

Als Proof of Concept ist die Frage beantwortet: **Ja, es funktioniert.** Paperless-ngx läuft mit einer Dokumentenablage in Proton Drive, liefert Dokumente transparent aus, besteht seine eigene Integritätsprüfung, und Suche wie Oberfläche bleiben von der Cloud unabhängig. Der Preis sind knapp zwei Sekunden beim ersten Öffnen eines Dokuments, was für ein Archiv vertretbar ist.

Für den Produktivbetrieb reicht das nicht. Die fehlende Selbstheilung nach einem Mount-Ausfall wäre der Punkt, an dem man am Wochenende feststellt, dass seit Tagen kein Dokument mehr geöffnet werden kann. Dazu kommt eine Authentifizierung, die entweder unbeaufsichtigt scheitert oder den zweiten Faktor entwertet, und ein Backend, das der Hersteller ausdrücklich als Beta führt.

Der ehrlichere Schluss betrifft aber die Ausgangsfrage. In dem Fall, der diesen Test angestossen hat, belegte die Dokumentenablage 71 MB, während das Systemprotokoll 2.9 GB und alte Paketversionen 1.4 GB beanspruchten. Das Speicherproblem lag gar nicht bei den Dokumenten. Wer wirklich Platz braucht, fährt mit einer grösseren Platte oder einem Speicher im lokalen Netz besser: Beides bietet ein echtes Dateisystem mit niedrigen Latenzen, statt ein solches über eine Objektschnittstelle nachzubilden.

Proton Drive bleibt für diesen Zweck sehr wohl sinnvoll, nur an anderer Stelle: als verschlüsseltes Ziel für ein regelmässiges Backup des Dokumenten-Exports. Dafür ist die Schnittstelle gebaut, dort spielt Latenz keine Rolle, und ein abgelaufenes Token verhindert dann höchstens einen Backup-Lauf, nicht den Zugriff auf das Archiv.

## Quellen

1.  [rclone: Proton Drive](https://rclone.org/protondrive/) — Dokumentation des Backends samt Optionen für Zwei-Faktor-Authentifizierung und dem Hinweis auf den Beta-Status.

2.  [rclone mount](https://rclone.org/commands/rclone_mount/) — Beschreibung der VFS-Cache-Modi, der Grössen- und Altersgrenzen sowie ihrer Einschränkungen.

3.  [Paperless-ngx: Administration](https://docs.paperless-ngx.com/administration/) — Sanity-Checker, Export und Import sowie die geplanten Hintergrundaufgaben.

4.  [Paperless-ngx: Releases](https://github.com/paperless-ngx/paperless-ngx/releases) — Versionsstand 3.0.2 und die Breaking Changes der Version 3.0.

5.  [Proton Drive SDK preview](https://proton.me/blog/proton-drive-sdk-preview) — Stand des offiziellen SDK, das auf Dateitransfer ausgelegt ist und laut Proton noch nicht für Drittanbieter freigegeben ist.
