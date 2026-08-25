---
title: "SMTP load testing with Apache JMeter in practice: 10,000 emails, five rule paths, one HTML report"
navTitle: "JMeter load test"
description: "An end-to-end load test: a test plan with a message mix along the ruleset paths of an encryption gateway, a portable setup without installation, 10,000 emails in a burst, and analysis using the JMeter HTML report, including the pitfalls that actually occurred."
date: "2026-08-24"
kategorie: "SMTP and mail flow"
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
url: https://rafaelpfister.ch/no/blog/smtp-load-testing-with-apache-jmeter-in-practice-10-000-emails-five-rule-paths-one-html-report
translationSourceHash: a41d58b7a4a717db179b3fec1ef8fac7961ff3ee12069f65627ddb48338aef0a
translationModel: gpt-5.6-terra
translatedAt: 2026-08-25T04:10:05.241Z
translationReview: required
---

# SMTP load testing with Apache JMeter in practice: 10,000 emails, five rule paths, one HTML report

The [overview article on email load tests](/blog/mail-lasttest-tools-linux-windows-vergleich) compared the tools and outlined the test plan. This article puts it to the test: a fully executed JMeter load test with 10,000 emails, a message mix based on real gateway rule paths, and the HTML report for analysis. All figures shown come from the actual run, including the errors that occurred along the way.

The scenario is modelled on a real project: an Apache James-based email encryption gateway (Totemomail) sits as a smarthost loop behind Exchange Online and decides per message on encryption, signing and special routing. The Mailet ruleset has several paths for this: subject triggers such as (sec), (sign) and (unsec), keywords such as VERTRAULICH for routing to an industry gateway, and the default path with certificate checking and plaintext fallback. A load test that delivers only one type of message would always measure the same path through this rule set; the test plan therefore represents five classes whose mix corresponds to the expected traffic.

## The setup: no installation required

The test ran on a Windows machine without Java or JMeter. Both can be used portably, which is crucial on admin workstations with restricted installation permissions: download the Temurin JRE as a ZIP from Adoptium, JMeter as a ZIP from apache.org, unpack both, set `JAVA_HOME` to the JRE directory, and you are done.

```bash
export JAVA_HOME="$PWD/jdk-21-jre"
export PATH="$JAVA_HOME/bin:$PATH"
./apache-jmeter-5.6.3/bin/jmeter -n -t gateway-lasttest.jmx -l lauf.jtl -e -o report
```

A local SMTP black box based on aiosmtpd served as the sink, just over 40 lines of Python: it accepts every message with `250`, discards the content, keeps a count, and assigns each email to a class based on its subject line. This independent count on the receiving side is the test's control experiment; if the generator and sink counts do not match, something was lost along the way.

```python
from aiosmtpd.controller import Controller

class SinkHandler:
    def __init__(self):
        self.count = 0

    async def handle_DATA(self, server, session, envelope):
        self.count += 1
        # Hent emnet fra headeren for klassestatistikken,
        # Innholdet forkastes
        return "250 Message accepted for delivery"

controller = Controller(SinkHandler(), hostname="127.0.0.1", port=2525)
controller.start()
```

Important for interpretation: the generator and sink ran on the same machine, without TLS and with no network in between. The measured figures therefore say nothing about a gateway, but rather represent the generator self-test from the overview article: proof that the load setup can generate the target rate at all, and the upper bound against which later measurements on the real test system can be compared.

## The test plan: five message classes, one mix

The core of the plan is a Thread Group with 20 threads, a 10-second ramp-up and 500 loops, resulting in 10,000 iterations. Beneath it are five Throughput Controllers in "Percent Executions" mode, each with exactly one SMTP Sampler:

| Class (Sampler label) | Share | Rule path in the gateway |
|---|---|---|
| 01 Standard without trigger | 60% | AutoGenerated check, certificate check, plaintext fallback |
| 02 Trigger (sec) | 15% | TRE envelope for recipients without a certificate |
| 03 Trigger (sign) | 10% | Certificate Exchange: sign, include key |
| 04 Keyword VERTRAULICH | 10% | Special routing to the industry gateway |
| 05 Trigger (unsec) | 5% | Plaintext enforced |

Splitting the workload into five separate Samplers rather than one Sampler with a variable subject has a practical reason: the HTML report groups all metrics by Sampler label. Five labels produce five rows in the statistics with their own percentiles per class; a single Sampler with a CSV-fed subject would produce one combined row, and the differences between rule paths would be invisible in the analysis.

Each Sampler fills in the usual fields: target host and port as user-defined variables (`${zielhost}`, `${zielport}`), allowing the same plan to run against the sink, test environment or PreProd without changes, along with sender, recipient, a subject with a clear marker (here, the word LOADTEST in the subject) and a text body of around 1 to 2 KB. The "Include timestamp in subject" option adds the delivery time in milliseconds; in a later run against a real multi-stage system, this can be combined with the sink's receipt timestamps to calculate end-to-end latency per message.

