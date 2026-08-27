---
title: "AuthMechanism 10 und AuthAs Internal: Wie Exchange die Einlieferung im Header klassifiziert"
navTitle: "AuthMechanism 10"
description: "Der Header X-MS-Exchange-Organization-AuthMechanism dokumentiert, wie sich ein einliefernder Server authentifiziert hat. Der Wert 10 steht für einen Receive Connector mit Externally Secured und stuft externe Mails als intern ein: mit Folgen für Spamfilter, Mailflow-Regeln und Spoofing-Schutz."
date: "2026-08-26"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "8 Min. Lesezeit"
themen:
  - "exchange-onprem-hybrid"
  - "microsoft-365-exchange"
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange"
  - "exchange-hybrid"
protokolle:
  - "smtp"
  - "troubleshooting"
related:
  - "exchange-hybrid-header-intern-extern"
  - "exchange-message-tracking-und-receive-connectoren-analysieren"
  - "typische-ursachen-fuer-mail-loops-und-deren-behebung"
slug: "exchange-authmechanism-10-authas-internal"
translationId: "article-0df383d5c49016da"
url: "https://rafaelpfister.ch/blog/exchange-authmechanism-10-authas-internal"
---

# AuthMechanism 10 und AuthAs Internal: Wie Exchange die Einlieferung im Header klassifiziert

Bei der Analyse von Spam-, Spoofing- und Mailfluss-Fällen in Exchange-Umgebungen sind drei Kopfzeilen entscheidend, die Exchange beim Empfang stempelt:

```text
X-MS-Exchange-Organization-AuthAs: Internal
X-MS-Exchange-Organization-AuthSource: EX01.example.com
X-MS-Exchange-Organization-AuthMechanism: 10
```

`AuthAs` hält fest, als was der Absender gegenüber dem Transport aufgetreten ist. `AuthSource` nennt den Server, der die Bewertung vorgenommen hat. `AuthMechanism` dokumentiert, über welchen Mechanismus die Authentifizierung zustande kam. Zusammen bestimmen sie, ob Exchange eine Nachricht als intern oder extern behandelt, und diese Einstufung hat erhebliche Konsequenzen.

## Warum die Einstufung zählt

`AuthAs` kennt in der Praxis zwei Werte: `Internal` und `Anonymous`. Eine als `Internal` eingestufte Nachricht wird anders behandelt als externe Post:

- Mailflow-Regeln mit der Bedingung „Absender ausserhalb der Organisation" greifen nicht.
- Die Nachricht darf an Verteiler und Postfächer zustellen, die authentifizierte Absender verlangen (`RequireSenderAuthenticationEnabled`).
- Anti-Spam- und Anti-Spoofing-Prüfungen fallen milder aus oder entfallen; in Hybrid-Umgebungen wird der externe Disclaimer nicht angefügt und Outlook zeigt keinen „Extern"-Hinweis.
- Der Anzeigename aus dem Adressbuch wird aufgelöst, die Mail wirkt für Empfänger wie interne Post.

Genau deshalb gehört die Frage „AuthAs Internal oder Anonymous?" an den Anfang jeder Header-Analyse: Damit lässt sich klären, warum eine offensichtliche Spoofing-Mail am Spamfilter vorbeikam oder warum eine Mailflow-Regel nie ausgelöst hat.

## Die AuthMechanism-Werte

Microsoft dokumentiert die Kodierung von `AuthMechanism` nicht vollständig öffentlich. Zwei Werte sind für die Fehlersuche relevant und gut belegt:

| Wert | Bedeutung |
|---|---|
| `04` | Authentifizierter Exchange-Verkehr: Postfach zu Postfach innerhalb der Organisation sowie Hybrid-Verkehr über die vom Hybrid Configuration Wizard eingerichteten Connectoren. |
| `10` | Receive Connector mit der Authentifizierungsoption `ExternalAuthoritative` („Extern geschützt" / „Externally secured"): Die Verbindung gilt als ausserhalb von Exchange abgesichert, alles darüber Eingelieferte wird als intern behandelt. |

Weitere Werte tauchen in Headern auf, sind aber ohne offizielle Referenz. Für die Praxis reicht die Unterscheidung: `04` bedeutet echte Exchange-Authentifizierung, `10` bedeutet Vertrauen per Connector-Konfiguration.

## Was Externally Secured wirklich bedeutet

Die Option `ExternalAuthoritative` auf einem Receive Connector sagt Exchange: Die Absicherung dieser Verbindung übernimmt jemand anderes, zum Beispiel eine Firewall, ein dediziertes Netzsegment oder IPsec. Exchange prüft dann nichts mehr, sondern behandelt jede Einlieferung über diesen Connector als authentifiziert und intern, inklusive des Rechts, beliebige interne Absenderadressen zu verwenden.

