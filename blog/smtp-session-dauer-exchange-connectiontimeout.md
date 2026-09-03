---
title: "Wie lange bleibt eine SMTP-Session offen? ConnectionTimeout 00:10:00 in Exchange und die Systeme, für die das zu kurz ist"
navTitle: "SMTP-Session-Dauer"
description: "Exchange beendet jede eingehende SMTP-Session nach zehn Minuten, auch wenn sie gerade Daten überträgt. Welche Absender so lange auf einer Verbindung bleiben, wie Sie die tatsächliche Session-Dauer aus dem Protokoll-Log auslesen und wann ConnectionTimeout und ConnectionInactivityTimeout auf einem Relay-Connector angepasst werden sollten."
date: "2026-09-03"
kategorie: "SMTP und Mailflow"
timeToRead: "10 Min. Lesezeit"
themen:
  - "smtp-mailflow"
  - "exchange-onprem-hybrid"
produkte:
  - "exchange-on-premises"
  - "exchange"
protokolle:
  - "smtp"
  - "troubleshooting"
slug: "smtp-session-dauer-exchange-connectiontimeout"
translationId: "article-b40497933bbe0a88"
url: "https://rafaelpfister.ch/blog/smtp-session-dauer-exchange-connectiontimeout"
aiPrompt: |
  Du bist mein Exchange- und Mailflow-Assistent. Hilf mir, die SMTP-Session-Dauer auf einem Exchange-Receive-Connector zu beurteilen: 1. Frage mich, welche Systeme (Relays, Gateways, Applikationen, Scanner) über den Connector einliefern und ob sie Verbindungen über mehrere Nachrichten hinweg offen halten. 2. Lass dir die Ausgabe der Session-Auswertung aus dem Protokoll-Log geben (IP, Mails, Dauer, Timeout-Kennzeichen) und erkläre mir, welche Sessions am ConnectionTimeout abgebrochen wurden. 3. Empfiehl pro Connector konkrete Werte für ConnectionTimeout und ConnectionInactivityTimeout und begründe, warum der internetseitige Connector unverändert bleibt. 4. Nenne mir, was ich stattdessen auf der Client-Seite ändern kann, damit die Verbindung nach einer festen Anzahl Nachrichten neu aufgebaut wird.
---
# Wie lange bleibt eine SMTP-Session offen? ConnectionTimeout 00:10:00 in Exchange und die Systeme, für die das zu kurz ist

Kurzfazit: Eine SMTP-Session hat kein natürliches Ende. RFC 5321 begrenzt nur die Wartezeit auf den jeweils nächsten Schritt, und ein Client darf auf einer offenen Verbindung so lange Nachrichten nachliefern, wie der Server die Verbindung offen hält. Exchange hält sie auf Receive-Connectoren standardmässig zehn Minuten lang offen, dann beendet der Server die Verbindung, unabhängig davon, ob gerade Daten fliessen. Für Exchange-zu-Exchange-Verkehr und für die meisten MTAs ist das bedeutungslos, weil diese Absender von sich aus nach Sekunden neu verbinden. Für Applikationen, Gateways und Lastgeneratoren, die eine einzige Verbindung für einen ganzen Versandlauf verwenden, ist der Wert dagegen die Ursache für Abbrüche, die im Client als Verbindungsfehler und im Exchange-Protokoll-Log als `421 4.4.1 Connection timed out` erscheinen.

## Zwei Timeouts mit unterschiedlicher Bedeutung

Ein Receive-Connector kennt zwei Zeitlimits, die häufig verwechselt werden:

| Parameter | Bedeutung | Standard Mailbox-Server | Standard Edge Transport |
|---|---|---|---|
| `ConnectionInactivityTimeout` | maximale Leerlaufzeit ohne Client-Aktivität, danach wird die Verbindung geschlossen | 00:05:00 | 00:01:00 |
| `ConnectionTimeout` | maximale Gesamtdauer der Verbindung, auch wenn sie aktiv Daten überträgt | 00:10:00 | 00:05:00 |

Beide Werte akzeptieren 1 Sekunde bis 1 Tag (`1.00:00:00`), und `ConnectionTimeout` muss grösser sein als `ConnectionInactivityTimeout`. Die Werte gelten pro Connector, also getrennt für den internetseitigen `Default Frontend <Server>`, für den Transportdienst-Connector `Default <Server>` auf Port 2525 und für jeden selbst angelegten Relay-Connector.

