---
title: "Mail-Lasttests planen: Tools für 10'000-Mail-Bursts unter Linux und Windows im Vergleich"
navTitle: "Mail-Lasttests"
description: "Wer ein Gateway migriert oder eine Mail-Umgebung dimensioniert, braucht belastbare Zahlen statt Bauchgefühl. Welche Tools Bursts von mehreren zehntausend Mails erzeugen, wie ein sauberer Testplan aussieht und wie Sie die Ergebnisse aus den Logs auswerten."
date: "2026-08-24"
kategorie: "SMTP und Mailflow"
timeToRead: "12 Min. Lesezeit"
themen:
  - "smtp-mailflow"
  - "testing"
produkte:
  - "uebergreifend"
protokolle:
  - "testing"
  - "smtp"
  - "tcp"
  - "tls"
  - "troubleshooting"
slug: "mail-lasttest-tools-linux-windows-vergleich"
translationId: "article-14a98de0cef45565"
url: "https://rafaelpfister.ch/blog/mail-lasttest-tools-linux-windows-vergleich"
aiPrompt: |
  Du bist mein Assistent für Mail-Lasttests. Hilf mir Schritt für Schritt, einen Lasttest gegen mein eigenes Test-Mailgateway zu planen: Zieldefinition (Durchsatz, Latenz, Queue-Verhalten), Wahl des Lastgenerators (smtp-source, Postal, JMeter oder Skript), Aufbau einer Mail-Senke, Testablauf (Baseline, Burst, Soak) und Auswertung der Logs mit Perzentilen. Frage zuerst nach Plattform, Zielsystem und erwartetem Mailvolumen.
---
# Mail-Lasttests planen: Tools für 10'000-Mail-Bursts unter Linux und Windows im Vergleich

Ob ein neues Mailgateway die Spitzenlast einer Rechnungsläufe-Nacht verkraftet, zeigt sich nicht im Datenblatt, sondern im Test. Wer eine Appliance ablöst, eine Exchange-Umgebung dimensioniert oder einen Newsletter-Versand über die eigene Infrastruktur plant, braucht vorher belastbare Zahlen: Wie viele Nachrichten pro Sekunde nimmt das System an, wie verhält sich die Queue unter Druck, und ab welchem Punkt beginnen Deferrals? Dieser Artikel vergleicht die gängigen Lastgeneratoren unter Linux und Windows und zeigt, wie ein Test mit Bursts von mehreren zehntausend Mails geplant, durchgeführt und ausgewertet wird.

Vorweg die wichtigste Regel: Lasttests gehören ausschliesslich in die eigene Infrastruktur oder in eine ausdrücklich dafür freigegebene Testumgebung. Ein Burst gegen fremde Systeme ist ein Angriff, und ein Test mit erfundenen Absenderadressen gegen produktive Ziele erzeugt Backscatter, der auf Blocklisten führt. Der saubere Aufbau besteht aus einem Lastgenerator, dem zu testenden System und einer kontrollierten Senke, die die Mails am Ende annimmt und verwirft.

## Was ein Mail-Lasttest messen soll

Bevor ein Tool zur Sprache kommt, lohnt sich die Frage, welche Grösse eigentlich interessiert. In der Praxis sind es vier verschiedene, und sie verlangen unterschiedliche Testaufbauten:

1. **Annahmerate:** Wie viele Nachrichten pro Sekunde nimmt der erste Hop per SMTP entgegen? Das ist die klassische Durchsatzmessung und der Wert, den Lastgeneratoren direkt liefern.
2. **Session-Latenz:** Wie lange dauert eine einzelne SMTP-Transaktion vom Verbindungsaufbau bis zum `250` nach `DATA`? Unter Last steigt dieser Wert oft lange bevor die Annahmerate einbricht.
3. **Ende-zu-Ende-Latenz:** Wie lange braucht eine Nachricht vom Einliefern bis zur Zustellung an die Senke, über alle Zwischenstationen hinweg? Das ist die Grösse, die Anwender wahrnehmen.
4. **Queue-Verhalten:** Wie tief wächst die Warteschlange während des Bursts, und wie schnell läuft sie danach leer? Ein Gateway, das 50'000 Mails annimmt und dann drei Stunden abarbeitet, besteht den Annahmetest und fällt trotzdem durch.

