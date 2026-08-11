---
title: "Exchange-Mailfluss analysieren: Message Tracking und Receive-Connectoren"
navTitle: "Mailfluss analysieren"
description: "Wie Sie in Exchange OnPrem und Hybrid systematisch herausfinden, wo eine Nachricht geblieben ist: die Abfragen, die Reihenfolge, in der sie sinnvoll sind, und die vier Fallstricke, die regelmässig auf falsche Fährten führen."
date: "2026-08-11"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "14 Min. Lesezeit"
themen:
  - "exchange-onprem-hybrid"
  - "smtp-mailflow"
  - "microsoft-365-exchange"
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange"
  - "exchange-on-premises"
  - "exchange-hybrid"
protokolle:
  - "smtp"
  - "powershell"
  - "troubleshooting"
related:
  - "typische-ursachen-fuer-mail-loops-und-deren-behebung"
  - "ghost-sender-exchange-online-nebeneingang"
  - "mailflow-fehlersuche-kontrollierte-experimente"
slug: "exchange-message-tracking-und-receive-connectoren-analysieren"
translationId: "article-ad93c41ab6cd20e6"
url: "https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren"
draft: false
---

# Exchange-Mailfluss analysieren: Message Tracking und Receive-Connectoren

Die häufigste Frage im Mailbetrieb lautet: Eine Nachricht ist nicht angekommen, wo ist sie geblieben? Das Message Tracking beantwortet das zuverlässig, aber nur, wenn Sie wissen, was es Ihnen nicht sagt. Dieser Artikel beschreibt das Vorgehen in der Reihenfolge, in der es sich bewährt hat, und benennt vier Fallstricke, die regelmässig Stunden kosten, weil sie plausible, aber falsche Schlüsse nahelegen.

Alle Beispiele verwenden generische Namen: `SRV-MAIL01` und `SRV-MAIL02` als Transportserver, `example.com` als Domäne.

## Der Grundsatz: erst lokalisieren, dann erklären

Der Reflex ist, sofort nach der Ursache zu suchen. Effizienter ist es, zuerst zu bestimmen, wie weit die Nachricht überhaupt gekommen ist. Das grenzt den Suchraum in einem Schritt drastisch ein, denn danach wissen Sie, ob Sie im eigenen System, beim vorgelagerten Gateway oder beim Ziel suchen müssen.

Die Reihenfolge lautet deshalb: Nachricht finden, letztes Ereignis lesen, Fehlergrund lesen, Einzelfall oder Muster bestimmen, und erst dann den Einlieferungsweg rekonstruieren.

## Schritt 1: Die Nachricht finden