Der Leerlauf-Timeout ist unkritisch: Fünf Minuten entsprechen exakt dem Minimum, das RFC 5321 einem Server als Wartezeit auf das nächste Kommando vorgibt, und ein Client, der fünf Minuten lang nichts sendet, hat die Verbindung in aller Regel selbst vergessen. Der Gesamt-Timeout ist die Eigenheit von Exchange: Er zählt ab dem Verbindungsaufbau und läuft weiter, während der Client Nachricht um Nachricht einliefert. Nach zehn Minuten schliesst Exchange die Verbindung an der Stelle, an der sich der Dialog gerade befindet, notfalls mitten in einem `DATA`-Block.

Auf der Sendeseite gibt es das Gegenstück nicht: Ein Send-Connector hat nur `ConnectionInactivityTimeOut` (Standard zehn Minuten) und begrenzt Sessions stattdessen über `SmtpMaxMessagesPerConnection`, standardmässig 20 Nachrichten. Exchange beendet als Client also selbst jede Verbindung nach spätestens 20 Nachrichten und baut eine neue auf. Das ist der Grund, weshalb der Gesamt-Timeout zwischen Exchange-Servern nie auffällt: Die Sessions dauern Sekunden.

## Was RFC 5321 vorgibt

Der Standard definiert in Abschnitt 4.5.3.2 Mindestwartezeiten, die ein Client pro Protokollschritt einhalten soll, bevor er die Verbindung aufgibt:

| Schritt | Mindest-Timeout auf Client-Seite |
|---|---|
| Warten auf das `220`-Banner | 5 Minuten |
| Antwort auf `MAIL` | 5 Minuten |
| Antwort auf `RCPT` | 5 Minuten |
| Antwort auf `DATA` (die `354`) | 2 Minuten |
| Senden eines Datenblocks | 3 Minuten |
| Antwort auf den abschliessenden Punkt | 10 Minuten |
| Server: Warten auf das nächste Kommando | mindestens 5 Minuten |

Eine Obergrenze für die Gesamtdauer einer Session gibt es im RFC nicht. Ein Client, der auf derselben Verbindung dreissig Minuten lang Nachrichten einliefert und dabei nie länger als ein paar Sekunden schweigt, verhält sich standardkonform. Auffällig ist der letzte Client-Wert: Zehn Minuten Wartezeit auf die Antwort nach dem abschliessenden Punkt, weil der Server in dieser Phase die Nachricht annimmt und übernimmt. Bricht der Client hier zu früh ab, ist die Nachricht bereits zugestellt und wird beim nächsten Versuch ein zweites Mal geliefert. Dieselbe Situation entsteht spiegelbildlich, wenn der Server die Verbindung in diesem Moment wegen des Gesamt-Timeouts schliesst.

Schliesst ein Server die Verbindung mit `421`, soll der Client die laufende Transaktion nach Abschnitt 3.8 so behandeln, als hätte er eine `451` erhalten, also als temporären Fehler mit Wiederholung. Ein MTA mit Queue tut genau das. Eine Applikation ohne Queue meldet stattdessen eine Exception und überlässt den Rest dem Aufrufer.

## Wie lange Absender ihre Sessions tatsächlich offen halten

Die Session-Dauer wird vom Client bestimmt, und die Unterschiede zwischen den Absendertypen sind gross:

| Absender | Typische Session-Dauer | Begrenzt durch |
|---|---|---|
| Exchange Send-Connector | Sekunden | `SmtpMaxMessagesPerConnection` = 20 |
| Postfix mit Verbindungs-Cache | maximal 5 Minuten | `smtp_connection_reuse_time_limit` = 300s |
| Postfix ohne Verbindungs-Cache | eine Nachricht pro Verbindung | Standardverhalten des `smtp`-Clients |
| Applikation mit `.NET SmtpClient`, `JavaMail Transport`, Python `smtplib` | so lange, wie das Objekt lebt: bei einem Batch-Lauf der ganze Lauf | nur durch den Programmcode |
| Quarantäne-Benachrichtigungen von Mail-Gateways | eine Session pro Benachrichtigungslauf | Produktverhalten, teils mit `NOOP`-Keepalive |
| Multifunktionsgeräte, Scan-to-Mail | eine Nachricht pro Verbindung, bei grossen Scans über langsame Leitungen mehrere Minuten | Dateigrösse und Bandbreite |
| Lastgeneratoren wie `smtp-source -d` | bis zum Ende des Laufs | Aufrufparameter |

