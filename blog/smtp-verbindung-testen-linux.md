---
title: "SMTP unter Linux testen: von der TCP-Verbindung bis zur zugestellten Mail"
navTitle: "SMTP testen"
description: "Wenn eine Appliance keine Mails mehr ausliefert, hilft ein manueller SMTP-Test mehr als jedes Log. Wie Sie mit Bordmitteln Schicht für Schicht prüfen, welche Fehlerbilder was bedeuten und warum ein Load Balancer die Diagnose verfälscht."
date: "2026-07-31"
kategorie: "SMTP und Mailflow"
timeToRead: "10 Min. Lesezeit"
themen:
  - "smtp-mailflow"
  - "e-mail-verschluesselung"
produkte:
  - "uebergreifend"
protokolle:
  - "smtp"
  - "tcp"
  - "tls"
  - "dns"
  - "verschluesselung"
slug: "smtp-verbindung-testen-linux"
translationId: "article-cb44a92c03a47bc0"
url: "https://rafaelpfister.ch/blog/smtp-verbindung-testen-linux"
---
# SMTP unter Linux testen: von der TCP-Verbindung bis zur zugestellten Mail

Wenn ein Mailgateway plötzlich nichts mehr zustellt, liefern die Logs der Appliance oft nur die Endstufe der Geschichte: eine Zustellung schlägt fehl, die Queue wächst, eine Fehlermeldung nennt einen Timeout. Woran es tatsächlich liegt, zeigt erst ein manueller Test von der Kommandozeile aus. SMTP ist ein Klartextprotokoll, das sich vollständig von Hand sprechen lässt, und genau das macht es zu einem der dankbarsten Diagnosewerkzeuge im Mailbetrieb.

Der zweite Grund für den manuellen Test: Auf Appliances lässt sich meist nichts installieren. Kein Paketmanager, keine Root-Rechte, kein `swaks`. Alle folgenden Schritte funktionieren deshalb mit dem, was auf praktisch jedem Linux-System ohnehin vorhanden ist.

## Die Schichten trennen

Ein fehlgeschlagener Mailversand kann auf fünf verschiedenen Ebenen scheitern, und jede erzeugt ein anderes Fehlerbild:

1. **Namensauflösung:** Der Zielhost lässt sich nicht in eine IP-Adresse übersetzen.
2. **TCP-Verbindung:** Die Verbindung zum Port kommt nicht zustande oder wird zurückgesetzt.
3. **SMTP-Dialog:** Die Verbindung steht, aber der Server lehnt Absender, Empfänger oder Inhalt ab.
4. **Transportverschlüsselung:** STARTTLS fehlt, das Zertifikat ist ungültig oder die TLS-Version passt nicht.
5. **Absenderprüfung:** Die Mail wird angenommen und beim Empfänger wegen SPF, DKIM oder DMARC verworfen.

Die Diagnose gewinnt enorm, wenn Sie diese Ebenen nacheinander und einzeln prüfen, statt gleich eine vollständige Testmail abzuschicken. Ein misslungener Gesamtversuch sagt Ihnen nur, dass etwas nicht geht. Die Schichtprüfung sagt Ihnen, was.

## Schritt 1: Namensauflösung

```bash
getent hosts relay.example.com
```

Bleibt die Ausgabe leer, ist auf diesem Host kein Nameserver erreichbar oder er beantwortet externe Namen nicht. Das ist häufiger, als man denkt: Appliances in abgeschotteten Zonen bekommen oft nur einen internen Resolver, der ausschliesslich eigene Zonen kennt.

```bash
cat /etc/resolv.conf; grep hosts: /etc/nsswitch.conf
```

Fehlt die Auflösung, testen Sie im Folgenden direkt gegen die IP-Adresse. Das ist für die Diagnose völlig ausreichend und trennt das DNS-Problem sauber vom Transportproblem ab. Für den Produktivbetrieb bleibt die fehlende Auflösung natürlich ein eigener Befund, der behoben gehört.

## Schritt 2: Erreichbarkeit des Ports

