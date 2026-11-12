---
title: "SQLite: «database is locked» verstehen und verhindern"
navTitle: "SQLite-Sperren"
description: "Warum SQLite mit SQLITE_BUSY und «database is locked» antwortet, wie WAL-Modus, busy_timeout, kurze Transaktionen und Retries Sperren vermeiden und weshalb langsame Datenträger und I/O-Last auf einer NAS das Problem erst auslösen. Mit einem Praxisfall aus Grafana."
date: "2026-11-12"
kategorie: "Datenbanken"
timeToRead: "11 Min. Lesezeit"
themen:
  - "datenbanken"
produkte:
  - "sqlite"
protokolle:
  - "troubleshooting"
  - "container"
slug: "sqlite-database-locked-wal-io-last"
translationId: "article-2a328dabf9164dae"
url: "https://rafaelpfister.ch/blog/sqlite-database-locked-wal-io-last"
---

SQLite steckt in vielen selbst betriebenen Diensten: Grafana, Home Assistant, Vaultwarden, Uptime Kuma, Jellyfin und zahlreiche weitere Anwendungen legen ihre Daten standardmässig in einer einzelnen Datei ab. Das funktioniert, bis im Log die Meldung `database is locked` erscheint. Sie meldet, dass ein Zugriff innerhalb der erlaubten Wartezeit keine Sperre bekommen hat; die Datenbankdatei selbst ist dabei intakt. Ob das selten oder dauernd passiert, hängt von drei Dingen ab: dem Journal-Modus, dem Verhalten der Anwendung bei Konflikten und der Latenz des Datenträgers.

## Praxisfall: Grafana auf einer NAS

Grafana lief als Docker-Container auf einer Synology-NAS, die SQLite-Datei lag in einem Named Volume. Auf derselben NAS liefen Prometheus, Loki, Paperless-ngx und mehrere restic-Sicherungen. Die Datenträger waren Festplatten mit Shingled Magnetic Recording (SMR).

Zuerst meldete der Health-Endpoint von Grafana `"database": "failing"`. Im Log standen in kurzen Abständen Einträge dieser Art:

```text
level=error msg="Failed to save history batch"
  error="database is locked (5) (SQLITE_BUSY)"
```

Die Meldung stammt aus der Alert-State-History: Grafana schreibt Zustandswechsel von Alert-Regeln standardmässig als Annotationen in die eigene Datenbank. Bei vielen Regeln entstehen so laufend kleine Schreibtransaktionen. Nach einem Neustart des Containers kam Grafana nicht mehr hoch und startete in einer Schleife neu:

```text
Failed to stop all modules ... kv get:
  [sqlstore.max-retries-reached] retry 1: database is locked
```

Behoben wurde der Fall mit zwei Einstellungen in der `docker-compose.yml`: WAL-Modus aktivieren und Grafana bei Sperren mehrfach wiederholen lassen. Warum genau diese beiden Werte wirken, ergibt sich aus der Funktionsweise der SQLite-Sperren.

## Wie SQLite sperrt

SQLite hat keinen Serverprozess, der Zugriffe koordiniert. Jede Verbindung öffnet die Datei selbst und regelt den gemeinsamen Zugriff über Dateisperren des Betriebssystems. Daraus folgt die wichtigste Eigenschaft: **Zu jedem Zeitpunkt darf nur eine Verbindung schreiben.** Das gilt in allen Journal-Modi.

Im klassischen Rollback-Journal-Modus (Standard, `journal_mode=DELETE`) durchläuft eine Schreibtransaktion mehrere Sperrstufen:

| Sperre | Bedeutung |
|---|---|
| `SHARED` | Die Verbindung liest. Beliebig viele Leser gleichzeitig sind erlaubt. |
| `RESERVED` | Die Verbindung will schreiben und bereitet Änderungen vor. Leser dürfen weiterlesen, ein zweiter Schreiber nicht. |
| `PENDING` | Die Verbindung wartet auf das Ende der laufenden Lesezugriffe. Neue Leser werden abgewiesen. |
| `EXCLUSIVE` | Die Verbindung schreibt in die Datenbankdatei. Niemand sonst darf lesen oder schreiben. |

Für den Commit braucht der Schreiber also die exklusive Sperre, und die bekommt er erst, wenn kein Leser mehr aktiv ist. Ein lang laufender Lesezugriff hält damit jeden Schreiber auf, und während des Commits sind alle Leser blockiert.

