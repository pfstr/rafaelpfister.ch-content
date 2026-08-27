---
title: "smtp-source ohne Postfix-Installation: Lasttest-Werkzeuge aus dem RPM entpacken"
navTitle: "smtp-source entpacken"
description: "smtp-source und smtp-sink gehören zu Postfix, laufen aber auch ohne installierten Mailserver. Wie Sie die beiden Werkzeuge auf RHEL aus dem Paket entpacken, warum die Ausführung aus /tmp an der Mount-Option noexec scheitern kann und welche Bibliotheken mitkommen müssen."
date: "2026-08-27"
kategorie: "SMTP und Mailflow"
timeToRead: "7 Min. Lesezeit"
themen:
  - "smtp-mailflow"
  - "smtp-lasttests"
produkte:
  - "uebergreifend"
protokolle:
  - "smtp"
  - "troubleshooting"
slug: "smtp-source-ohne-postfix-installation"
translationId: "article-d0a27da11509d24b"
url: "https://rafaelpfister.ch/blog/smtp-source-ohne-postfix-installation"
---
# smtp-source ohne Postfix-Installation: Lasttest-Werkzeuge aus dem RPM entpacken

Für SMTP-Lasttests ist `smtp-source` eine gute Wahl: Das Werkzeug öffnet parallele Sessions, hält sie über mehrere Nachrichten offen und bildet damit das Verbindungsverhalten eines Massenversenders deutlich realistischer ab als Testwerkzeuge, die pro Mail eine neue Verbindung aufbauen. Das Gegenstück `smtp-sink` nimmt Mails an und verwirft sie, ohne etwas zuzustellen. Beide gehören zum Lieferumfang von Postfix.

Genau da liegt das Problem: Auf dem System, von dem aus Sie testen wollen, ist oft kein Postfix installiert. Auf einer Mailgateway-Appliance ist eine Installation auch nicht erwünscht, denn ein zusätzlicher Postfix bringt eine eigene Konfiguration unter `/etc/postfix` und einen Systemdienst mit, der im ungünstigsten Fall Port 25 belegt und damit das eigentliche Mailsystem blockiert. Dazu kommt die Frage, was der Hersteller-Support von nachinstallierten Paketen auf seiner Appliance hält.

Beide Werkzeuge lassen sich aber auch ohne Installation betreiben: RPM herunterladen, Binaries samt Bibliotheken entpacken, fertig. Der Weg dorthin hat zwei Eigenheiten, die dieser Beitrag anhand eines RHEL-8-Systems zeigt. Root-Rechte brauchen Sie dafür nicht, nur Zugriff auf die Paketquellen.

## Ist smtp-source schon vorhanden?

Zuerst prüfen Sie, ob das Werkzeug nicht doch schon auf dem System liegt. `smtp-source` befindet sich je nach Distribution ausserhalb des normalen PATH:

```bash
command -v smtp-source || \
  ls /usr/sbin/smtp-source /usr/lib/postfix/sbin/smtp-source 2>/dev/null
```

Bleibt die Ausgabe leer, fehlt auch das zugehörige Paket. Auf RPM-Systemen bestätigen Sie das und prüfen zugleich, ob die Repositories Postfix anbieten:

```bash
rpm -qa | grep -i postfix
```

```bash
yum list available postfix
```

Auf dem Testsystem war kein Postfix installiert, das BaseOS-Repository bot aber `postfix-3.5.8-8.el8_10` an. Damit ist der Weg frei: Das Paket lässt sich herunterladen, ohne es zu installieren.

## Das RPM nur herunterladen

`yum download` (aus dem Plugin-Paket `dnf-plugins-core`, auf RHEL 8 üblicherweise vorhanden) lädt ein Paket in das aktuelle Verzeichnis, ohne es zu installieren. Das funktioniert ohne Root-Rechte, solange das Zielverzeichnis beschreibbar ist:

```bash
cd /tmp && yum download postfix
```

Meldet yum `No such command: download`, fehlt das Plugin. Mit Root-Rechten erreichen Sie dasselbe über den Installationsbefehl mit `--downloadonly`:

```bash
sudo yum install --downloadonly --downloaddir=/tmp postfix
```

Ganz ohne beides bleibt der Umweg über ein zweites System gleicher RHEL-Version: RPM dort herunterladen und per `scp` auf das Zielsystem kopieren.

## Binaries und Bibliotheken entpacken

`rpm2cpio` wandelt das RPM in einen cpio-Archivstrom um, aus dem `cpio` gezielt einzelne Pfade extrahiert. Neben den beiden Binaries brauchen Sie auch die Postfix-Bibliotheken, denn auf RHEL sind die Werkzeuge dynamisch gegen `libpostfix-*.so` gelinkt:

```bash
cd /tmp && rpm2cpio postfix-*.rpm | \
  cpio -idmv ./usr/sbin/smtp-source ./usr/sbin/smtp-sink \
  './usr/lib64/postfix/*'
```

Die Dateien liegen danach unterhalb von `/tmp/usr/`. Die Pfadangaben beginnen mit `./`, weil cpio die Pfade exakt so erwartet, wie sie im Archiv stehen.

## Problem 1: /tmp ist mit noexec gemountet

Der naheliegende Start direkt aus /tmp schlägt auf gehärteten Systemen fehl:

```text
bash: /tmp/usr/sbin/smtp-sink: Permission denied
[1]+  Exit 126
```

