---
title: "Exchange: Nur ausgewählte externe Absender an eine Verteilerliste zulassen"
navTitle: "Verteiler: externe Absender"
description: "Eine Verteilerliste in Exchange lässt sich so einrichten, dass nur bestimmte externe Absender sie erreichen. Zwei Wege mit PowerShell: Zustellbeschränkung mit E-Mail-Kontakten oder Transportregel. Dazu die Grenzen der Absenderprüfung und wie DMARC sie schliesst."
date: "2026-11-05"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "7 Min. Lesezeit"
themen:
  - "exchange-onprem-hybrid"
  - "microsoft-365-exchange"
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange"
  - "exchange-on-premises"
protokolle:
  - "smtp"
  - "mail-auth"
  - "powershell"
related:
  - "exchange-authmechanism-10-authas-internal"
  - "exchange-message-tracking-und-receive-connectoren-analysieren"
slug: "exchange-verteiler-externe-absender"
translationId: "article-b3dc70052f5e8729"
url: "https://rafaelpfister.ch/blog/exchange-verteiler-externe-absender"
---

# Exchange: Nur ausgewählte externe Absender an eine Verteilerliste zulassen

Eine Verteilerliste in Exchange nimmt standardmässig nur Nachrichten von authentifizierten, also internen Absendern an. Soll ein Partner, ein Lieferant oder ein Kunde die Liste ebenfalls erreichen, ohne dass sie damit für das ganze Internet offen steht, gibt es zwei Möglichkeiten: eine Zustellbeschränkung an der Gruppe selbst oder eine Transportregel. Beide funktionieren in Exchange Server 2016, 2019 und der Subscription Edition.

## Ausgangslage: RequireSenderAuthenticationEnabled

Ob eine Gruppe externe Nachrichten überhaupt annimmt, steuert die Eigenschaft `RequireSenderAuthenticationEnabled`. Steht sie auf `$true` (Standard bei neuen Verteilergruppen), weist Exchange jede Nachricht eines nicht authentifizierten Absenders ab. Beide folgenden Varianten setzen deshalb voraus, dass diese Prüfung abgeschaltet ist:

```powershell
Set-DistributionGroup -Identity "Projektteam" -RequireSenderAuthenticationEnabled $false
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-Identity` | Die Verteilergruppe (Name, Alias oder E-Mail-Adresse). |
| `-RequireSenderAuthenticationEnabled $false` | Die Gruppe nimmt auch Nachrichten von nicht authentifizierten, externen Absendern an. |

</details>

Ohne weitere Einschränkung kann nach diesem Schritt jeder externe Absender an die Liste schreiben. Die Einschränkung auf einzelne Absender folgt im nächsten Schritt, idealerweise im selben Wartungsfenster.

## Variante 1: Zustellbeschränkung mit E-Mail-Kontakten

Die Liste der erlaubten Absender (`AcceptMessagesOnlyFromSendersOrMembers`) akzeptiert nur Objekte aus dem eigenen Verzeichnis: Postfächer, E-Mail-Benutzer, E-Mail-Kontakte und Gruppen. Eine reine SMTP-Adresse lässt sich dort nicht eintragen. Für jeden externen Absender legen Sie daher einen E-Mail-Kontakt an. Damit der Kontakt nicht im globalen Adressbuch erscheint, blenden Sie ihn aus:

```powershell
$kontakt = @{
    Name                 = "Extern Max Muster"
    ExternalEmailAddress = "max.muster@partner.example"
    OrganizationalUnit   = "example.local/Kontakte/Extern"
}
New-MailContact @kontakt
Set-MailContact -Identity "Extern Max Muster" -HiddenFromAddressListsEnabled $true
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `Name` | Anzeigename des Kontakts im Active Directory. Ein einheitliches Präfix wie „Extern" erleichtert die spätere Pflege. |
| `ExternalEmailAddress` | Die externe Adresse, mit der der Absender schreibt. |
| `OrganizationalUnit` | Ziel-OU im Active Directory. Ohne Angabe landet der Kontakt im Standardcontainer `Users`. |
| `-HiddenFromAddressListsEnabled $true` | Blendet den Kontakt aus dem globalen Adressbuch und allen Adresslisten aus. |

</details>

Anschliessend tragen Sie den Kontakt als erlaubten Absender ein. Die Hashtable mit `Add` ergänzt die bestehende Liste, statt sie zu überschreiben:

```powershell
$erlaubt = @{ Add = "Extern Max Muster", "Extern Anna Beispiel" }
Set-DistributionGroup -Identity "Projektteam" -AcceptMessagesOnlyFromSendersOrMembers $erlaubt
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-AcceptMessagesOnlyFromSendersOrMembers` | Liste der Absender, die die Gruppe erreichen dürfen. Einzelne Absender und Gruppen sind möglich; bei Gruppen dürfen alle Mitglieder senden. |
| `@{ Add = ... }` | Fügt Einträge hinzu, ohne vorhandene zu entfernen. Mit `@{ Remove = ... }` werden Einträge entfernt; ein Wert ohne Hashtable ersetzt die ganze Liste. |

</details>

Sobald diese Liste mindestens einen Eintrag enthält, weist Exchange **alle anderen Absender** ab, auch interne. Sollen Mitarbeitende weiterhin an die Liste schreiben, tragen Sie eine passende Gruppe zusätzlich ein, zum Beispiel die Verteilergruppe selbst (dann dürfen alle Mitglieder senden) oder eine Gruppe mit allen Mitarbeitenden. Abgewiesene Absender erhalten einen Unzustellbarkeitsbericht mit dem Status `5.7.1`.

Den aktuellen Stand prüfen Sie so:

```powershell
Get-DistributionGroup -Identity "Projektteam" |
    Format-List Name, RequireSenderAuthenticationEnabled,
        AcceptMessagesOnlyFrom, AcceptMessagesOnlyFromDLMembers
```

`AcceptMessagesOnlyFromSendersOrMembers` ist ein Sammelparameter: Exchange verteilt die Einträge auf die Eigenschaften `AcceptMessagesOnlyFrom` (einzelne Absender) und `AcceptMessagesOnlyFromDLMembers` (Gruppen). Deshalb zeigt die Abfrage diese beiden Eigenschaften. Dieselbe Einstellung finden Sie im Exchange Admin Center in den Eigenschaften der Gruppe unter der Zustellungsverwaltung (Delivery management).

Diese Variante eignet sich, wenn nur wenige externe Absender dazukommen und die Liste der erlaubten Absender am Gruppenobjekt selbst sichtbar sein soll. Der Nachteil ist der Pflegeaufwand: Jeder neue Absender braucht einen Kontakt im Active Directory.

## Variante 2: Transportregel ohne Kontakte

Sollen intern weiterhin alle an die Liste schreiben und nur externe Absender eingeschränkt werden, ist eine Transportregel einfacher. Sie benötigt keine Kontakte und lässt sich auch auf ganze Domains anwenden. `RequireSenderAuthenticationEnabled` muss auch hier auf `$false` stehen.

```powershell
$regel = @{
    Name                              = "Projektteam: nur freigegebene externe Absender"
    SentTo                            = "projektteam@example.com"
    FromScope                         = "NotInOrganization"
    ExceptIfFromAddressMatchesPatterns = '^max\.muster@partner\.example$',
                                         '@lieferant\.example$'
    RejectMessageReasonText           = "Sie sind nicht berechtigt, an diese Adresse zu senden."
    RejectMessageEnhancedStatusCode   = "5.7.1"
}
New-TransportRule @regel
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `Name` | Name der Regel in der Liste der Transportregeln. |
| `SentTo` | Bedingung: Die Nachricht ist an diese Gruppe adressiert. |
| `FromScope NotInOrganization` | Bedingung: Der Absender gehört nicht zur eigenen Organisation. Interne Nachrichten sind damit nicht betroffen. |
| `ExceptIfFromAddressMatchesPatterns` | Ausnahme: Absenderadressen, die einem der regulären Ausdrücke entsprechen. `^…$` verankert eine einzelne Adresse, `@domain$` gibt eine ganze Domain frei. Punkte mit `\.` maskieren. |
| `RejectMessageReasonText` | Text im Unzustellbarkeitsbericht an den abgewiesenen Absender. |
| `RejectMessageEnhancedStatusCode` | Erweiterter Statuscode im Unzustellbarkeitsbericht; `5.7.1` steht für eine fehlende Berechtigung. |

</details>

Ohne die Verankerung mit `^` und `$` würde ein Muster wie `max\.muster@partner\.example` auch auf `max.muster@partner.example.net` passen. Neue Absender ergänzen Sie mit `Set-TransportRule` und der vollständigen, erweiterten Liste der Muster; der Parameter ersetzt die bestehende Liste.

