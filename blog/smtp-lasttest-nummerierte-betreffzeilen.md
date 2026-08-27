---
title: "SMTP-Lasttest mit nummerierten Betreffzeilen: 50'000 Mails nachvollziehbar versenden"
navTitle: "Nummerierte Lasttests"
description: "Ein Lasttest ist nur so gut wie seine Auswertung. Wie Sie mit smtp-source 50'000 Mails mit durchnummerierten Betreffzeilen an ein Zielsystem senden, fehlende Nummern automatisch finden und was die Nummerierung an Durchsatz kostet."
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
  - "troubleshooting"
slug: "smtp-lasttest-nummerierte-betreffzeilen"
translationId: "article-47f507432200b9da"
url: "https://rafaelpfister.ch/blog/smtp-lasttest-nummerierte-betreffzeilen"
---
# SMTP-Lasttest mit nummerierten Betreffzeilen: 50'000 Mails nachvollziehbar versenden

Wer einen Lasttest mit 50'000 Mails fährt, will hinterher zwei Fragen beantworten können: Sind alle angekommen, und wenn nicht, welche fehlen? Mit identischen Testmails lässt sich nur zählen, und eine Differenz von 49'987 zu 50'000 sagt nichts darüber, wann und wo die 13 fehlenden Nachrichten verloren gingen. Trägt dagegen jede Mail eine fortlaufende Nummer im Betreff, wird aus dem Zählen ein Abgleich: Jede Nummer ist in den Logs des Zielsystems einzeln auffindbar, Lücken zeigen den Zeitpunkt des Verlusts, und die Reihenfolge der Zustellung lässt sich prüfen.

