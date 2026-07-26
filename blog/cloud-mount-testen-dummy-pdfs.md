---
title: "Cloud-Mounts realistisch testen: PDFs erzeugen, Zugriffe messen, Ausfälle simulieren"
navTitle: "Cloud-Mounts testen"
description: "Eine nachvollziehbare Testmethode für Cloud-Mounts unter Paperless-ngx: reproduzierbare PDF-Dateien, getrennte Cache-Ebenen, Integritätsprüfungen und harte Ausfalltests."
date: "2026-07-26"
kategorie: "Paperless-ngx"
timeToRead: "8 Min. Lesezeit"
themen:
  - "paperless-ngx"
related:
  - "paperless-dokumente-proton-drive-auslagern"
draft: true
slug: "cloud-mount-testen-dummy-pdfs"
url: "https://rafaelpfister.ch/blog/cloud-mount-testen-dummy-pdfs"
---

Ein Cloud-Mount ist erst dann einsatzbereit, wenn nicht nur `ls` funktioniert. Entscheidend ist, wie lange ein kalter Zugriff dauert, was die Anwendung bei einem Ausfall noch leisten kann und ob jede Datei unverändert zurückkommt.

Die folgende Methode entstand beim [Test einer Cloud-Ablage für Paperless-ngx](/blog/paperless-dokumente-proton-drive-auslagern). Sie verwendet künstliche Dokumente, misst auf Anwendungs- statt nur auf Dateisystemebene und prüft die Wiederherstellung nach einem hart unterbrochenen Mount. Die Werte stammen von einer konkreten Maschine und zeigen Grössenordnungen, keine allgemein gültigen Benchmarks.

## Testdokumente generieren statt echte hochladen

Private Dokumente gehören nicht in eine Testumgebung. Für Übertragungs- und Cache-Messungen zählen Dateigrösse, Seitenzahl und Textebene; der Inhalt ist unwichtig.

Das folgende Skript schreibt PDFs ohne zusätzliche Bibliothek direkt in PDF-Syntax. Jede Seite erhält Zufallstext als Content-Stream; über die Seitenzahl lässt sich die Dateigrösse steuern. Der **feste Seed** macht die Verteilung reproduzierbar. Eine **echte Textebene** verhindert, dass die Dokumentenverwaltung eine Texterkennung startet: Gemessen werden soll der Speicherpfad, nicht Tesseract.

```python
import os, random
random.seed(42)                      # fester Seed: reproduzierbare Verteilung
outdir = "dummy"
os.makedirs(outdir, exist_ok=True)
WORDS = ("Rechnung Beleg Vertrag Kunde Betrag Datum Position Menge Preis "
         "Summe Steuer Netto Brutto Lieferung Zahlung Konto Referenz").split()

def build(idx, npages):
    objects, kids, objnum = {}, [], 4
    for p in range(1, npages + 1):
        lines = ["Testdokument %03d - Seite %d von %d" % (idx, p, npages)]
        for _ in range(45):
            lines.append(" ".join(random.choice(WORDS) for _ in range(11))
                         + " %d" % random.randint(100, 99999))
        parts = ["BT /F1 11 Tf 40 800 Td 14 TL"]
        parts += ["(%s) Tj T*" % l for l in lines]
        parts.append("ET")
        stream = "\n".join(parts).encode("latin-1")
        cnum, pnum = objnum, objnum + 1
        objnum += 2
        objects[cnum] = (b"<< /Length " + str(len(stream)).encode()
                         + b" >>\nstream\n" + stream + b"\nendstream")
        objects[pnum] = ("<< /Type /Page /Parent 2 0 R /MediaBox [0 0 595 842] "
                         "/Resources << /Font << /F1 3 0 R >> >> "
                         "/Contents %d 0 R >>" % cnum).encode()
        kids.append(pnum)
    objects[1] = b"<< /Type /Catalog /Pages 2 0 R >>"
    objects[2] = ("<< /Type /Pages /Kids [%s] /Count %d >>"
                  % (" ".join("%d 0 R" % k for k in kids), npages)).encode()
    objects[3] = b"<< /Type /Font /Subtype /Type1 /BaseFont /Helvetica >>"
    mx = max(objects)
    buf, off = bytearray(b"%PDF-1.4\n"), {}
    for n in range(1, mx + 1):
        off[n] = len(buf)
        buf += ("%d 0 obj\n" % n).encode() + objects[n] + b"\nendobj\n"
    x = len(buf)
    buf += ("xref\n0 %d\n" % (mx + 1)).encode() + b"0000000000 65535 f \n"
    for n in range(1, mx + 1):
        buf += ("%010d 00000 n \n" % off[n]).encode()
    buf += ("trailer\n<< /Size %d /Root 1 0 R >>\nstartxref\n%d\n%%%%EOF\n"
            % (mx + 1, x)).encode()
    return bytes(buf)

for i in range(1, 41):
    pages = random.choice([10, 25, 40, 60, 90, 130])
    open(os.path.join(outdir, "dummy-%03d.pdf" % i), "wb").write(build(i, pages))
```

