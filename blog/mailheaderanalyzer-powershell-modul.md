---
title: "MailHeaderAnalyzer: E-Mail-Header in PowerShell auswerten, ohne Netzwerkzugriff"
navTitle: "MailHeaderAnalyzer (PowerShell)"
description: "Referenz zum PowerShell-Modul MailHeaderAnalyzer: Parameterübersicht, Syntax, Beschreibung, Beispiele und Parametereigenschaften von Get-MailHeaderAnalysis und ConvertTo-MailHeaderReport, dazu das Ausgabeobjekt mit Zustellkette, Authentifizierungsergebnissen, Exchange-Online-Klassifizierung und allen Finding-Codes."
date: "2026-09-24"
kategorie: "SMTP & Mailflow"
timeToRead: "16 Min. Lesezeit"
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

MailHeaderAnalyzer ist ein PowerShell-Modul mit zwei Cmdlets. `Get-MailHeaderAnalysis` wertet den Header einer E-Mail aus: Zustellkette mit Verzögerungen und TLS-Angaben, SPF-, DKIM-, DMARC- und ARC-Ergebnisse samt Prüfung, ob diese Ergebnisse vom empfangenden Server stammen, DMARC-Alignment, die Hybrid-Klassifizierung von Exchange Online, die Bewertungen von Microsoft Defender, SpamAssassin und Rspamd sowie Auffälligkeiten wie doppelte `From`-Zeilen oder Unicode-Steuerzeichen. `ConvertTo-MailHeaderReport` erzeugt daraus einen Bericht für Tickets. Das Modul arbeitet vollständig offline: keine DNS-Abfragen, keine HTTP-Verbindungen. Es ist die Kommandozeilen-Fassung des [Header-Analyzers auf dieser Website](/tools/header-analyzer) und verwendet dieselbe Auswertungslogik.

