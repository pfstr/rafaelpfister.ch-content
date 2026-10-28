---
title: "MailHeaderAnalyzer: analysera e-posthuvuden i PowerShell utan nätverksåtkomst"
navTitle: "MailHeaderAnalyzer (PowerShell)"
description: "Referens för PowerShell-modulen MailHeaderAnalyzer: parameteröversikt, syntax, beskrivning, exempel och parameteregenskaper för Get-MailHeaderAnalysis och ConvertTo-MailHeaderReport, samt utdataobjektet med leveranskedja, autentiseringsresultat, Exchange Online-klassificering och alla Finding-koder."
date: "2026-09-24"
kategorie: "SMTP och e-postflöde"
timeToRead: "16 min lästid"
themen:
  - smtp-mailflow
  - microsoft-365-exchange
hauptthema: "smtp-mailflow"
produkte:
  - "exchange-online"
protokolle:
  - "powershell"
  - "mail-auth"
  - "smtp"
  - "troubleshooting"
related:
  - e-mail-header-analysieren-ohne-upload
  - microsoft-365-compauth-reason-codes
  - exchange-hybrid-header-intern-extern
slug: "mailheaderanalyzer-analysera-e-posthuvuden-i-powershell-utan-natverksatkomst"
translationId: "article-041d7b2f9615f670"
translationOf: mailheaderanalyzer-powershell-modul
translationSourceHash: 7f1ed02181dcd5665641de26ab47fc5acaeaa73a230c60093063ab8bb99e15d3
translationModel: gpt-5.6-terra
translatedAt: 2026-10-27T09:34:11.731Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/mailheaderanalyzer-analysera-e-posthuvuden-i-powershell-utan-natverksatkomst
---

# MailHeaderAnalyzer: analysera e-posthuvuden i PowerShell utan nätverksåtkomst

MailHeaderAnalyzer är en PowerShell-modul med två cmdletar. `Get-MailHeaderAnalysis` analyserar en e-posts header: leveranskedja med fördröjningar och TLS-information, SPF-, DKIM-, DMARC- och ARC-resultat inklusive ursprungskontroll mot authserv-id för din gateway, DMARC-alignment, hybridklassificeringen i Exchange Online, bedömningar från Microsoft Defender, SpamAssassin och Rspamd samt avvikelser som dubbla `From`-rader eller Unicode-kontrolltecken. `ConvertTo-MailHeaderReport` skapar en rapport för ärenden. Modulen arbetar helt offline: inga DNS-frågor, inga HTTP-anslutningar. Den är kommandoradsversionen av [Header Analyzer på den här webbplatsen](/tools/header-analyzer) och använder samma analyslogik.