Für die reine TCP-Prüfung genügt bash. Das Pseudogerät `/dev/tcp` öffnet eine Verbindung, ohne dass `nc` oder `telnet` installiert sein müssen:

```bash
timeout 10 bash -c 'exec 3<>/dev/tcp/192.0.2.25/25' ; echo "exit=$?"
```

Der Exit-Code ist hier die eigentliche Information:

| exit | Bedeutung |
|---|---|
| `0` | Verbindung steht, der Port ist offen |
| `124` | Timeout: Pakete werden verworfen, typisch für eine Firewall mit DROP-Regel |
| `1` | Sofortige Ablehnung (RST) oder fehlende Route |

Der Unterschied zwischen 124 und 1 ist in der Praxis der wichtigste Hinweis überhaupt. Ein Timeout bedeutet, dass jemand auf dem Weg schweigend verwirft, und das ist fast immer eine Firewall-Regel. Ein sofortiges RST kommt dagegen von einem System, das antwortet, den Dienst aber nicht anbietet.

Prüfen Sie gleich beide relevanten Ports und zusätzlich ein beliebiges anderes Ziel, um zu sehen, ob der Host überhaupt ausgehende Verbindungen aufbauen darf:

```bash
for t in "192.0.2.25 25" "192.0.2.25 587" "1.1.1.1 443"; do set -- $t; timeout 8 bash -c "exec 3<>/dev/tcp/$1/$2" 2>/dev/null; echo "$1:$2 -> exit=$?"; done
```

Läuft auch die Gegenprobe ins Leere, hat das System generell keinen direkten Ausgang und der Verkehr gehört über ein internes Relay oder einen Proxy. Weiter unten mehr dazu, warum dieser Fall besonders tückisch ist.

Fehlt `/dev/tcp`, ist die Shell keine bash. Unter `sh`, `ash` oder `ksh` gibt es das Feature nicht, was gerne als vermeintliches Netzwerkproblem fehlgedeutet wird:

```bash
ps -p $$ -o comm= ; echo "BASH_VERSION=${BASH_VERSION:-keine bash}"
```

## Schritt 3: Erst zuhören, nicht senden

Ein SMTP-Server begrüsst von sich aus mit einem `220`-Banner. Der aussagekräftigste Einzeltest besteht deshalb darin, eine Verbindung zu öffnen und nichts zu tun:

```bash
{ exec 3<>/dev/tcp/192.0.2.25/25 && timeout 15 cat <&3; echo "[ende exit=$?]"; }
```

Diese wenigen Zeichen trennen zwei völlig verschiedene Situationen. Kommt ein `220 mail.example.com ESMTP`, spricht die Gegenstelle und alle weiteren Fehler liegen im Dialog. Kommt nichts, liegt es nicht an einem falsch formulierten Kommando Ihrerseits, denn Sie haben ja keines geschickt.

Der Filedeskriptor bleibt danach in der Shell offen. Schliessen Sie ihn, bevor Sie den nächsten Test starten, sonst arbeiten Sie unter Umständen mit einer alten, halbtoten Verbindung weiter:

```bash
exec 3<&- 3>&-
```

## Schritt 4: Der SMTP-Dialog von Hand

Steht das Banner, führen Sie den vollständigen Dialog. Wichtig ist ein mitlaufender Leseprozess, damit Sie jede Antwort in dem Moment sehen, in dem sie kommt. Ein Skript, das erst alles sendet und danach liest, zeigt Ihnen bei einem Abbruch mitten im Dialog gar nichts:

```bash
{
exec 3<>/dev/tcp/192.0.2.25/25
cat <&3 & R=$!
sleep 1; printf 'EHLO host.example.com\r\n' >&3
sleep 2; printf 'MAIL FROM:<absender@example.com>\r\n' >&3
sleep 2; printf 'RCPT TO:<empfaenger@example.net>\r\n' >&3
sleep 2; printf 'DATA\r\n' >&3
sleep 2; printf 'From: absender@example.com\r\nTo: empfaenger@example.net\r\nSubject: Relay-Test\r\nDate: %s\r\nMessage-ID: <%s@example.com>\r\n\r\nTestnachricht\r\n.\r\n' "$(date -R)" "$(date +%s).$$" >&3
sleep 3; printf 'QUIT\r\n' >&3
sleep 2; kill $R 2>/dev/null
}
```