Heraus kommen 40 Dateien mit 45 bis 596 KB, zusammen 13.9 MB. Das ist eine realistische Verteilung für ein privates Dokumentenarchiv. Paperless nahm sie in gut 8 Sekunden pro Dokument auf, im Protokoll bestätigt `pdftotext exited 0` die funktionierende Textebene.

Die Testinstanz selbst lief vollständig getrennt von der Produktion: eigene Datenbank, eigene Volumes, eigener Port. Ein Detail, das Zeit kostete: Bei `postgres:18` muss das Volume auf `/var/lib/postgresql` zeigen, nicht mehr auf `/var/lib/postgresql/data`. Sonst startet der Container in einer Endlosschleife.

## Wo in diesem System kalt und warm liegen

Bevor man misst, muss klar sein, **welche Cache-Ebenen** zwischen Anwendung und Cloud liegen. Sonst misst man etwas anderes als behauptet. Bei einem rclone-Mount mit `--vfs-cache-mode full` sind es drei:

| Ebene | Ort | Was dort liegt |
|---|---|---|
| Cloud | beim Anbieter | die einzige vollständige Wahrheit |
| VFS-Cache | lokale Platte (`--cache-dir`) | Kopien zuletzt gelesener Dateien, begrenzt über `--vfs-cache-max-size` |
| Page-Cache | RAM | wird transparent vom Kernel gehalten, sowohl für die Datei, wie sie durch den FUSE-Mount gelesen wird, als auch für die VFS-Kopie selbst |

**Kalt** heisst in diesem Setup: nicht im VFS-Cache, die Datei kommt übers Netz. **Warm** heisst: Die VFS-Kopie liegt lokal und beim wiederholten Lesen praktisch immer zusätzlich im RAM. Die einfache Zweipunktmessung erfasst also die Grenze Netz/lokal:

```bash
S=$(date +%s%3N); cat "$D/$F" > /dev/null; echo "kalt: $(( $(date +%s%3N) - S )) ms"
S=$(date +%s%3N); cat "$D/$F" > /dev/null; echo "warm: $(( $(date +%s%3N) - S )) ms"
```

Im Test: 1765 ms kalt gegen 19 bis 24 ms warm. Diese Grenze ist für die Cloud-Frage auch die relevante. Genau genommen bedeutet „warm" hier allerdings „lokal, RAM-unterstützt" und nicht „von der Platte".

## Messhygiene: Page-Cache räumen und das Räumen beweisen

Wer die lokalen Ebenen trennen will (RAM gegen Platte), muss den Page-Cache vor der Messung leeren und das **nachweisen**. Der übliche Weg ist `sync` gefolgt von `vmtouch -e`, die Kontrolle übernimmt `fincore` aus util-linux:

```bash
sync
vmtouch -e datei.pdf          # Seiten dieser Datei aus dem RAM werfen
fincore datei.pdf             # Beweis: RES muss 0 sein
```

Ist `vmtouch` nicht installiert, tut es ein Dreizeiler ohne Zusatzpaket:

```python
import os
fd = os.open("datei.pdf", os.O_RDONLY)
os.posix_fadvise(fd, 0, 0, os.POSIX_FADV_DONTNEED)
os.close(fd)
```

Zwei Eigenheiten dieses Setups machen den Nachweis zur Pflicht statt zur Kür:

**Die Eviction muss alle Pfade zur selben Datei treffen**: die Datei, wie der Mount sie zeigt, *und* die VFS-Kopie unter `--cache-dir`. Diese liegt im Unterverzeichnis `vfs/`; `vfsMeta/` enthält nur Metadaten.

**Und sie kann stillschweigend scheitern.** In meinem Kontrollversuch zeigte `fincore` die VFS-Kopie nach `sync` und `POSIX_FADV_DONTNEED` unverändert vollständig im RAM (600K resident, vorher wie nachher): rclone hält die Cache-Datei offen, und der Kernel gab die Seiten nicht frei. Ohne den `fincore`-Beweis hätte ich eine „von der Platte"-Zahl veröffentlicht, die in Wahrheit aus dem RAM kam. Wer die Platten-Ebene wirklich isolieren will, muss den rclone-Prozess für die Messung beenden oder als root `echo 3 > /proc/sys/vm/drop_caches` verwenden.

Die Einordnung dazu: Bei typischen Dokumentgrössen lohnt die Trennung kaum. Heisse Lesungen durch den Mount lagen bei mir bei 19 bis 35 ms, weil die FUSE-Umwege dominieren, nicht das Speichermedium. Die Ebene, die das Nutzererlebnis bestimmt, bleibt Netz gegen lokal mit Faktor 60 bis 90. Aber behaupten darf man das erst, wenn die Messung die Ebenen nachweislich auseinanderhält.