Ein Test, der nur die Annahmerate misst, sagt über eine mehrstufige Umgebung mit Gateway, Verschlüsselungsstufe und Zielserver wenig aus. Gerade dort entscheidet die Ende-zu-Ende-Sicht.

## Werkzeuge unter Linux

**smtp-source und smtp-sink** aus dem Postfix-Paket sind der Standard für rohe SMTP-Last und auf praktisch jedem System verfügbar, das Postfix installiert hat. `smtp-source` erzeugt Nachrichten mit einstellbarer Grösse, Parallelität und Anzahl, `smtp-sink` ist das Gegenstück: ein SMTP-Server, der alles annimmt und verwirft. Ein Burst von 10'000 Mails mit 50 parallelen Sessions und 5-KB-Nachrichten ist ein Einzeiler:

```bash
time smtp-source -s 50 -m 10000 -l 5120 -c -f last@test.example -t senke@test.example gateway.test.example:25
```

Die Option `-c` zählt die abgesetzten Nachrichten live mit, `time` liefert die Gesamtdauer und damit die Rate. Wichtige Grenzen: `smtp-source` misst keine Latenz-Perzentile und die Nachrichten sind synthetisch gleichförmig. Für die Frage "wie viel nimmt das System maximal an" ist es dennoch erste Wahl, weil es selbst auf schwacher Hardware zehntausende Nachrichten pro Minute erzeugt und der Generator praktisch nie zum Engpass wird.

**Postal** ist der klassische dedizierte Mailserver-Benchmark unter Linux. Es variiert Absender, Empfänger und Nachrichtengrösse automatisch, hält eine Zielrate über lange Zeiträume und schreibt minütliche Statistiken. Damit eignet es sich besser als `smtp-source` für Soak-Tests, also Dauerlast über Stunden. Das zugehörige `bhm` (Black Hole Mailer) übernimmt die Rolle der Senke. Postal ist alt, aber genau dafür gebaut und in den Paketquellen der meisten Distributionen enthalten.

**swaks** ist kein Lastgenerator, gehört aber in jeden Testplan. Es spricht eine einzelne SMTP-Transaktion mit voller Kontrolle über jeden Schritt: Authentifizierung, STARTTLS, beliebige Header, Anhänge. Vor jedem Lasttest gehört ein swaks-Durchlauf als Funktionstest, damit der Burst nicht an einem falschen Empfänger oder einem TLS-Problem scheitert und die Messung wertlos macht. In einer Schleife mit `xargs -P` lässt sich swaks auch als kleiner Lastgenerator missbrauchen, für zehntausende Mails ist der Prozess-Overhead aber zu gross.

**Eigene Skripte** in Python (smtplib, aiosmtplib) oder Go sind der Weg, wenn der Test realistische Nachrichtenmixe braucht: unterschiedliche Grössen, echte Anhänge, variierende Empfängerzahlen pro Transaktion, gezielte Fehlerfälle. Der Aufwand ist höher, dafür misst das Skript genau das, was die eigene Umgebung später sieht, und kann pro Nachricht Zeitstempel für die Latenzauswertung mitschreiben.

## Werkzeuge unter Windows

**Apache JMeter** ist unter Windows die erste Empfehlung. Der eingebaute SMTP Sampler beherrscht Auth, STARTTLS, Anhänge und EML-Dateien als Nachrichtenquelle, und die JMeter-Mechanik liefert das, was den Postfix-Tools fehlt: Thread-Gruppen für gestufte Lastprofile, Antwortzeit-Perzentile, Fehlerraten und Reports. Für Bursts jenseits von ein paar tausend Mails pro Minute gilt die übliche JMeter-Regel: GUI nur zum Erstellen des Testplans, die Messung selbst im CLI-Modus fahren, sonst misst man die Oberfläche mit.

