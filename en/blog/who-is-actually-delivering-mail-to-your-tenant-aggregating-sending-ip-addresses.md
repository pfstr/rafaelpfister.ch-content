---
title: "Who Is Actually Delivering Mail to Your Tenant? Aggregating Sending IP Addresses"
navTitle: "Sending IPs"
description: "A single analysis shows which systems are actually delivering mail to your tenant: forgotten connectors, applications sending directly, and service providers nobody documented. Including the pitfalls of paging logic and interpretation."
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
url: https://rafaelpfister.ch/en/blog/who-is-actually-delivering-mail-to-your-tenant-aggregating-sending-ip-addresses
translationSourceHash: 9dc48329a06945f705380eb3db428efb548f0c36a1fe3c4f2fb7de1185fee879
translationModel: gpt-5.6-terra
translatedAt: 2026-08-12T05:11:56.986Z
translationReview: required
---

# Who Is Actually Delivering Mail to Your Tenant? Aggregating Sending IP Addresses

Hardly any mail environment still has a complete picture of who delivers mail to it. Over the years, connectors from migrations accumulate, along with applications that send directly, service providers whose contracts expired long ago, and test setups that were never dismantled. As long as mail keeps flowing, nobody notices.

A single analysis provides clarity: grouping all incoming messages by their source IP address. It takes two minutes to create, and the results are regularly surprising. This article shows the query, explains how to get it **complete**, and, above all, how to read the figures correctly. Because interpretation is the harder part.

## Why it is worth doing

The list answers four questions that would otherwise be tedious to clarify individually. Which systems send to your tenant at all? Does everything go through the paths you documented, or is there a second entry point? Is a connector you want to retire still in use? And: Does an application send directly to the service, bypassing your gateway and therefore your filtering?

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

Before drawing conclusions, two things must be correct: the list must be complete, and you must know what the entries mean.

## Pitfall 1: The list is almost always incomplete

`Get-MessageTraceV2` returns results in pages, with a maximum of 5,000 rows per call. With high volume, one page covers only a fraction of your time window. You then group an excerpt and mistake the result for the whole picture.

You can recognize this by the following warning:

```text
WARNING: There are more results, use the following command to get more.
Get-MessageTraceV2 -StartDate "2026-08-11T07:25:19Z" -EndDate "2026-08-11T09:05:46Z"
  -StartingRecipientAddress "naechster@example.com" -ResultSize 5000
```

**If this warning appears, your analysis is worthless.** In particular, a missing entry must not be interpreted as absence. An address with three messages per day will not show up in an excerpt anyway.

There are two ways out. The simple one: reduce the time window until the warning no longer appears. At 5,000 messages per hour, that means 55 minutes rather than seven days. For the question “which systems send at all,” a complete short window is usually entirely sufficient.

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

For 24 hours in a medium-sized environment, expect a runtime of several minutes. For a one-time inventory, that is time well spent.

## Pitfall 2: The numbers do not mean what they appear to mean

The results list contains four fundamentally different types of entries, and anyone who throws them all into one bucket will draw incorrect conclusions.

**`255.255.255.255` does not represent a system.** This value appears when there was no incoming SMTP connection from outside for the message. This applies to messages generated within the service itself: journal reports, non-delivery reports, out-of-office replies, and messages between mailboxes in the same tenant. In nearly every environment, this is the largest item, and it is entirely unremarkable. Do not be alarmed.

**Private RFC 1918 addresses** originate from your own network. In hybrid environments, you will see the on-premises transport servers here, because their internal address is retained when handing off to the service. These are the large numbers in the list and, in most cases, the expected main path.

**Addresses of security and filtering services** can be identified by their operator, not by their numeric value. Cloud proxies, upstream mail gateways, and web security services appear with many adjacent addresses and medium volumes. They usually belong there, but they should be listed in the operations manual.

**Public individual addresses with small numbers** are the interesting ones. This is exactly where forgotten applications, former service providers, and systems nobody remembers anymore are hiding.

## Resolution: turning addresses into names

For anything you cannot immediately identify, reverse DNS lookup helps. It is not always configured and not always honest, but in most cases it provides the crucial clue:

```powershell
$unbekannt = '198.51.100.77','203.0.113.9'

$unbekannt | ForEach-Object {
    $name = try { (Resolve-DnsName $_ -Type PTR -ErrorAction Stop).NameHost } catch { '(kein PTR)' }
    [pscustomobject]@{ IP = $_; Name = $name }
} | Format-Table -AutoSize
```

```text
IP            Name
--            ----
198.51.100.77 mail-out-03.newsletter-provider.example
203.0.113.9   (kein PTR)
```

