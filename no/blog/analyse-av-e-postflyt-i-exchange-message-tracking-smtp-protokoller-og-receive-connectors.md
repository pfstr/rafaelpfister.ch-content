---
title: "Analyzing Exchange mail flow: Message tracking, SMTP logs and receive connectors"
navTitle: "Analyze mail flow"
description: "How to systematically find out where a message ended up in Exchange OnPrem, Hybrid and Exchange Online: queries with example output, how to read SMTP logs correctly, and the pitfalls that regularly lead to false conclusions."
date: "2026-08-11"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "22 min read"
themen:
  - exchange-onprem-hybrid
  - smtp-mailflow
  - microsoft-365-exchange
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange"
  - "exchange-on-premises"
  - "exchange-hybrid"
  - "exchange-online"
protokolle:
  - "smtp"
  - "powershell"
  - "troubleshooting"
related:
  - typische-ursachen-fuer-mail-loops-und-deren-behebung
  - einliefernde-ip-adressen-aggregieren
  - mailflow-fehlersuche-kontrollierte-experimente
slug: "analyse-av-e-postflyt-i-exchange-message-tracking-smtp-protokoller-og-receive-connectors"
translationId: "article-ad93c41ab6cd20e6"
draft: false
translationOf: exchange-message-tracking-und-receive-connectoren-analysieren
translationSourceHash: 867f3a9a27b8db05d2721cce6282149f4616c4170576d0ab3f39a3cbde3701e3
translationModel: gpt-5.6-terra
translatedAt: 2026-10-30T03:28:35.532Z
translationReview: required
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/no/blog/analyse-av-e-postflyt-i-exchange-message-tracking-smtp-protokoller-og-receive-connectors
---

# Analyzing Exchange mail flow: Message tracking, SMTP logs and receive connectors

The most common question in mail operations is: A message did not arrive—where did it go? Message tracking answers this reliably, but only if you know what it does **not** tell you. This article describes the procedure in the order that has proven effective, shows the typical output for each query, and identifies the sources of error that regularly cost hours because they suggest plausible but incorrect conclusions.

