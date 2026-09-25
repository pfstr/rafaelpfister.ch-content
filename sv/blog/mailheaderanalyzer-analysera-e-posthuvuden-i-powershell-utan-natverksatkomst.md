---
title: "MailHeaderAnalyzer: analysera e-posthuvuden i PowerShell utan nätverksåtkomst"
navTitle: "MailHeaderAnalyzer (PowerShell)"
description: "Referens för PowerShell-modulen MailHeaderAnalyzer: parameteröversikt, syntax, beskrivning, exempel och parameteregenskaper för Get-MailHeaderAnalysis och ConvertTo-MailHeaderReport, samt utdataobjektet med leveranskedja, autentiseringsresultat, Exchange Online-klassificering och alla Finding-koder."
date: "2026-09-24"
kategorie: "SMTP & Mailflow"
timeToRead: "16 min lästid"
themen:
  - smtp-mailflow
  - microsoft-365-exchange
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
  - e-mail-header-analysieren-ohne-upload
  - microsoft-365-compauth-reason-codes
  - exchange-hybrid-header-intern-extern
slug: "mailheaderanalyzer-analysera-e-posthuvuden-i-powershell-utan-natverksatkomst"
translationId: "article-041d7b2f9615f670"
translationOf: mailheaderanalyzer-powershell-modul
url: https://rafaelpfister.ch/sv/blog/mailheaderanalyzer-analysera-e-posthuvuden-i-powershell-utan-natverksatkomst
translationSourceHash: 7c01d42e83d3aa175486c74e2f88855bc7d9dee519026ce748edb239fdd355cc
translationModel: gpt-5.6-terra
translatedAt: 2026-09-24T18:55:38.912Z
translationReview: automatic
---

# MailHeaderAnalyzer: analysera e-posthuvuden i PowerShell utan nätverksåtkomst

MailHeaderAnalyzer är en PowerShell-modul med två cmdlets. `Get-MailHeaderAnalysis` analyserar huvudet i ett e-postmeddelande: leveranskedjan med fördröjningar och TLS-information, SPF-, DKIM-, DMARC- och ARC-resultat inklusive kontroll av om resultaten kommer från den mottagande servern, DMARC-alignment, hybridklassificeringen i Exchange Online, bedömningarna från Microsoft Defender, SpamAssassin och Rspamd samt avvikelser som dubbla `From`-rader eller Unicode-styrtecken. `ConvertTo-MailHeaderReport` skapar en rapport för ärenden utifrån detta. Modulen arbetar helt offline: inga DNS-frågor, inga HTTP-anslutningar. Den är kommandoradsversionen av [Header Analyzer på denna webbplats](/tools/header-analyzer) och använder samma analyslogik.

