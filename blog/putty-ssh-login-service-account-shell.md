---
title: "Passwortlos auf Linux-Server: SSH-Key-Login mit PuTTY, Pageant und Co. einrichten"
navTitle: "PuTTY passwortlos"
description: "Wer als Admin täglich auf Linux-Server zugreift, tippt mit Passwort-Login jedes Mal Benutzername und Passwort. Ein SSH-Schlüsselpaar reduziert das auf einen Doppelklick: Schlüssel mit PuTTYgen erzeugen, Public Key auf dem Server hinterlegen, Pageant laden. Derselbe Schlüssel funktioniert in WinSCP, MobaXterm und OpenSSH, und wer will, landet direkt in der Shell eines Service-Accounts."
date: "2026-08-28"
kategorie: "Linux und SSH"
timeToRead: "9 Min. Lesezeit"
themen:
  - "totemomail"
  - "windows-client"
produkte:
  - "totemomail"
  - "uebergreifend"
protokolle:
  - "ssh"
  - "haertung"
slug: "putty-ssh-login-service-account-shell"
translationId: "article-9f94fa6eb8b95bcf"
url: "https://rafaelpfister.ch/blog/putty-ssh-login-service-account-shell"
aiPrompt: |
  Du bist mein Linux- und SSH-Assistent. Hilf mir Schritt für Schritt, einen passwortlosen SSH-Login von Windows auf meine Linux-Server einzurichten: 1. Ein Ed25519-Schlüsselpaar mit PuTTYgen erzeugen und den Public Key in authorized_keys eintragen. 2. Die PuTTY-Session mit Schlüsseldatei und Auto-login username vervollständigen und Pageant mit Autostart einrichten. 3. Optional ein Remote command hinterlegen, das mich direkt in die Shell eines Service-Accounts bringt, samt minimaler NOPASSWD-Regel unter /etc/sudoers.d. Weise mich auf typische Fehler hin: Key im falschen Home-Verzeichnis, mehrzeiliger Public Key, falsche Berechtigungen, sudoers-Befehl stimmt nicht exakt mit dem Remote command überein.
---
# Passwortlos auf Linux-Server: SSH-Key-Login mit PuTTY, Pageant und Co. einrichten

Wer als Admin täglich auf Linux-Server zugreift, wiederholt mit Passwort-Login bei jeder Verbindung dieselben Eingaben: Benutzername, Passwort, bei Service-Accounts noch ein sudo-Befehl hinterher. Mit einem SSH-Schlüsselpaar entfällt das alles. Nach der Einrichtung öffnet ein Doppelklick auf die gespeicherte Session eine fertige Shell, und derselbe Schlüssel funktioniert in PuTTY, WinSCP, MobaXterm und im OpenSSH-Client von Windows.

Diese Anleitung richtet den passwortlosen Login von Windows aus komplett ein: Schlüsselpaar erzeugen, Public Key auf dem Server hinterlegen, PuTTY-Session vervollständigen, Pageant als Schlüssel-Agent einrichten. Dazu kommen die Fehlersuche für den häufigsten Befund (der Server fragt trotz Schlüssel weiter nach dem Passwort) und als Erweiterung der Direkteinstieg in die Shell eines Service-Accounts wie `totemo`.

## Warum Schlüssel statt Passwort

Der Komfortgewinn ist der sichtbarste Effekt, aber nicht der wichtigste. Ein Ed25519-Schlüssel ist gegen Brute-Force-Angriffe praktisch immun, während ein Passwort nur so stark ist wie seine Länge und die Disziplin, es nirgendwo wiederzuverwenden. Auf Servern, deren Benutzer vollständig auf Schlüssel umgestellt sind, lässt sich die Passwort-Authentifizierung in der sshd-Konfiguration ganz abschalten (`PasswordAuthentication no`), womit automatisierte Login-Versuche aus dem Internet ins Leere laufen. Schalten Sie die Passwort-Authentifizierung erst ab, wenn der Schlüssel-Login nachweislich funktioniert und ein zweiter Zugangsweg (Konsole, zweiter Schlüssel) besteht.

Das Prinzip: Der private Schlüssel bleibt auf Ihrem Windows-Rechner, der öffentliche wird auf dem Server hinterlegt. Der Server prüft beim Verbindungsaufbau, ob die Gegenseite den passenden privaten Schlüssel besitzt, ohne dass dieser den Rechner je verlässt.

