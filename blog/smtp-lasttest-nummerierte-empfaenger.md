---
title: "SMTP-Lasttest mit nummerierten Empfängern: 50'000 Mails nachvollziehbar versenden"
navTitle: "Nummerierte Lasttests"
description: "Ein Lasttest ist nur so gut wie seine Auswertung. smtp-source nummeriert mit der Option -N jede Mail über die Empfängeradresse durch, ohne den Durchsatz zu opfern. Wie der Lauf mit 50'000 Mails aufgebaut wird, wie viele Sessions sinnvoll sind und wie fehlende Nummern automatisch gefunden werden."
date: "2026-08-27"
kategorie: "SMTP und Mailflow"
timeToRead: "8 Min. Lesezeit"
themen:
  - "smtp-lasttests"
  - "smtp-mailflow"
produkte:
  - "uebergreifend"
protokolle:
  - "smtp"
  - "testing"
slug: "smtp-lasttest-nummerierte-empfaenger"
translationId: "article-57f09c758baf6e1e"
url: "https://rafaelpfister.ch/blog/smtp-lasttest-nummerierte-empfaenger"
---
# SMTP-Lasttest mit nummerierten Empfängern: 50'000 Mails nachvollziehbar versenden

Wer einen Lasttest mit 50'000 Mails fährt, will hinterher zwei Fragen beantworten können: Sind alle angekommen, und wenn nicht, welche fehlen? Mit identischen Testmails lässt sich nur zählen, und eine Differenz von 49'987 zu 50'000 sagt nichts darüber, wann und wo die 13 fehlenden Nachrichten verloren gingen. Trägt dagegen jede Mail eine fortlaufende Nummer, wird aus dem Zählen ein Abgleich: Jede Nummer ist in den Logs des Zielsystems einzeln auffindbar, Lücken zeigen den Zeitpunkt des Verlusts, und die Reihenfolge der Zustellung lässt sich prüfen.

Die verbreitete Reflexreaktion ist ein Skript, das den Betreff hochzählt. Das funktioniert, kostet aber Durchsatz, denn der Lastgenerator `smtp-source` aus dem Postfix-Paket setzt den Betreff pro Aufruf fix, und eine Schleife mit einem Aufruf pro Mail erzwingt für jede Nachricht einen eigenen Verbindungsaufbau. Die bessere Nachrichtenkennung ist bereits eingebaut: Die Option `-N` nummeriert die Empfängeradresse pro Nachricht durch, und zwar innerhalb eines einzigen Aufrufs mit parallelen Sessions. Für die Auswertung ist die Empfängeradresse genauso brauchbar wie der Betreff, denn sie steht in jedem Tracking-Log.

