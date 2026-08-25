---
title: "Planning Email Load Tests: A Comparison of Tools for 10,000-Email Bursts on Linux and Windows"
navTitle: "Email Load Tests"
description: "Anyone migrating a gateway or sizing an email environment needs reliable figures rather than gut feelings. Which tools generate bursts of tens of thousands of emails, what a sound test plan looks like, and how to evaluate the results from logs."
date: "2026-08-24"
kategorie: "SMTP and Mail Flow"
timeToRead: "12 min read"
themen:
  - smtp-mailflow
  - testing
produkte:
  - "uebergreifend"
protokolle:
  - "testing"
  - "smtp"
  - "tcp"
  - "tls"
  - "troubleshooting"
slug: "planning-email-load-tests-a-comparison-of-tools-for-10-000-email-bursts-on-linux-and-windows"
translationId: "article-14a98de0cef45565"
aiPrompt: |
  Du bist mein Assistent für Mail-Lasttests. Hilf mir Schritt für Schritt, einen Lasttest gegen mein eigenes Test-Mailgateway zu planen: Zieldefinition (Durchsatz, Latenz, Queue-Verhalten), Wahl des Lastgenerators (smtp-source, Postal, JMeter oder Skript), Aufbau einer Mail-Senke, Testablauf (Baseline, Burst, Soak) und Auswertung der Logs mit Perzentilen. Frage zuerst nach Plattform, Zielsystem und erwartetem Mailvolumen.
translationOf: mail-lasttest-tools-linux-windows-vergleich
url: https://rafaelpfister.ch/en/blog/planning-email-load-tests-a-comparison-of-tools-for-10-000-email-bursts-on-linux-and-windows
translationSourceHash: c9b76f3c9887117756e07c71a3dc30d1ee99aeb8a322c50dee994a07df46cb97
translationModel: gpt-5.6-terra
translatedAt: 2026-08-25T04:10:29.468Z
translationReview: automatic
---

# Planning Email Load Tests: A Comparison of Tools for 10,000-Email Bursts on Linux and Windows

Whether a new email gateway can handle the peak load of a night when invoices are processed is not revealed in the datasheet, but in testing. Anyone replacing an appliance, sizing an Exchange environment, or planning to send a newsletter through their own infrastructure needs reliable figures beforehand: How many messages per second does the system accept, how does the queue behave under pressure, and at what point do deferrals begin? This article compares common load generators on Linux and Windows and shows how to plan, run, and evaluate a test with bursts of tens of thousands of emails.

First, the most important rule: Load tests belong exclusively in your own infrastructure or in a test environment explicitly approved for that purpose. A burst against third-party systems is an attack, and a test using invented sender addresses against production targets generates backscatter that leads to blocklists. A proper setup consists of a load generator, the system under test, and a controlled sink that accepts and discards the emails at the end.

## What an Email Load Test Should Measure

Before discussing tools, it is worth asking which metric actually matters. In practice, there are four different ones, and they require different test setups:

1. **Acceptance rate:** How many messages per second does the first hop accept via SMTP? This is the classic throughput measurement and the metric load generators provide directly.
2. **Session latency:** How long does an individual SMTP transaction take from connection setup to `250` after `DATA`? Under load, this value often rises long before the acceptance rate drops.
3. **End-to-end latency:** How long does a message take from submission to delivery to the sink, across all intermediate stages? This is the metric users notice.
4. **Queue behavior:** How large does the queue grow during the burst, and how quickly does it drain afterward? A gateway that accepts 50,000 emails and then processes them for three hours passes the acceptance test but still fails overall.

A test that measures only the acceptance rate says little about a multistage environment with a gateway, encryption layer, and destination server. End-to-end visibility is especially important there.

## Tools on Linux

**smtp-source and smtp-sink** from the Postfix package are the standard for raw SMTP load and are available on virtually every system with Postfix installed. `smtp-source` generates messages with configurable size, concurrency, and quantity, while `smtp-sink` is its counterpart: an SMTP server that accepts and discards everything. A burst of 10,000 emails with 50 concurrent sessions and 5 KB messages is a one-liner:

```bash
time smtp-source -s 50 -m 10000 -l 5120 -c -f last@test.example -t senke@test.example gateway.test.example:25
```

The `-c` option counts submitted messages live, while `time` provides the total duration and thus the rate. Important limitations: `smtp-source` does not measure latency percentiles, and its messages are synthetically uniform. For the question “how much can the system accept at most,” it is still the first choice because it generates tens of thousands of messages per minute even on modest hardware, and the generator is almost never the bottleneck.

**Postal** is the classic dedicated mail server benchmark on Linux. It automatically varies senders, recipients, and message sizes, maintains a target rate over long periods, and writes minute-by-minute statistics. This makes it better suited than `smtp-source` for soak tests, meaning sustained load over hours. The accompanying `bhm` (Black Hole Mailer) acts as the sink. Postal is old, but built exactly for this purpose and included in the package repositories of most distributions.