Dasselbe gilt übrigens für die generierten Testdateien: Direkt nach dem Erzeugen liegen sie vollständig im Page-Cache. Für die Upload-Messung ist das egal, weil das Netz der Engpass ist. Wer aber lokale Lesewerte über sie erheben will, muss zuerst räumen.

Zur laufenden Beobachtung gehört ausserdem der Blick auf den VFS-Cache nach jedem Schritt (`du -sh` auf das Cache-Verzeichnis, `find … -type f | wc -l`) und das rclone-Protokoll über `--log-file` samt `--log-level INFO`, in dem das Verdrängungsverhalten steht. Dort zeigte sich auch, dass das konfigurierte Cache-Limit weich ist: 4 MB konfiguriert, 12.7 MiB Spitzenwert, bis der minütliche Aufräumlauf griff.

## Auf Anwendungsebene prüfen, nicht nur im Dateisystem

Dass `ls` Dateien zeigt, heisst nicht, dass die Anwendung sie lesen kann. Benutzerrechte auf FUSE-Mounts sind ein eigenes Thema. Deshalb sollte der Zugriff durch die Anwendung selbst geprüft werden, bei Django-Anwendungen wie Paperless über eine Shell im Container:

```bash
docker compose exec webserver python3 -c "
import os; os.environ.setdefault('DJANGO_SETTINGS_MODULE','paperless.settings')
import django; django.setup()
from documents.models import Document
d = Document.objects.first()
print(os.path.exists(d.source_path), len(open(d.source_path,'rb').read()))
"
```

Die härteste Prüfung liefert die Anwendung gleich mit: Paperless' `document_sanity_checker` liest jede Datei vollständig und vergleicht die Prüfsumme mit der Datenbank. „No issues detected" nach einem Lauf über den Cloud-Mount belegt, dass jede Datei bitgenau zurückkommt. Für den Schreibpfad gilt eine Eigenheit: Die Testdatei muss **einzigartig** sein, eine Kopie eines vorhandenen Dokuments weist Paperless als Duplikat ab.

## Ausfälle hart simulieren und die Wiederherstellung beweisen

Ein Ausfalltest, der nur den Dienst sauber stoppt, testet nichts. Hart heisst: den rclone-Prozess töten, den Mount aushängen, und dann vergleichen, was Host und Anwendung jeweils sehen:

```bash
pkill -f "rclone mount" ; fusermount3 -u /pfad/zum/mount
docker compose exec webserver ls /usr/src/paperless/media/documents/originals
```

Zwei Dinge haben sich dabei als messbar wichtig erwiesen. Erstens der Unterschied zwischen den Perspektiven: Nach einem neu aufgebauten Mount sah der Host sofort wieder alles, der Container blieb ohne die richtige Mount-Propagation auf `Transport endpoint is not connected` hängen. Die Details dazu stehen im [Artikel zu rclone in Docker](/blog/rclone-mount-in-docker-container). Zweitens der **Neustartzähler als Beweismittel**: `docker inspect -f 'restarts={{.RestartCount}}'` vor und nach dem Test belegt, ob sich ein Container wirklich ohne Neustart erholt hat oder ob Docker still nachgeholfen hat.

Ebenso wichtig: prüfen, was **ohne** Cloud noch funktioniert. Bei entferntem Mount liefen Dokumentliste, Volltextsuche und Vorschaubilder unverändert. Der erkannte Text lag mit knapp 244'000 Zeichen für ein Testdokument in der Datenbank. Nur das Öffnen der Originaldatei schlug fehl. Solche Negativproben gehören in jedes Testprotokoll, weil sie zeigen, welcher Schaden ein Ausfall wirklich anrichtet.

## Protokolle mit präzisen Mustern auswerten

Bei der Auswertung müssen Suchmuster genau genug sein. Eine Suche nach `422` lieferte in meinem Test 39 vermeintliche Fehler. Tatsächlich stammten die Treffer aus harmlosen Grössenangaben wie `422504 bytes`. Ein Wortgrenzen-Muster (`grep -E '\b422\b'`) oder die Suche nach der vollständigen Fehlermeldung vermeidet solche Fehlalarme.

## Quellen

1.  [Paperless-ngx: Administration](https://docs.paperless-ngx.com/administration/): der Sanity-Checker und die übrigen Verwaltungsbefehle, die hier als Prüfwerkzeuge dienten.

2.  [rclone mount](https://rclone.org/commands/rclone_mount/): die VFS-Cache-Modi und Protokolloptionen, deren Verhalten hier vermessen wurde.

3.  [PDF 1.7 Referenz (Adobe)](https://opensource.adobe.com/dc-acrobat-sdk-docs/pdfstandards/PDF32000_2008.pdf): die Struktur aus Objekten, Content-Streams und xref-Tabelle, die der Generator direkt schreibt.