Bekommt eine Verbindung die benötigte Sperre nicht, liefert SQLite den Fehlercode `SQLITE_BUSY` (5) mit dem Text `database is locked`. Davon zu unterscheiden ist `SQLITE_LOCKED` (6, `database table is locked`): Dieser Code betrifft Konflikte innerhalb derselben Verbindung oder im Shared-Cache-Modus und hat andere Ursachen.

Entscheidend ist, wie lange SQLite vor dieser Meldung wartet. Ohne Konfiguration wartet SQLite überhaupt nicht: Der Standardwert von `busy_timeout` ist 0, die Meldung kommt sofort, sobald eine andere Verbindung die Sperre hält. Viele Bibliotheken setzen einen eigenen Wert; Pythons Modul `sqlite3` wartet beispielsweise 5 Sekunden.

## WAL-Modus: Leser und Schreiber trennen

Im Write-Ahead-Log-Modus (`journal_mode=WAL`) hängt SQLite Änderungen an eine separate Datei `<name>-wal` an, statt sie direkt in die Datenbankdatei zu schreiben. Leser sehen den Stand zum Beginn ihrer Lesetransaktion und lesen dafür aus Datenbankdatei und WAL. Daraus ergeben sich zwei Vorteile:

- Leser blockieren den Schreiber nicht, und der Schreiber blockiert die Leser nicht.
- Ein Commit ist ein Anhängen an eine Datei statt mehrerer verstreuter Schreibvorgänge mit Journal. Pro Transaktion fällt weniger I/O an, und die Schreibsperre wird kürzer gehalten.

Der WAL-Modus ändert nichts daran, dass nur ein Schreiber gleichzeitig aktiv sein kann. Er verkürzt aber die Zeit, in der die Sperre gehalten wird, und beseitigt die Konflikte zwischen Lesen und Schreiben. Bei einer Anwendung wie Grafana, die gleichzeitig Dashboards ausliefert und Alert-Zustände speichert, ist das die wirksamste Massnahme.

Der Modus wird einmal gesetzt und bleibt in der Datei gespeichert:

```sql
PRAGMA journal_mode = WAL;
PRAGMA busy_timeout = 5000;
PRAGMA synchronous = NORMAL;
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `journal_mode = WAL` | Schaltet auf Write-Ahead-Log um. Die Einstellung ist persistent; neue Verbindungen öffnen die Datei automatisch im WAL-Modus. |
| `busy_timeout = 5000` | Wartezeit in Millisekunden, in der SQLite bei einer belegten Sperre wiederholt versucht, sie zu bekommen. Gilt nur für die aktuelle Verbindung und muss bei jedem Öffnen neu gesetzt werden. |
| `synchronous = NORMAL` | Im WAL-Modus kein `fsync` pro Commit, sondern nur bei Checkpoints. Die Datenbank bleibt bei Stromausfall konsistent; die zuletzt bestätigten Transaktionen können verloren gehen. Gilt pro Verbindung. |

</details>

Der WAL-Modus hat Voraussetzungen. Alle Prozesse, die auf die Datei zugreifen, müssen auf demselben Rechner laufen, weil sie sich über eine Shared-Memory-Datei `<name>-shm` abstimmen. Das Verzeichnis braucht Schreibrechte für die beiden Zusatzdateien, auch für Prozesse, die nur lesen. Und beim Sichern müssen die Dateien zusammen betrachtet werden (dazu weiter unten mehr).

## Checkpoints: wenn das WAL wächst

Das WAL wächst, bis ein Checkpoint die Änderungen in die eigentliche Datenbankdatei überträgt. SQLite löst ihn automatisch aus, sobald das WAL rund 1000 Seiten umfasst (`wal_autocheckpoint`, bei 4-KiB-Seiten etwa 4 MB). Der Checkpoint schreibt die Seiten an ihre Position in der Datenbankdatei und ruft danach `fsync` auf. Auf einem langsamen Datenträger dauert genau dieser Schritt am längsten.

Ein Checkpoint kann Seiten nur übertragen, solange kein Leser einen älteren Stand benötigt. Läuft dauernd mindestens eine Lesetransaktion, bleibt der Checkpoint unvollständig und das WAL wächst weiter. Ein WAL von mehreren hundert Megabyte ist ein Hinweis auf lang laufende Lesetransaktionen oder auf Checkpoints, die wegen I/O-Last nicht fertig werden. Prüfen lässt sich das so:

```sql
PRAGMA wal_checkpoint(TRUNCATE);
```

Die Ausgabe besteht aus drei Zahlen: ob der Checkpoint blockiert wurde (`1` = ja), wie viele Seiten im WAL stehen und wie viele davon übertragen wurden. Mit `TRUNCATE` wird das WAL danach auf null Byte gekürzt, sofern der Checkpoint vollständig war.

## Busy-Timeout und Retries

Mit einem Busy-Timeout wartet SQLite bei einem Konflikt, statt sofort abzubrechen. Der Fehler erscheint nur noch, wenn die Sperre länger gehalten wird als die eingestellte Zeit. Für Anwendungen mit wenigen, kurzen Schreibvorgängen reichen 5 bis 10 Sekunden.

Es gibt einen Fall, in dem der Busy-Timeout nicht greift. Beginnt eine Transaktion lesend (`BEGIN` ohne Zusatz, also `DEFERRED`) und will später schreiben, während ein anderer Schreiber inzwischen einen Commit abgeschlossen hat, liefert SQLite sofort `SQLITE_BUSY`. Die Transaktion arbeitet auf einem veralteten Stand und kann nicht zur Schreibtransaktion werden; Warten würde daran nichts ändern. Im Rollback-Modus gibt es eine vergleichbare Konstellation, wenn zwei Verbindungen gleichzeitig von `SHARED` auf eine Schreibsperre wechseln wollen.

Die Abhilfe ist, Schreibtransaktionen von Anfang an als solche zu kennzeichnen:

```sql
BEGIN IMMEDIATE;
UPDATE alert_state SET state = 'firing' WHERE id = 42;
COMMIT;
```

`BEGIN IMMEDIATE` fordert die Schreibsperre beim Start an. Ist sie belegt, greift der Busy-Timeout regulär. In Python sieht das so aus:

```python
import sqlite3