**swaks** is not a load generator, but it belongs in every test plan. It performs a single SMTP transaction with full control over every step: authentication, STARTTLS, arbitrary headers, and attachments. Before every load test, run swaks as a functional test so that the burst does not fail because of an incorrect recipient or a TLS issue, rendering the measurement useless. In a loop with `xargs -P`, swaks can also be misused as a small load generator, but for tens of thousands of emails, the process overhead is too high.

**Custom scripts** in Python (smtplib, aiosmtplib) or Go are the right approach when the test needs realistic message mixes: different sizes, real attachments, varying recipient counts per transaction, and targeted error cases. The effort is greater, but the script measures exactly what the environment will later see and can record timestamps per message for latency analysis.

## Tools on Windows

**Apache JMeter** is the top recommendation on Windows. Its built-in SMTP Sampler supports authentication, STARTTLS, attachments, and EML files as a message source, while the JMeter framework provides what the Postfix tools lack: thread groups for staged load profiles, response-time percentiles, error rates, and reports. For bursts beyond a few thousand emails per minute, follow the standard JMeter rule: use the GUI only to create the test plan, and run the measurement itself in CLI mode; otherwise, you are measuring the interface too.

**PowerShell with MailKit** is the built-in-tools approach. Microsoft itself marks the formerly common `Send-MailMessage` as obsolete and recommends moving on; MailKit can be loaded through NuGet and parallelized from PowerShell 7 using runspaces. This realistically allows several hundred to a few thousand emails per minute—enough for functional and regression tests, but too little for maximum-load measurement. The advantage is that the script runs without additional installation on any administrator workstation and can write results directly as CSV for analysis.

**Microsoft Exchange Load Generator (LoadGen)** was for years the official tool for loading Exchange environments with simulated user profiles (Outlook, ActiveSync, OWA). Microsoft stopped maintaining it after Exchange 2013 and removed the download. LoadGen was the wrong tool for pure SMTP load anyway; anyone wanting to simulate Exchange mailbox load today has no official tool and is better off testing the SMTP path directly.

**WSL** deserves its own mention: Anyone working on a Windows machine who needs Linux tools can install `smtp-source` and Postal in a WSL distribution and gain the full Linux toolset without a separate test VM. For the loads discussed here, the WSL network path is not a relevant bottleneck.

## Comparison

| Tool | Platform | Strength | Limitation |
|---|---|---|---|
| smtp-source / smtp-sink | Linux (Postfix) | Maximum raw load with minimal effort, generator and sink from one source | No latency percentiles, uniform messages |
| Postal / bhm | Linux | Sustained load with target rate, varying messages, minute-by-minute statistics | Aging tooling, analysis must be built yourself |
| swaks | Linux, Windows (Perl) | Fully controllable individual test, ideal as a functional check before the burst | Not a load generator |
| JMeter (SMTP Sampler) | Windows, Linux (Java) | Load profiles, percentiles, reports, EML message sources | Java overhead, GUI trap at high rates |
| PowerShell + MailKit | Windows | No additional installation on any admin workstation, CSV output | Limited throughput, parallelization must be built yourself |
| Custom script (Python/Go) | both | Realistic message mix, custom measurement points | Development effort, generator must be validated yourself |

## The Sink: Where the Emails Go

The underestimated half of the test setup is the destination. Three options have proven effective:

- **smtp-sink** or `bhm` as a black hole: accepts everything, discards everything, and measures the pure transport chain. `smtp-sink` can optionally generate artificial response delays and error codes, allowing you to test how the system under test behaves with a slow or uncooperative destination.
- **Postfix with discard transport** as a more realistic sink when the destination itself should be a full SMTP server with queueing.
- **A small number of real seed mailboxes** in addition to the sink, to spot-check that messages arrive intact in terms of content, including the encryption or signing layer.

Tools with a web interface such as Mailpit are intended for development and quickly become a bottleneck themselves with tens of thousands of emails. They are unsuitable as a sink for a load test; the measurement would benchmark the analysis tool instead of the system under test.

## Planning the Test

A reliable test runs in three stages, each with its own question:

1. **Baseline:** A moderate, known load (around 10 percent of the expected peak) for several minutes. It provides baseline values for latency and resource consumption and exposes configuration errors before they are obscured in the burst measurement.
2. **Burst:** The actual peak-load measurement, for example 10,000 to 50,000 emails as fast as possible or at a defined target rate. Multiple runs with increasing concurrency show where the acceptance rate levels off and latency tips over.
3. **Soak:** The expected daily load over several hours. Only here do memory leaks, filling spool partitions, log rotation under load, and connection limits emerge—issues a short burst never reaches.

For the message mix, use as much realism as necessary. A measurement using only 5 KB text emails overestimates the throughput of an environment whose everyday traffic includes PDF attachments, potentially by multiples. A mix from your own inventory is sensible, such as 70 percent small, 25 percent with a typical attachment, and 5 percent large. TLS should likewise be included if production uses TLS: The handshake costs significantly more per connection than message transfer itself, and generators that open a new connection per email otherwise primarily measure TLS termination.

