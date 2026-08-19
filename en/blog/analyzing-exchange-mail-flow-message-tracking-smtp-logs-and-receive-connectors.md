---
title: "Analyzing Exchange mail flow: message tracking, SMTP logs, and receive connectors"
navTitle: "Analyze mail flow"
description: "How to systematically determine where a message ended up in Exchange OnPrem, Hybrid, and Exchange Online: queries with sample output, how to read the SMTP log correctly, and the pitfalls that regularly lead you down the wrong path."
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
slug: "analyzing-exchange-mail-flow-message-tracking-smtp-logs-and-receive-connectors"
translationId: "article-ad93c41ab6cd20e6"
draft: false
translationOf: exchange-message-tracking-und-receive-connectoren-analysieren
url: https://rafaelpfister.ch/en/blog/analyzing-exchange-mail-flow-message-tracking-smtp-logs-and-receive-connectors
translationSourceHash: 646cb713e4dd97300a2cd068ee8f04953f2e80a99aec63ed11eddc46e1981f13
translationModel: gpt-5.6-terra
translatedAt: 2026-08-12T05:14:41.085Z
translationReview: required
---

# Analyzing Exchange mail flow: message tracking, SMTP logs, and receive connectors

The most common question in mail operations is: A message did not arrive—where did it go? Message tracking answers that reliably, but only if you know what it **does not** tell you. This article describes the process in the order that has proven effective, shows typical output for each query, and identifies the pitfalls that regularly cost hours because they suggest plausible but incorrect conclusions.