con = sqlite3.connect("app.db", timeout=10.0,
                      isolation_level=None)
con.execute("PRAGMA journal_mode = WAL")
con.execute("BEGIN IMMEDIATE")
con.execute("UPDATE jobs SET status = ? WHERE id = ?",
            ("done", 42))
con.execute("COMMIT")
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `"app.db"` | Pfad zur Datenbankdatei. |
| `timeout=10.0` | Busy-Timeout in Sekunden; entspricht `PRAGMA busy_timeout = 10000`. |
| `isolation_level=None` | Schaltet die automatische Transaktionssteuerung des Moduls ab, damit `BEGIN IMMEDIATE` und `COMMIT` explizit gesetzt werden können. |

</details>

Retries auf Anwendungsebene ergänzen den Busy-Timeout. Sie fangen `SQLITE_BUSY` ab, warten kurz und führen die ganze Transaktion erneut aus. Das deckt auch den Fall mit veraltetem Lesestand ab, weil die Wiederholung mit einem neuen Stand beginnt. Grafana bietet dafür eigene Einstellungen (siehe Abschnitt zur Lösung).

## Kurze Transaktionen und ein Schreiber

Die Dauer, für die eine Schreibsperre gehalten wird, bestimmt die Anwendung. Drei Regeln halten sie kurz:

- **Keine Arbeit innerhalb der Transaktion, die nichts mit der Datenbank zu tun hat.** HTTP-Aufrufe, Dateizugriffe oder Berechnungen gehören vor `BEGIN` oder nach `COMMIT`.
- **Viele kleine Schreibvorgänge bündeln.** 500 Einzel-Commits kosten 500 Sperrzyklen und im Rollback-Modus 500 Mal mehrere `fsync`-Aufrufe. Eine Transaktion mit 500 Zeilen hält die Sperre einmal und etwas länger.
- **Lesecursor schliessen.** Ein nicht vollständig abgerufenes Abfrageergebnis hält die Lesetransaktion offen und blockiert im WAL-Modus den Checkpoint.

Wo mehrere Threads oder Prozesse schreiben, ist ein einzelner Schreiber mit Warteschlange die zuverlässigste Lösung: Ein Thread oder Prozess besitzt die einzige schreibende Verbindung, alle anderen übergeben ihm Aufträge. Sperrkonflikte unter Schreibern entfallen damit vollständig, Leser arbeiten im WAL-Modus parallel weiter.

