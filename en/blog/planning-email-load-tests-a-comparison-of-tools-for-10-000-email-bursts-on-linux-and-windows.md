---
title: "Planning email load tests: Tools for 10,000-email bursts under Linux and Windows compared"
navTitle: "Email load tests"
description: "Anyone migrating a gateway or sizing an email environment needs reliable figures rather than gut feelings. Which tools generate bursts of tens of thousands of emails, what a clean test plan looks like, and how to evaluate the results from logs."
date: "2026-08-24"
kategorie: "SMTP and mail flow"
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
translationSourceHash: 2fd0b1bd0748b9fb44be85907a946bbf85604b5eb7c85107170fa7443068efd7
translationModel: gpt-5.6-terra
translatedAt: 2026-08-30T09:25:09.102Z
translationReview: required
url: https://rafaelpfister.ch/en/blog/planning-email-load-tests-a-comparison-of-tools-for-10-000-email-bursts-on-linux-and-windows
---

# Planning email load tests: Tools for 10,000-email bursts under Linux and Windows compared

Whether a new mail gateway can handle the peak load of a nightly invoice run can only be verified with a load test. Anyone replacing an appliance, sizing an Exchange environment, or planning a newsletter mailing through their own infrastructure needs reliable figures in advance: How many messages per second does the system accept, how does the queue behave under pressure, and at what point do deferrals begin? This article compares common load generators under Linux and Windows and shows how to plan, run, and evaluate a test with bursts of tens of thousands of emails.

First, the most important rule: Load tests belong exclusively in your own infrastructure or in a test environment explicitly approved for that purpose. A burst against third-party systems is an attack, and a test using fictitious sender addresses against production targets generates backscatter that leads to blocklisting. A proper setup consists of a load generator, the system under test, and a controlled sink that accepts and discards the emails at the end.

## What an email load test should measure

Before discussing a tool, it is worth asking what metric is actually of interest. In practice, there are four different ones, and they require different test setups:

1. **Acceptance rate:** How many messages per second does the first hop accept via SMTP? This is the classic throughput measurement and the value delivered directly by load generators.
2. **Session latency:** How long does a single SMTP transaction take from connection establishment to `250` after `DATA`? Under load, this value often rises long before the acceptance rate drops.
3. **End-to-end latency:** How long does a message take from submission to delivery at the sink, across all intermediate stages? This is the metric users notice.
4. **Queue behavior:** How deeply does the queue grow during the burst, and how quickly does it drain afterward? A gateway that accepts 50,000 emails and then processes them for three hours passes the acceptance test but still fails overall.

A test that measures only the acceptance rate says little about a multi-stage environment with a gateway, encryption layer, and destination server. That is precisely where the end-to-end view matters.

## The load profile determines the tool

In addition to the metric, a second question determines the choice of tool, and it is often skipped: What connection behavior does the load being simulated have? Two load profiles must be distinguished.

A **bulk sender with open sessions** is the load profile of invoice runs, payroll statements, and newsletter systems: A single system establishes a small number of connections and sends hundreds to thousands of messages over them in one go. Connection overhead occurs once per session, not once per message, and the gateway sees few connections with many transactions.

**Many independent submitters** are the load profile of application landscapes and user traffic: Numerous systems each submit individual messages over separate connections. Here, connection establishment including TLS and AUTH is part of every message.

For sizing a bulk mailing, the first load profile matters, and the load generator must be able to keep sessions open: `smtp-source` does this (many messages distributed across a few sessions), as do Postal and custom scripts with persistent connections. JMeter cannot; the reasons are explained in the Windows section. For the peak load of an invoice run, this session criterion is therefore decisive, not the platform; under Windows, this means using WSL.

## Tools under Linux

**smtp-source and smtp-sink** from the Postfix package are the standard for raw SMTP load and are available on virtually every system with Postfix installed. `smtp-source` generates messages with configurable size, parallelism, and quantity; `smtp-sink` is its counterpart: an SMTP server that accepts and discards everything. A burst of 10,000 emails with 50 parallel sessions and 5 KB messages is a one-liner:

```bash
time smtp-source -s 50 -m 10000 -l 5120 -c -f last@test.example -t senke@test.example gateway.test.example:25
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `time` | Measures the total duration of the command; this yields the rate in emails per second |
| `-s 50` | 50 parallel SMTP sessions |
| `-m 10000` | Total number of messages, distributed across the sessions |
| `-l 5120` | Message body size in bytes (without headers), 5 KB here |
| `-c` | Running count of submitted messages as a progress indicator |
| `-f last@test.example` | Sender address |
| `-t senke@test.example` | Recipient address |
| `gateway.test.example:25` | Target host and port for submission |

</details>

Important limitations: `smtp-source` does not measure latency percentiles, and the messages are synthetically uniform. For the question “how much can the system accept at most,” it is still the first choice because it produces tens of thousands of messages per minute even on modest hardware and the generator virtually never becomes the bottleneck.

**Postal** is the classic dedicated mail server benchmark under Linux. It automatically varies sender, recipient, and message size, maintains a target rate over long periods, and writes minute-by-minute statistics. This makes it better suited than `smtp-source` for soak tests, meaning sustained load over hours. The associated `bhm` (Black Hole Mailer) takes on the role of the sink. Postal is old, but built precisely for this purpose and included in the package repositories of most distributions.

**swaks** is not a load generator, but it belongs in every test plan. It performs a single SMTP transaction with full control over every step: authentication, STARTTLS, arbitrary headers, and attachments. Before every load test, run swaks as a functional test so that the burst does not fail because of an incorrect recipient or a TLS issue, rendering the measurement worthless. In a loop with `xargs -P`, swaks can also be misused as a small load generator, but process overhead is too high for tens of thousands of emails.

**Custom scripts** in Python (smtplib, aiosmtplib) or Go are the path when the test needs realistic message mixes: different sizes, real attachments, varying numbers of recipients per transaction, and targeted error cases. The effort is greater, but the script measures exactly what the environment will later see and can write timestamps for each message for latency evaluation.

## Tools under Windows

**Apache JMeter** is the right tool under Windows when the load profile consists of many independent submitters or when percentiles, message mix, and reports are the priority. The built-in SMTP Sampler supports Auth, STARTTLS, attachments, and EML files as message sources, and the JMeter mechanism provides what the Postfix tools lack: thread groups for graduated load profiles, response-time percentiles, error rates, and reports. For bursts beyond a few thousand emails per minute, the usual JMeter rule applies: Use the GUI only to create the test plan; run the measurement itself in CLI mode, otherwise you are measuring the interface as well.

One limitation of the SMTP Sampler must be understood: JMeter cannot keep SMTP sessions open. Every sample run opens a new connection, goes through the complete dialog of TCP handshake, EHLO, potentially STARTTLS and AUTH, sends exactly one message, and closes the connection with QUIT. Multiple messages over the same open connection, as bulk senders do with session reuse, cannot be modeled; `smtp-source`, by contrast, distributes many messages across a few open sessions. The reason lies in the architecture: JMeter is a cross-protocol load-testing framework, not an SMTP tool. Its execution model treats every sampler as a self-contained, independently measured unit, because only this makes timers, assertions, and percentile evaluation work consistently across all supported protocols. The SMTP Sampler is accordingly a thin wrapper around the JavaMail library, which as a client API establishes and closes a connection for each send operation; connection reuse across samples, such as the HTTP Sampler offers with Keep-Alive, was never implemented for SMTP. For measurement, this means: JMeter produces the load profile of many individual submitters, not that of a bulk sender with an open session. The measured throughput includes full connection and TLS overhead for every message, and connection limits at the gateway therefore take effect earlier than with session reuse. For the bulk-sender load profile of an invoice run, JMeter is therefore not the right tool; under Windows, the WSL approach with `smtp-source` is the better choice.

**PowerShell with MailKit** is the built-in approach. Microsoft itself labels the formerly common `Send-MailMessage` as obsolete and recommends migrating; MailKit can be installed via NuGet and parallelized from PowerShell 7 using runspaces. This realistically supports a few hundred to a few thousand emails per minute, enough for functional and regression tests but too little for maximum-load measurement. The advantage: The script runs without additional installation on any administrator workstation and can write results directly as CSV for evaluation.

**Microsoft Exchange Load Generator (LoadGen)** was for years the official tool for loading Exchange environments with simulated user profiles (Outlook, ActiveSync, OWA). Microsoft stopped maintaining it after Exchange 2013 and removed the download. LoadGen was the wrong tool for pure SMTP load anyway; anyone who wants to simulate Exchange mailbox load today has no official tool and is better off testing the SMTP path directly.

**WSL** deserves its own point: Anyone working on a Windows machine but needing Linux tools can install `smtp-source` and Postal in a WSL distribution, gaining the full Linux toolset without a separate test VM. For the loads discussed here, the WSL network path is not a relevant bottleneck.

## Comparison

| Tool | Platform | Strength | Limitation |
|---|---|---|---|
| smtp-source / smtp-sink | Linux (Postfix) | Maximum raw load with minimal effort, generator and sink from one source | No latency percentiles, uniform messages |
| Postal / bhm | Linux | Sustained load with target rate, varying messages, minute-by-minute statistics | Aging tooling, build evaluation yourself |
| swaks | Linux, Windows (Perl) | Fully controllable individual test, ideal as a functional check before the burst | Not a load generator |
| JMeter (SMTP Sampler) | Windows, Linux (Java) | Load profiles, percentiles, reports, EML message sources | Java overhead, GUI mode unsuitable for high rates, one connection per message (no session reuse) |
| PowerShell + MailKit | Windows | No additional installation on any admin workstation, CSV output | Limited throughput, build parallelization yourself |
| Custom script (Python/Go) | both | Realistic message mix, custom measurement points | Development effort, validate the generator yourself |

## The sink: where the emails go

The underestimated half of the test setup is the target. Three options have proven effective:

- **smtp-sink** or `bhm` as a black hole: accepts everything, discards everything, and measures the pure transport chain. `smtp-sink` can optionally generate artificial response delays and error codes, thus also testing how the system under test behaves with a slow or faulty responding target.
- **Postfix with discard transport** as a more realistic sink when the target itself should be a full SMTP server with queueing.
- **A small number of real seed mailboxes** in addition to the sink, to spot-check that messages arrive intact, including encryption or signing layers.

Tools with a web interface such as Mailpit are intended for development and quickly become a bottleneck themselves with tens of thousands of emails. They are unsuitable as a sink for a load test; the measurement would benchmark the analysis tool rather than the system under test.

## Planning the test

A reliable test runs in three stages, each with its own question:

1. **Baseline:** A moderate, known load (around 10 percent of the expected peak) for several minutes. It provides reference values for latency and resource consumption and reveals configuration errors before they disappear in the burst measurement.
2. **Burst:** The actual peak-load measurement, for example 10,000 to 50,000 emails as fast as possible or at a defined target rate. Multiple runs with increasing parallelism show where the acceptance rate levels off and latency tips over.
3. **Soak:** The expected daily load over several hours. Only here do memory leaks, full spool partitions, log rotation under load, and connection limits emerge—issues a short burst never reaches.

For the message mix, the rule is: as realistic as necessary. A measurement with only 5 KB plain-text emails overestimates the throughput of an environment whose daily operation includes PDF attachments by a multiple. A mix from your own inventory makes sense, for example 70 percent small, 25 percent with a typical attachment, and 5 percent large. TLS also belongs in the test if production uses TLS: The handshake costs significantly more per connection than message transmission itself, and generators that open a new connection for every email otherwise primarily measure TLS termination.

For later evaluation, every test message needs a unique marker, most easily a dedicated header such as `X-Loadtest-Id` with run number and timestamp, plus a recognizable subject convention. This cleanly separates test runs in the logs from each other and from other traffic, and lets you remove test emails from quarantines and journals in a targeted manner after the run.

Three points are regularly forgotten during planning: First, rate limits and tarpitting on the test path; a gateway that throttles after 100 emails per minute per source IP otherwise tests only its own throttling (explicitly exempt it for maximum-load measurement; deliberately keep it enabled for a reality check). Second, DNS: If the system under test resolves recipient domains or performs DNSBL queries for every message, a resolver must be part of the test environment, otherwise the test measures upstream DNS. Third, the generator itself: Before the first run against the target system, run the generator directly against the sink to prove that it can generate the target rate at all.

## Evaluation

The load generator’s measurements are only half the truth because they show the submitter’s perspective. The other half is in the logs of the system under test.

Under Postfix, the mail log provides the fields `delay` and `delays` for each message, with the latter broken down into time in the queue, connection establishment, and transfer. An evaluation across a test run can be done with built-in tools:

```bash
grep "status=sent" /var/log/mail.log |
  grep -o "delay=[0-9.]*" | cut -d= -f2 | sort -n |
  awk '{a[NR]=$1} END {print "n="NR, "p50="a[int(NR*0.5)], "p95="a[int(NR*0.95)], "p99="a[int(NR*0.99)], "max="a[NR]}'
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `grep "status=sent" /var/log/mail.log` | Filters the mail log for successfully delivered messages |
| `grep -o "delay=[0-9.]*"` | `-o` outputs only the match itself, here the `delay` field with its value |
| `cut -d= -f2` | Splits on `=` (`-d`) and retains the second field (`-f2`), i.e., the numeric value |
| `sort -n` | Sorts numerically rather than alphabetically; required for calculating percentiles |
| `awk '…'` | Collects the sorted values in an array and outputs the count, p50, p95, p99, and maximum |