## Schritt 1: Schlüsselpaar mit PuTTYgen erzeugen

1. **PuTTYgen** starten (Teil des PuTTY-Pakets), als Typ **Ed25519** wählen, **Generate** klicken und die Maus über die Fläche bewegen.
2. Eine **Passphrase** in beide Felder eintragen und den privaten Schlüssel mit **Save private key** als `.ppk`-Datei speichern.
3. Das Textfeld oben (die Zeile beginnend mit `ssh-ed25519 AAAA...`) vollständig kopieren. Das ist der öffentliche Schlüssel im Format, das der Server erwartet.

Speichern Sie den privaten Schlüssel mit Passphrase. Ohne Passphrase ist jede Kopie der Datei ein fertiger Serverzugang, mit Passphrase ist die Datei allein wertlos. Der Komfort-Nachteil entfällt durch Pageant (Schritt 4) fast vollständig. Ein Schlüssel ohne Passphrase ist nur für unbeaufsichtigte Automatisierung vertretbar, nicht für den interaktiven Login.

## Schritt 2: Public Key auf dem Server hinterlegen

Auf dem Server, eingeloggt als der Benutzer, mit dem Sie sich künftig verbinden:

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo 'ssh-ed25519 AAAA... kommentar' >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
chmod go-w ~
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Befehl / Option | Wirkung |
|---|---|
| `mkdir -p ~/.ssh` | Legt das SSH-Verzeichnis an; `-p` unterdrückt den Fehler, falls es schon existiert |
| `chmod 700 ~/.ssh` | Nur der Besitzer darf das Verzeichnis lesen, schreiben und betreten |
| `echo '...' >> ~/.ssh/authorized_keys` | Hängt den Public Key als neue Zeile an die Datei an (`>>` statt `>`, sonst überschreiben Sie bestehende Schlüssel) |
| `chmod 600 ~/.ssh/authorized_keys` | Nur der Besitzer darf die Schlüsseldatei lesen und schreiben |
| `chmod go-w ~` | Entzieht Gruppe und Anderen das Schreibrecht am Home-Verzeichnis |

</details>

Die letzte Zeile wirkt unscheinbar, gehört aber dazu: Ein gruppen- oder weltbeschreibbares Home-Verzeichnis lässt den SSH-Daemon den Schlüssel stillschweigend ignorieren, und der Server fragt weiter nach dem Passwort, ohne einen Grund zu nennen.

## Schritt 3: PuTTY-Session vervollständigen

1. PuTTY öffnen, die gespeicherte Session markieren und mit **Load** laden.
2. **Connection → SSH → Auth → Credentials**: bei **Private key file for authentication** die `.ppk`-Datei auswählen.
3. **Connection → Data**: bei **Auto-login username** den Benutzernamen eintragen, sonst fragt PuTTY weiterhin bei jedem Verbinden danach.
4. Zurück zur Kategorie **Session**, den Session-Namen wieder markieren und **Save** klicken.

Der häufigste Bedienfehler dabei: **Load** vor der Änderung oder **Save** danach auslassen. Ohne Load bearbeiten Sie nur die Default-Einstellungen, ohne Save ist die Änderung beim nächsten Start von PuTTY verloren.

## Schritt 4: Pageant, Passphrase einmal pro Windows-Sitzung

Pageant ist der Schlüssel-Agent von PuTTY. Er hält den entschlüsselten privaten Schlüssel im Speicher, sodass die Passphrase pro Windows-Sitzung nur einmal fällig wird:

1. Pageant starten (Icon erscheint im System-Tray).
2. Rechtsklick auf das Icon, **Add Key**, die `.ppk`-Datei wählen, Passphrase eingeben.
3. Ab dann laufen alle Verbindungen ohne Abfrage, bis zum nächsten Neustart.

Damit Pageant automatisch mitstartet, legen Sie eine Verknüpfung in den Autostart-Ordner (`Win+R`, dann `shell:startup`) und übergeben den Schlüssel als Argument:

```text
"C:\Program Files\PuTTY\pageant.exe" "C:\Pfad\zum\schluessel.ppk"
```

Windows fragt dann einmal nach der Anmeldung nach der Passphrase, der Rest des Arbeitstags läuft ohne.

