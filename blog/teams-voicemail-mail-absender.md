---
title: "Teams-Voicemail per E-Mail: neuer Absender und neuer Betreff ab März 2027 (MC1511072)"
navTitle: "Voicemail-Absender"
description: "Microsoft stellt die Zustellung von Teams-Voicemails per E-Mail um: Absender ist künftig «Microsoft Teams» aus der eigenen Tenant-Domain, die Anruferdaten wandern in den Betreff. Der Artikel zeigt, wer betroffen ist, wie Sie Transportregeln, Zulassungslisten und Automatisierungen prüfen und was vor März 2027 anzupassen ist."
date: "2026-11-02"
kategorie: "Microsoft Teams"
timeToRead: "4 Min. Lesezeit"
themen:
  - "microsoft-teams"
  - "microsoft-365-exchange"
produkte:
  - "teams"
  - "exchange-online"
protokolle:
  - "smtp"
  - "mail-auth"
  - "powershell"
hauptthema: "microsoft-teams"
slug: "teams-voicemail-mail-absender"
translationId: "article-b75c975f086dd0d5"
url: "https://rafaelpfister.ch/blog/teams-voicemail-mail-absender"
---

Mit der Message-Center-Meldung MC1511072 vom 16. Oktober 2026 kündigt Microsoft eine neue Zustellung der Voicemail-E-Mails von Teams an. Der Rollout beginnt nach aktuellem Plan Anfang März 2027 und soll bis Ende Juni 2027 abgeschlossen sein; Microsoft behält sich Verschiebungen vor und will die verbindlichen Termine sowie allfällige Kundenaktionen in einer weiteren Meldung nachreichen. Hintergrund ist die Unterstützung eigener Absenderdomains (Bring Your Own Domain), damit die Voicemail-Mails unter den SPF-, DKIM- und DMARC-Richtlinien der Organisation laufen.

## Was sich ändert

Das Feld «Von» enthält künftig weder Namen noch Telefonnummer oder E-Mail-Adresse des Anrufers. Diese Angaben stehen neu im Betreff und weiterhin im Nachrichtentext. In Teams selbst bleibt die Anzeige unverändert.

| Merkmal | Heute | Ab Rollout |
|---|---|---|
| Absender (persönliche Voicemail) | Name oder Nummer des Anrufers | `"Microsoft Teams" no-reply@<Standard-Tenant-Domain>` oder eigene Domain, falls konfiguriert |
| Absenderdomain | `skype.voicemail.microsoft.com` bei PSTN-Anrufen, Tenant-Domain bei Teams-zu-Teams-Anrufen | Standard-Tenant-Domain oder eigene Domain |
| Betreff (persönliche Voicemail) | `Voice mail (22 seconds)` | `Identified: John Smith (+19371817273) - Voicemail (00:22)` bzw. `Unidentified: +19371817273 - Voicemail (00:22)` |
| Anrufwarteschlange (geteilte Voicemail) | `Shared Voicemail (<CQ-Name>)` | `John Smith (+37251223123) - Shared voicemail in <CQ-Name> (00:05)` |
| Automatische Telefonzentrale (geteilte Voicemail) | `Shared Voicemail (<AA-Name>)` | `John Smith - Shared voicemail in <AA-Name> (00:11)` |

Für Benutzer hat das zwei sichtbare Folgen: Im Posteingang erscheinen alle Voicemails mit demselben Absender «Microsoft Teams», und weil die Adresse eine No-Reply-Adresse ist, erreicht die Funktion «Antworten» den Anrufer nicht mehr.

## Wer betroffen ist

Betroffen sind alle Organisationen, die Teams Calling mit Voicemail nutzen, also Teams Phone mit Calling Plan, Operator Connect oder Direct Routing sowie Anrufwarteschlangen und automatische Telefonzentralen mit geteilter Voicemail. Die Meldung nennt keine Lizenz- oder Cloud-Einschränkung; der Rollout ist weltweit angekündigt.

Handlungsbedarf besteht dort, wo etwas am heutigen Absender oder Betreff hängt:

- Exchange-Transportregeln, die auf `skype.voicemail.microsoft.com`, auf den Anrufer als Absender oder auf den Betreff `Voice mail` bzw. `Shared Voicemail` prüfen
- Einträge in Zulassungslisten: Tenant Allow/Block List, Spoof-Zulassungen, `AllowedSenderDomains` in Anti-Spam-Richtlinien
- Posteingangsregeln von Benutzern und Gruppenpostfächern der Anrufwarteschlangen, die Voicemails anhand von Absender oder Betreff einsortieren oder weiterleiten
- Power-Automate-Flows, Ticketsysteme und CRM-Anbindungen, die aus dem Absender die Telefonnummer des Anrufers lesen
- vorgelagerte oder nachgelagerte Gateways (zum Beispiel bei zentralisiertem Mailfluss im Hybrid-Betrieb), die Regeln oder Ausnahmen für die Voicemail-Domain enthalten

## Wie Sie prüfen, ob Sie betroffen sind

Die folgenden Abfragen laufen in einer Exchange-Online-PowerShell-Sitzung (`Connect-ExchangeOnline`).

### Transportregeln durchsuchen

