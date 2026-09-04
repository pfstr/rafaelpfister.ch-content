---
title: "Wenn das Log die Platte füllt: log4j2 RollingFile richtig begrenzen, am Beispiel totemomail"
navTitle: "log4j2 Plattenplatz"
description: "Ein volllaufendes Log-Volume legt im schlimmsten Fall das ganze Gateway lahm. Warum die Kombination aus Zeit- und Grössenrotation ohne %i eine einzelne Riesendatei erzeugt, wie strategy.max die Aufbewahrung deckelt, welche Rolle das Log-Level spielt und wo totemomail diese Werte versteckt."
date: "2026-09-04"
kategorie: "Totemomail"
timeToRead: "9 Min. Lesezeit"
themen:
  - "totemomail"
produkte:
  - "totemomail"
protokolle:
  - "troubleshooting"
  - "storage"
slug: "log4j2-rollingfile-plattenplatz-totemomail"
translationId: "article-c400eee99d90052d"
url: "https://rafaelpfister.ch/blog/log4j2-rollingfile-plattenplatz-totemomail"
---
# Wenn das Log die Platte füllt: log4j2 RollingFile richtig begrenzen, am Beispiel totemomail

Ein Mailgateway auf Java-Basis schreibt im DEBUG-Betrieb erstaunliche Mengen. Ein einziger Lasttag kann mehrere Gigabyte Trace-Log erzeugen, und wenn das Log-Volume klein bemessen ist, läuft es voll. Die Folge ist unangenehm: Der Java-Prozess kann nicht mehr in sein Log schreiben, der Logging-Framework fällt in einen Fehlerzustand, und selbst nachdem wieder Platz frei ist, schreibt er erst nach einem Neustart wieder mit. Bei einem Mailgateway kann eine volle Platte zudem das Spooling und die Zustellung stören. Der Auslöser ist fast immer eine Log-Rotation, die zwar konfiguriert ist, aber nicht so wirkt, wie man annimmt.

Der folgende Beitrag erklärt die Rotation von log4j2 an genau dieser Stelle, allgemein und dann konkret für totemomail (das auf Apache James und log4j2 aufsetzt). Der Kern ist eine einzelne, leicht zu übersehende Angabe im Dateimuster.

## Wie log4j2 rotiert

Der `RollingFileAppender` von log4j2 kombiniert zwei Bausteine: eine oder mehrere **TriggeringPolicies** entscheiden, *wann* rotiert wird, und eine **RolloverStrategy** entscheidet, *wie* die Archivdateien benannt und wie viele behalten werden. Typisch sind zwei Policies gleichzeitig:

- `TimeBasedTriggeringPolicy`: rotiert nach Zeit, meist täglich.
- `SizeBasedTriggeringPolicy`: rotiert, sobald die aktive Datei eine Grösse erreicht, etwa 100 MB.

Beim Rollover wird die aktive Datei umbenannt und archiviert. Wie die Archivdatei heisst, bestimmt das `filePattern`, und darin stecken zwei Platzhalter, deren Zusammenspiel den entscheidenden Unterschied macht.

<details class="options-details">
<summary>Optionen im Überblick</summary>

| Platzhalter | Bedeutung |
|---|---|
| `%d{...}` | Datum/Zeit des Rollovers nach dem angegebenen Muster, z. B. `%d{yyyy-MM-dd}` für den Tag |
| `%i` | Der berechnete Index der Archivdatei, ein Zähler, der bei jedem Rollover hochzählt |
| `%03i` | Derselbe Index, nullgepolstert auf drei Stellen |
| `.gz` / `.zip` am Musterende | Archiv wird beim Rollover komprimiert |

</details>

Die vollständige Referenz steht in der log4j2-Dokumentation zum Rolling File Appender; die Tabelle oben nennt nur die für die Grössen- und Zeitrotation wesentlichen Elemente.

## Die %i-Falle

Genau hier liegt der Fehler, der Platten füllt. Wer nur nach Datum benennt, also `filePattern = trace.log.%d{yyyy-MM-dd}`, und gleichzeitig eine Grössen-Policy von 100 MB setzt, bekommt nicht viele 100-MB-Dateien pro Tag, sondern eine einzige, die ungebremst weiterwächst. Die Grössenrotation hat kein eigenes Ziel, in das sie das nächste Stück schreiben könnte, weil im Muster kein Zähler vorkommt. Die log4j2-Dokumentation ist an dieser Stelle deutlich:

> When combined with a time-based triggering policy, the filePattern attribute of the Appender should contain an `%i` conversion pattern. Otherwise, the target file will be overwritten on each rollover.

Ohne `%i` ist die Kombination aus Zeit- und Grössenrotation also fehlerhaft; je nach Strategie wird die Datei entweder überschrieben oder sie wächst über die eingestellte Grösse hinaus. In der Praxis heisst das: Die 100-MB-Grenze greift nie, ein Lasttag schreibt alles in eine Datei, und die wird mehrere Gigabyte gross. Der Fix ist eine Ergänzung des Musters:

```text
filePattern = trace.log.%d{yyyy-MM-dd}.%i
```

Damit legt jeder 100-MB-Rollover eine eigene indexierte Datei an (`trace.log.2026-09-04.1`, `.2`, `.3`), und die Grössenbegrenzung wirkt wie gedacht.

## Aufbewahrung über strategy.max

Der Index ist zugleich die Voraussetzung dafür, dass die Aufbewahrung funktioniert. Die `DefaultRolloverStrategy` besitzt ein Attribut `max`, das die maximale Anzahl behaltener Archivdateien angibt; über diese Grenze hinaus werden die ältesten gelöscht. Ohne `%i` gibt es keinen Index, den `max` zählen könnte, also wird auch nichts gelöscht, und alte datierte Dateien sammeln sich an.

<details class="options-details">
<summary>Optionen erklärt</summary>

| Attribut | Wirkung |
|---|---|
| `max` | Höchstzahl behaltener Archivdateien; darüber hinaus werden die ältesten entfernt |
| `min` | Kleinster Indexwert (Standard 1) |
| `fileIndex="min"` | Neueste Datei erhält Index `min`, älteste `max` |
| `fileIndex="max"` (Standard) | Älteste Datei erhält Index `min`, neueste `max` |
| `fileIndex="nomax"` | Es wird nie gelöscht, neue Archive bekommen fortlaufend steigende Indizes |

</details>

Aus Grösse und Anzahl ergibt sich die Gesamt-Obergrenze: 100 MB pro Datei mal `max=10` deckelt das Log auf rund ein Gigabyte, unabhängig davon, wie viel geschrieben wird. Wer eine feinere Kontrolle über das Alter statt die Anzahl braucht, ergänzt in der Strategie eine `Delete`-Aktion mit `IfLastModified` (Alter) oder `IfAccumulatedFileSize` (Gesamtgrösse); für die meisten Fälle reicht die Kombination aus Grösse pro Datei und `max`.

## Das Log-Level als eigentlicher Mengentreiber

Rotation und Aufbewahrung begrenzen den Platzverbrauch, aber sie ändern nichts daran, wie viel überhaupt geschrieben wird. Der grösste Hebel ist das Log-Level. Ein produktiv auf DEBUG laufendes Gateway protokolliert jeden Verarbeitungsschritt jeder Nachricht, und das summiert sich unter Last zu Gigabyte pro Tag. Für den Normalbetrieb gehört das Level auf INFO oder höher; DEBUG ist ein Werkzeug für eine begrenzte Analyse, nicht für den Dauerbetrieb. Steht das Level auf INFO und ist zusätzlich die Grössenrotation mit `%i` korrekt gesetzt, greift beides ineinander: INFO hält die Tagesmenge klein, und die Rotation deckelt selbst einen DEBUG-Ausreisser.

## Wo totemomail diese Werte hält

Bei totemomail sind diese Einstellungen nicht in einer lokalen `log4j2.xml` zu finden, und das führt bei der Fehlersuche leicht in die Irre. Die Konfiguration entsteht zur Laufzeit aus Properties mit dem Präfix `totemo.log4j2.*`, und diese Properties werden zentral über die Management-Console verwaltet (Bereich Logging + Tracking). Eine Suche nach `log4j2.xml` auf dem Dateisystem läuft deshalb ins Leere; eine `log4j.xml` im Konfigverzeichnis gehört zu einer mitgelieferten Komponente (openjms) und hat mit dem Trace-Log nichts zu tun.

Die relevanten Properties und ihre Bedeutung:

<details class="options-details">
<summary>Optionen erklärt</summary>