Mehrere Container, die dieselbe SQLite-Datei über ein gemeinsames Volume öffnen, sind dagegen eine häufige Ursache für Sperren. Jeder Container ist ein eigener Schreiber, und keiner weiss vom anderen. Skaliert eine Anwendung horizontal, ist ein Datenbankserver wie PostgreSQL oder MySQL die passende Wahl.

## Keine Netzwerk-Dateisysteme

SQLite verlässt sich darauf, dass Dateisperren korrekt und atomar funktionieren und dass ein `fsync` die Daten tatsächlich auf den Datenträger bringt. Bei NFS, SMB/CIFS und vielen FUSE-Dateisystemen ist beides nicht garantiert. Die SQLite-Entwickler raten ausdrücklich davon ab, eine Datenbank über ein Netzwerk-Dateisystem von mehreren Rechnern aus zu nutzen. Die Folgen reichen von dauerhaften Sperrmeldungen bis zu einer beschädigten Datenbank, und der WAL-Modus funktioniert dort grundsätzlich nicht, weil der Shared-Memory-Abgleich nur lokal möglich ist.

Für Docker bedeutet das: Ein Named Volume oder ein Bind-Mount auf einen lokalen Datenträger ist geeignet. Ein Bind-Mount auf eine eingebundene SMB- oder NFS-Freigabe ist es nicht, auch wenn er im Normalbetrieb zu funktionieren scheint.

## I/O-Last: warum die Sperre plötzlich zu lange dauert

Die Sperre ist nur ein Symptom. Sie wird zum Problem, wenn sie länger gehalten wird als der Busy-Timeout erlaubt, und das hängt fast immer an der Schreiblatenz des Datenträgers.

Jeder Commit endet mit einem `fsync`. Erst wenn der Datenträger bestätigt, dass die Daten dauerhaft geschrieben sind, gibt SQLite die Sperre frei. Auf einer SSD dauert das unter einer Millisekunde, auf einer ruhigen Festplatte einige Millisekunden. Muss der Datenträger gleichzeitig andere Aufträge abarbeiten, wartet der `fsync` in der Warteschlange, und die Sperre bleibt so lange bestehen. Alle anderen Schreiber warten mit, bis ihr Busy-Timeout abläuft.

Mehrere Faktoren treiben diese Latenz auf einer NAS nach oben:

- **SMR-Festplatten.** Bei Shingled Magnetic Recording überlappen sich die Spuren. Wahlfreie Schreibvorgänge landen zunächst in einem Cache-Bereich und werden später in die überlappenden Zonen umsortiert. Ist der Cache-Bereich unter Dauerlast voll, steigt die Schreiblatenz von Millisekunden auf mehrere Sekunden. Datenbanken mit vielen kleinen, verstreuten Schreibvorgängen sind dafür das ungünstigste Lastprofil.
- **Parallele Sicherungen.** restic, Hyper Backup oder rsync lesen grosse Datenmengen und konkurrieren mit jedem `fsync` um dieselben Spindeln.
- **Scrubbing und RAID-Konsistenzprüfungen.** Ein Datenträger-Scrub liest den gesamten Datenbestand und läuft auf vielen NAS-Systemen nach Zeitplan, oft über Stunden.
- **Andere Datenbanken auf demselben Volume.** Prometheus und Loki schreiben laufend Blöcke und komprimieren sie periodisch. Diese Compactions erzeugen Lastspitzen.
- **Checkpoints im WAL-Modus.** Der Checkpoint schreibt viele Seiten auf einmal und schliesst mit `fsync` ab. Fällt er in eine Phase hoher Last, verlängert er sich entsprechend.

Im Praxisfall trafen mehrere dieser Faktoren zusammen: SMR-Festplatten, Sicherungen und eine Monitoring-Umgebung mit mehreren schreibenden Diensten. Grafana lief im Rollback-Modus ohne Retries. Jeder Commit der Alert-State-History benötigte mehrere `fsync`-Aufrufe, und jeder davon wartete hinter der übrigen Last. Beim Start liest Grafana unter anderem Werte aus seinem Schlüssel-Wert-Speicher in der Datenbank. Ohne Wiederholung brach der Start beim ersten Sperrkonflikt ab, und der Container wurde neu gestartet.

