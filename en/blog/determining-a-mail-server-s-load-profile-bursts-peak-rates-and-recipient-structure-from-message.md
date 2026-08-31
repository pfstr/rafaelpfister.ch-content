---
title: "Determining a Mail Server’s Load Profile: Bursts, Peak Rates, and Recipient Structure from Message Tracking"
navTitle: "Determine load profile"
description: "How many emails per minute does your mail server really process, and how high are the peaks? How to use PowerShell and Exchange Message Tracking to determine the real load profile: rates per minute and hour, burst duration, recipient structure, message sizes, and common analysis mistakes."
date: "2026-08-25"
kategorie: "SMTP and mail flow"
timeToRead: "9 min read"
themen:
  - smtp-mailflow
  - exchange-onprem-hybrid
produkte:
  - "exchange-on-premises"
  - "uebergreifend"
protokolle:
  - "smtp"
  - "powershell"
  - "troubleshooting"
slug: "determining-a-mail-server-s-load-profile-bursts-peak-rates-and-recipient-structure-from-message"
translationId: "article-1ff17a188d73e289"
aiPrompt: |
  Du bist mein Mailflow-Assistent. Hilf mir Schritt für Schritt, das Lastprofil meines Mailservers zu ermitteln: 1. Die richtige Datenquelle wählen (Message Tracking, Gateway-Logs) und das passende Event pro Nachricht bestimmen. 2. Raten pro Minute, Stunde und Tag berechnen und Bursts mit Dauer und Peak charakterisieren. 3. Empfängerstruktur, Domain-Verteilung und Nachrichtengrössen auswerten. Weise mich auf Doppelzählungen, Export-Limits und Zeitzonen-Fallen hin.
translationOf: mailserver-lastprofil-ermitteln
translationSourceHash: 298fabdf5f8f248539ea8a119681be130cd76f5c8ebc35db5d0c61e1126251b0
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:27:38.783Z
translationReview: required
url: https://rafaelpfister.ch/en/blog/determining-a-mail-server-s-load-profile-bursts-peak-rates-and-recipient-structure-from-message
---

# Determining a Mail Server’s Load Profile: Bursts, Peak Rates, and Recipient Structure from Message Tracking

Whether you need to replace a gateway, size a server, or plan a maintenance window, sooner or later every mail administrator needs an answer to the question of how much their system actually processes. Gut feeling is regularly wrong here, because mail traffic is rarely consistent. A system that sees an average of 20 emails per minute over the course of a day may need to process 400 per minute for an hour during an invoice run. If you only know the average, you are sizing for the wrong problem.

A useful load profile consists of four metrics: the average rate (per minute, hour, day), bursts (how high is the peak, how long does it last, when does it occur), recipient structure (how many distinct recipients, which destination domains), and message sizes. All four are available in Message Tracking, and on Exchange they can be calculated with a few lines of PowerShell.

## The data source: Message Tracking

Exchange logs every message in the Message Tracking Log. Before analyzing it, check how far back the data goes; the default is 30 days, but a tight size limit can significantly reduce the actual retention period:

```powershell
Get-TransportService |
    Select-Object Name, MessageTrackingLogMaxAge,
        MessageTrackingLogMaxDirectorySize, MessageTrackingLogPath
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Get-TransportService` | Lists all Transport servers in the organization; without parameters, all servers |
| `Select-Object Name, MessageTrackingLog…` | Reduces the output to the specified properties: retention period, size limit of the log directory, and log path |

</details>

For a load profile, the period should cover at least one complete company batch cycle: monthly invoice runs, payroll, newsletters. One week is the minimum; one month is better.

## Collecting raw data: one event per message

The most important initial decision: Which event counts as “one email”? Message Tracking writes several entries per message (RECEIVE when accepted, SEND when passed to the next hop, DELIVER when delivered to a mailbox, plus AGENTINFO, HAREDIRECT, and others). Simply counting all rows overestimates the volume several times over. For inbound load, count RECEIVE; for outbound load toward the smart host or the Internet, count SEND.

```powershell
$start = (Get-Date).AddDays(-7)
$events = Get-TransportService | ForEach-Object {
    Get-MessageTrackingLog -Server $_.Name -ResultSize Unlimited `
        -Start $start -EventId RECEIVE
}
"{0} Nachrichten seit {1:yyyy-MM-dd}" -f $events.Count, $start
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-Server $_.Name` | Queries the tracking log of each Transport server passed through the pipeline |
| `-ResultSize Unlimited` | Removes the default limit of 1,000 returned entries |
| `-Start $start` | Lower time limit for the query; here, the last seven days |
| `-EventId RECEIVE` | Filters for exactly one event per message, here acceptance by the Transport service |
| `-f` | Format operator: inserts the values on the right into the `{0}` and `{1}` placeholders in the string |

</details>

The query deliberately runs across all Transport servers, because each server logs only its own share. Querying only one server in a cluster shows only a fraction of the load.

## Rates per minute and hour: this is where bursts become visible

The aggregation is a Group-Object on the rounded timestamp. The top minutes are your burst candidates right away:

```powershell
$proMinute = $events |
    Group-Object { $_.Timestamp.ToString("yyyy-MM-dd HH:mm") } |
    Sort-Object Count -Descending