Die ersten beiden Zeilen erklären, warum der Wert in klassischen Umgebungen jahrelang niemandem auffällt: MTAs bauen Verbindungen von sich aus kurz. Postfix zum Beispiel verwendet eine gecachte Verbindung höchstens fünf Minuten lang und öffnet danach eine neue, und Exchange trennt nach 20 Nachrichten. Beide bleiben damit unter jedem Exchange-Standardwert.

Die Applikationszeile ist der häufigste Problemfall. Ein Batch-Job, der Rechnungen, Lohnabrechnungen oder Systemmeldungen versendet, erzeugt typischerweise ein Client-Objekt, ruft darauf in einer Schleife die Sendemethode auf und schliesst es am Ende. `System.Net.Mail.SmtpClient` verwendet seit .NET Framework 4 für aufeinanderfolgende `Send`-Aufrufe dieselbe Verbindung und sendet `QUIT` erst beim `Dispose`; JavaMail verhält sich mit einem einmal geöffneten `Transport` gleich. Läuft der Job länger als zehn Minuten, fällt irgendwo mittendrin der `421`, und der Job bricht mit einer Exception ab, bei .NET etwa mit dem Text `Service not available, closing transmission channel. The server response was: 4.4.1 Connection timed out`. Welche Nachricht betroffen ist, hängt von der Laufzeit ab, deshalb wirkt der Fehler zufällig: Mal sind es 800 Nachrichten bis zum Abbruch, mal 1200, je nach Nachrichtengrösse und Serverlast.

Die Gateway-Zeile beschreibt einen dokumentierten Fall: Das Symantec (heute Broadcom) Messaging Gateway versendet Spam-Quarantäne-Benachrichtigungen über eine einzige Verbindung und sendet zwischen den Nachrichten `NOOP` als Keepalive. Exchange beantwortet `NOOP` mit der Tarpit-Verzögerung von fünf Sekunden, so dass in zehn Minuten höchstens etwa 120 Benachrichtigungen durchkommen, bevor die Session mit `421 4.4.1` endet und das Gateway neu verbinden muss.

Die Scanner-Zeile ist ein Grössenproblem statt eines Mengenproblems: Ein 60-MB-Scan über eine 2-Mbit/s-Anbindung braucht rund vier Minuten reine Übertragungszeit, bei 100 MB sind es fast sieben Minuten. Auf einem Edge-Transport-Server mit fünf Minuten Gesamt-Timeout reicht das bereits für einen Abbruch, auf einem Mailbox-Server bleibt Reserve, aber nicht viel.

## Was beim Abbruch passiert

Läuft der Gesamt-Timeout ab, schreibt Exchange die Antwort `421 4.4.1 Connection timed out` ins Protokoll-Log, sendet sie an den Client und schliesst die Verbindung. Für die gerade laufende Transaktion gilt: Wurde der abschliessende Punkt noch nicht gesendet, ist die Nachricht nicht angenommen und muss vollständig wiederholt werden. Wurde der Punkt gesendet und die Verbindung vor der `250`-Antwort geschlossen, hat der Client keine Information darüber, ob Exchange die Nachricht übernommen hat; ein sauber implementierter Client wiederholt sie, und der Empfänger erhält sie unter Umständen doppelt. Die Wahrscheinlichkeit dafür ist klein, aber bei tausenden Nachrichten pro Lauf nicht null.

Zu beachten ist ausserdem der Proxy-Pfad: Der Front-End-Transportdienst nimmt die Verbindung auf Port 25 an und reicht sie als eigene SMTP-Session an den Transportdienst auf Port 2525 weiter, wo der Connector `Default <Server>` mit denselben Standardwerten gilt. Eine lange Session erscheint deshalb in beiden Logs, und eine Anpassung muss beide Connectoren umfassen.

## Die tatsächliche Session-Dauer aus dem Protokoll-Log auslesen