Zwei Details entscheiden über Erfolg oder Frust. SMTP verlangt CRLF als Zeilenende, deshalb `printf` mit `\r\n` und nicht `echo`. Und der Punkt auf einer eigenen Zeile beendet den Nachrichtenteil; er muss als `\r\n.\r\n` gesendet werden.

Der erwartete Verlauf: `220` beim Verbindungsaufbau, `250` auf EHLO, `250 2.1.0` auf MAIL FROM, `250 2.1.5` auf RCPT TO, `354` auf DATA und am Ende `250 2.0.0 Ok: queued as <id>`. Notieren Sie sich die Queue-ID. Damit lässt sich die Nachricht beim betreibenden Provider nachverfolgen, falls sie beim Empfänger nie ankommt.

Der EHLO-Name verdient Beachtung: Manche Relays prüfen ihn gegen Forward- und Reverse-DNS und antworten sonst mit `501` oder `504`. Verwenden Sie den tatsächlichen FQDN des sendenden Systems, nicht den Kurznamen.

## Schritt 5: STARTTLS und Zertifikat

Für die verschlüsselte Verbindung übernimmt `openssl s_client` die STARTTLS-Verhandlung selbst und übergibt danach den Kanal an die Standardeingabe:

```bash
openssl s_client -connect 192.0.2.25:25 -starttls smtp -tls1_2 -brief </dev/null
```

Verbinden Sie sich über die IP-Adresse, weil DNS fehlt, greift die Hostnamensprüfung ins Leere. Der Zertifikatsname passt dann nicht zur numerischen Adresse. SNI und Prüfnamen lassen sich explizit setzen, ganz ohne DNS-Abfrage:

```bash
openssl s_client -connect 192.0.2.25:25 -servername mail.example.com -verify_hostname mail.example.com -starttls smtp -tls1_2 -brief </dev/null
```

Zwei Fehlerbilder tauchen hier regelmässig auf und werden gerne falsch gedeutet.

**«Didn't find STARTTLS in server response, trying anyway»** bedeutet, dass der Server in seiner EHLO-Antwort kein STARTTLS angeboten hat. `openssl` schickt trotzdem ein TLS-ClientHello, der Server sieht darin Protokollmüll und die Verbindung endet mit `wrong version number` oder `write:errno=32` (EPIPE). Beide Meldungen sind Folgefehler. Die eigentliche Information ist: kein STARTTLS. Sehen Sie mit dem Klartext-Dialog aus Schritt 4 nach, welche Capabilities der Server tatsächlich meldet.

**Kein STARTTLS auf einem internen Hop** ist oft völlig korrekt. Wenn ein Load Balancer die Verbindung auf Layer 4 weiterreicht, verhandelt nicht er das TLS, sondern erst das dahinterliegende System gegenüber dem eigentlichen Ziel. Auf dem internen Segment im Klartext zu testen ist dann kein Sicherheitsmangel, sondern schlicht die Architektur.

## Schritt 6: Python als Alternative

Ist Python vorhanden, ersparen Sie sich das Timing-Gebastel mit `sleep`. Die Standardbibliothek reicht aus, es muss nichts nachinstalliert werden:

```python
#!/usr/bin/env python3
import smtplib, ssl
from email.message import EmailMessage
from email.utils import formatdate, make_msgid

msg = EmailMessage()
msg["From"] = "absender@example.com"
msg["To"] = "empfaenger@example.net"
msg["Subject"] = "Relay-Test"
msg["Date"] = formatdate(localtime=True)
msg["Message-ID"] = make_msgid(domain="example.com")
msg.set_content("Testnachricht\n")

ctx = ssl.create_default_context()
ctx.minimum_version = ssl.TLSVersion.TLSv1_2

s = smtplib.SMTP("192.0.2.25", 25, timeout=30, local_hostname="host.example.com")
s.set_debuglevel(1)
s.ehlo()
if s.has_extn("starttls"):
    s.starttls(context=ctx, server_hostname="mail.example.com")
    s.ehlo()
    print("TLS:", s.sock.version(), s.sock.cipher()[0])
s.send_message(msg)
s.quit()
```

