---
title: "MailHeaderAnalyzer: Analyze Email Headers in PowerShell Without Network Access"
navTitle: "MailHeaderAnalyzer (PowerShell)"
description: "Reference for the MailHeaderAnalyzer PowerShell module: parameter overview, syntax, description, examples, and parameter properties for Get-MailHeaderAnalysis and ConvertTo-MailHeaderReport, plus the output object with delivery chain, authentication results, Exchange Online classification, and all finding codes."
date: "2026-09-24"
kategorie: "SMTP & Mail Flow"
timeToRead: "16 min read"
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
slug: "mailheaderanalyzer-analyze-email-headers-in-powershell-without-network-access"
translationId: "article-041d7b2f9615f670"
translationOf: mailheaderanalyzer-powershell-modul
url: https://rafaelpfister.ch/en/blog/mailheaderanalyzer-analyze-email-headers-in-powershell-without-network-access
translationSourceHash: 7c01d42e83d3aa175486c74e2f88855bc7d9dee519026ce748edb239fdd355cc
translationModel: gpt-5.6-terra
translatedAt: 2026-09-24T18:49:54.505Z
translationReview: automatic
---

# MailHeaderAnalyzer: Analyze Email Headers in PowerShell Without Network Access

MailHeaderAnalyzer is a PowerShell module with two cmdlets. `Get-MailHeaderAnalysis` analyzes an email header: the delivery chain with delays and TLS details, SPF, DKIM, DMARC, and ARC results including verification of whether those results originated from the receiving server, DMARC alignment, the Exchange Online hybrid classification, ratings from Microsoft Defender, SpamAssassin, and Rspamd, as well as anomalies such as duplicate `From` lines or Unicode control characters. `ConvertTo-MailHeaderReport` creates a report for tickets. The module works entirely offline: no DNS lookups, no HTTP connections. It is the command-line version of the [Header Analyzer on this website](/tools/header-analyzer) and uses the same analysis logic.

