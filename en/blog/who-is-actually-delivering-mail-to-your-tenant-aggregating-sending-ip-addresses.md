---
title: "Who Is Actually Delivering Mail to Your Tenant? Aggregating Sending IP Addresses"
navTitle: "Sending IPs"
description: "A single analysis shows which systems actually deliver mail to your tenant: forgotten connectors, applications that send directly, and vendors nobody documented, including the typical analysis errors involving paging logic and interpretation."
date: "2026-08-11"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "12 min read"
themen:
  - microsoft-365-exchange
  - smtp-mailflow
  - exchange-onprem-hybrid
hauptthema: "microsoft-365-exchange"
produkte:
  - "exchange-online"
  - "exchange-hybrid"
protokolle:
  - "smtp"
  - "powershell"
  - "haertung"
related:
  - exchange-message-tracking-und-receive-connectoren-analysieren
  - ghost-sender-exchange-online-nebeneingang
  - mailflow-fehlersuche-kontrollierte-experimente
slug: "who-is-actually-delivering-mail-to-your-tenant-aggregating-sending-ip-addresses"
translationId: "article-5879cc0eb17ed951"
draft: false
translationOf: einliefernde-ip-adressen-aggregieren
translationSourceHash: 291ed92e756d5fa011212a5cf9d3ce9061df96d73857f508c838183965ba25b4
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:37:44.648Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/en/blog/who-is-actually-delivering-mail-to-your-tenant-aggregating-sending-ip-addresses
---

# Who Is Actually Delivering Mail to Your Tenant? Aggregating Sending IP Addresses

Hardly any mail environment still has a complete picture of who delivers mail to it. Over the years, connectors from migrations accumulate, as do applications that send directly, vendors whose contracts expired long ago, and test setups that were never dismantled. As long as mail keeps flowing, nobody notices.

A single analysis provides clarity: grouping all incoming messages by source IP address. It takes two minutes, and the resulting list is regularly surprising. This article shows the query, explains how to make it **complete**, and above all, how to read the figures correctly. Because interpretation is the harder part.

## Why It Is Worth Doing

The list answers four questions that would otherwise be tedious to clarify individually. Which systems send mail to your tenant at all? Does everything go through the paths you have documented, or is there a second entry point? Is a connector you want to retire still in use? And: Does an application send directly to the service, bypassing your gateway and therefore your filtering?

The last question in particular is security-relevant. Anyone delivering mail directly bypasses not only filtering, but often also the logging you want to rely on in an incident.

## The Query

In the tenant, group message tracing by `FromIP`:

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-2) `
    -EndDate (Get-Date) `
    -ResultSize 5000 |
  Group-Object FromIP |
  Sort-Object Count -Descending |
  Format-Table Count, Name -AutoSize
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-StartDate (Get-Date).AddHours(-2)` | Start of the query window, two hours ago in this case |
| `-EndDate (Get-Date)` | End of the query window, the current time |
| `-ResultSize 5000` | Maximum number of rows per call; 5000 is also the maximum value |
| `Group-Object FromIP` | Groups messages by the delivering IP address |
| `Sort-Object Count -Descending` | Sorts groups in descending order by message count |
| `Format-Table Count, Name -AutoSize` | Two-column output (count, IP address) with automatic column width |

</details>

Typical output:

```text
Count Name
----- ----
 1771 255.255.255.255
 1649 10.0.20.23
  260 10.0.20.21
   49 2603:10a6:150:1f3::17
   46 165.225.94.87
   36 136.226.192.164
   35 147.161.246.105
   12 198.51.100.77
    3 203.0.113.9
```

Before drawing conclusions from it, two things must be correct: the list must be complete, and you must know what the entries mean.

## Source of Error 1: The List Is Almost Always Incomplete

`Get-MessageTraceV2` returns results in pages, with a maximum of 5000 rows per call. With high volumes, one page covers only a fraction of your time window. You then group a subset and mistake the result for the whole.

You can recognize this by the following warning:

```text
WARNING: There are more results, use the following command to get more.
Get-MessageTraceV2 -StartDate "2026-08-11T07:25:19Z" -EndDate "2026-08-11T09:05:46Z"
  -StartingRecipientAddress "naechster@example.com" -ResultSize 5000
```

**If this warning appears, your analysis is worthless.** In particular, a missing entry must not then be interpreted as absence. An address with three messages per day will not appear in a subset anyway.

There are two ways out. The simple one: reduce the time window until the warning no longer appears. With 5000 messages per hour, that means 55 minutes rather than seven days. For the question “which systems send mail at all,” a complete short window is usually entirely sufficient.

The thorough approach pages through all results and collects them:

```powershell
$start = (Get-Date).AddHours(-24)
$ende  = Get-Date
$alle  = @()
$naechster = $null