`set_debuglevel(1)` protokolliert den kompletten Dialog inklusive aller Antwortcodes, und `smtplib` liest jede Antwort synchron. Ein Abbruch erscheint als `SMTPServerDisconnected` samt der letzten empfangenen Zeile, statt als stiller Broken Pipe.

Zwei Fallstricke: `server_hostname` ist beim Verbinden über eine IP-Adresse zwingend, sonst prüft Python das Zertifikat gegen die numerische Adresse. Und wenn Sie die Prüfung bewusst abschalten, muss `check_hostname = False` vor `verify_mode = ssl.CERT_NONE` stehen, sonst wirft Python einen `ValueError`.

## Absenderadresse, SPF und Alignment

Ein Test schlägt erstaunlich oft nicht am Transport fehl, sondern an der gewählten Absenderadresse. Drei Punkte gehören vorab geprüft.

Die Absenderdomain muss ein FQDN sein. Eine Adresse wie `test@meine-testdomain` ohne Top-Level-Domain wird von vielen MTAs bereits beim MAIL FROM mit `501` oder `553` abgelehnt.

Die Domain muss den verwendeten Versandweg autorisieren. Ein Blick in den SPF-Record zeigt, ob die ausgehende Adresse abgedeckt ist:

```bash
dig +short TXT example.com | grep spf1
```

Und bei aktivem DMARC entscheidet das Alignment. Steht im Record `aspf=s`, müssen die Domain im Envelope (MAIL FROM) und die Domain im `From:`-Header exakt übereinstimmen, nicht nur verwandt sein:

```bash
dig +short TXT _dmarc.example.com
```

Bei `p=reject` verschwindet eine Testmail mit unpassendem Alignment beim Empfänger kommentarlos, obwohl Ihr Relay sie mit `250 queued` angenommen hat. Das ist die häufigste Ursache für Nachrichten, die sendeseitig als erfolgreich gelten und trotzdem nie ankommen.

## Wenn ein Load Balancer dazwischensteht

In grösseren Umgebungen sendet eine Appliance selten direkt ins Internet. Üblich ist ein virtueller Server auf einem Load Balancer, der die Verbindung annimmt, per Source-NAT auf eine definierte Adresse umschreibt und erst dann nach aussen weiterleitet. Für die Diagnose hat das eine unangenehme Konsequenz.

Ein virtueller Server, der auf Layer 4 arbeitet, quittiert den TCP-Handshake sofort, bevor er selbst eine Verbindung zum Ziel aufgebaut hat. Scheitert diese zweite Verbindung, sehen Sie auf dem Client eine erfolgreich aufgebaute und unmittelbar danach zurückgesetzte Verbindung: `Connection reset by peer`, ohne jedes SMTP-Banner. Der Fehler liegt dann nicht bei Ihnen und auch nicht am Ziel, sondern im Pool hinter dem virtuellen Server, etwa weil ein Member als down markiert ist oder der hinterlegte FQDN nicht aufgelöst wird.

Das erklärt auch, warum ein Test direkt gegen das Internetziel scheitern muss, wenn die Weiterleitungsregel nur Verkehr von der bereits umgeschriebenen SNAT-Adresse akzeptiert. Verbindungen mit der ursprünglichen Quelladresse passen auf keine Regel und werden verworfen. Testen Sie in solchen Umgebungen immer gegen den vorgesehenen virtuellen Server, nicht gegen das eigentliche Ziel.

Welche Quelladresse Ihr System für ein bestimmtes Ziel verwendet, beantwortet eine einzige Zeile. Der Wert hinter `src` ist genau die Angabe, die das Netzwerkteam für die Freischaltung braucht:

```bash
ip route get 192.0.2.25
```

