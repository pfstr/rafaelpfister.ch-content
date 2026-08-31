---
title: "Who is actually delivering mail into your tenant? Aggregating inbound IP addresses"
navTitle: "Inbound IPs"
description: "A single analysis shows which systems actually deliver mail into your tenant: forgotten connectors, applications sending directly, and service providers nobody documented, including the typical analysis errors involving pagination logic and interpretation."
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
slug: "hvem-leverer-egentlig-e-post-til-tenant-en-din-aggreger-innkommende-ip-adresser"
translationId: "article-5879cc0eb17ed951"
draft: false
translationOf: einliefernde-ip-adressen-aggregieren
translationSourceHash: 9209720819061360cb72bfa01ab6261e6af80e547a398c25f6802edfbe49bb6c
translationModel: gpt-5.6-terra
translatedAt: 2026-08-31T10:07:20.989Z
translationReview: automatic
url: https://rafaelpfister.ch/no/blog/hvem-leverer-egentlig-e-post-til-tenant-en-din-aggreger-innkommende-ip-adresser
---

# Who is actually delivering mail into your tenant? Aggregating inbound IP addresses

Hardly any mail environment still knows exactly who delivers mail into it. Over the years, connectors from migrations accumulate, as do applications that send directly, service providers whose contracts expired long ago, and test setups that were never dismantled. As long as mail keeps flowing, nobody notices.

A single analysis provides clarity: grouping all incoming messages by their source IP address. It takes two minutes, and the resulting list is regularly surprising. This article shows the query, explains how to make it **complete**, and above all how to read the figures correctly. Because interpretation is the harder part.

## Why it is worth doing

The list answers four questions that would otherwise be laborious to clarify individually. Which systems send mail into your tenant at all? Does everything go through the routes you have documented, or is there a second entry point? Is a connector you want to decommission still being used? And: Is an application sending directly to the service, bypassing your gateway and therefore your filtering?

The last question in particular is security-relevant. Anyone delivering mail directly bypasses not only filtering, but often also the logging you want to rely on in an incident.

## The query

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
| `-StartDate (Get-Date).AddHours(-2)` | Start of the query window, here two hours ago |
| `-EndDate (Get-Date)` | End of the query window, the current time |
| `-ResultSize 5000` | Maximum number of rows per call; 5000 is also the maximum value |
| `Group-Object FromIP` | Groups messages by the delivering IP address |
| `Sort-Object Count -Descending` | Sorts groups in descending order by message count |
| `Format-Table Count, Name -AutoSize` | Two-column output (count, IP address) with automatic column width |

</details>

A typical output:

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

Before drawing conclusions from it, two things must be right: the list must be complete, and you need to know what the entries mean.

## Source of error 1: The list is almost always incomplete

`Get-MessageTraceV2` returns results in pages, with a maximum of 5000 rows per call. With high volumes, one page covers only a fraction of your time window. You then group a subset and mistake the result for the whole.

You can recognise this by the following warning:

```text
WARNING: There are more results, use the following command to get more.
Get-MessageTraceV2 -StartDate "2026-08-11T07:25:19Z" -EndDate "2026-08-11T09:05:46Z"
  -StartingRecipientAddress "naechster@example.com" -ResultSize 5000
```

**If this warning appears, your analysis is worthless.** In particular, a missing entry must not be interpreted as absence. An address with three messages per day will not appear in a subset anyway.

There are two ways out. The simple one: reduce the time window until the warning no longer appears. At 5000 messages per hour, that means 55 minutes rather than seven days. For the question “which systems send mail at all”, a complete short window is usually entirely sufficient.

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
    $naechster = if ($seite.Count -eq 5000) { $seite[-1].RecipientAddress } else { $null }
    Write-Host "Gesammelt: $($alle.Count)"
} while ($naechster)