## Wenn der Server weiter nach dem Passwort fragt

Die Fehlersuche beginnt im **Event Log** von PuTTY (Rechtsklick auf die Titelleiste der Terminal-Sitzung). Dort steht, ob der Schlüssel überhaupt angeboten wurde:

| Befund im Event Log | Ursache und Abhilfe |
|---|---|
| Kein Eintrag zu einem Public Key | Die `.ppk` ist nicht in der gespeicherten Session hinterlegt, oder die falsche Session wurde bearbeitet. Session laden, Key setzen, speichern. |
| `Server refused our key` | Der Server findet oder akzeptiert den Schlüssel nicht: falsches Home, falsches Format oder falsche Berechtigungen (siehe unten). |
| `Access granted`, danach Passwortabfrage | Der Key-Login hat funktioniert; die Abfrage kommt von einem nachgelagerten Programm, typischerweise sudo. Dazu die Erweiterung unten. |

Die drei häufigsten Ursachen für `Server refused our key`:

- **Schlüssel im falschen Home.** Der Public Key muss in der `authorized_keys` des Benutzers stehen, mit dem die Verbindung aufgebaut wird. Wer beim Eintragen bereits mit `sudo -u` oder `su` zu einem anderen Konto gewechselt hat, legt die Datei in dessen Home ab statt im eigenen. `whoami` vor dem Eintragen zeigt, in wessen Home der Schlüssel landet.
- **Falsches Format.** Der Public Key muss als eine einzige Zeile in `authorized_keys` stehen, im Format aus dem Textfeld oben in PuTTYgen. Die Datei aus **Save public key** hat ein anderes, mehrzeiliges Format (`---- BEGIN SSH2 PUBLIC KEY ----`) und funktioniert in `authorized_keys` nicht.
- **Berechtigungen.** `~/.ssh` auf `700`, `authorized_keys` auf `600`, Home-Verzeichnis nicht gruppen- oder weltbeschreibbar.

Bleibt der Befund unklar, hilft serverseitig ein Blick in `/var/log/secure` beziehungsweise `journalctl -u sshd`, dort begründet der SSH-Daemon die Ablehnung.

## Derselbe Schlüssel in anderen Tools

Die Einrichtung auf dem Server ist werkzeugunabhängig, der Schlüssel lässt sich überall weiterverwenden:

| Tool | Einrichtung |
|---|---|
| **WinSCP** | Nutzt `.ppk`-Dateien direkt (Anmeldung → Erweitert → SSH → Authentifizierung) und bedient sich automatisch bei Pageant, wenn der Schlüssel dort geladen ist |
| **MobaXterm** | `.ppk` unter Session settings → SSH → Advanced → Use private key; versteht auch das OpenSSH-Format |
| **FileZilla** | `.ppk` unter Einstellungen → SFTP eintragen oder Pageant laufen lassen |
| **OpenSSH (Windows-Terminal, `ssh`)** | Braucht das OpenSSH-Format: in PuTTYgen über **Conversions → Export OpenSSH key** exportieren und in `~/.ssh/` ablegen |

Für den OpenSSH-Client gehört zum Komfort-Login ein Eintrag in `~/.ssh/config` auf dem Windows-Rechner:

```text
Host mailgw
    HostName server.example.com
    User mmuster
    IdentityFile ~/.ssh/id_ed25519
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `Host mailgw` | Frei wählbarer Alias; `ssh mailgw` genügt danach als Aufruf |
| `HostName` | Tatsächlicher Servername oder die IP-Adresse |
| `User` | Benutzername, entspricht dem Auto-login username in PuTTY |
| `IdentityFile` | Pfad zum privaten Schlüssel im OpenSSH-Format |

</details>

## Erweiterung: direkt in der Service-Account-Shell landen

Viele Linux-Server im Applikations- und Mail-Umfeld werden nicht mit dem persönlichen Konto administriert, sondern über einen Service-Account: Totemomail läuft unter `totemo`, andere Gateways und Applikationen haben ihre eigenen Funktionskonten. Nach dem Login folgt darum routinemässig derselbe Befehl:

```bash
sudo -u totemo /bin/bash -l
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `sudo` | Führt den folgenden Befehl mit anderen Rechten aus und protokolliert den Aufruf |
| `-u totemo` | Zielbenutzer ist `totemo` statt des Standards `root` |
| `/bin/bash` | Der auszuführende Befehl: eine neue Bash-Shell |
| `-l` | Startet die Bash als Login-Shell; sie liest das Profil des Zielbenutzers ein (`.bash_profile`, Umgebungsvariablen, Pfade) |