Steht das System hinter NAT, sieht die Gegenstelle nicht diese, sondern die öffentliche Adresse des Perimeters. Ermitteln lässt sich die von innen nicht, solange gar kein Verkehr durchkommt; sie steht in der NAT-Regel.

## Fehlerbilder auf einen Blick

| Beobachtung | Wahrscheinliche Ursache |
|---|---|
| `Name or service not known` | Keine Namensauflösung auf dem Host |
| Timeout, exit 124 | Firewall verwirft stumm (DROP) |
| `Connection refused` | Kein Dienst auf dem Port oder REJECT-Regel |
| Verbindung steht, kein Banner, dann RST | Load Balancer nimmt an, Backend nicht erreichbar |
| `Didn't find STARTTLS` | Server bietet keine Transportverschlüsselung an |
| `wrong version number`, `errno=32` | Folgefehler nach erzwungenem TLS ohne STARTTLS |
| `501` / `553` auf MAIL FROM | Absenderdomain kein FQDN oder nicht erlaubt |
| `554 relay access denied` | Quell-IP beim Relay nicht freigeschaltet |
| `250 queued`, aber keine Zustellung | SPF, DKIM oder DMARC-Alignment beim Empfänger |

## Lasttests und Rate Limits

Für Volumentests gilt eine Regel, die im Alltag oft übersehen wird: Nicht die Anzahl Nachrichten ist das Problem, sondern die Anzahl Verbindungen. Typische Relays erlauben einige Hundert Verbindungen pro Minute, aber Zehntausende Nachrichten. Halten Sie deshalb eine Session offen und schicken Sie viele Envelopes darüber, statt für jede Nachricht neu zu verbinden.

In `smtplib` heisst das schlicht, dasselbe Verbindungsobjekt mehrfach zu verwenden und die Session nach einer festen Zahl Nachrichten kontrolliert neu aufzubauen. Wer dagegen pro Mail eine neue Verbindung öffnet, reisst das Verbindungslimit weit vor dem Nachrichtenlimit und provoziert Ablehnungen, die wie ein Problem der Gegenstelle aussehen.

## Fazit

Der manuelle SMTP-Test ist kein Notbehelf für Umgebungen ohne Werkzeuge, sondern die präziseste Diagnose, die im Mailbetrieb zur Verfügung steht. Er trennt Namensauflösung, Erreichbarkeit, Protokolldialog und Verschlüsselung sauber voneinander und liefert für jede Ebene ein eindeutiges Ergebnis. Wer zuerst nur lauscht, dann den Dialog von Hand führt und die Antwortcodes ernst nimmt, kommt in wenigen Minuten zu einer Aussage, mit der sich ein Ticket beim Netzwerk- oder Providerteam belegen lässt: mit Quelladresse, Zielport, beobachtetem Verhalten und Exit-Code.

## Quellen

1.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): Definiert den SMTP-Dialog, die Kommandoreihenfolge und die Bedeutung der Antwortcodes.

2.  [RFC 3207: SMTP Service Extension for Secure SMTP over TLS](https://www.rfc-editor.org/rfc/rfc3207): Beschreibt STARTTLS als Erweiterung, inklusive des Verhaltens, wenn der Server sie nicht anbietet.

3.  [RFC 7208: Sender Policy Framework (SPF)](https://www.rfc-editor.org/rfc/rfc7208): Aufbau und Auswertung des SPF-Records für die Autorisierung sendender Systeme.

4.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc7489): Regelt Alignment zwischen Envelope- und Header-Absender sowie die Policy-Auswertung.

5.  [OpenSSL: s_client](https://docs.openssl.org/master/man1/openssl-s_client/): Referenz zu den verwendeten Optionen, unter anderem `-starttls`, `-servername` und `-verify_hostname`.

6.  [Python-Dokumentation: smtplib](https://docs.python.org/3/library/smtplib.html): Standardbibliothek für SMTP-Sitzungen inklusive STARTTLS und Debug-Ausgabe.

7.  [Bash Reference Manual: Redirections](https://www.gnu.org/software/bash/manual/bash.html#Redirections): Dokumentiert `/dev/tcp` als bash-eigenes Pseudogerät für Netzwerkverbindungen.
