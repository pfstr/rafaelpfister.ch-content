---
title: "Exchange-Mailfluss analysieren: Message Tracking, SMTP-Protokolle und Receive-Connectoren"
navTitle: "Mailfluss analysieren"
description: "Wie Sie in Exchange OnPrem, Hybrid und Exchange Online systematisch herausfinden, wo eine Nachricht geblieben ist: die Abfragen mit Beispielausgaben, das SMTP-Protokoll richtig lesen, und die Punkte, die regelmässig zu Fehlschlüssen führen."
date: "2026-08-11"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "22 Min. Lesezeit"
themen:
  - "exchange-onprem-hybrid"
  - "smtp-mailflow"
  - "microsoft-365-exchange"
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange"
  - "exchange-on-premises"
  - "exchange-hybrid"
  - "exchange-online"
protokolle:
  - "smtp"
  - "powershell"
  - "troubleshooting"
related:
  - "typische-ursachen-fuer-mail-loops-und-deren-behebung"
  - "einliefernde-ip-adressen-aggregieren"
  - "mailflow-fehlersuche-kontrollierte-experimente"
slug: "exchange-message-tracking-und-receive-connectoren-analysieren"
translationId: "article-ad93c41ab6cd20e6"
url: "https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren"
draft: false
---

# Exchange-Mailfluss analysieren: Message Tracking, SMTP-Protokolle und Receive-Connectoren

Die häufigste Frage im Mailbetrieb lautet: Eine Nachricht ist nicht angekommen, wo ist sie geblieben? Das Message Tracking beantwortet das zuverlässig, aber nur, wenn Sie wissen, was es Ihnen **nicht** sagt. Dieser Artikel beschreibt das Vorgehen in der Reihenfolge, in der es sich bewährt hat, zeigt zu jeder Abfrage die typische Ausgabe, und benennt die Fehlerquellen, die regelmässig Stunden kosten, weil sie plausible, aber falsche Schlüsse nahelegen.