Exit-Code 126 trotz korrekt gesetztem Execute-Bit ist das typische Bild für ein Dateisystem mit der Mount-Option `noexec`. Der Kernel verweigert dann jede Programmausführung von diesem Dateisystem, unabhängig von den Dateirechten. Prüfen lässt sich das direkt:

```bash
mount | grep ' /tmp '
```

Die Lösung: Binaries und Bibliotheken in ein Verzeichnis kopieren, dessen Dateisystem Ausführung erlaubt, zum Beispiel das eigene Home:

```bash
mkdir -p ~/bin && \
  cp /tmp/usr/sbin/smtp-source /tmp/usr/sbin/smtp-sink \
     /tmp/usr/lib64/postfix/libpostfix-*.so ~/bin/ && \
  chmod +x ~/bin/smtp-source ~/bin/smtp-sink
```

Beachten Sie, dass `noexec` auch das Laden von Shared Libraries betrifft. Es genügt also nicht, nur die Binaries zu verschieben und die Bibliotheken in /tmp zu lassen.

## Problem 2: der Library-Pfad

Ohne weitere Angaben sucht der dynamische Linker die Postfix-Bibliotheken unter `/usr/lib64/postfix`, wo sie mangels Installation nicht liegen:

```text
smtp-sink: error while loading shared libraries: libpostfix-global.so:
cannot open shared object file: No such file or directory
```

`LD_LIBRARY_PATH` ergänzt den Suchpfad des Linkers um das eigene Verzeichnis. Jeder Aufruf bekommt die Variable vorangestellt:

```bash
LD_LIBRARY_PATH=~/bin ~/bin/smtp-source ...
```

Mit `ldd ~/bin/smtp-source` sehen Sie vorab, ob alle Abhängigkeiten auflösbar sind. Ausser den Postfix-Bibliotheken hängen die Werkzeuge nur an Standardbibliotheken des Systems.

## Funktionstest im Loopback

Ob alles läuft, prüfen Sie ohne eine einzige echte Mail: `smtp-sink` lauscht als Wegwerf-Empfänger auf einem hohen Port, `smtp-source` liefert dagegen ein. Der ganze Verkehr bleibt auf localhost.

```bash
LD_LIBRARY_PATH=~/bin ~/bin/smtp-sink -v 127.0.0.1:2525 100 &
```

```bash
LD_LIBRARY_PATH=~/bin ~/bin/smtp-source -s 2 -m 10 -l 5120 \
  -f test@example.com -t test@example.com 127.0.0.1:2525
```

Die Parameter im Überblick: `-s 2` öffnet zwei parallele Sessions, `-m 10` verteilt insgesamt zehn Nachrichten darauf, `-l 5120` setzt die Nachrichtengrösse auf 5 KB, `-f` und `-t` setzen Absender und Empfänger. Beim smtp-sink steht `100` für die maximale Zahl gleichzeitiger Verbindungen, `-v` protokolliert jeden Dialogschritt.

Bei Erfolg erzeugt `smtp-source` keine Ausgabe, während der smtp-sink für jede Nachricht den vollständigen SMTP-Dialog von `HELO` bis `QUIT` ausgibt. Danach den Hintergrundprozess beenden und die Reste aus /tmp entfernen:

```bash
kill %1
```

```bash
rm -rf /tmp/usr /tmp/postfix-*.rpm
```

## Hinweise für den echten Lasttest

Für belastbare Durchsatzmessungen gehört der Lastgenerator auf eine separate Maschine im gleichen Netzsegment, nicht auf das Testobjekt selbst. Läuft `smtp-source` auf dem geprüften Gateway, konkurrieren Generator und Mailsystem um CPU und I/O, und die Messung zeigt diese Konkurrenz statt der tatsächlichen Kapazität. Lokal auf dem Zielsystem taugt das entpackte Werkzeug vor allem für Funktionstests des Regelwerks und für erste Plausibilitätsprüfungen.

Sobald der Test gegen den echten Port 25 geht, sind es reale Mails, die das Regelwerk des Gateways durchlaufen und je nach Konfiguration zugestellt werden. Verwenden Sie deshalb Empfängeradressen, die kontrolliert enden: ein dediziertes Testpostfach, eine Regel, die die Testabsender verwirft, oder eine vom Provider dafür vorgesehene Verwerf-Domain. Produktive Adressen gehören nicht in einen Lasttest.

Das beschriebene Vorgehen eignet sich über die beiden SMTP-Werkzeuge hinaus für jedes Kommandozeilenprogramm, das ein Paket mitbringt, dessen Installation auf dem Zielsystem nicht in Frage kommt. Die Kombination aus `yum download`, `rpm2cpio` und einem ausführbaren Verzeichnis im Home ist auf jedem RPM-System gleich.

## Quellen

1.  [Postfix: smtp-source(1)](https://www.postfix.org/smtp-source.1.html): Manpage mit allen Parametern des Lastgenerators, inklusive Session- und Nachrichtensteuerung.

2.  [Postfix: smtp-sink(1)](https://www.postfix.org/smtp-sink.1.html): Manpage des Test-Empfängers, unter anderem mit Optionen für künstliche Verzögerungen und Fehlerantworten.

3.  [Red Hat: How to download a package without installing it](https://access.redhat.com/solutions/10154): dokumentiert `yum download` und die Alternative `--downloadonly`.

4.  [man7.org: mount(8)](https://man7.org/linux/man-pages/man8/mount.8.html): Beschreibung der Mount-Option `noexec` und ihrer Wirkung auf Programmausführung.