$proMinute | Select-Object -First 10 Name, Count
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Group-Object { … }` | Groups by the return value of the script block, here the timestamp truncated to the minute |
| `Sort-Object Count -Descending` | Sorts groups in descending order by count; the busiest minutes appear first |
| `Select-Object -First 10 Name, Count` | Outputs only the ten largest groups, reduced to minute and count |

</details>

The same applies per hour and as a daily pattern (which time of day is typically under how much load):

```powershell
$events |
    Group-Object { $_.Timestamp.ToString("yyyy-MM-dd HH") } |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count

$events |
    Group-Object { $_.Timestamp.ToString("HH") } |
    Sort-Object Name |
    Format-Table Name, Count
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Group-Object { … ToString("yyyy-MM-dd HH") }` | Groups by full hours of a specific day |
| `Group-Object { … ToString("HH") }` | Groups only by time of day, aggregating across all days: the daily pattern |
| `Sort-Object Count -Descending` | Busiest hours first |
| `Sort-Object Name` | Sorts the daily pattern chronologically by time of day rather than by count |
| `Format-Table Name, Count` | Tabular output of the two columns |

</details>

A burst is only characterized once you know its duration in addition to the peak. A peak of 400/min that lasts two minutes presents a different requirement than the same peak sustained for an hour. Count the minutes above a threshold:

```powershell
$schwelle = 100
$burstMinuten = $proMinute | Where-Object Count -ge $schwelle
"{0} Minuten mit >= {1}/min, Peak: {2}/min" -f $burstMinuten.Count,
    $schwelle, ($proMinute | Select-Object -First 1).Count
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Where-Object Count -ge $schwelle` | Filters for minutes with at least the threshold number of messages (simplified syntax without a script block) |
| `Select-Object -First 1` | First group in the descending-sorted list, therefore the busiest minute |
| `-f` | Format operator: inserts count, threshold, and peak into placeholders `{0}` through `{2}` |

</details>

If the burst minutes are consecutive (directly visible in the output of `$burstMinuten | Sort-Object Name`), it is a batch run. Note the start time, duration, and repetition pattern, because this is exactly the window the infrastructure must support.

## Recipient structure: how many destinations, which domains

For gateways, recipient diversity is often more important than the raw rate, because lookups are performed per recipient (routing, policies, encryption rules). An email to a distribution list with 5,000 members creates a different load than 5,000 individual emails. The `RecipientCount` field and the recipient list provide both views:

```powershell
"Nachrichten: {0}, Empfänger-Zustellungen: {1}" -f $events.Count,
    ($events | Measure-Object RecipientCount -Sum).Sum

$alleEmpfaenger = $events | ForEach-Object { $_.Recipients } |
    ForEach-Object { $_.ToLower() }
"Eindeutige Empfänger: {0}" -f ($alleEmpfaenger | Sort-Object -Unique).Count
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Measure-Object RecipientCount -Sum` | Sums the `RecipientCount` field across all events: the number of recipient deliveries |
| `ForEach-Object { $_.Recipients }` | Expands each event’s recipient list into individual addresses |
| `ForEach-Object { $_.ToLower() }` | Normalizes addresses to lowercase so duplicates are recognized as such |
| `Sort-Object -Unique` | Sorts and removes duplicates; `Count` then returns the unique addresses |

</details>

The domain distribution shows where traffic is flowing. If Gmail and Microsoft dominate, their rate limits and your own IP reputation determine the achievable throughput, not your hardware:

```powershell
$alleEmpfaenger |
    ForEach-Object { ($_ -split "@")[1] } |
    Group-Object |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `($_ -split "@")[1]` | Splits the address at `@` and retains the domain part |
| `Group-Object` | Groups without an argument by the value itself, here the domain |
| `Sort-Object Count -Descending` | Most frequent domains first |
| `Select-Object -First 10 Name, Count` | Limits the output to the top 10 |

</details>

And in the other direction: Which senders (applications, functional mailboxes) generate the load in the first place? This also answers which systems must be considered during a migration:

```powershell
$events |
    Group-Object Sender |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Group-Object Sender` | Groups by the `Sender` field (positional parameter `-Property`) |
| `Sort-Object Count -Descending` | Senders with the most messages first |
| `Select-Object -First 10 Name, Count` | Limits the output to the top 10 |

</details>

## Message sizes: bytes per second instead of emails per second

Gateway throughput figures often refer to data volume rather than message count. Two systems with identical email rates differ by a factor of 100 if one sends notifications of 50 KB and the other sends invoice PDFs of 5 MB. The `TotalBytes` field provides the distribution:

```powershell
$events | Measure-Object TotalBytes -Average -Maximum -Sum |
    Select-Object @{n = "MittelKB"; e = { [math]::Round($_.Average / 1KB) } },
        @{n = "MaxMB"; e = { [math]::Round($_.Maximum / 1MB, 1) } },
        @{n = "TotalGB"; e = { [math]::Round($_.Sum / 1GB, 1) } }
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Measure-Object TotalBytes -Average -Maximum -Sum` | Calculates the average, maximum, and sum of the `TotalBytes` field in one pass |
| `@{n = "…"; e = { … }}` | Calculated property: `n` names the column, `e` returns the value through a script block, here the conversion to KB, MB, and GB |