**PowerShell mit MailKit** ist der Bordmittel-Weg. Das früher übliche `Send-MailMessage` markiert Microsoft selbst als veraltet und empfiehlt den Umstieg; MailKit lässt sich per NuGet laden und aus PowerShell 7 heraus mit Runspaces parallelisieren. Realistisch sind damit einige hundert bis wenige tausend Mails pro Minute, genug für Funktions- und Regressionstests, zu wenig für die Maximallastmessung. Der Vorteil: Das Skript läuft ohne Zusatzinstallation auf jedem Admin-Arbeitsplatz und kann Ergebnisse direkt als CSV für die Auswertung schreiben.

**Microsoft Exchange Load Generator (LoadGen)** war jahrelang das offizielle Werkzeug, um Exchange-Umgebungen mit simulierten Benutzerprofilen (Outlook, ActiveSync, OWA) zu belasten. Microsoft hat es nach Exchange 2013 nicht weitergepflegt und den Download eingestellt. Für reine SMTP-Last war LoadGen ohnehin das falsche Werkzeug; wer heute Exchange-Postfachlast simulieren will, steht ohne offizielles Tool da und testet den SMTP-Pfad besser direkt.

**WSL** verdient einen eigenen Punkt: Wer auf einer Windows-Maschine sitzt, aber Linux-Tools braucht, kann `smtp-source` und Postal in einer WSL-Distribution installieren und hat damit die vollen Linux-Werkzeuge ohne separate Test-VM. Für die hier diskutierten Lasten ist der WSL-Netzwerkpfad kein relevanter Engpass.

## Vergleich

| Tool | Plattform | Stärke | Grenze |
|---|---|---|---|
| smtp-source / smtp-sink | Linux (Postfix) | Maximale Rohlast mit minimalem Aufwand, Generator und Senke aus einer Hand | Keine Latenz-Perzentile, gleichförmige Nachrichten |
| Postal / bhm | Linux | Dauerlast mit Zielrate, variierende Nachrichten, Minutenstatistik | Betagtes Tooling, Auswertung selbst bauen |
| swaks | Linux, Windows (Perl) | Voll kontrollierbarer Einzeltest, ideal als Funktionscheck vor dem Burst | Kein Lastgenerator |
| JMeter (SMTP Sampler) | Windows, Linux (Java) | Lastprofile, Perzentile, Reports, EML-Nachrichtenquellen | Java-Overhead, GUI-Falle bei hohen Raten |
| PowerShell + MailKit | Windows | Ohne Zusatzinstallation auf jedem Admin-Rechner, CSV-Ausgabe | Durchsatz begrenzt, Parallelisierung selbst bauen |
| Eigenes Skript (Python/Go) | beide | Realistischer Nachrichtenmix, eigene Messpunkte | Entwicklungsaufwand, Generator selbst validieren |

## Die Senke: wohin mit den Mails

Die unterschätzte Hälfte des Testaufbaus ist das Ziel. Drei Varianten haben sich bewährt:

- **smtp-sink** oder `bhm` als schwarzes Loch: nimmt alles an, verwirft alles, misst die reine Transportkette. `smtp-sink` kann auf Wunsch künstliche Antwortverzögerungen und Fehlercodes erzeugen und damit auch das Verhalten des Testsystems bei einem langsamen oder fehlerhaft antwortenden Ziel prüfen.
- **Postfix mit discard-Transport** als realistischere Senke, wenn das Ziel selbst ein vollwertiger SMTP-Server mit Queueing sein soll.
- **Einige wenige echte Seed-Postfächer** zusätzlich zur Senke, um stichprobenartig zu prüfen, dass Nachrichten inhaltlich unversehrt ankommen, inklusive Verschlüsselungs- oder Signaturstufe.

Werkzeuge mit Web-Oberfläche wie Mailpit sind für die Entwicklung gedacht und bei zehntausenden Mails schnell selbst der Engpass. Als Senke für einen Lasttest sind sie ungeeignet; die Messung würde das Analysewerkzeug statt das Testsystem vermessen.

## Den Test planen

Ein belastbarer Test läuft in drei Stufen, jede mit eigener Fragestellung:

