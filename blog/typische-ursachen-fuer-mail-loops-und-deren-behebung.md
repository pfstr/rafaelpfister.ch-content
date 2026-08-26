---
title: "Typische Ursachen für Mail-Loops und deren Behebung"
navTitle: "Mail-Loops beheben"
description: "Wie sich SMTP-Mail-Loops in Exchange Online, Hybrid-Umgebungen und vorgeschalteten Mailgateways anhand von NDR, Headern, Message Trace, Empfängerobjekten und Connectors systematisch finden und beheben lassen."
date: "2026-08-07"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "12 Min. Lesezeit"
themen:
  - "exchange-onprem-hybrid"
  - "microsoft-365-exchange"
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange"
protokolle:
  - "smtp"
  - "powershell"
  - "troubleshooting"
related:
  - "exchange-mailboxen-ohne-remote-move-cloud-neu-erstellen"
  - "totemomail-m365"
  - "ghost-sender-exchange-online-nebeneingang"
slug: "typische-ursachen-fuer-mail-loops-und-deren-behebung"
translationId: "article-4c91e7b2a8605fd3"
url: "https://rafaelpfister.ch/blog/typische-ursachen-fuer-mail-loops-und-deren-behebung"
draft: false
---

# Typische Ursachen für Mail-Loops und deren Behebung

Ein Mail-Loop entsteht, wenn mindestens zwei Transportsysteme dieselbe Nachricht immer wieder aneinander übergeben. Keines der Systeme erkennt sich als endgültiges Ziel, beide kennen aber einen vermeintlich passenden nächsten Hop. Die Schleife endet erst, wenn ein Server die zulässige Anzahl Transportstationen überschritten sieht und einen NDR erzeugt.

Bei Exchange sind zwei Meldungen besonders aussagekräftig:

- `554 5.4.6 Hop count exceeded - possible mail loop` wird typischerweise vom lokalen Exchange erzeugt.
- `554 5.4.14 Hop count exceeded - possible mail loop ATTR34` wird von Exchange Online erzeugt.

Die Hop-Grenze ist nicht die Ursache, sondern die Sicherung gegen eine endlose Wiederholung. Sie zu erhöhen, behebt deshalb nichts. Gesucht wird der Punkt, an dem die Nachricht entgegen der Zielarchitektur an ein bereits durchlaufenes System zurückgegeben wird.

## Das Schleifenmuster im Header erkennen

Der NDR und die vollständigen ursprünglichen Nachrichtenköpfe sollten vor jeder Änderung gesichert werden. `Received`-Zeilen werden von unten nach oben gelesen: Die unterste Zeile ist der früheste dokumentierte Hop, die oberste der jüngste.

Eine Schleife zeigt sich meist als wiederkehrende Sequenz:

```text
Internet
  → Exchange Online Protection
  → Mailgateway
  → Exchange Online Protection
  → Mailgateway
  → Exchange Online Protection
  → ...
```

Nicht jeder mehrfach auftauchende Microsoft-Hostname ist bereits eine Schleife. Exchange Online verarbeitet Nachrichten intern über mehrere Transportrollen. Auffällig ist die wiederholte Rückkehr zwischen denselben administrativen Grenzen, beispielsweise zwischen Exchange Online und einem lokalen Gateway. Zeitstempel, sendende IP, empfangender Host und `Message-ID` helfen, die Runde eindeutig zu erkennen.

Für die erste Analyse werden diese Fragen beantwortet:

1. Welches System hat den NDR erzeugt?
2. Welche zwei oder drei Hops wiederholen sich?
3. Welches System hätte die Nachricht endgültig zustellen sollen?
4. Aufgrund welcher Domain-, Empfänger-, Connector- oder Regelentscheidung wurde sie weitergeleitet?
5. Welche Änderung hat den Mailflow zuletzt beeinflusst?

## Diagnose in Exchange Online

Mit `Get-MessageTraceV2` lässt sich die Verarbeitung der letzten 90 Tage untersuchen; pro Abfrage sind höchstens zehn Tage zulässig. Ein enges Zeitfenster und die konkrete Empfängeradresse liefern die brauchbarsten Ergebnisse:

```powershell
$start = (Get-Date).AddHours(-2)
$end = Get-Date
$recipient = "user01@contoso.com"

$trace = Get-MessageTraceV2 `
    -RecipientAddress $recipient `
    -StartDate $start `
    -EndDate $end `
    -ResultSize 5000

$trace |
    Select-Object Received,SenderAddress,RecipientAddress,Subject,
        Status,FromIP,ToIP,MessageTraceId,MessageId |
    Sort-Object Received
```