Beginnen Sie über den Empfänger, denn den kennen Sie fast immer. Wichtig ist, die Abfrage über **alle** Transportserver laufen zu lassen, nicht nur über einen.

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object { Get-MessageTrackingLog -Server $_ -Start (Get-Date).AddHours(-6) -ResultSize Unlimited -Recipients "empfaenger@example.com" } | Sort-Object Timestamp | Format-Table Timestamp,ServerHostname,EventId,Source,ConnectorId,MessageId -AutoSize -Wrap
```

Findet die Abfrage nichts, prüfen Sie, ob der Empfänger über einen Verteiler expandiert wurde. Dann suchen Sie besser über den Absender:

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object { Get-MessageTrackingLog -Server $_ -Start (Get-Date).AddHours(-6) -ResultSize Unlimited } | Where-Object { $_.Sender -like "*@example.com" } | Sort-Object Timestamp | Format-Table Timestamp,EventId,Sender,@{n='To';e={$_.Recipients -join ','}},MessageSubject -AutoSize -Wrap
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

Steckt die Nachricht fest, zeigt die Queue den Grund meist im Klartext:

```powershell
Get-Queue -Server SRV-MAIL01 | Where-Object { $_.MessageCount -gt 0 } | Format-Table Identity,DeliveryType,Status,MessageCount,NextHopDomain,LastError -AutoSize -Wrap
```

## Fallstrick 1: Tracking ist serverbezogen, und die Hälfte der Einträge sind Schattenkopien

Wenn Sie in der Ausgabe Paare aus `HARECEIVE` und `HADISCARD` sehen, oft mit dem Zusatz `ExplicitlyDiscarded`, dann hat dieser Server die Nachricht **nicht verarbeitet**. Er hielt nur eine Schattenkopie im Rahmen der Shadow Redundancy, während ein anderer Server die eigentliche Zustellung übernahm. Sobald der primäre Server Erfolg meldet, verwirft der Partner seine Kopie.

Praktisch heisst das zweierlei. Erstens sind solche Zeilen kein Hinweis auf ein Problem, sondern Normalbetrieb. Zweitens müssen Sie zwingend alle Transportserver abfragen, sonst sehen Sie zufällig nur die Schattenseite eines Vorgangs und schliessen daraus auf einen Fehler, den es nicht gibt.

## Fallstrick 2: `Format-Table` schneidet genau die entscheidenden Spalten ab

Der Fehlergrund steht in `RecipientStatus`, und dieses Feld ist lang. In einer Tabelle fällt es entweder ganz weg oder wird abgeschnitten. Genau das führt dazu, dass man den `FAIL` sieht, aber nicht den Grund, und stattdessen zu raten beginnt.

Sobald Sie einen Fehlerfall gefunden haben, wechseln Sie deshalb auf `Format-List` und lösen die Sammelfelder auf:

```powershell
Get-MessageTrackingLog -Server SRV-MAIL01 -Start (Get-Date).AddHours(-6) -ResultSize Unlimited -Recipients "empfaenger@example.com" -EventId FAIL | Format-List Timestamp,Sender,@{n='To';e={$_.Recipients -join ','}},@{n='Status';e={$_.RecipientStatus -join ' | '}},MessageSubject,MessageId,SourceContext
```

`RecipientStatus` enthält die vollständige SMTP-Antwort der Gegenstelle samt deren Hostnamen. In den meisten Fällen ist die Diagnose damit bereits erledigt, weil das Zielsystem im Klartext sagt, warum es abgelehnt hat.

## Schritt 3: Einzelfall oder Muster?

Bevor Sie sich in einen einzelnen Fall vertiefen, klären Sie den Umfang. Diese eine Abfrage entscheidet, ob Sie es mit einer Randnotiz oder mit einem Vorfall zu tun haben:

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object { Get-MessageTrackingLog -Server $_ -Start (Get-Date).AddHours(-8) -EventId FAIL -ResultSize Unlimited } | Where-Object { ($_.RecipientStatus -join '') -like "*5.1.8*" } | Group-Object { ($_.Sender -split '@')[-1] } | Sort-Object Count -Descending | Format-Table Count,Name -AutoSize
```

Ersetzen Sie `5.1.8` durch den Statuscode, den Sie untersuchen. Steht dort eine einzige Absenderdomäne, ist das Problem eng begrenzt. Stehen dort viele, haben Sie einen Vorfall, und alles andere hat Vorrang. Diese Unterscheidung so früh zu treffen, spart erfahrungsgemäss am meisten Zeit.

## Fallstrick 3: Die `ConnectorId` verrät nicht den echten Receive-Connector

Das ist der teuerste Fallstrick, weil die Ausgabe seriös aussieht. Mail, die ein Client oder ein Fremdsystem auf Port 25 einliefert, trifft zuerst den **Front End Transport**. Dieser reicht die Nachricht an den **Transport Service** auf Port 2525 weiter. Das Message Tracking wird erst dort geschrieben, der Front End Transport schreibt kein eigenes Tracking.

Die Folge: Im Tracking steht als `ConnectorId` praktisch immer `Default <Servername>`, also der interne Connector auf 2525, und als `ClientIp` die Adresse des **proxyenden Servers**, nicht die des ursprünglichen Einlieferers. Welchen der konfigurierten Connectoren auf Port 25 ein System tatsächlich getroffen hat, steht im Tracking schlicht nicht drin.