1. **Baseline:** Eine moderate, bekannte Last (etwa 10 Prozent der erwarteten Spitze) über einige Minuten. Sie liefert die Referenzwerte für Latenz und Ressourcenverbrauch und deckt Konfigurationsfehler auf, bevor sie in der Burstmessung untergehen.
2. **Burst:** Die eigentliche Spitzenlastmessung, zum Beispiel 10'000 bis 50'000 Mails so schnell wie möglich oder mit definierter Zielrate. Mehrere Durchläufe mit steigender Parallelität zeigen, wo die Annahmerate abflacht und die Latenz kippt.
3. **Soak:** Die erwartete Tageslast über mehrere Stunden. Erst hier zeigen sich Speicherlecks, volllaufende Spool-Partitionen, Log-Rotation unter Last und Verbindungs-Limits, die ein kurzer Burst nie erreicht.

Beim Nachrichtenmix gilt: so realistisch wie nötig. Eine Messung mit ausschliesslich 5-KB-Textmails überschätzt den Durchsatz einer Umgebung, deren Alltag aus PDF-Anhängen besteht, um ein Mehrfaches. Sinnvoll ist ein Mix aus dem eigenen Bestand, etwa 70 Prozent klein, 25 Prozent mit typischem Anhang, 5 Prozent gross. Ebenso gehört TLS in den Test, wenn die Produktion TLS spricht: Der Handshake kostet pro Verbindung deutlich mehr als die Nachrichtenübertragung selbst, und Generatoren, die pro Mail eine neue Verbindung öffnen, messen sonst primär die TLS-Terminierung.

Für die spätere Auswertung bekommt jede Testnachricht einen eindeutigen Marker, am einfachsten einen eigenen Header wie `X-Loadtest-Id` mit Lauf-Nummer und Zeitstempel und eine wiedererkennbare Betreffkonvention. Damit lassen sich Testläufe in den Logs sauber voneinander und vom übrigen Verkehr trennen, und die Testmails lassen sich nach dem Lauf gezielt aus Quarantänen und Journalen räumen.

Drei Punkte, die in der Planung regelmässig vergessen gehen: Erstens Rate-Limits und Tarpitting auf dem Testpfad; ein Gateway, das nach 100 Mails pro Minute pro Quell-IP drosselt, testet sonst nur seine eigene Drossel (für die Maximallastmessung gezielt ausnehmen, für den Realitätscheck bewusst drinlassen). Zweitens DNS: Wenn das Testsystem für jede Nachricht Empfängerdomains auflöst oder DNSBL-Abfragen stellt, gehört ein Resolver mit in die Testumgebung, sonst misst der Test den Upstream-DNS. Drittens der Generator selbst: Vor dem ersten Lauf gegen das Zielsystem gehört ein Lauf des Generators direkt gegen die Senke, um zu belegen, dass der Generator die Zielrate überhaupt erzeugen kann.

## Auswerten

Die Messwerte des Lastgenerators sind nur die halbe Wahrheit, denn sie zeigen die Sicht des Einlieferers. Die andere Hälfte steht in den Logs des Testsystems.

Unter Postfix liefert das Maillog pro Nachricht die Felder `delay` und `delays`, letzteres aufgeschlüsselt nach Zeit in der Queue, Verbindungsaufbau und Übertragung. Eine Auswertung über einen Testlauf ist mit Bordmitteln erledigt:

```bash
grep "status=sent" /var/log/mail.log |
  grep -o "delay=[0-9.]*" | cut -d= -f2 | sort -n |
  awk '{a[NR]=$1} END {print "n="NR, "p50="a[int(NR*0.5)], "p95="a[int(NR*0.95)], "p99="a[int(NR*0.99)], "max="a[NR]}'
```

Auf Exchange-Seite ist das Message Tracking Log die zentrale Quelle. Für einen Testlauf mit Betreffkonvention:

```powershell
$p = @{
    Start          = "24.08.2026 14:00"
    End            = "24.08.2026 15:00"
    MessageSubject = "LOADTEST"
    ResultSize     = "Unlimited"
}
Get-MessageTrackingLog @p | Group-Object EventId |
    Sort-Object Count -Descending | Format-Table Name, Count
```

Die Differenz der Zeitstempel zwischen RECEIVE- und DELIVER-Ereignis derselben MessageId ergibt die Ende-zu-Ende-Latenz pro Nachricht; exportiert als CSV lässt sich daraus die Perzentilverteilung rechnen.

