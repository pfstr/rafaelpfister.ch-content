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
translationSourceHash: 41c3cb7c83800c0cc30f797ad6b2d9ec480d6c32bc054d2e01bd037b3dcd6c9b
translationModel: gpt-5.6-terra
translatedAt: 2026-09-26T08:54:50.629Z
translationReview: automatic
url: https://rafaelpfister.ch/en/blog/mailheaderanalyzer-analyze-email-headers-in-powershell-without-network-access
---

# MailHeaderAnalyzer: Analyze Email Headers in PowerShell Without Network Access

MailHeaderAnalyzer is a PowerShell module with two cmdlets. `Get-MailHeaderAnalysis` analyzes an email header: the delivery chain with delays and TLS details, SPF, DKIM, DMARC, and ARC results including origin verification against your gateway's authserv-id, DMARC alignment, Exchange Online hybrid classification, assessments from Microsoft Defender, SpamAssassin, and Rspamd, as well as anomalies such as duplicate `From` lines or Unicode control characters. `ConvertTo-MailHeaderReport` creates a report for tickets from it. The module operates fully offline: no DNS queries, no HTTP connections. It is the command-line version of the [Header Analyzer on this website](/tools/header-analyzer) and uses the same analysis logic.

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
| `Get-MailHeaderAnalysis` | `-TrustedAuthServId` | `String[]` | No | No | authserv-id(s) of your inbound gateway; only verification lines with one of these IDs count as substantiated |
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
| `-Scope CurrentUser` | Installs in the user's module directory without administrator privileges |

On a system without Internet access, download the module on another computer with `Save-Module -Name MailHeaderAnalyzer -Path C:\Temp` and copy the `MailHeaderAnalyzer` folder into a directory from `$env:PSModulePath`, for example `%USERPROFILE%\Documents\WindowsPowerShell\Modules` (Windows PowerShell 5.1) or `%USERPROFILE%\Documents\PowerShell\Modules` (PowerShell 7). `Update-Module -Name MailHeaderAnalyzer` retrieves updates, and `Get-Module -Name MailHeaderAnalyzer -ListAvailable` displays the installed version.

## Get-MailHeaderAnalysis

Analyzes an email header and returns an analysis object.

### Syntax

#### Text (default)

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

### Description