All examples use generic names: `SRV-MAIL01` and `SRV-MAIL02` as transport servers, `example.com` as the domain. If you want to assemble the commands for your environment rather than type them out: the [Command Generator](https://rafaelpfister.ch/tools/command-builder) contains common message-tracking and capture commands for PowerShell and Unix shells side by side, entirely locally in the browser.

## The principle: locate first, then explain

The instinct is to immediately look for the cause. It is more efficient to first determine how far the message got at all. This drastically narrows down the search area in one step, because you then know whether to look in your own system, at the upstream gateway, or at the destination.

The order is therefore: find the message, read the last event, read the reason for the error, determine whether it is an isolated case or a pattern, and only then reconstruct the submission path.

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

If the query finds nothing, check whether the recipient was expanded through a distribution group. In that case, it is better to search by sender:

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

## Step 2: Read the last event

The entire diagnosis depends on the **last** `EventId` for the message. It tells you which search area to investigate next.

| Last EventId | Meaning | Next step |
|---|---|---|
| `RECEIVE`, then nothing | Message is stuck | Check queues |
| `SEND` or `SENDEXTERNAL` | Successfully handed off | Continue investigating at the next hop |
| `FAIL` | Permanently failed | Read the reason in `RecipientStatus` |
| `DEFER` | Retry is in progress | Check the queue and destination system |
| `DROP` or `POISONMESSAGE` | Discarded | Check transport rule or agent |
| `DELIVER` | Delivered to a local mailbox | Check mailbox rules |
| `RESOLVE` | Recipient was rewritten | Read the destination address in the entry |

`RESOLVE` is the most informative intermediate step in hybrid environments because it shows the rewrite to the cloud routing address:

```text
EventId : RESOLVE
Source  : ROUTING
Sender  : absender@example.com
To      : BENUTZER@example.mail.onmicrosoft.com
```

If the expected `onmicrosoft.com` address appears there, the recipient object is configured correctly and you can close the issue. If the original address still appears there, the target address is missing from the local object and Exchange is attempting local delivery.

If the message is stuck, the queue usually shows the reason in plain text:

```powershell
Get-Queue -Server SRV-MAIL01 |
    Where-Object { $_.MessageCount -gt 0 } |
    Format-Table Identity, DeliveryType, Status, MessageCount, NextHopDomain, LastError `
        -AutoSize -Wrap
```

## Pitfall 1: Tracking is server-specific, and many entries are shadow copies

If the output shows pairs of `HARECEIVE` and `HADISCARD`, often with the additional `ExplicitlyDiscarded`, that server did **not process** the message. It merely held a shadow copy as part of Shadow Redundancy while another server handled the actual delivery. As soon as the primary server reports success, the partner discards its copy.

This is what it looks like when you queried only the wrong server:

```text
Timestamp           EventId    SourceContext
---------           -------    -------------
11.08.2026 09:47:13 HARECEIVE  1a2aae6b-f3a3-4e04-9233-7de460b92223
11.08.2026 09:48:07 HADISCARD  ExplicitlyDiscarded;1a2aae6b-f3a3-4e04-9233-7de460b92223
```

Two lines, no error, no delivery. Anyone concluding from this that the message disappeared is looking in the wrong place. The actual processing is in the tracking logs of the partner server.

In practical terms, this means two things. First, such lines are not an indication of a problem, but normal operation. Second, you must query all transport servers.

## Pitfall 2: `Format-Table` removes precisely the crucial columns

The reason for the error is in `RecipientStatus`, and that field is long. In a table, it either disappears entirely or gets truncated. This is exactly what causes people to see `FAIL` but not the reason, and to start guessing instead.

As soon as you find a failure case, switch to `Format-List` and expand the collection fields:

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

This is what the difference looks like. First, the table view, which explains nothing:

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

The diagnosis is now clear without needing a single assumption: The remote system objects to the sender. `LED` contains the full SMTP response, `FQDN` and `IP` identify the responding system, and `LRT` identifies the time of the last attempt.

## Step 3: Isolated case or pattern?

Before digging into an individual case, clarify the scope. This single query determines whether you are dealing with a side note or an incident:

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

Replace `5.1.8` with the status code you are investigating. The output answers the question in one line:

```text
Count Name
----- ----
    9 example-test.com
```

A single sender domain means a narrowly limited problem, not an incident, and you can continue investigating calmly. If twenty different domains appeared there, you would have an ongoing outage, and everything else would have to wait. Making this distinction early saves the most time in practice.

## Pitfall 3: The `ConnectorId` does not reveal the actual receive connector

This is the most costly pitfall because the output looks trustworthy. Mail submitted by a client or third-party system on port 25 first reaches the **Front End Transport**. It forwards the message to the **Transport Service** on port 2525. Message tracking is written only there; the Front End Transport does not write its own tracking logs.

You can see the consequence in this line:

```text
EventId        : RECEIVE
ConnectorId    : SRV-MAIL01\Default SRV-MAIL01
ClientIp       : 10.0.1.11
ClientHostname : srv-mail01.intern.example.com
```

The `ConnectorId` identifies the internal connector on port 2525, and the `ClientIp` is the address of the **proxying server**, not that of the original submitting system. Tracking simply does not show which of the configured connectors on port 25 a system actually reached. Anyone trusting this information looks for the issue in a connector that was not involved at all.

There are two ways to get the answer. The first is reconstruction through the configuration:

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object { Get-ReceiveConnector -Server $_ } |
    Format-List Identity, Enabled,
        @{n='Bindings';       e={$_.Bindings -join ','}},
        @{n='RemoteIPRanges'; e={$_.RemoteIPRanges -join ','}},
        PermissionGroups, AuthMechanism
```

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

Determine the source IP of the submitting system and find the connector whose `RemoteIPRanges` includes it. If it does not fall into any of the restricted connectors, the default Front End connector remains, which usually accepts the entire address space. Use `Format-List` here as well, because `RemoteIPRanges` and `PermissionGroups` are regularly truncated in tables.

The second way is the SMTP log, and it deserves its own section.

## The SMTP log: the only place with the full truth

The Front End Transport log records the complete SMTP session: which connector was addressed, which IP connected, and what the client and server said to each other. It is the only source that resolves the pitfall above.

### Enable logging

By default, logging is **disabled** on most connectors. Enable it per connector:

```powershell
Set-ReceiveConnector -Identity "SRV-MAIL01\Default Frontend SRV-MAIL01" `
    -ProtocolLoggingLevel Verbose
```

For outbound connections, use `Set-SendConnector` accordingly. Remember to set the value back to `None` after the analysis, because verbose logging consumes disk space and writes significant amounts of data under heavy load.

### Where the files are located

Exchange separates logs by service and direction. There is no need to hard-code paths; query them:

```powershell
Get-FrontendTransportService SRV-MAIL01 |
    Format-List ReceiveProtocolLogPath, SendProtocolLogPath,
        ReceiveProtocolLogMaxAge, ReceiveProtocolLogMaxDirectorySize

Get-TransportService SRV-MAIL01 |
    Format-List ReceiveProtocolLogPath, SendProtocolLogPath
```

They are typically located below the installation path in `TransportRoles\Logs\FrontEnd\ProtocolLog\SmtpReceive` for Front End Transport and in `TransportRoles\Logs\Hub\ProtocolLog\SmtpReceive` for the Transport Service. **This is the key point:** client connections on port 25 are found exclusively in the `FrontEnd` path; the `Hub` path contains only internal forwarding traffic on 2525.

Note the retention settings. `ReceiveProtocolLogMaxAge` is often set to 30 days, while `ReceiveProtocolLogMaxDirectorySize` additionally limits disk usage. Under high load, the size limit takes effect significantly earlier than the age limit, and your logs may then be only a few days old.

### Understanding the format

The files are CSV files with header lines beginning with `#`. The most important columns are `date-time`, `connector-id`, `session-id`, `sequence-number`, `local-endpoint`, `remote-endpoint`, `event`, and `data`.

The key column is `event`, a single character:

| Character | Meaning |
|---|---|
| `+` | Connection established |
| `-` | Connection closed |
| `>` | Server sends to client |
| `<` | Client sends to server |
| `*` | Server information, no SMTP traffic |

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

Everything missing from message tracking is shown here: the **actual** connector (`smtp-noauth`), the **actual** source IP (`10.0.20.22`), and the name the system uses to identify itself in `EHLO`.

### Searching specifically

For individual cases, a text filter is much faster than object-based analysis. Search for the sender address or the `EHLO` name and have it return the session ID:

```powershell
$pfad = (Get-FrontendTransportService SRV-MAIL01).ReceiveProtocolLogPath
Select-String -Path "$pfad\*.log" -Pattern "dienst@example-test.com" -SimpleMatch |
    Select-Object -First 5 Line
```

Use the identified `session-id` to retrieve the complete session:

```powershell
Select-String -Path "$pfad\*.log" -Pattern "08DEF44EC454A414" -SimpleMatch |
    ForEach-Object { $_.Line } | Select-Object -First 40
```

If you only want to know which connectors see traffic at all, count connection establishments. For large files, this is orders of magnitude faster than parsing every line:

```powershell
Select-String -Path "$pfad\*.log" -Pattern ',\+,' |
    ForEach-Object { ($_.Line -split ',')[1] } |
    Group-Object | Sort-Object Count -Descending |
    Format-Table Count, Name -AutoSize
```

```text
Count Name
----- ----
51479 SRV-MAIL01\Default Frontend SRV-MAIL01
50756 SRV-MAIL01\smtp-auth SRV-MAIL01
19405 SRV-MAIL01\smtp-intern SRV-MAIL01
15789 SRV-MAIL01\smtp-noauth SRV-MAIL01
```

This distribution answers a question that message tracking cannot answer: Which paths do your applications actually use? Before changing a connector, this is the most important number of all.

### When nothing was logged

If there is no line at the time in question, there are three usual reasons: logging was disabled on the connector in question, the retention limit has already removed the file, or you are looking in the wrong path—in the `Hub` directory instead of the `FrontEnd` directory. Check in that order.

## Step 4: Check permissions

If a submission is rejected, or conversely more is allowed than expected, the next step is the connector permissions. There is a technical pitfall here: `Get-ADPermission` requires the **DistinguishedName**. If you pass the familiar identity in the form `Server\Connectorname`, the call fails in a remote session with the misleading message that the object cannot be found.

```powershell
$dn = (Get-ReceiveConnector "SRV-MAIL01\Default Frontend SRV-MAIL01").DistinguishedName
Get-ADPermission -Identity $dn -User "NT AUTHORITY\ANONYMOUS LOGON" |
    Where-Object { $_.ExtendedRights -like "*ms-Exch-SMTP*" } |
    Format-Table User, @{n='Rights'; e={$_.ExtendedRights}} -AutoSize
```

```text
User                         Rights
----                         ------
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Submit
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Accept-Any-Sender
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Accept-Authoritative-Domain-Sender
```

The evaluation is simpler than it looks if you distinguish four rights:

| Right | Meaning |
|---|---|
| `ms-Exch-SMTP-Submit` | May submit at all |
| `ms-Exch-SMTP-Accept-Any-Sender` | May use arbitrary sender addresses |
| `ms-Exch-SMTP-Accept-Authoritative-Domain-Sender` | May present itself as its own domain |
| `ms-Exch-SMTP-Accept-Any-Recipient` | **May relay to external domains** |

The first three are the standard set required for anonymous submission and receiving Internet mail. Only the fourth right turns an inbound connector into a relay. On a connector that accepts the entire address space, it is an open relay. On a connector with a tight IP restriction, however, it is the usual and intended way for application servers to send externally.

Do not confuse `Accept-Any-Sender` with `Accept-Any-Recipient`. The first is harmless and necessary; the second is the security-relevant setting.

## Step 5: Verify with your own submission

If the analysis remains ambiguous, submit a message yourself. This gives you full control over the sender, recipient, and submission point:

```powershell
Send-MailMessage -SmtpServer 10.0.1.11 -Port 25 `
    -From 'test@example.com' `
    -To 'empfaenger@example.net' `
    -Subject 'Diagnose, bitte ignorieren' `
    -Body 'Testeinlieferung' `
    -Encoding UTF8
```

`Send-MailMessage` is officially deprecated, but for diagnostic purposes it remains the fastest tool and is available on every Windows server. On success, it produces no output, which takes some getting used to.

If you test a TLS connection on port 587 and the remote system presents a certificate that does not match the name used—for example, because you address the IP address—the call fails with a certificate error. For testing, you can disable validation for the session:

```powershell
[Net.ServicePointManager]::ServerCertificateValidationCallback = { $true }
```

This applies only to the current PowerShell session. Set it deliberately, and never in scripts that run in production.

If the test message arrives and you want to know what happened to it along the way, the [Mail Header Analyzer](https://rafaelpfister.ch/tools/header-analyzer) helps: it breaks down the headers, maps the path across hops, and shows the results of authentication checks, entirely locally in the browser without the message leaving your device.

## Exchange Online: the same question, a different tool

Different rules apply in the tenant, and this is where familiar approaches fail. Expect these differences:

| | Exchange OnPrem | Exchange Online |
|---|---|---|
| Query | `Get-MessageTrackingLog` | `Get-MessageTraceV2` |
| Granularity | Every transport event | One line per message and recipient |
| Connector visible | Yes (with limitations; see above) | **No** |
| Server-specific | Yes, query each server | Not applicable |
| SMTP log | Available | **Not available** |
| Retention | Your configuration | About 10 days through the cmdlet |
| Delay | Almost immediate | A few minutes |

The three most important practical consequences: There is **no connector mapping**; instead, you use `FromIP` and `ToIP`. There is **no SMTP log**; the SMTP conversation cannot be reconstructed. And the data appears **with a delay**; a message just sent does not appear immediately.

### The basic query

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-4) `
    -EndDate (Get-Date) `
    -RecipientAddress "empfaenger@example.com" `
    -ResultSize 1000 |
  Sort-Object Received |
  Format-Table Received, SenderAddress, RecipientAddress, Status, FromIP, Size -AutoSize
```

```text
Received            SenderAddress          RecipientAddress          Status    FromIP
--------            -------------          ----------------          ------    ------
11.08.2026 08:27:16 emma@partner.example   empfaenger@example.com    Delivered 10.0.20.23
11.08.2026 09:05:24 dienst@example-test.com empfaenger@example.com   Failed    10.0.20.23
```

The most important values of `Status` are `Delivered`, `Failed`, `Pending`, `Quarantined`, `FilteredAsSpam`, and `Expanded` for expanded distribution groups. `Pending` means that delivery attempts are still in progress, not that something is broken.

### Message details

The status alone says nothing about the reason. For that, you need the detail view, which requires the message identifier from the basic query:

```powershell
$n = Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-4) -EndDate (Get-Date) `
        -RecipientAddress "empfaenger@example.com" -ResultSize 100 |
     Where-Object { $_.Status -eq 'Failed' } | Select-Object -First 1

Get-MessageTraceDetailV2 -MessageTraceId $n.MessageTraceId `
    -RecipientAddress $n.RecipientAddress |
  Format-List Date, Event, Action, Detail
