---
title: "Claude Code auf dem eigenen VPS betreiben und den Server richtig absichern"
navTitle: "VPS für Claude"
description: "Warum ein eigener VPS eine solide Basis für Claude Code ist, und die vollständige Absicherungs-Checkliste dazu: eigener Benutzer, SSH nur mit passphrasengeschützten Ed25519-Schlüsseln, gehärtete sshd-Konfiguration, restriktive Firewall, Prüfung der Angriffsfläche und Zugriff vom iPhone."
date: "2026-07-21"
kategorie: "Linux"
timeToRead: "12 min to read"
themen:
  - "server-sicherheit"
slug: "claude-code-vps-debian-absichern"
url: "https://rafaelpfister.ch/blog/claude-code-vps-debian-absichern"
aiPrompt: |
  Du bist mein Linux-Server-Assistent. Hilf mir, einen frisch installierten Debian-13-VPS Schritt für Schritt abzusichern:
  1. System vollständig aktualisieren und einen eigenen Benutzer mit sudo-Rechten anlegen.
  2. SSH-Login auf passphrasengeschützte Ed25519-Schlüssel umstellen, Root- und Passwort-Login deaktivieren.
  3. SSH auf einen hohen Port legen (systemd-Socket-Aktivierung beachten) und die Konfiguration mit `sshd -t` prüfen, bevor ich neu lade.
  4. Firewall so setzen, dass eingehend nur der SSH-Port offen ist (Default DROP), ausgehend erlaubt.
  5. Mit `ss -lntup` und `systemctl --failed` prüfen, welche Dienste öffentlich lauschen und ob etwas fehlschlägt.
  Frag mich vor jedem Schritt, der mich aussperren könnte, und lass meine bestehende Sitzung offen, bis der neue Zugang getestet ist.
---

Claude Code lokal auf dem Arbeitsrechner zu starten ist bequem, bis der Laptop zugeklappt wird, das WLAN wechselt oder ein länger laufender Task mitten in der Nacht abbricht. Ein eigener VPS löst das: eine Sitzung, die durchläuft, erreichbar vom PC und vom iPhone, mit voller Kontrolle darüber, welche Daten dort liegen. Der Preis dafür ist, dass der Server ab der ersten Minute öffentlich erreichbar und damit Ziel automatisierter Scans ist. Der Betrieb lohnt sich trotzdem, und ein frisch installierter Debian-Server lässt sich so absichern, dass er diesem Dauerbeschuss standhält.

Die Schritte sind nicht Claude-spezifisch. Es ist die gleiche Härtungs-Checkliste, die für jeden öffentlich erreichbaren Linux-Server gilt.

## Warum ein eigener VPS?

Drei Gründe sprechen für den eigenen Server statt der lokalen Installation:

- **Persistenz.** In einer `tmux`-Sitzung läuft Claude weiter, auch wenn die SSH-Verbindung getrennt wird. Ein Task, der zehn Minuten oder eine Stunde braucht, läuft zu Ende, ohne dass der Laptop offen bleiben muss.
- **Erreichbarkeit.** Dieselbe Sitzung ist vom Desktop, vom Laptop und vom iPhone aus zugänglich. Man setzt am Schreibtisch einen Task an und schaut unterwegs nach dem Ergebnis.
- **Datenkontrolle.** Was auf dem Server liegt, entscheidet man selbst. Kein Sync-Dienst, keine versehentlich mitgesicherten Zugangsdaten, vorausgesetzt, man geht bei der Migration sorgfältig vor (siehe unten).

`tmux` ist dabei ein reines Verfügbarkeits- und Komfortmerkmal, keine Sicherheitsmassnahme. Die eigentliche Arbeit steckt in der Absicherung.

## Ausgangslage

Als Basis dient Debian 13 (Trixie), minimal installiert, ohne Desktop und ohne zusätzliche Netzwerkdienste. Der Provider stellt eine vorgelagerte Firewall bereit, die unabhängig vom Betriebssystem greift. Ziel ist ein Server, auf dem ausschliesslich SSH von aussen erreichbar ist, und auch das nur mit passphrasengeschützten Schlüsseln.

## 1. System aktualisieren

Direkt nach der Installation den kompletten Paketstand aktualisieren:

```bash
sudo apt update
sudo apt full-upgrade
```