```powershell
$muster = 'skype\.voicemail|voice ?mail|shared voicemail'
Get-TransportRule |
  Where-Object { ($_ | Format-List * | Out-String) -match $muster } |
  Select-Object Name, State, Priority
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `$muster` | Regulärer Ausdruck für die heutige Absenderdomain und die heutigen Betreff-Texte |
| `Format-List *` | Gibt alle Eigenschaften der Regel aus, damit Bedingungen und Ausnahmen mitdurchsucht werden |
| `Out-String` | Wandelt die Ausgabe in Text um, auf den `-match` angewendet wird |
| `Select-Object Name, State, Priority` | Zeigt nur Name, Status und Reihenfolge der gefundenen Regeln |

</details>

Jede gefundene Regel prüfen Sie auf Bedingungen wie `FromAddressContainsWords`, `SenderDomainIs` oder `SubjectContainsWords`.

### Zulassungslisten und Anti-Spam-Richtlinien

```powershell
Get-TenantAllowBlockListItems -ListType Sender |
  Where-Object Value -match 'voicemail'
Get-TenantAllowBlockListSpoofItems |
  Where-Object SpoofedUser -match 'voicemail'
Get-HostedContentFilterPolicy |
  Select-Object Name, AllowedSenderDomains, AllowedSenders
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-ListType Sender` | Liest die Absender-Einträge der Tenant Allow/Block List |
| `Where-Object Value -match 'voicemail'` | Filtert Einträge, deren Wert die Voicemail-Domain enthält |
| `Get-TenantAllowBlockListSpoofItems` | Listet die Spoof-Zulassungen und -Sperren |
| `AllowedSenderDomains, AllowedSenders` | Zulassungen direkt in den Anti-Spam-Richtlinien |

</details>

### Tatsächliches Mailaufkommen ermitteln

Die Nachrichtenablaufverfolgung zeigt, ob und wohin heute Voicemails zugestellt werden:

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddDays(-10) -EndDate (Get-Date) -ResultSize 5000 |
  Where-Object { $_.SenderAddress -like '*@skype.voicemail.microsoft.com' -or $_.Subject -match 'voice ?mail' } |
  Group-Object RecipientAddress | Sort-Object Count -Descending |
  Select-Object Count, Name
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-StartDate` / `-EndDate` | Zeitraum der Suche; Exchange Online liefert maximal die letzten 90 Tage, pro Abfrage höchstens 10 Tage |
| `-ResultSize 5000` | Höchstzahl der Treffer pro Abfrage |
| `Where-Object …` | Behält Mails von der PSTN-Voicemail-Domain oder mit Voicemail-Betreff |
| `Group-Object RecipientAddress` | Zählt die Voicemails pro Empfänger |

</details>

Die Empfängerliste zeigt, welche Postfächer und Gruppenpostfächer Voicemails erhalten. Dort lohnt sich ein Blick auf Posteingangsregeln (`Get-InboxRule -Mailbox <Adresse>`) und auf angebundene Flows.

## Was zu tun ist

1.  Regeln auf Betreff statt Absender umstellen: Neue Bedingungen auf die künftigen Betreffmuster `- Voicemail (` und `- Shared voicemail in ` ausrichten. Bis der Rollout den eigenen Tenant erreicht hat, alte und neue Bedingung parallel führen.

2.  Zulassungen für `skype.voicemail.microsoft.com` nach dem Rollout entfernen. Mails aus der eigenen Tenant-Domain brauchen keine Ausnahme; eine Zulassung für die eigene Domain wäre ein Sicherheitsrisiko, weil sie Spoofing erleichtert.

3.  Automatisierungen anpassen, die die Anrufernummer aus dem Absender lesen. Die Nummer steht künftig im Betreff in Klammern bzw. bei unbekannten Anrufern direkt nach `Unidentified:`.

4.  Eigene Absenderdomain vorbereiten: Die Konfigurationsanleitung für den Versand unter einer eigenen Domain kündigt Microsoft für die Folgemeldung an. Ist DMARC für diese Domain auf `p=reject` gesetzt, müssen SPF und DKIM den Versand aus Exchange Online abdecken, sonst landen die Voicemails in Quarantäne.

5.  Benutzer und Servicedesk informieren: Absender «Microsoft Teams», Anruferdaten im Betreff, keine Antwort an den Anrufer per E-Mail mehr möglich.

## Quellen

1.  [MC1511072: Microsoft Teams: Upcoming change to voicemail message delivery by email](https://mc.merill.net/message/MC1511072): archivierter Text der Message-Center-Meldung mit Zeitplan und Gegenüberstellung von heutigem und künftigem Absender und Betreff.

2.  [Microsoft Learn: Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2): Parameter und Grenzen der Nachrichtenablaufverfolgung in Exchange Online.

3.  [Microsoft Learn: Get-TenantAllowBlockListItems](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-tenantallowblocklistitems): Auslesen der Tenant Allow/Block List.

4.  [Microsoft Learn: Set up voicemail for Teams](https://learn.microsoft.com/en-us/microsoftteams/set-up-phone-system-voicemail): Grundlagen der Teams-Voicemail und ihrer Zustellung per E-Mail.
