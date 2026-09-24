---
title: "MailHeaderAnalyzer: E-Mail-Header in PowerShell auswerten, ohne Netzwerkzugriff"
navTitle: "MailHeaderAnalyzer (PowerShell)"
description: "Referenz zum PowerShell-Modul MailHeaderAnalyzer: Installation, Syntax und Parameter von Get-MailHeaderAnalysis und ConvertTo-MailHeaderReport, das Ausgabeobjekt mit Zustellkette, Authentifizierungsergebnissen und Exchange-Online-Klassifizierung, alle Finding-Codes und Beispiele für Einzelfall und Stapelverarbeitung."
date: "2026-09-24"
kategorie: "SMTP & Mailflow"
timeToRead: "14 Min. Lesezeit"
themen:
  - "smtp-mailflow"
  - "microsoft-365-exchange"
hauptthema: "smtp-mailflow"
produkte:
  - "exchange-online"
  - "uebergreifend"
protokolle:
  - "powershell"
  - "mail-auth"
  - "smtp"
  - "troubleshooting"
related:
  - "e-mail-header-analysieren-ohne-upload"
  - "microsoft-365-compauth-reason-codes"
  - "exchange-hybrid-header-intern-extern"
slug: "mailheaderanalyzer-powershell-modul"
translationId: "article-041d7b2f9615f670"
url: "https://rafaelpfister.ch/blog/mailheaderanalyzer-powershell-modul"
---

# MailHeaderAnalyzer: E-Mail-Header in PowerShell auswerten, ohne Netzwerkzugriff

MailHeaderAnalyzer ist ein PowerShell-Modul, das den Header einer E-Mail auswertet: Zustellkette mit Verzögerungen und TLS-Angaben, SPF-, DKIM-, DMARC- und ARC-Ergebnisse samt Prüfung, ob diese Ergebnisse tatsächlich vom empfangenden Server stammen, DMARC-Alignment, die Hybrid-Klassifizierung von Exchange Online, die Bewertungen von Microsoft Defender, SpamAssassin und Rspamd sowie Auffälligkeiten wie doppelte `From`-Zeilen oder Unicode-Steuerzeichen. Das Modul arbeitet vollständig offline. Es löst keine DNS-Namen auf und öffnet keine HTTP-Verbindung; der Header verlässt den Rechner nicht. Es ist die Kommandozeilen-Fassung des [Header-Analyzers auf dieser Website](/tools/header-analyzer) und nutzt dieselbe Auswertungslogik.