`full-upgrade` löst im Gegensatz zu `upgrade` auch Abhängigkeiten auf, die neue oder entfernte Pakete erfordern. Bei einem frischen System ist das der richtige Weg, um wirklich alle verfügbaren Sicherheitsaktualisierungen einzuspielen. Nach Kernel-Updates einmal neu starten.

## 2. Eigener Benutzer statt root

Als root zu arbeiten ist unnötig riskant: Jeder Tippfehler wirkt systemweit, und der direkte root-Login ist das erste, was automatisierte Angriffe versuchen. Deshalb ein eigener Benutzer (hier `claude`) mit sudo-Rechten für die Fälle, in denen sie gebraucht werden:

```bash
sudo adduser claude
sudo usermod -aG sudo claude
```

Ab jetzt läuft alle Administration über `claude` und `sudo`, nicht mehr über den direkten root-Zugang.

## 3. Ed25519-Schlüssel mit Passphrase, einer pro Gerät

Die Anmeldung soll ausschliesslich über SSH-Schlüssel laufen, nicht über Passwörter. Ed25519 ist der aktuelle Standard: kurz, schnell, kryptografisch solide. Entscheidend ist, dass der Schlüssel auf dem Client erzeugt wird, also auf dem PC, nicht auf dem Server, und mit einer Passphrase geschützt ist. Die Passphrase ist die zweite Verteidigungslinie, falls der private Schlüssel je in falsche Hände gerät.

Auf dem PC:

```bash
ssh-keygen -t ed25519 -C "pc-thinkpad"
```

Der Kommentar (`-C`) benennt das Gerät. Das zahlt sich später aus: Für jedes Gerät wird ein eigener Schlüssel erzeugt: einer für den PC, ein separater fürs iPhone. Geht ein Gerät verloren, entfernt man gezielt dessen öffentlichen Schlüssel aus `~/.ssh/authorized_keys`, ohne alle anderen Zugänge neu ausrollen zu müssen.