Die Details eines Treffers zeigen einzelne Transportereignisse:

```powershell
$trace | ForEach-Object {
    Get-MessageTraceDetailV2 `
        -MessageTraceId $_.MessageTraceId `
        -RecipientAddress $_.RecipientAddress
} | Format-Table Date,Event,Action,Detail -AutoSize
```

Danach werden Domain, Empfänger und Connectoren gemeinsam aufgenommen:

```powershell
Get-AcceptedDomain |
    Format-Table Name,DomainName,DomainType,MatchSubDomains -AutoSize

Get-EXORecipient -Identity $recipient |
    Format-List DisplayName,RecipientTypeDetails,PrimarySmtpAddress,
        ExternalEmailAddress,EmailAddresses

Get-OutboundConnector -IncludeTestModeConnectors |
    Format-List Name,Enabled,ConnectorType,RecipientDomains,SmartHosts,
        UseMXRecord,RouteAllMessagesViaOnPremises,TlsSettings

Get-InboundConnector |
    Format-List Name,Enabled,ConnectorType,SenderDomains,SenderIPAddresses,
        TlsSenderCertificateName,RequireTls,RestrictDomainsToIPAddresses,
        RestrictDomainsToCertificate
```

Entscheidend ist nicht, ob ein einzelnes Objekt plausibel aussieht. Domain-Typ, tatsächlicher Empfängertyp und anwendbarer Connector müssen zusammen denselben Zielort beschreiben.

## Diagnose im lokalen Exchange

In einer Hybrid-Umgebung wird derselbe Empfänger auch lokal geprüft. Die Abfragen unterscheiden zwischen einem echten lokalen Postfach, einer RemoteMailbox und einem MailUser:

```powershell
Get-Recipient -Identity $recipient |
    Format-List DisplayName,RecipientType,RecipientTypeDetails,
        PrimarySmtpAddress,EmailAddresses

Get-Mailbox -Identity $recipient -ErrorAction SilentlyContinue |
    Format-List RecipientTypeDetails,ServerName,Database,PrimarySmtpAddress

Get-RemoteMailbox -Identity $recipient -ErrorAction SilentlyContinue |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,RemoteRoutingAddress

Get-MailUser -Identity $recipient -ErrorAction SilentlyContinue |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,ExternalEmailAddress
```

Für den Transportpfad werden Send- und Receive-Connectoren sowie die Tracking-Logs benötigt:

```powershell
Get-SendConnector |
    Format-List Name,Enabled,AddressSpaces,DNSRoutingEnabled,SmartHosts,
        SourceTransportServers,CloudServicesMailEnabled,TlsDomain

Get-ReceiveConnector |
    Format-List Identity,Enabled,Bindings,RemoteIPRanges,PermissionGroups

$servers = Get-ExchangeServer |
    Where-Object { $_.IsMailboxServer -or $_.IsHubTransportServer }

$servers |
    Get-MessageTrackingLog `
        -Start $start `
        -End $end `
        -Recipients $recipient `
        -ResultSize Unlimited |
    Select-Object Timestamp,ServerHostname,ClientHostname,Source,EventId,
        ConnectorId,Sender,Recipients,MessageId,NetworkMessageId |
    Sort-Object Timestamp
```

Ein `SEND` zu Exchange Online, gefolgt von einem erneuten `RECEIVE` derselben Nachricht aus Exchange Online, macht die Rückgabe sichtbar. Mit `MessageId` und `NetworkMessageId` lässt sich vermeiden, verschiedene Testnachrichten miteinander zu verwechseln.

## Die häufigsten Ursachen im Überblick