```

It contains the processing steps in the service, such as rule applications, filtering decisions, and the reason for a rejection.

### Beyond ten days

The cmdlet goes back about ten days. For older periods, there is the asynchronous historical search, which provides the result as a CSV file and supports a range of up to 90 days:

```powershell
Start-HistoricalSearch -ReportTitle "Analyse Nachtlauf" `
    -StartDate (Get-Date).AddDays(-45) `
    -EndDate (Get-Date).AddDays(-30) `
    -ReportType MessageTrace `
    -SenderAddress "dienst@example-test.com" `
    -NotifyAddress "admin@example.com"

Get-HistoricalSearch | Format-Table JobId, ReportTitle, Status, SubmitDate -AutoSize
```

Allow time for this; depending on the scope, such jobs can run for hours.

### Pitfall 4: Missing results do not prove that there was no traffic

This is the most subtle pitfall in the tenant. `Get-MessageTraceV2` returns results in pages, with a maximum of 5,000 lines per call. Under high volume, one page may cover only a few minutes even though you queried seven days. If you then filter locally, for example by a source IP, you are filtering a tiny excerpt.

You can recognize this from the warning that indicates further results:

```text
WARNING: There are more results, use the following command to get more.
Get-MessageTraceV2 -StartDate "2026-08-11T07:25:19Z" -EndDate "2026-08-11T09:05:46Z"
  -StartingRecipientAddress "naechster@example.com" -ResultSize 5000
```

