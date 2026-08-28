---
title: "Mailstorm auf totemomail: Server stoppen, Queues prüfen und kontrolliert bereinigen"
navTitle: "Queues bereinigen"
description: "Wenn ein Mailstorm die Queues von totemomail füllt, hilft ein kontrolliertes Vorgehen: Dienst über systemd und das Tanuki-Control-Script stoppen, Queue-Bestand pro Repository zählen, einzelne Nachrichten inspizieren, erst dann bereinigen und den Dienst wieder starten."
date: "2026-08-28"
kategorie: "Totemomail"
timeToRead: "9 Min. Lesezeit"
themen:
  - "totemomail"
produkte:
  - "totemomail"
  - "apache-james"
protokolle:
  - "smtp"
  - "troubleshooting"
slug: "totemomail-mailstorm-queues-bereinigen"
translationId: "article-ceab4ac53bb7acdd"
url: "https://rafaelpfister.ch/blog/totemomail-mailstorm-queues-bereinigen"
---

# Mailstorm auf totemomail: Server stoppen, Queues prüfen und kontrolliert bereinigen

Ein Mailstorm hat viele mögliche Auslöser: eine Mailschlaufe zwischen Gateway und Exchange, eine fehlerhafte Weiterleitungsregel, ein Lasttest mit falschem Ziel oder ein Massenversand, der ungebremst auf das Gateway trifft. Auf einem totemomail-Gateway (heute Kiteworks Email Protection Gateway) füllen sich dann die Queues, und jede weitere Verarbeitungsrunde verschärft das Problem.

In dieser Situation hat sich ein festes Vorgehen bewährt: Dienst anhalten, Bestand aufnehmen, gezielt bereinigen, wieder anfahren. Regeländerungen im laufenden Betrieb verschärfen das Problem eher, weil jede Analyse auf einem Bestand arbeitet, der sich dabei laufend ändert. Dieser Beitrag zeigt jeden dieser Schritte mit den konkreten Befehlen, inklusive der Frage, wie der Dienst überhaupt sauber gestoppt wird. Das Verarbeitungsmodell dahinter (Processors, Repositories, Dateiformate) ist im Beitrag [Mailrouting zwischen totemomail und Exchange Online verstehen](/blog/totemomail-m365) beschrieben.

Alle Pfade beziehen sich auf eine Installation unter `/opt/totemomail` mit dem Servicebenutzer `totemo`. Passen Sie die Pfade an Ihre Umgebung an.

## Wie totemomail gestartet und gestoppt wird

Bevor Sie einen Dienst stoppen, sollten Sie wissen, wie er läuft. Bei totemomail sind drei Schichten beteiligt:

- Eine **systemd-Unit** `totemomail.service` als äusserste Steuerungsebene.
- Das **Control-Script** `/opt/totemomail/bin/totemomail`, das die Unit bei Start und Stopp aufruft.
- Der **Tanuki Java Service Wrapper**: ein nativer `wrapper`-Prozess, der den eigentlichen Java-Prozess startet, überwacht und bei einem Absturz neu starten kann.

Den Aufbau können Sie auf Ihrem System nachvollziehen, ohne die Unit-Datei lesen zu dürfen. `systemctl show` fragt die Eigenschaften direkt bei systemd ab und funktioniert auch dann, wenn die Datei unter `/etc/systemd/system/` nur für root lesbar ist:

```bash
systemctl show totemomail.service -p Type -p User -p ExecStart -p ExecStop \
  -p KillMode -p TimeoutStopUSec --no-pager
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `show totemomail.service` | Zeigt die Laufzeit-Eigenschaften der Unit, wie systemd sie geladen hat |
| `-p <Property>` | Beschränkt die Ausgabe auf die genannte Eigenschaft; mehrfach angebbar |
| `--no-pager` | Gibt direkt auf die Konsole aus, statt einen Pager wie `less` zu öffnen |

</details>

Eine typische Ausgabe sieht so aus:

```text
Type=oneshot
TimeoutStopUSec=1min 30s
ExecStart={ path=/opt/totemomail/bin/totemomail ; argv[]=/opt/totemomail/bin/totemomail start ; ... }
ExecStop={ path=/opt/totemomail/bin/totemomail ; argv[]=/opt/totemomail/bin/totemomail stop ; ... }
User=totemo
KillMode=control-group
```

Daraus lassen sich die wichtigen Eigenschaften ablesen: `systemctl stop totemomail` ruft das Control-Script mit dem Argument `stop` auf, wartet bis zu 90 Sekunden auf ein sauberes Ende und beendet danach über `KillMode=control-group` alle noch verbliebenen Prozesse der Unit. Der Stopp über systemd ist damit gleichwertig zum direkten Script-Aufruf, räumt aber zusätzlich auf, falls das Script hängen bleibt.

Der Status `active (exited)` bei `systemctl status totemomail` ist bei diesem Aufbau normal und kein Fehler: Die Unit ist `Type=oneshot`, das Start-Script beendet sich nach dem Start, und der Wrapper läuft als eigenständiger, von systemd nur indirekt verwalteter Daemon weiter. Ob der Dienst wirklich läuft, zeigt deshalb nicht der Unit-Status, sondern die Prozessliste:

```bash
ps -ef | grep -E 'wrapper|TotemoBootStrapper' | grep -v grep
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-e` | Zeigt alle Prozesse, nicht nur die der eigenen Sitzung |
| `-f` | Volles Ausgabeformat mit kompletter Befehlszeile |
| `grep -E 'wrapper\|TotemoBootStrapper'` | Filtert auf den Wrapper-Prozess und die Java-Hauptklasse |
| `grep -v grep` | Entfernt die grep-Prozesse selbst aus der Trefferliste |

</details>

Im Normalbetrieb erscheinen zwei Prozesse: der native `wrapper` (gestartet mit `../conf/wrapper.conf` und dem PID-File `totemomail.pid`) und der Java-Prozess mit der Hauptklasse `ch.totemo.bootstrapper.TotemoBootStrapper`. Fehlt einer der beiden, ist der Dienst nicht vollständig gestartet.

## Schritt 1: Dienst stoppen

Bei einem Mailstorm stoppen Sie den Dienst als Erstes. Solange totemomail läuft, nimmt es weiter Nachrichten an, verarbeitet die Queues und stellt zu; erst der Stopp friert den Bestand für die Analyse ein.

```bash
sudo systemctl stop totemomail
```

Kontrollieren Sie danach, dass Wrapper- und Java-Prozess beendet sind:

```bash
ps -ef | grep -E 'wrapper|TotemoBootStrapper' | grep -v grep
```

Die Ausgabe muss leer sein. Zusätzlich verschwindet das PID-File `/opt/totemomail/bin/totemomail.pid`. Bleibt ein Prozess nach Ablauf des Stop-Timeouts stehen, beendet systemd ihn über die Control-Group; prüfen Sie in diesem Fall `journalctl -u totemomail`, bevor Sie weiterarbeiten.

Vergessen Sie die vorgelagerte Ebene nicht: Wenn der Mailstorm von aussen kommt, stauen sich die Nachrichten nach dem Stopp beim einliefernden System (etwa in der Exchange-Warteschlange oder beim vorgelagerten Relay). Das ist gewollt. Dort lassen sie sich ebenfalls anhalten oder verwerfen, und seriöse Absender stellen nach dem Wiederanlauf automatisch erneut zu.

## Schritt 2: Queue-Bestand aufnehmen

Die Queues von totemomail sind dateibasierte Mail-Repositories des zugrunde liegenden Apache James. Sie liegen unterhalb des James-Anwendungsverzeichnisses, hier `/opt/totemomail/mailer/apps/james/var/mail/`. Jedes Unterverzeichnis ist ein Repository, jede Nachricht besteht aus zwei Dateien: `*.FileStreamStore` enthält die vollständige MIME-Nachricht, `*.FileObjectStore` das serialisierte Statusobjekt mit Metadaten.

Einen Überblick über den Bestand liefert eine Zählung der `FileObjectStore`-Dateien pro Verzeichnis:

```bash
for d in /opt/totemomail/mailer/apps/james/var/mail/*/; do \
  printf '%-22s %s\n' "$(basename "$d")" \
  "$(find "$d" -maxdepth 1 -name '*.FileObjectStore' | wc -l)"; \
done
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `for d in .../*/` | Iteriert über alle Repository-Verzeichnisse (der abschliessende `/` beschränkt auf Verzeichnisse) |
| `printf '%-22s %s\n'` | Formatiert die Ausgabe zweispaltig; `%-22s` füllt den Namen linksbündig auf 22 Zeichen auf |
| `basename "$d"` | Reduziert den vollen Pfad auf den Verzeichnisnamen |
| `find "$d" -maxdepth 1` | Sucht nur direkt im Verzeichnis, ohne Unterverzeichnisse |
| `-name '*.FileObjectStore'` | Zählt eine Datei pro Nachricht; das Stream-Pendant würde die Zahl verdoppeln |
| `wc -l` | Zählt die gefundenen Dateien |

</details>

Das Ergebnis ist eine Zeile pro Queue mit der Anzahl Nachrichten, zum Beispiel:

```text
DBUnavailable          0
error                  12
incoming               121
outgoing               0
spool                  0
```

Die Standard-Repositories bedeuten: `spool` enthält angenommene, noch unverarbeitete Nachrichten, `incoming` intern zuzustellende, `outgoing` ausgehende, `error` fehlgeschlagene und `DBUnavailable` Nachrichten, die wegen eines nicht erreichbaren Backends geparkt wurden. Je nach Konfiguration existieren weitere Repositories für spezielle Routen; sie folgen demselben Dateischema.

Läuft `find` aus einem Verzeichnis heraus, auf das der Servicebenutzer keinen Zugriff hat (etwa dem Home eines anderen Benutzers nach `sudo -u totemo`), erscheint pro Aufruf die Warnung `Failed to restore initial working directory`. Sie ist harmlos und verschwindet nach einem `cd ~`.

## Schritt 3: In die Nachrichten schauen

Zahlen allein reichen für eine Entscheidung nicht. Bevor Sie etwas löschen, sollten Sie wissen, was in den Queues liegt: Sind es die Sturm-Nachrichten, oder stecken dazwischen legitime Mails, die nach dem Wiederanlauf zugestellt werden sollen?

Die `FileStreamStore`-Dateien sind unveränderte RFC-822-Nachrichten. Die wichtigsten Header lassen sich deshalb direkt auslesen:

```bash
for f in /opt/totemomail/mailer/apps/james/var/mail/incoming/*.FileStreamStore; do \
  awk 'BEGIN{IGNORECASE=1} /^(From|To|Subject|Date):/{print} /^\r?$/{exit}' "$f"; \
  echo ---; \
done | less
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `BEGIN{IGNORECASE=1}` | Header-Namen ohne Beachtung der Gross-/Kleinschreibung vergleichen (GNU awk) |
| `/^(From\|To\|Subject\|Date):/{print}` | Gibt nur die vier relevanten Header-Zeilen aus |
| `/^\r?$/{exit}` | Bricht an der Leerzeile zwischen Header und Body ab; der Nachrichteninhalt wird nicht gelesen |
| `echo ---` | Trennlinie zwischen den Nachrichten |
| `less` | Blättern statt Durchscrollen bei vielen Nachrichten |

</details>

Bei grossen Beständen ist die Verteilung aussagekräftiger als die Einzelansicht. Die häufigsten Absender zeigt:

```bash
grep -him1 '^From:' /opt/totemomail/mailer/apps/james/var/mail/incoming/*.FileStreamStore \
  | sort | uniq -c | sort -rn | head
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-h` | Unterdrückt die Dateinamen in der Ausgabe, damit identische Absender zusammenfallen |
| `-i` | Gross-/Kleinschreibung ignorieren |
| `-m1` | Nur den ersten Treffer pro Datei (den Header, nicht zitierte `From:`-Zeilen im Body) |
| `sort \| uniq -c` | Gruppiert identische Absenderzeilen und zählt sie |
| `sort -rn \| head` | Sortiert absteigend nach Häufigkeit und zeigt die zehn häufigsten |

</details>

Dominiert ein einzelner Absender oder ein einzelner Betreff mit hunderten Kopien, haben Sie die Sturm-Nachrichten identifiziert. Ein Blick auf die Dateizeitstempel (`ls -lt`) grenzt zusätzlich den Zeitraum ein und zeigt, ob ältere, legitime Nachrichten zwischen den Sturm-Mails liegen.

## Schritt 4: Kontrolliert bereinigen

Erst jetzt wird gelöscht, und auch jetzt noch mit einem Zwischenschritt: Verschieben Sie den Bestand zuerst in ein Sicherungsverzeichnis, statt ihn direkt zu löschen. Das Ergebnis für den Mailbetrieb ist dasselbe (die Queue ist leer), aber der Schritt ist umkehrbar, und einzelne legitime Nachrichten lassen sich später aus der Sicherung zurücklegen oder als `.eml` weiterverwenden.

```bash
mkdir -p /opt/totemomail/queue-backup-$(date +%F)
mv /opt/totemomail/mailer/apps/james/var/mail/incoming/* \
   /opt/totemomail/queue-backup-$(date +%F)/
```

Wichtig dabei: Die Repository-Verzeichnisse selbst bleiben stehen, nur ihr Inhalt wird verschoben. Und Stream- und Object-Datei einer Nachricht gehören zusammen; wer nur eine der beiden entfernt, hinterlässt verwaiste Dateien, die beim nächsten Start Fehler im Log erzeugen.

Ist die Sicherung geprüft oder der Inhalt zweifelsfrei wertlos (etwa reine Lasttest-Nachrichten), löschen Sie den gesamten Queue-Inhalt über alle Repositories:

```bash
find /opt/totemomail/mailer/apps/james/var/mail/ -mindepth 2 -maxdepth 2 -type f \
  \( -name '*.FileStreamStore' -o -name '*.FileObjectStore' \) -delete
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-mindepth 2 -maxdepth 2` | Trifft nur Dateien direkt in den Repository-Verzeichnissen, nicht `var/mail` selbst und keine tieferen Ebenen |
| `-type f` | Nur reguläre Dateien; die Verzeichnisse bleiben erhalten |
| `\( -name ... -o -name ... \)` | Beide Dateitypen einer Nachricht, Stream und Statusobjekt |
| `-delete` | Löscht die Treffer direkt; vorab ohne diese Option ausführen, um die Trefferliste zu prüfen |

</details>

Danach dieselbe Zählung wie in Schritt 2 ausführen: Alle Repositories müssen 0 zeigen.

## Schritt 5: Dienst wieder starten

```bash
sudo systemctl start totemomail
```

Der Start ruft das Control-Script mit `start` auf, das den Wrapper daemonisiert; der Wrapper startet anschliessend den Java-Prozess. Kontrollieren Sie beides über die Prozessliste aus dem ersten Abschnitt und werfen Sie einen Blick in die Logdateien unter `/opt/totemomail/bin/`: `wrapper.log` protokolliert den Start des Wrappers und der JVM, `console.log` und `console.err` die Ausgaben der Anwendung selbst.

Zum Abschluss gehört ein Funktionstest mit einer einzelnen Testnachricht durch das Gateway, bevor der reguläre Mailfluss wieder freigegeben wird. Und falls die Ursache des Sturms eine Regel oder eine Schlaufe war: erst die Ursache beheben, dann den Verkehr wieder zulassen. Sonst beginnt die Aufnahme des Queue-Bestands von vorn.

## Zusammenfassung

| Schritt | Befehl | Kontrolle |
|---|---|---|
| Stoppen | `sudo systemctl stop totemomail` | `ps`-Filter leer, PID-File weg |
| Bestand zählen | `find`-Schleife über `var/mail/*/` | Anzahl pro Repository |
| Inspizieren | `awk`-Header-Auszug, `grep`-Absenderstatistik | Sturm-Mails von legitimen trennen |
| Bereinigen | `mv` in Sicherung, dann `find ... -delete` | Zählung zeigt überall 0 |
| Starten | `sudo systemctl start totemomail` | Prozesse, `wrapper.log`, Testnachricht |

## Quellen

1.  [Apache James Server 2: Provided Mailets](https://james.apache.org/server/2/provided_mailets.html): Dokumentation der Mailets und Repositories, auf denen die Queue-Struktur von totemomail basiert.

2.  [Tanuki Software: Java Service Wrapper](https://wrapper.tanukisoftware.com/doc/english/introduction.html): Funktionsweise des Wrappers, der den totemomail-Java-Prozess startet und überwacht, inklusive PID-File und `wrapper.conf`.

3.  [systemd.service(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html): Bedeutung von `Type=oneshot`, `ExecStop` und `TimeoutStopSec` bei Units, die ein externes Control-Script aufrufen.

4.  [systemd.kill(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.kill.html): `KillMode=control-group` als Absicherung, die nach dem Stop-Script verbliebene Prozesse der Unit beendet.

5.  [RFC 5322: Internet Message Format](https://datatracker.ietf.org/doc/html/rfc5322): Aufbau der Nachrichten-Header, die beim Inspizieren der `FileStreamStore`-Dateien ausgelesen werden.