| Muster | Typische Ursache | Behebung |
| --- | --- | --- |
| Unbekannte Empfänger pendeln zwischen zwei Systemen | Accepted Domain steht auf `InternalRelay`, aber beide Seiten leiten unbekannte Empfänger weiter | Eindeutige Zuständigkeit definieren; bei vollständiger EXO-Zustellung `Authoritative` verwenden oder für Split-Domain einen einzigen abschliessenden Hop festlegen |
| EXO sendet zum lokalen Exchange, dieser sofort zurück zu EXO | Hybrid-Connector oder Centralized Mail Transport passt nicht mehr zur Mailbox-Lokation | HCW-Konfiguration und `RouteAllMessagesViaOnPremises` prüfen; veraltete Zentralroute deaktivieren oder lokale Empfängerauflösung korrigieren |
| Nachricht pendelt zwischen EXO und einem Security-, Signatur- oder Verschlüsselungsgateway | Rückkehrende Nachrichten erfüllen erneut die Ausgangsregel | Vom Gateway gesetzten Header beziehungsweise dokumentierten Loop-Prevention-Mechanismus als Ausnahme verwenden; Ein- und Ausgangsconnector eindeutig authentisieren |
| Nur ein Empfänger ist betroffen | Veraltetes oder falsches `targetAddress`, falscher RemoteMailbox-Typ oder widersprüchliche Proxy-Adressen | Source of Authority bestimmen, Empfängerobjekt dort korrigieren und synchronisieren |
| Nur weitergeleitete Nachrichten laufen | Transportregel, Mailbox-Weiterleitung oder Inbox-Regel adressiert den ursprünglichen Pfad erneut | Regel deaktivieren, Ziel korrigieren und eine belastbare Ausnahme definieren |
| Nur eine Subdomain oder Anwendung ist betroffen | Übergeordnete Domain deckt die Subdomain im erwarteten Connectorpfad nicht korrekt ab | Subdomain explizit als Accepted Domain und im passenden Send Connector konfigurieren |
| Alle Nachrichten laufen nach einer Gateway- oder DNS-Änderung | Smart Host oder MX zeigt auf den Eingang des sendenden Systems | Next Hop korrigieren und DNS-, NAT- sowie Load-Balancer-Ziele getrennt prüfen |

## Ursache 1: Falscher Typ der Accepted Domain

Eine authoritative Domain bedeutet: Alle gültigen Empfänger dieser Domain sind in der Exchange-Organisation bekannt; unbekannte Empfänger werden abgewiesen. Eine Internal-Relay-Domain bedeutet: Ein Teil der Empfänger liegt in einem anderen System und muss über einen Send- oder Outbound-Connector weitergeleitet werden.

Die problematische Konstellation entsteht, wenn Exchange Online unbekannte Empfänger an ein lokales System sendet und dieses dieselbe Domain ebenfalls nicht abschliessend behandelt, sondern per MX oder Smart Host wieder an Exchange Online zurückgibt.

```powershell
Get-AcceptedDomain -Identity contoso.com |
    Format-List DomainName,DomainType,MatchSubDomains
```

Wenn nach Abschluss einer Migration alle Empfänger in Exchange Online liegen, ist `Authoritative` meist der richtige Zielzustand:

```powershell
# Erst nach vollständiger Empfänger- und Routingprüfung ausführen.
Set-AcceptedDomain -Identity contoso.com -DomainType Authoritative
```

Bei einer echten Split-Domain darf `InternalRelay` korrekt sein. Dann braucht es jedoch einen klaren Connector zu dem System, das die verbleibenden Empfänger kennt. Dieses Ziel darf unbekannte Adressen nicht wieder an den Ausgangspunkt zurücksenden.

## Ursache 2: Überlappende Hybrid-Connectoren und Centralized Mail Transport

Centralized Mail Transport leitet ausgehende Exchange-Online-Nachrichten bewusst über den lokalen Exchange. Das ist für bestimmte Compliance-Anforderungen sinnvoll, erzeugt aber zusätzliche Transportwege. Bleibt die Option nach einer Migration aktiv, obwohl das lokale System Nachrichten über den eigenen MX wieder an Exchange Online sendet, kann ein Kreis entstehen.

```powershell
Get-OutboundConnector -IncludeTestModeConnectors |
    Format-Table Name,Enabled,ConnectorType,RouteAllMessagesViaOnPremises,
        RecipientDomains,SmartHosts,UseMXRecord -AutoSize
```

Mehrere Connectoren mit überlappendem Scope sollten ebenfalls geprüft werden. Microsoft empfiehlt für Hybrid-Mailflow einen dedizierten On-Premises-Connector; eine Reparatur über den Hybrid Configuration Wizard ist häufig sicherer als isolierte Einzeländerungen.

Wenn Centralized Mail Transport nachweislich nicht mehr benötigt wird, kann die Einstellung gezielt deaktiviert werden:

```powershell
# Nur nach Prüfung der Compliance- und Gateway-Anforderungen.
Set-OutboundConnector `
    -Identity "Outbound to On-Premises" `
    -RouteAllMessagesViaOnPremises:$false
```

## Ursache 3: Ein Gateway verarbeitet seine Rückläufer erneut

Bei einem In-and-out-Szenario sendet Exchange Online eine Nachricht zur Signatur, Verschlüsselung oder Archivierung an einen Zusatzdienst. Dieser gibt sie anschliessend an Exchange Online zurück. Die Ausgangsregel muss den Rückläufer erkennen; sonst wird er erneut zum Dienst geschickt.