Bei der Interpretation zählen drei Grundsätze. Erstens: Perzentile statt Mittelwerte. Ein Durchschnitt von zwei Sekunden kann bedeuten, dass alles zwei Sekunden dauert, oder dass 95 Prozent in einer halben Sekunde durch sind und der Rest in der Queue hing; p50, p95 und p99 unterscheiden diese Fälle. Zweitens: SMTP-Antwortcodes pivotieren. Die Verteilung der 4xx-Antworten über die Zeit zeigt, wann das System zu drosseln beginnt, und welche Codes es sind (Verbindungs-Limit, Queue-Schutz, Greylisting) zeigt, welcher Mechanismus zuerst greift. Drittens: Queue-Tiefe über die Zeit auftragen, unter Postfix per `qshape` beziehungsweise `postqueue -j`, auf Exchange per `Get-Queue` im Minutentakt. Die Fläche unter dieser Kurve, nicht die Annahmerate, entscheidet, ob die Umgebung einen Burst wegsteckt oder ihn nur einlagert.

Parallel zu den Maillogs gehören die Systemmetriken des Testsystems in die Auswertung: CPU, I/O-Wartezeiten auf der Spool-Partition, Dateideskriptoren, Verbindungszähler. Der häufigste Befund bei mehrstufigen Umgebungen ist, dass nicht der Mailprozess limitiert, sondern eine Content-Inspection-Stufe (Virenscanner, Verschlüsselungsmodul, DLP) mit fixer Worker-Zahl. Genau solche Befunde sind der eigentliche Wert des Tests: Sie benennen die Stellschraube, bevor die Produktion sie findet.

## Fazit

Für die schnelle Maximallastmessung unter Linux führt kein Weg an `smtp-source` mit `smtp-sink` vorbei; Postal ergänzt den Dauerlastfall. Unter Windows liefert JMeter die vollständigste Messung, PowerShell mit MailKit deckt Funktions- und Regressionstests ab, und WSL holt bei Bedarf die Linux-Werkzeuge auf den Admin-Arbeitsplatz. Wichtiger als das Tool ist der Plan: getrennte Messung von Annahme, Latenz und Queue-Verhalten, ein realistischer Nachrichtenmix, ein markierter Testlauf und eine Auswertung, die Perzentile und Logs des Zielsystems einbezieht statt nur den Zähler des Generators.

## Quellen

1.  [smtp-source(1), Postfix-Manual](https://www.postfix.org/smtp-source.1.html): Referenz des Lastgenerators mit allen Optionen für Parallelität, Nachrichtengrösse und TLS.

2.  [smtp-sink(1), Postfix-Manual](https://www.postfix.org/smtp-sink.1.html): Referenz der Mail-Senke, inklusive künstlicher Verzögerungen und Fehlerantworten.

3.  [Postal-Dokumentation, Russell Coker](https://doc.coker.com.au/projects/postal/): Beschreibung des Mailserver-Benchmarks mit Zielrate, Nachrichtenvariation und bhm-Senke.

4.  [swaks, John Jetmore](https://www.jetmore.org/john/code/swaks/): Der SMTP-Funktionstester für den Vorab-Check jedes Testpfads.

5.  [Apache JMeter Component Reference: SMTP Sampler](https://jmeter.apache.org/usermanual/component_reference.html): Funktionsumfang des SMTP Samplers inklusive Auth, TLS und EML-Quellen.

6.  [Send-MailMessage, Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/send-mailmessage): Offizieller Hinweis von Microsoft, dass das Cmdlet veraltet ist, mit Verweis auf Alternativen wie MailKit.

7.  [MailKit, Jeffrey Stedfast](https://github.com/jstedfast/MailKit): Die .NET-Mailbibliothek für eigene Versandskripte unter PowerShell 7.

8.  [Get-MessageTrackingLog, Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog): Referenz für die Auswertung des Exchange Message Tracking Logs nach einem Testlauf.

9.  [qshape(1), Postfix-Manual](https://www.postfix.org/qshape.1.html): Werkzeug zur Analyse der Queue-Verteilung während und nach dem Burst.