A missing PTR record is not proof of anything malicious, but it is a good reason to take a closer look. For such addresses, examine the associated messages:

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-2) -EndDate (Get-Date) -ResultSize 5000 |
  Where-Object { $_.FromIP -eq '203.0.113.9' } |
  Format-Table Received, SenderAddress, RecipientAddress, Subject, Status -AutoSize
```

The sender and subject will usually tell you immediately which application is behind it.

## The comparison: which address belongs to which connector?

This is where the real insight comes in. Compare your results list with the configured connectors:

```powershell
Get-InboundConnector |
    Format-List Name, Enabled, ConnectorType, ConnectorSource,
        @{n='SenderIPAddresses'; e={$_.SenderIPAddresses -join ', '}},
        @{n='SenderDomains';     e={$_.SenderDomains -join ', '}},
        RequireTls, TlsSenderCertificateName,
        RestrictDomainsToCertificate, CloudServicesMailEnabled, TreatMessagesAsInternal
```

Three situations are revealing.

**An address delivers mail but is not listed in any connector.** The mail is then coming in as ordinary internet mail. That is permitted, but it means this application receives no special handling and its messages are subject to full filtering. If someone claims a connector was configured for this system, that is clearly no longer true.

**A connector lists addresses from which nothing arrives.** This is a candidate for retirement. Before deleting it, check whether these are seasonal or infrequently used systems, and disable it first rather than removing it immediately.

**A connector sets `TreatMessagesAsInternal` or `CloudServicesMailEnabled` to true.** This warrants close scrutiny. Both settings cause messages arriving through this path to be treated as internal to the organization. If mail from the internet arrives through it, it bypasses checks intended for external messages, including protection against spoofed senders from your own domain. This is correct for a dedicated hybrid connector; for a connector through which arbitrary systems deliver mail, it is a finding.

## What you typically find

From practice, without claiming to be exhaustive: a test connector from a migration that has been active for years. A line-of-business application that sends directly to the service even though everyone believes it goes through the gateway. A newsletter provider whose contract has expired but is still allowed to deliver. And regularly, a connector with overly broad conditions that someone once created to solve an urgent problem.

None of these findings is dramatic on its own. Together, they paint the picture of an environment that nobody fully understands anymore, and that is the actual risk.

## Limitations of the method

There are three limitations you should know.

Message tracing through the cmdlet only goes back about ten days. For longer periods, you need historical search, which runs asynchronously and covers up to 90 days. Otherwise, you will miss infrequent systems that send monthly.

`FromIP` does not mean the same thing everywhere. For mail from the internet, it is the address of the delivering server. For hybrid mail, it is the address of your on-premises transport server, not that of the original sender. The analysis therefore shows you the **last hop before the service**, not the origin.

And the assignment to a connector is not directly visible in the tenant. You infer it from the address, certificate, and sender domain. For a reliable statement about the use of an individual connector, the connector report in the Exchange Admin Center under Reports and Mail flow is the better source because it aggregates server-side over longer periods.

## As a recurring check

This analysis is well suited as a quarterly routine. Save the result and compare it during the next run. New addresses in the list are either documented changes or something you want to know about.

If you are already reviewing your domains’ mail configuration: the [Mail DNS Check](https://rafaelpfister.ch/tools/mail-check) shows SPF, DKIM, DMARC, and the other mail standards for any domain directly in the browser, including secondary and marketing domains that experience shows are often forgotten during such inventories. And for the queries themselves, the [Command Generator](https://rafaelpfister.ch/tools/command-builder) provides ready-made building blocks for PowerShell and Unix shells.

How to investigate individual suspicious messages further is explained in [Analyzing Exchange mail flow: Message Tracking, SMTP logs, and Receive Connectors](https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren).

## Sources

1.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2): field list including FromIP and ToIP, as well as paging logic with StartingRecipientAddress.

2.  [Start-HistoricalSearch](https://learn.microsoft.com/en-us/powershell/module/exchange/start-historicalsearch): asynchronous message tracing for periods up to 90 days in the past.

3.  [Get-InboundConnector](https://learn.microsoft.com/en-us/powershell/module/exchange/get-inboundconnector): parameters of inbound connectors, including SenderIPAddresses and TreatMessagesAsInternal.

4.  [Configure mail flow using connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow): how connector types work together and when each applies.

5.  [RFC 1918: Address Allocation for Private Internets](https://www.rfc-editor.org/rfc/rfc1918): defines the private address ranges that you must distinguish from public addresses in the analysis.