Nur der öffentliche Schlüssel gehört auf den Server. Der private Schlüssel verlässt das Gerät nie. In `authorized_keys` stehen am Ende ausschliesslich öffentliche Schlüssel, jeweils mit ihrem Gerätekommentar:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...pc  pc-thinkpad
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...ios iphone-15
```

Den öffentlichen PC-Schlüssel initial übertragen. Solange die Passwortanmeldung noch aktiv ist, geht das am einfachsten mit:

```bash
ssh-copy-id claude@SERVER
```

Danach testen, dass der Login mit Schlüssel funktioniert, bevor im nächsten Schritt die Passwortanmeldung abgeschaltet wird. Die Dateirechte müssen stimmen, sonst ignoriert sshd die Datei: `~/.ssh` auf `700`, `authorized_keys` auf `600`.

## 4. SSH härten: kein root, kein Passwort

Die Server-Konfiguration liegt in `/etc/ssh/sshd_config` und (bei Debian 13) in Drop-in-Dateien unter `/etc/ssh/sshd_config.d/`. Änderungen gehören in eine eigene Drop-in-Datei; so bleibt die Hauptdatei unangetastet und Paket-Updates überschreiben nichts. Datei `/etc/ssh/sshd_config.d/99-haertung.conf` anlegen:

```text
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
```

Das deaktiviert den direkten root-Login und die Passwortanmeldung. Ab jetzt kommt nur noch herein, wer einen passenden privaten Schlüssel besitzt. Vor dem Neuladen die Konfiguration syntaktisch prüfen:

```bash
sudo sshd -t
```

Meldet `sshd -t` nichts, ist die Datei valide. Erst dann neu laden:

```bash
sudo systemctl reload ssh
```

**Wichtig:** Die bestehende SSH-Sitzung offen lassen und den neuen Zugang in einem zweiten Terminal testen. Erst wenn der Login mit Schlüssel dort nachweislich klappt, darf die alte Sitzung geschlossen werden. Diese Vorsichtsmassnahme reduziert das Aussperrrisiko auf praktisch null. Ein Fehler in der Konfiguration kostet sonst den kompletten Zugang.

## 5. SSH auf einen unüblichen Port legen

Der Standardport 22 wird rund um die Uhr von Bots durchprobiert. Ein Wechsel auf einen hohen, frei gewählten Port (im Beispiel `61417`) lässt den Grossteil dieses automatisierten Rauschens ins Leere laufen. Das ist ausdrücklich kein Sicherheitsgewinn im eigentlichen Sinn: Ein Portwechsel ersetzt keine starke Authentifizierung, er reduziert nur die Logmenge und die Scan-Last. Die Schlüsselpflicht aus Schritt 4 bleibt die eigentliche Absicherung.

Welcher Port das wird, ist nicht beliebig. Die IANA unterscheidet drei Zonen: **0–1023 (well-known ports)** sind Standarddiensten vorbehalten (SSH selbst auf 22, HTTP auf 80, HTTPS auf 443), erfordern root zum Binden und haben auf einem selbst gewählten SSH-Port nichts verloren, genau diese Ports erwarten Scanner ebenso wie später installierte Standarddienste. **1024–49151 (registrierte Ports)** sind auf Anfrage einzelnen Anwendungen zugeteilt, etwa 3306 (MySQL), 5432 (PostgreSQL), 6379 (Redis) oder 8080/8443 als verbreitete HTTP-Alternativen; ein zufällig gewählter Port aus diesem Bereich kollidiert später leicht mit Software, die genau ihren registrierten Port erwartet. **49152–65535 (dynamische/private Ports)** sind laut IANA keinem Dienst zugewiesen und für temporäre, private Zwecke gedacht, der richtige Bereich für einen dauerhaften, selbst gewählten Port.

Ein Vorbehalt bleibt: Viele Linux-Systeme, auch Debian, nutzen einen Teil desselben Bereichs als Quellport für eigene ausgehende Verbindungen (`net.ipv4.ip_local_port_range`, standardmässig um 32768–60999). Ein dauerhaft lauschender Dienst kollidiert dadurch nicht wirklich, der Kernel vergibt keinen bereits gebundenen Port, aber ein Port oberhalb von 60999 umgeht auch diese theoretische Unschärfe. Das Beispiel in diesem Artikel (`61417`) liegt deshalb bewusst dort. Vor der Umstellung zusätzlich mit `ss -lntup` (siehe Schritt 7) prüfen, ob der gewählte Port auf dem eigenen Server nicht schon belegt ist.

Bei Debian 13 gibt es hier einen Stolperstein: SSH kann über systemd-Socket-Aktivierung gestartet werden. Ist das der Fall, wird die `Port`-Angabe in der `sshd_config` schlicht ignoriert; der Port muss dann am Socket gesetzt werden. Zuerst prüfen, welcher Fall vorliegt:

```bash
systemctl is-enabled ssh.socket
```

Antwortet der Befehl mit `enabled`, läuft SSH über den Socket. Dann den Port dort ändern:

```bash
sudo systemctl edit ssh.socket
```

Im Editor folgende Zeilen eintragen. Die erste, leere `ListenStream=`-Zeile löscht den voreingestellten Port 22, die zweite setzt den neuen:

```text
[Socket]
ListenStream=
ListenStream=61417
```

Anschliessend übernehmen:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
```

Ist die Socket-Aktivierung nicht aktiv (`disabled`), gehört stattdessen `Port 61417` in die Drop-in-Datei aus Schritt 4, gefolgt von `sudo sshd -t` und `sudo systemctl restart ssh`.

Auch hier gilt: Erst den neuen Port in der Firewall öffnen (nächster Schritt), dann verbinden und testen, und die alte Sitzung offen lassen, bis der Zugang über den neuen Port bestätigt ist.

## 6. Firewall: standardmässig dicht

Die vorgelagerte Provider-Firewall ist die wirksamste Grenze, weil sie Pakete abfängt, bevor sie das Betriebssystem überhaupt erreichen. Zwei Grundregeln:

- **Eingehende Standardaktion auf DROP.** Alles, was nicht ausdrücklich erlaubt ist, wird verworfen, kommentarlos, ohne Rückmeldung an den Absender.
- **Eine einzige Ausnahme:** eingehendes TCP auf den Ziel-Port `61417`. Mehr muss von aussen nicht erreichbar sein.

Der ausgehende Verkehr bleibt erlaubt. Das ist bewusst so: Der Server muss Pakete laden, die Zeit synchronisieren und für Claude Code die API erreichen. Eine restriktive Ausgangsfilterung bringt bei einem Einzelserver wenig zusätzlichen Schutz, macht den Betrieb aber spürbar umständlicher.

Wer zusätzliche Tiefenstaffelung möchte, kann dieselben Regeln hostseitig mit `nftables` oder `ufw` doppeln. Für den beschriebenen Aufbau genügt die Provider-Firewall.

## 7. Angriffsfläche prüfen