The cmdlet splits the raw header into fields, unfolds RFC 5322 folding, and decodes RFC 2047 values in subjects and addresses. From the `Received` lines, it builds the delivery chain in chronological order, calculates the delay at each station, and reads the TLS version, cipher, and protocol class according to RFC 3848. From `Authentication-Results`, `Received-SPF`, `DKIM-Signature` and the ARC chain, it determines the authentication results and assigns an origin to each verification line (RFC 8601, section 5): substantiated if its authserv-id is in `-TrustedAuthServId`, otherwise merely plausible or unsubstantiated; see [AuthTrust](#authtrust-herkunft-der-prüfergebnisse). It also includes DMARC alignment, Exchange Online hybrid classification, spam filter assessments, and a list of anomalies.

The cmdlet performs no DNS queries and opens no network connection. `Spf`, `Dkim`, `Dmarc`, and `Arc` are therefore always the receiving server's verdict. DKIM and ARC signatures are not cryptographically revalidated; for the ARC chain, the cmdlet checks only its structure (`ArcStructure`).

Input is read tolerantly: a blank line ends the header, and any subsequent message body is ignored. Lines without a field name and without leading whitespace, such as those created when copying from client dialogs, belong to the preceding field. An mbox separator line `From ...` before the first field is skipped, and a byte order mark is removed. At most 200 `Received` lines are analyzed, counted from delivery.

### Examples

#### Example 1

```powershell
Get-MailHeaderAnalysis -FromClipboard
```

Analyzes the header currently in the clipboard. In Outlook for Windows, find the header under File, Properties, Internet headers; in Outlook on the web, find it in message options under “View message details.”

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

Displays the delivery chain as a table, with the first hop first.

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

Reads the header line by line from a text file and displays only anomalies. `-Raw` with `Get-Content` is not required; the cmdlet combines the lines itself.

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
| `Select-Object … @{ Name; Expression }` | Calculated column that combines all finding codes into one string |
| `Export-Csv -NoTypeInformation -Encoding UTF8` | CSV without a type header, UTF-8 for umlauts in subject lines |

#### Example 6

```powershell
Get-ChildItem .\export\*.eml |
    Get-MailHeaderAnalysis -TrustedAuthServId 'mx.example.org' |
    Where-Object AuthTrust -ne 'Trusted' |
    Select-Object Source, AuthTrust, AuthServId, DeliveredBy
```

Lists messages whose verification results do not originate from your own inbound gateway `mx.example.org`. For phishing analysis, this is an initial filter. Without `-TrustedAuthServId`, you can filter only for `Unmatched`; a forgery that also includes a matching `Received` line then appears as `Matched` and passes through the filter.

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-TrustedAuthServId 'mx.example.org'` | authserv-id that your gateway writes in `Authentication-Results`; exact match, without subdomains |
| `Where-Object AuthTrust -ne 'Trusted'` | Retains all messages whose authoritative verification line does not have a trusted authserv-id |
| `Select-Object Source, AuthTrust, AuthServId, DeliveredBy` | File, origin level, authserv-id of the verification line, and delivering station |

</details>

#### Example 7

```powershell
$a = Get-MailHeaderAnalysis -Path .\nachricht.eml
$a.Hops[$a.SlowestHopIndex - 1] | Format-List Index, FromHost, ByHost, Delay
```

Displays the station with the longest delay. `SlowestHopIndex` is 1-based; the `Hops` array is 0-based.

#### Example 8

```powershell
Get-MailHeaderAnalysis -Path .\nachricht.eml | ConvertTo-Json -Depth 6 | Set-Content .\analyse.json
```

Writes the complete analysis as JSON. `-Depth 6` is required because `Hops`, `DkimSignatures` and `Exchange` are nested objects; the default value of 2 would output only their type names.

### Parameters

#### -Header

The header as text. The parameter accepts one string containing the complete header or multiple strings; pipeline input is collected and combined into one header at the end, which is why `Get-Content datei | Get-MailHeaderAnalysis` works without `-Raw`. To analyze multiple headers separately, use `-Path` with multiple files.

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

Path to a file containing the header or a complete `.eml` message. Relative paths are resolved against the current directory. The file is read with `[System.IO.File]::ReadAllText`: a byte order mark is honored; without a BOM, UTF-8 is used. Each file produces a separate result object; missing files produce a non-terminating error.

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

Reads the header from the clipboard with `Get-Clipboard -Raw`. The parameter is available only on Windows; on Linux and macOS, the cmdlet terminates with an error, as it also does when the clipboard is empty.

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

#### -TrustedAuthServId

The authserv-id or authserv-ids that your inbound gateway writes in `Authentication-Results`, for example `mx.example.org`. The cmdlet compares exactly, case-insensitively and without subdomains. Verification lines with one of these IDs receive `AuthTrust = Trusted`, and only they are then included in `Spf`, `Dkim`, `Dmarc`, and `Arc`. The same applies to the `receiver=` of a `Received-SPF` line.

The result is only as reliable as the gateway: it must remove incoming `Authentication-Results` lines that claim its own authserv-id (RFC 8601, section 5). Whether it does so cannot be determined from a header. If two lines with a trusted ID conflict, the cmdlet reports `AuthTrustedConflict`. Microsoft 365 does not write an authserv-id in its verification line; if Exchange Online receives your email, omit the parameter.

For an entire session, you can set the value as a default, for example in the PowerShell profile:

```powershell
$PSDefaultParameterValues['Get-MailHeaderAnalysis:TrustedAuthServId'] = 'mx.example.org'
```

| Parameter property | Value |
|---|---|
| Type | `String[]` |
| Default value | None |
| Supports wildcards | No |

| Parameter set (all) | Value |
|---|---|
| Position | Named |
| Required | No |
| Value from pipeline | No |
| Value from pipeline by property name | No |

### Inputs

`System.String`: Header lines or the complete header, bound to `-Header`.

`System.IO.FileInfo`: File objects from `Get-ChildItem`, whose `FullName` is bound to `-Path`.

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

The report includes the subject, sender, date, and Message-ID; authentication results with origin and alignment; all findings; the delivery chain; the Exchange classification; and spam filter values. Unicode directional control characters remain visible in the report as `<U+...>` so they cannot enter a ticket system through the report. The last line states the module version.

### Examples

#### Example 1

```powershell
Get-MailHeaderAnalysis -Path .\nachricht.eml | ConvertTo-MailHeaderReport | Set-Clipboard
```

Creates a Markdown report and places it on the clipboard.

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

Outputs the report as text without Markdown markup, for example for emails or console logs.

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

- `Markdown`: Headings, lists, and the delivery chain as a table. Default.
- `Text`: Headings in uppercase, indented lines, and the delivery chain as a numbered list.

| Parameter property | Value |
|---|---|
| Type | `String` |
| Valid values | `Markdown`, `Text` |
| Default value | `Markdown` |
| Supports wildcards | No |

| Parameter set (all) | Value |
|---|---|
| Position | Named |
| Required | No |
| Value from pipeline | No |
| Value from pipeline by property name | No |

### Inputs

`MailHeaderAnalyzer.Analysis`: the result from `Get-MailHeaderAnalysis`.

### Outputs

`System.String`: the report, one string per analysis object.

## Output object

`Get-MailHeaderAnalysis` returns one object of type `MailHeaderAnalyzer.Analysis` for each input. The default view shows the summary from Example 1; all properties are accessible through `Select-Object`, `Format-List *`, or `ConvertTo-Json`.

### MailHeaderAnalyzer.Analysis

| Property | Type | Content |
|---|---|---|
| `Source` | String | File path, `Clipboard` or `Text` |
| `Subject` | String | Subject, RFC 2047 decoded |
| `From`, `ReplyTo`, `ReturnPath` | Address object | `Name`, `Address`, `Domain`, `Display`; `$null` if the field is absent |
| `Date` | DateTime (UTC) | Value of the `Date` field |
| `MessageId` | String | `Message-ID` |
| `MailFromDomain` | String | Envelope sender domain from `smtp.mailfrom` of the SPF check, otherwise from `Return-Path` |
| `Spf`, `Dkim`, `Dmarc`, `Arc`, `CompAuth` | String | Result according to the authoritative `Authentication-Results` line (`pass`, `fail`, `none`, `softfail` and others); `$null` if not checked. `Arc` is the recipient's verdict on the ARC chain |
| `CompAuthReason`, `CompAuthReasonMeaning` | String | Reason code for Microsoft 365 composite authentication and its meaning |
| `AuthTrust` | String | `Trusted`, `Matched`, `Unmatched`, `Absent` or `None`, see [AuthTrust](#authtrust-herkunft-der-prüfergebnisse) |
| `AuthServId` | String | authserv-id of the authoritative verification line |
| `AuthenticationResults` | Object[] | All `Authentication-Results` lines with `AuthServId`, `Methods`, `Trust`, `Raw` |
| `ReceivedSpf` | Object | The `Received-SPF` line with `Result` and `Properties` |
| `SpfAlignment`, `DkimAlignment` | String | `Strict`, `Relaxed`, `None` or `$null` |
| `Hops` | Hop[] | Delivery chain in chronological order; see [Hop object](#mailheaderanalyzerhop) |
| `HopCount`, `TotalDuration`, `SlowestHopIndex`, `HasClockSkew` | Int, TimeSpan, Int, Bool | Chain metrics |
| `DeliveredBy` | String | `by` host of the most recent `Received` line, i.e., the delivering station |
| `DkimSignatures` | Object[] | Per signature: `Domain`, `Selector`, `Algorithm`, `Canonicalization`, `SignedHeaders`, `BodyLength`, `Timestamp`, `Expires`, `ReceiverResult`, `Tags` |
| `ArcChain` | Object[] | ARC instances with `Instance`, `SealDomain`, `ChainValidation`, `Methods` |
| `ArcStructure`, `ArcStructureIssues` | String, String[] | ARC chain structure: `Consistent`, `Inconsistent` or `$null` without ARC headers, plus detected deviations. Structure check only, no signature validation; see [ARC chain](#arc-kette-aufbau-und-urteil) |
| `Exchange` | Object | Exchange Online hybrid classification; see [Exchange object](#mailheaderanalyzerexchangeclassification); `$null` without corresponding headers |
| `Spam` | Object | Spam filter assessments; see [Spam object](#mailheaderanalyzerspamassessment); `$null` without corresponding headers |
| `List` | Object | `ListId`, `Unsubscribe`, `OneClick` (RFC 8058); `$null` without list headers |
| `Findings` | Finding[] | Anomalies with `Severity`, `Code`, `Message`, see [Findings](#findings) |
| `Fields` | Object[] | All fields with `Name`, `Value` (unfolded), and `Raw` |
| `HadBody` | Bool | Whether a message body followed the header |

### MailHeaderAnalyzer.Hop

Each entry in `Hops` corresponds to one `Received` line. The order is chronological, thus the reverse of the order in the header.

| Property | Content |
|---|---|
| `Index` | Sequential number, 1 = submission |
| `FromHost`, `Helo`, `ReverseDns`, `IPAddress`, `IsPrivateIP` | Details of the submitting system from the `from` part; the IP comes from the square brackets in the comment, the rDNS name from the preceding comment |
| `ByHost`, `Software` | Receiving system and its software (comment following `by`) |
| `Protocol`, `ProtocolClass` | `with` value and class according to RFC 3848: `TlsAuthenticated` (ESMTPSA), `Tls` (ESMTPS or TLS information present), `Authenticated` (ESMTPA), `Plain`, `Http`, `Mapi`, `Local` |
| `TlsVersion`, `TlsCipher` | From the formats used by Microsoft (`version=TLS1_2, cipher=…`), Postfix (`using TLSv1.3 with cipher …`) and Exim |
| `Id`, `For`, `Via` | Other `Received` components |
| `Date` | Timestamp after the semicolon, UTC |
| `Delay` | TimeSpan to the preceding hop; negative with clock skew |
| `Provider` | Detected provider or gateway based on host names (Microsoft 365, Google Workspace, SEPPmail, HIN, Proofpoint, Mimecast, and others) |
| `Attested` | `$true` only for the final hop: only this line was written by the receiving system itself; all lines below it were already in the message |
| `Raw` | The original line |

### AuthTrust: origin of verification results

An `Authentication-Results` line can be written into a message by any sender, as can a matching `Received` line. Therefore, the header alone cannot prove who wrote a verification line. Under RFC 8601, section 5, only the line whose authserv-id the receiving organization recognizes as its own is authoritative, and the inbound gateway must remove incoming lines with that ID. The cmdlet implements this rule with `-TrustedAuthServId`. Without this parameter, it compares the authserv-id only with the `by` hosts of the `Received` chain (same domain or subdomain, always at the dot boundary, no substring); this is a plausibility check.

| Value | Meaning |
|---|---|
| `Trusted` | The authserv-id is in `-TrustedAuthServId`. Once such a line exists, only lines at this level are included in `Spf`, `Dkim`, `Dmarc`, and `Arc`. Reliable provided that the gateway removes foreign lines with this ID |
| `Matched` | The authserv-id occurs as a `by` host in the chain. Plausible but not proof: a forgery can include the matching `Received` line. Without `-TrustedAuthServId`, finding `AuthPlausibleOnly` indicates this. Lines at this level count when no `Trusted` line is present |
| `Unmatched` | The authserv-id is neither trusted nor found in the chain. The results are displayed but treated as an unsubstantiated claim; finding `AuthUnverified` indicates this |
| `Absent` | The line has no authserv-id. Microsoft 365 writes its verification line in this form; it begins directly with `spf=` |
| `None` | No verification line present |

If the header contains verification lines from multiple origins, finding `AuthMixedOrigins` reports this. If a verification line is absent but a `Received-SPF` line is present, its result is adopted as `Spf` and marked with `ReceivedSpfOnly`.

With `-TrustedAuthServId`, two findings are added: `AuthNotTrusted` if no verification line has a trusted ID, and `AuthTrustedConflict` if two such lines report different results for `spf`, `dmarc`, `arc`, or `compauth`. The latter case means that at least one line does not originate from the gateway and the gateway did not remove it. In that case, the cmdlet adopts the topmost line; which of the two is genuine cannot be determined from the header. DKIM is excluded from this comparison because multiple signatures can legitimately have different results.

### ARC chain: structure and verdict

For ARC, the cmdlet provides two separate details. `Arc` is the `arc=` result from the authoritative verification line, meaning the verdict of the recipient that verified the chain signatures. `ArcStructure` is the module's own check and concerns only the structure specified by RFC 8617: consecutive instance numbers `i=1` through `i=n` (maximum 50), exactly one `ARC-Seal`, one `ARC-Message-Signature`, and one `ARC-Authentication-Results` per instance, plus `cv=none` for instance 1 and `cv=pass` for all subsequent instances. The module does not revalidate signatures; `Consistent` therefore says nothing about whether the chain is genuine. Deviations appear in `ArcStructureIssues` and finding `ArcStructureInconsistent`.

The information in `ARC-Authentication-Results` consists of statements by the respective forwarder. Finding `DkimBrokenAfterForward` therefore classifies a DKIM failure as the result of forwarding only if the recipient itself reports `arc=pass` and an earlier instance recorded a DKIM `pass` for the same domain.

### DMARC alignment

`SpfAlignment` compares the envelope sender domain with the `From` domain; `DkimAlignment` compares the `d=` domain of the verified signature with the `From` domain. `Strict` means identical domain, `Relaxed` means the same organizational domain, and `None` means no match. The organizational domain is determined heuristically: the final two labels, or the final three for known multi-part suffixes such as `co.uk` or `com.au`. A complete Public Suffix List is not included.

### MailHeaderAnalyzer.ExchangeClassification

If the header contains fields `X-MS-Exchange-Organization-*`, `X-OriginatorOrg` or `X-MS-Exchange-CrossTenant-*`, the cmdlet populates the `Exchange` property. The meanings follow the Exchange Team article “Demystifying hybrid mail flow”; background information is available in the article [Exchange hybrid headers: internal or external?](/blog/exchange-hybrid-header-intern-extern).

| Property | Content |
|---|---|
| `Directionality`, `DirectionalityMeaning` | `Originating` or `Incoming` with explanation |
| `AuthAs`, `AuthAsMeaning` | `Internal` or `Anonymous` with implications for EOP filtering |
| `AuthSource` | Server that assigned the classification |
| `AuthMechanism`, `AuthMechanismMeaning` | Mechanism code. Only value 10 (Externally Secured) is publicly documented; for all other codes, the module explicitly states that Microsoft does not document them |
| `OriginatorOrg` | Default domain of the sending tenant, the non-forgeable tenant attribute when receiving from Microsoft 365 |
| `CrossTenantAuthAs`, `CrossTenantAuthSource`, `CrossTenantId` | Classification and tenant ID at the tenant boundary |
| `CrossTenantFromEntity`, `CrossTenantFromMeaning` | `Internet`, `Hosted` or `HybridOnPrem` with explanation |
| `WrongTenantAttribution` | Value of `X-MS-Exchange-CrossTenant-OriginalAttributedTenantConnectingIp` if the message was assigned to another tenant |
| `OrganizationHeadersPreserved`, `CrossPremisesHeadersFiltered` | Markers for organization headers received or removed by the sending connector, respectively |

### MailHeaderAnalyzer.SpamAssessment

The `Spam` property combines assessments from known filters. Values originate from external systems and are decoded but not evaluated.

| Property | Source | Content |
|---|---|---|
| `Scl`, `SclMeaning` | `X-Forefront-Antispam-Report`, `X-Microsoft-Antispam` | Spam Confidence Level with meaning (-1 trusted, 0/1 not spam, 5/6 suspected spam, 9 very likely spam) |
| `Bcl` | `X-Microsoft-Antispam` | Bulk Complaint Level 0 through 9 |
| `Category`, `CategoryMeaning` | `CAT:` | Classification such as `SPM`, `PHSH`, `HPHSH`, `MALW`, `SPOOF`, `DIMP`, `UIMP`, `BULK` |
| `SpamFilterVerdict`, `SpamFilterMeaning` | `SFV:` | Filter result such as `NSPM`, `SPM`, `SKA` (allowlist), `SKI` (intra-organizational) |
| `IPVerdict`, `IPVerdictMeaning` | `IPV:` | `CAL` (IP on connection allowlist) or `NLI` (no reputation) |
| `Direction`, `DirectionMeaning` | `DIR:` | `INB`, `OUT`, `INT` |
| `ConnectingIP`, `Country` | `CIP:`, `CTRY:` | Submitting IP and country of origin |
| `Forefront` | `X-Forefront-Antispam-Report` | All key-value pairs in the field |
| `SpamAssassinScore`, `SpamAssassinTests` | `X-Spam-Status` | Score and triggered tests |
| `RspamdSymbols` | `X-Spamd-Result`, `X-Spam-Report` | Symbols with scores |

The complete list of `compauth` reason codes is available in the article [Microsoft 365 compauth: Reason Codes](/blog/microsoft-365-compauth-reason-codes).

### Findings

The cmdlet returns anomalies as objects in `Findings`, each with `Severity` (`Info`, `Warning`, `Fail`), a stable `Code` for filters and scripts, and an explanation in `Message`.

| Code | Severity | Meaning |
|---|---|---|
| `DuplicateField` | Warning | A field limited by RFC 5322 to one instance (`From`, `Subject`, `Date`, `Message-ID` and others) occurs multiple times. Email clients and filters may select different instances; a known forgery pattern |
| `BidiControls` | Warning | Unicode directional control characters in a field. They reverse reading direction, making `fdp.exe` appear as `exe.pdf`. The module displays them as `<U+202E>` |
| `HopOverflow` | Warning | More than 200 `Received` lines; the excess lines were not analyzed |
| `AuthPlausibleOnly` | Info | The authserv-id occurs in the delivery chain, but `-TrustedAuthServId` was not specified: plausible, not proof |
| `AuthNotTrusted` | Warning | `-TrustedAuthServId` was specified, but no verification line has one of these IDs |
| `AuthTrustedConflict` | Warning | Two verification lines with a trusted ID report different results for the same method; the gateway apparently does not remove foreign lines |
| `AuthUnverified` | Warning | The verification results have an authserv-id that is neither trusted nor present in the delivery chain |
| `AuthMixedOrigins` | Warning | Verification lines from multiple origins are present |
| `ReceivedSpfForeign` | Warning | The `receiver=` of the `Received-SPF` line does not occur in the chain |
| `ReceivedSpfOnly` | Info | The SPF result comes only from `Received-SPF`, not from a verification line |
| `NoAuthResults` | Info | No verification results in the header |
| `DmarcFail` | Fail | DMARC failed according to the receiving server |
| `SpfNotPass` | Warning | SPF result `fail`, `softfail`, `permerror` or `temperror` |
| `DkimNotPass` | Warning | DKIM result `fail`, `permerror` or `temperror`, without the recipient confirming the ARC chain |
| `DkimBrokenAfterForward` | Info | DKIM failed at the recipient, but the recipient reports `arc=pass`, and an earlier ARC instance recorded DKIM `pass` for the same domain: typical of forwarding and mailing lists |
| `DkimWeakHash` | Warning | Signature with `rsa-sha1` (RFC 8301 classifies SHA-1 as obsolete) |
| `DkimBodyLength` | Warning | The `l=` tag limits the signed length of the message body; appended content is not covered |
| `DkimExpired` | Warning | The `x=` timestamp is in the past |
| `DkimFromUnsigned` | Warning | The `From` field is not included in `h=`, although RFC 6376 requires it |
| `ArcStructureInconsistent` | Warning | The ARC headers do not form a formally complete chain (gaps, missing or duplicate headers, incorrect `cv=` sequence); structure check only |
| `ClockSkew` | Info | A hop has an earlier timestamp than its predecessor; delays are only approximate values |
| `ReplyToMismatch` | Info | `Reply-To` domain differs from the `From` domain; common for newsletters, a phishing pattern |
| `SpfNotAligned` | Info | Envelope sender domain and `From` domain belong to different organizations; SPF therefore does not contribute to DMARC |
| `ExchangeWrongTenant` | Warning | Message was assigned to another tenant; a classic cause is an inbound connector from another tenant using the same certificate or IP addresses |
| `ExchangeHeadersFiltered` | Warning | The sending connector removed the cross-premises headers (KB3212872) |
| `ExchangeExternallySecured` | Warning | AuthMechanism 10: received through a receive connector with “Externally Secured,” EOP filtering skipped |
| `SpamCategory` | Warning | Microsoft assigned a category other than `NONE` |
| `SpamConfidence` | Warning | SCL 5 or higher |

## How it works and limitations

The module reads what is in the header and derives what can be substantiated without external queries. This entails several limitations:

- **No cryptographic validation.** DKIM and ARC signatures are not revalidated, and DNS records are not queried. `Spf`, `Dkim`, `Dmarc`, and `Arc` are always the receiving server's verdict; `ArcStructure` checks only the chain structure.
- **Origin can be substantiated only with knowledge of the gateway.** Without `-TrustedAuthServId`, `AuthTrust` is a plausibility check. With the parameter, the statement depends on the gateway removing foreign verification lines with its authserv-id; the module cannot verify this.
- **Only the final `Received` line is substantiated.** The sender supplied all lines below it and can construct them arbitrarily. `Attested` marks this distinction; delays at earlier hops rely on the information in those lines.
- **Organizational domains are heuristic.** For relaxed alignment, the module uses a short list of multi-part suffixes rather than a complete Public Suffix List.
- **AuthMechanism is only partly documented.** Apart from value 10, Microsoft has not published the codes; the module does not invent meanings.
- **Character sets.** RFC 2047 values are decoded using encodings recognized by .NET on the respective system. Unknown character sets remain unchanged.

## Privacy

A complete header contains internal host names, IP addresses, senders, recipients, and subjects. The article [Analyze email headers without uploading the email](/blog/e-mail-header-analysieren-ohne-upload) explains why this information does not belong in an online tool. The same promise applies to the module as to the browser version: no network access. The module's test suite includes a test that fails as soon as a network or DNS cmdlet appears in the source code.

## Source code and versions

The source code is available under the MIT License on [GitHub](https://github.com/pfstr/MailHeaderAnalyzer). The analysis logic is a port of the library also used by the Header Analyzer on this website; both share test cases. Continuous integration checks every change using PSScriptAnalyzer and Pester on Windows PowerShell 5.1, PowerShell 7 on Windows, Ubuntu, and macOS. Releases to the PowerShell Gallery are automatically published from versioned tags; changes in each version are listed in the [changelog](https://github.com/pfstr/MailHeaderAnalyzer/blob/main/CHANGELOG.md).

Version 0.2.0 from September 26, 2026 tightened the origin model following a review note by @saltyslugga: parameter `-TrustedAuthServId`, level `Trusted`, and `Matched` now only as plausibility. The `ArcValid` property was removed and replaced with `ArcStructure` and `ArcStructureIssues`; scripts that evaluate `ArcValid` must be updated.

I accept bugs and feature requests as [issues on GitHub](https://github.com/pfstr/MailHeaderAnalyzer/issues). Please anonymize test headers before submitting bug reports; the included test cases use only example domains under RFC 2606 and addresses under RFC 5737.

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

10.  [RFC 8617: The Authenticated Received Chain (ARC) Protocol](https://www.rfc-editor.org/rfc/rfc8617): chain structure consisting of `ARC-Seal`, `ARC-Message-Signature`, and `ARC-Authentication-Results`, instance numbers, and `cv=` values.

11.  [Microsoft Learn: Anti-spam message headers in Microsoft 365](https://learn.microsoft.com/en-us/defender-office-365/message-headers-eop-mdo): meaning of SCL, BCL, CAT, SFV, IPV, and `compauth` reason codes.

12.  [Exchange Team Blog: Demystifying hybrid mail flow](https://techcommunity.microsoft.com/blog/exchange/demystifying-hybrid-mail-flow-when-is-a-message-internal/1420838): source of the meanings of MessageDirectionality, AuthAs, and AuthMechanism.

13.  [Header Analyzer on rafaelpfister.ch](/tools/header-analyzer): the browser version with the same analysis logic.