Alle Beispiele verwenden generische Namen: `SRV-MAIL01` und `SRV-MAIL02` als Transportserver, `example.com` als Domäne. Wenn Sie die Befehle für Ihre Umgebung zusammenklicken wollen, statt sie abzutippen: Der [Befehls-Generator](https://rafaelpfister.ch/tools/command-builder) enthält die gängigen Message-Tracking- und Mitschnitt-Befehle für PowerShell und Unix-Shell nebeneinander, komplett lokal im Browser.

## Der Grundsatz: erst lokalisieren, dann erklären

Der Reflex ist, sofort nach der Ursache zu suchen. Effizienter ist es, zuerst zu bestimmen, wie weit die Nachricht überhaupt gekommen ist. Das grenzt den Suchraum in einem Schritt drastisch ein, denn danach wissen Sie, ob Sie im eigenen System, beim vorgelagerten Gateway oder beim Ziel suchen müssen.

Die Reihenfolge lautet deshalb: Nachricht finden, letztes Ereignis lesen, Fehlergrund lesen, Einzelfall oder Muster bestimmen, und erst dann den Einlieferungsweg rekonstruieren.

## Schritt 1: Die Nachricht finden

Beginnen Sie über den Empfänger, denn den kennen Sie fast immer. Wichtig ist, die Abfrage über **alle** Transportserver laufen zu lassen, nicht nur über einen.

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object {
    Get-MessageTrackingLog -Server $_ `
        -Start (Get-Date).AddHours(-6) `
        -ResultSize Unlimited `
        -Recipients "empfaenger@example.com"
} | Sort-Object Timestamp |
    Format-Table Timestamp, ServerHostname, EventId, Source, ConnectorId, MessageId `
        -AutoSize -Wrap
```

Eine typische Ausgabe für eine Nachricht, die sauber durchgelaufen ist:

```text
Timestamp           ServerHostname EventId      Source  ConnectorId
---------           -------------- -------      ------  -----------
11.08.2026 08:27:15 SRV-MAIL02     HARECEIVE    SMTP
11.08.2026 08:27:15 SRV-MAIL01     RECEIVE      SMTP    SRV-MAIL01\Default SRV-MAIL01
11.08.2026 08:27:15 SRV-MAIL01     HAREDIRECT   SMTP
11.08.2026 08:27:15 SRV-MAIL01     RESOLVE      ROUTING
11.08.2026 08:27:15 SRV-MAIL01     AGENTINFO    AGENT
11.08.2026 08:27:16 SRV-MAIL01     SENDEXTERNAL SMTP    Outbound-to-O365
11.08.2026 08:27:53 SRV-MAIL02     HADISCARD    SMTP
```

Findet die Abfrage nichts, prüfen Sie, ob der Empfänger über einen Verteiler expandiert wurde. Dann suchen Sie besser über den Absender:

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object {
    Get-MessageTrackingLog -Server $_ `
        -Start (Get-Date).AddHours(-6) `
        -ResultSize Unlimited
} | Where-Object { $_.Sender -like "*@example.com" } |
    Sort-Object Timestamp |
    Format-Table Timestamp, EventId, Sender,
        @{n='To'; e={$_.Recipients -join ','}}, MessageSubject -AutoSize -Wrap
```

## Schritt 2: Das letzte Ereignis lesen

Die gesamte Diagnose hängt am **letzten** `EventId` der Nachricht. Er sagt Ihnen, welcher Suchraum als Nächstes dran ist.

| Letzter EventId | Bedeutung | Nächster Schritt |
|---|---|---|
| `RECEIVE`, danach nichts | Nachricht steckt fest | Queues prüfen |
| `SEND` oder `SENDEXTERNAL` | erfolgreich übergeben | beim nächsten Hop weitersuchen |
| `FAIL` | endgültig gescheitert | Grund in `RecipientStatus` lesen |
| `DEFER` | Wiederholung läuft | Queue und Zielsystem prüfen |
| `DROP` oder `POISONMESSAGE` | verworfen | Transportregel oder Agent |
| `DELIVER` | in ein lokales Postfach zugestellt | Postfachregeln prüfen |
| `RESOLVE` | Empfänger wurde umgeschrieben | Zieladresse im Eintrag lesen |

`RESOLVE` ist in Hybrid-Umgebungen der aufschlussreichste Zwischenschritt, weil dort die Umschreibung auf die Cloud-Routingadresse sichtbar wird:

```text
EventId : RESOLVE
Source  : ROUTING
Sender  : absender@example.com
To      : BENUTZER@example.mail.onmicrosoft.com
```

Steht dort die erwartete `onmicrosoft.com`-Adresse, ist das Empfängerobjekt korrekt konfiguriert, und Sie können das Thema abhaken. Steht dort weiterhin die ursprüngliche Adresse, fehlt am lokalen Objekt die Zieladresse, und Exchange versucht lokal zuzustellen.

Steckt die Nachricht fest, zeigt die Queue den Grund meist im Klartext:

```powershell
Get-Queue -Server SRV-MAIL01 |
    Where-Object { $_.MessageCount -gt 0 } |
    Format-Table Identity, DeliveryType, Status, MessageCount, NextHopDomain, LastError `
        -AutoSize -Wrap
```

## Fehlerquelle 1: Tracking ist serverbezogen, und viele Einträge sind Schattenkopien

Wenn Sie in der Ausgabe Paare aus `HARECEIVE` und `HADISCARD` sehen, oft mit dem Zusatz `ExplicitlyDiscarded`, dann hat dieser Server die Nachricht **nicht verarbeitet**. Er hielt nur eine Schattenkopie im Rahmen der Shadow Redundancy, während ein anderer Server die eigentliche Zustellung übernahm. Sobald der primäre Server Erfolg meldet, verwirft der Partner seine Kopie.

So sieht das aus, wenn Sie nur den falschen Server abgefragt haben:

```text
Timestamp           EventId    SourceContext
---------           -------    -------------
11.08.2026 09:47:13 HARECEIVE  1a2aae6b-f3a3-4e04-9233-7de460b92223
11.08.2026 09:48:07 HADISCARD  ExplicitlyDiscarded;1a2aae6b-f3a3-4e04-9233-7de460b92223
```

Zwei Zeilen, kein Fehler, keine Zustellung. Wer daraus schliesst, die Nachricht sei verschwunden, sucht am falschen Ort. Die eigentliche Verarbeitung steht im Tracking des Partnerservers.

Praktisch heisst das zweierlei. Erstens sind solche Zeilen kein Hinweis auf ein Problem, sondern Normalbetrieb. Zweitens müssen Sie zwingend alle Transportserver abfragen.

## Fehlerquelle 2: `Format-Table` schneidet genau die entscheidenden Spalten ab

Der Fehlergrund steht in `RecipientStatus`, und dieses Feld ist lang. In einer Tabelle fällt es entweder ganz weg oder wird abgeschnitten. Genau das führt dazu, dass man den `FAIL` sieht, aber nicht den Grund, und stattdessen zu raten beginnt.

Sobald Sie einen Fehlerfall gefunden haben, wechseln Sie deshalb auf `Format-List` und lösen die Sammelfelder auf:

```powershell
Get-MessageTrackingLog -Server SRV-MAIL01 `
    -Start (Get-Date).AddHours(-6) `
    -ResultSize Unlimited `
    -Recipients "empfaenger@example.com" `
    -EventId FAIL |
  Format-List Timestamp, Sender,
    @{n='To';     e={$_.Recipients -join ','}},
    @{n='Status'; e={$_.RecipientStatus -join ' | '}},
    MessageSubject, MessageId, SourceContext
```

Und so sieht der Unterschied aus. Erst die Tabellenansicht, die nichts erklärt:

```text
Timestamp           EventId ConnectorId
---------           ------- -----------
11.08.2026 09:47:13 FAIL    Outbound-to-O365
```

Dann dieselbe Nachricht als Liste:

```text
Timestamp      : 11.08.2026 09:47:13
Sender         : dienst@example-test.com
To             : BENUTZER@example.mail.onmicrosoft.com
Status         : [{LED=550 5.1.8 Access denied, bad outbound sender AS(42000001)
                 [XX1PEPF00000000.eurprd02.prod.outlook.com]};{MSG=};
                 {FQDN=10.0.0.40};{IP=10.0.0.40};{LRT=11.08.2026 09:47:13}]
MessageSubject : Statusmeldung Nachtlauf
MessageId      : <1897281176.1319@app01.intern.example.com>
```

Die Diagnose steht damit fest, ohne dass Sie eine einzige Vermutung gebraucht hätten: Die Gegenstelle beanstandet den Absender. `LED` enthält die vollständige SMTP-Antwort, `FQDN` und `IP` benennen das System, das geantwortet hat, und `LRT` den Zeitpunkt des letzten Versuchs.

## Schritt 3: Einzelfall oder Muster?

Bevor Sie sich in einen einzelnen Fall vertiefen, klären Sie den Umfang. Diese eine Abfrage entscheidet, ob Sie es mit einer Randnotiz oder mit einem Vorfall zu tun haben:

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object {
    Get-MessageTrackingLog -Server $_ `
        -Start (Get-Date).AddHours(-8) `
        -EventId FAIL -ResultSize Unlimited
} | Where-Object { ($_.RecipientStatus -join '') -like "*5.1.8*" } |
    Group-Object { ($_.Sender -split '@')[-1] } |
    Sort-Object Count -Descending |
    Format-Table Count, Name -AutoSize
```

Ersetzen Sie `5.1.8` durch den Statuscode, den Sie untersuchen. Die Ausgabe beantwortet die Frage in einer Zeile:

```text
Count Name
----- ----
    9 example-test.com
```

Eine einzige Absenderdomäne bedeutet: eng begrenztes Problem, kein Vorfall, Sie können in Ruhe weitersuchen. Stünden dort zwanzig verschiedene Domänen, hätten Sie einen laufenden Ausfall, und alles andere müsste warten. Diese Unterscheidung so früh zu treffen, spart erfahrungsgemäss am meisten Zeit.

## Fehlerquelle 3: Die `ConnectorId` nennt nicht den echten Receive-Connector

Das ist die teuerste Fehlerquelle, weil die Ausgabe seriös aussieht. Mail, die ein Client oder ein Fremdsystem auf Port 25 einliefert, trifft zuerst den **Front End Transport**. Dieser reicht die Nachricht an den **Transport Service** auf Port 2525 weiter. Das Message Tracking wird erst dort geschrieben, der Front End Transport schreibt kein eigenes Tracking.

Die Folge sehen Sie an dieser Zeile:

```text
EventId        : RECEIVE
ConnectorId    : SRV-MAIL01\Default SRV-MAIL01
ClientIp       : 10.0.1.11
ClientHostname : srv-mail01.intern.example.com
```

Die `ConnectorId` nennt den internen Connector auf Port 2525, und die `ClientIp` ist die Adresse des **proxyenden Servers**, nicht die des ursprünglichen Einlieferers. Welchen der konfigurierten Connectoren auf Port 25 ein System tatsächlich getroffen hat, steht im Tracking schlicht nicht drin. Wer dieser Angabe glaubt, sucht den Fehler bei einem Connector, der gar nicht beteiligt war.

Es gibt zwei Wege zur Antwort. Der erste ist die Rekonstruktion über die Konfiguration:

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object { Get-ReceiveConnector -Server $_ } |
    Format-List Identity, Enabled,
        @{n='Bindings';       e={$_.Bindings -join ','}},
        @{n='RemoteIPRanges'; e={$_.RemoteIPRanges -join ','}},
        PermissionGroups, AuthMechanism
```

```text
Identity         : SRV-MAIL01\Default Frontend SRV-MAIL01
Bindings         : 10.0.1.11:25
RemoteIPRanges   : 0.0.0.0-255.255.255.255
PermissionGroups : AnonymousUsers, ExchangeServers, ExchangeLegacyServers
AuthMechanism    : Tls, Integrated, BasicAuth, BasicAuthRequireTLS, ExchangeServer

Identity         : SRV-MAIL01\smtp-noauth SRV-MAIL01
Bindings         : 10.0.1.13:25
RemoteIPRanges   : 10.0.20.22,10.0.21.11,10.0.21.12
PermissionGroups : AnonymousUsers, Custom
AuthMechanism    : Tls
```

Bestimmen Sie die Quell-IP des einliefernden Systems und suchen Sie den Connector, dessen `RemoteIPRanges` sie enthält. Fällt sie in keinen der eingeschränkten Connectoren, bleibt der Default-Frontend-Connector, der üblicherweise den gesamten Adressraum annimmt. Nutzen Sie auch hier `Format-List`, denn `RemoteIPRanges` und `PermissionGroups` werden in Tabellen regelmässig abgeschnitten.

Der zweite Weg ist das SMTP-Protokoll, und das verdient einen eigenen Abschnitt.

## Das SMTP-Protokoll: die einzige vollständige Quelle

Das Protokoll des Front End Transport zeichnet die vollständige SMTP-Sitzung auf: welcher Connector angesprochen wurde, welche IP verbunden hat, was Client und Server einander gesagt haben. Es ist die einzige Quelle, die das oben beschriebene Problem mit der `ConnectorId` auflöst.

### Protokollierung einschalten

Standardmässig ist die Protokollierung auf den meisten Connectoren **ausgeschaltet**. Sie schalten sie pro Connector ein:

```powershell
Set-ReceiveConnector -Identity "SRV-MAIL01\Default Frontend SRV-MAIL01" `
    -ProtocolLoggingLevel Verbose
```

Für ausgehende Verbindungen entsprechend über `Set-SendConnector`. Denken Sie daran, den Wert nach der Analyse wieder auf `None` zu setzen, denn ausführliche Protokollierung kostet Plattenplatz und schreibt bei hohem Aufkommen erhebliche Datenmengen.

### Wo die Dateien liegen

Exchange trennt die Protokolle nach Dienst und Richtung. Die Pfade fest zu verdrahten ist unnötig, fragen Sie sie ab:

```powershell
Get-FrontendTransportService SRV-MAIL01 |
    Format-List ReceiveProtocolLogPath, SendProtocolLogPath,
        ReceiveProtocolLogMaxAge, ReceiveProtocolLogMaxDirectorySize

Get-TransportService SRV-MAIL01 |
    Format-List ReceiveProtocolLogPath, SendProtocolLogPath
```

Typischerweise liegen sie unterhalb des Installationspfads in `TransportRoles\Logs\FrontEnd\ProtocolLog\SmtpReceive` für den Front End Transport und in `TransportRoles\Logs\Hub\ProtocolLog\SmtpReceive` für den Transport Service. **Das ist der Kern der Sache:** Client-Verbindungen auf Port 25 finden Sie ausschliesslich im `FrontEnd`-Pfad, im `Hub`-Pfad steht nur der interne Weiterreichungsverkehr auf 2525.

Beachten Sie die Aufbewahrung. `ReceiveProtocolLogMaxAge` steht oft auf 30 Tagen, `ReceiveProtocolLogMaxDirectorySize` begrenzt zusätzlich den Platzverbrauch. Bei hohem Aufkommen greift die Grössenbegrenzung deutlich früher als die Altersgrenze, und dann sind Ihre Protokolle nur noch wenige Tage alt.

### Das Format verstehen

Die Dateien sind CSV mit Kopfzeilen, die mit `#` beginnen. Die wichtigsten Spalten sind `date-time`, `connector-id`, `session-id`, `sequence-number`, `local-endpoint`, `remote-endpoint`, `event` und `data`.

Entscheidend ist die Spalte `event`, ein einzelnes Zeichen:

| Zeichen | Bedeutung |
|---|---|
| `+` | Verbindung aufgebaut |
| `-` | Verbindung beendet |
| `>` | Server sendet an Client |
| `<` | Client sendet an Server |
| `*` | Information des Servers, kein SMTP-Verkehr |

Eine Sitzung erkennen Sie an der gemeinsamen `session-id`; die `sequence-number` gibt die Reihenfolge innerhalb der Sitzung. Ein typischer Ausschnitt sieht so aus:

```text
2026-08-11T09:47:10.4Z,SRV-MAIL01\smtp-noauth SRV-MAIL01,08DEF44EC454A414,0,
  10.0.1.13:25,10.0.20.22:51244,+,,
2026-08-11T09:47:10.4Z,SRV-MAIL01\smtp-noauth SRV-MAIL01,08DEF44EC454A414,1,
  10.0.1.13:25,10.0.20.22:51244,>,"220 srv-mail01.intern.example.com Microsoft ESMTP",
2026-08-11T09:47:10.5Z,SRV-MAIL01\smtp-noauth SRV-MAIL01,08DEF44EC454A414,2,
  10.0.1.13:25,10.0.20.22:51244,<,EHLO app01.intern.example.com,
2026-08-11T09:47:10.6Z,SRV-MAIL01\smtp-noauth SRV-MAIL01,08DEF44EC454A414,6,
  10.0.1.13:25,10.0.20.22:51244,<,MAIL FROM:<dienst@example-test.com>,
2026-08-11T09:47:10.7Z,SRV-MAIL01\smtp-noauth SRV-MAIL01,08DEF44EC454A414,8,
  10.0.1.13:25,10.0.20.22:51244,>,"250 2.1.5 Recipient OK",
```

Hier steht alles, was im Message Tracking fehlte: der **echte** Connector (`smtp-noauth`), die **echte** Quell-IP (`10.0.20.22`) und der Name, mit dem sich das System im `EHLO` meldet.

### Gezielt suchen

Für Einzelfälle ist ein Textfilter deutlich schneller als eine Objektauswertung. Suchen Sie nach der Absenderadresse oder dem `EHLO`-Namen und lassen Sie sich die Sitzungskennung geben:

```powershell
$pfad = (Get-FrontendTransportService SRV-MAIL01).ReceiveProtocolLogPath
Select-String -Path "$pfad\*.log" -Pattern "dienst@example-test.com" -SimpleMatch |
    Select-Object -First 5 Line
```

Mit der gefundenen `session-id` holen Sie die vollständige Sitzung:

```powershell
Select-String -Path "$pfad\*.log" -Pattern "08DEF44EC454A414" -SimpleMatch |
    ForEach-Object { $_.Line } | Select-Object -First 40
```

Wollen Sie nur wissen, welche Connectoren überhaupt Verkehr sehen, zählen Sie die Verbindungsaufbauten. Das ist bei grossen Dateien um Grössenordnungen schneller, als jede Zeile zu parsen:

```powershell
Select-String -Path "$pfad\*.log" -Pattern ',\+,' |
    ForEach-Object { ($_.Line -split ',')[1] } |
    Group-Object | Sort-Object Count -Descending |
    Format-Table Count, Name -AutoSize
```

```text
Count Name
----- ----
51479 SRV-MAIL01\Default Frontend SRV-MAIL01
50756 SRV-MAIL01\smtp-auth SRV-MAIL01
19405 SRV-MAIL01\smtp-intern SRV-MAIL01
15789 SRV-MAIL01\smtp-noauth SRV-MAIL01
```

Diese Verteilung beantwortet eine Frage, die im Message Tracking nicht zu beantworten ist: Welche Wege nutzen Ihre Applikationen tatsächlich? Vor einer Connector-Umstellung ist das die wichtigste Zahl überhaupt.

### Wenn nichts protokolliert wurde

Fehlt zum fraglichen Zeitpunkt jede Zeile, gibt es drei übliche Gründe: Die Protokollierung war auf dem betreffenden Connector aus, die Aufbewahrungsgrenze hat die Datei bereits verdrängt, oder Sie schauen im falschen Pfad, also im `Hub`- statt im `FrontEnd`-Verzeichnis. Prüfen Sie in dieser Reihenfolge.

## Schritt 4: Berechtigungen prüfen

Wenn eine Einlieferung abgelehnt wird oder umgekehrt mehr erlaubt ist als gedacht, führt der Weg über die Berechtigungen des Connectors. Hier gibt es eine technische Besonderheit: `Get-ADPermission` benötigt den **DistinguishedName**. Übergeben Sie die gewohnte Identität in der Form `Server\Connectorname`, scheitert der Aufruf in einer Remote-Sitzung mit der irreführenden Meldung, das Objekt sei nicht auffindbar.

```powershell
$dn = (Get-ReceiveConnector "SRV-MAIL01\Default Frontend SRV-MAIL01").DistinguishedName
Get-ADPermission -Identity $dn -User "NT AUTHORITY\ANONYMOUS LOGON" |
    Where-Object { $_.ExtendedRights -like "*ms-Exch-SMTP*" } |
    Format-Table User, @{n='Rights'; e={$_.ExtendedRights}} -AutoSize
```

```text
User                         Rights
----                         ------
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Submit
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Accept-Any-Sender
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Accept-Authoritative-Domain-Sender
```

Die Auswertung ist einfacher, als sie aussieht, wenn Sie vier Rechte auseinanderhalten:

| Recht | Bedeutung |
|---|---|
| `ms-Exch-SMTP-Submit` | darf überhaupt einliefern |
| `ms-Exch-SMTP-Accept-Any-Sender` | darf beliebige Absenderadressen verwenden |
| `ms-Exch-SMTP-Accept-Authoritative-Domain-Sender` | darf als eigene Domäne auftreten |
| `ms-Exch-SMTP-Accept-Any-Recipient` | **darf an fremde Domänen weiterleiten** |

Die ersten drei sind der Standardsatz für anonyme Einlieferung und für den Empfang von Internet-Mail nötig. Erst das vierte Recht macht aus einem Eingangs-Connector ein Relay. Auf einem Connector, der aus dem gesamten Adressraum annimmt, ist es ein offenes Relay. Auf einem Connector mit enger IP-Beschränkung ist es dagegen der übliche und beabsichtigte Weg, damit Applikationsserver extern versenden können.

Verwechseln Sie `Accept-Any-Sender` nicht mit `Accept-Any-Recipient`. Das erste ist harmlos und notwendig, das zweite ist die sicherheitsrelevante Einstellung.

## Schritt 5: Kontrolltest durch eigene Einlieferung

Wenn die Auswertung mehrdeutig bleibt, liefern Sie selbst ein. Damit kontrollieren Sie Absender, Empfänger und Einlieferungspunkt vollständig:

```powershell
Send-MailMessage -SmtpServer 10.0.1.11 -Port 25 `
    -From 'test@example.com' `
    -To 'empfaenger@example.net' `
    -Subject 'Diagnose, bitte ignorieren' `
    -Body 'Testeinlieferung' `
    -Encoding UTF8
```

`Send-MailMessage` ist offiziell abgekündigt, für Diagnosezwecke aber weiterhin das schnellste Werkzeug und auf jedem Windows-Server vorhanden. Bei Erfolg gibt es keine Ausgabe, was gewöhnungsbedürftig ist.

Testen Sie eine TLS-Strecke auf Port 587 und die Gegenstelle präsentiert ein Zertifikat, das nicht zum verwendeten Namen passt, etwa weil Sie die IP-Adresse ansprechen, bricht der Aufruf mit einem Zertifikatsfehler ab. Für den Test können Sie die Prüfung in der Sitzung aussetzen:

```powershell
[Net.ServicePointManager]::ServerCertificateValidationCallback = { $true }
```

Das gilt nur für die laufende PowerShell-Sitzung. Setzen Sie es bewusst und nie in Skripten, die im Betrieb laufen.

Kommt die Testnachricht an und Sie wollen wissen, was unterwegs mit ihr passiert ist, hilft der [Mail-Header-Analyzer](https://rafaelpfister.ch/tools/header-analyzer): Er zerlegt die Kopfzeilen, zeichnet den Weg über die Hops und zeigt die Ergebnisse der Authentifizierungsprüfungen, komplett lokal im Browser, ohne dass die Nachricht Ihr Gerät verlässt.

## Exchange Online: dieselbe Frage, ein anderes Werkzeug

Im Tenant gelten andere Regeln, und das ist der Punkt, an dem gewohnte Vorgehensweisen scheitern. Rechnen Sie mit diesen Unterschieden:

| | Exchange OnPrem | Exchange Online |
|---|---|---|
| Abfrage | `Get-MessageTrackingLog` | `Get-MessageTraceV2` |
| Granularität | jedes Transportereignis | eine Zeile je Nachricht und Empfänger |
| Connector sichtbar | ja (mit Einschränkung, siehe oben) | **nein** |
| Serverbezug | ja, pro Server abfragen | entfällt |
| SMTP-Protokoll | vorhanden | **nicht verfügbar** |
| Aufbewahrung | Ihre Konfiguration | rund 10 Tage über das Cmdlet |
| Verzögerung | nahezu sofort | einige Minuten |

Die drei praktisch wichtigsten Konsequenzen: Es gibt **keine Connector-Zuordnung**, Sie behelfen sich mit `FromIP` und `ToIP`. Es gibt **kein SMTP-Protokoll**, die SMTP-Konversation ist nicht rekonstruierbar. Und die Daten erscheinen **verzögert**, eine gerade versendete Nachricht taucht nicht sofort auf.

### Die Basisabfrage

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-4) `
    -EndDate (Get-Date) `
    -RecipientAddress "empfaenger@example.com" `
    -ResultSize 1000 |
  Sort-Object Received |
  Format-Table Received, SenderAddress, RecipientAddress, Status, FromIP, Size -AutoSize
```

```text
Received            SenderAddress          RecipientAddress          Status    FromIP
--------            -------------          ----------------          ------    ------
11.08.2026 08:27:16 emma@partner.example   empfaenger@example.com    Delivered 10.0.20.23
11.08.2026 09:05:24 dienst@example-test.com empfaenger@example.com   Failed    10.0.20.23
```

Die wichtigsten Werte von `Status`: `Delivered`, `Failed`, `Pending`, `Quarantined`, `FilteredAsSpam` und `Expanded` für expandierte Verteiler. `Pending` heisst, dass noch Zustellversuche laufen, nicht dass etwas kaputt ist.

### Die Einzelheiten zu einer Nachricht

Der Status allein sagt nichts über den Grund. Dafür brauchen Sie die Detailansicht, und die verlangt die Nachrichtenkennung aus der Basisabfrage:

```powershell
$n = Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-4) -EndDate (Get-Date) `
        -RecipientAddress "empfaenger@example.com" -ResultSize 100 |
     Where-Object { $_.Status -eq 'Failed' } | Select-Object -First 1

Get-MessageTraceDetailV2 -MessageTraceId $n.MessageTraceId `
    -RecipientAddress $n.RecipientAddress |
  Format-List Date, Event, Action, Detail
```

Dort stehen die Verarbeitungsschritte im Dienst, etwa Regelanwendungen, Filterentscheide und der Grund einer Ablehnung.

### Über zehn Tage hinaus

Das Cmdlet reicht rund zehn Tage zurück. Für ältere Zeiträume gibt es die historische Suche, die asynchron läuft und das Ergebnis als CSV bereitstellt, mit einem Bereich von bis zu 90 Tagen:

```powershell
Start-HistoricalSearch -ReportTitle "Analyse Nachtlauf" `
    -StartDate (Get-Date).AddDays(-45) `
    -EndDate (Get-Date).AddDays(-30) `
    -ReportType MessageTrace `
    -SenderAddress "dienst@example-test.com" `
    -NotifyAddress "admin@example.com"

Get-HistoricalSearch | Format-Table JobId, ReportTitle, Status, SubmitDate -AutoSize
```

Planen Sie Zeit ein, solche Aufträge laufen je nach Umfang Stunden.

### Fehlerquelle 4: Fehlende Treffer sind kein Beweis für fehlenden Verkehr

Das ist die subtilste Fehlerquelle im Tenant. `Get-MessageTraceV2` liefert seitenweise, maximal 5000 Zeilen je Aufruf. Bei hohem Aufkommen deckt eine Seite unter Umständen nur wenige Minuten ab, obwohl Sie sieben Tage abgefragt haben. Filtern Sie danach lokal, etwa nach einer Quell-IP, dann filtern Sie über einen winzigen Ausschnitt.

Erkennbar ist das an der Warnung, die auf weitere Ergebnisse hinweist:

```text
WARNING: There are more results, use the following command to get more.
Get-MessageTraceV2 -StartDate "2026-08-11T07:25:19Z" -EndDate "2026-08-11T09:05:46Z"
  -StartingRecipientAddress "naechster@example.com" -ResultSize 5000
```

Erscheint sie, ist Ihre Auswertung **unvollständig**. Kommt kein Treffer zurück, lautet das korrekte Ergebnis: nicht im Ausschnitt gefunden. Es lautet nicht: existiert nicht.

Es gibt zwei saubere Auswege. Entweder Sie verkleinern das Zeitfenster so weit, dass eine Seite es vollständig abdeckt, erkennbar am Ausbleiben der Warnung. Oder Sie hangeln sich mit den Fortsetzungsangaben aus der Warnung durch alle Seiten. Für die Frage, ob etwas **nie** vorkommt, ist eine Konfigurationsprüfung ohnehin überlegen: Wenn ein System keine Route auf ein Ziel besitzt, kann es dorthin nicht zustellen, unabhängig von jedem Beobachtungsfenster.

Die vollständige Auswertung aller einliefernden Adressen ist ein Thema für sich, mit eigenen heiklen Punkten bei der Interpretation. Sie steht in [Wer liefert eigentlich in Ihren Tenant ein? Einliefernde IP-Adressen aggregieren](https://rafaelpfister.ch/blog/einliefernde-ip-adressen-aggregieren).

## Ein Vorgehen, das sich bewährt hat

Zusammengefasst hat sich diese Reihenfolge als schnellste erwiesen. Nachricht über alle Server suchen und das letzte Ereignis bestimmen. Bei einem Fehlschlag sofort auf `Format-List` wechseln und die vollständige SMTP-Antwort lesen, statt aus dem Ereignistyp zu schliessen. Danach den Umfang klären, also gruppieren und zählen. Erst wenn der Fall eng begrenzt ist, den Einlieferungsweg über Connector-Konfiguration und SMTP-Protokoll rekonstruieren. Und zuletzt, wenn nötig, mit einer eigenen Einlieferung gegenprüfen.

Die häufigsten Zeitfresser sind dagegen immer dieselben: Man liest eine abgeschnittene Tabelle statt der vollständigen Fehlermeldung, man hält Schattenkopien für Verarbeitungsschritte, man glaubt der `ConnectorId` im Tracking, und man hält eine leere Stichprobe für einen Beweis. Wer diese vier kennt, kommt in der Regel in wenigen Minuten zur richtigen Ebene.

## Quellen

1.  [Message tracking in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): Feldbeschreibung und vollständige Liste der Ereignistypen im Message Tracking.

2.  [Protocol logging in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/protocol-logging): Speicherorte, Format und Aufbewahrung der SMTP-Protokolle, inklusive Front End Transport.

3.  [Shadow redundancy in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/shadow-redundancy): erklärt die Ereignisse rund um Schattenkopien und deren Verwerfen.

4.  [Mail routing in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing): Zusammenspiel von Front End Transport und Transport Service, Grundlage des Proxy-Verhaltens.

5.  [Receive connectors in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/receive-connectors): Bindungen, Berechtigungsgruppen und Authentifizierungsmechanismen.

6.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2): Nachfolger von Get-MessageTrace inklusive Seitenlogik und Feldliste.

7.  [Start-HistoricalSearch](https://learn.microsoft.com/en-us/powershell/module/exchange/start-historicalsearch): asynchrone Nachrichtenverfolgung über bis zu 90 Tage.