Nach der Härtung kontrollieren, was der Server tatsächlich nach aussen anbietet. Zwei Befehle reichen. Erstens: Welche Dienste lauschen auf welchen Adressen?

```bash
sudo ss -lntup
```

`ss` listet alle lauschenden TCP- und UDP-Sockets samt zugehörigem Prozess (`sudo` ist nötig, um die Prozessnamen zu sehen). Entscheidend ist die Adressspalte: Ein Dienst auf `0.0.0.0` oder `[::]` ist von aussen erreichbar, einer auf `127.0.0.1` oder `[::1]` nur lokal. Im abgesicherten Zustand sollte ausschliesslich SSH öffentlich erscheinen. Dienste wie `chronyd` (Zeitsynchronisation) dürfen auftauchen, aber nur an lokale Adressen gebunden. Hört `chronyd` ausschliesslich auf `127.0.0.1` und `::1`, ist er nicht von aussen ansprechbar und damit unkritisch.

Zweitens: Gibt es fehlgeschlagene Systemdienste, die auf ein Konfigurationsproblem hindeuten?

```bash
systemctl --failed
```

Die Antwort sollte `0 loaded units listed` lauten, kein einziger fehlgeschlagener Dienst. Fehlerhafte Units sind nicht nur ein Betriebs-, sondern potenziell auch ein Sicherheitsproblem, wenn dahinter ein halb gestarteter, falsch konfigurierter Netzwerkdienst steckt.

## 8. Claude Code installieren und betreiben

Claude Code braucht eine aktuelle Node.js-Laufzeitumgebung. Nach deren Installation die CLI gemäss der offiziellen Anleitung einrichten und auf dem Server frisch authentifizieren, nicht die lokalen Zugangsdaten hochladen (dazu gleich mehr).

Für den dauerhaften Betrieb `tmux`:

```bash
tmux new -s claude
```

Innerhalb der Sitzung Claude starten. Mit `Ctrl-b`, dann `d` löst man sich von der Sitzung, ohne sie zu beenden; Claude läuft weiter. Zurück geht es mit:

```bash
tmux attach -t claude
```

So überlebt eine laufende Aufgabe getrennte Verbindungen, Gerätewechsel und die Nachtruhe des Laptops.

## 9. Datenhygiene bei der Migration

Der heikelste Teil beim Umzug auf den Server ist nicht die Technik, sondern die Frage, was man mitnimmt. Drei Regeln:

- **Keine privaten Schlüssel auf den Server.** In `authorized_keys` liegen ausschliesslich öffentliche Schlüssel. Private Schlüssel bleiben auf den Endgeräten.
- **Keine Zugangsdaten pauschal kopieren.** Sensible lokale Dateien wie eine `.credentials.json` gehören nicht ungeprüft auf den VPS. Stattdessen authentifiziert man sich frisch auf dem Server.
- **Konfiguration erst in einen Migrationsordner.** Bestehende Claude-Memories und -Konfiguration nicht direkt in die aktiven Konfigurationspfade schreiben, sondern zunächst in einen separaten Migrationsordner übertragen und dort prüfen, was wirklich übernommen werden soll. Was man nicht mehr braucht, etwa alte MCP-Einträge oder verwaiste Einstellungen, bleibt bewusst zurück, statt unbesehen mitzuwandern.

## 10. Web-Vorschauen über einen SSH-Tunnel

Für Web-Vorschauen, etwa einen lokalen Entwicklungsserver, den Claude startet, ist die Versuchung gross, einfach einen weiteren Port zu öffnen. Das sollte man nicht tun. Jeder zusätzliche offene Port ist zusätzliche Angriffsfläche. Stattdessen läuft die Vorschau durch einen verschlüsselten SSH-Port-Tunnel: Der Dienst hört nur lokal auf dem Server, und SSH reicht ihn auf den Client durch.

Vom PC aus einen lokal auf Port 4321 laufenden Dienst erreichbar machen:

```bash
ssh -p 61417 -L 4321:localhost:4321 claude@SERVER
```

Danach öffnet man im lokalen Browser `http://localhost:4321`. Der Verkehr läuft vollständig durch die bestehende, authentifizierte SSH-Verbindung, ohne dass in der Firewall auch nur ein einziger weiterer Port geöffnet werden muss.

## Zugriff vom iPhone