| | |
|---|---|
| Module | [MailHeaderAnalyzer in the PowerShell Gallery](https://www.powershellgallery.com/packages/MailHeaderAnalyzer) |
| Source code | [pfstr/MailHeaderAnalyzer on GitHub](https://github.com/pfstr/MailHeaderAnalyzer), MIT License |
| Applies to | Windows PowerShell 5.1, PowerShell 7.x; Windows, Linux, macOS; Exchange Management Shell |
| Cmdlets | `Get-MailHeaderAnalysis`, `ConvertTo-MailHeaderReport` |

## Parameter overview

| Cmdlet | Parameter | Type | Required | Pipeline | Effect |
|---|---|---|---|---|---|
| `Get-MailHeaderAnalysis` | `-Header` | `String[]` | Yes (Text parameter set) | Yes, by value | The header as text. Lines from the pipeline are combined into one header |
| `Get-MailHeaderAnalysis` | `-Path` | `String[]` | Yes (Path parameter set) | Yes, by property name | File containing the header or a complete `.eml` message; accepts objects from `Get-ChildItem` |
| `Get-MailHeaderAnalysis` | `-FromClipboard` | `Switch` | Yes (Clipboard parameter set) | No | Reads the header from the clipboard (Windows only) |
| `ConvertTo-MailHeaderReport` | `-Analysis` | `MailHeaderAnalyzer.Analysis` | Yes | Yes, by value | The result object from `Get-MailHeaderAnalysis` |
| `ConvertTo-MailHeaderReport` | `-Format` | `String` | No | No | `Markdown` (default) or `Text` |

Both cmdlets support the common parameters `-Verbose`, `-ErrorAction`, `-ErrorVariable`, `-OutVariable` and the others from [about_CommonParameters](https://learn.microsoft.com/de-de/powershell/module/microsoft.powershell.core/about/about_commonparameters).

## Installation

```powershell
Install-Module -Name MailHeaderAnalyzer -Scope CurrentUser
```

| Option | Effect |
|---|---|
| `-Name MailHeaderAnalyzer` | Name of the module in the PowerShell Gallery |
| `-Scope CurrentUser` | Installs in the user's module directory without administrator rights |

On a system without internet access, download the module on another computer with `Save-Module -Name MailHeaderAnalyzer -Path C:\Temp` and copy the `MailHeaderAnalyzer` folder to a directory in `$env:PSModulePath`, for example `%USERPROFILE%\Documents\WindowsPowerShell\Modules` (Windows PowerShell 5.1) or `%USERPROFILE%\Documents\PowerShell\Modules` (PowerShell 7). `Update-Module -Name MailHeaderAnalyzer` retrieves updates, and `Get-Module -Name MailHeaderAnalyzer -ListAvailable` shows the installed version.

## Get-MailHeaderAnalysis

Analyzes an email header and returns an analysis object.

### Syntax

#### Text (default)

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

### Description

The cmdlet splits the raw header into fields, unfolds RFC 5322 folding, and decodes RFC 2047 values in subjects and addresses. From the `Received` lines, it builds the delivery chain in chronological order, calculates the delay at each station, and reads the TLS version, cipher, and protocol class according to RFC 3848. From `Authentication-Results`, `Received-SPF`, `DKIM-Signature`, and the ARC chain, it determines authentication results and checks whether the verification line actually originated from a station in the delivery chain (RFC 8601, section 5). It also includes DMARC alignment, the Exchange Online hybrid classification, spam filter ratings, and a list of anomalies.

The cmdlet performs no DNS lookups and opens no network connection. `Spf`, `Dkim`, and `Dmarc` are therefore always the receiving server's judgment, supplemented by verification of whether that judgment originated from it. DKIM signatures are not cryptographically revalidated.

Input is read tolerantly: a blank line ends the header, and any following message body is ignored. Lines without a field name and without leading whitespace, as can occur when copying from client dialogs, belong to the preceding field. An mbox separator line `From ...` before the first field is skipped, and a Byte Order Mark is removed. At most 200 `Received` lines are analyzed, counted from delivery.

### Examples

#### Example 1

```powershell
Get-MailHeaderAnalysis -FromClipboard
```

Analyzes the header in the clipboard. In Outlook for Windows, you can find the header under File, Properties, Internet headers; in Outlook on the web, find it in Message options under “View message details.”

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

#### Example 2

```powershell
Get-MailHeaderAnalysis -Path .\nachricht.eml
```

Analyzes a saved message. The file may contain only the header or the complete message; the message body is ignored.

#### Example 3

```powershell
(Get-MailHeaderAnalysis -Path .\nachricht.eml).Hops
```

Shows the delivery chain as a table, first hop first.

```text
#   From                 IP              By                                     Protocol   TLS      Time (UTC)           Delay
-   ----                 --              --                                     --------   ---      ----------           -----
1   client.example.net   198.51.100.34   mail.example.org                       ESMTPSA    -        2026-08-03 09:14:28  -
2   mail.example.org     203.0.113.25    mx.eur02.prod.protection.outlook.com   Microsoft… TLS 1.3  2026-08-03 09:15:09  41 s
3   AM0EUR02FT056.eop…   -               ZR0P278MB0570.CHEP278.PROD.OUTLOOK.COM Microsoft… TLS 1.2  2026-08-03 09:15:10  1 s
```

#### Example 4

```powershell
Get-Content -Path .\header.txt | Get-MailHeaderAnalysis | Select-Object -ExpandProperty Findings
```

Reads the header line by line from a text file and shows only anomalies. `-Raw` with `Get-Content` is not necessary; the cmdlet combines the lines itself.

#### Example 5

```powershell
Get-ChildItem -Path .\export\*.eml |
    Get-MailHeaderAnalysis |
    Select-Object Source, Subject, Spf, Dkim, Dmarc, AuthTrust, HopCount,
        @{ Name = 'Findings'; Expression = { ($_.Findings.Code | Sort-Object -Unique) -join ',' } } |
    Export-Csv -Path .\auswertung.csv -NoTypeInformation -Encoding UTF8
```

Analyzes all messages in a folder and writes a CSV file with one row per message. The calculated column combines the finding codes.

| Option | Effect |
|---|---|
| `Get-ChildItem -Path .\export\*.eml` | Returns file objects; `-Path` accepts their `FullName` property |
| `Select-Object … @{ Name; Expression }` | Calculated column that combines all finding codes into a string |
| `Export-Csv -NoTypeInformation -Encoding UTF8` | CSV without type header, UTF-8 for accented characters in subject lines |

#### Example 6

```powershell
Get-ChildItem .\export\*.eml |
    Get-MailHeaderAnalysis |
    Where-Object AuthTrust -eq 'Unmatched' |
    Select-Object Source, AuthServId, DeliveredBy
```

Lists messages whose `Authentication-Results` line does not originate from the delivering system. This is a quick first filter in phishing analysis.

#### Example 7

```powershell
$a = Get-MailHeaderAnalysis -Path .\nachricht.eml
$a.Hops[$a.SlowestHopIndex - 1] | Format-List Index, FromHost, ByHost, Delay
```

Shows the station with the longest delay. `SlowestHopIndex` is 1-based, while the `Hops` array is 0-based.

#### Example 8

```powershell
Get-MailHeaderAnalysis -Path .\nachricht.eml | ConvertTo-Json -Depth 6 | Set-Content .\analyse.json
```

Writes the complete analysis as JSON. `-Depth 6` is necessary because `Hops`, `DkimSignatures`, and `Exchange` are nested objects; the default value of 2 would output only their type names.

### Parameters

#### -Header

The header as text. The parameter accepts a single string containing the entire header or multiple strings; pipeline input is collected and combined into one header at the end, so `Get-Content datei | Get-MailHeaderAnalysis` works without `-Raw`. To analyze multiple headers separately, use `-Path` with multiple files.

Aliases: `Text`, `Raw`, `InputObject`

| Parameter property | Value |
|---|---|
| Type | `String[]` |
| Default value | None |
| Supports wildcards | No |

| Text parameter set | Value |
|---|---|
| Position | 0 |
| Required | Yes |
| Value from pipeline | Yes |
| Value from pipeline by property name | No |

#### -Path

Path to a file containing the header or a complete `.eml` message. Relative paths are resolved against the current directory. The file is read with `[System.IO.File]::ReadAllText`: a Byte Order Mark is honored; without a BOM, UTF-8 is used. Each file produces a separate result object; missing files produce a non-terminating error.

Aliases: `FullName`, `PSPath`, `LiteralPath`

| Parameter property | Value |
|---|---|
| Type | `String[]` |
| Default value | None |
| Supports wildcards | No |

| Path parameter set | Value |
|---|---|
| Position | Named |
| Required | Yes |
| Value from pipeline | No |
| Value from pipeline by property name | Yes |

#### -FromClipboard

Reads the header from the clipboard with `Get-Clipboard -Raw`. The parameter is available only on Windows; on Linux and macOS, the cmdlet terminates with an error, as it does if the clipboard is empty.

| Parameter property | Value |
|---|---|
| Type | `SwitchParameter` |
| Default value | `False` |
| Supports wildcards | No |

| Clipboard parameter set | Value |
|---|---|
| Position | Named |
| Required | Yes |
| Value from pipeline | No |
| Value from pipeline by property name | No |

### Inputs

`System.String`: header lines or the entire header, to `-Header`.

`System.IO.FileInfo`: file objects from `Get-ChildItem`, whose `FullName` is bound to `-Path`.

### Outputs

`MailHeaderAnalyzer.Analysis`: one object per analyzed header. The properties are described in the [Output object](#ausgabeobjekt) section.

### Notes

The analysis text (explanations in `Findings`, `CompAuthReasonMeaning`, and the meaning fields) is in English so it can be used unchanged in international tickets. You can display the default view in full with `Format-List *`.

## ConvertTo-MailHeaderReport

Creates a report in Markdown or text from an analysis object.

### Syntax

```powershell
ConvertTo-MailHeaderReport
    [-Analysis] <Object>
    [-Format <String>]
    [<CommonParameters>]
```

### Description

The report includes the subject, sender, date, and Message-ID; authentication results with origin and alignment; all findings; the delivery chain; the Exchange classification; and spam filter values. Unicode control characters for text direction remain visible in the report as `<U+...>` so they cannot reach a ticket system through the report. The last line states the module version.

### Examples

#### Example 1

```powershell
Get-MailHeaderAnalysis -Path .\nachricht.eml | ConvertTo-MailHeaderReport | Set-Clipboard
```

Creates a Markdown report and places it in the clipboard.

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

#### Example 2

```powershell
Get-MailHeaderAnalysis -FromClipboard | ConvertTo-MailHeaderReport -Format Text
```

Outputs the report as text without Markdown formatting, for example for emails or console logs.

#### Example 3

```powershell
Import-Module MailHeaderAnalyzer
Get-MailHeaderAnalysis -Path 'C:\Temp\bounce.eml' | ConvertTo-MailHeaderReport -Format Text
```

Use in the Exchange Management Shell. If `$env:PSModulePath` is restricted there by Group Policy, load the module with `Import-Module` and the full path to the `.psd1` file.

### Parameters

#### -Analysis

The analysis object from `Get-MailHeaderAnalysis`. The cmdlet rejects other object types with a binding error.

| Parameter property | Value |
|---|---|
| Type | `MailHeaderAnalyzer.Analysis` |
| Default value | None |
| Supports wildcards | No |

| Parameter set (all) | Value |
|---|---|
| Position | 0 |
| Required | Yes |
| Value from pipeline | Yes |
| Value from pipeline by property name | No |

#### -Format

The output format. Valid values:

- `Markdown`: headings, lists, and the delivery chain as a table. Default.
- `Text`: uppercase headings, indented lines, and the delivery chain as a numbered list.

| Parameter property | Value |
|---|---|
| Type | `String` |
| Allowed values | `Markdown`, `Text` |
| Default value | `Markdown` |
| Supports wildcards | No |

| Parameter set (all) | Value |
|---|---|
| Position | Named |
| Required | No |
| Value from pipeline | No |
| Value from pipeline by property name | No |

### Inputs

`MailHeaderAnalyzer.Analysis`: the result of `Get-MailHeaderAnalysis`.

### Outputs

`System.String`: the report, one string per analysis object.

## Output object

`Get-MailHeaderAnalysis` returns one object of type `MailHeaderAnalyzer.Analysis` per input. The default view shows the summary from Example 1; all properties are available through `Select-Object`, `Format-List *`, or `ConvertTo-Json`.

### MailHeaderAnalyzer.Analysis

| Property | Type | Content |
|---|---|---|
| `Source` | String | File path, `Clipboard` or `Text` |
| `Subject` | String | Subject, RFC 2047 decoded |
| `From`, `ReplyTo`, `ReturnPath` | Address object | `Name`, `Address`, `Domain`, `Display`; `$null` if the field is missing |
| `Date` | DateTime (UTC) | Value of the `Date` field |
| `MessageId` | String | `Message-ID` |
| `MailFromDomain` | String | Envelope sender domain from `smtp.mailfrom` of the SPF check, otherwise from `Return-Path` |
| `Spf`, `Dkim`, `Dmarc`, `Arc`, `CompAuth` | String | Result according to the authoritative `Authentication-Results` line (`pass`, `fail`, `none`, `softfail` and others); `$null` if not checked |
| `CompAuthReason`, `CompAuthReasonMeaning` | String | Reason code of Microsoft 365 composite authentication and its meaning |
| `AuthTrust` | String | `Matched`, `Unmatched`, `Absent` or `None`, see [AuthTrust](#authtrust-herkunft-der-prüfergebnisse) |
| `AuthServId` | String | authserv-id of the authoritative verification line |
| `AuthenticationResults` | Object[] | All `Authentication-Results` lines with `AuthServId`, `Methods`, `Trust`, `Raw` |
| `ReceivedSpf` | Object | The `Received-SPF` line with `Result` and `Properties` |
| `SpfAlignment`, `DkimAlignment` | String | `Strict`, `Relaxed`, `None` or `$null` |
| `Hops` | Hop[] | Delivery chain in chronological order, see [Hop object](#mailheaderanalyzerhop) |
| `HopCount`, `TotalDuration`, `SlowestHopIndex`, `HasClockSkew` | Int, TimeSpan, Int, Bool | Chain metrics |
| `DeliveredBy` | String | `by` host of the most recent `Received` line, i.e., the delivering station |
| `DkimSignatures` | Object[] | Per signature: `Domain`, `Selector`, `Algorithm`, `Canonicalization`, `SignedHeaders`, `BodyLength`, `Timestamp`, `Expires`, `ReceiverResult`, `Tags` |
| `ArcChain`, `ArcValid` | Object[], Bool | ARC instances with `Instance`, `SealDomain`, `ChainValidation`, `Methods`; `ArcValid` is `$null` without an ARC chain |
| `Exchange` | Object | Exchange Online hybrid classification, see [Exchange object](#mailheaderanalyzerexchangeclassification); `$null` without corresponding headers |
| `Spam` | Object | Spam filter ratings, see [Spam object](#mailheaderanalyzerspamassessment); `$null` without corresponding headers |
| `List` | Object | `ListId`, `Unsubscribe`, `OneClick` (RFC 8058); `$null` without list headers |
| `Findings` | Finding[] | Anomalies with `Severity`, `Code`, `Message`, see [Findings](#findings) |
| `Fields` | Object[] | All fields with `Name`, `Value` (unfolded), and `Raw` |
| `HadBody` | Bool | Whether a message body followed the header |

### MailHeaderAnalyzer.Hop

Each entry in `Hops` corresponds to one `Received` line. The order is chronological, thus the reverse of the order in the header.

| Property | Content |
|---|---|
| `Index` | Sequence number, 1 = submission |
| `FromHost`, `Helo`, `ReverseDns`, `IPAddress`, `IsPrivateIP` | Details about the submitting system from the `from` part; the IP comes from the square brackets in the comment, the rDNS name from the preceding comment |
| `ByHost`, `Software` | Receiving system and its software (comment after `by`) |
| `Protocol`, `ProtocolClass` | `with` value and class according to RFC 3848: `TlsAuthenticated` (ESMTPSA), `Tls` (ESMTPS or TLS indication present), `Authenticated` (ESMTPA), `Plain`, `Http`, `Mapi`, `Local` |
| `TlsVersion`, `TlsCipher` | From Microsoft (`version=TLS1_2, cipher=…`), Postfix (`using TLSv1.3 with cipher …`), and Exim notation |
| `Id`, `For`, `Via` | Other `Received` components |
| `Date` | Timestamp after the semicolon, UTC |
| `Delay` | TimeSpan to the preceding hop; negative with clock skew |
| `Provider` | Identified provider or gateway based on host names (Microsoft 365, Google Workspace, SEPPmail, HIN, Proofpoint, Mimecast, and others) |
| `Attested` | `$true` only for the last hop: only this line was written by the receiving system itself; all lines below it were already in the message |
| `Raw` | The original line |

### AuthTrust: Origin of verification results

Any sender can write an `Authentication-Results` line into a message. Under RFC 8601, section 5, only the line from the receiving organization is authoritative, and its authserv-id must be assignable to a station in the delivery chain. The cmdlet compares each line's authserv-id with the `by` hosts in the `Received` chain (same domain or subdomain, always at a dot boundary, never a substring).

| Value | Meaning |
|---|---|
| `Matched` | The authserv-id occurs as a `by` host in the chain. Only such lines are included in `Spf`, `Dkim`, and `Dmarc` once at least one exists |
| `Unmatched` | The authserv-id does not occur in the chain. The results are displayed but considered an unsupported claim; finding `AuthUnverified` indicates this |
| `Absent` | The line has no authserv-id. Microsoft 365 writes its verification line in this form; it begins directly with `spf=` |
| `None` | No verification line present |

If the header contains verification lines from multiple origins, finding `AuthMixedOrigins` reports this. If a verification line is missing but a `Received-SPF` line is present, its result is adopted as `Spf` and labeled `ReceivedSpfOnly`.

### DMARC alignment

`SpfAlignment` compares the envelope sender domain with the `From` domain, while `DkimAlignment` compares the `d=` domain of the verified signature with the `From` domain. `Strict` means identical domains, `Relaxed` means the same organizational domain, and `None` means no match. The organizational domain is determined heuristically: the last two labels, or the last three for known multi-part suffixes such as `co.uk` or `com.au`. A complete Public Suffix List is not included.

### MailHeaderAnalyzer.ExchangeClassification

If the header contains `X-MS-Exchange-Organization-*`, `X-OriginatorOrg`, or `X-MS-Exchange-CrossTenant-*` fields, the cmdlet populates the `Exchange` property. The meanings follow the Exchange Team article “Demystifying hybrid mail flow”; background information is available in the article [Exchange Hybrid Headers: Internal or External?](/blog/exchange-hybrid-header-intern-extern).

| Property | Content |
|---|---|
| `Directionality`, `DirectionalityMeaning` | `Originating` or `Incoming` with explanation |
| `AuthAs`, `AuthAsMeaning` | `Internal` or `Anonymous` with implications for EOP filtering |
| `AuthSource` | Server that assigned the classification |
| `AuthMechanism`, `AuthMechanismMeaning` | Mechanism code. Only value 10 (Externally Secured) is publicly documented; for all other codes, the module explicitly notes that Microsoft does not document them |
| `OriginatorOrg` | Default domain of the sending tenant, the non-spoofable tenant characteristic when received from Microsoft 365 |
| `CrossTenantAuthAs`, `CrossTenantAuthSource`, `CrossTenantId` | Classification and tenant ID at the tenant boundary |
| `CrossTenantFromEntity`, `CrossTenantFromMeaning` | `Internet`, `Hosted`, or `HybridOnPrem` with explanation |
| `WrongTenantAttribution` | Value of `X-MS-Exchange-CrossTenant-OriginalAttributedTenantConnectingIp` if the message was assigned to another tenant |
| `OrganizationHeadersPreserved`, `CrossPremisesHeadersFiltered` | Markers for received organizational headers and organizational headers removed by the send connector, respectively |

### MailHeaderAnalyzer.SpamAssessment

The `Spam` property summarizes ratings from known filters. Values originate from third-party systems and are decoded but not assessed.

| Property | Source | Content |
|---|---|---|
| `Scl`, `SclMeaning` | `X-Forefront-Antispam-Report`, `X-Microsoft-Antispam` | Spam Confidence Level with meaning (-1 trusted, 0/1 not spam, 5/6 suspected spam, 9 very likely spam) |
| `Bcl` | `X-Microsoft-Antispam` | Bulk Complaint Level 0 through 9 |
| `Category`, `CategoryMeaning` | `CAT:` | Classification such as `SPM`, `PHSH`, `HPHSH`, `MALW`, `SPOOF`, `DIMP`, `UIMP`, `BULK` |
| `SpamFilterVerdict`, `SpamFilterMeaning` | `SFV:` | Filter result such as `NSPM`, `SPM`, `SKA` (Allowlist), `SKI` (intraorganizational) |
| `IPVerdict`, `IPVerdictMeaning` | `IPV:` | `CAL` (IP on connection allow list) or `NLI` (no reputation) |
| `Direction`, `DirectionMeaning` | `DIR:` | `INB`, `OUT`, `INT` |
| `ConnectingIP`, `Country` | `CIP:`, `CTRY:` | Submitting IP and country of origin |
| `Forefront` | `X-Forefront-Antispam-Report` | All key-value pairs in the field |
| `SpamAssassinScore`, `SpamAssassinTests` | `X-Spam-Status` | Score and triggered tests |
| `RspamdSymbols` | `X-Spamd-Result`, `X-Spam-Report` | Symbols with score |

The complete list of `compauth` reason codes is available in the article [Microsoft 365 compauth: Reason Codes](/blog/microsoft-365-compauth-reason-codes).

### Findings

The cmdlet returns anomalies as objects in `Findings`, each with `Severity` (`Info`, `Warning`, `Fail`), a stable `Code` for filters and scripts, and an explanation in `Message`.

| Code | Severity | Meaning |
|---|---|---|
| `DuplicateField` | Warning | A field that RFC 5322 limits to one instance (`From`, `Subject`, `Date`, `Message-ID` and others) occurs multiple times. Email clients and filters may select different instances; a known pattern in forgeries |
| `BidiControls` | Warning | Unicode text-direction control characters in a field. They reverse reading direction, so `fdp.exe` then appears as `exe.pdf`. The module displays them as `<U+202E>` |
| `HopOverflow` | Warning | More than 200 `Received` lines; the excess lines were not analyzed |
| `AuthUnverified` | Warning | The verification results have an authserv-id that does not occur in the delivery chain |
| `AuthMixedOrigins` | Warning | Verification lines from multiple origins are present |
| `ReceivedSpfForeign` | Warning | The `receiver=` of the `Received-SPF` line does not occur in the chain |
| `ReceivedSpfOnly` | Info | The SPF result originates only from `Received-SPF`, not from a verification line |
| `NoAuthResults` | Info | No verification results in the header |
| `DmarcFail` | Fail | DMARC did not pass according to the receiving server |
| `SpfNotPass` | Warning | SPF result `fail`, `softfail`, `permerror` or `temperror` |
| `DkimNotPass` | Warning | DKIM result `fail`, `permerror` or `temperror` without ARC witness |
| `DkimBrokenAfterForward` | Info | DKIM did not pass at the recipient, but an ARC seal from the same domain attests to a previously valid signature: typical of forwarding and mailing lists |
| `DkimWeakHash` | Warning | Signature with `rsa-sha1` (RFC 8301 classifies SHA-1 as obsolete) |
| `DkimBodyLength` | Warning | The `l=` tag limits the signed length of the message body; appended content is not covered |
| `DkimExpired` | Warning | The `x=` timestamp is in the past |
| `DkimFromUnsigned` | Warning | The `From` field is not included in `h=`, although RFC 6376 requires it |
| `ClockSkew` | Info | A hop has an earlier timestamp than its predecessor; delays are approximate only |
| `ReplyToMismatch` | Info | The `Reply-To` domain differs from the `From` domain; common for newsletters, but a phishing pattern |
| `SpfNotAligned` | Info | The envelope sender domain and `From` domain belong to different organizations; SPF therefore does not contribute to DMARC |
| `ExchangeWrongTenant` | Warning | The message was assigned to another tenant; the classic cause is an inbound connector from another tenant with the same certificate or IP addresses |
| `ExchangeHeadersFiltered` | Warning | The send connector removed the cross-premises headers (KB3212872) |
| `ExchangeExternallySecured` | Warning | AuthMechanism 10: received through a connector with “Externally Secured”; EOP filtering skipped |
| `SpamCategory` | Warning | Microsoft assigned a category other than `NONE` |
| `SpamConfidence` | Warning | SCL 5 or higher |

## How it works and limitations

The module reads what is in the header and infers what can be established without external queries. This results in several limitations:

- **No cryptographic verification.** DKIM signatures are not revalidated and DNS records are not queried. `Spf`, `Dkim`, and `Dmarc` are always the receiving server's judgment, supplemented by verification of whether that judgment originated from it at all.
- **Only the last `Received` line is substantiated.** The sender supplied all lines below it and could have composed them arbitrarily. `Attested` marks this distinction; delays for earlier hops are based on the information in those lines.
- **Organizational domains are heuristic.** For relaxed alignment, the module uses a short list of multi-part suffixes, not a complete Public Suffix List.
- **AuthMechanism is only partially documented.** Apart from value 10, Microsoft has not published the codes; the module does not invent meanings.
- **Character sets.** RFC 2047 values are decoded using the encodings known to .NET on the respective system. Unknown character sets remain unchanged.

## Privacy

A complete header contains internal host names, IP addresses, senders, recipients, and subjects. The article [Analyze email headers without uploading the email](/blog/e-mail-header-analysieren-ohne-upload) explains why this information does not belong in an online tool. The same promise applies to the module as to the browser version: no network access. The module's test suite includes a test that fails as soon as a network or DNS cmdlet appears in the source code.

## Source code and versions

The source code is available under the MIT License on [GitHub](https://github.com/pfstr/MailHeaderAnalyzer). The analysis logic is a port of the library also used by the Header Analyzer on this website; both share the test cases. Continuous Integration checks every change with PSScriptAnalyzer and Pester on Windows PowerShell 5.1, PowerShell 7 on Windows, Ubuntu, and macOS. Releases to the PowerShell Gallery are made automatically from versioned tags; changes for each version are listed in the [Changelog](https://github.com/pfstr/MailHeaderAnalyzer/blob/main/CHANGELOG.md).

I accept bugs and feature requests as [GitHub issues](https://github.com/pfstr/MailHeaderAnalyzer/issues). Please anonymize test headers before submitting bug reports; the included test cases use only example domains under RFC 2606 and addresses under RFC 5737.

## Sources

1.  [MailHeaderAnalyzer in the PowerShell Gallery](https://www.powershellgallery.com/packages/MailHeaderAnalyzer): package page with installation command and version history.

2.  [pfstr/MailHeaderAnalyzer on GitHub](https://github.com/pfstr/MailHeaderAnalyzer): source code, test suite, changelog, and issues.

3.  [Microsoft Learn: Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2?view=exchange-ps): template for the structure of this reference (syntax, description, examples, parameter properties).

4.  [RFC 8601: Message Header Field for Indicating Message Authentication Status](https://www.rfc-editor.org/rfc/rfc8601): structure of `Authentication-Results` and the rule that only the receiving organization's line is authoritative (section 5).

5.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): structure of `Received` lines and the recommendation of an upper limit as loop protection (section 6.3).

6.  [RFC 3848: ESMTP and LMTP Transmission Types Registration](https://www.rfc-editor.org/rfc/rfc3848): the `with` values `ESMTPS`, `ESMTPA`, and `ESMTPSA`.

7.  [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://www.rfc-editor.org/rfc/rfc6376): tags of `DKIM-Signature`, requirement to sign `From`.

8.  [RFC 8301: Cryptographic Algorithm and Key Usage Update to DKIM](https://www.rfc-editor.org/rfc/rfc8301): classification of `rsa-sha1` as obsolete.

9.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc7489): strict and relaxed alignment.

10.  [RFC 8617: The Authenticated Received Chain (ARC) Protocol](https://www.rfc-editor.org/rfc/rfc8617): `ARC-Seal`, `ARC-Authentication-Results`, and `cv=` chain validation.

11.  [Microsoft Learn: Anti-spam message headers in Microsoft 365](https://learn.microsoft.com/en-us/defender-office-365/message-headers-eop-mdo): meaning of SCL, BCL, CAT, SFV, IPV, and `compauth` reason codes.

12.  [Exchange Team Blog: Demystifying hybrid mail flow](https://techcommunity.microsoft.com/blog/exchange/demystifying-hybrid-mail-flow-when-is-a-message-internal/1420838): source for the meanings of MessageDirectionality, AuthAs, and AuthMechanism.

13.  [Header Analyzer on rafaelpfister.ch](/tools/header-analyzer): the browser version with the same analysis logic.