Die Prüfung beginnt bei allen Regeln, die Connectoren auswählen, Empfänger umleiten oder Header auswerten:

```powershell
Get-TransportRule |
    Sort-Object Priority |
    Format-List Name,State,Mode,Priority,FromScope,SentToScope,
        RedirectMessageTo,RouteMessageOutboundConnector,
        SetHeaderName,SetHeaderValue,ExceptIfHeaderContainsMessageHeader,
        ExceptIfHeaderContainsWords
```

Die konkrete Ausnahme muss der Dokumentation des Gateway-Herstellers folgen. Üblich ist ein vom Dienst gesetzter, nicht vom Internet vertrauenswürdig fälschbarer Header. Zusätzlich sollten Inbound-Connectoren den Dienst über Zertifikat oder feste Absender-IP identifizieren. Eine pauschale Ausnahme für alle «intern» erscheinenden Nachrichten ist zu breit.

## Ursache 4: Empfängerobjekt und tatsächliche Mailbox liegen nicht am selben Ort

Ein Objekt kann in Exchange Online als `MailUser` erscheinen, obwohl das aktive Postfach lokal liegt. Das ist in einer synchronisierten Hybrid-Umgebung nicht automatisch ein Duplikat. Auch eine `ExternalEmailAddress`, die der primären SMTP-Adresse entspricht, beweist für sich allein noch keine Fehlkonfiguration.

Massgeblich ist die Kombination aller Abfragen:

- `Get-Mailbox` lokal liefert ein Ergebnis: Das aktive Postfach liegt lokal.
- `Get-RemoteMailbox` lokal liefert ein Ergebnis: Das verwaltete Ziel liegt in Exchange Online.
- `Get-EXOMailbox` liefert ein Ergebnis: In der Cloud existiert ein echtes Postfach.
- `Get-EXORecipient` liefert nur einen MailUser: Das Objekt ist ein Routingziel, keine Cloud-Mailbox.

Problematisch sind veraltete Objekte nach einer Migration, falsche Remote-Routing-Domains oder manuell gesetzte `targetAddress`-Werte, deren Domain über denselben Transportweg wieder zurückführt. Änderungen erfolgen am Source of Authority: In synchronisierten Umgebungen also mit Exchange-Managementwerkzeugen lokal und nicht durch direktes Editieren einzelner Attribute in Exchange Online.

## Ursache 5: Weiterleitungen und Transportregeln bilden einen Kreis

Eine Regel kann von Adresse A nach B umleiten, während B über eine zweite Regel, eine Mailbox-Weiterleitung oder ein externes System wieder an A sendet. Solche Schleifen betreffen oft nur einzelne Empfänger oder Nachrichtentypen.

```powershell
Get-TransportRule |
    Sort-Object Priority |
    Select-Object Name,State,Mode,Priority,RedirectMessageTo,
        BlindCopyTo,AddToRecipients,RouteMessageOutboundConnector

Get-Mailbox -ResultSize Unlimited |
    Select-Object DisplayName,PrimarySmtpAddress,
        ForwardingAddress,ForwardingSmtpAddress,DeliverToMailboxAndForward

Get-InboxRule -Mailbox user01@contoso.com |
    Select-Object Name,Enabled,Priority,ForwardTo,RedirectTo,ForwardAsAttachmentTo
```

Die Behebung besteht nicht nur darin, eine Regel kurzfristig abzuschalten. Die vollständige Kette muss aufgelöst werden, und Regeln für externe Dienste benötigen eine Ausnahme, die bereits verarbeitete Nachrichten sicher erkennt.

## Ursache 6: MX, Smart Host oder Subdomain zeigt zurück

Ein Gateway kann intern einen anderen Next Hop benötigen als externe Absender. Verwendet es für die Weiterleitung einfach den öffentlichen MX, zeigt dieser unter Umständen wieder auf das Gateway selbst. Dasselbe Problem entsteht, wenn ein Smart Host durch NAT oder Load Balancing auf den eigenen Listener zurückführt.

```powershell
Resolve-DnsName -Type MX contoso.com
Resolve-DnsName -Type MX app.contoso.com

Get-SendConnector |
    Format-List Name,AddressSpaces,DNSRoutingEnabled,SmartHosts
```

Subdomains verdienen eine eigene Prüfung. Microsoft dokumentiert Fälle, in denen eine Anwendungs-Subdomain explizit als Internal-Relay-Domain angelegt und zu den Edge-Systemen synchronisiert werden muss:

```powershell
New-AcceptedDomain `
    -Name "app.contoso.com" `
    -DomainName app.contoso.com `
    -DomainType InternalRelay