If it appears, your analysis is **incomplete**. If no result is returned, the correct conclusion is: not found in the excerpt. It is not: does not exist.

There are two clean ways out. Either narrow the time window until a page covers it completely, indicated by the absence of the warning. Or work through all pages using the continuation information in the warning. For the question of whether something **never** occurs, a configuration check is preferable anyway: If a system has no route to a destination, it cannot deliver there, regardless of any observation window.

A complete analysis of all submitting addresses is a topic in its own right, with its own interpretation pitfalls. It is covered in [Who is actually submitting mail to your tenant? Aggregating submitting IP addresses](https://rafaelpfister.ch/blog/einliefernde-ip-adressen-aggregieren).

## A process that has proven effective

In summary, this sequence has proven to be the fastest. Search for the message across all servers and determine the last event. In case of failure, immediately switch to `Format-List` and read the full SMTP response rather than inferring from the event type. Then clarify the scope by grouping and counting. Only when the case is narrowly limited should you reconstruct the submission path through connector configuration and SMTP logs. Finally, if necessary, verify with your own submission.

The most common time-wasters are always the same: reading a truncated table instead of the complete error message, treating shadow copies as processing steps, trusting the `ConnectorId` in tracking, and treating an empty sample as proof. Those who know these four usually reach the correct level within minutes.

## Sources

1.  [Message tracking in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): Field descriptions and the complete list of event types in message tracking.

2.  [Protocol logging in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/protocol-logging): Locations, format, and retention of SMTP logs, including Front End Transport.

3.  [Shadow redundancy in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/shadow-redundancy): Explains events related to shadow copies and their discard.

4.  [Mail routing in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing): How Front End Transport and the Transport Service work together, forming the basis of proxy behavior.

5.  [Receive connectors in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/receive-connectors): Bindings, permission groups, and authentication mechanisms.

6.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2): Successor to Get-MessageTrace, including paging logic and field list.

7.  [Start-HistoricalSearch](https://learn.microsoft.com/en-us/powershell/module/exchange/start-historicalsearch): Asynchronous message tracing for up to 90 days.