</details>

Multiply the burst rate by the average size in the burst window, and you have the bandwidth requirement that a new gateway or WAN link must support.

## Live rates without tracking: a look at the queues

For a current snapshot (is the server processing a lot right now, is anything backing up?), you do not need tracking; the queues show it directly. `IncomingRate` and `OutgoingRate` are emails per minute, smoothed over the last few minutes:

```powershell
Get-TransportService |
    ForEach-Object { Get-Queue -Server $_.Name } |
    Sort-Object MessageCount -Descending |
    Select-Object Identity, Status, MessageCount, IncomingRate,
        OutgoingRate, NextHopDomain, LastError |
    Format-Table -AutoSize
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Get-Queue -Server $_.Name` | Lists the Transport queues of each server passed through the pipeline |
| `Sort-Object MessageCount -Descending` | Fullest queues first |
| `Select-Object Identity, Status, …` | Limits the output to the fields relevant for assessing load |
| `Format-Table -AutoSize` | Adjusts column widths to the content instead of truncating columns |

</details>

How to read it: A `Submission` queue with a high rate and a depth of 0 means the server is processing the load without backing up. `MessageCount` high with `OutgoingRate` near zero means a backlog. `Status Retry` with a 4xx message in `LastError` means the remote endpoint is throttling. `Shadow` queues with messages, on the other hand, are normal: they are redundancy copies for the partner server, not a backlog.

For a continuous curve during a load window, the Transport queues performance counter is suitable, here every five seconds for one minute:

```powershell
Get-Counter "\MSExchangeTransport Queues(_total)\Messages Completed Delivery Per Second" `
    -SampleInterval 5 -MaxSamples 12
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `"\MSExchangeTransport Queues(_total)\…"` | Performance counter path (positional parameter `-Counter`); the `_total` instance sums across all queues |
| `-SampleInterval 5` | Interval between two measurements in seconds |
| `-MaxSamples 12` | Number of measurements; 12 measurements every 5 seconds make one minute |

</details>

## Other systems: the same principle with CSV

Rather than PowerShell objects, gateways and appliances usually provide a CSV export of tracking data. The procedure remains the same (choose one event per email, group by time windows); only the tool changes, for example to Python:

```python
import csv, collections, datetime

per_min = collections.Counter()
with open("tracking-export.csv", encoding="utf-8") as f:
    reader = csv.reader(f)
    next(reader)
    for row in reader:
        if "response '2" not in row[6]:   # nur finale Zustellungen
            continue
        d = datetime.datetime.strptime(row[0][:16], "%Y-%m-%d %H:%M")
        per_min[d.strftime("%Y-%m-%d %H:%M")] += 1

print(per_min.most_common(10))
```

## The five common analysis mistakes

**Multiple events per email.** The most common source of error: counting rows instead of messages. Use `$events | Group-Object EventId` to check what is actually in your dataset, and filter for exactly one event per message.

**Truncated exports.** Many export functions return a maximum of 10,000 or 50,000 rows and then silently truncate, often in the middle of the largest burst. A suspiciously round row count is an alarm signal. Always check whether the data period matches the requested period.

**Gateway loops.** If mail flow passes through an intermediate station (encryption gateway, hygiene appliance) and returns again, the same email appears multiple times in tracking. Deduplicate using the Message ID or filter to a unique point in the chain.

**Time zones.** `Get-MessageTrackingLog` returns timestamps in local server time, while CSV exports from appliances are often in UTC. A burst that apparently occurs at 1 PM may actually be the 3 PM batch. Clarify the time basis before interpreting the data.

**Windows that are too short.** A load profile based on two quiet days is worthless if it misses the monthly invoice run. The analysis window must include the known batch cycles; ask the application owners about their sending schedules before defining the window.

## What to do with the profile

In the end, you have four numbers on one page: average rate, burst (peak, duration, time, repetition pattern), recipient structure (unique recipients per run, top domains), and size distribution. This lets you size gateways, schedule maintenance windows during nighttime hours with truly zero load, and formulate acceptance criteria, for example: The new system must process twice the measured peak without errors. The article [SMTP Load Testing with Apache JMeter in Practice](/blog/jmeter-smtp-lasttest-html-report) shows how to turn such a profile into a reproducible load test.

## Sources

1.  [Microsoft Learn: Get-MessageTrackingLog](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog): Reference for the tracking query, including all fields such as EventId, RecipientCount, and TotalBytes.

2.  [Microsoft Learn: Message tracking](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): Structure of the tracking logs, event types, and configuration of retention and directory size.

3.  [Microsoft Learn: Get-Queue](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue): Reference for the queue query, including the IncomingRate, OutgoingRate, and Velocity fields.

4.  [Microsoft Learn: Queues and messages in queues](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues): Queue types, Shadow Redundancy, and the meaning of status values.
