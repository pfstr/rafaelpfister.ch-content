---
title: "SMTP Load Testing with Apache JMeter in Practice: 10,000 Emails, Five Rule Paths, One HTML Report"
navTitle: "JMeter Load Test"
description: "A complete load test from A to Z: a test plan with a message mix along the ruleset paths of an encryption gateway, a portable setup without installation, 10,000 emails in a burst, and analysis using the JMeter HTML report—including the issues that actually occurred."
date: "2026-08-24"
kategorie: "SMTP and Mail Flow"
timeToRead: "11 min read"
themen:
  - smtp-mailflow
  - testing
  - totemomail
produkte:
  - "uebergreifend"
  - "totemomail"
  - "apache-james"
protokolle:
  - "testing"
  - "smtp"
  - "troubleshooting"
related:
  - mail-lasttest-tools-linux-windows-vergleich
image: "../images/jmeter-report-dashboard.png"
slug: "smtp-load-testing-with-apache-jmeter-in-practice-10-000-emails-five-rule-paths-one-html-report"
translationId: "article-fc3f25272e051f92"
aiPrompt: |
  Du bist mein Assistent für Mail-Lasttests mit Apache JMeter. Hilf mir Schritt für Schritt, einen SMTP-Lasttest aufzubauen: portables Setup (JRE + JMeter ohne Installation), lokale SMTP-Senke mit aiosmtpd, Testplan mit Thread Group, Throughput Controllern für den Nachrichtenmix und SMTP Samplern, Lauf im CLI-Modus mit HTML-Report und Auswertung der Perzentile pro Nachrichtenklasse. Frage zuerst nach Zielsystem, Nachrichtenklassen und gewünschtem Volumen.
translationOf: jmeter-smtp-lasttest-html-report
translationSourceHash: 26c09e391d2252b6203dceb5dc45edd23beba797820fe0b95273bf48a9afc181
translationModel: gpt-5.6-terra
translatedAt: 2026-08-30T09:22:14.810Z
translationReview: required
url: https://rafaelpfister.ch/en/blog/smtp-load-testing-with-apache-jmeter-in-practice-10-000-emails-five-rule-paths-one-html-report
---

# SMTP Load Testing with Apache JMeter in Practice: 10,000 Emails, Five Rule Paths, One HTML Report

The [overview article on email load testing](/blog/mail-lasttest-tools-linux-windows-vergleich) compared the tools and outlined the test plan. This article covers the practical execution: a complete JMeter load test with 10,000 emails, a message mix along real gateway rule paths, and the HTML report for analysis. All values shown come from the actual run, including the errors that occurred along the way.

The scenario is modeled on a real project: An Apache James-based email encryption gateway (Totemomail) sits in a smarthost loop behind Exchange Online and decides per message whether to encrypt, sign, or apply special routing. The Mailet ruleset has several paths for this: subject triggers such as (sec), (sign), and (unsec), keywords such as VERTRAULICH for routing to an industry gateway, and the standard path with certificate checking and plaintext fallback. A load test that submits only one type of message would always measure the same path through this ruleset; the test plan therefore models five classes whose mix corresponds to the expected traffic.

Important context: This test plan generates the load pattern of many independent submitters, because JMeter opens a separate connection for each message (the background is covered in the limitations at the end). For demonstrating that a ruleset works correctly and fast enough under parallel mixed traffic, this is the appropriate pattern. However, the plan does not model the peak load of a single bulk sender with open sessions; for this load pattern, `smtp-source` from the [overview article](/blog/mail-lasttest-tools-linux-windows-vergleich) is the right tool.

## The most important jmeter options

For orientation, here are the command-line options used in this article, translated in essence from the documentation:

<details class="options-details">
<summary>Options at a glance</summary>

| Option | Meaning |
|---|---|
| `-n` | CLI mode (non-GUI): runs the test plan without a graphical interface |
| `-t datei` | Path to the JMX file containing the test plan |
| `-l datei` | Path to the JTL results file where measurements are written |
| `-e` | Generates the HTML dashboard report directly after the run |
| `-o verzeichnis` | Target directory for the report; must be empty or must not exist yet |
| `-g datei` | Generates the report afterward from an existing JTL file, without a new run |
| `-J<property>=<wert>` | Sets a JMeter property for this invocation only |

</details>