</details>

Das `-l` ist bei Service-Accounts entscheidend: Ohne Login-Shell fehlen die Umgebungsvariablen aus dem Profil des Funktionskontos, etwa Pfade zu Applikationsverzeichnissen oder Java-Installationen, und applikationseigene Kommandos schlagen mit irreführenden Fehlermeldungen fehl.

Der direkte SSH-Login als Service-Account wäre noch kürzer, ist aber aus gutem Grund meist gar nicht möglich: Funktionskonten haben oft kein Passwort oder eine gesperrte Login-Shell, und ein von mehreren Personen geteilter Direktzugang würde den personalisierten Audit-Trail beseitigen. Beim Weg über sudo bleibt nachvollziehbar, welche Person wann in die Service-Account-Shell gewechselt hat. Die folgende Automatisierung ändert daran nichts, sie spart nur die Tipparbeit.

### Remote command in der PuTTY-Session

PuTTY kann pro gespeicherter Session einen Befehl hinterlegen, der nach dem Login anstelle der normalen Shell ausgeführt wird:

1. Session mit **Load** laden.
2. Links im Baum zu **Connection → SSH** navigieren (der Hauptknoten, nicht ein Unterpunkt).
3. Im Feld **Remote command** eintragen: `sudo -u totemo /bin/bash -l`
4. Unter **Session** speichern.

Drei Eigenheiten des Remote command sollten Sie kennen:

- Ein `exit` in der Service-Account-Shell beendet die Verbindung komplett, statt in Ihre persönliche Shell zurückzukehren. Der Befehl ersetzt die Login-Shell, er verschachtelt sie nicht.
- Wenn Sie gelegentlich mit dem persönlichen Konto auf dem Server arbeiten, speichern Sie eine zweite Session ohne Remote command (Session laden, Feld leeren, unter neuem Namen sichern).
- Dateitransfer-Werkzeuge wie WinSCP oder `pscp` sind nicht betroffen. Sie bauen eigene Verbindungen auf und ignorieren das Remote command der PuTTY-Session.

Falls die Verbindung sich nach dem Aufbau sofort wieder schliesst oder sudo ein fehlendes Terminal meldet: Unter **Connection → SSH → TTY** prüfen, dass **Don't allocate a pseudo-terminal** nicht angehakt ist. Standardmässig ist es das nicht. Wichtig für Schritt 2 oben: Mit aktivem Remote command sind Sie beim Eintragen des Public Key bereits der Service-Account; der Schlüssel gehört aber ins Home des persönlichen Benutzers.

Im OpenSSH-Client leisten zwei Zeilen im `Host`-Eintrag dasselbe: `RequestTTY yes` und `RemoteCommand sudo -u totemo /bin/bash -l`.

### sudo ohne Passwortabfrage

Nach Schlüssel und Remote command bleibt eine einzige Eingabe übrig: die Passwortabfrage von sudo. Sie verschwindet nur über die sudoers-Konfiguration auf dem Server, und dafür braucht es Root-Rechte. Auf einem verwalteten Firmenserver ist das eine Anfrage an den Server-Admin, keine Einstellung in PuTTY.

Die Regel gehört in eine eigene Datei unter `/etc/sudoers.d/` und wird mit `visudo` bearbeitet:

```bash
visudo -f /etc/sudoers.d/totemo-shell
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `visudo` | Öffnet die sudoers-Datei im Editor und prüft die Syntax vor dem Speichern |
| `-f /etc/sudoers.d/totemo-shell` | Bearbeitet die angegebene Datei statt der zentralen `/etc/sudoers` |

</details>

Inhalt der Datei, mit dem persönlichen Benutzernamen (hier `mmuster` als Beispiel):

```text
mmuster ALL=(totemo) NOPASSWD: /bin/bash -l
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Bestandteil | Wirkung |
|---|---|
| `mmuster` | Die Regel gilt nur für diesen Benutzer |
| `ALL=` | Auf allen Hosts (relevant bei zentral verteilten sudoers-Dateien) |
| `(totemo)` | Nur für Befehle als Zielbenutzer `totemo`, nicht als root |
| `NOPASSWD:` | Keine Passwortabfrage für die folgenden Befehle |
| `/bin/bash -l` | Genau dieser Befehl mit genau diesem Argument, nichts anderes |