Das Modul liegt in der [PowerShell Gallery](https://www.powershellgallery.com/packages/MailHeaderAnalyzer), der Quellcode unter MIT-Lizenz auf [GitHub](https://github.com/pfstr/MailHeaderAnalyzer). Diese Seite ist die Referenzdokumentation.

## Voraussetzungen

| Anforderung | Details |
|---|---|
| PowerShell | Windows PowerShell 5.1 oder PowerShell 7.x |
| Betriebssystem | Windows, Linux, macOS (`-FromClipboard` nur unter Windows) |
| Rechte | Keine. Installation im Benutzerprofil mit `-Scope CurrentUser` |
| Netzwerk | Nur für die Installation. Die Analyse selbst benötigt keine Verbindung |
| Abhängigkeiten | Keine weiteren Module |

Das Modul läuft auch in der Exchange Management Shell, die auf Windows PowerShell 5.1 basiert.

## Installation

```powershell
Install-Module -Name MailHeaderAnalyzer -Scope CurrentUser
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-Name MailHeaderAnalyzer` | Name des Moduls in der PowerShell Gallery |
| `-Scope CurrentUser` | Installiert in das Modulverzeichnis des Benutzers, ohne Administratorrechte |

</details>

Auf einem System ohne Internetzugang (etwa einem Exchange-Server im Rechenzentrum) laden Sie das Modul auf einem anderen Rechner mit `Save-Module -Name MailHeaderAnalyzer -Path C:\Temp` herunter und kopieren den Ordner `MailHeaderAnalyzer` in ein Verzeichnis aus `$env:PSModulePath`, zum Beispiel `%USERPROFILE%\Documents\WindowsPowerShell\Modules` (5.1) oder `%USERPROFILE%\Documents\PowerShell\Modules` (7.x).

Aktualisieren:

```powershell
Update-Module -Name MailHeaderAnalyzer
```

Version prüfen:

```powershell
Get-Module -Name MailHeaderAnalyzer -ListAvailable | Select-Object Name, Version
```

## Schnellstart

Kopieren Sie den Header aus dem Mailclient in die Zwischenablage. In Outlook für Windows finden Sie ihn unter Datei, Eigenschaften, Internetkopfzeilen; in Outlook im Web über die Nachrichtenoptionen unter „Nachrichtendetails anzeigen“. Danach:

```powershell
Get-MailHeaderAnalysis -FromClipboard
```

```text
Subject     : Service-Report für März
From        : Beispiel Newsletter <news@example.org>
Date        : 2026-08-03 09:14:27Z
SPF         : pass
DKIM        : pass
DMARC       : pass
ARC         : -
CompAuth    : pass (reason 100)
AuthTrust   : Absent (no authserv-id, Microsoft 365 style)
Hops        : 3 (total 00:00:42)
DeliveredBy : zr0p278mb0570.chep278.prod.outlook.com
Findings    : none
```

Aus einer Datei, wahlweise nur der Header oder eine vollständige `.eml`-Nachricht (der Nachrichtentext wird ignoriert):

```powershell
Get-MailHeaderAnalysis -Path .\nachricht.eml
```

Die Zustellkette als Tabelle:

```powershell
(Get-MailHeaderAnalysis -Path .\nachricht.eml).Hops
```

```text
#   From                 IP              By                                     Protocol   TLS      Time (UTC)           Delay
-   ----                 --              --                                     --------   ---      ----------           -----
1   client.example.net   198.51.100.34   mail.example.org                       ESMTPSA    -        2026-08-03 09:14:28  -
2   mail.example.org     203.0.113.25    mx.eur02.prod.protection.outlook.com   Microsoft… TLS 1.3  2026-08-03 09:15:09  41 s
3   AM0EUR02FT056.eop…   -               ZR0P278MB0570.CHEP278.PROD.OUTLOOK.COM Microsoft… TLS 1.2  2026-08-03 09:15:10  1 s
```

## Get-MailHeaderAnalysis

Wertet einen E-Mail-Header aus und gibt ein Analyseobjekt zurück.

### Syntax

```powershell
Get-MailHeaderAnalysis [-Header] <String[]> [<CommonParameters>]

Get-MailHeaderAnalysis -Path <String[]> [<CommonParameters>]

Get-MailHeaderAnalysis -FromClipboard [<CommonParameters>]
```

### Parameter

<details class="options-details">
<summary>Optionen erklärt</summary>

| Parameter | Typ | Wirkung |
|---|---|---|
| `-Header` | `String[]`, Position 0, Pipeline | Der Header als Text. Pipeline-Eingaben werden zeilenweise gesammelt und zu einem Header zusammengesetzt; `Get-Content datei | Get-MailHeaderAnalysis` funktioniert deshalb ohne `-Raw`. Aliase: `Text`, `Raw`, `InputObject` |
| `-Path` | `String[]`, Pipeline nach Eigenschaftsname | Pfad zu einer Datei mit dem Header oder einer vollständigen `.eml`-Nachricht. Nimmt Objekte von `Get-ChildItem` entgegen (Alias `FullName`). Eine Byte Order Mark wird berücksichtigt, Standardkodierung ist UTF-8. Pro Datei ein Ergebnisobjekt |
| `-FromClipboard` | `Switch` | Liest den Header mit `Get-Clipboard -Raw`. Nur unter Windows verfügbar; auf anderen Plattformen bricht der Aufruf mit einer Fehlermeldung ab |

</details>

Die drei Parameter schliessen sich gegenseitig aus. Ohne Parameter wartet das Cmdlet auf Pipeline-Eingaben für `-Header`.

### Eingabeformat

Das Cmdlet erwartet den Rohheader, also Zeilen der Form `Name: Wert` mit RFC-5322-Faltung (Fortsetzungszeilen beginnen mit Leerzeichen oder Tabulator). Drei Abweichungen davon werden toleriert:

- Eine leere Zeile beendet den Header. Folgt danach Text, wird er als Nachrichtentext erkannt und ignoriert (`HadBody` im Ergebnis ist dann `$true`).
- Zeilen ohne Feldnamen und ohne führendes Leerzeichen, wie sie beim Kopieren aus manchen Client-Dialogen entstehen, werden dem vorhergehenden Feld zugeschlagen.
- Eine mbox-Trennzeile `From ...` vor dem ersten Feld wird übersprungen.

RFC-2047-kodierte Werte (`=?UTF-8?Q?...?=`) in Betreff und Adressen werden dekodiert. Ausgewertet werden höchstens 200 `Received`-Zeilen, gezählt von der Zustellung her; darüber hinausgehende Zeilen meldet das Finding `HopOverflow`.

### Ausgabe

Ein Objekt vom Typ `MailHeaderAnalyzer.Analysis` je Eingabe. Die Standardansicht zeigt die Zusammenfassung aus dem Schnellstart; alle Eigenschaften sind über `Select-Object`, `Format-List *` oder `ConvertTo-Json` zugänglich.

<details class="options-details">
<summary>Eigenschaften des Analyseobjekts</summary>

| Eigenschaft | Typ | Inhalt |
|---|---|---|
| `Source` | String | Dateipfad, `Clipboard` oder `Text` |
| `Subject` | String | Betreff, RFC 2047 dekodiert |
| `From`, `ReplyTo`, `ReturnPath` | Adressobjekt | `Name`, `Address`, `Domain`, `Display`; `$null`, wenn das Feld fehlt |
| `Date` | DateTime (UTC) | Wert des `Date`-Felds |
| `MessageId` | String | `Message-ID` |
| `MailFromDomain` | String | Envelope-Absenderdomain aus `smtp.mailfrom` der SPF-Prüfung, sonst aus `Return-Path` |
| `Spf`, `Dkim`, `Dmarc`, `Arc`, `CompAuth` | String | Ergebnis laut massgebender `Authentication-Results`-Zeile (`pass`, `fail`, `none`, `softfail`, …); `$null`, wenn nicht geprüft |
| `CompAuthReason`, `CompAuthReasonMeaning` | String | Reason-Code der zusammengesetzten Authentifizierung von Microsoft 365 und seine Bedeutung |
| `AuthTrust` | String | `Matched`, `Unmatched`, `Absent` oder `None`, siehe unten |
| `AuthServId` | String | authserv-id der massgebenden Prüfzeile |
| `AuthenticationResults` | Objekt[] | Alle `Authentication-Results`-Zeilen mit `AuthServId`, `Methods`, `Trust`, `Raw` |
| `ReceivedSpf` | Objekt | Die `Received-SPF`-Zeile mit `Result` und `Properties` |
| `SpfAlignment`, `DkimAlignment` | String | `Strict`, `Relaxed`, `None` oder `$null` |
| `Hops` | Hop[] | Zustellkette in chronologischer Reihenfolge (erster Hop zuerst) |
| `HopCount`, `TotalDuration`, `SlowestHopIndex`, `HasClockSkew` | Int, TimeSpan, Int, Bool | Kennzahlen der Kette |
| `DeliveredBy` | String | `by`-Host der jüngsten `Received`-Zeile, also die zustellende Station |
| `DkimSignatures` | Objekt[] | Je Signatur `Domain`, `Selector`, `Algorithm`, `Canonicalization`, `SignedHeaders`, `BodyLength`, `Timestamp`, `Expires`, `ReceiverResult`, `Tags` |
| `ArcChain`, `ArcValid` | Objekt[], Bool | ARC-Instanzen mit `Instance`, `SealDomain`, `ChainValidation`, `Methods`; `ArcValid` ist `$null` ohne ARC-Kette |
| `Exchange` | Objekt | Hybrid-Klassifizierung von Exchange Online, siehe unten; `$null` ohne entsprechende Header |
| `Spam` | Objekt | Bewertungen der Spamfilter, siehe unten; `$null` ohne entsprechende Header |
| `List` | Objekt | `ListId`, `Unsubscribe`, `OneClick` (RFC 8058); `$null` ohne Listen-Header |
| `Findings` | Finding[] | Auffälligkeiten mit `Severity`, `Code`, `Message` |
| `Fields` | Objekt[] | Alle Felder mit `Name`, `Value` (entfaltet) und `Raw` |
| `HadBody` | Bool | Ob nach dem Header ein Nachrichtentext folgte |

</details>

### Hop-Objekt

Jeder Eintrag in `Hops` entspricht einer `Received`-Zeile. Die Reihenfolge ist chronologisch, also umgekehrt zur Reihenfolge im Header.

<details class="options-details">
<summary>Eigenschaften des Hop-Objekts</summary>

| Eigenschaft | Inhalt |
|---|---|
| `Index` | Laufende Nummer, 1 = Einlieferung |
| `FromHost`, `Helo`, `ReverseDns`, `IPAddress`, `IsPrivateIP` | Angaben zum einliefernden System aus dem `from`-Teil; die IP stammt aus der eckigen Klammer im Kommentar, der rDNS-Name aus dem Kommentar davor |
| `ByHost`, `Software` | Empfangendes System und dessen Software (Kommentar hinter `by`) |
| `Protocol`, `ProtocolClass` | `with`-Wert und Klasse nach RFC 3848: `TlsAuthenticated` (ESMTPSA), `Tls` (ESMTPS oder TLS-Angabe vorhanden), `Authenticated` (ESMTPA), `Plain`, `Http`, `Mapi`, `Local` |
| `TlsVersion`, `TlsCipher` | Aus den Schreibweisen von Microsoft (`version=TLS1_2, cipher=…`), Postfix (`using TLSv1.3 with cipher …`) und Exim |
| `Id`, `For`, `Via` | Weitere `Received`-Bestandteile |
| `Date` | Zeitstempel hinter dem Semikolon, UTC |
| `Delay` | TimeSpan zum vorhergehenden Hop; negativ bei Uhrenversatz |
| `Provider` | Erkannter Anbieter oder Gateway anhand der Hostnamen (Microsoft 365, Google Workspace, SEPPmail, HIN, Proofpoint, Mimecast und weitere) |
| `Attested` | `$true` nur beim letzten Hop: allein diese Zeile hat das empfangende System selbst geschrieben, alle darunter standen bereits in der Nachricht |
| `Raw` | Die Originalzeile |

</details>

### AuthTrust: Herkunft der Prüfergebnisse

Eine `Authentication-Results`-Zeile kann jeder Absender selbst in eine Nachricht schreiben. Nach RFC 8601, Abschnitt 5, ist nur die Zeile der empfangenden Organisation massgebend, und deren authserv-id muss sich einer Station der Zustellkette zuordnen lassen. Das Cmdlet vergleicht die authserv-id jeder Zeile mit den `by`-Hosts der `Received`-Kette (gleiche Domain oder Subdomain, immer an der Punktgrenze, kein Teilstring).

| Wert | Bedeutung |
|---|---|
| `Matched` | Die authserv-id kommt als `by`-Host in der Kette vor. Nur solche Zeilen fliessen in `Spf`, `Dkim`, `Dmarc` ein, sobald mindestens eine existiert |
| `Unmatched` | Die authserv-id kommt in der Kette nicht vor. Die Ergebnisse werden zwar angezeigt, gelten aber als unbelegte Behauptung; das Finding `AuthUnverified` weist darauf hin |
| `Absent` | Die Zeile trägt keine authserv-id. Microsoft 365 schreibt seine Prüfzeile in dieser Form, sie beginnt direkt mit `spf=` |
| `None` | Keine Prüfzeile vorhanden |

Enthält der Header Prüfzeilen mehrerer Herkünfte, meldet das Finding `AuthMixedOrigins` den Sachverhalt. Fehlt eine Prüfzeile, aber eine `Received-SPF`-Zeile ist vorhanden, wird deren Ergebnis als `Spf` übernommen und mit `ReceivedSpfOnly` gekennzeichnet.

### DMARC-Alignment

`SpfAlignment` vergleicht die Envelope-Absenderdomain mit der `From`-Domain, `DkimAlignment` die `d=`-Domain der geprüften Signatur mit der `From`-Domain. `Strict` bedeutet identische Domain, `Relaxed` dieselbe Organisationsdomain, `None` keine Übereinstimmung. Die Organisationsdomain wird heuristisch bestimmt: die letzten zwei Labels, bei bekannten mehrteiligen Endungen wie `co.uk` oder `com.au` die letzten drei. Eine vollständige Public Suffix List ist nicht enthalten.

### Exchange-Objekt

Wenn der Header Felder `X-MS-Exchange-Organization-*`, `X-OriginatorOrg` oder `X-MS-Exchange-CrossTenant-*` enthält, füllt das Cmdlet die Eigenschaft `Exchange`. Die Bedeutungen folgen dem Artikel „Demystifying hybrid mail flow“ des Exchange-Teams; die Hintergründe stehen im Artikel [Exchange-Hybrid-Header: intern oder extern?](/blog/exchange-hybrid-header-intern-extern).

<details class="options-details">
<summary>Eigenschaften des Exchange-Objekts</summary>

| Eigenschaft | Inhalt |
|---|---|
| `Directionality`, `DirectionalityMeaning` | `Originating` oder `Incoming` mit Erklärung |
| `AuthAs`, `AuthAsMeaning` | `Internal` oder `Anonymous` mit den Folgen für die EOP-Filterung |
| `AuthSource` | Server, der die Einstufung vorgenommen hat |
| `AuthMechanism`, `AuthMechanismMeaning` | Mechanismus-Code. Nur der Wert 10 (Externally Secured) ist öffentlich dokumentiert; für alle anderen Codes nennt das Modul ausdrücklich, dass Microsoft sie nicht dokumentiert |
| `OriginatorOrg` | Standard-Domäne des sendenden Tenants, das nicht fälschbare Tenant-Merkmal beim Empfang aus Microsoft 365 |
| `CrossTenantAuthAs`, `CrossTenantAuthSource`, `CrossTenantId` | Einstufung und Tenant-ID an der Tenant-Grenze |
| `CrossTenantFromEntity`, `CrossTenantFromMeaning` | `Internet`, `Hosted` oder `HybridOnPrem` mit Erklärung |
| `WrongTenantAttribution` | Wert von `X-MS-Exchange-CrossTenant-OriginalAttributedTenantConnectingIp`, wenn die Nachricht einem fremden Tenant zugeordnet wurde |
| `OrganizationHeadersPreserved`, `CrossPremisesHeadersFiltered` | Marker für erhaltene beziehungsweise vom Sendeconnector entfernte Organisations-Header |

</details>

### Spam-Objekt

Die Eigenschaft `Spam` fasst die Bewertungen der bekannten Filter zusammen. Die Werte stammen aus fremden Systemen und werden dekodiert, aber nicht bewertet.

<details class="options-details">
<summary>Eigenschaften des Spam-Objekts</summary>

| Eigenschaft | Quelle | Inhalt |
|---|---|---|
| `Scl`, `SclMeaning` | `X-Forefront-Antispam-Report`, `X-Microsoft-Antispam` | Spam Confidence Level mit Bedeutung (-1 vertrauenswürdig, 0/1 kein Spam, 5/6 Spamverdacht, 9 sehr wahrscheinlich Spam) |
| `Bcl` | `X-Microsoft-Antispam` | Bulk Complaint Level 0 bis 9 |
| `Category`, `CategoryMeaning` | `CAT:` | Klassifizierung wie `SPM`, `PHSH`, `HPHSH`, `MALW`, `SPOOF`, `DIMP`, `UIMP`, `BULK` |
| `SpamFilterVerdict`, `SpamFilterMeaning` | `SFV:` | Filterergebnis wie `NSPM`, `SPM`, `SKA` (Allowlist), `SKI` (intraorganisatorisch) |
| `IPVerdict`, `IPVerdictMeaning` | `IPV:` | `CAL` (IP auf Verbindungs-Allowlist) oder `NLI` (keine Reputation) |
| `Direction`, `DirectionMeaning` | `DIR:` | `INB`, `OUT`, `INT` |
| `ConnectingIP`, `Country` | `CIP:`, `CTRY:` | Einliefernde IP und Herkunftsland |
| `Forefront` | `X-Forefront-Antispam-Report` | Alle Schlüssel-Wert-Paare des Felds |
| `SpamAssassinScore`, `SpamAssassinTests` | `X-Spam-Status` | Punktzahl und ausgelöste Tests |
| `RspamdSymbols` | `X-Spamd-Result`, `X-Spam-Report` | Symbole mit Punktzahl |

</details>

Die vollständige Liste der `compauth`-Reason-Codes steht im Artikel [Microsoft 365 compauth: Reason-Codes](/blog/microsoft-365-compauth-reason-codes).

### Findings

Auffälligkeiten liefert das Cmdlet als Objekte in `Findings`, jeweils mit `Severity` (`Info`, `Warning`, `Fail`), einem stabilen `Code` für Filter und Skripte sowie einer Erklärung in `Message`.

<details class="options-details">
<summary>Alle Finding-Codes</summary>

| Code | Schweregrad | Bedeutung |
|---|---|---|
| `DuplicateField` | Warning | Ein Feld, das RFC 5322 auf eine Instanz begrenzt (`From`, `Subject`, `Date`, `Message-ID` und weitere), kommt mehrfach vor. Mailclients und Filter wählen unter Umständen verschiedene Instanzen; ein bekanntes Muster bei Fälschungen |
| `BidiControls` | Warning | Unicode-Steuerzeichen für die Schreibrichtung in einem Feld. Sie kehren die Leserichtung um, `fdp.exe` erscheint dann als `exe.pdf`. Das Modul zeigt sie als `<U+202E>` |
| `HopOverflow` | Warning | Mehr als 200 `Received`-Zeilen; die überzähligen wurden nicht ausgewertet |
| `AuthUnverified` | Warning | Die Prüfergebnisse tragen eine authserv-id, die in der Zustellkette nicht vorkommt |
| `AuthMixedOrigins` | Warning | Prüfzeilen mehrerer Herkünfte vorhanden |
| `ReceivedSpfForeign` | Warning | Der `receiver=` der `Received-SPF`-Zeile kommt in der Kette nicht vor |
| `ReceivedSpfOnly` | Info | Das SPF-Ergebnis stammt nur aus `Received-SPF`, nicht aus einer Prüfzeile |
| `NoAuthResults` | Info | Keine Prüfergebnisse im Header |
| `DmarcFail` | Fail | DMARC laut Empfangsserver nicht bestanden |
| `SpfNotPass` | Warning | SPF-Ergebnis `fail`, `softfail`, `permerror` oder `temperror` |
| `DkimNotPass` | Warning | DKIM-Ergebnis `fail`, `permerror` oder `temperror` ohne ARC-Zeugen |
| `DkimBrokenAfterForward` | Info | DKIM beim Empfänger nicht bestanden, aber ein ARC-Siegel derselben Domain bezeugt eine zuvor gültige Signatur: typisch für Weiterleitungen und Mailinglisten |
| `DkimWeakHash` | Warning | Signatur mit `rsa-sha1` (RFC 8301 stuft SHA-1 als veraltet ein) |
| `DkimBodyLength` | Warning | `l=`-Tag begrenzt die signierte Länge des Nachrichtentexts; angehängter Inhalt ist nicht abgedeckt |
| `DkimExpired` | Warning | `x=`-Zeitpunkt liegt in der Vergangenheit |
| `DkimFromUnsigned` | Warning | Das `From`-Feld ist nicht in `h=` enthalten, obwohl RFC 6376 es verlangt |
| `ClockSkew` | Info | Ein Hop trägt einen früheren Zeitstempel als sein Vorgänger; die Verzögerungen sind nur Näherungswerte |
| `ReplyToMismatch` | Info | `Reply-To`-Domain weicht von der `From`-Domain ab; bei Newslettern üblich, bei Phishing ein Muster |
| `SpfNotAligned` | Info | Envelope-Absenderdomain und `From`-Domain gehören zu verschiedenen Organisationen; SPF trägt dann nicht zu DMARC bei |
| `ExchangeWrongTenant` | Warning | Nachricht wurde einem fremden Tenant zugeordnet; klassische Ursache ist ein Inbound-Connector eines anderen Tenants mit demselben Zertifikat oder denselben IP-Adressen |
| `ExchangeHeadersFiltered` | Warning | Der Sendeconnector hat die Cross-Premises-Header entfernt (KB3212872) |
| `ExchangeExternallySecured` | Warning | AuthMechanism 10: Eingang über einen Empfangsconnector mit „Externally Secured“, EOP-Filterung übersprungen |
| `SpamCategory` | Warning | Microsoft hat eine Kategorie ungleich `NONE` vergeben |
| `SpamConfidence` | Warning | SCL 5 oder höher |

</details>

## ConvertTo-MailHeaderReport

Erzeugt aus einem Analyseobjekt einen Bericht als Markdown oder Text, etwa für ein Ticket oder eine Übergabe.

### Syntax

```powershell
ConvertTo-MailHeaderReport [-Analysis] <Object> [-Format <String>] [<CommonParameters>]
```

### Parameter

<details class="options-details">
<summary>Optionen erklärt</summary>

| Parameter | Typ | Wirkung |
|---|---|---|
| `-Analysis` | `MailHeaderAnalyzer.Analysis`, Pipeline | Das Objekt von `Get-MailHeaderAnalysis` |
| `-Format` | `Markdown` (Standard) oder `Text` | Ausgabeformat. Markdown enthält die Zustellkette als Tabelle, Text eine eingerückte Liste |

</details>

Der Bericht umfasst Betreff, Absender, Datum und Message-ID, die Authentifizierungsergebnisse mit Herkunftsangabe und Alignment, alle Findings, die Zustellkette, die Exchange-Klassifizierung und die Spamfilter-Werte. Steuerzeichen für die Schreibrichtung bleiben im Bericht als `<U+...>` sichtbar, damit sie über den Bericht nicht in ein Ticketsystem gelangen.

```powershell
Get-MailHeaderAnalysis -Path .\nachricht.eml | ConvertTo-MailHeaderReport | Set-Clipboard
```

```markdown
# Email header analysis

- Subject: Service-Report für März
- From: Beispiel Newsletter <news@example.org>
- Date: 2026-08-03T09:14:27Z
- Message-ID: <20260803091427.4Wq2Wx5RbTz@mail.example.org>

## Authentication

- spf: pass
- dkim: pass
- dmarc: pass
- compauth: pass (reason=100: passed authentication)
- Results without authserv-id (Microsoft 365 style)
- DMARC alignment: SPF Strict, DKIM Strict

## Delivery chain (total: 42 s)

| # | From | By | Protocol | TLS | Time (UTC) | Delay |
|---|---|---|---|---|---|---|
| 1 | client.example.net (198.51.100.34) | mail.example.org | ESMTPSA | - | 2026-08-03T09:14:28Z | - |
| 2 | mail.example.org (203.0.113.25) | mx.eur02.prod.protection.outlook.com | Microsoft SMTP Server | TLS 1.3 | 2026-08-03T09:15:09Z | 41 s |
| 3 | AM0EUR02FT056.eop-EUR02.prod.protection.outlook.com | ZR0P278MB0570.CHEP278.PROD.OUTLOOK.COM | Microsoft SMTP Server | TLS 1.2 | 2026-08-03T09:15:10Z | 1 s |
```

## Beispiele

### Nur die Auffälligkeiten anzeigen

```powershell
(Get-MailHeaderAnalysis -FromClipboard).Findings
```

### Die langsamste Station finden

```powershell
$a = Get-MailHeaderAnalysis -Path .\nachricht.eml
$a.Hops[$a.SlowestHopIndex - 1] | Format-List Index, FromHost, ByHost, Delay
```

`SlowestHopIndex` ist die 1-basierte Nummer des Hops mit der grössten Verzögerung; das Array `Hops` ist 0-basiert.

### Stapelverarbeitung mit CSV-Export

```powershell
Get-ChildItem -Path .\export\*.eml |
    Get-MailHeaderAnalysis |
    Select-Object Source, Subject, Spf, Dkim, Dmarc, AuthTrust, HopCount,
        @{ Name = 'Findings'; Expression = { ($_.Findings.Code | Sort-Object -Unique) -join ',' } } |
    Export-Csv -Path .\auswertung.csv -NoTypeInformation -Encoding UTF8
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `Get-ChildItem -Path .\export\*.eml` | Liefert die Dateiobjekte; `-Path` von `Get-MailHeaderAnalysis` übernimmt deren `FullName` |
| `Select-Object … @{ Name; Expression }` | Berechnete Spalte, die alle Finding-Codes zu einer Zeichenkette zusammenfasst |
| `Export-Csv -NoTypeInformation -Encoding UTF8` | CSV ohne Typkopfzeile, UTF-8 für Umlaute in Betreffzeilen |

</details>

### Nachrichten mit unbelegten Prüfergebnissen herausfiltern

```powershell
Get-ChildItem .\export\*.eml |
    Get-MailHeaderAnalysis |
    Where-Object AuthTrust -eq 'Unmatched' |
    Select-Object Source, AuthServId, DeliveredBy
```

Das Ergebnis listet Nachrichten, deren `Authentication-Results`-Zeile nicht vom zustellenden System stammt. Bei Phishing-Analysen ist das ein schneller erster Filter.

### Vollständige Ausgabe als JSON

```powershell
Get-MailHeaderAnalysis -Path .\nachricht.eml | ConvertTo-Json -Depth 6 | Set-Content .\analyse.json
```

`-Depth 6` ist nötig, weil `Hops`, `DkimSignatures` und `Exchange` verschachtelte Objekte sind; der Standardwert 2 würde sie als Typnamen ausgeben.

### In der Exchange Management Shell

```powershell
Import-Module MailHeaderAnalyzer
Get-MailHeaderAnalysis -Path 'C:\Temp\bounce.eml' | ConvertTo-MailHeaderReport -Format Text
```

Die Exchange Management Shell lädt keine Module automatisch aus dem Benutzerprofil nach, wenn `$env:PSModulePath` durch Gruppenrichtlinien eingeschränkt ist. In dem Fall hilft `Import-Module` mit dem vollständigen Pfad zur `.psd1`-Datei.

## Funktionsweise und Grenzen

Das Modul liest, was im Header steht, und leitet daraus ab, was sich ohne externe Abfrage belegen lässt. Daraus folgen einige Grenzen:

- **Keine kryptografische Prüfung.** DKIM-Signaturen werden nicht nachgerechnet und DNS-Einträge nicht abgefragt. `Spf`, `Dkim` und `Dmarc` sind immer das Urteil des empfangenden Servers, ergänzt um die Prüfung, ob dieses Urteil überhaupt von ihm stammt.
- **Nur die letzte `Received`-Zeile ist belegt.** Alle Zeilen darunter hat der Absender mitgeliefert und kann sie beliebig gestaltet haben. `Attested` markiert diesen Unterschied; die Verzögerungen früherer Hops beruhen auf den Angaben dieser Zeilen.
- **Organisationsdomänen heuristisch.** Für das Relaxed-Alignment nutzt das Modul eine kurze Liste mehrteiliger Endungen, keine vollständige Public Suffix List.
- **AuthMechanism nur teilweise dokumentiert.** Ausser dem Wert 10 hat Microsoft die Codes nicht veröffentlicht; das Modul erfindet keine Bedeutungen.
- **Zeichensätze.** RFC-2047-Werte werden mit den Kodierungen dekodiert, die .NET auf dem jeweiligen System kennt. Unbekannte Zeichensätze bleiben unverändert stehen.

## Datenschutz

Ein vollständiger Header enthält interne Hostnamen, IP-Adressen, Absender, Empfänger und Betreff. Warum diese Angaben nicht in ein Online-Tool gehören, steht im Artikel [E-Mail-Header analysieren, ohne die Mail hochzuladen](/blog/e-mail-header-analysieren-ohne-upload). Für das Modul gilt dasselbe Versprechen wie für die Browser-Fassung: keine Netzwerkzugriffe. Die Testsuite des Moduls enthält einen Test, der fehlschlägt, sobald im Quellcode ein Netzwerk- oder DNS-Cmdlet auftaucht.

## Quellcode und Versionen

Der Quellcode steht unter MIT-Lizenz auf [GitHub](https://github.com/pfstr/MailHeaderAnalyzer). Die Auswertungslogik ist eine Portierung der Bibliothek, die auch der Header-Analyzer auf dieser Website verwendet; beide teilen sich die Testfälle. Die Continuous Integration prüft jede Änderung mit PSScriptAnalyzer und Pester auf Windows PowerShell 5.1, PowerShell 7 unter Windows, Ubuntu und macOS. Veröffentlichungen in der PowerShell Gallery erfolgen automatisch aus versionierten Tags; die Änderungen je Version stehen im [Changelog](https://github.com/pfstr/MailHeaderAnalyzer/blob/main/CHANGELOG.md).

Fehler und Erweiterungswünsche nehme ich als [Issue auf GitHub](https://github.com/pfstr/MailHeaderAnalyzer/issues) entgegen. Testheader für Fehlerberichte bitte vorher anonymisieren; die mitgelieferten Testfälle verwenden ausschliesslich Beispieldomänen nach RFC 2606 und Adressen nach RFC 5737.

## Quellen

1.  [MailHeaderAnalyzer in der PowerShell Gallery](https://www.powershellgallery.com/packages/MailHeaderAnalyzer): Paketseite mit Installationsbefehl und Versionsverlauf.

2.  [pfstr/MailHeaderAnalyzer auf GitHub](https://github.com/pfstr/MailHeaderAnalyzer): Quellcode, Testsuite, Changelog und Issues.

3.  [RFC 8601: Message Header Field for Indicating Message Authentication Status](https://www.rfc-editor.org/rfc/rfc8601): Aufbau von `Authentication-Results` und die Regel, dass nur die Zeile der empfangenden Organisation massgebend ist (Abschnitt 5).

4.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): Aufbau der `Received`-Zeilen und die Empfehlung einer Obergrenze als Schleifenschutz (Abschnitt 6.3).

5.  [RFC 3848: ESMTP and LMTP Transmission Types Registration](https://www.rfc-editor.org/rfc/rfc3848): Die `with`-Werte `ESMTPS`, `ESMTPA` und `ESMTPSA`.

6.  [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://www.rfc-editor.org/rfc/rfc6376): Tags der `DKIM-Signature`, Pflicht zur Signierung von `From`.

7.  [RFC 8301: Cryptographic Algorithm and Key Usage Update to DKIM](https://www.rfc-editor.org/rfc/rfc8301): Einstufung von `rsa-sha1` als veraltet.

8.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc7489): Strict und Relaxed Alignment.

9.  [RFC 8617: The Authenticated Received Chain (ARC) Protocol](https://www.rfc-editor.org/rfc/rfc8617): `ARC-Seal`, `ARC-Authentication-Results` und die `cv=`-Kettenprüfung.

10.  [Microsoft Learn: Anti-spam message headers in Microsoft 365](https://learn.microsoft.com/en-us/defender-office-365/message-headers-eop-mdo): Bedeutung von SCL, BCL, CAT, SFV, IPV und der `compauth`-Reason-Codes.

11.  [Exchange Team Blog: Demystifying hybrid mail flow](https://techcommunity.microsoft.com/blog/exchange/demystifying-hybrid-mail-flow-when-is-a-message-internal/1420838): Herkunft der Bedeutungen von MessageDirectionality, AuthAs und AuthMechanism.

12.  [Header-Analyzer auf rafaelpfister.ch](/tools/header-analyzer): Die Browser-Fassung mit derselben Auswertungslogik.