All examples use generic names: `SRV-MAIL01` and `SRV-MAIL02` as transport servers, `example.com` as the domain. If you want to assemble the commands for your environment rather than type them: the [Command Generator](https://rafaelpfister.ch/tools/command-builder) contains the common message tracking and capture commands for PowerShell and Unix shells side by side, entirely locally in the browser.

## The principle: locate first, then explain

The instinct is to look for the cause immediately. It is more efficient to first determine how far the message got at all. This drastically narrows the search space in one step, because you then know whether to look in your own system, at the upstream gateway, or at the destination.

The order is therefore: find the message, read the last event, read the reason for the error, determine whether it is an isolated case or a pattern, and only then reconstruct the delivery path.

## Step 1: Find the message

Start with the recipient, because you almost always know it. It is important to run the query across **all** transport servers, not just one.

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object {
    Get-MessageTrackingLog -Server $_ `
        -Start (Get-Date).AddHours(-6) `
        -ResultSize Unlimited `
        -Recipients "empfaenger@example.com"
} | Sort-Object Timestamp |
    Format-Table Timestamp, ServerHostname, EventId, Source, ConnectorId, MessageId `
        -AutoSize -Wrap
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-Server` | Transport server whose tracking log is queried; here, both servers are processed sequentially through the pipeline |
| `-Start` | Lower time limit for the search, here the last six hours |
| `-ResultSize Unlimited` | Removes the default limit of 1,000 entries |
| `-Recipients` | Filters for messages to this recipient address |
| `Sort-Object Timestamp` | Sorts the combined results from both servers chronologically |
| `-AutoSize -Wrap` | Adjusts column widths to the content and wraps long values instead of truncating them |

</details>

Typical output for a message that passed through cleanly:

```text
Timestamp           ServerHostname EventId      Source  ConnectorId
---------           -------------- -------      ------  -----------
11.08.2026 08:27:15 SRV-MAIL02     HARECEIVE    SMTP
11.08.2026 08:27:15 SRV-MAIL01     RECEIVE      SMTP    SRV-MAIL01\Default SRV-MAIL01
11.08.2026 08:27:15 SRV-MAIL01     HAREDIRECT   SMTP
11.08.2026 08:27:15 SRV-MAIL01     RESOLVE      ROUTING
11.08.2026 08:27:15 SRV-MAIL01     AGENTINFO    AGENT
11.08.2026 08:27:16 SRV-MAIL01     SENDEXTERNAL SMTP    Outbound-to-O365
11.08.2026 08:27:53 SRV-MAIL02     HADISCARD    SMTP
```

If the query finds nothing, check whether the recipient was expanded through a distribution group. In that case, search by sender instead:

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object {
    Get-MessageTrackingLog -Server $_ `
        -Start (Get-Date).AddHours(-6) `
        -ResultSize Unlimited
} | Where-Object { $_.Sender -like "*@example.com" } |
    Sort-Object Timestamp |
    Format-Table Timestamp, EventId, Sender,
        @{n='To'; e={$_.Recipients -join ','}}, MessageSubject -AutoSize -Wrap
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-Server` | Transport server whose tracking log is queried |
| `-Start` | Lower time limit for the search |
| `-ResultSize Unlimited` | Removes the default limit of 1,000 entries |
| `Where-Object` | Filters client-side for senders from your own domain, since `-Sender` accepts only exact addresses |
| `@{n=…; e=…}` | Calculated column: combines the collection field `Recipients` into a comma-separated string |

</details>

## Step 2: Read the last event

The entire diagnosis depends on the message's **last** `EventId`. It tells you which search area to investigate next.

| Last EventId | Meaning | Next step |
|---|---|---|
| `RECEIVE`, then nothing | Message is stuck | Check queues |
| `SEND` or `SENDEXTERNAL` | Successfully handed off | Continue searching at the next hop |
| `FAIL` | Permanently failed | Read the reason in `RecipientStatus` |
| `DEFER` | Retry in progress | Check the queue and destination system |
| `DROP` or `POISONMESSAGE` | Discarded | Check transport rule or agent |
| `DELIVER` | Delivered to a local mailbox | Check mailbox rules |
| `RESOLVE` | Recipient was rewritten | Read the destination address in the entry |

`RESOLVE` is the most informative intermediate step in hybrid environments, because this is where the rewrite to the cloud routing address is visible:

```text
EventId : RESOLVE
Source  : ROUTING
Sender  : absender@example.com
To      : BENUTZER@example.mail.onmicrosoft.com
```

If the expected `onmicrosoft.com` address appears there, the recipient object is configured correctly and you can rule this out. If the original address still appears, the local object lacks the target address and Exchange attempts local delivery.

If the message is stuck, the queue usually shows the reason in plain text:

```powershell
Get-Queue -Server SRV-MAIL01 |
    Where-Object { $_.MessageCount -gt 0 } |
    Format-Table Identity, DeliveryType, Status, MessageCount, NextHopDomain, LastError `
        -AutoSize -Wrap
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-Server` | Server whose transport queues are queried |
| `Where-Object` | Hides empty queues and shows only queues with pending messages |
| `-AutoSize -Wrap` | Prevents the long `LastError` column from being truncated |

</details>

## Source of error 1: Tracking is server-specific, and many entries are shadow copies

If you see pairs of `HARECEIVE` and `HADISCARD` in the output, often with the addition `ExplicitlyDiscarded`, this server did **not process** the message. It merely held a shadow copy as part of Shadow Redundancy, while another server handled the actual delivery. Once the primary server reports success, the partner discards its copy.

This is what it looks like when you queried only the wrong server:

```text
Timestamp           EventId    SourceContext
---------           -------    -------------
11.08.2026 09:47:13 HARECEIVE  1a2aae6b-f3a3-4e04-9233-7de460b92223
11.08.2026 09:48:07 HADISCARD  ExplicitlyDiscarded;1a2aae6b-f3a3-4e04-9233-7de460b92223
```

Two lines, no error, no delivery. Anyone concluding from this that the message disappeared is looking in the wrong place. The actual processing is in the partner server's tracking log.

In practice, this means two things. First, such lines are not an indication of a problem, but normal operation. Second, you must query all transport servers.

## Source of error 2: `Format-Table` truncates exactly the crucial columns

The reason for the error is in `RecipientStatus`, and this field is long. In a table, it either disappears entirely or is truncated. This is exactly what causes you to see `FAIL` but not the reason, and start guessing instead.

As soon as you find an error case, therefore switch to `Format-List` and expand the collection fields:

```powershell
Get-MessageTrackingLog -Server SRV-MAIL01 `
    -Start (Get-Date).AddHours(-6) `
    -ResultSize Unlimited `
    -Recipients "empfaenger@example.com" `
    -EventId FAIL |
  Format-List Timestamp, Sender,
    @{n='To';     e={$_.Recipients -join ','}},
    @{n='Status'; e={$_.RecipientStatus -join ' | '}},
    MessageSubject, MessageId, SourceContext
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-Server` | Transport server whose tracking log is queried |
| `-Start` | Lower time limit for the search |
| `-ResultSize Unlimited` | Removes the default limit of 1,000 entries |
| `-Recipients` | Filters for messages to this recipient address |
| `-EventId FAIL` | Only entries with a final delivery failure |
| `Format-List` | Displays every field on its own line at full length; nothing is truncated |
| `@{n=…; e=…}` | Calculated fields: expand the collection fields `Recipients` and `RecipientStatus` into readable strings |

</details>

And this is what the difference looks like. First, the table view that explains nothing:

```text
Timestamp           EventId ConnectorId
---------           ------- -----------
11.08.2026 09:47:13 FAIL    Outbound-to-O365
```

Then the same message as a list:

```text
Timestamp      : 11.08.2026 09:47:13
Sender         : dienst@example-test.com
To             : BENUTZER@example.mail.onmicrosoft.com
Status         : [{LED=550 5.1.8 Access denied, bad outbound sender AS(42000001)
                 [XX1PEPF00000000.eurprd02.prod.outlook.com]};{MSG=};
                 {FQDN=10.0.0.40};{IP=10.0.0.40};{LRT=11.08.2026 09:47:13}]
MessageSubject : Statusmeldung Nachtlauf
MessageId      : <1897281176.1319@app01.intern.example.com>
```

The diagnosis is now clear without needing a single assumption: The remote side objects to the sender. `LED` contains the complete SMTP response, `FQDN` and `IP` identify the system that responded, and `LRT` the time of the last attempt.

## Step 3: Isolated case or pattern?

Before you delve into a single case, clarify the scope. This one query determines whether you are dealing with a minor issue or an incident:

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object {
    Get-MessageTrackingLog -Server $_ `
        -Start (Get-Date).AddHours(-8) `
        -EventId FAIL -ResultSize Unlimited
} | Where-Object { ($_.RecipientStatus -join '') -like "*5.1.8*" } |
    Group-Object { ($_.Sender -split '@')[-1] } |
    Sort-Object Count -Descending |
    Format-Table Count, Name -AutoSize
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-Start` | Lower time limit, here the last eight hours |
| `-EventId FAIL` | Only permanently failed deliveries |
| `-ResultSize Unlimited` | Removes the default limit of 1,000 entries |
| `Where-Object` | Filters for the SMTP status code under investigation in the `RecipientStatus` field |
| `Group-Object` | Groups by sender domain (the part after `@`) |
| `Sort-Object Count -Descending` | Most frequent domain at the top |

</details>

Replace `5.1.8` with the status code you are investigating. The output answers the question in one line:

```text
Count Name
----- ----
    9 example-test.com
```

A single sender domain means: a narrowly scoped problem, not an incident; you can continue investigating at your leisure. If twenty different domains appeared there, you would have an ongoing outage and everything else would have to wait. Making this distinction early saves the most time in practice.

## Source of error 3: The `ConnectorId` does not name the actual receive connector

This is the costliest source of error because the output looks credible. Mail submitted by a client or external system on port 25 first reaches the **Front End Transport**. This passes the message on to the **Transport Service** on port 2525. Message tracking is written only there; Front End Transport does not write its own tracking.

You can see the consequence in this line:

```text
EventId        : RECEIVE
ConnectorId    : SRV-MAIL01\Default SRV-MAIL01
ClientIp       : 10.0.1.11
ClientHostname : srv-mail01.intern.example.com
```

The `ConnectorId` names the internal connector on port 2525, and the `ClientIp` is the address of the **proxying server**, not that of the original submitting system. Which configured connector on port 25 a system actually reached simply does not appear in tracking. Anyone who trusts this value will look for the error in a connector that was not involved at all.

There are two ways to get the answer. The first is reconstruction through the configuration:

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object { Get-ReceiveConnector -Server $_ } |
    Format-List Identity, Enabled,
        @{n='Bindings';       e={$_.Bindings -join ','}},
        @{n='RemoteIPRanges'; e={$_.RemoteIPRanges -join ','}},
        PermissionGroups, AuthMechanism
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-Server` | Server whose receive connectors are listed |
| `Format-List` | Full field lengths; `RemoteIPRanges` and `PermissionGroups` would be truncated in tables |
| `@{n=…; e=…}` | Calculated fields: combines the collection fields `Bindings` and `RemoteIPRanges` into comma-separated strings |

</details>

```text
Identity         : SRV-MAIL01\Default Frontend SRV-MAIL01
Bindings         : 10.0.1.11:25
RemoteIPRanges   : 0.0.0.0-255.255.255.255
PermissionGroups : AnonymousUsers, ExchangeServers, ExchangeLegacyServers
AuthMechanism    : Tls, Integrated, BasicAuth, BasicAuthRequireTLS, ExchangeServer

Identity         : SRV-MAIL01\smtp-noauth SRV-MAIL01
Bindings         : 10.0.1.13:25
RemoteIPRanges   : 10.0.20.22,10.0.21.11,10.0.21.12
PermissionGroups : AnonymousUsers, Custom
AuthMechanism    : Tls
```

Determine the source IP of the submitting system and find the connector whose `RemoteIPRanges` contains it. If it falls within none of the restricted connectors, the default frontend connector remains, which usually accepts the entire address space. Use `Format-List` here too, because `RemoteIPRanges` and `PermissionGroups` are regularly truncated in tables.

The second way is the SMTP log, and it deserves its own section.

## The SMTP log: the only complete source

The Front End Transport log records the complete SMTP session: which connector was addressed, which IP connected, and what the client and server said to each other. It is the only source that resolves the issue described above with `ConnectorId`.

### Enable logging

By default, logging is **disabled** on most connectors. Enable it per connector:

```powershell
Set-ReceiveConnector -Identity "SRV-MAIL01\Default Frontend SRV-MAIL01" `
    -ProtocolLoggingLevel Verbose
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-Identity` | The connector to modify in the form `Server\Connectorname` |
| `-ProtocolLoggingLevel Verbose` | Enables SMTP logging; `None` disables it again |

</details>

For outgoing connections, use `Set-SendConnector` accordingly. Remember to set the value back to `None` after the analysis, because detailed logging consumes disk space and writes considerable amounts of data under high load.

### Where the files are located

Exchange separates logs by service and direction. There is no need to hard-code the paths; query them:

```powershell
Get-FrontendTransportService SRV-MAIL01 |
    Format-List ReceiveProtocolLogPath, SendProtocolLogPath,
        ReceiveProtocolLogMaxAge, ReceiveProtocolLogMaxDirectorySize

Get-TransportService SRV-MAIL01 |
    Format-List ReceiveProtocolLogPath, SendProtocolLogPath
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `SRV-MAIL01` | Positional parameter `-Identity`: the server to query |
| `ReceiveProtocolLogPath`, `SendProtocolLogPath` | Storage paths for logs of incoming and outgoing connections respectively |
| `ReceiveProtocolLogMaxAge` | Maximum age of log files; older files are deleted |
| `ReceiveProtocolLogMaxDirectorySize` | Maximum disk space usage of the log directory |

</details>

They are typically located below the installation path in `TransportRoles\Logs\FrontEnd\ProtocolLog\SmtpReceive` for Front End Transport and in `TransportRoles\Logs\Hub\ProtocolLog\SmtpReceive` for the Transport Service. **This is the key point:** You will find client connections on port 25 exclusively in the `FrontEnd` path; the `Hub` path contains only internal forwarding traffic on 2525.

Note the retention settings. `ReceiveProtocolLogMaxAge` is often set to 30 days, while `ReceiveProtocolLogMaxDirectorySize` additionally limits disk usage. Under high load, the size limit takes effect considerably earlier than the age limit, and your logs may then be only a few days old.

### Understanding the format

The files are CSV with header lines beginning with `#`. The most important columns are `date-time`, `connector-id`, `session-id`, `sequence-number`, `local-endpoint`, `remote-endpoint`, `event` and `data`.

The decisive column is `event`, a single character:

| Character | Meaning |
|---|---|
| `+` | Connection established |
| `-` | Connection closed |
| `>` | Server sends to client |
| `<` | Client sends to server |
| `*` | Server information, not SMTP traffic |

You can identify a session by the shared `session-id`; `sequence-number` gives the order within the session. A typical excerpt looks like this:

```text
2026-08-11T09:47:10.4Z,SRV-MAIL01\smtp-noauth SRV-MAIL01,08DEF44EC454A414,0,
  10.0.1.13:25,10.0.20.22:51244,+,,
2026-08-11T09:47:10.4Z,SRV-MAIL01\smtp-noauth SRV-MAIL01,08DEF44EC454A414,1,
  10.0.1.13:25,10.0.20.22:51244,>,"220 srv-mail01.intern.example.com Microsoft ESMTP",
2026-08-11T09:47:10.5Z,SRV-MAIL01\smtp-noauth SRV-MAIL01,08DEF44EC454A414,2,
  10.0.1.13:25,10.0.20.22:51244,<,EHLO app01.intern.example.com,
2026-08-11T09:47:10.6Z,SRV-MAIL01\smtp-noauth SRV-MAIL01,08DEF44EC454A414,6,
  10.0.1.13:25,10.0.20.22:51244,<,MAIL FROM:<dienst@example-test.com>,
2026-08-11T09:47:10.7Z,SRV-MAIL01\smtp-noauth SRV-MAIL01,08DEF44EC454A414,8,
  10.0.1.13:25,10.0.20.22:51244,>,"250 2.1.5 Recipient OK",
```

It contains everything missing from message tracking: the **actual** connector (`smtp-noauth`), the **actual** source IP (`10.0.20.22`) and the name the system uses in `EHLO`.

### Search specifically

For individual cases, a text filter is significantly faster than object parsing. Search for the sender address or the `EHLO` name and retrieve the session ID:

```powershell
$pfad = (Get-FrontendTransportService SRV-MAIL01).ReceiveProtocolLogPath
Select-String -Path "$pfad\*.log" -Pattern "dienst@example-test.com" -SimpleMatch |
    Select-Object -First 5 Line
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-Path "$pfad\*.log"` | Searches all log files in the previously queried path |
| `-Pattern` | The search term, here the sender address |
| `-SimpleMatch` | Treats the pattern as text rather than a regular expression; the dot in the address therefore needs no escaping |
| `-First 5` | Limits output to the first five hits |

</details>

With the discovered `session-id`, retrieve the complete session:

```powershell
Select-String -Path "$pfad\*.log" -Pattern "08DEF44EC454A414" -SimpleMatch |
    ForEach-Object { $_.Line } | Select-Object -First 40
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-Pattern` | The session ID from the first hit |
| `-SimpleMatch` | Literal search without regex evaluation |
| `-First 40` | Limits output to the first 40 lines of the session |

</details>

If you only want to know which connectors see traffic at all, count connection establishments. For large files, this is orders of magnitude faster than parsing every line:

```powershell
Select-String -Path "$pfad\*.log" -Pattern ',\+,' |
    ForEach-Object { ($_.Line -split ',')[1] } |
    Group-Object | Sort-Object Count -Descending |
    Format-Table Count, Name -AutoSize
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-Pattern ',\+,'` | Regular expression for event `+` (connection establishment) between two CSV commas; the plus sign is escaped |
| `ForEach-Object { … -split ',' }` | Splits the matching line at commas and accesses the second column, `connector-id` |
| `Group-Object` | Counts connection establishments per connector |
| `Sort-Object Count -Descending` | Most-used connector at the top |

</details>

```text
Count Name
----- ----
51479 SRV-MAIL01\Default Frontend SRV-MAIL01
50756 SRV-MAIL01\smtp-auth SRV-MAIL01
19405 SRV-MAIL01\smtp-intern SRV-MAIL01
15789 SRV-MAIL01\smtp-noauth SRV-MAIL01
```

This distribution answers a question that message tracking cannot answer: Which routes do your applications actually use? Before changing a connector, this is the most important number of all.

### When nothing was logged

If every line is missing for the time in question, there are three common reasons: Logging was disabled on the relevant connector, the retention limit has already removed the file, or you are looking in the wrong path—namely the `Hub` directory instead of the `FrontEnd` directory. Check in that order.

## Step 4: Check permissions

If a submission is rejected or, conversely, more is permitted than expected, investigate the connector's permissions. There is a technical peculiarity here: `Get-ADPermission` requires the **DistinguishedName**. If you pass the familiar identity in the form `Server\Connectorname`, the call fails in a remote session with the misleading message that the object cannot be found.

```powershell
$dn = (Get-ReceiveConnector "SRV-MAIL01\Default Frontend SRV-MAIL01").DistinguishedName
Get-ADPermission -Identity $dn -User "NT AUTHORITY\ANONYMOUS LOGON" |
    Where-Object { $_.ExtendedRights -like "*ms-Exch-SMTP*" } |
    Format-Table User, @{n='Rights'; e={$_.ExtendedRights}} -AutoSize
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-Identity $dn` | The object to check as DistinguishedName; the `Server\Connectorname` form fails in remote sessions |
| `-User` | Limits output to the permissions of this security principal, here anonymous access |
| `Where-Object` | Filters for SMTP-relevant Extended Rights |

</details>

```text
User                         Rights
----                         ------
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Submit
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Accept-Any-Sender
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Accept-Authoritative-Domain-Sender
```

The evaluation is simpler than it looks when you distinguish four rights:

| Right | Meaning |
|---|---|
| `ms-Exch-SMTP-Submit` | May submit mail at all |
| `ms-Exch-SMTP-Accept-Any-Sender` | May use arbitrary sender addresses |
| `ms-Exch-SMTP-Accept-Authoritative-Domain-Sender` | May present itself as its own domain |
| `ms-Exch-SMTP-Accept-Any-Recipient` | **May relay to external domains** |

The first three are the standard set required for anonymous submission and receiving internet mail. Only the fourth right turns an inbound connector into a relay. On a connector accepting the entire address space, it is an open relay. On a connector with strict IP restrictions, however, it is the usual and intended way for application servers to send externally.

Do not confuse `Accept-Any-Sender` with `Accept-Any-Recipient`. The first is harmless and necessary; the second is the security-relevant setting.

## Step 5: Verification test through your own submission

If the analysis remains ambiguous, submit a message yourself. This gives you complete control over sender, recipient and submission point:

```powershell
Send-MailMessage -SmtpServer 10.0.1.11 -Port 25 `
    -From 'test@example.com' `
    -To 'empfaenger@example.net' `
    -Subject 'Diagnose, bitte ignorieren' `
    -Body 'Testeinlieferung' `
    -Encoding UTF8
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-SmtpServer` | Submission target host, deliberately an IP address here to reach a specific endpoint |
| `-Port 25` | Target port; 25 for unauthenticated server-to-server submission |
| `-From` | Envelope and header sender of the test message |
| `-To` | Recipient address |
| `-Subject` | Subject line |
| `-Body` | Message text |
| `-Encoding UTF8` | Character encoding for subject and text; prevents issues with umlauts |

</details>

`Send-MailMessage` is officially deprecated, but remains the fastest tool for diagnostics and is available on every Windows Server. On success, it produces no output, which takes some getting used to.

If you test a TLS connection on port 587 and the remote side presents a certificate that does not match the name used—for example, because you address the IP address—the call fails with a certificate error. For testing, you can suspend validation in the session:

```powershell
[Net.ServicePointManager]::ServerCertificateValidationCallback = { $true }
```

This applies only to the current PowerShell session. Set it deliberately and never in scripts running in production.

If the test message arrives and you want to know what happened to it along the way, the [Mail Header Analyzer](https://rafaelpfister.ch/tools/header-analyzer) helps: It parses the headers, maps the path through the hops and shows the results of the authentication checks, entirely locally in the browser without the message leaving your device.

## Exchange Online: the same question, a different tool

Different rules apply in the tenant, and this is where familiar methods fail. Expect these differences:

| | Exchange OnPrem | Exchange Online |
|---|---|---|
| Query | `Get-MessageTrackingLog` | `Get-MessageTraceV2` |
| Granularity | Every transport event | One line per message and recipient |
| Connector visible | Yes (with limitations, see above) | **No** |
| Server-specific | Yes, query per server | Not applicable |
| SMTP log | Available | **Not available** |
| Retention | Your configuration | 90 days via the cmdlet, maximum 10 days per query |
| Delay | Almost immediate | A few minutes |

The three most important practical consequences: There is **no connector assignment**; you make do with `FromIP` and `ToIP`. There is **no SMTP log**, so the SMTP conversation cannot be reconstructed. And data appears **with a delay**; a message just sent does not appear immediately.

### The basic query

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-4) `
    -EndDate (Get-Date) `
    -RecipientAddress "empfaenger@example.com" `
    -ResultSize 1000 |
  Sort-Object Received |
  Format-Table Received, SenderAddress, RecipientAddress, Status, FromIP, Size -AutoSize
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-StartDate` | Lower time limit of the query, here the last four hours |
| `-EndDate` | Upper time limit; the cmdlet requires both limits |
| `-RecipientAddress` | Filters for messages to this recipient address |
| `-ResultSize 1000` | Maximum rows on this page; the limit is 5,000 |

</details>

```text
Received            SenderAddress          RecipientAddress          Status    FromIP
--------            -------------          ----------------          ------    ------
11.08.2026 08:27:16 emma@partner.example   empfaenger@example.com    Delivered 10.0.20.23
11.08.2026 09:05:24 dienst@example-test.com empfaenger@example.com   Failed    10.0.20.23
```

The most important values of `Status`: `Delivered`, `Failed`, `Pending`, `Quarantined`, `FilteredAsSpam` and `Expanded` for expanded distribution groups. `Pending` means delivery attempts are still in progress, not that something is broken.

### The details for a message

The status alone says nothing about the reason. For that, you need the detail view, which requires the message ID from the basic query:

```powershell
$n = Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-4) -EndDate (Get-Date) `
        -RecipientAddress "empfaenger@example.com" -ResultSize 100 |
     Where-Object { $_.Status -eq 'Failed' } | Select-Object -First 1

Get-MessageTraceDetailV2 -MessageTraceId $n.MessageTraceId `
    -RecipientAddress $n.RecipientAddress |
  Format-List Date, Event, Action, Detail
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-MessageTraceId` | Unique message identifier from the basic query; required |
| `-RecipientAddress` | Recipient whose processing steps are shown; also required because a message can have multiple recipients |

</details>

This displays the processing steps in the service, such as rule applications, filtering decisions and the reason for a rejection.

### Beyond ten days

`Get-MessageTraceV2` provides data from the last 90 days, but no more than 10 days per query; split longer periods across multiple queries. Alternatively, there is the historical search, which runs asynchronously and provides the result as CSV, covering up to 90 days:

```powershell
Start-HistoricalSearch -ReportTitle "Analyse Nachtlauf" `
    -StartDate (Get-Date).AddDays(-45) `
    -EndDate (Get-Date).AddDays(-30) `
    -ReportType MessageTrace `
    -SenderAddress "dienst@example-test.com" `
    -NotifyAddress "admin@example.com"

Get-HistoricalSearch | Format-Table JobId, ReportTitle, Status, SubmitDate -AutoSize
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-ReportTitle` | Freely selectable job name under which the result can later be found |
| `-StartDate`, `-EndDate` | Period examined, up to 90 days back |
| `-ReportType MessageTrace` | Type of report; `MessageTrace` provides the message overview as CSV |
| `-SenderAddress` | Filters for this sender address |
| `-NotifyAddress` | Recipient of the completion notification; must be an address from an Accepted Domain of the tenant |

</details>

Allow time for this: depending on the scope, such jobs can run for hours.

### Source of error 4: Missing hits do not prove there was no traffic

This is the most subtle source of error in the tenant. `Get-MessageTraceV2` returns results in pages, with a maximum of 5,000 rows per call. Under high volume, one page may cover only a few minutes even though you queried seven days. If you then filter locally, for example by a source IP, you are filtering a tiny sample.

You can recognize this by the warning indicating further results:

```text
WARNING: There are more results, use the following command to get more.
Get-MessageTraceV2 -StartDate "2026-08-11T07:25:19Z" -EndDate "2026-08-11T09:05:46Z"
  -StartingRecipientAddress "naechster@example.com" -ResultSize 5000
```

If it appears, your analysis is **incomplete**. If no hit is returned, the correct result is: not found in this sample. It is not: does not exist.

There are two clean ways out. Either reduce the time window until one page covers it completely, indicated by the absence of the warning. Or work through all pages using the continuation information in the warning. For the question of whether something **never** occurs, a configuration check is preferable anyway: If a system has no route to a destination, it cannot deliver there, regardless of any observation window.

A complete analysis of all submitting addresses is a subject in its own right, with its own pitfalls of interpretation. It is covered in [Who actually submits mail to your tenant? Aggregating submitting IP addresses](https://rafaelpfister.ch/blog/einliefernde-ip-adressen-aggregieren).

## A procedure that has proven effective

In summary, this sequence has proven the fastest. Search for the message across all servers and determine the last event. If it fails, immediately switch to `Format-List` and read the complete SMTP response instead of inferring from the event type. Then clarify the scope by grouping and counting. Only once the case is narrowly scoped should you reconstruct the submission path through connector configuration and SMTP logs. Finally, if necessary, verify with your own submission.

The most common time-wasters are always the same: Reading a truncated table instead of the complete error message, mistaking shadow copies for processing steps, trusting the `ConnectorId` in tracking, and considering an empty sample proof. Those who know these four usually reach the right level within a few minutes.

## Change log

- **25 October 2026:** Corrected: Get-MessageTraceV2 returns data from the last 90 days, but no more than 10 days per query. Updated source links to the new Microsoft Learn addresses.

## Kilder

1.  [Message tracking in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): Field description and complete list of event types in message tracking.

2.  [Protocol logging in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/protocol-logging): Locations, format and retention of SMTP logs, including Front End Transport.

3.  [Shadow redundancy in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/shadow-redundancy): Explains events related to shadow copies and their disposal.

4.  [Mail routing in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing): Interaction between Front End Transport and Transport Service, the basis of proxy behavior.

5.  [Receive connectors in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/receive-connectors): Bindings, permission groups and authentication mechanisms.

6.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2): Successor to Get-MessageTrace, including paging logic, field list and query range (90 days, maximum 10 days per query).

7.  [Start-HistoricalSearch](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/start-historicalsearch): Asynchronous message tracing for up to 90 days.