Dieser Testaufbau sendet, anders als ein reiner Loopback-Funktionstest, an ein anderes System über das Netz. Als Lastgenerator dient `smtp-source` aus dem Postfix-Paket; falls auf dem Quellsystem kein Postfix installiert ist, zeigt der Beitrag [smtp-source ohne Postfix-Installation](https://rafaelpfister.ch/blog/smtp-source-ohne-postfix-installation), wie Sie die Werkzeuge aus dem RPM entpacken.

## Warum smtp-source allein nicht reicht

`smtp-source` kennt mit `-S` eine Betreffzeile, setzt sie aber für den gesamten Aufruf fix: Alle Nachrichten eines Laufs tragen denselben Betreff. Eine Option, die den Betreff pro Nachricht hochzählt, gibt es nicht. Die einzige eingebaute Nummerierung ist `-N`, und die hängt einen Zähler an die Empfängeradresse an, nicht an den Betreff.

Für nummerierte Betreffzeilen bleibt deshalb nur eine Schleife: ein `smtp-source`-Aufruf pro Mail mit `-m 1` und einem Betreff, den die Shell hochzählt. Das verändert das Lastbild, denn jede Mail bekommt ihre eigene TCP-Verbindung samt SMTP-Handshake. Was das für die Messung bedeutet, folgt weiter unten; zuerst der Aufbau.

## Das Testskript

Die Eckdaten stehen am Anfang, damit sich der Lauf ohne Eingriff in die Logik anpassen lässt:

```bash
#!/usr/bin/env bash
set -u

ZIEL="gateway.example.com:25"
VON="lasttest@example.com"
AN="test@example.com"
GESAMT=50000
WORKER=4
GROESSE=5120
```

Die Nummern werden mit führenden Nullen formatiert (`00001` bis `50000`). Das ist kein Selbstzweck: Bei fester Stellenzahl entspricht die alphabetische Sortierung der numerischen, und der spätere Abgleich mit `comm` funktioniert ohne Umwege.

```bash
worker() {
  local i betreff
  for ((i = $1; i <= $2; i++)); do
    betreff=$(printf 'Lasttest %05d' "$i")
    smtp-source -s 1 -m 1 -l "$GROESSE" \
      -S "$betreff" -f "$VON" -t "$AN" \
      "$ZIEL" || echo "$i" >> fehlend.log
  done
}
```

Schlägt ein Sendeversuch fehl, landet die Nummer in `fehlend.log`. Damit ist schon auf der Sendeseite dokumentiert, welche Mails das Quellsystem gar nie verlassen haben; das Zielsystem muss diese Nummern beim Abgleich nicht suchen.

Ein einzelner sequenzieller Sender wäre durch die Netzwerklatenz begrenzt, deshalb teilen sich mehrere Worker den Nummernkreis in zusammenhängende Bereiche. Bei vier Workern sendet der erste die Nummern 1 bis 12'500, der zweite 12'501 bis 25'000 und so weiter:

```bash
PRO_WORKER=$(( GESAMT / WORKER ))
start=$(date +%s)
for ((w = 0; w < WORKER; w++)); do
  worker $(( w * PRO_WORKER + 1 )) \
         $(( (w + 1) * PRO_WORKER )) &
done
wait
dauer=$(( $(date +%s) - start ))
echo "$GESAMT Mails in $dauer Sekunden"
```

Die Aufteilung in Bereiche statt in verschränkte Nummern hat einen praktischen Grund: Bricht ein Worker ab, fehlt ein zusammenhängender Block, und der Lauf lässt sich mit angepassten Bereichsgrenzen fortsetzen, statt von vorn zu beginnen. Die auf der Sendeseite protokollierten Ausfälle senden Sie nach dem Lauf gezielt nach:

```bash
while read -r i; do
  smtp-source -s 1 -m 1 -l "$GROESSE" \
    -S "$(printf 'Lasttest %05d' "$i")" \
    -f "$VON" -t "$AN" "$ZIEL"
done < fehlend.log
```

## Auswertung auf der Empfangsseite

Steht auf dem Zielsystem ein eigener Testempfänger, übernimmt `smtp-sink` die Protokollierung gleich mit. Die Option `-d` schreibt jede angenommene Nachricht in eine eigene Datei, der Dateiname entsteht aus einem strftime-Muster plus einer eindeutigen Kennung:

```bash
smtp-sink -c -d "mails/%Y%m%d-%H%M%S." 0.0.0.0:2525 200
```

Bewusst fehlt hier das aus Funktionstests bekannte `-v`: Es protokolliert jeden einzelnen SMTP-Dialog von `HELO` bis `QUIT` und erzeugt bei 50'000 Mails hunderttausende Logzeilen ohne Mehrwert für die Auswertung. Stattdessen zeigt `-c` laufende Zähler an, die bei jedem Sitzungsende und jeder angenommenen Nachricht aktualisiert werden: eine einzeilige Zusammenfassung, an der sich der Fortschritt des Laufs live ablesen lässt. Dasselbe gilt für die Sendeseite: `smtp-source -c` blendet einen laufenden Zähler abgeschlossener Zustellungen ein, was vor allem beim Durchsatz-Lauf ohne Schleife (dazu unten) den Fortschritt sichtbar macht.

Nach dem Lauf extrahieren Sie die empfangenen Nummern aus den Betreffzeilen und vergleichen sie mit der Sollmenge:

```bash
grep -rhoE '^Subject: Lasttest [0-9]{5}' mails/ | \
  awk '{print $3}' | sort -u > empfangen.txt
```

```bash
seq -f '%05g' 1 50000 | comm -23 - empfangen.txt
```

`comm -23` gibt genau die Nummern aus, die in der Sollmenge stehen, aber nicht in der Empfangsliste: die fehlenden Mails. Eine leere Ausgabe bedeutet vollständige Zustellung. Tauchen Nummern doppelt auf (erkennbar am Unterschied zwischen `sort` und `sort -u`), hat unterwegs ein System die Nachricht dupliziert, was ebenfalls ein Befund ist.

Ist das Ziel ein produktnahes System statt eines smtp-sink, übernimmt dessen Logging die Rolle der Dump-Dateien. Auf einem Exchange-Server etwa liefert `Get-MessageTrackingLog -MessageSubject "Lasttest"` die angekommenen Nummern, auf einem Postfix-System ein `grep` über das Maillog. Entscheidend ist nur, dass der Betreff oder eine daraus abgeleitete Kennung in den Logs des Ziels auftaucht; genau dafür steht die Nummer im Betreff und nicht im Nachrichtentext.

## Was die Nummerierung an Durchsatz kostet

Die Schleife mit `-m 1` erzwingt pro Mail einen vollständigen Verbindungsaufbau: TCP-Handshake, SMTP-Banner, `HELO`, Versand, `QUIT`. Ein Massenversender mit stehenden Verbindungen erreicht auf derselben Strecke ein Mehrfaches davon, denn `smtp-source` liefert mit `-s` und `-m` in einem einzigen Aufruf viele Mails über gehaltene Sessions. Der nummerierte Test misst also nicht den maximalen Durchsatz des Zielsystems, sondern einen bewusst verbindungsintensiven Fall.

Daraus folgt eine Zweiteilung, die sich in der Praxis bewährt: Den reinen Durchsatz messen Sie mit `smtp-source` ohne Schleife (`-c -s 4 -m 50000` in einem Aufruf, alle Mails mit gleichem Betreff und laufendem Zähler statt Einzelprotokoll), die Vollständigkeit und Nachverfolgbarkeit mit dem nummerierten Lauf. Wer beides in einem Lauf will, kann auf die eingebaute Empfänger-Nummerierung `-N` ausweichen: Die Sessions bleiben stehen, und der Zähler steckt in der Empfängeradresse statt im Betreff, sofern das Zielsystem beliebige lokale Adressteile annimmt.

Mehr Durchsatz im nummerierten Lauf holen Sie über die Worker-Zahl. Da jeder Worker sequenziell arbeitet, skaliert die Rate bis zur Sättigung von CPU oder Zielsystem etwa linear mit `WORKER`; Werte zwischen 4 und 16 sind ein brauchbarer Startbereich.

## Regeln für Tests gegen andere Systeme

Sobald der Test das eigene System verlässt, gelten drei Bedingungen. Erstens: Der Betreiber des Zielsystems weiss Bescheid und hat dem Zeitfenster zugestimmt; 50'000 Mails sehen für jedes Monitoring wie ein Angriff oder eine Spamwelle aus. Zweitens: Die Empfängeradresse endet kontrolliert, in einem dedizierten Testpostfach, einer Verwerf-Regel auf dem Ziel oder einer vom Provider dafür vorgesehenen Verwerf-Domain; produktive Adressen gehören nicht in einen Lasttest. Drittens: Ein Abbruchkriterium steht vor dem Start fest, etwa eine wachsende Queue auf dem Ziel oder eine Fehlerrate über einem Schwellwert, und jemand beobachtet diese Werte während des Laufs.

Mit diesen drei Punkten und der Nummerierung liefert der Lauf am Ende nicht nur eine Durchsatzzahl, sondern eine belegbare Aussage: welche der 50'000 Mails angekommen sind, welche fehlen und wo auf der Strecke sie zuletzt gesehen wurden.

## Quellen

1.  [Postfix: smtp-source(1)](https://www.postfix.org/smtp-source.1.html): Manpage des Lastgenerators; belegt, dass `-S` den Betreff pro Aufruf fix setzt und `-N` nur Empfängeradressen nummeriert.

2.  [Postfix: smtp-sink(1)](https://www.postfix.org/smtp-sink.1.html): Manpage des Test-Empfängers mit den Dump-Optionen `-d` und `-D` samt strftime-Dateinamensmuster.

3.  [GNU Coreutils: comm](https://www.gnu.org/software/coreutils/manual/html_node/comm-invocation.html): Mengenvergleich zweier sortierter Listen, hier für den Abgleich von Soll- und Empfangsnummern.