Es gibt zwei Wege zur Antwort. Der erste ist die Rekonstruktion über die Konfiguration:

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object { Get-ReceiveConnector -Server $_ } | Format-List Identity,Enabled,@{n='Bindings';e={$_.Bindings -join ','}},@{n='RemoteIPRanges';e={$_.RemoteIPRanges -join ','}},PermissionGroups,AuthMechanism
```

Bestimmen Sie die Quell-IP des einliefernden Systems und suchen Sie den Connector, dessen `RemoteIPRanges` sie enthält. Fällt sie in keinen der eingeschränkten Connectoren, bleibt der Default-Frontend-Connector, der üblicherweise den gesamten Adressraum annimmt. Nutzen Sie auch hier `Format-List`, denn `RemoteIPRanges` und `PermissionGroups` werden in Tabellen regelmässig abgeschnitten.

Der zweite Weg ist das Protokoll des Front End Transport. Nur dort steht die tatsächliche SMTP-Sitzung samt Connector. Die Dateien liegen unterhalb des Exchange-Installationspfads in `TransportRoles\Logs\FrontEnd\ProtocolLog\SmtpReceive`. Für gezielte Suchen ist ein Textfilter deutlich schneller als eine Objektauswertung über PowerShell:

```powershell
Select-String -Path "$env:ExchangeInstallPath\TransportRoles\Logs\FrontEnd\ProtocolLog\SmtpReceive\*.log" -Pattern "absender@example.com" -SimpleMatch | Select-Object -First 20
```

Wollen Sie nur wissen, welche Connectoren überhaupt Verkehr sehen, zählen Sie die Sitzungen statt sie einzeln auszuwerten. Das ist bei grossen Protokolldateien um Grössenordnungen schneller.

## Schritt 4: Berechtigungen prüfen

Wenn eine Einlieferung abgelehnt wird oder umgekehrt mehr erlaubt ist als gedacht, führt der Weg über die Berechtigungen des Connectors. Hier lauert ein technischer Fallstrick: `Get-ADPermission` benötigt den **DistinguishedName**. Übergeben Sie die gewohnte Identität in der Form `Server\Connectorname`, scheitert der Aufruf in einer Remote-Sitzung mit einer irreführenden Meldung, das Objekt sei nicht auffindbar.

```powershell
$dn = (Get-ReceiveConnector "SRV-MAIL01\Default Frontend SRV-MAIL01").DistinguishedName; Get-ADPermission -Identity $dn -User "NT AUTHORITY\ANONYMOUS LOGON" | Where-Object { $_.ExtendedRights -like "*ms-Exch-SMTP*" } | Format-Table User,@{n='Rights';e={$_.ExtendedRights}} -AutoSize
```

Die Auswertung ist einfacher, als sie aussieht, wenn Sie zwei Rechte auseinanderhalten:

| Recht | Bedeutung |
|---|---|
| `ms-Exch-SMTP-Submit` | darf überhaupt einliefern |
| `ms-Exch-SMTP-Accept-Any-Sender` | darf beliebige Absenderadressen verwenden |
| `ms-Exch-SMTP-Accept-Authoritative-Domain-Sender` | darf als eigene Domäne auftreten |
| `ms-Exch-SMTP-Accept-Any-Recipient` | **darf an fremde Domänen weiterleiten** |

Die ersten drei sind der Standardsatz für anonyme Einlieferung und für den Empfang von Internet-Mail nötig. Erst das vierte Recht macht aus einem Eingangs-Connector ein Relay. Auf einem Connector, der aus dem gesamten Adressraum annimmt, ist es ein offenes Relay. Auf einem Connector mit enger IP-Beschränkung ist es dagegen der übliche und beabsichtigte Weg, damit Applikationsserver extern versenden können.

Verwechseln Sie `Accept-Any-Sender` nicht mit `Accept-Any-Recipient`. Das erste ist harmlos und notwendig, das zweite ist die sicherheitsrelevante Einstellung.

## Schritt 5: Gegenprobe durch eigene Einlieferung

Wenn die Auswertung mehrdeutig bleibt, liefern Sie selbst ein. Damit kontrollieren Sie Absender, Empfänger und Einlieferungspunkt vollständig:

```powershell
Send-MailMessage -SmtpServer 10.0.0.10 -Port 25 -From 'test@example.com' -To 'empfaenger@example.net' -Subject 'Diagnose, bitte ignorieren' -Body 'Testeinlieferung' -Encoding UTF8
```

`Send-MailMessage` ist offiziell abgekündigt, für Diagnosezwecke aber weiterhin das schnellste Werkzeug und auf jedem Windows-Server vorhanden. Bei Erfolg gibt es keine Ausgabe, was gewöhnungsbedürftig ist.

Testen Sie eine TLS-Strecke auf Port 587 und die Gegenstelle präsentiert ein Zertifikat, das nicht zum verwendeten Namen passt, etwa weil Sie die IP-Adresse ansprechen, bricht der Aufruf mit einem Zertifikatsfehler ab. Für den Test können Sie die Prüfung in der Sitzung aussetzen:

```powershell
[Net.ServicePointManager]::ServerCertificateValidationCallback = { $true }
```

Das gilt nur für die laufende PowerShell-Sitzung. Setzen Sie es bewusst und nie in Skripten, die im Betrieb laufen.

## Fallstrick 4: Fehlende Treffer sind kein Beweis für fehlenden Verkehr

In Exchange Online ist `Get-MessageTrace` abgekündigt und durch `Get-MessageTraceV2` ersetzt. Beide liefern seitenweise. Bei hohem Aufkommen deckt eine Seite unter Umständen nur wenige Minuten ab, obwohl Sie sieben Tage abgefragt haben. Filtern Sie danach lokal, etwa nach einer Quell-IP, dann filtern Sie über einen winzigen Ausschnitt.

Kommt nichts zurück, beweist das **nicht**, dass es diesen Verkehr nicht gab. Es beweist nur, dass er nicht im Ausschnitt lag. Achten Sie auf die Warnung über weitere Ergebnisse: Erscheint sie, ist Ihre Auswertung unvollständig.

Es gibt zwei saubere Auswege. Entweder Sie verkleinern das Zeitfenster so weit, dass eine Seite es vollständig abdeckt, und gruppieren dann serverseitig vollständig:

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-2) -EndDate (Get-Date) -ResultSize 5000 | Group-Object FromIP | Sort-Object Count -Descending | Format-Table Count,Name -AutoSize
```