| Property | Bedeutung |
|---|---|
| `totemo.log4j2.appender.a1.filePattern` | Das Dateimuster; hier gehört das `%i` hinein |
| `totemo.log4j2.appender.a1.policies.size.size` | Grösse pro Datei für die SizeBasedTriggeringPolicy, z. B. `100MB` |
| `totemo.log4j2.appender.a1.strategy.max` | Anzahl behaltener Archivdateien |
| `totemo.log4j2.rootLogger.level` | Level des log4j2-Root-Loggers |
| `totemo.log.priority` | Übergreifende Protokoll-Priorität der Anwendung, der eigentliche DEBUG-Schalter |
| `totemo.tracking` | Detailgrad des Nachrichten-Trackings; `debug` erzeugt die Zeilen pro Mailet |

</details>

Wichtig ist die Doppelnatur: Die log4j2-Logger können auf `warn` oder `error` stehen und trotzdem eine DEBUG-Flut im Trace-Log erzeugen, weil `totemo.log.priority` und `totemo.tracking` als eigene, übergeordnete Schalter wirken. Wer das Volumen senken will, setzt `totemo.log.priority` auf INFO und `totemo.tracking` von `debug` auf `on`; das entfernt die ausführlichen Verarbeitungszeilen. Weil die Werte über die Console verwaltet werden, gelten sie clusterweit, und einige verlangen einen Neustart der Instanz, um zu greifen (das ist am jeweiligen Property vermerkt).

## Der Neustart nach dem Volllaufen

Ein Detail, das leicht übersehen wird: Nachdem die Platte einmal voll war, kehrt das Logging nicht von selbst zurück, auch wenn man Platz freiräumt. Der Datei-Appender bleibt in seinem Fehlerzustand, bis der Java-Prozess neu startet. Man erkennt das daran, dass das Gateway noch Mail annimmt und verarbeitet (der SMTP-Banner zeigt die korrekte Uhrzeit), das Trace-Log aber am Zeitpunkt des Volllaufens stehen bleibt. Ein kontrollierter Neustart der Instanz stellt das Logging wieder her und aktiviert zugleich geänderte Appender-Einstellungen wie das neue `filePattern`.

## Diagnose in wenigen Befehlen

Die volle Partition und ihr Verursacher lassen sich schnell eingrenzen. Zuerst zeigt sich, welches Dateisystem betroffen ist:

```bash
df -h
```

Ist das Log-Volume bei 100 Prozent, benennt eine nach Grösse sortierte Auflistung den Hauptverursacher:

```bash
du -sh /pfad/zu/logs/* | sort -rh | head
```

Findet sich dort eine einzelne, viele Gigabyte grosse Tagesdatei statt vieler kleiner indexierter Archive, ist das die `%i`-Falle. Nach dem Fix und einem Neustart bestätigt die Dateiliste, dass die Rotation greift:

```bash
ls -laht /pfad/zu/logs/trace.log*
```

Erwartet werden `trace.log` plus indexierte Archive `trace.log.<datum>.1`, `.2` und so weiter, jede etwa in der eingestellten Maximalgrösse.

## Zusammenfassung

Wer log4j2 mit Zeit- und Grössenrotation betreibt, braucht zwingend ein `%i` im `filePattern`, sonst wächst eine einzelne Datei ungebremst und die Grössengrenze bleibt wirkungslos. Über `strategy.max` (zusammen mit dem Index) deckelt die Anzahl der Archive den Platzverbrauch, und das Log-Level entscheidet über die Menge an der Quelle. Bei totemomail stehen diese Werte in der Management-Console unter `totemo.log4j2.*` sowie in den übergeordneten Schaltern `totemo.log.priority` und `totemo.tracking`; nach einem Volllaufen der Platte gehört ein Neustart der Instanz dazu, damit das Logging wieder schreibt.

## Quellen

1.  [Apache Logging Services: Log4j RollingFileAppender](https://logging.apache.org/log4j/2.x/manual/appenders/rolling-file.html): Referenz zu filePattern, den TriggeringPolicies und der DefaultRolloverStrategy inklusive der Aussage zum `%i` bei zeitbasierter Rotation.

2.  [Apache Logging Services: Log4j Architecture](https://logging.apache.org/log4j/2.x/manual/architecture.html): Einordnung von Appender, Layout und Logger-Hierarchie, für das Verständnis von Root-Logger und Log-Level.