</details>

On the Exchange side, the Message Tracking Log is the central source. For a test run with a subject convention:

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

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Start` / `End` | Time window for the log search; passed here via splatting (`@p`) |
| `MessageSubject "LOADTEST"` | Filters for messages whose subject contains the marker |
| `ResultSize Unlimited` | Removes the default limit of 1,000 returned entries |
| `Group-Object EventId` | Groups tracking events by type (RECEIVE, DELIVER, DEFER, …) |
| `Sort-Object Count -Descending` | Sorts event groups in descending order by frequency |
| `Format-Table Name, Count` | Displays the count for each event type |

</details>

The difference between the timestamps of RECEIVE and DELIVER events for the same MessageId gives the end-to-end latency for each message; exported as CSV, this can be used to calculate the percentile distribution.

Three principles matter when interpreting the results. First: Percentiles instead of averages. An average of two seconds can mean that everything takes two seconds, or that 95 percent complete in half a second while the rest sit in the queue; p50, p95, and p99 distinguish these cases. Second: Pivot SMTP response codes. The distribution of 4xx responses over time shows when the system begins throttling, and which codes they are (connection limit, queue protection, greylisting) shows which mechanism takes effect first. Third: Plot queue depth over time, under Postfix using `qshape` or `postqueue -j`, and under Exchange using `Get-Queue` at one-minute intervals. The area beneath this curve, not the acceptance rate, determines whether the environment absorbs a burst or merely stores it.

System metrics from the system under test should be included in the evaluation alongside the mail logs: CPU, I/O wait times on the spool partition, file descriptors, and connection counters. The most common finding in multi-stage environments is that the mail process is not the limiting factor, but rather a content-inspection stage (antivirus scanner, encryption module, DLP) with a fixed number of workers. Findings like these are the real value of the test: They identify the adjustment point before production finds it.

## Conclusion

For quick maximum-load measurement under Linux, there is no alternative to `smtp-source` with `smtp-sink`; Postal complements it for sustained-load scenarios. Under Windows, JMeter provides the most complete measurement, PowerShell with MailKit covers functional and regression testing, and WSL brings Linux tools to the administrator workstation when needed. More important than the tool is the plan: Separate measurement of acceptance, latency, and queue behavior; a realistic message mix; a marked test run; and an evaluation that includes percentiles and logs from the target system rather than just the generator’s counter.

## Sources

1.  [smtp-source(1), Postfix Manual](https://www.postfix.org/smtp-source.1.html): Reference for the load generator with all options for parallelism, message size, and TLS.

2.  [smtp-sink(1), Postfix Manual](https://www.postfix.org/smtp-sink.1.html): Reference for the mail sink, including artificial delays and error responses.

3.  [Postal documentation, Russell Coker](https://doc.coker.com.au/projects/postal/): Description of the mail server benchmark with target rate, message variation, and bhm sink.

4.  [swaks, John Jetmore](https://www.jetmore.org/john/code/swaks/): The SMTP functional tester for checking every test path in advance.

5.  [Apache JMeter Component Reference: SMTP Sampler](https://jmeter.apache.org/usermanual/component_reference.html): SMTP Sampler functionality, including Auth, TLS, and EML sources.

6.  [Send-MailMessage, Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/send-mailmessage): Official Microsoft notice that the cmdlet is obsolete, with references to alternatives such as MailKit.

7.  [MailKit, Jeffrey Stedfast](https://github.com/jstedfast/MailKit): The .NET mail library for custom sending scripts under PowerShell 7.

8.  [Get-MessageTrackingLog, Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog): Reference for evaluating the Exchange Message Tracking Log after a test run.

9.  [qshape(1), Postfix Manual](https://www.postfix.org/qshape.1.html): Tool for analyzing queue distribution during and after the burst.