$alle | Group-Object FromIP | Sort-Object Count -Descending |
    Format-Table Count, Name -AutoSize
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-StartDate` / `-EndDate` | Query window, here the last 24 hours |
| `-StartingRecipientAddress` | Continuation point for pagination: the recipient address at which the next page starts |
| `-ResultSize 5000` | Page size; a full page signals that further results follow |
| `Group-Object FromIP` | Groups the complete dataset by the delivering IP address |
| `Sort-Object Count -Descending` | Sorts groups in descending order by message count |
| `Format-Table Count, Name -AutoSize` | Outputs the count per address with automatic column width |

</details>

The loop retrieves further pages for as long as a page returns exactly 5000 rows, each time continuing from the last recipient address of the previous page; only the complete dataset is grouped.

For 24 hours in a medium-sized environment, expect a runtime of several minutes. For a one-off inventory, that is time well invested.

## Source of error 2: The figures do not mean what they appear to mean

The result list contains four fundamentally different types of entries, and anyone who lumps them together will draw the wrong conclusions.

**`255.255.255.255` does not stand for a system.** This value appears when there was no incoming SMTP connection from outside for the message. It applies to messages generated within the service itself: journal reports, non-delivery reports, out-of-office replies, and messages between mailboxes in the same tenant. In almost every environment, this is the largest item, and it is completely unremarkable.

**Private RFC 1918 addresses** originate from your own network. In hybrid environments, you see the on-premises transport servers here, because their internal address is retained when they hand off to the service. These are the large figures in the list and, as a rule, the expected primary route.

**Addresses of security and filtering services** can be identified by their operator, not by the numerical value. Cloud proxies, upstream mail gateways and web security services appear with many neighbouring addresses and medium figures. They usually belong there, but should be documented in the operations manual.

**Individual public addresses with small figures** are the interesting ones. This is exactly where forgotten applications, former service providers and systems nobody remembers hide.

## Resolution: turning addresses into names

For anything you cannot immediately identify, reverse lookup helps. It is not always configured and not always reliable, but in most cases it provides the decisive clue:

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
| `-ErrorAction Stop` | Turns a missing entry into a catchable error for the `try`/`catch` block |
| `[pscustomobject]@{ … }` | Creates an object per address with IP and resolved name for table output |
| `Format-Table -AutoSize` | Output with automatic column width |

</details>

```text
IP            Name
--            ----
198.51.100.77 mail-out-03.newsletter-provider.example
203.0.113.9   (kein PTR)
```

A missing PTR is not in itself an indication of a problem, but it is a good reason to look more closely. For such addresses, inspect the associated messages:

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
| `Format-Table Received, SenderAddress, RecipientAddress, Subject, Status -AutoSize` | Shows receipt time, sender, recipient, subject and delivery status for each message |

</details>

Sender and subject will generally tell you immediately which application is behind it.

## Comparison: which address belongs to which connector?

Compare your result list with the configured connectors:

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
| `Get-InboundConnector` | Lists all inbound connectors in the tenant; deliberately without restrictive parameters here |
| `Format-List <Eigenschaften>` | Outputs the specified properties as a list, one per line |
| `@{n='…'; e={…}}` | Calculated property with name (`n`) and expression (`e`) |
| `-join ', '` | Turns the array of addresses or domains into a readable, comma-separated line |

</details>

Three scenarios are revealing.

**An address delivers mail but is not listed in any connector.** The mail then arrives as ordinary internet mail. That is permissible, but it means this application receives no special treatment and its messages are subject to full filtering. If someone claims a connector has been configured for this system, that is clearly no longer true.

**A connector lists addresses from which no mail arrives.** A candidate for decommissioning. Before deleting it, check whether these are seasonal or infrequently used systems, and disable it first rather than removing it immediately.

**A connector sets `TreatMessagesAsInternal` or `CloudServicesMailEnabled` to true.** This warrants close attention. Both settings cause messages arriving through this route to be treated as internal to the organisation. If mail from the internet arrives through it, it bypasses checks intended for external messages, including protection against spoofed senders from your own domain. This is correct for a pure hybrid connector; for a connector through which arbitrary systems deliver mail, it is a finding.

## What you typically find

From practice, without claiming completeness: a test connector from a migration that has been active for years. A line-of-business application sending directly to the service even though everyone believes it goes through the gateway. A newsletter service provider whose contract has expired but is still allowed to deliver mail. And regularly, a connector with overly broad conditions that someone once created to solve an urgent problem.

None of these findings is dramatic on its own. Together, they paint the picture of an environment nobody fully understands any more, and that is the real risk.

## Limitations of the method

There are three limitations you should know.

Message tracing via the cmdlet only goes back around ten days. For longer periods, you need historical search, which runs asynchronously and covers up to 90 days. Otherwise, you will miss systems that send monthly.

`FromIP` does not mean the same thing everywhere. For mail from the internet, it is the address of the delivering server. For hybrid mail, it is the address of your on-premises transport server, not that of the original sender. The analysis therefore shows you the **last hop before the service**, not the origin.

And the association with a connector is not directly visible in the tenant. You infer it from the address, certificate and sender domain. For a reliable statement about the usage of an individual connector, the connector report in the Exchange Admin Center under Reports and Mail flow is the better source, because it aggregates server-side over longer periods.

## As a recurring review

This analysis is well suited as a quarterly routine. Save the result and compare it during the next run. New addresses in the list are either documented changes or something you need to know about.

If you are reviewing your domains' mail configuration anyway: the [Mail DNS Check](https://rafaelpfister.ch/tools/mail-check) shows SPF, DKIM, DMARC and the other mail standards for any domain directly in the browser, including secondary and marketing domains that experience shows are often forgotten during such inventories. And for the queries themselves, the [Command Generator](https://rafaelpfister.ch/tools/command-builder) provides ready-made building blocks for PowerShell and Unix shells.

How to investigate individual suspicious messages further is explained in [Analysing Exchange mail flow: Message Tracking, SMTP logs and Receive Connectors](https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren).

## Kilder

1.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2): field list including FromIP and ToIP, as well as pagination logic with StartingRecipientAddress.

2.  [Start-HistoricalSearch](https://learn.microsoft.com/en-us/powershell/module/exchange/start-historicalsearch): asynchronous message tracing over up to 90 days for older periods.

3.  [Get-InboundConnector](https://learn.microsoft.com/en-us/powershell/module/exchange/get-inboundconnector): parameters for inbound connectors, including SenderIPAddresses and TreatMessagesAsInternal.

4.  [Configure mail flow using connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow): interaction of connector types and when each applies.

5.  [RFC 1918: Address Allocation for Private Internets](https://www.rfc-editor.org/rfc/rfc1918): defines the private address ranges that you need to distinguish from public addresses in the analysis.