do {
    $seite = if ($naechster) {
        Get-MessageTraceV2 -StartDate $start -EndDate $ende `
            -StartingRecipientAddress $naechster -ResultSize 5000
    } else {
        Get-MessageTraceV2 -StartDate $start -EndDate $ende -ResultSize 5000
    }

    $alle += $seite
    if ($seite.Count -eq 5000) {
        $naechster = $seite[-1].RecipientAddress
        $ende      = $seite[-1].Received
    } else {
        $naechster = $null
    }
    Write-Host "Gesammelt: $($alle.Count)"
} while ($naechster)

$alle | Group-Object FromIP | Sort-Object Count -Descending |
    Format-Table Count, Name -AutoSize
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-StartDate` / `-EndDate` | Query window, the last 24 hours in this case |
| `-StartingRecipientAddress` | Continuation point for the paging logic: the recipient address of the last entry on the previous page; together with `-EndDate` = its receive time (`Received`), it starts the next page there |
| `-ResultSize 5000` | Page size; a full page signals that more results follow |
| `Group-Object FromIP` | Groups the complete data set by the delivering IP address |
| `Sort-Object Count -Descending` | Sorts groups in descending order by message count |
| `Format-Table Count, Name -AutoSize` | Outputs the count per address with automatic column width |

</details>

The loop retrieves additional pages as long as a page returns exactly 5000 rows, each time continuing with the recipient address and receive time of the last entry on the previous page, as specified by the documentation for `Get-MessageTraceV2`; only the complete data set is grouped.

For 24 hours in a medium-sized environment, expect a runtime of several minutes. For a one-time inventory, that is time well spent.

## Source of Error 2: The Numbers Do Not Mean What They Seem to Mean

The results list contains four fundamentally different types of entries, and anyone who lumps them together will draw the wrong conclusions.

**`255.255.255.255` does not represent a system.** This value appears when there was no incoming SMTP connection from outside for the message. It applies to messages generated within the service itself: journal reports, non-delivery reports, out-of-office replies, and messages between mailboxes in the same tenant. In almost every environment, this is the largest category, and it is completely unremarkable.

**Private addresses from RFC 1918** originate from your own network. In hybrid environments, you will see the on-premises transport servers here, because their internal address is retained when handing mail off to the service. These are the large figures in the list and, as a rule, the expected primary path.

**Addresses of security and filtering services** can be identified by their operator, not by the numerical value. Cloud proxies, upstream mail gateways, and web security services appear with many neighboring addresses and moderate figures. They usually belong there, but should be documented in the operations manual.

**Individual public addresses with low figures** are the interesting ones. This is exactly where forgotten applications, former vendors, and systems nobody remembers hide.

## Resolution: Turning Addresses into Names

For anything you cannot immediately identify, reverse resolution helps. It is not always configured and not always reliable, but in most cases it provides the decisive clue:

```powershell
$unbekannt = '198.51.100.77','203.0.113.9'

$unbekannt | ForEach-Object {
    $name = try { (Resolve-DnsName $_ -Type PTR -ErrorAction Stop).NameHost } catch { '(kein PTR)' }
    [pscustomobject]@{ IP = $_; Name = $name }
} | Format-Table -AutoSize
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Resolve-DnsName $_ -Type PTR` | Queries the reverse record (PTR) for each IP address |
| `-ErrorAction Stop` | Turns a missing record into a catchable error for the `try`/`catch` block |
| `[pscustomobject]@{ … }` | Creates an object per address with the IP and resolved name for table output |
| `Format-Table -AutoSize` | Output with automatic column width |

</details>

```text
IP            Name
--            ----
198.51.100.77 mail-out-03.newsletter-provider.example
203.0.113.9   (kein PTR)
```

A missing PTR alone is not an indication of a problem, but it is a good reason to look more closely. For such addresses, examine the associated messages:

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-2) -EndDate (Get-Date) -ResultSize 5000 |
  Where-Object { $_.FromIP -eq '203.0.113.9' } |
  Format-Table Received, SenderAddress, RecipientAddress, Subject, Status -AutoSize
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-StartDate` / `-EndDate` / `-ResultSize` | Query window and page size as in the main query |
| `Where-Object { $_.FromIP -eq '203.0.113.9' }` | Filters client-side for the source address in question |
| `Format-Table Received, SenderAddress, RecipientAddress, Subject, Status -AutoSize` | Shows receive time, sender, recipient, subject, and delivery status for each message |

</details>

The sender and subject will usually tell you immediately which application is behind it.

## Comparison: Which Address Belongs to Which Connector?

Compare your results list with the configured connectors:

```powershell
Get-InboundConnector |
    Format-List Name, Enabled, ConnectorType, ConnectorSource,
        @{n='SenderIPAddresses'; e={$_.SenderIPAddresses -join ', '}},
        @{n='SenderDomains';     e={$_.SenderDomains -join ', '}},
        RequireTls, TlsSenderCertificateName,
        RestrictDomainsToCertificate, CloudServicesMailEnabled, TreatMessagesAsInternal
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Get-InboundConnector` | Lists all inbound connectors in the tenant; intentionally without restrictive parameters here |
| `Format-List <Eigenschaften>` | Outputs the specified properties as a list, one per line |
| `@{n='…'; e={…}}` | Calculated property with name (`n`) and expression (`e`) |
| `-join ', '` | Turns the array of addresses or domains into a readable, comma-separated line |

</details>

Three scenarios are revealing.

**An address delivers mail but is not listed in any connector.** Then the mail arrives as ordinary Internet mail. That is permissible, but means this application receives no special treatment and its messages are subject to full filtering. If someone claims a connector is configured for this system, that is clearly no longer true.

**A connector lists addresses from which nothing arrives.** A candidate for retirement. Before deleting it, check whether these are seasonal or infrequently used systems, and disable it first rather than removing it immediately.

**A connector sets `TreatMessagesAsInternal` or `CloudServicesMailEnabled` to true.** This warrants close scrutiny. Both settings cause messages received through this path to be treated as internal to the organization. If mail from the Internet arrives through it, it bypasses checks intended for external messages, including protection against spoofed senders from your own domain. This is correct for a pure hybrid connector; for a connector through which arbitrary systems deliver mail, it is a finding.

## What You Typically Find

From practice, without claiming to be exhaustive: a test connector from a migration that has been active for years. A line-of-business application that sends directly to the service even though everyone believes it goes through the gateway. A newsletter vendor whose contract has expired but that can still deliver. And regularly, a connector with overly broad conditions that someone once created to solve an urgent problem.

None of these findings is dramatic on its own. Together, they paint the picture of an environment that nobody fully understands anymore, and that is the actual risk.

## Limitations of the Method

There are three limitations you should know.

`Get-MessageTraceV2` covers the last 90 days, but returns no more than ten days per query. For longer periods, query multiple time windows or use the historical search, which runs asynchronously and also covers up to 90 days. Otherwise, you may miss systems that send monthly.

`FromIP` does not mean the same thing everywhere. For mail from the Internet, it is the address of the delivering server. For hybrid mail, it is the address of your on-premises transport server, not that of the original sender. The analysis therefore shows you the **last hop before the service**, not the origin.

And the association with a connector is not directly visible in the tenant. You infer it from the address, certificate, and sender domain. For a reliable statement about usage of an individual connector, the connector report in the Exchange Admin Center under Reports and Mail flow is the better source because it aggregates server-side over longer periods.

## As a Recurring Check

This analysis is well suited as a quarterly routine. Save the result and compare it on the next run. New addresses in the list are either documented changes or something you want to know about.

If you are reviewing your domains’ mail configuration anyway: the [Mail DNS Check](https://rafaelpfister.ch/tools/mail-check) shows SPF, DKIM, DMARC, and the other mail standards for any domain directly in the browser, including secondary and marketing domains that experience shows are often forgotten during such inventories. And for the queries themselves, the [Command Generator](https://rafaelpfister.ch/tools/command-builder) provides ready-made building blocks for PowerShell and Unix shells.

How to investigate individual suspicious messages further is explained in [Analyzing Exchange Mail Flow: Message Tracking, SMTP Logs, and Receive Connectors](https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren).

## Change Log

- **October 25, 2026:** Corrected: The paging logic of Get-MessageTraceV2 continues with the recipient address and receive time (EndDate) of the last entry. Corrected: The cmdlet covers the last 90 days, but returns no more than ten days per query.

## Sources

1.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2): Field list including FromIP and ToIP, query period (90 days, no more than ten days per query), and continuation using StartingRecipientAddress and EndDate.

2.  [Start-HistoricalSearch](https://learn.microsoft.com/en-us/powershell/module/exchange/start-historicalsearch): Asynchronous message tracing across up to 90 days for older time periods.

3.  [Get-InboundConnector](https://learn.microsoft.com/en-us/powershell/module/exchange/get-inboundconnector): Parameters of inbound connectors, including SenderIPAddresses and TreatMessagesAsInternal.

4.  [Configure mail flow using connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow): Interaction of connector types and when each applies.

5.  [RFC 1918: Address Allocation for Private Internets](https://www.rfc-editor.org/rfc/rfc1918): Defines the private address ranges that you must distinguish from public addresses in the analysis.