One pitfall from this run that can be generalised: the first attempt failed with 10,000 errors in 10 seconds, all with `java.lang.ClassCastException ... NullProperty cannot be cast to ... CollectionProperty` instead of an SMTP response. The cause was a manually built JMX file in which the Sampler's header list was missing; the Sampler requires this property even if it is empty. The lesson is less about the specific property than the pattern: build and save test plans in the GUI rather than writing XML by hand, and before every burst, perform a very small run and check at the sink that the subject and content actually arrive. An error rate of 100 percent with a response time of 0 ms almost always means the error occurred before the network, so the test never reached the target system.

## The run

The measurement itself runs in CLI mode; the GUI is only the editor. A single command generates the run, raw data and report:

```bash
jmeter -n -t gateway-lasttest.jmx -l lauf-10k.jtl -e -o report-10k
```

The Summariser on the console shows progress live; the final result of the run:

```text
summary =  10000 in 00:00:13 =  781.8/s Avg: 6 Min: 1 Max: 27 Err: 0 (0.00%)
```

10,000 messages in 12.8 seconds, 782 messages per second on average, no errors. Independently, the sink confirmed exactly 10,000 accepted emails with the mix 6000 / 1500 / 1000 / 1000 / 500, so the Throughput Controller mix matched to the exact message.

## The HTML report

The argument for JMeter over leaner generators such as smtp-source is its analysis, and the dashboard report provides it without extra work:

![JMeter dashboard for the run: APDEX 1.000 for all five classes, Requests Summary 100 percent PASS, statistics table with percentiles per message class](../images/jmeter-report-dashboard.png)

The statistics table is the most important part of the report. For each Sampler label, and thus each message class, it shows count, error rate, average, median, 90th, 95th and 99th percentile, maximum and throughput. In this specific run: median 7 ms, p95 at 11 ms, p99 at 12 ms, maximum 27 ms, practically identical across all five classes. With a local sink that treats every message identically, this is exactly the expected picture and at the same time the reference value: if the same plan is later run against the real gateway and the (sec) class suddenly shows many times the standard median, that is the additional work of the encryption path, cleanly isolated by rule branch.

The APDEX block above condenses the same information into one figure per class (1.000 everywhere here, because all responses were well below the 500 ms tolerance threshold); the thresholds can be adapted to your own service targets in the report properties. The Errors block remains empty in this run, but is the first place to look when testing real systems: it groups errors by response text, making `421` throttling by the target system immediately distinguishable from connection drops.

There is a pitfall here too, and it affects every short burst: by default, the report's time-series charts use a granularity of one minute. A 13-second run therefore collapses into a single data point, and the curves under "Charts" look like a measurement error. The report can be regenerated from the existing JTL file without a new run, using a finer resolution:

```bash
jmeter -g lauf-10k.jtl -o report-fein -Jjmeter.reportgenerator.overall_granularity=1000
```

With second-level granularity, the single point becomes the actual load profile:

![Hits per Second with 1-second granularity: increase during the 10-second ramp-up to a plateau of around 840 messages per second, then a steep drop at the end of the test](../images/jmeter-report-hits-per-second.png)

The curve shows the 10-second ramp-up, a plateau of around 840 messages per second, and the drop at the end when the first threads complete their 500 loops. For interpretation, the plateau matters, not the average over the entire run: the average of 782/s includes ramp-up and wind-down and underestimates the sustained rate achieved.

## What this run proves and what it does not

This run proves the following: the test plan is functionally correct (small test run with content verification at the sink), the mix is exact, and the generator can achieve at least 840 messages per second on this machine without TLS. Anyone wanting to test a gateway designed for 100 emails per second has a factor-of-eight reserve and can confidently attribute bottlenecks to the target system.

Everything else is not proven, and this distinction belongs in every test report: there is no statement about TLS handshake costs (the real path uses STARTTLS), about the gateway's queue behaviour, or about the processing time of the rule paths. For that, the same plan points to the gateway test environment with the variables changed to `zielhost`/`zielport`; the analysis then runs identically, supplemented by the gateway logs and queue observation from the overview article. This reusability—a single plan for sink, test environment and PreProd with identical analysis—is the real reason to invest the effort in creating a clean JMeter plan once.

## Kilder

1.  [Apache JMeter User's Manual: Component Reference, SMTP Sampler](https://jmeter.apache.org/usermanual/component_reference.html): Reference for the Sampler fields, including headers, timestamp option and EML sending.

2.  [Apache JMeter: Generating Dashboard Report](https://jmeter.apache.org/usermanual/generating-dashboard.html): Generating the HTML report from the run or afterwards from the JTL, including granularity and APDEX properties.

3.  [Apache JMeter: Test Plan, Logic Controllers](https://jmeter.apache.org/usermanual/component_reference.html#Throughput_Controller): How the Throughput Controller works in Percent Executions mode for the message mix.

4.  [aiosmtpd, documentation](https://aiosmtpd.aio-libs.org/): The asyncio-based SMTP server used to create the sink in a few lines of Python.

5.  [Eclipse Temurin, Adoptium](https://adoptium.net/temurin/releases/): Portable JRE archives for running JMeter without a Java installation.