Bevor Sie einen Wert ändern, lohnt sich der Blick auf die realen Sessions. Voraussetzung ist ausführliches Protokoll-Logging auf dem betroffenen Connector; auf `Default Frontend <Server>` ist es bereits aktiv, auf allen anderen Connectoren nicht:

```powershell
Set-ReceiveConnector -Identity 'EX01\Relay Applikationen' -ProtocolLoggingLevel Verbose
```

Die Logs liegen unter `%ExchangeInstallPath%TransportRoles\Logs\FrontEnd\ProtocolLog\SmtpReceive` (Front-End) und `%ExchangeInstallPath%TransportRoles\Logs\Hub\ProtocolLog\SmtpReceive` (Transportdienst), benannt nach UTC-Stunde als `RECVyyyyMMddhh-nnnn.log`. Jede Zeile ist ein Protokollereignis mit den Feldern `date-time`, `connector-id`, `session-id`, `sequence-number`, `local-endpoint`, `remote-endpoint`, `event`, `data` und `context`. Alle Zeilen einer Session tragen dieselbe `session-id`, die Session-Dauer ist also die Differenz zwischen dem ersten und dem letzten Zeitstempel dieser ID.

Das folgende Skript wertet die jüngste Logdatei des Tages für einen Connector aus, fasst die Zeilen pro Session zusammen und zeigt die 15 längsten Sessions mit Anzahl Nachrichten, Dauer und der Information, ob Exchange sie mit `421 4.4.1` beendet hat:

```powershell
$logPfad = Join-Path $env:ExchangeInstallPath 'TransportRoles\Logs\FrontEnd\ProtocolLog\SmtpReceive'
$connector = 'Relay Applikationen'
$tag = (Get-Date).ToUniversalTime().ToString('yyyyMMdd')
$datei = Get-ChildItem $logPfad -Filter "RECV$tag*.log" |
    Sort-Object Name -Descending |
    Select-Object -First 1

$sessions = @{}
Get-Content $datei.FullName |
    Where-Object { $_ -notlike '#*' -and $_ -like "*$connector*" } |
    ForEach-Object {
        $c = $_ -split ','
        $s = $c[2]
        if (-not $sessions[$s]) {
            $sessions[$s] = [pscustomobject]@{
                IP = ($c[5] -split ':')[0]; Start = $c[0]; Ende = $c[0]
                Zeilen = 0; Mails = 0; Timeout = $false
            }
        }
        $sessions[$s].Ende = $c[0]
        $sessions[$s].Zeilen++
        if ($c[7] -like 'MAIL FROM*') { $sessions[$s].Mails++ }
        if ($c[7] -like '421 4.4.1*') { $sessions[$s].Timeout = $true }
    }

$sessions.Values |
    Sort-Object Zeilen -Descending |
    Select-Object -First 15 IP, Mails, Zeilen, Timeout,
        @{ n = 'Dauer_s'
           e = { [math]::Round(([datetime]$_.Ende - [datetime]$_.Start).TotalSeconds, 1) } } |
    Format-Table -AutoSize
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Element | Wirkung |
|---|---|
| `$logPfad` | Log-Verzeichnis des Front-End-Transportdiensts; für den Transportdienst `Hub` statt `FrontEnd` einsetzen |
| `$connector` | Namensbestandteil des Connectors; filtert über das Feld `connector-id`, das als `Server\Name` protokolliert wird |
| `$tag` | UTC-Datum, weil die Logdateien nach UTC-Stunde benannt sind |
| `-Filter "RECV$tag*.log"` | nur Receive-Logs des heutigen Tages |
| `Sort-Object Name -Descending`, `Select-Object -First 1` | die jüngste Datei (höchste Stunde, höchste Instanznummer) |
| `$_ -notlike '#*'` | überspringt die Kopfzeilen `#Software`, `#Version`, `#Log-Type`, `#Date`, `#Fields` |
| `$_ -split ','` | zerlegt die CSV-Zeile; die verwendeten Felder 0, 2, 5 und 7 liegen vor dem ersten Freitext und sind damit stabil |
| `$c[2]` | `session-id`, der Gruppierungsschlüssel |
| `($c[5] -split ':')[0]` | IPv4-Adresse aus dem `remote-endpoint` (bei IPv6-Endpunkten ist die Zerlegung anzupassen) |
| `$c[0]` als `Start` und `Ende` | erster und letzter Zeitstempel der Session; `Ende` wird mit jeder Zeile überschrieben |
| `$c[7] -like 'MAIL FROM*'` | zählt Nachrichten über das empfangene `MAIL FROM`-Kommando |
| `$c[7] -like '421 4.4.1*'` | markiert Sessions, die Exchange wegen des Gesamt-Timeouts beendet hat |
| `Sort-Object Zeilen -Descending` | die aktivsten Sessions zuerst; alternativ nach `Dauer_s` sortieren |
| `Dauer_s` | Differenz der ISO-8601-Zeitstempel in Sekunden, auf eine Nachkommastelle gerundet |