| | |
|---|---|
| Modul | [MailHeaderAnalyzer i PowerShell Gallery](https://www.powershellgallery.com/packages/MailHeaderAnalyzer) |
| Källkod | [pfstr/MailHeaderAnalyzer på GitHub](https://github.com/pfstr/MailHeaderAnalyzer), MIT-licens |
| Gäller för | Windows PowerShell 5.1, PowerShell 7.x; Windows, Linux, macOS; Exchange Management Shell |
| Cmdletar | `Get-MailHeaderAnalysis`, `ConvertTo-MailHeaderReport` |

## Parameteröversikt

| Cmdlet | Parameter | Typ | Obligatorisk | Pipeline | Effekt |
|---|---|---|---|---|---|
| `Get-MailHeaderAnalysis` | `-Header` | `String[]` | Ja (parameteruppsättning Text) | Ja, efter värde | Headern som text. Rader från pipelinen sätts samman till en header |
| `Get-MailHeaderAnalysis` | `-Path` | `String[]` | Ja (parameteruppsättning Path) | Ja, efter egenskapsnamn | Fil med headern eller ett komplett `.eml`-meddelande; tar emot objekt från `Get-ChildItem` |
| `Get-MailHeaderAnalysis` | `-FromClipboard` | `Switch` | Ja (parameteruppsättning Clipboard) | Nej | Läser headern från urklipp (endast Windows) |
| `Get-MailHeaderAnalysis` | `-TrustedAuthServId` | `String[]` | Nej | Nej | authserv-id:n för din inkommande gateway; endast kontrollrader med ett av dessa ID:n räknas som styrkta |
| `ConvertTo-MailHeaderReport` | `-Analysis` | `MailHeaderAnalyzer.Analysis` | Ja | Ja, efter värde | Resultatobjektet från `Get-MailHeaderAnalysis` |
| `ConvertTo-MailHeaderReport` | `-Format` | `String` | Nej | Nej | `Markdown` (standard) eller `Text` |

Båda cmdletarna stöder de vanliga parametrarna `-Verbose`, `-ErrorAction`, `-ErrorVariable`, `-OutVariable` och övriga från [about_CommonParameters](https://learn.microsoft.com/de-de/powershell/module/microsoft.powershell.core/about/about_commonparameters).

## Installation

```powershell
Install-Module -Name MailHeaderAnalyzer -Scope CurrentUser
```

| Alternativ | Effekt |
|---|---|
| `-Name MailHeaderAnalyzer` | Modulens namn i PowerShell Gallery |
| `-Scope CurrentUser` | Installerar i användarens modulkatalog, utan administratörsbehörighet |

På ett system utan internetåtkomst laddar du ned modulen på en annan dator med `Save-Module -Name MailHeaderAnalyzer -Path C:\Temp` och kopierar mappen `MailHeaderAnalyzer` till en katalog från `$env:PSModulePath`, till exempel `%USERPROFILE%\Documents\WindowsPowerShell\Modules` (Windows PowerShell 5.1) eller `%USERPROFILE%\Documents\PowerShell\Modules` (PowerShell 7). Uppdateringar hämtas med `Update-Module -Name MailHeaderAnalyzer`, och den installerade versionen visas med `Get-Module -Name MailHeaderAnalyzer -ListAvailable`.

## Get-MailHeaderAnalysis

Analyserar en e-posts header och returnerar ett analysobjekt.

### Syntax

#### Text (standard)

```powershell
Get-MailHeaderAnalysis
    [-Header] <String[]>
    [-TrustedAuthServId <String[]>]
    [<CommonParameters>]
```

#### Path

```powershell
Get-MailHeaderAnalysis
    -Path <String[]>
    [-TrustedAuthServId <String[]>]
    [<CommonParameters>]
```

#### Clipboard

```powershell
Get-MailHeaderAnalysis
    -FromClipboard
    [-TrustedAuthServId <String[]>]
    [<CommonParameters>]
```

### Beskrivning

Cmdleten delar upp råheadern i fält, vecklar ut RFC 5322-radbrytningar och avkodar RFC 2047-värden i ämne och adresser. Från `Received`-raderna skapar den leveranskedjan i kronologisk ordning, beräknar fördröjningen per station och läser TLS-version, chiffer och protokollklass enligt RFC 3848. Från `Authentication-Results`, `Received-SPF`, `DKIM-Signature` och ARC-kedjan fastställer den autentiseringsresultaten och tilldelar varje kontrollrad ett ursprung (RFC 8601, avsnitt 5): styrkt om dess authserv-id finns i `-TrustedAuthServId`, annars endast plausibelt eller obestyrkt, se [AuthTrust](#authtrust-herkunft-der-prüfergebnisse). Därtill kommer DMARC-alignment, hybridklassificeringen i Exchange Online, spamfiltrens bedömningar och en lista över avvikelser.

Cmdleten utför inga DNS-frågor och öppnar ingen nätverksanslutning. `Spf`, `Dkim`, `Dmarc` och `Arc` är därför alltid den mottagande serverns bedömning. DKIM- och ARC-signaturer beräknas inte kryptografiskt på nytt; för ARC-kedjan kontrollerar cmdleten endast strukturen (`ArcStructure`).

Indata läses tolerant: En tom rad avslutar headern, och efterföljande meddelandetext ignoreras. Rader utan fältnamn och utan inledande blanksteg, såsom de som uppstår vid kopiering från klientdialoger, hör till föregående fält. En mbox-avgränsningsrad `From ...` före det första fältet hoppas över och en Byte Order Mark tas bort. Högst 200 `Received`-rader analyseras, räknat från leveransen.

### Exempel

#### Exempel 1

```powershell
Get-MailHeaderAnalysis -FromClipboard
```

Analyserar headern som finns i urklipp. I Outlook för Windows hittar du headern under Arkiv, Egenskaper, Internetheaders; i Outlook på webben finns den i meddelandealternativen under ”Visa meddelandeinformation”.

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

#### Exempel 2

```powershell
Get-MailHeaderAnalysis -Path .\nachricht.eml
```

Analyserar ett sparat meddelande. Filen får innehålla enbart headern eller hela meddelandet; meddelandetexten ignoreras.

#### Exempel 3

```powershell
(Get-MailHeaderAnalysis -Path .\nachricht.eml).Hops
```

Visar leveranskedjan som en tabell, med första hoppet först.

```text
#   From                 IP              By                                     Protocol   TLS      Time (UTC)           Delay
-   ----                 --              --                                     --------   ---      ----------           -----
1   client.example.net   198.51.100.34   mail.example.org                       ESMTPSA    -        2026-08-03 09:14:28  -
2   mail.example.org     203.0.113.25    mx.eur02.prod.protection.outlook.com   Microsoft… TLS 1.3  2026-08-03 09:15:09  41 s
3   AM0EUR02FT056.eop…   -               ZR0P278MB0570.CHEP278.PROD.OUTLOOK.COM Microsoft… TLS 1.2  2026-08-03 09:15:10  1 s
```

#### Exempel 4

```powershell
Get-Content -Path .\header.txt | Get-MailHeaderAnalysis | Select-Object -ExpandProperty Findings
```

Läser headern rad för rad från en textfil och visar endast avvikelserna. `-Raw` för `Get-Content` behövs inte, eftersom cmdleten själv sätter samman raderna.

#### Exempel 5

```powershell
Get-ChildItem -Path .\export\*.eml |
    Get-MailHeaderAnalysis |
    Select-Object Source, Subject, Spf, Dkim, Dmarc, AuthTrust, HopCount,
        @{ Name = 'Findings'; Expression = { ($_.Findings.Code | Sort-Object -Unique) -join ',' } } |
    Export-Csv -Path .\auswertung.csv -NoTypeInformation -Encoding UTF8
```

Analyserar alla meddelanden i en mapp och skriver en CSV-fil med en rad per meddelande. Den beräknade kolumnen sammanfattar Finding-koderna.

| Alternativ | Effekt |
|---|---|
| `Get-ChildItem -Path .\export\*.eml` | Levererar filobjekten; `-Path` tar över deras egenskap `FullName` |
| `Select-Object … @{ Name; Expression }` | Beräknad kolumn som sammanfattar alla Finding-koder till en sträng |
| `Export-Csv -NoTypeInformation -Encoding UTF8` | CSV utan typheader, UTF-8 för diakritiska tecken i ämnesrader |

#### Exempel 6

```powershell
Get-ChildItem .\export\*.eml |
    Get-MailHeaderAnalysis -TrustedAuthServId 'mx.example.org' |
    Where-Object AuthTrust -ne 'Trusted' |
    Select-Object Source, AuthTrust, AuthServId, DeliveredBy
```

Listar meddelanden vars kontrollresultat inte kommer från den egna inkommande gatewayen `mx.example.org`. Vid phishinganalyser är detta ett första filter. Utan `-TrustedAuthServId` kan du bara filtrera efter `Unmatched`; en förfalskning som dessutom innehåller en matchande `Received`-rad visas då som `Matched` och passerar filtret.

<details class="options-details">
<summary>Förklaring av alternativen</summary>

| Alternativ | Effekt |
|---|---|
| `-TrustedAuthServId 'mx.example.org'` | authserv-id som din gateway skriver i `Authentication-Results`; exakt jämförelse, utan subdomäner |
| `Where-Object AuthTrust -ne 'Trusted'` | Behåller alla meddelanden där den avgörande kontrollraden saknar en betrodd authserv-id |
| `Select-Object Source, AuthTrust, AuthServId, DeliveredBy` | Fil, ursprungsnivå, authserv-id för kontrollraden och levererande station |

</details>

#### Exempel 7

```powershell
$a = Get-MailHeaderAnalysis -Path .\nachricht.eml
$a.Hops[$a.SlowestHopIndex - 1] | Format-List Index, FromHost, ByHost, Delay
```

Visar stationen med längst fördröjning. `SlowestHopIndex` är 1-baserat, medan arrayen `Hops` är 0-baserad.

#### Exempel 8

```powershell
Get-MailHeaderAnalysis -Path .\nachricht.eml | ConvertTo-Json -Depth 6 | Set-Content .\analyse.json
```

Skriver den fullständiga analysen som JSON. `-Depth 6` krävs eftersom `Hops`, `DkimSignatures` och `Exchange` är kapslade objekt; standardvärdet 2 skulle endast visa dem som typnamn.

### Parametrar

#### -Header

Headern som text. Parametern tar emot en enskild sträng med hela headern eller flera strängar; pipelineindata samlas in och sätts samman till en header i slutet, därför fungerar `Get-Content datei | Get-MailHeaderAnalysis` utan `-Raw`. Om flera headers ska analyseras separat använder du `-Path` med flera filer.

Alias: `Text`, `Raw`, `InputObject`

| Parameteregenskap | Värde |
|---|---|
| Typ | `String[]` |
| Standardvärde | Inget |
| Jokertecken stöds | Nej |

| Parameteruppsättning Text | Värde |
|---|---|
| Position | 0 |
| Obligatorisk | Ja |
| Värde från pipeline | Ja |
| Värde från pipeline efter egenskapsnamn | Nej |

#### -Path

Sökväg till en fil som innehåller headern eller ett fullständigt `.eml`-meddelande. Relativa sökvägar löses mot den aktuella katalogen. Filen läses med `[System.IO.File]::ReadAllText`: En Byte Order Mark beaktas, utan BOM används UTF-8. Ett eget resultatobjekt skapas för varje fil; saknade filer ger ett icke-avbrytande fel.

Alias: `FullName`, `PSPath`, `LiteralPath`

| Parameteregenskap | Värde |
|---|---|
| Typ | `String[]` |
| Standardvärde | Inget |
| Jokertecken stöds | Nej |

| Parameteruppsättning Path | Värde |
|---|---|
| Position | Namngiven |
| Obligatorisk | Ja |
| Värde från pipeline | Nej |
| Värde från pipeline efter egenskapsnamn | Ja |

#### -FromClipboard

Läser headern med `Get-Clipboard -Raw` från urklipp. Parametern är endast tillgänglig i Windows; i Linux och macOS avbryts cmdleten med ett felmeddelande, liksom om urklipp är tomt.

| Parameteregenskap | Värde |
|---|---|
| Typ | `SwitchParameter` |
| Standardvärde | `False` |
| Jokertecken stöds | Nej |

| Parameteruppsättning Clipboard | Värde |
|---|---|
| Position | Namngiven |
| Obligatorisk | Ja |
| Värde från pipeline | Nej |
| Värde från pipeline efter egenskapsnamn | Nej |

#### -TrustedAuthServId

De authserv-id:n som din inkommande gateway skriver i `Authentication-Results`, till exempel `mx.example.org`. Cmdleten jämför exakt, utan hänsyn till stora och små bokstäver och utan subdomäner. Kontrollrader med något av dessa ID:n får `AuthTrust = Trusted`, och endast de tas sedan med i `Spf`, `Dkim`, `Dmarc` och `Arc`. Detsamma gäller `receiver=` i en `Received-SPF`-rad.

Resultatet är bara så tillförlitligt som gatewayen: Den måste ta bort inkommande `Authentication-Results`-rader som gör anspråk på dess egen authserv-id (RFC 8601, avsnitt 5). Huruvida den gör det går inte att utläsa ur en header. Om två rader med betrott ID motsäger varandra rapporterar cmdleten `AuthTrustedConflict`. Microsoft 365 skriver ingen authserv-id i sin kontrollrad; om Exchange Online tar emot din e-post lämnar du parametern utelämnad.

För en hel session kan värdet sparas som standard, exempelvis i PowerShell-profilen:

```powershell
$PSDefaultParameterValues['Get-MailHeaderAnalysis:TrustedAuthServId'] = 'mx.example.org'
```

| Parameteregenskap | Värde |
|---|---|
| Typ | `String[]` |
| Standardvärde | Inget |
| Jokertecken stöds | Nej |

| Parameteruppsättning (alla) | Värde |
|---|---|
| Position | Namngiven |
| Obligatorisk | Nej |
| Värde från pipeline | Nej |
| Värde från pipeline efter egenskapsnamn | Nej |

### Indata

`System.String`: Headerrader eller hela headern, till `-Header`.

`System.IO.FileInfo`: Filobjekt från `Get-ChildItem`, vars `FullName` binds till `-Path`.

### Utdata

`MailHeaderAnalyzer.Analysis`: ett objekt per analyserad header. Egenskaperna beskrivs i avsnittet [Utdataobjekt](#ausgabeobjekt).

### Anmärkningar

Analystexten (förklaringar i `Findings`, `CompAuthReasonMeaning` och betydelsefälten) är på engelska, så att den kan användas oförändrad i internationella ärenden. Utdata i standardvyn kan visas fullständigt med `Format-List *`.

## ConvertTo-MailHeaderReport

Skapar en rapport som Markdown eller text från ett analysobjekt.

### Syntax

```powershell
ConvertTo-MailHeaderReport
    [-Analysis] <Object>
    [-Format <String>]
    [<CommonParameters>]
```

### Beskrivning

Rapporten omfattar ämne, avsändare, datum och Message-ID, autentiseringsresultat med ursprungsangivelse och alignment, alla findings, leveranskedjan, Exchange-klassificeringen och spamfiltrens värden. Unicode-kontrolltecken för skrivriktning förblir synliga i rapporten som `<U+...>`, så att de inte förs vidare till ett ärendesystem via rapporten. Den sista raden anger modulversionen.

### Exempel

#### Exempel 1

```powershell
Get-MailHeaderAnalysis -Path .\nachricht.eml | ConvertTo-MailHeaderReport | Set-Clipboard
```

Skapar en Markdown-rapport och lägger den i urklipp.

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

#### Exempel 2

```powershell
Get-MailHeaderAnalysis -FromClipboard | ConvertTo-MailHeaderReport -Format Text
```

Ger ut rapporten som text utan Markdown-markering, till exempel för e-post eller konsolloggar.

#### Exempel 3

```powershell
Import-Module MailHeaderAnalyzer
Get-MailHeaderAnalysis -Path 'C:\Temp\bounce.eml' | ConvertTo-MailHeaderReport -Format Text
```

Användning i Exchange Management Shell. Om `$env:PSModulePath` där begränsas genom grupprinciper laddar du modulen med `Import-Module` och den fullständiga sökvägen till filen `.psd1`.

### Parametrar

#### -Analysis

Analysobjektet från `Get-MailHeaderAnalysis`. Andra objekttyper avvisas av cmdleten med ett bindningsfel.

| Parameteregenskap | Värde |
|---|---|
| Typ | `MailHeaderAnalyzer.Analysis` |
| Standardvärde | Inget |
| Jokertecken stöds | Nej |

| Parameteruppsättning (alla) | Värde |
|---|---|
| Position | 0 |
| Obligatorisk | Ja |
| Värde från pipeline | Ja |
| Värde från pipeline efter egenskapsnamn | Nej |

#### -Format

Utdataformatet. Giltiga värden:

- `Markdown`: Rubriker, punktlistor och leveranskedjan som tabell. Standard.
- `Text`: Rubriker med versaler, indragna rader, leveranskedjan som numrerad lista.

| Parameteregenskap | Värde |
|---|---|
| Typ | `String` |
| Tillåtna värden | `Markdown`, `Text` |
| Standardvärde | `Markdown` |
| Jokertecken stöds | Nej |

| Parameteruppsättning (alla) | Värde |
|---|---|
| Position | Namngiven |
| Obligatorisk | Nej |
| Värde från pipeline | Nej |
| Värde från pipeline efter egenskapsnamn | Nej |

### Indata

`MailHeaderAnalyzer.Analysis`: resultatet från `Get-MailHeaderAnalysis`.

### Utdata

`System.String`: rapporten, en sträng per analysobjekt.

## Utdataobjekt

`Get-MailHeaderAnalysis` returnerar ett objekt av typen `MailHeaderAnalyzer.Analysis` per indata. Standardvyn visar sammanfattningen från exempel 1; alla egenskaper är åtkomliga via `Select-Object`, `Format-List *` eller `ConvertTo-Json`.

### MailHeaderAnalyzer.Analysis

| Egenskap | Typ | Innehåll |
|---|---|---|
| `Source` | String | Filsökväg, `Clipboard` eller `Text` |
| `Subject` | String | Ämne, RFC 2047-avkodat |
| `From`, `ReplyTo`, `ReturnPath` | Adressobjekt | `Name`, `Address`, `Domain`, `Display`; `$null` om fältet saknas |
| `Date` | DateTime (UTC) | Värdet för fältet `Date` |
| `MessageId` | String | `Message-ID` |
| `MailFromDomain` | String | Kuvertavsändardomän från `smtp.mailfrom` i SPF-kontrollen, annars från `Return-Path` |
| `Spf`, `Dkim`, `Dmarc`, `Arc`, `CompAuth` | String | Resultat enligt den avgörande `Authentication-Results`-raden (`pass`, `fail`, `none`, `softfail` och fler); `$null` om det inte har kontrollerats. `Arc` är mottagarens bedömning av ARC-kedjan |
| `CompAuthReason`, `CompAuthReasonMeaning` | String | Reason Code för den sammansatta autentiseringen i Microsoft 365 och dess innebörd |
| `AuthTrust` | String | `Trusted`, `Matched`, `Unmatched`, `Absent` eller `None`, se [AuthTrust](#authtrust-herkunft-der-prüfergebnisse) |
| `AuthServId` | String | authserv-id för den avgörande kontrollraden |
| `AuthenticationResults` | Objekt[] | Alla `Authentication-Results`-rader med `AuthServId`, `Methods`, `Trust`, `Raw` |
| `ReceivedSpf` | Objekt | `Received-SPF`-raden med `Result` och `Properties` |
| `SpfAlignment`, `DkimAlignment` | String | `Strict`, `Relaxed`, `None` eller `$null` |
| `Hops` | Hop[] | Leveranskedja i kronologisk ordning, se [Hop-objekt](#mailheaderanalyzerhop) |
| `HopCount`, `TotalDuration`, `SlowestHopIndex`, `HasClockSkew` | Int, TimeSpan, Int, Bool | Nyckeltal för kedjan |
| `DeliveredBy` | String | `by`-värden för den senaste `Received`-raden, alltså den levererande stationen |
| `DkimSignatures` | Objekt[] | Per signatur `Domain`, `Selector`, `Algorithm`, `Canonicalization`, `SignedHeaders`, `BodyLength`, `Timestamp`, `Expires`, `ReceiverResult`, `Tags` |
| `ArcChain` | Objekt[] | ARC-instanser med `Instance`, `SealDomain`, `ChainValidation`, `Methods` |
| `ArcStructure`, `ArcStructureIssues` | String, String[] | ARC-kedjans struktur: `Consistent`, `Inconsistent` eller `$null` utan ARC-headers, samt de avvikelser som hittades. Endast strukturkontroll, ingen signaturkontroll, se [ARC-kedja](#arc-kette-aufbau-und-urteil) |
| `Exchange` | Objekt | Hybridklassificering från Exchange Online, se [Exchange-objekt](#mailheaderanalyzerexchangeclassification); `$null` utan motsvarande headers |
| `Spam` | Objekt | Spamfiltrens bedömningar, se [Spam-objekt](#mailheaderanalyzerspamassessment); `$null` utan motsvarande headers |
| `List` | Objekt | `ListId`, `Unsubscribe`, `OneClick` (RFC 8058); `$null` utan listhuvuden |
| `Findings` | Finding[] | Avvikelser med `Severity`, `Code`, `Message`, se [Findings](#findings) |
| `Fields` | Objekt[] | Alla fält med `Name`, `Value` (utvecklat) och `Raw` |
| `HadBody` | Bool | Om meddelandetext följde efter headern |

### MailHeaderAnalyzer.Hop

Varje post i `Hops` motsvarar en `Received`-rad. Ordningen är kronologisk, alltså omvänd mot ordningen i headern.

| Egenskap | Innehåll |
|---|---|
| `Index` | Löpnummer, 1 = inlämning |
| `FromHost`, `Helo`, `ReverseDns`, `IPAddress`, `IsPrivateIP` | Information om det inlämnande systemet från delen `from`; IP-adressen kommer från hakparenteserna i kommentaren, rDNS-namnet från kommentaren före den |
| `ByHost`, `Software` | Mottagande system och dess programvara (kommentar efter `by`) |
| `Protocol`, `ProtocolClass` | `with`-värde och klass enligt RFC 3848: `TlsAuthenticated` (ESMTPSA), `Tls` (ESMTPS eller TLS-information finns), `Authenticated` (ESMTPA), `Plain`, `Http`, `Mapi`, `Local` |
| `TlsVersion`, `TlsCipher` | Från skrivsätten hos Microsoft (`version=TLS1_2, cipher=…`), Postfix (`using TLSv1.3 with cipher …`) och Exim |
| `Id`, `For`, `Via` | Andra `Received`-komponenter |
| `Date` | Tidsstämpel efter semikolonet, UTC |
| `Delay` | TimeSpan till föregående hopp; negativ vid klockförskjutning |
| `Provider` | Identifierad leverantör eller gateway utifrån värdnamnen (Microsoft 365, Google Workspace, SEPPmail, HIN, Proofpoint, Mimecast och fler) |
| `Attested` | `$true` endast för det sista hoppet: bara denna rad har skrivits av det mottagande systemet självt, alla rader under den fanns redan i meddelandet |
| `Raw` | Originalraden |

### AuthTrust: kontrollresultatens ursprung

En `Authentication-Results`-rad kan varje avsändare själv skriva in i ett meddelande, liksom en matchande `Received`-rad. Det går därför inte att styrka enbart utifrån headern vem som skrivit en kontrollrad. Enligt RFC 8601, avsnitt 5, är endast den rad avgörande vars authserv-id den mottagande organisationen känner igen som sitt eget, och den inkommande gatewayen måste ta bort inkommande rader med detta ID. Cmdleten implementerar denna regel med `-TrustedAuthServId`. Utan denna parameter jämför den authserv-id endast med `by`-värdarna i `Received`-kedjan (samma domän eller subdomän, alltid vid punktgränsen, ingen delsträng); detta är en rimlighetskontroll.

| Värde | Betydelse |
|---|---|
| `Trusted` | authserv-id finns i `-TrustedAuthServId`. Så snart en sådan rad finns tas endast rader på denna nivå med i `Spf`, `Dkim`, `Dmarc` och `Arc`. Tillförlitligt förutsatt att gatewayen tar bort främmande rader med detta ID |
| `Matched` | authserv-id förekommer som `by`-värd i kedjan. Plausibelt, men inte styrkt: En förfalskning kan innehålla en matchande `Received`-rad. Utan `-TrustedAuthServId` uppmärksammar finding `AuthPlausibleOnly` detta. Rader på denna nivå räknas om ingen `Trusted`-rad finns |
| `Unmatched` | authserv-id är varken betrott eller finns i kedjan. Resultaten visas, men betraktas som obestyrkta påståenden; finding `AuthUnverified` uppmärksammar detta |
| `Absent` | Raden saknar authserv-id. Microsoft 365 skriver sin kontrollrad i denna form, den börjar direkt med `spf=` |
| `None` | Ingen kontrollrad finns |

Om headern innehåller kontrollrader från flera ursprung rapporterar finding `AuthMixedOrigins` detta. Om en kontrollrad saknas men en `Received-SPF`-rad finns, övertas dess resultat som `Spf` och märks med `ReceivedSpfOnly`.

Med `-TrustedAuthServId` tillkommer två findings: `AuthNotTrusted` om ingen kontrollrad har ett betrott ID, och `AuthTrustedConflict` om två sådana rader rapporterar olika resultat för `spf`, `dmarc`, `arc` eller `compauth`. Det andra fallet innebär att minst en rad inte kommer från gatewayen och att gatewayen inte har tagit bort den. Cmdleten använder då den översta raden; vilken av de två som är äkta går inte att avgöra från headern. DKIM är undantaget från denna jämförelse eftersom flera signaturer legitimt kan ha olika resultat.

### ARC-kedja: struktur och bedömning

För ARC tillhandahåller cmdleten två separata uppgifter. `Arc` är resultatet `arc=` från den avgörande kontrollraden, alltså mottagarens bedömning som kontrollerat kedjans signaturer. `ArcStructure` är modulens egen kontroll och avser endast strukturen enligt RFC 8617: löpande instansnummer `i=1` till `i=n` (högst 50), exakt en `ARC-Seal`, en `ARC-Message-Signature` och en `ARC-Authentication-Results` per instans, samt `cv=none` för instans 1 och `cv=pass` för alla efterföljande. Modulen beräknar inte om signaturer; `Consistent` säger därför inget om huruvida kedjan är äkta. Avvikelser anges i `ArcStructureIssues` och i finding `ArcStructureInconsistent`.

Uppgifterna i `ARC-Authentication-Results` är påståenden från respektive vidarebefordran. Finding `DkimBrokenAfterForward` klassificerar därför ett DKIM-fel som en följd av vidarebefordran endast om mottagaren själv rapporterar `arc=pass` och en tidigare instans har registrerat DKIM-`pass` för samma domän.

### DMARC-alignment

`SpfAlignment` jämför kuvertavsändardomänen med `From`-domänen, `DkimAlignment` jämför `d=`-domänen för den kontrollerade signaturen med `From`-domänen. `Strict` betyder identisk domän, `Relaxed` samma organisationsdomän, `None` ingen överensstämmelse. Organisationsdomänen bestäms heuristiskt: de två sista etiketterna, eller de tre sista vid kända sammansatta toppdomäner som `co.uk` eller `com.au`. En fullständig Public Suffix List ingår inte.

### MailHeaderAnalyzer.ExchangeClassification

Om headern innehåller fälten `X-MS-Exchange-Organization-*`, `X-OriginatorOrg` eller `X-MS-Exchange-CrossTenant-*`, fyller cmdleten egenskapen `Exchange`. Betydelserna följer Exchange-teamets artikel ”Demystifying hybrid mail flow”; bakgrunden finns i artikeln [Exchange-hybridheaders: internt eller externt?](/blog/exchange-hybrid-header-intern-extern).

| Egenskap | Innehåll |
|---|---|
| `Directionality`, `DirectionalityMeaning` | `Originating` eller `Incoming` med förklaring |
| `AuthAs`, `AuthAsMeaning` | `Internal` eller `Anonymous` med följderna för EOP-filtreringen |
| `AuthSource` | Server som gjorde klassificeringen |
| `AuthMechanism`, `AuthMechanismMeaning` | Mekanismkod. Endast värdet 10 (Externally Secured) är offentligt dokumenterat; för alla övriga koder anger modulen uttryckligen att Microsoft inte dokumenterar dem |
| `OriginatorOrg` | Standardsdomänen för den sändande klientorganisationen, det oförfalskbara klientorganisationsmärket vid mottagning från Microsoft 365 |
| `CrossTenantAuthAs`, `CrossTenantAuthSource`, `CrossTenantId` | Klassificering och klientorganisations-ID vid klientorganisationsgränsen |
| `CrossTenantFromEntity`, `CrossTenantFromMeaning` | `Internet`, `Hosted` eller `HybridOnPrem` med förklaring |
| `WrongTenantAttribution` | Värdet för `X-MS-Exchange-CrossTenant-OriginalAttributedTenantConnectingIp` när meddelandet har tilldelats en annan klientorganisation |
| `OrganizationHeadersPreserved`, `CrossPremisesHeadersFiltered` | Markörer för mottagna respektive av sändningsanslutningen borttagna organisationsheaders |

### MailHeaderAnalyzer.SpamAssessment

Egenskapen `Spam` sammanfattar bedömningarna från de kända filtren. Värdena kommer från externa system och avkodas, men utvärderas inte.

| Egenskap | Källa | Innehåll |
|---|---|---|
| `Scl`, `SclMeaning` | `X-Forefront-Antispam-Report`, `X-Microsoft-Antispam` | Spam Confidence Level med betydelse (-1 betrott, 0/1 inte spam, 5/6 misstänkt spam, 9 mycket sannolikt spam) |
| `Bcl` | `X-Microsoft-Antispam` | Bulk Complaint Level 0 till 9 |
| `Category`, `CategoryMeaning` | `CAT:` | Klassificering som `SPM`, `PHSH`, `HPHSH`, `MALW`, `SPOOF`, `DIMP`, `UIMP`, `BULK` |
| `SpamFilterVerdict`, `SpamFilterMeaning` | `SFV:` | Filterresultat som `NSPM`, `SPM`, `SKA` (tillåtelselista), `SKI` (intraorganisatoriskt) |
| `IPVerdict`, `IPVerdictMeaning` | `IPV:` | `CAL` (IP på anslutningens tillåtelselista) eller `NLI` (inget rykte) |
| `Direction`, `DirectionMeaning` | `DIR:` | `INB`, `OUT`, `INT` |
| `ConnectingIP`, `Country` | `CIP:`, `CTRY:` | Inlämnande IP och ursprungsland |
| `Forefront` | `X-Forefront-Antispam-Report` | Alla nyckel-värde-par i fältet |
| `SpamAssassinScore`, `SpamAssassinTests` | `X-Spam-Status` | Poäng och utlösta tester |
| `RspamdSymbols` | `X-Spamd-Result`, `X-Spam-Report` | Symboler med poäng |

Den fullständiga listan över `compauth`-Reason Codes finns i artikeln [Microsoft 365 compauth: Reason Codes](/blog/microsoft-365-compauth-reason-codes).

### Findings

Cmdleten levererar avvikelser som objekt i `Findings`, vart och ett med `Severity` (`Info`, `Warning`, `Fail`), en stabil `Code` för filter och skript samt en förklaring i `Message`.

| Kod | Allvarlighetsgrad | Betydelse |
|---|---|---|
| `DuplicateField` | Warning | Ett fält som RFC 5322 begränsar till en instans (`From`, `Subject`, `Date`, `Message-ID` med flera) förekommer flera gånger. E-postklienter och filter kan välja olika instanser; ett känt mönster vid förfalskningar |
| `BidiControls` | Warning | Unicode-kontrolltecken för skrivriktning i ett fält. De vänder läsriktningen, `fdp.exe` visas då som `exe.pdf`. Modulen visar dem som `<U+202E>` |
| `HopOverflow` | Warning | Fler än 200 `Received`-rader; de överskjutande analyserades inte |
| `AuthPlausibleOnly` | Info | authserv-id förekommer i leveranskedjan, men `-TrustedAuthServId` angavs inte: plausibelt, inte styrkt |
| `AuthNotTrusted` | Warning | `-TrustedAuthServId` angavs, men ingen kontrollrad har något av dessa ID:n |
| `AuthTrustedConflict` | Warning | Två kontrollrader med betrott ID rapporterar olika resultat för samma metod; gatewayen tar uppenbarligen inte bort främmande rader |
| `AuthUnverified` | Warning | Kontrollresultaten har en authserv-id som varken är betrodd eller förekommer i leveranskedjan |
| `AuthMixedOrigins` | Warning | Kontrollrader från flera ursprung finns |
| `ReceivedSpfForeign` | Warning | `receiver=` i `Received-SPF`-raden förekommer inte i kedjan |
| `ReceivedSpfOnly` | Info | SPF-resultatet kommer endast från `Received-SPF`, inte från en kontrollrad |
| `NoAuthResults` | Info | Inga kontrollresultat i headern |
| `DmarcFail` | Fail | DMARC har enligt den mottagande servern inte godkänts |
| `SpfNotPass` | Warning | SPF-resultat `fail`, `softfail`, `permerror` eller `temperror` |
| `DkimNotPass` | Warning | DKIM-resultat `fail`, `permerror` eller `temperror`, utan att mottagaren bekräftar ARC-kedjan |
| `DkimBrokenAfterForward` | Info | DKIM har inte godkänts hos mottagaren, men mottagaren rapporterar `arc=pass`, och en tidigare ARC-instans har registrerat DKIM-`pass` för samma domän: typiskt för vidarebefordringar och e-postlistor |
| `DkimWeakHash` | Warning | Signatur med `rsa-sha1` (RFC 8301 klassificerar SHA-1 som föråldrat) |
| `DkimBodyLength` | Warning | Taggen `l=` begränsar den signerade längden på meddelandetexten; bifogat innehåll täcks inte |
| `DkimExpired` | Warning | Tidpunkten `x=` ligger i det förflutna |
| `DkimFromUnsigned` | Warning | Fältet `From` ingår inte i `h=` trots att RFC 6376 kräver det |
| `ArcStructureInconsistent` | Warning | ARC-headers bildar inte en formellt fullständig kedja (luckor, saknade eller dubbla headers, felaktig `cv=`-följd); endast strukturkontroll |
| `ClockSkew` | Info | Ett hopp har en tidigare tidsstämpel än föregående hopp; fördröjningarna är endast ungefärliga värden |
| `ReplyToMismatch` | Info | `Reply-To`-domänen avviker från `From`-domänen; vanligt för nyhetsbrev, ett mönster vid phishing |
| `SpfNotAligned` | Info | Kuvertavsändardomänen och `From`-domänen tillhör olika organisationer; SPF bidrar då inte till DMARC |
| `ExchangeWrongTenant` | Warning | Meddelandet har tilldelats en annan klientorganisation; en klassisk orsak är en inkommande anslutning från en annan klientorganisation med samma certifikat eller IP-adresser |
| `ExchangeHeadersFiltered` | Warning | Sändningsanslutningen har tagit bort Cross-Premises-headers (KB3212872) |
| `ExchangeExternallySecured` | Warning | AuthMechanism 10: inkommande via en mottagningsanslutning med ”Externally Secured”, EOP-filtrering hoppades över |
| `SpamCategory` | Warning | Microsoft har tilldelat en annan kategori än `NONE` |
| `SpamConfidence` | Warning | SCL 5 eller högre |

## Funktion och begränsningar

Modulen läser vad som står i headern och drar slutsatser om vad som kan styrkas utan externa frågor. Detta medför vissa begränsningar:

- **Ingen kryptografisk kontroll.** DKIM- och ARC-signaturer beräknas inte på nytt och DNS-poster frågas inte. `Spf`, `Dkim`, `Dmarc` och `Arc` är alltid den mottagande serverns bedömning; `ArcStructure` kontrollerar endast kedjans struktur.
- **Ursprung kan bara styrkas med kännedom om gatewayen.** Utan `-TrustedAuthServId` är `AuthTrust` en rimlighetskontroll. Med parametern är utsagan beroende av att gatewayen tar bort främmande kontrollrader med sin authserv-id; modulen kan inte kontrollera detta.
- **Endast den sista `Received`-raden är styrkt.** Alla rader under den följde med från avsändaren och kan ha utformats godtyckligt. `Attested` markerar skillnaden; fördröjningarna i tidigare hopp bygger på uppgifterna i dessa rader.
- **Organisationsdomäner bestäms heuristiskt.** För Relaxed-alignment använder modulen en kort lista över sammansatta toppdomäner, inte en fullständig Public Suffix List.
- **AuthMechanism är endast delvis dokumenterad.** Förutom värdet 10 har Microsoft inte publicerat koderna; modulen hittar inte på betydelser.
- **Teckenuppsättningar.** RFC 2047-värden avkodas med de kodningar som .NET känner till på respektive system. Okända teckenuppsättningar lämnas oförändrade.

## Dataskydd

En fullständig header innehåller interna värdnamn, IP-adresser, avsändare, mottagare och ämne. Varför dessa uppgifter inte hör hemma i ett onlineverktyg förklaras i artikeln [Analysera e-postheaders utan att ladda upp e-postmeddelandet](/blog/e-mail-header-analysieren-ohne-upload). Samma löfte gäller modulen som webbläsarversionen: inga nätverksanrop. Modulens testsvit innehåller ett test som misslyckas så snart ett nätverks- eller DNS-cmdlet förekommer i källkoden.

## Källkod och versioner

Källkoden finns under MIT-licens på [GitHub](https://github.com/pfstr/MailHeaderAnalyzer). Analyslogiken är en portering av biblioteket som även Header Analyzer på den här webbplatsen använder; båda delar testfallen. Continuous Integration kontrollerar varje ändring med PSScriptAnalyzer och Pester på Windows PowerShell 5.1, PowerShell 7 under Windows, Ubuntu och macOS. Publiceringar i PowerShell Gallery sker automatiskt från versionsmärkta taggar; ändringarna för varje version finns i [Changelog](https://github.com/pfstr/MailHeaderAnalyzer/blob/main/CHANGELOG.md).

Version 0.2.0 från den 26 september 2026 skärpte ursprungsmodellen efter en granskningsanmärkning från @saltyslugga: parametern `-TrustedAuthServId`, nivån `Trusted`, `Matched` endast som rimlighet. Egenskapen `ArcValid` har utgått och ersatts av `ArcStructure` och `ArcStructureIssues`; skript som utvärderar `ArcValid` måste anpassas.

Jag tar emot felrapporter och förbättringsförslag som [Issue på GitHub](https://github.com/pfstr/MailHeaderAnalyzer/issues). Anonymisera testheaders för felrapporter i förväg; de medföljande testfallen använder uteslutande exempel-domäner enligt RFC 2606 och adresser enligt RFC 5737.

## Ändringshistorik

- **25 oktober 2026:** Källan RFC 7489 ersatt med RFC 9989, som ersätter DMARC-specifikationen sedan maj 2026.

## Källor

1.  [MailHeaderAnalyzer i PowerShell Gallery](https://www.powershellgallery.com/packages/MailHeaderAnalyzer): Paketsida med installationskommando och versionshistorik.

2.  [pfstr/MailHeaderAnalyzer på GitHub](https://github.com/pfstr/MailHeaderAnalyzer): Källkod, testsvit, Changelog och Issues.

3.  [Microsoft Learn: Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2?view=exchange-ps): Mall för strukturen i denna referens (syntax, beskrivning, exempel, parameteregenskaper).

4.  [RFC 8601: Message Header Field for Indicating Message Authentication Status](https://www.rfc-editor.org/rfc/rfc8601): Strukturen för `Authentication-Results` och regeln att endast den mottagande organisationens rad är avgörande (avsnitt 5).

5.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): Strukturen för `Received`-raderna och rekommendationen om en övre gräns som loopskydd (avsnitt 6.3).

6.  [RFC 3848: ESMTP and LMTP Transmission Types Registration](https://www.rfc-editor.org/rfc/rfc3848): `with`-värdena `ESMTPS`, `ESMTPA` och `ESMTPSA`.

7.  [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://www.rfc-editor.org/rfc/rfc6376): Taggar i `DKIM-Signature`, krav på signering av `From`.

8.  [RFC 8301: Cryptographic Algorithm and Key Usage Update to DKIM](https://www.rfc-editor.org/rfc/rfc8301): Klassificering av `rsa-sha1` som föråldrat.

9.  [RFC 9989: Domain-Based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc9989): Strict och Relaxed Alignment; ersätter RFC 7489 sedan maj 2026.

10.  [RFC 8617: The Authenticated Received Chain (ARC) Protocol](https://www.rfc-editor.org/rfc/rfc8617): Kedjans struktur med `ARC-Seal`, `ARC-Message-Signature` och `ARC-Authentication-Results`, instansnummer och `cv=`-värden.

11.  [Microsoft Learn: Anti-spam message headers in Microsoft 365](https://learn.microsoft.com/en-us/defender-office-365/message-headers-eop-mdo): Betydelsen av SCL, BCL, CAT, SFV, IPV och `compauth`-Reason Codes.

12.  [Exchange Team Blog: Demystifying hybrid mail flow](https://techcommunity.microsoft.com/blog/exchange/demystifying-hybrid-mail-flow-when-is-a-message-internal/1420838): Ursprunget till betydelserna av MessageDirectionality, AuthAs och AuthMechanism.

13.  [Header Analyzer på rafaelpfister.ch](/tools/header-analyzer): Webbläsarversionen med samma analyslogik.
