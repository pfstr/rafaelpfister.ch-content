---
title: "Determining a mail server's load profile: bursts, peak rates, and recipient structure from message tracking"
navTitle: "Determine load profile"
description: "How many emails per minute does your mail server actually process, and how high are the peaks? How to use PowerShell to determine the real load profile from Exchange Message Tracking: rates per minute and hour, burst duration, recipient structure, message sizes. Including the typical analysis pitfalls."
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
url: https://rafaelpfister.ch/en/blog/determining-a-mail-server-s-load-profile-bursts-peak-rates-and-recipient-structure-from-message
translationSourceHash: 16095cf53ce6f67abe31387ce2f02958eacc3898d3a42b61ad8c7b885ab7ce5d
translationModel: gpt-5.6-terra
translatedAt: 2026-08-26T04:09:14.524Z
translationReview: required
---

# Determining a mail server's load profile: bursts, peak rates, and recipient structure from message tracking

Whether you need to replace a gateway, size a server, or plan a maintenance window, every mail administrator eventually needs an answer to the question of how much their system actually processes. Intuition is regularly wrong here, because mail traffic is rarely uniform. A system that sees 20 emails per minute on average over the day may need to process 400 per minute for an hour during an invoice run. If you only know the average, you are sizing for the wrong problem.

A useful load profile consists of four metrics: the average rate (per minute, hour, and day), bursts (how high is the peak, how long does it last, and when does it occur), recipient structure (how many different recipients, which destination domains), and message sizes. All four are available in Message Tracking, and on Exchange they can be calculated with a few lines of PowerShell.

## The data source: Message Tracking

Exchange logs every message in the Message Tracking Log. Before analyzing it, check how far back the data goes; the default is 30 days, but a tight size limit can significantly shorten actual retention:

```powershell
Get-TransportService |
    Select-Object Name, MessageTrackingLogMaxAge,
        MessageTrackingLogMaxDirectorySize, MessageTrackingLogPath
```

For a load profile, the period should cover at least one full company batch cycle: monthly invoice runs, payroll, newsletters. One week is the minimum; one month is better.

## Collecting raw data: one event per message

The most important initial decision is: Which event counts as "one email"? Message Tracking writes multiple entries per message (RECEIVE when accepted, SEND when handed off to the next hop, DELIVER when delivered to a mailbox, plus AGENTINFO, HAREDIRECT, and others). Simply counting all rows overestimates volume several times over. For inbound load, count RECEIVE; for outbound load toward the smarthost or Internet, count SEND.

```powershell
$start = (Get-Date).AddDays(-7)
$events = Get-TransportService | ForEach-Object {
    Get-MessageTrackingLog -Server $_.Name -ResultSize Unlimited `
        -Start $start -EventId RECEIVE
}
"{0} Nachrichten seit {1:yyyy-MM-dd}" -f $events.Count, $start
```

The query deliberately runs across all transport servers, because each server logs only its own share. Querying just one server in a cluster shows only a fraction of the load.

## Rates per minute and hour: this is where bursts become visible

The aggregation is a Group-Object on the rounded timestamp. The top minutes are your burst candidates directly:

```powershell
$proMinute = $events |
    Group-Object { $_.Timestamp.ToString("yyyy-MM-dd HH:mm") } |
    Sort-Object Count -Descending

$proMinute | Select-Object -First 10 Name, Count
```

The same by hour and as an intraday pattern (which time of day typically has how much load):

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

A burst is only characterized once you know its duration as well as its peak. A peak of 400/min that lasts two minutes is a different requirement from the same peak lasting an hour. Count the minutes above a threshold:

```powershell
$schwelle = 100
$burstMinuten = $proMinute | Where-Object Count -ge $schwelle
"{0} Minuten mit >= {1}/min, Peak: {2}/min" -f $burstMinuten.Count,
    $schwelle, ($proMinute | Select-Object -First 1).Count
```

If the burst minutes are contiguous (directly visible in the output of `$burstMinuten | Sort-Object Name`), it is a batch run. Note the start time, duration, and recurrence pattern, because this is exactly the window the infrastructure must handle.

## Recipient structure: how many destinations, which domains

For gateways, recipient diversity is often more important than the raw rate, because each recipient requires lookups (routing, policies, encryption rules). One email to a distribution list with 5,000 members puts a different load on the system than 5,000 individual emails. The `RecipientCount` field and the recipient list provide both perspectives:

```powershell
"Nachrichten: {0}, Empfänger-Zustellungen: {1}" -f $events.Count,
    ($events | Measure-Object RecipientCount -Sum).Sum

