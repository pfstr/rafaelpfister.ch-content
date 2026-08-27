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

## Wie -N die Empfänger nummeriert

Die Manpage beschreibt `-N` als per-Prozess-Zähler, der an den Local-Part der Empfängeradresse angehängt wird. Drei Eigenschaften davon bestimmen den Testaufbau, alle drei sind im Quellcode von `smtp-source.c` nachlesbar:

Erstens wird die Nummer direkt zwischen Local-Part und `@domain` eingefügt. Aus `-t test@example.com` wird also `test0@example.com`, `test1@example.com` und so weiter. Damit die Adressen als Varianten eines einzigen Postfachs erkennbar bleiben, empfiehlt die Manpage Plus-Adressierung: Mit `-t 'test+@example.com'` entstehen `test+0@example.com` bis `test+49999@example.com`, und ein Zielsystem mit Subadressierung ordnet alles demselben Postfach zu.

Zweitens ist der Zähler prozessweit und wird von allen parallelen Sessions geteilt. Bei `-s 8` vergeben die acht Sessions die Nummern gemeinsam, jede Nummer kommt genau einmal vor. Die Reihenfolge über die Sessions hinweg ist nicht deterministisch, die Vollständigkeit der Nummernmenge aber garantiert.

Drittens beginnt der Zähler bei 0, nicht bei 1. Die 50'000 Mails tragen die Nummern 0 bis 49'999, und die Sollmenge für den Abgleich muss entsprechend lauten. Ein konfigurierbarer Startwert existiert nicht; wer zwingend bei 1 beginnen will, ist wieder beim Betreff-Skript am Ende dieses Beitrags.

## Der Testlauf in einem Aufruf

```bash
smtp-source -c -d -N -s 8 -m 50000 -l 5120 \
  -f lasttest@example.com \
  -t 'test+@example.com' \
  gateway.example.com:25
```

Die Optionen im Einzelnen: `-N` aktiviert die Empfänger-Nummerierung, `-s 8` fährt acht parallele Sessions, `-m 50000` verteilt die Gesamtmenge darauf, `-l 5120` setzt die Nachrichtengrösse auf 5 KB. `-d` ist für das Lastbild entscheidend: Ohne diese Option trennt `smtp-source` nach jeder Nachricht die Verbindung und baut für die nächste eine neue auf; mit `-d` bleiben die acht Verbindungen stehen und liefern nacheinander alle Nachrichten aus, wie es ein Massenversender tut.

Bewusst fehlt das aus Funktionstests bekannte `-v`: Es protokolliert jeden einzelnen SMTP-Dialog von `HELO` bis `QUIT` und erzeugt bei 50'000 Mails hunderttausende Logzeilen ohne Mehrwert für die Auswertung. Stattdessen zeigt `-c` einen laufenden Zähler abgeschlossener Zustellungen an: eine einzeilige Zusammenfassung, an der sich der Fortschritt des Laufs live ablesen lässt. Die Gesamtdauer für die Ratenberechnung liefert ein vorangestelltes `time`.

Voraussetzung für den ganzen Ansatz: Das Zielsystem nimmt die generierten Adressen an. Ein `smtp-sink`, eine Catch-all-Domain, eine Verwerf-Domain des Providers oder ein Gateway, das Empfänger erst nach der Annahme auflöst, erfüllen das. Prüft das Ziel dagegen jeden Empfänger gegen ein Verzeichnis, lehnt es die nummerierten Adressen ab, und es bleibt nur die Betreff-Variante.

## Wie viele Sessions?

Die passende Session-Zahl ermitteln Sie am zuverlässigsten durch Messung. Ein kurzer Kalibrierlauf mit steigender Session-Zahl und einem Bruchteil der Mailmenge zeigt, ab wann zusätzliche Sessions nichts mehr bringen:

```bash
for s in 1 2 4 8 16 32; do
  t0=$(date +%s%N)
  smtp-source -d -N -s "$s" -m 2000 -l 5120 \
    -f lasttest@example.com -t 'test+@example.com' \
    gateway.example.com:25
  t1=$(date +%s%N)
  echo "$s Sessions: $(( 2000000000000 / (t1 - t0) )) Mails/s"
done
```