| | |
|---|---|
| Modul | [MailHeaderAnalyzer in der PowerShell Gallery](https://www.powershellgallery.com/packages/MailHeaderAnalyzer) |
| Quellcode | [pfstr/MailHeaderAnalyzer auf GitHub](https://github.com/pfstr/MailHeaderAnalyzer), MIT-Lizenz |
| Gilt für | Windows PowerShell 5.1, PowerShell 7.x; Windows, Linux, macOS; Exchange Management Shell |
| Cmdlets | `Get-MailHeaderAnalysis`, `ConvertTo-MailHeaderReport` |

## Parameterübersicht

| Cmdlet | Parameter | Typ | Pflicht | Pipeline | Wirkung |
|---|---|---|---|---|---|
| `Get-MailHeaderAnalysis` | `-Header` | `String[]` | Ja (Parametersatz Text) | Ja, nach Wert | Der Header als Text. Zeilen aus der Pipeline werden zu einem Header zusammengesetzt |
| `Get-MailHeaderAnalysis` | `-Path` | `String[]` | Ja (Parametersatz Path) | Ja, nach Eigenschaftsname | Datei mit dem Header oder vollständige `.eml`-Nachricht; nimmt Objekte von `Get-ChildItem` entgegen |
| `Get-MailHeaderAnalysis` | `-FromClipboard` | `Switch` | Ja (Parametersatz Clipboard) | Nein | Liest den Header aus der Zwischenablage (nur Windows) |
| `ConvertTo-MailHeaderReport` | `-Analysis` | `MailHeaderAnalyzer.Analysis` | Ja | Ja, nach Wert | Das Ergebnisobjekt von `Get-MailHeaderAnalysis` |
| `ConvertTo-MailHeaderReport` | `-Format` | `String` | Nein | Nein | `Markdown` (Standard) oder `Text` |

Beide Cmdlets unterstützen die Common Parameters `-Verbose`, `-ErrorAction`, `-ErrorVariable`, `-OutVariable` und die übrigen aus [about_CommonParameters](https://learn.microsoft.com/de-de/powershell/module/microsoft.powershell.core/about/about_commonparameters).

## Installation

```powershell
Install-Module -Name MailHeaderAnalyzer -Scope CurrentUser
```

| Option | Wirkung |
|---|---|
| `-Name MailHeaderAnalyzer` | Name des Moduls in der PowerShell Gallery |
| `-Scope CurrentUser` | Installiert in das Modulverzeichnis des Benutzers, ohne Administratorrechte |

Auf einem System ohne Internetzugang laden Sie das Modul auf einem anderen Rechner mit `Save-Module -Name MailHeaderAnalyzer -Path C:\Temp` herunter und kopieren den Ordner `MailHeaderAnalyzer` in ein Verzeichnis aus `$env:PSModulePath`, zum Beispiel `%USERPROFILE%\Documents\WindowsPowerShell\Modules` (Windows PowerShell 5.1) oder `%USERPROFILE%\Documents\PowerShell\Modules` (PowerShell 7). Aktualisierungen holt `Update-Module -Name MailHeaderAnalyzer`, die installierte Version zeigt `Get-Module -Name MailHeaderAnalyzer -ListAvailable`.

## Get-MailHeaderAnalysis

Wertet den Header einer E-Mail aus und gibt ein Analyseobjekt zurück.

### Syntax

#### Text (Standard)

```powershell
Get-MailHeaderAnalysis
    [-Header] <String[]>
    [<CommonParameters>]
```

#### Path

```powershell
Get-MailHeaderAnalysis
    -Path <String[]>
    [<CommonParameters>]
```

#### Clipboard

```powershell
Get-MailHeaderAnalysis
    -FromClipboard
    [<CommonParameters>]
```

### Beschreibung

Das Cmdlet zerlegt den Rohheader in Felder, entfaltet RFC-5322-Faltungen und dekodiert RFC-2047-Werte in Betreff und Adressen. Aus den `Received`-Zeilen bildet es die Zustellkette in chronologischer Reihenfolge, berechnet die Verzögerung je Station und liest TLS-Version, Cipher und Protokollklasse nach RFC 3848. Aus `Authentication-Results`, `Received-SPF`, `DKIM-Signature` und der ARC-Kette ermittelt es die Authentifizierungsergebnisse und prüft, ob die Prüfzeile tatsächlich von einer Station der Zustellkette stammt (RFC 8601, Abschnitt 5). Dazu kommen das DMARC-Alignment, die Hybrid-Klassifizierung von Exchange Online, die Bewertungen der Spamfilter und eine Liste von Auffälligkeiten.

Das Cmdlet führt keine DNS-Abfragen durch und öffnet keine Netzwerkverbindung. `Spf`, `Dkim` und `Dmarc` sind deshalb immer das Urteil des empfangenden Servers, ergänzt um die Prüfung, ob dieses Urteil von ihm stammt. DKIM-Signaturen werden nicht kryptografisch nachgerechnet.

Eingaben werden tolerant gelesen: Eine Leerzeile beendet den Header, ein folgender Nachrichtentext wird ignoriert. Zeilen ohne Feldnamen und ohne führendes Leerzeichen, wie sie beim Kopieren aus Client-Dialogen entstehen, gehören zum vorhergehenden Feld. Eine mbox-Trennzeile `From ...` vor dem ersten Feld wird übersprungen, eine Byte Order Mark entfernt. Ausgewertet werden höchstens 200 `Received`-Zeilen, gezählt von der Zustellung her.

### Beispiele

#### Beispiel 1

```powershell
Get-MailHeaderAnalysis -FromClipboard
```

Wertet den Header aus, der sich in der Zwischenablage befindet. In Outlook für Windows finden Sie den Header unter Datei, Eigenschaften, Internetkopfzeilen; in Outlook im Web in den Nachrichtenoptionen unter „Nachrichtendetails anzeigen“.

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

#### Beispiel 2

```powershell
Get-MailHeaderAnalysis -Path .\nachricht.eml
```

Wertet eine gespeicherte Nachricht aus. Die Datei darf nur den Header oder die vollständige Nachricht enthalten; der Nachrichtentext wird ignoriert.

#### Beispiel 3

```powershell
(Get-MailHeaderAnalysis -Path .\nachricht.eml).Hops
```

Zeigt die Zustellkette als Tabelle, erster Hop zuerst.

```text
#   From                 IP              By                                     Protocol   TLS      Time (UTC)           Delay
-   ----                 --              --                                     --------   ---      ----------           -----
1   client.example.net   198.51.100.34   mail.example.org                       ESMTPSA    -        2026-08-03 09:14:28  -
2   mail.example.org     203.0.113.25    mx.eur02.prod.protection.outlook.com   Microsoft… TLS 1.3  2026-08-03 09:15:09  41 s
3   AM0EUR02FT056.eop…   -               ZR0P278MB0570.CHEP278.PROD.OUTLOOK.COM Microsoft… TLS 1.2  2026-08-03 09:15:10  1 s
```

#### Beispiel 4

```powershell
Get-Content -Path .\header.txt | Get-MailHeaderAnalysis | Select-Object -ExpandProperty Findings
```

Liest den Header zeilenweise aus einer Textdatei und zeigt nur die Auffälligkeiten. `-Raw` bei `Get-Content` ist nicht nötig, das Cmdlet setzt die Zeilen selbst zusammen.

#### Beispiel 5

```powershell
Get-ChildItem -Path .\export\*.eml |
    Get-MailHeaderAnalysis |
    Select-Object Source, Subject, Spf, Dkim, Dmarc, AuthTrust, HopCount,
        @{ Name = 'Findings'; Expression = { ($_.Findings.Code | Sort-Object -Unique) -join ',' } } |
    Export-Csv -Path .\auswertung.csv -NoTypeInformation -Encoding UTF8
```

Wertet alle Nachrichten eines Ordners aus und schreibt eine CSV-Datei mit einer Zeile je Nachricht. Die berechnete Spalte fasst die Finding-Codes zusammen.

| Option | Wirkung |
|---|---|
| `Get-ChildItem -Path .\export\*.eml` | Liefert die Dateiobjekte; `-Path` übernimmt deren Eigenschaft `FullName` |
| `Select-Object … @{ Name; Expression }` | Berechnete Spalte, die alle Finding-Codes zu einer Zeichenkette zusammenfasst |
| `Export-Csv -NoTypeInformation -Encoding UTF8` | CSV ohne Typkopfzeile, UTF-8 für Umlaute in Betreffzeilen |

#### Beispiel 6

```powershell
Get-ChildItem .\export\*.eml |
    Get-MailHeaderAnalysis |
    Where-Object AuthTrust -eq 'Unmatched' |
    Select-Object Source, AuthServId, DeliveredBy
```

Listet Nachrichten, deren `Authentication-Results`-Zeile nicht vom zustellenden System stammt. Bei Phishing-Analysen ist das ein schneller erster Filter.

#### Beispiel 7

```powershell
$a = Get-MailHeaderAnalysis -Path .\nachricht.eml
$a.Hops[$a.SlowestHopIndex - 1] | Format-List Index, FromHost, ByHost, Delay
```

Zeigt die Station mit der grössten Verzögerung. `SlowestHopIndex` ist 1-basiert, das Array `Hops` 0-basiert.

#### Beispiel 8

```powershell
Get-MailHeaderAnalysis -Path .\nachricht.eml | ConvertTo-Json -Depth 6 | Set-Content .\analyse.json
```

Schreibt die vollständige Analyse als JSON. `-Depth 6` ist nötig, weil `Hops`, `DkimSignatures` und `Exchange` verschachtelte Objekte sind; der Standardwert 2 würde sie nur als Typnamen ausgeben.

### Parameter

#### -Header

Der Header als Text. Der Parameter nimmt einen einzelnen String mit dem ganzen Header oder mehrere Strings entgegen; Pipeline-Eingaben werden gesammelt und am Ende zu einem Header zusammengesetzt, deshalb funktioniert `Get-Content datei | Get-MailHeaderAnalysis` ohne `-Raw`. Sollen mehrere Header getrennt ausgewertet werden, verwenden Sie `-Path` mit mehreren Dateien.

Aliase: `Text`, `Raw`, `InputObject`

| Parametereigenschaft | Wert |
|---|---|
| Typ | `String[]` |
| Standardwert | Kein |
| Platzhalter unterstützt | Nein |

| Parametersatz Text | Wert |
|---|---|
| Position | 0 |
| Pflicht | Ja |
| Wert aus Pipeline | Ja |
| Wert aus Pipeline nach Eigenschaftsname | Nein |

#### -Path

Pfad zu einer Datei, die den Header oder eine vollständige `.eml`-Nachricht enthält. Relative Pfade werden gegen das aktuelle Verzeichnis aufgelöst. Die Datei wird mit `[System.IO.File]::ReadAllText` gelesen: Eine Byte Order Mark wird berücksichtigt, ohne BOM gilt UTF-8. Für jede Datei entsteht ein eigenes Ergebnisobjekt; fehlende Dateien erzeugen einen nicht abbrechenden Fehler.

Aliase: `FullName`, `PSPath`, `LiteralPath`

| Parametereigenschaft | Wert |
|---|---|
| Typ | `String[]` |
| Standardwert | Kein |
| Platzhalter unterstützt | Nein |

| Parametersatz Path | Wert |
|---|---|
| Position | Benannt |
| Pflicht | Ja |
| Wert aus Pipeline | Nein |
| Wert aus Pipeline nach Eigenschaftsname | Ja |

#### -FromClipboard

Liest den Header mit `Get-Clipboard -Raw` aus der Zwischenablage. Der Parameter ist nur unter Windows verfügbar; auf Linux und macOS bricht das Cmdlet mit einer Fehlermeldung ab, ebenso bei leerer Zwischenablage.

| Parametereigenschaft | Wert |
|---|---|
| Typ | `SwitchParameter` |
| Standardwert | `False` |
| Platzhalter unterstützt | Nein |

| Parametersatz Clipboard | Wert |
|---|---|
| Position | Benannt |
| Pflicht | Ja |
| Wert aus Pipeline | Nein |
| Wert aus Pipeline nach Eigenschaftsname | Nein |

### Eingaben

`System.String`: Headerzeilen oder der ganze Header, an `-Header`.

`System.IO.FileInfo`: Dateiobjekte von `Get-ChildItem`, deren `FullName` an `-Path` gebunden wird.

### Ausgaben

`MailHeaderAnalyzer.Analysis`: ein Objekt je ausgewertetem Header. Die Eigenschaften sind im Abschnitt [Ausgabeobjekt](#ausgabeobjekt) beschrieben.

### Hinweise

Der Analysetext (Erklärungen in `Findings`, `CompAuthReasonMeaning` und den Bedeutungsfeldern) ist englisch, damit er unverändert in internationale Tickets passt. Die Ausgabe der Standardansicht lässt sich mit `Format-List *` vollständig anzeigen.

## ConvertTo-MailHeaderReport

Erzeugt aus einem Analyseobjekt einen Bericht als Markdown oder Text.

### Syntax

```powershell
ConvertTo-MailHeaderReport
    [-Analysis] <Object>
    [-Format <String>]
    [<CommonParameters>]
```

### Beschreibung

Der Bericht umfasst Betreff, Absender, Datum und Message-ID, die Authentifizierungsergebnisse mit Herkunftsangabe und Alignment, alle Findings, die Zustellkette, die Exchange-Klassifizierung und die Werte der Spamfilter. Unicode-Steuerzeichen für die Schreibrichtung bleiben im Bericht als `<U+...>` sichtbar, damit sie über den Bericht nicht in ein Ticketsystem gelangen. Die letzte Zeile nennt die Modulversion.

### Beispiele

#### Beispiel 1

```powershell
Get-MailHeaderAnalysis -Path .\nachricht.eml | ConvertTo-MailHeaderReport | Set-Clipboard
```

Erzeugt einen Markdown-Bericht und legt ihn in die Zwischenablage.

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

#### Beispiel 2

```powershell
Get-MailHeaderAnalysis -FromClipboard | ConvertTo-MailHeaderReport -Format Text
```

Gibt den Bericht als Text ohne Markdown-Auszeichnung aus, etwa für E-Mails oder Konsolenprotokolle.

#### Beispiel 3

```powershell
Import-Module MailHeaderAnalyzer
Get-MailHeaderAnalysis -Path 'C:\Temp\bounce.eml' | ConvertTo-MailHeaderReport -Format Text
```

Verwendung in der Exchange Management Shell. Ist `$env:PSModulePath` dort durch Gruppenrichtlinien eingeschränkt, laden Sie das Modul mit `Import-Module` und dem vollständigen Pfad zur `.psd1`-Datei.

### Parameter

#### -Analysis

Das Analyseobjekt von `Get-MailHeaderAnalysis`. Andere Objekttypen weist das Cmdlet mit einem Bindungsfehler ab.

| Parametereigenschaft | Wert |
|---|---|
| Typ | `MailHeaderAnalyzer.Analysis` |
| Standardwert | Kein |
| Platzhalter unterstützt | Nein |

| Parametersatz (alle) | Wert |
|---|---|
| Position | 0 |
| Pflicht | Ja |
| Wert aus Pipeline | Ja |
| Wert aus Pipeline nach Eigenschaftsname | Nein |

#### -Format

Das Ausgabeformat. Gültige Werte:

- `Markdown`: Überschriften, Aufzählungen und die Zustellkette als Tabelle. Standard.
- `Text`: Überschriften in Grossbuchstaben, eingerückte Zeilen, die Zustellkette als nummerierte Liste.

| Parametereigenschaft | Wert |
|---|---|
| Typ | `String` |
| Zulässige Werte | `Markdown`, `Text` |
| Standardwert | `Markdown` |
| Platzhalter unterstützt | Nein |

| Parametersatz (alle) | Wert |
|---|---|
| Position | Benannt |
| Pflicht | Nein |
| Wert aus Pipeline | Nein |
| Wert aus Pipeline nach Eigenschaftsname | Nein |

### Eingaben

`MailHeaderAnalyzer.Analysis`: das Ergebnis von `Get-MailHeaderAnalysis`.

### Ausgaben

`System.String`: der Bericht, ein String je Analyseobjekt.

## Ausgabeobjekt

`Get-MailHeaderAnalysis` gibt je Eingabe ein Objekt vom Typ `MailHeaderAnalyzer.Analysis` zurück. Die Standardansicht zeigt die Zusammenfassung aus Beispiel 1; alle Eigenschaften sind über `Select-Object`, `Format-List *` oder `ConvertTo-Json` zugänglich.

### MailHeaderAnalyzer.Analysis

| Eigenschaft | Typ | Inhalt |
|---|---|---|
| `Source` | String | Dateipfad, `Clipboard` oder `Text` |
| `Subject` | String | Betreff, RFC 2047 dekodiert |
| `From`, `ReplyTo`, `ReturnPath` | Adressobjekt | `Name`, `Address`, `Domain`, `Display`; `$null`, wenn das Feld fehlt |
| `Date` | DateTime (UTC) | Wert des `Date`-Felds |
| `MessageId` | String | `Message-ID` |
| `MailFromDomain` | String | Envelope-Absenderdomain aus `smtp.mailfrom` der SPF-Prüfung, sonst aus `Return-Path` |
| `Spf`, `Dkim`, `Dmarc`, `Arc`, `CompAuth` | String | Ergebnis laut massgebender `Authentication-Results`-Zeile (`pass`, `fail`, `none`, `softfail` und weitere); `$null`, wenn nicht geprüft |
| `CompAuthReason`, `CompAuthReasonMeaning` | String | Reason-Code der zusammengesetzten Authentifizierung von Microsoft 365 und seine Bedeutung |
| `AuthTrust` | String | `Matched`, `Unmatched`, `Absent` oder `None`, siehe [AuthTrust](#authtrust-herkunft-der-prüfergebnisse) |
| `AuthServId` | String | authserv-id der massgebenden Prüfzeile |
| `AuthenticationResults` | Objekt[] | Alle `Authentication-Results`-Zeilen mit `AuthServId`, `Methods`, `Trust`, `Raw` |
| `ReceivedSpf` | Objekt | Die `Received-SPF`-Zeile mit `Result` und `Properties` |
| `SpfAlignment`, `DkimAlignment` | String | `Strict`, `Relaxed`, `None` oder `$null` |
| `Hops` | Hop[] | Zustellkette in chronologischer Reihenfolge, siehe [Hop-Objekt](#mailheaderanalyzerhop) |
| `HopCount`, `TotalDuration`, `SlowestHopIndex`, `HasClockSkew` | Int, TimeSpan, Int, Bool | Kennzahlen der Kette |
| `DeliveredBy` | String | `by`-Host der jüngsten `Received`-Zeile, also die zustellende Station |
| `DkimSignatures` | Objekt[] | Je Signatur `Domain`, `Selector`, `Algorithm`, `Canonicalization`, `SignedHeaders`, `BodyLength`, `Timestamp`, `Expires`, `ReceiverResult`, `Tags` |
| `ArcChain`, `ArcValid` | Objekt[], Bool | ARC-Instanzen mit `Instance`, `SealDomain`, `ChainValidation`, `Methods`; `ArcValid` ist `$null` ohne ARC-Kette |
| `Exchange` | Objekt | Hybrid-Klassifizierung von Exchange Online, siehe [Exchange-Objekt](#mailheaderanalyzerexchangeclassification); `$null` ohne entsprechende Header |
| `Spam` | Objekt | Bewertungen der Spamfilter, siehe [Spam-Objekt](#mailheaderanalyzerspamassessment); `$null` ohne entsprechende Header |
| `List` | Objekt | `ListId`, `Unsubscribe`, `OneClick` (RFC 8058); `$null` ohne Listen-Header |
| `Findings` | Finding[] | Auffälligkeiten mit `Severity`, `Code`, `Message`, siehe [Findings](#findings) |
| `Fields` | Objekt[] | Alle Felder mit `Name`, `Value` (entfaltet) und `Raw` |
| `HadBody` | Bool | Ob nach dem Header ein Nachrichtentext folgte |

### MailHeaderAnalyzer.Hop

Jeder Eintrag in `Hops` entspricht einer `Received`-Zeile. Die Reihenfolge ist chronologisch, also umgekehrt zur Reihenfolge im Header.

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

### AuthTrust: Herkunft der Prüfergebnisse

Eine `Authentication-Results`-Zeile kann jeder Absender selbst in eine Nachricht schreiben. Nach RFC 8601, Abschnitt 5, ist nur die Zeile der empfangenden Organisation massgebend, und deren authserv-id muss sich einer Station der Zustellkette zuordnen lassen. Das Cmdlet vergleicht die authserv-id jeder Zeile mit den `by`-Hosts der `Received`-Kette (gleiche Domain oder Subdomain, immer an der Punktgrenze, kein Teilstring).

| Wert | Bedeutung |
|---|---|
| `Matched` | Die authserv-id kommt als `by`-Host in der Kette vor. Nur solche Zeilen fliessen in `Spf`, `Dkim`, `Dmarc` ein, sobald mindestens eine existiert |
| `Unmatched` | Die authserv-id kommt in der Kette nicht vor. Die Ergebnisse werden angezeigt, gelten aber als unbelegte Behauptung; das Finding `AuthUnverified` weist darauf hin |
| `Absent` | Die Zeile trägt keine authserv-id. Microsoft 365 schreibt seine Prüfzeile in dieser Form, sie beginnt direkt mit `spf=` |
| `None` | Keine Prüfzeile vorhanden |

Enthält der Header Prüfzeilen mehrerer Herkünfte, meldet das Finding `AuthMixedOrigins` den Sachverhalt. Fehlt eine Prüfzeile, aber eine `Received-SPF`-Zeile ist vorhanden, wird deren Ergebnis als `Spf` übernommen und mit `ReceivedSpfOnly` gekennzeichnet.

### DMARC-Alignment

`SpfAlignment` vergleicht die Envelope-Absenderdomain mit der `From`-Domain, `DkimAlignment` die `d=`-Domain der geprüften Signatur mit der `From`-Domain. `Strict` bedeutet identische Domain, `Relaxed` dieselbe Organisationsdomain, `None` keine Übereinstimmung. Die Organisationsdomain wird heuristisch bestimmt: die letzten zwei Labels, bei bekannten mehrteiligen Endungen wie `co.uk` oder `com.au` die letzten drei. Eine vollständige Public Suffix List ist nicht enthalten.

### MailHeaderAnalyzer.ExchangeClassification

Enthält der Header Felder `X-MS-Exchange-Organization-*`, `X-OriginatorOrg` oder `X-MS-Exchange-CrossTenant-*`, füllt das Cmdlet die Eigenschaft `Exchange`. Die Bedeutungen folgen dem Artikel „Demystifying hybrid mail flow“ des Exchange-Teams; die Hintergründe stehen im Artikel [Exchange-Hybrid-Header: intern oder extern?](/blog/exchange-hybrid-header-intern-extern).

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

### MailHeaderAnalyzer.SpamAssessment

Die Eigenschaft `Spam` fasst die Bewertungen der bekannten Filter zusammen. Die Werte stammen aus fremden Systemen und werden dekodiert, aber nicht bewertet.

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

Die vollständige Liste der `compauth`-Reason-Codes steht im Artikel [Microsoft 365 compauth: Reason-Codes](/blog/microsoft-365-compauth-reason-codes).

### Findings

Auffälligkeiten liefert das Cmdlet als Objekte in `Findings`, jeweils mit `Severity` (`Info`, `Warning`, `Fail`), einem stabilen `Code` für Filter und Skripte sowie einer Erklärung in `Message`.

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

3.  [Microsoft Learn: Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2?view=exchange-ps): Vorlage für die Gliederung dieser Referenz (Syntax, Beschreibung, Beispiele, Parametereigenschaften).

4.  [RFC 8601: Message Header Field for Indicating Message Authentication Status](https://www.rfc-editor.org/rfc/rfc8601): Aufbau von `Authentication-Results` und die Regel, dass nur die Zeile der empfangenden Organisation massgebend ist (Abschnitt 5).

5.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): Aufbau der `Received`-Zeilen und die Empfehlung einer Obergrenze als Schleifenschutz (Abschnitt 6.3).

6.  [RFC 3848: ESMTP and LMTP Transmission Types Registration](https://www.rfc-editor.org/rfc/rfc3848): Die `with`-Werte `ESMTPS`, `ESMTPA` und `ESMTPSA`.

7.  [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://www.rfc-editor.org/rfc/rfc6376): Tags der `DKIM-Signature`, Pflicht zur Signierung von `From`.

8.  [RFC 8301: Cryptographic Algorithm and Key Usage Update to DKIM](https://www.rfc-editor.org/rfc/rfc8301): Einstufung von `rsa-sha1` als veraltet.

9.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc7489): Strict und Relaxed Alignment.

10.  [RFC 8617: The Authenticated Received Chain (ARC) Protocol](https://www.rfc-editor.org/rfc/rfc8617): `ARC-Seal`, `ARC-Authentication-Results` und die `cv=`-Kettenprüfung.

11.  [Microsoft Learn: Anti-spam message headers in Microsoft 365](https://learn.microsoft.com/en-us/defender-office-365/message-headers-eop-mdo): Bedeutung von SCL, BCL, CAT, SFV, IPV und der `compauth`-Reason-Codes.

12.  [Exchange Team Blog: Demystifying hybrid mail flow](https://techcommunity.microsoft.com/blog/exchange/demystifying-hybrid-mail-flow-when-is-a-message-internal/1420838): Herkunft der Bedeutungen von MessageDirectionality, AuthAs und AuthMechanism.

13.  [Header-Analyzer auf rafaelpfister.ch](/tools/header-analyzer): Die Browser-Fassung mit derselben Auswertungslogik.