Dieser Testaufbau sendet, anders als ein reiner Loopback-Funktionstest, an ein anderes System über das Netz. Falls auf dem Quellsystem kein Postfix installiert ist, zeigt der Beitrag [smtp-source ohne Postfix-Installation](https://rafaelpfister.ch/blog/smtp-source-ohne-postfix-installation), wie Sie die Werkzeuge aus dem RPM entpacken.

## Die wichtigsten Optionen von smtp-source

Zur Orientierung vorab die Optionen, die in diesem Beitrag vorkommen, sinngemäss aus der Manpage übersetzt:

| Option | Bedeutung |
|---|---|
| `-s n` | Anzahl paralleler SMTP-Sessions (Standard: 1) |
| `-m n` | Anzahl zu sendender Nachrichten insgesamt (Standard: 1) |
| `-l n` | Grösse des Nachrichtentexts in Bytes, ohne Header |
| `-f adresse` | Absenderadresse |
| `-t adresse` | Empfängeradresse (Standard: `foo@hostname`) |
| `-S text` | Betreffzeile, fix für alle Nachrichten des Aufrufs |
| `-F datei` | Sendet Header und Body unverändert aus einer Datei; verdrängt `-l` und `-S` |
| `-N` | Nummeriert die Empfängeradresse pro Nachricht durch (per-Prozess-Zähler; Position und Startwert versionsabhängig, siehe unten) |
| `-r n` | Anzahl Empfänger pro Nachricht (Standard: 1), Adressbildung wie bei `-N` |
| `-d` | Nach einer Nachricht nicht trennen, die nächste über dieselbe Verbindung senden |
| `-c` | Laufenden Zähler anzeigen, der mit jedem abgeschlossenen `DATA` hochzählt |
| `-w n` | Feste Wartezeit von n Sekunden zwischen Nachrichten (pro Session) |
| `-v` | Ausführliche Ausgabe für die Fehlersuche |
| `host:port` | Ziel der Einlieferung via TCP; ohne Portangabe der Standardport smtp |

Die vollständige Liste inklusive TLS-, LMTP- und Timing-Optionen steht in der Manpage `smtp-source(1)`; das Gegenstück für die Empfangsseite ist `smtp-sink(1)` und kommt bei der Auswertung weiter unten zum Einsatz.

## Wie -N die Empfänger nummeriert

`-N` aktiviert einen per-Prozess-Zähler, der in die Empfängeradresse eingebaut wird. Drei Eigenschaften bestimmen den Testaufbau, alle drei sind im Quellcode von `smtp-source.c` nachlesbar:

Erstens hängt die genaue Adressform von der Postfix-Version ab. Postfix 3.5, wie es RHEL 8 ausliefert, stellt die Nummer der gesamten Adresse voran (`RCPT TO:<%d%s>`): Aus `-t test@example.com` werden `1test@example.com`, `2test@example.com` und so weiter, und der Zähler beginnt bei 1. Aktuelle Postfix-Versionen fügen die Nummer stattdessen am Ende des Local-Parts ein und beginnen bei 0 (`test0@` bis `test49999@`); für diese Variante empfiehlt die Manpage Plus-Adressierung (`-t 'test+@example.com'` ergibt `test+0@` und folgende), damit ein Zielsystem mit Subadressierung alles demselben Postfach zuordnet. Prüfen Sie die Form vor dem grossen Lauf mit einer Handvoll Mails gegen einen `smtp-sink` oder im Log des Ziels; davon hängen die Sollmenge und das Suchmuster der Auswertung ab.

Zweitens ist der Zähler prozessweit und wird von allen parallelen Sessions geteilt. Bei `-s 8` vergeben die acht Sessions die Nummern gemeinsam, jede Nummer kommt genau einmal vor. Die Reihenfolge über die Sessions hinweg ist nicht deterministisch, die Vollständigkeit der Nummernmenge aber garantiert.

Drittens ist der Startwert nicht konfigurierbar: 1 bei Postfix 3.5, 0 bei aktuellen Versionen. Die 50'000 Mails tragen also die Nummern 1 bis 50'000 beziehungsweise 0 bis 49'999, und die Sollmenge für den Abgleich muss dazu passen.

## Der Testlauf in einem Aufruf

```bash
smtp-source -c -d -N -s 8 -m 50000 -l 5120 \
  -f lasttest@example.com \
  -t test@example.com \
  gateway.example.com:25
```

| Option | Wirkung |
|---|---|
| `-c` | Laufender Zähler abgeschlossener Zustellungen als einzeilige Fortschrittsanzeige |
| `-d` | Verbindungen bleiben über alle Nachrichten stehen; ohne `-d` neue Verbindung pro Nachricht |
| `-N` | Empfänger-Nummerierung: hängt den per-Prozess-Zähler an den Local-Part an |
| `-s 8` | Acht parallele SMTP-Sessions |
| `-m 50000` | Gesamtzahl der Nachrichten, verteilt auf die Sessions |
| `-l 5120` | Nachrichtengrösse in Bytes (ohne Header), hier 5 KB |
| `-f` | Absenderadresse |
| `-t` | Empfänger-Basisadresse; `-N` macht daraus `1test@` bis `50000test@` (Postfix 3.5) bzw. `test0@` bis `test49999@` (aktuelle Versionen) |
| `gateway.example.com:25` | Zielhost und Port |

`-d` ist für das Lastbild entscheidend: Ohne diese Option trennt `smtp-source` nach jeder Nachricht die Verbindung und baut für die nächste eine neue auf; mit `-d` bleiben die acht Verbindungen stehen und liefern nacheinander alle Nachrichten aus, wie es ein Massenversender tut.

Bewusst fehlt das aus Funktionstests bekannte `-v`: Es protokolliert jeden einzelnen SMTP-Dialog von `HELO` bis `QUIT` und erzeugt bei 50'000 Mails hunderttausende Logzeilen ohne Mehrwert für die Auswertung. `-c` liefert stattdessen die Zusammenfassung, an der sich der Fortschritt des Laufs live ablesen lässt. Die Gesamtdauer für die Ratenberechnung liefert ein vorangestelltes `time`.

Voraussetzung für den ganzen Ansatz: Das Zielsystem nimmt die generierten Adressen an. Ein `smtp-sink`, eine Catch-all-Domain, eine Verwerf-Domain des Providers oder ein Gateway, das Empfänger erst nach der Annahme auflöst, erfüllen das. Prüft das Ziel dagegen jeden Empfänger gegen ein Verzeichnis, lehnt es die nummerierten Adressen ab, und es bleibt nur die Betreff-Variante.

## Eigene Header setzen

Manche Lasttests brauchen einen eigenen Header, etwa als Marker, an dem das Gateway die Testmails erkennt oder eine Regel greift. Eine Option dafür kennt `smtp-source` nicht, aber `-F` liest eine vollständig vorformatierte Nachricht aus einer Datei, und dort steht jeder gewünschte Header. Die Datei besteht aus den Header-Zeilen, einer Leerzeile und dem Body, alle Zeilen mit `\r\n` beendet:

```bash
{ printf 'X-Lasttest: aktiv\r\n'
  printf 'Subject: Lasttest\r\n'
  printf '\r\n'
  head -c 5120 /dev/zero | tr '\0' 'x'
  printf '\r\n'; } > lasttest.eml
```

```bash
smtp-source -c -d -N -s 8 -m 50000 -F lasttest.eml \
  -f lasttest@example.com \
  -t test@example.com \
  gateway.example.com:25
```

| Option | Wirkung |
|---|---|
| `-F datei` | Sendet Header und Body unverändert aus der Datei; ersetzt den generierten Nachrichteninhalt |

Zwei Konsequenzen: `-F` verdrängt `-l` und `-S`, weil Grösse und Betreff nun aus der Datei kommen (beides gehört deshalb hinein). `-N` bleibt dagegen wirksam, die Empfänger werden weiter durchnummeriert; der Header ist in allen Nachrichten identisch, da er aus der festen Datei stammt.

## Wie viele Sessions?

Die passende Session-Zahl ermitteln Sie am zuverlässigsten durch Messung. Ein kurzer Kalibrierlauf mit steigender Session-Zahl und einem Bruchteil der Mailmenge zeigt, ab wann zusätzliche Sessions nichts mehr bringen:

```bash
for s in 1 2 4 8 16 32; do
  t0=$(date +%s%N)
  smtp-source -d -N -s "$s" -m 2000 -l 5120 \
    -f lasttest@example.com -t test@example.com \
    gateway.example.com:25
  t1=$(date +%s%N)
  echo "$s Sessions: $(( 2000000000000 / (t1 - t0) )) Mails/s"
done
```

Die Rate steigt anfangs etwa linear mit der Session-Zahl, weil parallele Sessions die Wartezeit auf die Antworten des Ziels überdecken. Ab einem bestimmten Punkt flacht die Kurve ab: Dann ist entweder das Zielsystem gesättigt oder die Quelle am Limit. Nehmen Sie den kleinsten Wert, bei dem die Rate nicht mehr nennenswert zulegt; noch mehr Sessions erhöhen dann nur die Last durch Parallelität, nicht den Durchsatz. Typische Ergebnisse liegen im einstelligen bis niedrigen zweistelligen Bereich. Beachten Sie dabei die 2'000 Kalibrier-Mails pro Stufe: Auch die brauchen eine kontrollierte Empfängeradresse, und die Nummerierung beginnt in jedem Aufruf wieder beim Startwert.

## Auswertung auf der Empfangsseite

Steht auf dem Zielsystem ein eigener Testempfänger, übernimmt `smtp-sink` die Protokollierung gleich mit:

```bash
smtp-sink -c -d "mails/%Y%m%d-%H%M%S." 0.0.0.0:2525 200
```

| Option | Wirkung |
|---|---|
| `-c` | Laufende Zähler statt des vollen SMTP-Dialogs |
| `-d "mails/…"` | Beim Sink: Dump, nicht Verbindungshaltung. Schreibt jede angenommene Nachricht in eine eigene Datei (Namensmuster per strftime), inklusive eines `X-Rcpt-Args`-Headers mit der Empfängeradresse |
| `0.0.0.0:2525` | Lauscht auf allen Interfaces auf Port 2525 |
| `200` | Backlog: maximale Länge der Warteschlange wartender Verbindungen gemäss listen(2) |

Nach dem Lauf extrahieren Sie die empfangenen Nummern und vergleichen sie mit der Sollmenge. Da die Nummern keine führenden Nullen tragen, werden beide Listen vor dem Abgleich auf eine feste Stellenzahl gebracht, damit die alphabetische Sortierung von `comm` der numerischen entspricht. Das Suchmuster passt zur Adressform von Postfix 3.5 (Nummer vor der Adresse); bei aktuellen Versionen entsprechend `test[0-9]+@` und `seq 0 49999`:

```bash
grep -rhoE '[0-9]+test@example\.com' mails/ | \
  grep -oE '^[0-9]+' | sort -u | \
  awk '{printf "%08d\n", $1}' | sort > empfangen.txt
```

```bash
seq 1 50000 | awk '{printf "%08d\n", $1}' | \
  comm -23 - empfangen.txt
```

`comm -23` gibt genau die Nummern aus, die in der Sollmenge stehen, aber nicht in der Empfangsliste: die fehlenden Mails. Eine leere Ausgabe bedeutet vollständige Zustellung. Tauchen Nummern doppelt auf (erkennbar am Unterschied zwischen `sort` und `sort -u`), hat unterwegs ein System die Nachricht dupliziert, was ebenfalls ein Befund ist.

Ist das Ziel ein produktnahes System statt eines smtp-sink, übernimmt dessen Logging die Rolle der Dump-Dateien. Auf einem Exchange-Server etwa liefert `Get-MessageTrackingLog -Recipients` beziehungsweise ein Filter auf die Empfängeradresse die angekommenen Nummern, auf einem Postfix-System ein `grep` auf `to=` und die Basisadresse über das Maillog. Genau das ist der Vorteil der Nummer in der Adresse: Der Empfänger steht in jedem Message-Tracking, während der Betreff dort je nach System fehlt oder erst eingeschaltet werden muss.

## Wenn die Nummer im Betreff stehen muss

Manche Auswertungen hängen am Betreff, etwa wenn das Zielsystem Empfängeradressen umschreibt oder die Logs den Empfänger nur maskiert zeigen. Dann bleibt die Schleifen-Variante: ein `smtp-source`-Aufruf pro Mail mit `-m 1` und einem Betreff, den die Shell hochzählt, verteilt auf mehrere parallele Worker mit zusammenhängenden Nummernbereichen.

```bash
worker() {
  local i
  for ((i = $1; i <= $2; i++)); do
    smtp-source -s 1 -m 1 -l 5120 \
      -S "$(printf 'Lasttest %05d' "$i")" \
      -f lasttest@example.com -t test@example.com \
      gateway.example.com:25 || echo "$i" >> fehlend.log
  done
}
for w in 0 1 2 3; do
  worker $(( w * 12500 + 1 )) $(( (w + 1) * 12500 )) &
done
wait
```

Der Preis ist ein vollständiger Verbindungsaufbau pro Mail: TCP-Handshake, Banner, `HELO`, Versand, `QUIT`. Dieser Lauf misst damit nicht den maximalen Durchsatz des Zielsystems, sondern einen bewusst verbindungsintensiven Fall. Die Worker-Zahl ermitteln Sie analog zum Kalibrierlauf oben, nur mit der Worker-Schleife statt `-s`. Die führenden Nullen im Betreff ersparen beim Abgleich das Umformatieren, das die `-N`-Variante braucht.

## Regeln für Tests gegen andere Systeme

Sobald der Test das eigene System verlässt, gelten drei Bedingungen. Erstens: Der Betreiber des Zielsystems weiss Bescheid und hat dem Zeitfenster zugestimmt; 50'000 Mails sehen für jedes Monitoring wie ein Angriff oder eine Spamwelle aus. Zweitens: Die Empfängeradresse endet kontrolliert, in einem dedizierten Testpostfach, einer Verwerf-Regel auf dem Ziel oder einer vom Provider dafür vorgesehenen Verwerf-Domain; produktive Adressen gehören nicht in einen Lasttest. Drittens: Ein Abbruchkriterium steht vor dem Start fest, etwa eine wachsende Queue auf dem Ziel oder eine Fehlerrate über einem Schwellwert, und jemand beobachtet diese Werte während des Laufs.

Mit diesen drei Punkten und der Nummerierung liefert der Lauf am Ende nicht nur eine Durchsatzzahl, sondern eine belegbare Aussage: welche der 50'000 Mails angekommen sind, welche fehlen und wo auf der Strecke sie zuletzt gesehen wurden.

## Quellen

1.  [Postfix: smtp-source(1)](https://www.postfix.org/smtp-source.1.html): Manpage des Lastgenerators; beschreibt das `-N`-Verhalten der aktuellen Version (Zähler am Local-Part, Plus-Adressierung).

2.  [Postfix-Quellcode 3.5.8: smtp-source.c](https://github.com/vdukhovni/postfix/blob/v3.5.8/postfix/src/smtpstone/smtp-source.c): belegt für die RHEL-8-Version das Voranstellen der Nummer (`RCPT TO:<%d%s>`) mit Startwert 1; im [aktuellen Stand](https://github.com/vdukhovni/postfix/blob/master/postfix/src/smtpstone/smtp-source.c) wird die Nummer stattdessen am Local-Part angefügt, ab 0.

3.  [Postfix: smtp-sink(1)](https://www.postfix.org/smtp-sink.1.html): Manpage des Test-Empfängers mit den Dump-Optionen und den aufgezeichneten X-Headern.

4.  [GNU Coreutils: comm](https://www.gnu.org/software/coreutils/manual/html_node/comm-invocation.html): Mengenvergleich zweier sortierter Listen, hier für den Abgleich von Soll- und Empfangsnummern.