</details>

Zwei Punkte entscheiden darüber, ob die Regel greift und ob sie vertretbar ist:

- **Exakte Übereinstimmung.** Der Befehl in der Regel muss dem Remote command entsprechen, inklusive Argument. Steht in PuTTY `sudo -u totemo /bin/bash -l`, muss die Regel `/bin/bash -l` erlauben. Eine Regel für `/bin/bash` ohne `-l` deckt den Aufruf mit `-l` nicht ab, und sudo fragt weiter nach dem Passwort.
- **Minimaler Umfang.** Die Regel erlaubt einen einzigen Befehl als einen einzigen Zielbenutzer. Sie gibt weder Root-Rechte noch beliebige Befehle frei. In dieser Form ist sie auch auf verwalteten Servern eine übliche und begründbare Anfrage. Die sudo-Protokollierung bleibt vollständig erhalten, jeder Wechsel in die Service-Account-Shell steht weiterhin im Log.

`visudo` ist dabei nicht optional: Es prüft die Syntax vor dem Speichern. Ein Tippfehler, der direkt in die Datei geschrieben wird, kann sudo für alle Benutzer des Systems unbrauchbar machen. Aus demselben Grund ist eine eigene Datei unter `/etc/sudoers.d/` der Bearbeitung der zentralen `/etc/sudoers` vorzuziehen; sie übersteht Paket-Updates und lässt sich gefahrlos wieder entfernen.

## Das Ergebnis

Nach der Einrichtung sieht der Login so aus: Doppelklick auf die PuTTY-Session, und die Shell steht bereit. Kein Benutzername, kein Passwort, mit der Erweiterung auch keine sudo-Abfrage. Die Sicherheitslage hat sich dabei nicht verschlechtert, in einem Punkt sogar verbessert:

| Aspekt | Vorher | Nachher |
|---|---|---|
| Authentifizierung | Passwort bei jedem Login | Ed25519-Schlüssel mit Passphrase, gehalten in Pageant |
| Login-Identität | Persönliches Konto | Unverändert persönliches Konto |
| sudo-Protokollierung | Jeder Wechsel im Log | Unverändert jeder Wechsel im Log |
| NOPASSWD-Umfang | Keiner | Ein Befehl, ein Zielbenutzer, kein root |

## Quellen

1.  [PuTTY User Manual, Kapitel 4: Configuring PuTTY](https://the.earth.li/~sgtatham/putty/0.83/htmldoc/Chapter4.html): Dokumentation der Session-Einstellungen, darunter Remote command (SSH panel), Auto-login username (Data panel) und Pseudo-Terminal (TTY panel).

2.  [PuTTY User Manual, Kapitel 8: Using public keys for SSH authentication](https://the.earth.li/~sgtatham/putty/0.83/htmldoc/Chapter8.html): PuTTYgen, Schlüsseltypen, Passphrasen, OpenSSH-Export und das Eintragen des Public Key auf dem Server.

3.  [PuTTY User Manual, Kapitel 9: Using Pageant for authentication](https://the.earth.li/~sgtatham/putty/0.83/htmldoc/Chapter9.html): Funktionsweise des Agents, Laden von Schlüsseln und Start mit Schlüssel als Kommandozeilen-Argument.

4.  [ssh_config(5) Manual](https://man.openbsd.org/ssh_config.5): Client-Konfiguration des OpenSSH-Clients, darunter Host-Aliase, IdentityFile, RequestTTY und RemoteCommand.

5.  [sudoers(5) Manual](https://www.sudo.ws/docs/man/sudoers.man/): Syntax der sudoers-Regeln, Runas-Spezifikation und NOPASSWD-Tag.

6.  [sshd(8) Manual, Abschnitt AUTHORIZED_KEYS FILE FORMAT](https://man.openbsd.org/sshd.8): Format der authorized_keys-Datei und die Anforderungen an die Dateiberechtigungen.