Das ist für wenige Szenarien gedacht, etwa einen vollständig vertrauenswürdigen Applikationsserver im eigenen Rechenzentrum. Problematisch wird es, wenn der Connector auf ein vorgeschaltetes Mailgateway oder einen Spamfilter in der DMZ zeigt, über den auch Internet-Post hereinkommt. Dann trägt jede externe Mail nach der Einlieferung:

```text
X-MS-Exchange-Organization-AuthAs: Internal
X-MS-Exchange-Organization-AuthMechanism: 10
```

Die Folgen: Externe Mails gelten als intern, Mailflow-Regeln für externe Absender greifen nicht, der Spoofing-Schutz für die eigene Domain ist wirkungslos, und jeder, der das Gateway erreicht, kann mit internen Absenderadressen an Empfänger zustellen, die eigentlich authentifizierte Absender verlangen.

## Betroffene Connectoren finden

Welche Receive Connectoren mit `ExternalAuthoritative` konfiguriert sind, zeigt die Exchange Management Shell:

```powershell
Get-ReceiveConnector | Where-Object {
  $_.AuthMechanism -match "ExternalAuthoritative"
} | Format-Table Identity, RemoteIPRanges, AuthMechanism, PermissionGroups
```

Prüfen Sie bei jedem Treffer, welche `RemoteIPRanges` eingetragen sind und ob die Systeme dahinter dieses Vertrauen tatsächlich brauchen. Ein Gateway, das nur Mails weiterreichen soll, braucht es nicht.

## Die Alternative für Relay-Szenarien

Soll ein System lediglich anonym über Exchange relayen (Drucker, Applikationen, Monitoring), ist ein anonymer Relay-Connector die sauberere Lösung: anonyme Einlieferung plus das Recht, an beliebige Empfänger zuzustellen, aber ohne die Internal-Einstufung.

```powershell
New-ReceiveConnector -Name "Anonymous Relay" -TransportRole FrontendTransport `
  -RemoteIPRanges 192.0.2.10 -Bindings 0.0.0.0:25 -Usage Custom -PermissionGroups AnonymousUsers

Get-ReceiveConnector "EX01\Anonymous Relay" | Add-ADPermission `
  -User "NT AUTHORITY\ANONYMOUS LOGON" -ExtendedRights "ms-Exch-SMTP-Accept-Any-Recipient"
```

Mails über diesen Connector bleiben `AuthAs: Anonymous`, durchlaufen die normalen Prüfungen und können keine internen Absender vortäuschen. `ExternalAuthoritative` bleibt den Systemen vorbehalten, denen Sie das Recht auf interne Absenderadressen bewusst geben wollen.

## Header im Zusammenhang lesen

Ob eine konkrete Nachricht als intern oder extern eingestuft wurde und über welchen Weg sie kam, lässt sich am schnellsten am vollständigen Header ablesen: `AuthAs`, `AuthMechanism` und `AuthSource` zusammen mit der `Received`-Kette. Der [Mail-Header-Analyzer](/tools/header-analyzer) auf dieser Website wertet diese Felder direkt im Browser aus und markiert die Hybrid-Einstufung im Zustellweg; der Header verlässt den Browser dabei nicht.

Wie die Einstufung in Hybrid-Umgebungen zwischen Exchange Online und OnPrem erhalten bleibt und woran eine falsche Zuordnung zu erkennen ist, behandelt der Artikel [Intern oder extern? Exchange-Hybrid-Mails im Header einordnen](/blog/exchange-hybrid-header-intern-extern).

## Quellen

1.  [Microsoft Q&A: Exchange 2016 mail flow rule, which header is checked for "outside the organization"?](https://learn.microsoft.com/en-us/answers/questions/54418/exchange-2016-mail-flow-rule-which-header-is-check): Zuordnung von AuthAs und AuthMechanism 10 zur Externally-Secured-Konfiguration und deren Wirkung auf Mailflow-Regeln.

2.  [Microsoft Tech Community: Demystifying and troubleshooting hybrid mail flow: when is a message internal?](https://techcommunity.microsoft.com/blog/exchange/demystifying-and-troubleshooting-hybrid-mail-flow-when-is-a-message-internal/1420838): Offizielle Beschreibung der Internal-Einstufung und ihrer Konsequenzen im Hybrid-Mailfluss.

3.  [msxfaq: X-MS-Exchange-Organization-AuthAs](https://www.msxfaq.de/cloud/exchangeonline/transport/x-ms-exchange-organization-authas.htm): Beobachtete AuthAs-, AuthSource- und AuthMechanism-Werte in verschiedenen Einlieferungsszenarien.

4.  [Microsoft Learn: Allow anonymous relay on Exchange servers](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/allow-anonymous-relay): Einrichtung des anonymen Relay-Connectors als Alternative zu Externally Secured.