Der Zugriff von unterwegs funktioniert mit demselben Sicherheitsmodell wie vom PC. Man braucht nur einen SSH-Client mit Schlüsselverwaltung. Verbreitet sind **Termius**, **Blink Shell** und **Secure ShellFish**; alle können Ed25519-Schlüssel erzeugen und im iOS-Schlüsselbund ablegen, teils abgesichert per Face ID.

Das Vorgehen entspricht Schritt 3, nur auf dem iPhone:

1. Im SSH-Client einen eigenen Ed25519-Schlüssel fürs iPhone erzeugen, nicht den PC-Schlüssel kopieren. Der private Schlüssel bleibt im Schlüsselbund des Geräts.
2. Den öffentlichen Schlüssel des iPhones als zusätzliche Zeile in `~/.ssh/authorized_keys` auf dem Server eintragen, mit sprechendem Kommentar (`iphone-15`).
3. Im Client die Verbindung anlegen: Server-Adresse, Benutzer `claude`, Port `61417`, der iPhone-Schlüssel als Authentifizierung.

Genau deshalb lohnt sich der separate Schlüssel pro Gerät: Geht das iPhone verloren, löscht man auf dem Server die eine `iphone-15`-Zeile aus `authorized_keys`, und das Gerät ist ausgesperrt, während PC-Zugang und alle anderen Schlüssel unberührt weiterlaufen.

Nach dem Verbinden holt man die laufende Claude-Sitzung mit `tmux attach -t claude` zurück und arbeitet dort weiter, wo man am Schreibtisch aufgehört hat. Auch der Port-Tunnel aus Schritt 10 funktioniert von iOS aus; Termius und Secure ShellFish beherrschen Port-Weiterleitung.

## Checkliste

Zusammengefasst der komplette Ablauf:

1. Debian 13 installiert, mit `apt full-upgrade` vollständig aktualisiert.
2. Eigener Benutzer `claude` mit sudo-Rechten; direkter root-Login nicht mehr genutzt.
3. Passphrasengeschützte Ed25519-Schlüssel, pro Gerät einer, nur öffentliche Schlüssel in `authorized_keys`.
4. sshd gehärtet: `PermitRootLogin no`, `PasswordAuthentication no`; vor dem Neuladen mit `sshd -t` geprüft, bestehende Sitzung bis zum Test offen gelassen.
5. SSH auf Port 61417, bei Socket-Aktivierung am `ssh.socket` gesetzt, sonst in der sshd-Konfiguration.
6. Provider-Firewall: eingehend Default DROP, einzige Ausnahme TCP 61417; ausgehend erlaubt.
7. Angriffsfläche geprüft mit `ss -lntup` (nur SSH öffentlich, `chronyd` lokal) und `systemctl --failed` (keine Fehler).
8. Claude Code auf dem Server frisch authentifiziert, Betrieb in einer `tmux`-Sitzung.
9. Datenhygiene: keine privaten Schlüssel und keine Zugangsdaten auf den Server, Konfiguration erst über einen Migrationsordner geprüft.
10. Keine zusätzlichen Ports; Web-Vorschauen laufen durch einen SSH-Tunnel.

Der Aufwand ist einmalig, das Ergebnis dauerhaft: ein Server, der von aussen nur eine einzige, schlüsselgeschützte Tür zeigt, und eine Claude-Sitzung, die immer erreichbar ist, egal ob vom Schreibtisch oder unterwegs.

## Quellen

1.  [OpenSSH Manual – sshd_config(5)](https://man.openbsd.org/sshd_config) — Referenz aller sshd-Direktiven, darunter `PermitRootLogin`, `PasswordAuthentication` und `PubkeyAuthentication`.

2.  [Debian Wiki – SSH](https://wiki.debian.org/SSH) — Debian-spezifische Hinweise zur SSH-Konfiguration inklusive der Drop-in-Dateien unter `/etc/ssh/sshd_config.d/`.

3.  [systemd.socket(5) – freedesktop.org](https://www.freedesktop.org/software/systemd/man/latest/systemd.socket.html) — Funktionsweise der Socket-Aktivierung und der `ListenStream=`-Direktive, relevant für den SSH-Portwechsel unter Debian 13.

4.  [ss(8) – iproute2 Manpage](https://man7.org/linux/man-pages/man8/ss.8.html) — Optionen von `ss`, um lauschende Sockets samt Prozess und Bindungsadresse aufzulisten.

5.  [Claude Code – Offizielle Dokumentation](https://docs.claude.com/en/docs/claude-code/overview) — Installation, Authentifizierung und Betrieb von Claude Code.