Start-EdgeSynchronization
```

Diese Befehle sind kein universeller Fix. Sie passen nur, wenn `app.contoso.com` tatsächlich ausserhalb der Exchange-Organisation zugestellt wird und der Send Connector einen eindeutigen nächsten Hop besitzt.

## Sicheres Vorgehen bei einer aktiven Schleife

Während der Störung sollte zuerst die Vervielfachung gestoppt werden. Je nach Architektur wird die auslösende Transportregel oder der spezifische Connector kontrolliert deaktiviert, oder das Gateway hält die betroffene Queue zurück. Vorher werden Konfiguration und Nachrichtenbeispiele exportiert.

Danach folgt ein Test mit genau einem Absender, einem Empfänger und einer eindeutig erkennbaren Betreffzeile. Die Nachricht wird über Header, Message Trace und lokale Tracking-Logs lückenlos verfolgt. Erst wenn sie am vorgesehenen Ziel endet, wird der Mailflow schrittweise wieder geöffnet.

Nicht empfehlenswert sind:

- Hop-Limits erhöhen
- mehrere Connectoren gleichzeitig ändern
- Accepted Domains auf Verdacht zwischen `Authoritative` und `InternalRelay` umschalten
- eine problematische Queue ungeprüft wiederholt einspeisen
- synchronisierte Exchange-Attribute direkt in AD oder Exchange Online korrigieren
- TLS-, IP- oder Zertifikatsprüfungen als vermeintlichen Schnellfix abschalten

## Abschlusskontrolle

Nach der Korrektur sollte die Dokumentation für jede relevante Domain genau eine Aussage enthalten: Welches System kennt den Empfänger, welcher Connector ist anwendbar, und welcher Host ist der endgültige nächste Hop?

Die technische Abnahme umfasst mindestens:

- externe und interne Testnachricht
- unbekannter Empfänger derselben Domain
- Empfänger auf jeder Seite einer echten Split-Domain
- ausgehende Nachricht bei aktiviertem Gateway oder Centralized Mail Transport
- Header ohne wiederkehrende Hop-Sequenz
- Message Trace mit `Delivered` beziehungsweise erwarteter Übergabe
- lokales Tracking ohne erneutes `RECEIVE` nach einem `SEND` zum selben Ziel
- Connector-Validierung für alle weiterhin benötigten Connectoren

Ein behobener Mail-Loop ist erst dann abgeschlossen, wenn nicht nur die Testmail ankommt, sondern auch unbekannte Empfänger und alternative Mailflow-Pfade definiert enden. Genau dort liegen die meisten Rückfälle.

## Quellen

1. [Microsoft Learn – Fix NDR error 5.4.6 or 5.4.14 in Exchange Online](https://learn.microsoft.com/en-us/troubleshoot/exchange/email-delivery/ndr/fix-error-code-5-4-6-through-5-4-20-in-exchange-online): Bedeutung der Exchange-NDRs und typische Ursachen in Accepted Domains und Hybrid-Connectoren.

2. [Microsoft Learn – Manage accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains): Unterschiede zwischen `Authoritative` und `InternalRelay`.

3. [Microsoft Learn – Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains): Zuständigkeit, Relay-Domains und Recipient Lookup im lokalen Exchange.

4. [Microsoft Learn – Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing): Erwartete Transportwege mit und ohne Centralized Mail Transport.

5. [Microsoft Learn – Validate connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/validate-connectors): Connector-Validierung und Hinweise zu mehreren gleichzeitig passenden Connectoren.

6. [Microsoft Learn – Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud): Unterstützte Mailflow-Muster mit vorgeschalteten Drittanbieterdiensten.

7. [Microsoft Learn – Mail flow rules in Exchange Online](https://learn.microsoft.com/en-us/exchange/security-and-compliance/mail-flow-rules/mail-flow-rules): Verarbeitung, Priorität, Aktionen und Ausnahmen von Transportregeln.

8. [Microsoft Learn – Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2?view=exchange-ps): Suche nach Nachrichten im Exchange-Online-Transport.

9. [Microsoft Learn – Search message tracking logs](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/search-message-tracking-logs): Lokale Nachrichtenverfolgung über alle Exchange-Server.

10. [Microsoft Learn – Hop count exceeded for an on-premises application subdomain](https://learn.microsoft.com/en-us/troubleshoot/exchange/mailflow/hop-count-exceeded-possible-mail-loop): Dokumentiertes Subdomain-/EdgeSync-Szenario mit expliziter Internal-Relay-Domain.