### Latenz messen

Ob die Datenträger das Problem sind, lässt sich direkt messen. Die Auslastung pro Datenträger zeigt `iostat` aus dem Paket sysstat:

```bash
iostat -x 5
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-x` | Erweiterte Statistik pro Gerät, unter anderem `r_await`/`w_await` (mittlere Wartezeit in ms) und `%util` (Auslastung). |
| `5` | Intervall in Sekunden; die erste Ausgabe zeigt Durchschnittswerte seit dem Systemstart und ist für die aktuelle Lage nicht aussagekräftig. |

</details>

Werte für `w_await` im dreistelligen Millisekundenbereich und `%util` nahe 100 zeigen, dass der Datenträger Aufträge stauen muss. Die Latenz einzelner synchroner Schreibvorgänge lässt sich mit `dd` im Verzeichnis der Datenbank prüfen:

```bash
dd if=/dev/zero of=./fsync-test bs=4k count=500 \
   oflag=dsync
rm ./fsync-test
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `if=/dev/zero` | Quelle: Nullbytes, es wird nur die Schreibleistung gemessen. |
| `of=./fsync-test` | Zieldatei im aktuellen Verzeichnis; dieses muss auf dem Volume der Datenbank liegen. |
| `bs=4k` | Blockgrösse 4 KiB, entspricht der üblichen SQLite-Seitengrösse. |
| `count=500` | Anzahl Blöcke, insgesamt 2 MB. |
| `oflag=dsync` | Jeder Block wird synchron geschrieben, vergleichbar mit einem `fsync` pro Schreibvorgang. |

</details>

`dd` meldet am Ende Dauer und Durchsatz. 500 Blöcke in 2 bis 5 Sekunden entsprechen 4 bis 10 ms pro synchronem Schreibvorgang und sind für Festplatten normal. Dauert derselbe Test während einer Sicherung 60 Sekunden oder länger, liegt die Ursache der Sperren beim Datenträger.

## Lösung im Praxisfall

Grafana unterstützt beide Massnahmen über Konfigurationsparameter im Abschnitt `[database]`, die sich als Umgebungsvariablen setzen lassen:

```yaml
services:
  grafana:
    image: grafana/grafana-oss:13.0.2
    environment:
      GF_DATABASE_WAL: "true"
      GF_DATABASE_QUERY_RETRIES: "5"
    volumes:
      - grafana-data:/var/lib/grafana

volumes:
  grafana-data:
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `GF_DATABASE_WAL` | Entspricht `wal = true`: Grafana öffnet die SQLite-Datei im WAL-Modus. Standard ist `false`. |
| `GF_DATABASE_QUERY_RETRIES` | Entspricht `query_retries`: Anzahl Wiederholungen einer Abfrage, wenn SQLite `database is locked` meldet. Standard ist `0`. |
| `grafana-data:/var/lib/grafana` | Named Volume auf dem lokalen Datenträger der NAS; dort liegen `grafana.db` sowie nach der Umstellung `grafana.db-wal` und `grafana.db-shm`. |

</details>

Für Transaktionen gibt es zusätzlich `transaction_retries` (Standard `5`), das im Praxisfall nicht angepasst werden musste. Nach dem Neustart mit beiden Einstellungen startete Grafana ohne Fehler, der Health-Endpoint meldete wieder `"database": "ok"`, und die SQLITE_BUSY-Meldungen der Alert-State-History blieben aus. Ob die Umstellung gegriffen hat, zeigen die neuen Dateien im Volume:

```bash
docker exec grafana ls -l /var/lib/grafana
```

Erscheinen `grafana.db-wal` und `grafana.db-shm`, arbeitet die Datenbank im WAL-Modus.

Bei stark ausgelasteten Systemen lassen sich zwei weitere Massnahmen prüfen. Die Alert-State-History kann statt in die Datenbank nach Loki geschrieben werden (`[unified_alerting.state_history]` mit `backend = loki`), wenn ohnehin ein Loki läuft. Und für Installationen mit vielen Benutzern oder mehreren Grafana-Instanzen empfiehlt Grafana eine externe Datenbank (MySQL oder PostgreSQL) statt SQLite.

## Sichern ohne Sperrkonflikte