| | |
|---|---|
| Modul | [MailHeaderAnalyzer i PowerShell Gallery](https://www.powershellgallery.com/packages/MailHeaderAnalyzer) |
| Källkod | [pfstr/MailHeaderAnalyzer på GitHub](https://github.com/pfstr/MailHeaderAnalyzer), MIT-licens |
| Gäller för | Windows PowerShell 5.1, PowerShell 7.x; Windows, Linux, macOS; Exchange Management Shell |
| Cmdlets | `Get-MailHeaderAnalysis`, `ConvertTo-MailHeaderReport` |

## Parameteröversikt

| Cmdlet | Parameter | Typ | Obligatorisk | Pipeline | Effekt |
|---|---|---|---|---|---|
| `Get-MailHeaderAnalysis` | `-Header` | `String[]` | Ja (parameteruppsättning Text) | Ja, efter värde | Huvudet som text. Rader från pipelinen sammanfogas till ett huvud |
| `Get-MailHeaderAnalysis` | `-Path` | `String[]` | Ja (parameteruppsättning Path) | Ja, efter egenskapsnamn | Fil med huvudet eller ett fullständigt `.eml`-meddelande; tar emot objekt från `Get-ChildItem` |
| `Get-MailHeaderAnalysis` | `-FromClipboard` | `Switch` | Ja (parameteruppsättning Clipboard) | Nej | Läser huvudet från urklipp (endast Windows) |
| `ConvertTo-MailHeaderReport` | `-Analysis` | `MailHeaderAnalyzer.Analysis` | Ja | Ja, efter värde | Resultatobjektet från `Get-MailHeaderAnalysis` |
| `ConvertTo-MailHeaderReport` | `-Format` | `String` | Nej | Nej | `Markdown` (standard) eller `Text` |

Båda cmdlets stöder Common Parameters `-Verbose`, `-ErrorAction`, `-ErrorVariable`, `-OutVariable` och de övriga från [about_CommonParameters](https://learn.microsoft.com/de-de/powershell/module/microsoft.powershell.core/about/about_commonparameters).

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

Analyserar huvudet i ett e-postmeddelande och returnerar ett analysobjekt.

### Syntax

#### Text (standard)

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

### Beskrivning

Cmdleten delar upp råhuvudet i fält, vecklar ut RFC 5322-radbrytningar och avkodar RFC 2047-värden i ämne och adresser. Från `Received`-raderna bygger den leveranskedjan i kronologisk ordning, beräknar fördröjningen per station och läser TLS-version, chiffer och protokollklass enligt RFC 3848. Från `Authentication-Results`, `Received-SPF`, `DKIM-Signature` och ARC-kedjan fastställer den autentiseringsresultaten och kontrollerar om granskningsraden verkligen kommer från en station i leveranskedjan (RFC 8601, avsnitt 5). Därutöver kommer DMARC-alignment, hybridklassificeringen i Exchange Online, spamfilterbedömningarna och en lista över avvikelser.

Cmdleten utför inga DNS-frågor och öppnar ingen nätverksanslutning. `Spf`, `Dkim` och `Dmarc` är därför alltid den mottagande serverns bedömning, kompletterad med en kontroll av om bedömningen kommer från den.

Indata läses tolerant: En tom rad avslutar huvudet och efterföljande meddelandetext ignoreras. Rader utan fältnamn och utan inledande blanksteg, som uppstår vid kopiering från klientdialoger, hör till föregående fält. En mbox-avgränsningsrad `From ...` före det första fältet hoppas över och en Byte Order Mark tas bort. Högst 200 `Received`-rader analyseras, räknat från leveransen.

### Exempel

#### Exempel 1

```powershell
Get-MailHeaderAnalysis -FromClipboard
```

Analyserar huvudet som finns i urklipp. I Outlook för Windows hittar du huvudet under Arkiv, Egenskaper, Internetheaders; i Outlook på webben under meddelandealternativen, i ”Visa meddelandeinformation”.

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

Analyserar ett sparat meddelande. Filen får innehålla enbart huvudet eller hela meddelandet; meddelandetexten ignoreras.

#### Exempel 3

```powershell
(Get-MailHeaderAnalysis -Path .\nachricht.eml).Hops
```

Visar leveranskedjan som tabell, med första hoppet först.

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

Läser huvudet rad för rad från en textfil och visar bara avvikelserna. `-Raw` vid `Get-Content` behövs inte, eftersom cmdleten själv sammanfogar raderna.

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
| `Get-ChildItem -Path .\export\*.eml` | Returnerar filobjekten; `-Path` tar över deras egenskap `FullName` |
| `Select-Object … @{ Name; Expression }` | Beräknad kolumn som sammanfattar alla Finding-koder till en sträng |
| `Export-Csv -NoTypeInformation -Encoding UTF8` | CSV utan typrubrikrad, UTF-8 för diakritiska tecken i ämnesrader |

#### Exempel 6

```powershell
Get-ChildItem .\export\*.eml |
    Get-MailHeaderAnalysis |
    Where-Object AuthTrust -eq 'Unmatched' |
    Select-Object Source, AuthServId, DeliveredBy
```

Listar meddelanden vars `Authentication-Results`-rad inte kommer från det levererande systemet. Vid phishinganalyser är detta ett snabbt första filter.

#### Exempel 7

```powershell
$a = Get-MailHeaderAnalysis -Path .\nachricht.eml
$a.Hops[$a.SlowestHopIndex - 1] | Format-List Index, FromHost, ByHost, Delay
```

Visar stationen med den största fördröjningen. `SlowestHopIndex` är 1-baserad, arrayen `Hops` är 0-baserad.

#### Exempel 8

```powershell
Get-MailHeaderAnalysis -Path .\nachricht.eml | ConvertTo-Json -Depth 6 | Set-Content .\analyse.json
```

Skriver hela analysen som JSON. `-Depth 6` behövs eftersom `Hops`, `DkimSignatures` och `Exchange` är kapslade objekt; standardvärdet 2 skulle bara visa dem som typnamn.

### Parametrar

#### -Header

Huvudet som text. Parametern tar emot en enskild sträng med hela huvudet eller flera strängar; pipelineindata samlas in och sammanfogas till ett huvud i slutet, därför fungerar `Get-Content datei | Get-MailHeaderAnalysis` utan `-Raw`. Om flera huvuden ska analyseras separat använder du `-Path` med flera filer.

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

Sökväg till en fil som innehåller huvudet eller ett fullständigt `.eml`-meddelande. Relativa sökvägar löses mot den aktuella katalogen. Filen läses med `[System.IO.File]::ReadAllText`: en Byte Order Mark beaktas, utan BOM används UTF-8. Varje fil ger ett eget resultatobjekt; saknade filer ger ett icke-avslutande fel.

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

Läser huvudet från urklipp med `Get-Clipboard -Raw`. Parametern är endast tillgänglig i Windows; på Linux och macOS avslutas cmdleten med ett felmeddelande, likaså om urklipp är tomt.

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

### Indata

`System.String`: huvudrader eller hela huvudet, till `-Header`.

`System.IO.FileInfo`: filobjekt från `Get-ChildItem`, vars `FullName` binds till `-Path`.

### Utdata

`MailHeaderAnalyzer.Analysis`: ett objekt per analyserat huvud. Egenskaperna beskrivs i avsnittet [Utdataobjekt](#ausgabeobjekt).

### Anmärkningar

Analysetexten (förklaringar i `Findings`, `CompAuthReasonMeaning` och betydelsefälten) är på engelska för att kunna användas oförändrad i internationella ärenden. Utdata i standardvyn kan visas helt med `Format-List *`.

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

Rapporten omfattar ämne, avsändare, datum och Message-ID, autentiseringsresultaten med ursprungsangivelse och alignment, alla Findings, leveranskedjan, Exchange-klassificeringen och spamfiltrens värden. Unicode-styrtecken för skrivriktning förblir synliga i rapporten som `<U+...>`, så att de inte hamnar i ett ärendesystem via rapporten. Den sista raden anger modulversionen.

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

Ger ut rapporten som text utan Markdown-formatering, till exempel för e-post eller konsolloggar.

#### Exempel 3

```powershell
Import-Module MailHeaderAnalyzer
Get-MailHeaderAnalysis -Path 'C:\Temp\bounce.eml' | ConvertTo-MailHeaderReport -Format Text
```

Användning i Exchange Management Shell. Om `$env:PSModulePath` där begränsas av grupprinciper, läser du in modulen med `Import-Module` och den fullständiga sökvägen till filen `.psd1`.

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

- `Markdown`: rubriker, punktlistor och leveranskedjan som tabell. Standard.
- `Text`: rubriker med versaler, indragna rader, leveranskedjan som numrerad lista.

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

`Get-MailHeaderAnalysis` returnerar ett objekt av typen `MailHeaderAnalyzer.Analysis` per indata. Standardvyn visar sammanfattningen från exempel 1; alla egenskaper är tillgängliga via `Select-Object`, `Format-List *` eller `ConvertTo-Json`.

### MailHeaderAnalyzer.Analysis

| Egenskap | Typ | Innehåll |
|---|---|---|
| `Source` | String | Filsökväg, `Clipboard` eller `Text` |
| `Subject` | String | Ämne, RFC 2047-avkodat |
| `From`, `ReplyTo`, `ReturnPath` | Adressobjekt | `Name`, `Address`, `Domain`, `Display`; `$null` om fältet saknas |
| `Date` | DateTime (UTC) | Värdet i fältet `Date` |
| `MessageId` | String | `Message-ID` |
| `MailFromDomain` | String | Envelope-avsändardomän från `smtp.mailfrom` i SPF-kontrollen, annars från `Return-Path` |
| `Spf`, `Dkim`, `Dmarc`, `Arc`, `CompAuth` | String | Resultat enligt den relevanta `Authentication-Results`-raden (`pass`, `fail`, `none`, `softfail` med flera); `$null` om det inte har kontrollerats |
| `CompAuthReason`, `CompAuthReasonMeaning` | String | Reason-kod för sammansatt autentisering från Microsoft 365 och dess betydelse |
| `AuthTrust` | String | `Matched`, `Unmatched`, `Absent` eller `None`, se [AuthTrust](#authtrust-herkunft-der-prüfergebnisse) |
| `AuthServId` | String | authserv-id för den relevanta granskningsraden |
| `AuthenticationResults` | Objekt[] | Alla `Authentication-Results`-rader med `AuthServId`, `Methods`, `Trust`, `Raw` |
| `ReceivedSpf` | Objekt | `Received-SPF`-raden med `Result` och `Properties` |
| `SpfAlignment`, `DkimAlignment` | String | `Strict`, `Relaxed`, `None` eller `$null` |
| `Hops` | Hop[] | Leveranskedja i kronologisk ordning, se [Hop-objekt](#mailheaderanalyzerhop) |
| `HopCount`, `TotalDuration`, `SlowestHopIndex`, `HasClockSkew` | Int, TimeSpan, Int, Bool | Nyckeltal för kedjan |
| `DeliveredBy` | String | `by`-värden för den senaste `Received`-raden, alltså den levererande stationen |
| `DkimSignatures` | Objekt[] | Per signatur `Domain`, `Selector`, `Algorithm`, `Canonicalization`, `SignedHeaders`, `BodyLength`, `Timestamp`, `Expires`, `ReceiverResult`, `Tags` |
| `ArcChain`, `ArcValid` | Objekt[], Bool | ARC-instanser med `Instance`, `SealDomain`, `ChainValidation`, `Methods`; `ArcValid` är `$null` utan ARC-kedja |
| `Exchange` | Objekt | Hybridklassificering i Exchange Online, se [Exchange-objekt](#mailheaderanalyzerexchangeclassification); `$null` utan motsvarande huvuden |
| `Spam` | Objekt | Spamfilterbedömningar, se [Spam-objekt](#mailheaderanalyzerspamassessment); `$null` utan motsvarande huvuden |
| `List` | Objekt | `ListId`, `Unsubscribe`, `OneClick` (RFC 8058); `$null` utan listheaders |
| `Findings` | Finding[] | Avvikelser med `Severity`, `Code`, `Message`, se [Findings](#findings) |
| `Fields` | Objekt[] | Alla fält med `Name`, `Value` (utvecklat) och `Raw` |
| `HadBody` | Bool | Om meddelandetext följde efter huvudet |

### MailHeaderAnalyzer.Hop

Varje post i `Hops` motsvarar en `Received`-rad. Ordningen är kronologisk, alltså omvänd mot ordningen i huvudet.

| Egenskap | Innehåll |
|---|---|
| `Index` | Löpnummer, 1 = inlämning |
| `FromHost`, `Helo`, `ReverseDns`, `IPAddress`, `IsPrivateIP` | Uppgifter om det inlämnande systemet från `from`-delen; IP-adressen kommer från hakparentesen i kommentaren, rDNS-namnet från kommentaren före den |
| `ByHost`, `Software` | Mottagande system och dess programvara (kommentar efter `by`) |
| `Protocol`, `ProtocolClass` | `with`-värde och klass enligt RFC 3848: `TlsAuthenticated` (ESMTPSA), `Tls` (ESMTPS eller TLS-information finns), `Authenticated` (ESMTPA), `Plain`, `Http`, `Mapi`, `Local` |
| `TlsVersion`, `TlsCipher` | Från skrivsätten hos Microsoft (`version=TLS1_2, cipher=…`), Postfix (`using TLSv1.3 with cipher …`) och Exim |
| `Id`, `For`, `Via` | Andra `Received`-komponenter |
| `Date` | Tidsstämpel efter semikolon, UTC |
| `Delay` | TimeSpan till föregående hopp; negativ vid klockavvikelse |
| `Provider` | Identifierad leverantör eller gateway utifrån värdnamnen (Microsoft 365, Google Workspace, SEPPmail, HIN, Proofpoint, Mimecast med flera) |
| `Attested` | `$true` endast vid sista hoppet: enbart denna rad har skrivits av det mottagande systemet självt, alla rader nedanför fanns redan i meddelandet |
| `Raw` | Originalraden |

### AuthTrust: granskningsresultatens ursprung

En `Authentication-Results`-rad kan vilken avsändare som helst själv skriva in i ett meddelande. Enligt RFC 8601, avsnitt 5, är endast raden från den mottagande organisationen relevant, och dess authserv-id måste kunna kopplas till en station i leveranskedjan. Cmdleten jämför authserv-id för varje rad med `by`-värdarna i `Received`-kedjan (samma domän eller subdomän, alltid vid punktgränsen, ingen delsträng).

| Värde | Betydelse |
|---|---|
| `Matched` | authserv-id förekommer som `by`-värd i kedjan. Endast sådana rader ingår i `Spf`, `Dkim`, `Dmarc` så snart minst en finns |
| `Unmatched` | authserv-id förekommer inte i kedjan. Resultaten visas men räknas som ett obekräftat påstående; Finding `AuthUnverified` anger detta |
| `Absent` | Raden saknar authserv-id. Microsoft 365 skriver sin granskningsrad i denna form, direkt inledd med `spf=` |
| `None` | Ingen granskningsrad finns |

Om huvudet innehåller granskningsrader från flera ursprung rapporterar Finding `AuthMixedOrigins` detta. Saknas en granskningsrad men finns en `Received-SPF`-rad, tas dess resultat över som `Spf` och märks med `ReceivedSpfOnly`.

### DMARC-alignment

`SpfAlignment` jämför envelope-avsändardomänen med `From`-domänen, `DkimAlignment` jämför `d=`-domänen för den granskade signaturen med `From`-domänen. `Strict` betyder identisk domän, `Relaxed` samma organisationsdomän och `None` ingen överensstämmelse. Organisationsdomänen fastställs heuristiskt: de två sista etiketterna, eller de tre sista vid kända sammansatta ändelser som `co.uk` eller `com.au`. En fullständig Public Suffix List ingår inte.

### MailHeaderAnalyzer.ExchangeClassification

Om huvudet innehåller fälten `X-MS-Exchange-Organization-*`, `X-OriginatorOrg` eller `X-MS-Exchange-CrossTenant-*`, fyller cmdleten egenskapen `Exchange`. Betydelserna följer Exchange-teamets artikel ”Demystifying hybrid mail flow”; bakgrunden beskrivs i artikeln [Exchange-hybridheaders: interna eller externa?](/blog/exchange-hybrid-header-intern-extern).

| Egenskap | Innehåll |
|---|---|
| `Directionality`, `DirectionalityMeaning` | `Originating` eller `Incoming` med förklaring |
| `AuthAs`, `AuthAsMeaning` | `Internal` eller `Anonymous` med följderna för EOP-filtrering |
| `AuthSource` | Server som gjorde klassificeringen |
| `AuthMechanism`, `AuthMechanismMeaning` | Mekanismkod. Endast värdet 10 (Externally Secured) är offentligt dokumenterat; för alla andra koder anger modulen uttryckligen att Microsoft inte dokumenterar dem |
| `OriginatorOrg` | Standardsdomän för den sändande klientorganisationen, den oförfalskbara klientorganisationsmarkören vid mottagning från Microsoft 365 |
| `CrossTenantAuthAs`, `CrossTenantAuthSource`, `CrossTenantId` | Klassificering och klientorganisations-ID vid klientorganisationsgränsen |
| `CrossTenantFromEntity`, `CrossTenantFromMeaning` | `Internet`, `Hosted` eller `HybridOnPrem` med förklaring |
| `WrongTenantAttribution` | Värdet från `X-MS-Exchange-CrossTenant-OriginalAttributedTenantConnectingIp` om meddelandet har tilldelats en främmande klientorganisation |
| `OrganizationHeadersPreserved`, `CrossPremisesHeadersFiltered` | Markörer för mottagna respektive av sändningsanslutningen borttagna organisationsheaders |

### MailHeaderAnalyzer.SpamAssessment

Egenskapen `Spam` sammanfattar bedömningarna från de kända filtren. Värdena kommer från externa system och avkodas, men bedöms inte.

| Egenskap | Källa | Innehåll |
|---|---|---|
| `Scl`, `SclMeaning` | `X-Forefront-Antispam-Report`, `X-Microsoft-Antispam` | Spam Confidence Level med betydelse (-1 betrodd, 0/1 inte spam, 5/6 misstänkt spam, 9 mycket sannolikt spam) |
| `Bcl` | `X-Microsoft-Antispam` | Bulk Complaint Level 0 till 9 |
| `Category`, `CategoryMeaning` | `CAT:` | Klassificeringar som `SPM`, `PHSH`, `HPHSH`, `MALW`, `SPOOF`, `DIMP`, `UIMP`, `BULK` |
| `SpamFilterVerdict`, `SpamFilterMeaning` | `SFV:` | Filterresultat som `NSPM`, `SPM`, `SKA` (allowlist), `SKI` (inom organisationen) |
| `IPVerdict`, `IPVerdictMeaning` | `IPV:` | `CAL` (IP på anslutningsallowlist) eller `NLI` (inget rykte) |
| `Direction`, `DirectionMeaning` | `DIR:` | `INB`, `OUT`, `INT` |
| `ConnectingIP`, `Country` | `CIP:`, `CTRY:` | Inlämnande IP och ursprungsland |
| `Forefront` | `X-Forefront-Antispam-Report` | Alla nyckel/värde-par i fältet |
| `SpamAssassinScore`, `SpamAssassinTests` | `X-Spam-Status` | Poäng och utlösta tester |
| `RspamdSymbols` | `X-Spamd-Result`, `X-Spam-Report` | Symboler med poäng |

Den fullständiga listan över `compauth`-Reason-koder finns i artikeln [Microsoft 365 compauth: Reason-koder](/blog/microsoft-365-compauth-reason-codes).

### Findings

Cmdleten returnerar avvikelser som objekt i `Findings`, var och en med `Severity` (`Info`, `Warning`, `Fail`), en stabil `Code` för filter och skript samt en förklaring i `Message`.

| Kod | Allvarlighetsgrad | Betydelse |
|---|---|---|
| `DuplicateField` | Warning | Ett fält som RFC 5322 begränsar till en instans (`From`, `Subject`, `Date`, `Message-ID` med flera) förekommer flera gånger. E-postklienter och filter kan välja olika instanser; ett känt mönster vid förfalskningar |
| `BidiControls` | Warning | Unicode-styrtecken för skrivriktning i ett fält. De vänder läsriktningen, `fdp.exe` visas då som `exe.pdf`. Modulen visar dem som `<U+202E>` |
| `HopOverflow` | Warning | Fler än 200 `Received`-rader; de överskjutande analyserades inte |
| `AuthUnverified` | Warning | Granskningsresultaten har ett authserv-id som inte förekommer i leveranskedjan |
| `AuthMixedOrigins` | Warning | Granskningsrader från flera ursprung finns |
| `ReceivedSpfForeign` | Warning | `receiver=` i `Received-SPF`-raden förekommer inte i kedjan |
| `ReceivedSpfOnly` | Info | SPF-resultatet kommer endast från `Received-SPF`, inte från en granskningsrad |
| `NoAuthResults` | Info | Inga granskningsresultat i huvudet |
| `DmarcFail` | Fail | DMARC misslyckades enligt den mottagande servern |
| `SpfNotPass` | Warning | SPF-resultat `fail`, `softfail`, `permerror` eller `temperror` |
| `DkimNotPass` | Warning | DKIM-resultat `fail`, `permerror` eller `temperror` utan ARC-vittne |
| `DkimBrokenAfterForward` | Info | DKIM misslyckades hos mottagaren, men ett ARC-sigill från samma domän intygar en tidigare giltig signatur: typiskt för vidarebefordringar och e-postlistor |
| `DkimWeakHash` | Warning | Signatur med `rsa-sha1` (RFC 8301 klassar SHA-1 som föråldrat) |
| `DkimBodyLength` | Warning | Taggen `l=` begränsar den signerade längden på meddelandetexten; bifogat innehåll omfattas inte |
| `DkimExpired` | Warning | Tidpunkten `x=` ligger i det förflutna |
| `DkimFromUnsigned` | Warning | Fältet `From` ingår inte i `h=`, trots att RFC 6376 kräver det |
| `ClockSkew` | Info | Ett hopp har en tidigare tidsstämpel än sin föregångare; fördröjningarna är endast uppskattningar |
| `ReplyToMismatch` | Info | Domänen `Reply-To` skiljer sig från domänen `From`; vanligt i nyhetsbrev, ett mönster vid phishing |
| `SpfNotAligned` | Info | Envelope-avsändardomänen och domänen `From` tillhör olika organisationer; SPF bidrar då inte till DMARC |
| `ExchangeWrongTenant` | Warning | Meddelandet tilldelades en främmande klientorganisation; en klassisk orsak är en inbound-anslutning från en annan klientorganisation med samma certifikat eller IP-adresser |
| `ExchangeHeadersFiltered` | Warning | Sändningsanslutningen har tagit bort Cross-Premises-headers (KB3212872) |
| `ExchangeExternallySecured` | Warning | AuthMechanism 10: inkommande via en mottagningsanslutning med ”Externally Secured”, EOP-filtrering har hoppats över |
| `SpamCategory` | Warning | Microsoft har tilldelat en annan kategori än `NONE` |
| `SpamConfidence` | Warning | SCL 5 eller högre |

## Funktion och begränsningar

Modulen läser vad som står i huvudet och drar slutsatser om vad som kan beläggas utan externa frågor. Detta medför vissa begränsningar:

- **Ingen kryptografisk kontroll.** DKIM-signaturer räknas inte om och DNS-poster frågas inte efter. `Spf`, `Dkim` och `Dmarc` är alltid den mottagande serverns bedömning, kompletterad med kontrollen av om den bedömningen faktiskt kommer från den.
- **Endast den sista `Received`-raden är styrkt.** Alla rader under den följde med från avsändaren och kan utformas godtyckligt. `Attested` markerar skillnaden; fördröjningarna för tidigare hopp bygger på uppgifterna i dessa rader.
- **Organisationsdomäner fastställs heuristiskt.** För relaxed alignment använder modulen en kort lista med sammansatta ändelser, inte en fullständig Public Suffix List.
- **AuthMechanism är endast delvis dokumenterat.** Utöver värdet 10 har Microsoft inte publicerat koderna; modulen hittar inte på några betydelser.
- **Teckenkodningar.** RFC 2047-värden avkodas med de kodningar som .NET känner till på respektive system. Okända teckenkodningar lämnas oförändrade.

## Dataskydd

Ett komplett huvud innehåller interna värdnamn, IP-adresser, avsändare, mottagare och ämne. Varför dessa uppgifter inte hör hemma i ett onlineverktyg beskrivs i artikeln [Analysera e-posthuvuden utan att ladda upp e-postmeddelandet](/blog/e-mail-header-analysieren-ohne-upload). För modulen gäller samma löfte som för webbläsarversionen: inga nätverksanrop. Modulens testsvit innehåller ett test som misslyckas så snart en nätverks- eller DNS-cmdlet förekommer i källkoden.

## Källkod och versioner

Källkoden finns under MIT-licens på [GitHub](https://github.com/pfstr/MailHeaderAnalyzer). Analyslogiken är en portering av biblioteket som också används av Header Analyzer på denna webbplats; båda delar testfallen. Continuous Integration kontrollerar varje ändring med PSScriptAnalyzer och Pester i Windows PowerShell 5.1, PowerShell 7 på Windows, Ubuntu och macOS. Publiceringar i PowerShell Gallery sker automatiskt från versionerade taggar; ändringarna per version finns i [Changelog](https://github.com/pfstr/MailHeaderAnalyzer/blob/main/CHANGELOG.md).

Fel och önskemål om utökningar tar jag emot som [Issue på GitHub](https://github.com/pfstr/MailHeaderAnalyzer/issues). Anonymisera testhuvuden för felrapporter i förväg; de medföljande testfallen använder endast exempeldomäner enligt RFC 2606 och adresser enligt RFC 5737.

## Källor

1.  [MailHeaderAnalyzer i PowerShell Gallery](https://www.powershellgallery.com/packages/MailHeaderAnalyzer): paketsida med installationskommando och versionshistorik.

2.  [pfstr/MailHeaderAnalyzer på GitHub](https://github.com/pfstr/MailHeaderAnalyzer): källkod, testsvit, Changelog och Issues.

3.  [Microsoft Learn: Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2?view=exchange-ps): mall för strukturen i denna referens (syntax, beskrivning, exempel, parameteregenskaper).

4.  [RFC 8601: Message Header Field for Indicating Message Authentication Status](https://www.rfc-editor.org/rfc/rfc8601): struktur för `Authentication-Results` och regeln att endast raden från den mottagande organisationen är relevant (avsnitt 5).

5.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): struktur för `Received`-rader och rekommendationen om en övre gräns som skydd mot slingor (avsnitt 6.3).

6.  [RFC 3848: ESMTP and LMTP Transmission Types Registration](https://www.rfc-editor.org/rfc/rfc3848): `with`-värdena `ESMTPS`, `ESMTPA` och `ESMTPSA`.

7.  [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://www.rfc-editor.org/rfc/rfc6376): taggar för `DKIM-Signature`, krav på signering av `From`.

8.  [RFC 8301: Cryptographic Algorithm and Key Usage Update to DKIM](https://www.rfc-editor.org/rfc/rfc8301): klassificering av `rsa-sha1` som föråldrat.

9.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc7489): Strict och Relaxed Alignment.

10.  [RFC 8617: The Authenticated Received Chain (ARC) Protocol](https://www.rfc-editor.org/rfc/rfc8617): `ARC-Seal`, `ARC-Authentication-Results` och kedjekontrollen `cv=`.

11.  [Microsoft Learn: Anti-spam message headers in Microsoft 365](https://learn.microsoft.com/en-us/defender-office-365/message-headers-eop-mdo): betydelsen av SCL, BCL, CAT, SFV, IPV och `compauth`-Reason-koderna.

12.  [Exchange Team Blog: Demystifying hybrid mail flow](https://techcommunity.microsoft.com/blog/exchange/demystifying-hybrid-mail-flow-when-is-a-message-internal/1420838): ursprunget till betydelserna för MessageDirectionality, AuthAs och AuthMechanism.

13.  [Header Analyzer på rafaelpfister.ch](/tools/header-analyzer): webbläsarversionen med samma analyslogik.