For later analysis, give every test message a unique marker, most simply a custom header such as `X-Loadtest-Id` with a run number and timestamp, plus a recognizable subject convention. This lets you cleanly separate test runs from one another and from other traffic in the logs, and remove test emails from quarantines and journals selectively after the run.

Three points regularly forgotten in planning: First, rate limits and tarpitting on the test path; a gateway that throttles after 100 emails per minute per source IP otherwise tests only its own throttling (explicitly exempt it for maximum-load measurement, deliberately keep it in place for a reality check). Second, DNS: If the system under test resolves recipient domains or performs DNSBL lookups for every message, the test environment needs a resolver too; otherwise, the test measures upstream DNS. Third, the generator itself: Before the first run against the target system, run the generator directly against the sink to prove that it can produce the target rate at all.

## Evaluation

The load generator’s metrics are only half the truth because they show the submitter’s perspective. The other half is in the logs of the system under test.

With Postfix, the mail log provides the fields `delay` and `delays`, the latter broken down by time in the queue, connection setup, and transfer. You can evaluate a test run using built-in tools:

```bash
grep "status=sent" /var/log/mail.log |
  grep -o "delay=[0-9.]*" | cut -d= -f2 | sort -n |
  awk '{a[NR]=$1} END {print "n="NR, "p50="a[int(NR*0.5)], "p95="a[int(NR*0.95)], "p99="a[int(NR*0.99)], "max="a[NR]}'
```

On the Exchange side, the Message Tracking Log is the central source. For a test run using a subject convention:

```powershell
$p = @{
    Start          = "24.08.2026 14:00"
    End            = "24.08.2026 15:00"
    MessageSubject = "LOADTEST"
    ResultSize     = "Unlimited"
}
Get-MessageTrackingLog @p | Group-Object EventId |
    Sort-Object Count -Descending | Format-Table Name, Count
```

The difference between the timestamps of RECEIVE and DELIVER events for the same MessageId gives the end-to-end latency per message; once exported as CSV, it can be used to calculate the percentile distribution.

Three principles matter when interpreting results. First: percentiles instead of averages. An average of two seconds can mean everything takes two seconds, or that 95 percent completes in half a second while the rest sits in the queue; p50, p95, and p99 distinguish those cases. Second: pivot SMTP response codes. The distribution of 4xx responses over time shows when the system begins throttling, and which codes they are (connection limit, queue protection, greylisting) shows which mechanism engages first. Third: plot queue depth over time, under Postfix using `qshape` or `postqueue -j`, and under Exchange using `Get-Queue` at one-minute intervals. The area under this curve—not the acceptance rate—determines whether the environment absorbs a burst or merely stores it.

Alongside mail logs, include the system metrics of the system under test in the analysis: CPU, I/O wait times on the spool partition, file descriptors, and connection counts. The most common finding in multistage environments is that the mail process is not the limiting factor, but rather a content inspection layer (virus scanner, encryption module, DLP) with a fixed worker count. These findings are the real value of the test: They identify the adjustment point before production finds it.

## Conclusion

For a quick maximum-load measurement on Linux, there is no substitute for `smtp-source` with `smtp-sink`; Postal complements it for sustained-load scenarios. On Windows, JMeter provides the most complete measurement, PowerShell with MailKit covers functional and regression tests, and WSL brings Linux tools to the administrator workstation when needed. More important than the tool is the plan: separate measurement of acceptance, latency, and queue behavior; a realistic message mix; a marked test run; and an evaluation that includes percentiles and logs from the target system rather than only the generator counter.

## Sources

1.  [smtp-source(1), Postfix Manual](https://www.postfix.org/smtp-source.1.html): Reference for the load generator with all options for concurrency, message size, and TLS.

2.  [smtp-sink(1), Postfix Manual](https://www.postfix.org/smtp-sink.1.html): Reference for the mail sink, including artificial delays and error responses.

3.  [Postal Documentation, Russell Coker](https://doc.coker.com.au/projects/postal/): Description of the mail server benchmark with target rate, message variation, and the bhm sink.

4.  [swaks, John Jetmore](https://www.jetmore.org/john/code/swaks/): The SMTP functional tester for checking every test path in advance.

5.  [Apache JMeter Component Reference: SMTP Sampler](https://jmeter.apache.org/usermanual/component_reference.html): SMTP Sampler capabilities, including authentication, TLS, and EML sources.

6.  [Send-MailMessage, Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/send-mailmessage): Official Microsoft notice that the cmdlet is obsolete, with references to alternatives such as MailKit.

7.  [MailKit, Jeffrey Stedfast](https://github.com/jstedfast/MailKit): The .NET email library for custom sending scripts in PowerShell 7.

8.  [Get-MessageTrackingLog, Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog): Reference for evaluating the Exchange Message Tracking Log after a test run.

9.  [qshape(1), Postfix Manual](https://www.postfix.org/qshape.1.html): Tool for analyzing queue distribution during and after the burst.