Eine Dateisicherung von `grafana.db` während des Betriebs erfasst einen beliebigen Zwischenstand. Im WAL-Modus fehlen ohne `-wal`-Datei zudem alle noch nicht übertragenen Änderungen. Eine konsistente Kopie erzeugt SQLite selbst:

```bash
sqlite3 /var/lib/grafana/grafana.db \
  ".backup /backup/grafana-$(date +%F).db"
```

Die Online-Backup-API kopiert die Datenbank seitenweise und berücksichtigt dabei laufende Schreibvorgänge. Alternativ erzeugt `VACUUM INTO '/backup/grafana.db'` eine kompakte Kopie. Beide Varianten lassen sich vor der eigentlichen Dateisicherung ausführen, die dann die Kopie statt der Live-Datei sichert. Den Zeitplan für Sicherungen und Scrubs sollten Sie so legen, dass sie sich nicht überschneiden und nicht mit bekannten Lastspitzen anderer Dienste zusammenfallen.

## Checkliste

- WAL-Modus aktivieren, sofern alle Zugriffe vom selben Rechner kommen.
- Busy-Timeout auf jeder Verbindung setzen, 5 bis 10 Sekunden als Startwert.
- Schreibtransaktionen mit `BEGIN IMMEDIATE` beginnen und kurz halten.
- Wiederholungen bei `SQLITE_BUSY` auf Anwendungsebene vorsehen, in Grafana über `query_retries`.
- Nur einen schreibenden Prozess pro Datenbankdatei betreiben.
- Datenbankdateien nur auf lokalen Dateisystemen ablegen, nie auf SMB- oder NFS-Freigaben.
- Schreiblatenz messen, wenn Sperren gehäuft auftreten; SMR-Festplatten für Datenbanken meiden oder Datenbanken auf eine SSD verlagern.
- Sicherungen über `.backup` oder `VACUUM INTO` erstellen und zeitlich von Scrubs trennen.

## Quellen

1.  [SQLite: Result and Error Codes](https://www.sqlite.org/rescode.html): Bedeutung von `SQLITE_BUSY` (5), `SQLITE_LOCKED` (6) und der erweiterten Codes wie `SQLITE_BUSY_SNAPSHOT`.

2.  [SQLite: File Locking And Concurrency In SQLite Version 3](https://www.sqlite.org/lockingv3.html): die Sperrstufen SHARED, RESERVED, PENDING und EXCLUSIVE im Rollback-Modus.

3.  [SQLite: Write-Ahead Logging](https://www.sqlite.org/wal.html): Funktionsweise, Checkpoints, `wal_autocheckpoint` und die Einschränkung auf lokale Dateisysteme.

4.  [SQLite: PRAGMA Statements](https://www.sqlite.org/pragma.html): `journal_mode`, `busy_timeout`, `synchronous` und `wal_checkpoint`.

5.  [SQLite: BEGIN TRANSACTION](https://www.sqlite.org/lang_transaction.html): Unterschied zwischen DEFERRED, IMMEDIATE und EXCLUSIVE.

6.  [SQLite: SQLite Over a Network, Caveats and Considerations](https://www.sqlite.org/useovernet.html): warum Netzwerk-Dateisysteme für SQLite ungeeignet sind.

7.  [SQLite: How To Corrupt An SQLite Database File](https://www.sqlite.org/howtocorrupt.html): fehlerhafte Dateisperren und `fsync` als Ursachen für beschädigte Datenbanken.

8.  [SQLite: SQLite Backup API](https://www.sqlite.org/backup.html): konsistente Kopien einer Datenbank im laufenden Betrieb.

9.  [Python: sqlite3 DB-API 2.0 interface](https://docs.python.org/3/library/sqlite3.html): Parameter `timeout` und `isolation_level` von `sqlite3.connect()`.

10.  [Grafana: Configure Grafana, Abschnitt database](https://grafana.com/docs/grafana/latest/setup-grafana/configure-grafana/#database): Parameter `wal`, `query_retries` und `transaction_retries` sowie die Abbildung auf `GF_`-Umgebungsvariablen.

11.  [Wikipedia: Shingled magnetic recording](https://en.wikipedia.org/wiki/Shingled_magnetic_recording): Aufbau von SMR-Festplatten und ihr Verhalten bei wahlfreien Schreibvorgängen.