## Grenze beider Varianten: Die Absenderadresse ist fälschbar

Beide Varianten prüfen nur die Absenderadresse. Exchange stellt dabei nicht fest, ob die Nachricht tatsächlich vom freigegebenen Partner stammt. Wer die Adresse eines freigegebenen Absenders kennt, kann sie fälschen und die Liste damit erreichen. Bei einer Liste mit internem Teilnehmerkreis ist das ein Einfallstor für Phishing, das zudem den Anschein einer bekannten Adresse hat.

Die Lücke schliessen SPF, DKIM und DMARC. Hat die Domain des Partners eine DMARC-Richtlinie mit `p=reject` oder `p=quarantine` und setzt Ihr Mail-Gateway diese Richtlinie durch, erreichen gefälschte Nachrichten dieser Domain Exchange gar nicht erst. Bei Domains ohne DMARC-Richtlinie oder mit `p=none` fehlt dieser Schutz.

Schreibt Ihr Gateway das Prüfergebnis in den Header `Authentication-Results`, kann eine zweite Transportregel freigegebene Absender zusätzlich abweisen, wenn DMARC nicht bestanden ist:

```powershell
$regel = @{
    Name                            = "Projektteam: freigegebene Absender nur mit DMARC pass"
    SentTo                          = "projektteam@example.com"
    FromScope                       = "NotInOrganization"
    FromAddressMatchesPatterns      = '^max\.muster@partner\.example$',
                                      '@lieferant\.example$'
    ExceptIfHeaderMatchesMessageHeader = "Authentication-Results"
    ExceptIfHeaderMatchesPatterns   = 'dmarc=pass'
    RejectMessageReasonText         = "Absender konnte nicht verifiziert werden."
    RejectMessageEnhancedStatusCode = "5.7.1"
}
New-TransportRule @regel
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `FromAddressMatchesPatterns` | Bedingung: Die Regel greift nur für die freigegebenen Absender, mit denselben Mustern wie in der ersten Regel. |
| `ExceptIfHeaderMatchesMessageHeader` | Name des Headers, den die Ausnahme prüft. |
| `ExceptIfHeaderMatchesPatterns` | Ausnahme: Enthält der Header `dmarc=pass`, wird die Nachricht zugestellt. Alle übrigen Nachrichten der freigegebenen Absender werden abgewiesen. |

</details>

Diese Regel funktioniert nur zuverlässig, wenn das Gateway eingehende `Authentication-Results`-Header aus dem Internet entfernt oder überschreibt. Sonst kann ein Absender den Header mit `dmarc=pass` selbst mitliefern. Welche Header Ihr Gateway setzt und ob es fremde Header entfernt, ist in dessen Dokumentation beschrieben; Exchange selbst wertet DMARC nicht aus.

## Welche Variante passt

Die Zustellbeschränkung mit Kontakten passt zu Listen mit festem, kleinem Kreis externer Absender und schränkt auf Wunsch auch interne Absender ein. Die Transportregel passt, wenn intern alle senden dürfen, ganze Partnerdomains freigegeben werden sollen oder keine Kontakte im Active Directory entstehen sollen. Unabhängig von der Variante prüfen Sie für jede freigegebene Domain die DMARC-Richtlinie und dokumentieren, wer die Liste der erlaubten Absender pflegt.

## Quellen

1.  [Microsoft Learn: Set-DistributionGroup](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-distributiongroup?view=exchange-ps): Parameter AcceptMessagesOnlyFromSendersOrMembers und RequireSenderAuthenticationEnabled, zulässige Objekttypen.

2.  [Microsoft Learn: Recipient resolution in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/recipient-resolution): Wie der Categorizer Zustellbeschränkungen und die Absenderauthentifizierung auswertet.

3.  [Microsoft Learn: New-MailContact](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-mailcontact?view=exchange-ps): Anlegen von E-Mail-Kontakten für externe Adressen.

4.  [Microsoft Learn: New-TransportRule](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-transportrule?view=exchange-ps): Bedingungen, Ausnahmen und Aktionen für Transportregeln, darunter FromScope und HeaderMatchesPatterns.

5.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489): Spezifikation der DMARC-Richtlinien p=none, quarantine und reject.

6.  [RFC 8601: Message Header Field for Indicating Message Authentication Status](https://datatracker.ietf.org/doc/html/rfc8601): Aufbau des Headers Authentication-Results und Hinweis, dass empfangende Systeme fremde Instanzen entfernen sollen.