The complete list is shown by `jmeter -?`; the options are described in the non-GUI operation chapter of the [JMeter User's Manual](https://jmeter.apache.org/usermanual/get-started.html).

## The setup: no installation required

The test ran on a Windows machine without Java or JMeter. Both can be run portably, which is the decisive factor on admin workstations with restricted installation permissions: download the Temurin JRE as a ZIP from Adoptium, JMeter as a ZIP from apache.org, extract both, set `JAVA_HOME` to the JRE directory, and you are done.

```bash
export JAVA_HOME="$PWD/jdk-21-jre"
export PATH="$JAVA_HOME/bin:$PATH"
./apache-jmeter-5.6.3/bin/jmeter -n -t gateway-lasttest.jmx -l lauf.jtl -e -o report
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `export JAVA_HOME=…` | Points to the extracted JRE directory; JMeter finds the Java runtime through it without installation |
| `export PATH=…` | Places the JRE binaries at the beginning of the search path |
| `-n` | CLI mode without a graphical interface |
| `-t gateway-lasttest.jmx` | The test plan to execute |
| `-l lauf.jtl` | Results file containing the measurements for every sampler |
| `-e` | Generate the HTML report directly after the run |
| `-o report` | Target directory for the report |

</details>

A local SMTP black box based on aiosmtpd served as the sink, just over 40 lines of Python: it accepts every message with `250`, discards the content, counts messages, and assigns each email to a class based on its subject line. This independent count on the receiving side is the test's control experiment; if the generator and sink counts do not match, something was lost in transit.

```python
from aiosmtpd.controller import Controller

class SinkHandler:
    def __init__(self):
        self.count = 0

    async def handle_DATA(self, server, session, envelope):
        self.count += 1
        # Extract the subject from the header for class statistics,
        # Content is discarded
        return "250 Message accepted for delivery"

controller = Controller(SinkHandler(), hostname="127.0.0.1", port=2525)
controller.start()
```

Important context: The generator and sink ran on the same machine, without TLS and without a network in between. The measured figures therefore make no statement about a gateway, but rather constitute the generator self-test from the overview article: proof that the load setup can produce the target rate at all, and the upper bound against which later measurements of the actual test system can be compared.

## The test plan: five message classes, one mix

The heart of the plan is a Thread Group with 20 threads, a 10-second ramp-up, and 500 loops, for 10,000 iterations. Under it are five Throughput Controllers in "Percent Executions" mode, each with exactly one SMTP Sampler:

| Class (sampler label) | Share | Rule path in the gateway |
|---|---|---|
| 01 Standard without trigger | 60% | AutoGenerated check, certificate check, plaintext fallback |
| 02 Trigger (sec) | 15% | TRE envelope for recipients without a certificate |
| 03 Trigger (sign) | 10% | Certificate Exchange: sign, include key |
| 04 Keyword VERTRAULICH | 10% | Special routing to the industry gateway |
| 05 Trigger (unsec) | 5% | Plaintext forced |

Splitting the plan into five separate samplers rather than one sampler with a variable subject has a practical reason: The HTML report groups all metrics by sampler label. Five labels produce five rows in the statistics with their own percentiles per class; a single sampler with a CSV-fed subject would produce one aggregate row, and the difference between rule paths would be invisible in the analysis.

Each sampler populates the usual fields: target host and port as user-defined variables (`${zielhost}`, `${zielport}`), so the same plan can run unchanged against the sink, test environment, or PreProd, plus sender, recipient, a subject with a clear marker (here, the word LOADTEST in the subject), and a text body of roughly 1 to 2 KB. The "Include timestamp in subject" option adds the submission time in milliseconds; in a later run against a real multi-stage system, this can be combined with the sink's receipt timestamps to calculate end-to-end latency per message.

One error from this run that can be generalized: The first attempt failed with 10,000 errors in 10 seconds, all with `java.lang.ClassCastException ... NullProperty cannot be cast to ... CollectionProperty` instead of an SMTP response. The cause was a manually built JMX file in which the sampler's header list was missing; the sampler requires the property even when it is empty. The lesson is less about the specific property than the pattern: build and save test plans in the GUI rather than writing XML manually, and before every burst run a minimal test and check at the sink that the subject and content actually arrive. An error rate of 100 percent with a 0 ms response time almost always means that the error occurred before the network, so the test never reached the target system.

## The run

The measurement itself runs in CLI mode; the GUI is only the editor. A single invocation generates the run, raw data, and report:

```bash
jmeter -n -t gateway-lasttest.jmx -l lauf-10k.jtl -e -o report-10k
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-n` | CLI mode: the test plan runs without the GUI; only the summarizer writes to the console |
| `-t gateway-lasttest.jmx` | The test plan created in the GUI |
| `-l lauf-10k.jtl` | Raw run data; the report can be regenerated from this file later |
| `-e` | Generate the report immediately after the run |
| `-o report-10k` | Target directory for the HTML report |

</details>

The console summarizer shows progress live, followed by the final result of the run:

```text
summary =  10000 in 00:00:13 =  781.8/s Avg: 6 Min: 1 Max: 27 Err: 0 (0.00%)
```

10,000 messages in 12.8 seconds, 782 messages per second on average, no errors. The sink independently confirmed exactly 10,000 accepted emails with the mix 6000 / 1500 / 1000 / 1000 / 500, so the Throughput Controller mix matched down to the individual message.

## The HTML report

The argument for JMeter over leaner generators such as smtp-source is its analysis, and the Dashboard report provides it without additional work:

![JMeter dashboard for the run: APDEX 1.000 for all five classes, Requests Summary 100 percent PASS, statistics table with percentiles per message class](../images/jmeter-report-dashboard.png)

The statistics table is the most important part of the report. For each sampler label, and thus for each message class, it lists count, error rate, average, median, 90th, 95th, and 99th percentiles, maximum, and throughput. In the specific run: median 7 ms, p95 at 11 ms, p99 at 12 ms, maximum 27 ms, virtually identical across all five classes. With a local sink that treats every message the same, this is exactly the expected picture and also the reference value: If the same plan later runs against the real gateway and the (sec) class suddenly shows a multiple of the standard median, that is the additional work of the encryption path, cleanly isolated per rule branch.

The APDEX block above condenses the same data into one number per class (here 1.000 everywhere, because all responses were well below the 500 ms tolerance threshold); the thresholds can be adjusted in the report properties to match your own service targets. The Errors block remains empty in this run, but is the first place to look in tests against real systems: It groups errors by response text, making `421` throttling by the target system immediately distinguishable from connection drops.

A typical analysis error occurs here as well, and it affects every short burst: By default, the report's time-series charts use a granularity of one minute. A 13-second run therefore collapses into a single data point, and the curves under "Charts" look like a measurement error. The report can be regenerated from the existing JTL file without a new run at a finer resolution:

```bash
jmeter -g lauf-10k.jtl -o report-fein -Jjmeter.reportgenerator.overall_granularity=1000
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-g lauf-10k.jtl` | Generates the report from the existing JTL file without running the test again |
| `-o report-fein` | New target directory; the existing report directory remains unchanged |
| `-Jjmeter.reportgenerator.overall_granularity=1000` | Sets chart granularity for this invocation to 1,000 ms instead of the default minute |

</details>

With second-level granularity, the single point becomes the actual load profile:

![Hits per Second with 1-second granularity: rise during the 10-second ramp-up to a plateau around 840 messages per second, followed by a steep drop at the end of the test](../images/jmeter-report-hits-per-second.png)

The curve shows the 10-second ramp-up, a plateau around 840 messages per second, and the drop at the end when the first threads have completed their 500 loops. For interpretation, the plateau matters, not the average over the entire run: The average of 782/s includes ramp-up and wind-down and understates the sustained rate achieved.

## What this run demonstrates—and what it does not

This run demonstrates the following: The test plan is functionally correct (minimal run with content verification at the sink), the mix is exactly right, and the generator can achieve at least 840 messages per second on this machine without TLS. Anyone wanting to test a gateway designed for 100 emails per second has an eightfold margin and can confidently attribute bottlenecks to the target system.

It does not demonstrate anything else, and this limitation belongs in every test report: no statement about TLS handshake costs (the real path uses STARTTLS), none about the gateway's queue behavior, and none about the processing time of the rule paths. For that, the same plan with variables `zielhost`/`zielport` changed points to the gateway test environment; the analysis then runs identically, supplemented by gateway logs and queue monitoring from the overview article. This reusability—one plan for the sink, test environment, and PreProd with identical analysis—is the real reason to put the effort into a proper JMeter plan once.

A limitation of the tool itself also belongs in this discussion: JMeter cannot keep SMTP sessions open. The SMTP Sampler opens a new connection for each message, goes through EHLO, optionally STARTTLS and AUTH, and closes it after exactly one transaction with QUIT. The 840 messages per second therefore include a complete connection setup for every message. A bulk sender that sends hundreds of messages over an open session creates a different load pattern at the gateway, with fewer connections and more transactions per connection, and connection limits therefore take effect earlier with JMeter load. The reason is the framework architecture: JMeter measures every sampler as a self-contained, independent unit so timers, assertions, and percentiles work consistently across all supported protocols, and the SMTP Sampler is built on the JavaMail library, which connects and disconnects as a client API for each send operation. SMTP has no connection reuse comparable to the HTTP Sampler's Keep-Alive. For the load pattern of a bulk sender with an open session, `smtp-source` or a custom script is more suitable; the tool comparison in the overview article puts this in context.

## Sources

1.  [Apache JMeter User's Manual: Component Reference, SMTP Sampler](https://jmeter.apache.org/usermanual/component_reference.html): Reference for sampler fields, including headers, timestamp option, and EML sending.

2.  [Apache JMeter: Generating Dashboard Report](https://jmeter.apache.org/usermanual/generating-dashboard.html): Generating the HTML report from the run or afterward from the JTL, including granularity and APDEX properties.

3.  [Apache JMeter: Test Plan, Logic Controllers](https://jmeter.apache.org/usermanual/component_reference.html#Throughput_Controller): How the Throughput Controller works in Percent Executions mode for the message mix.

4.  [aiosmtpd documentation](https://aiosmtpd.aio-libs.org/): The asyncio-based SMTP server used to create the sink in just a few lines of Python.

5.  [Eclipse Temurin, Adoptium](https://adoptium.net/temurin/releases/): Portable JRE archives for running JMeter without a Java installation.

6.  [Apache JMeter: Getting Started, Non-GUI Mode](https://jmeter.apache.org/usermanual/get-started.html): Overview of command-line options for CLI operation, including `-n`, `-t`, `-l`, `-e`, `-o`, `-g` and `-J`.