$alleEmpfaenger = $events | ForEach-Object { $_.Recipients } |
    ForEach-Object { $_.ToLower() }
"Eindeutige Empfänger: {0}" -f ($alleEmpfaenger | Sort-Object -Unique).Count
```

The domain distribution shows where traffic is going. If Gmail and Microsoft dominate, their rate limits and your own IP reputation determine achievable throughput, not your hardware:

```powershell
$alleEmpfaenger |
    ForEach-Object { ($_ -split "@")[1] } |
    Group-Object |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count
```

And in the other direction: Which senders (applications, functional mailboxes) generate the load in the first place? This also answers which systems need to be considered during a migration:

```powershell
$events |
    Group-Object Sender |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count
```

## Message sizes: bytes per second instead of emails per second

Gateway throughput figures often refer to data volume, not message count. Two systems with identical email rates differ by a factor of 100 if one sends notifications of 50 KB and the other sends invoice PDFs of 5 MB. The `TotalBytes` field provides the distribution:

```powershell
$events | Measure-Object TotalBytes -Average -Maximum -Sum |
    Select-Object @{n = "MittelKB"; e = { [math]::Round($_.Average / 1KB) } },
        @{n = "MaxMB"; e = { [math]::Round($_.Maximum / 1MB, 1) } },
        @{n = "TotalGB"; e = { [math]::Round($_.Sum / 1GB, 1) } }
```

Multiply the burst rate by the average size in the burst window, and you have the bandwidth requirement that a new gateway or WAN link must handle.

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

How to read it: A `Submission` queue with a high rate and depth 0 means the server is processing the load without backing up. `MessageCount` high while `OutgoingRate` is near zero means a backlog. `Status Retry` with a 4xx message in `LastError` means the remote system is throttling. `Shadow` queues with items, on the other hand, are normal: they are redundancy copies for the partner server, not a backlog.

For a continuous chart during a load window, the transport queue performance counter is suitable; here, every five seconds for one minute:

```powershell
Get-Counter "\MSExchangeTransport Queues(_total)\Messages Completed Delivery Per Second" `
    -SampleInterval 5 -MaxSamples 12
```

## Other systems: the same principle with CSV

Instead of PowerShell objects, gateways and appliances usually provide a CSV export of tracking data. The approach remains identical (choose one event per email, group by time windows); only the tool changes, for example to Python:

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

## The five classic analysis pitfalls

**Multiple events per email.** The most common source of error: counting rows instead of messages. Use `$events | Group-Object EventId` to check what is actually in your data set, and filter for exactly one event per message.

**Truncated exports.** Many export functions return a maximum of 10,000 or 50,000 rows and then silently truncate the result, often right in the middle of the largest burst. A suspiciously round row count is an alarm signal. Always check whether the data period matches the requested period.

**Gateway loops.** If mail flow passes through an intermediate system (encryption gateway, hygiene appliance) and then back again, the same email appears multiple times in tracking. Deduplicate by Message-ID or filter for a unique point in the chain.

**Time zones.** `Get-MessageTrackingLog` provides timestamps in local server time, whereas CSV exports from appliances are often in UTC. A burst that appears to occur at 1 PM may actually be the 3 PM batch. Clarify the time basis before interpreting the data.

**Windows that are too short.** A load profile from two quiet days is worthless if it misses the monthly invoice run. The analysis window must include the known batch cycles; ask the application owners for their sending schedules before defining the window.

## What to do with the profile

At the end, you have four figures on one page: average rate, burst (peak, duration, timing, recurrence pattern), recipient structure (unique recipients per run, top domains), and size distribution. This lets you size gateways, schedule maintenance windows in nighttime hours with actual zero load, and define acceptance criteria, for example: The new system must process twice the measured peak without errors. The article [SMTP load testing with Apache JMeter in practice](/blog/jmeter-smtp-lasttest-html-report) shows how to turn such a profile into a reproducible load test.

## Sources

1.  [Microsoft Learn: Get-MessageTrackingLog](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog): Reference for the tracking query, including all fields such as EventId, RecipientCount, and TotalBytes.

2.  [Microsoft Learn: Message tracking](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): Structure of the tracking logs, event types, and configuration of retention and directory size.

3.  [Microsoft Learn: Get-Queue](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue): Reference for the queue query, including the IncomingRate, OutgoingRate, and Velocity fields.

4.  [Microsoft Learn: Queues and messages in queues](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues): Queue types, Shadow Redundancy, and the meaning of status values.