Oder Sie nehmen einen Bericht, der serverseitig aggregiert, etwa den Connector-Bericht im Exchange Admin Center unter Berichte und Mailfluss. Für die Frage, ob ein Connector noch genutzt wird, ist der Bericht dem Message Trace klar überlegen.

## Ein Vorgehen, das sich bewährt hat

Zusammengefasst hat sich diese Reihenfolge als schnellste erwiesen. Nachricht über alle Server suchen und das letzte Ereignis bestimmen. Bei einem Fehlschlag sofort auf `Format-List` wechseln und die vollständige SMTP-Antwort lesen, statt aus dem Ereignistyp zu schliessen. Danach den Umfang klären, also gruppieren und zählen. Erst wenn der Fall eng begrenzt ist, den Einlieferungsweg über Connector-Konfiguration und Frontend-Protokoll rekonstruieren. Und zuletzt, wenn nötig, mit einer eigenen Einlieferung gegenprüfen.

Die drei häufigsten Zeitfresser sind dagegen immer dieselben: Man liest eine abgeschnittene Tabelle statt der vollständigen Fehlermeldung, man hält Schattenkopien für Verarbeitungsschritte, und man glaubt der `ConnectorId` im Tracking. Wer diese drei kennt, kommt in der Regel in wenigen Minuten zur richtigen Ebene.

## Quellen

1.  [Message tracking in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): Feldbeschreibung und vollständige Liste der Ereignistypen im Message Tracking.

2.  [Protocol logging in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/protocol-logging): Speicherorte und Format der SMTP-Protokolle, inklusive Front End Transport.

3.  [Shadow redundancy in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/shadow-redundancy): erklärt die Ereignisse rund um Schattenkopien und deren Verwerfen.

4.  [Mail routing in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing): Zusammenspiel von Front End Transport und Transport Service, Grundlage des Proxy-Verhaltens.

5.  [Receive connectors in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/receive-connectors): Bindungen, Berechtigungsgruppen und Authentifizierungsmechanismen.

6.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2): Nachfolger von Get-MessageTrace inklusive Seitenlogik.