</details>

In der Ausgabe erkennen Sie die betroffenen Systeme daran, dass `Timeout` auf `True` steht und `Dauer_s` knapp bei 600 liegt: Die Session hat exakt so lange gelebt, wie der Connector erlaubt. Sessions mit vielen Nachrichten und einer Dauer deutlich unter 600 Sekunden sind unkritisch, auch wenn sie im Moment die längsten sind. Für einen Überblick, welche Quellen betroffen sind, genügt eine Gruppierung der markierten Sessions:

```powershell
$sessions.Values |
    Where-Object { $_.Timeout } |
    Group-Object IP |
    Select-Object Name, Count |
    Sort-Object Count -Descending
```

Zwei Einschränkungen des Ansatzes: Eine Session, die über eine Stundengrenze läuft, verteilt sich auf zwei Logdateien und erscheint in der Einzeldatei verkürzt; für eine Tagesauswertung lesen Sie alle Dateien des Tages ein. Und der Wert `Mails` zählt `MAIL FROM`-Kommandos, also Versuche, nicht angenommene Nachrichten.

## Werte anpassen: auf welchem Connector und wie weit

Die Standardwerte sind ein Schutz für den internetseitigen Connector, auf dem beliebige Gegenstellen Verbindungen belegen können. Dort bleiben sie unverändert; ein legitimer externer MTA verbindet ohnehin neu. Angepasst wird der dedizierte Connector, über den die identifizierten internen Systeme einliefern. Fehlt ein solcher Connector, lässt er sich mit `RemoteIPRanges` auf die Absender-IPs eingeschränkt anlegen; das ist besser, als den Wert auf `Default Frontend` zu erhöhen. Den aktuellen Stand aller Connectoren liefert:

```powershell
Get-ReceiveConnector |
    Format-Table Name, TransportRole, ConnectionTimeout, ConnectionInactivityTimeout, TarpitInterval -AutoSize
```

Die Anpassung selbst, hier mit einer Stunde Gesamtdauer und unverändertem Leerlauf-Timeout:

```powershell
$werte = @{
    ConnectionTimeout           = '01:00:00'
    ConnectionInactivityTimeout = '00:05:00'
}
Set-ReceiveConnector -Identity 'EX01\Relay Applikationen' @werte
Set-ReceiveConnector -Identity 'EX01\Default EX01' @werte
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Parameter | Wirkung |
|---|---|
| `ConnectionTimeout` | Gesamtdauer einer Verbindung; zulässig 00:00:01 bis 1.00:00:00, muss über `ConnectionInactivityTimeout` liegen |
| `ConnectionInactivityTimeout` | Leerlaufzeit bis zum Schliessen; fünf Minuten entsprechen dem RFC-Minimum und können bleiben |
| `-Identity 'EX01\Relay Applikationen'` | der Front-End-Connector der internen Absender |
| `-Identity 'EX01\Default EX01'` | der Transportdienst-Connector auf Port 2525, an den der Front-End die Session weiterreicht |
| `@werte` | Splatting: übergibt beide Parameter aus der Hashtabelle an das Cmdlet |

</details>

Für den Wert gilt: Er soll über der längsten legitimen Session liegen, die die Auswertung gezeigt hat, mit Reserve für Lastspitzen. Eine Stunde deckt die meisten Batch-Läufe ab; für einen nächtlichen Lauf von zwei Stunden ist entsprechend mehr nötig, bis zum Maximum von einem Tag. Beliebig hoch sollte der Wert jedoch auch auf einem internen Connector nicht sein, weil `MaxInboundConnectionPerSource` (Standard 20) und `MaxInboundConnection` (Standard 5000) mitzählen: Ein Client, der zusätzlich zu einer hängen gebliebenen Verbindung immer wieder neue öffnet, erreicht das Limit pro Quelle umso früher, je länger die alten Verbindungen bestehen bleiben.

Für Absender, die zwischen Nachrichten `NOOP` senden, sollte `TarpitInterval` auf demselben Connector auf `00:00:00` gesetzt werden. Die Tarpit-Verzögerung ist für authentifizierte oder per IP eingeschränkte interne Absender ohne Nutzen und verlängert jede Session künstlich.

Die Änderung auf der Exchange-Seite behebt das Symptom. Die stabilere Lösung liegt im Client: Er baut die Verbindung nach einer festen Anzahl Nachrichten neu auf, so wie es Exchange mit 20 und Postfix mit fünf Minuten tun. Bei `.NET SmtpClient` heisst das, das Objekt pro Block von beispielsweise 100 Nachrichten zu erzeugen und zu verwerfen; bei JavaMail wird der `Transport` entsprechend geschlossen und neu geöffnet. Damit funktioniert der Versand auch gegen Ziele, deren Timeouts sich nicht anpassen lassen, insbesondere Exchange Online, dessen Inbound-Connectoren keine Timeout-Parameter kennen.

## Weitere Zeitlimits auf dem Pfad

Der Exchange-Wert ist nicht das einzige Limit. Firewalls und Loadbalancer führen eigene Leerlauf-Timer für TCP-Verbindungen: Ein FastL4-Profil auf einer F5 BIG-IP steht standardmässig auf 300 Sekunden, ein Azure Load Balancer auf vier Minuten. Diese Timer messen Leerlauf, nicht Gesamtdauer, und greifen deshalb bei Sendepausen, etwa wenn ein Batch-Job zwischen zwei Blöcken Daten aus der Datenbank liest. Massgebend ist immer der kleinste Wert auf dem gesamten Pfad. Wie Sie die Timeouts auf einem Loadbalancer für stehende SMTP-Verbindungen dimensionieren, beschreibt der Beitrag [F5 BIG-IP als Outbound-Proxy für den Mail-Massenversand](https://rafaelpfister.ch/blog/f5-big-ip-outbound-smtp-massenversand).

## Quellen

1.  [Microsoft Learn: Set-ReceiveConnector](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-receiveconnector): Referenz mit den Standardwerten und Wertebereichen von `ConnectionTimeout`, `ConnectionInactivityTimeout`, `TarpitInterval`, `MaxInboundConnection` und `MaxInboundConnectionPerSource` für Mailbox- und Edge-Transport-Server.

2.  [Microsoft Learn: Set-SendConnector](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-sendconnector): `ConnectionInactivityTimeOut` und `SmtpMaxMessagesPerConnection` auf der Sendeseite.

3.  [Microsoft Learn: Protocol logging](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/protocol-logging): Speicherorte, Dateinamen und Feldaufbau der SMTP-Protokoll-Logs für Front-End- und Transportdienst.

4.  [Microsoft Learn: Mail routing in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing): der Front-End-Transportdienst als zustandsloser Proxy vor dem Transportdienst.

5.  [RFC 5321, Abschnitt 4.5.3.2 Timeouts](https://www.rfc-editor.org/rfc/rfc5321.html#section-4.5.3.2): Mindestwartezeiten pro Protokollschritt, die Begründung für die zehn Minuten nach dem abschliessenden Punkt und das Verhalten bei `421` in Abschnitt 3.8.

6.  [Postfix: postconf(5)](https://www.postfix.org/postconf.5.html): `smtp_connection_reuse_time_limit` (300s) und `smtpd_timeout` als Beispiel für einen MTA, der Sessions von sich aus kurz hält.

7.  [Broadcom Knowledge Base: Quarantine notification process appears to be failing, logs may show 421 4.4.1 Connection timed out](https://knowledge.broadcom.com/external/article/154389/quarantine-notification-process-appears.html): dokumentierter Fall eines Gateways, das mit `NOOP`-Keepalive und Tarpit in den Exchange-Gesamt-Timeout läuft.

8.  [Microsoft Learn: SmtpClient Class](https://learn.microsoft.com/en-us/dotnet/api/system.net.mail.smtpclient): Verbindungswiederverwendung über mehrere `Send`-Aufrufe und `QUIT` erst beim `Dispose`.