Die Rate steigt anfangs etwa linear mit der Session-Zahl, weil parallele Sessions die Wartezeit auf die Antworten des Ziels überdecken. Ab einem bestimmten Punkt flacht die Kurve ab: Dann ist entweder das Zielsystem gesättigt oder die Quelle am Limit. Nehmen Sie den kleinsten Wert, bei dem die Rate nicht mehr nennenswert zulegt; noch mehr Sessions erhöhen dann nur die Last durch Parallelität, nicht den Durchsatz. Typische Ergebnisse liegen im einstelligen bis niedrigen zweistelligen Bereich. Beachten Sie dabei die 2'000 Kalibrier-Mails pro Stufe: Auch die brauchen eine kontrollierte Empfängeradresse, und die Nummern beginnen in jedem Aufruf wieder bei 0.

## Auswertung auf der Empfangsseite

Steht auf dem Zielsystem ein eigener Testempfänger, übernimmt `smtp-sink` die Protokollierung gleich mit. Die Option `-d` (beim Sink: Dump, nicht Verbindungshaltung) schreibt jede angenommene Nachricht in eine eigene Datei, inklusive eines `X-Rcpt-Args`-Headers mit der Empfängeradresse; `-c` zeigt auch hier laufende Zähler statt des vollen Dialogs:

```bash
smtp-sink -c -d "mails/%Y%m%d-%H%M%S." 0.0.0.0:2525 200
```

Nach dem Lauf extrahieren Sie die empfangenen Nummern und vergleichen sie mit der Sollmenge. Da die Nummern keine führenden Nullen tragen, werden beide Listen vor dem Abgleich auf eine feste Stellenzahl gebracht, damit die alphabetische Sortierung von `comm` der numerischen entspricht:

```bash
grep -rhoE 'test\+[0-9]+@example\.com' mails/ | \
  grep -oE '[0-9]+' | sort -u | \
  awk '{printf "%08d\n", $1}' | sort > empfangen.txt
```

```bash
seq 0 49999 | awk '{printf "%08d\n", $1}' | \
  comm -23 - empfangen.txt
```

`comm -23` gibt genau die Nummern aus, die in der Sollmenge stehen, aber nicht in der Empfangsliste: die fehlenden Mails. Eine leere Ausgabe bedeutet vollständige Zustellung. Tauchen Nummern doppelt auf (erkennbar am Unterschied zwischen `sort` und `sort -u`), hat unterwegs ein System die Nachricht dupliziert, was ebenfalls ein Befund ist.

Ist das Ziel ein produktnahes System statt eines smtp-sink, übernimmt dessen Logging die Rolle der Dump-Dateien. Auf einem Exchange-Server etwa liefert `Get-MessageTrackingLog -Recipients` beziehungsweise ein Filter auf die Empfängeradresse die angekommenen Nummern, auf einem Postfix-System ein `grep to=<test+` über das Maillog. Genau das ist der Vorteil der Nummer in der Adresse: Der Empfänger steht in jedem Message-Tracking, während der Betreff dort je nach System fehlt oder erst eingeschaltet werden muss.

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

1.  [Postfix: smtp-source(1)](https://www.postfix.org/smtp-source.1.html): Manpage des Lastgenerators; beschreibt `-N` als per-Prozess-Zähler am Local-Part und empfiehlt die Plus-Adressierung.

2.  [Postfix-Quellcode: smtp-source.c](https://github.com/vdukhovni/postfix/blob/master/postfix/src/smtpstone/smtp-source.c): belegt das Einfügen der Nummer zwischen Local-Part und Domain, den geteilten Zähler und den Startwert 0.

3.  [Postfix: smtp-sink(1)](https://www.postfix.org/smtp-sink.1.html): Manpage des Test-Empfängers mit den Dump-Optionen und den aufgezeichneten X-Headern.

4.  [GNU Coreutils: comm](https://www.gnu.org/software/coreutils/manual/html_node/comm-invocation.html): Mengenvergleich zweier sortierter Listen, hier für den Abgleich von Soll- und Empfangsnummern.
