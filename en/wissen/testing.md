---
title: "Testing and Load Tests: Probing Infrastructure Under Controlled Conditions"
blatt: "testing"
description: "Methodology for functional and load testing in messaging and infrastructure environments: what to measure, the baseline-burst-soak sequence, load generators and sinks, marking test runs, and evaluating results with percentiles instead of averages."
fakten:
  - { label: "Basic sequence", wert: "functional test → baseline → burst → soak", href: "https://jmeter.apache.org/usermanual/best-practices.html" }
  - { label: "Four metrics", wert: "acceptance rate · session latency · end-to-end latency · queue behaviour" }
  - { label: "Evaluation", wert: "percentiles p50 · p95 · p99, not averages", href: "https://sre.google/sre-book/monitoring-distributed-systems/" }
  - { label: "Load generators", wert: "smtp-source · Postal · JMeter · custom scripts", href: "https://www.postfix.org/smtp-source.1.html" }
  - { label: "Sinks", wert: "smtp-sink · bhm · aiosmtpd · discard transport", href: "https://www.postfix.org/smtp-sink.1.html" }
  - { label: "Control experiment", wert: "independent count on the receiving side" }
  - { label: "Marking", wert: "unique header or subject marker per test run" }
  - { label: "Generator self-test", wert: "a run against the sink proves the achievable target rate" }
  - { label: "Rule number one", wert: "only your own or explicitly approved systems" }
werbung: ["newsletter"]
ctaThemen: ["smtp-mailflow"]
---

# Testing and Load Tests: Probing Infrastructure Under Controlled Conditions

Whether an environment survives its peak load, a ruleset behaves the same after a rebuild, or a storage mount holds up under real access patterns is decided by a test, not a data sheet. This page collects the methodology shared by all test types: what to measure, how a test sequence is structured, which tools generate and absorb load, and how raw data turns into defensible statements.

The most important rule comes before any methodology: load and fault tests belong exclusively on your own systems or in environments explicitly approved for them. A burst against someone else's infrastructure is an attack; a mail test with invented senders against production targets creates backscatter and blocklist entries.

## Separate Functional Tests From Load Tests

A test always answers exactly one question, and the two basic questions demand different setups. The **functional test** verifies that a path works correctly: a single SMTP transaction with full control over every step, a single file access, a single rule path through a gateway. The **load test** verifies how the system behaves under volume: throughput, latency distribution, queues, resource consumption.

The order is non-negotiable: functional test first, load second. A burst over a broken path measures nothing; it merely produces ten thousand identical errors. An error counter at 100 percent with response times near zero almost always means the failure happens before the network and the test never reached the target system.

## The Four Metrics

Load tests in multi-stage environments measure four different things, and each demands its own viewpoint:

1. **Acceptance rate:** How many operations per second does the first hop accept? This is the number load generators report directly.
2. **Session latency:** How long does a single transaction take? Under load this value often rises long before the acceptance rate collapses.
3. **End-to-end latency:** How long does an operation take across all intermediate stages? This is what users actually perceive.
4. **Queue behaviour:** How deep does the queue grow during the peak, and how quickly does it drain afterwards? A system that accepts everything and then works it off for hours passes the acceptance test and still fails.

## Test Sequence: Baseline, Burst, Soak

A defensible test runs in three stages, each with its own question. The **baseline** establishes reference values for latency and resource consumption at a moderate, known load and exposes configuration errors before they drown in the peak measurement. The **burst** is the actual peak-load measurement; several runs with increasing parallelism show where the acceptance rate flattens and latency tips over. The **soak** drives the expected sustained load for hours; only here do memory leaks, filling spool partitions, and connection limits appear that a short burst never reaches.

Three craft rules belong with this. First, the **generator self-test**: before the first run against the target system, the generator runs directly against the sink; this proves the setup can produce the target rate at all and provides the upper bound later measurements are compared against. Second, **marking**: every test operation carries a unique marker (custom header, subject convention, run number) so runs can be separated cleanly in the logs and test data can be removed completely afterwards. Third, the **control experiment**: an independent count on the receiving side confirms that what the generator sent actually arrived; if the numbers disagree, something was lost along the way.

## Tools: Generator and Sink

A load test always consists of three parts: load generator, system under test, and a controlled sink. For SMTP load, the common generators are [smtp-source](https://www.postfix.org/smtp-source.1.html) from the Postfix package (maximum raw load with minimal effort), [Postal](https://doc.coker.com.au/projects/postal/) (sustained load with a target rate and varying messages), [Apache JMeter](https://jmeter.apache.org/) (load profiles, percentiles, and an HTML report; the first choice on Windows), and custom scripts when the test needs realistic operation mixes. [swaks](https://www.jetmore.org/john/code/swaks/) is not a load generator but belongs in front of every burst as a functional tester.

On the receiving side, a sink accepts everything and discards it: [smtp-sink](https://www.postfix.org/smtp-sink.1.html), Postal's companion `bhm`, or a small [aiosmtpd](https://aiosmtpd.aio-libs.org/) server in a few lines of Python that handles the control count on the side. Developer tools with a web interface are unsuitable as load-test sinks; with tens of thousands of operations you end up measuring the analysis tool instead of the system under test.

## Evaluation: Percentiles, Pivots, Time Series

Raw data becomes a statement through three evaluation steps. **Percentiles instead of averages:** an average of two seconds can mean everything takes two seconds, or that 95 percent finish in half a second and the rest were stuck; p50, p95, and p99 distinguish these cases. **Pivot the errors:** the distribution of error codes over time shows when a system starts throttling and which protection mechanism engages first. **Time series instead of totals:** throughput and queue depth plotted over time reveal ramp-up, plateau, and drain; the plateau is what matters for interpretation, not the average over the whole run, which includes ramp-up and drain and understates the sustained rate.

The final part of any evaluation is honest scoping: every test report states what the run proves and what it does not. A measurement without TLS says nothing about TLS handshake costs; a local sink nothing about the real target's queue behaviour; a burst nothing about memory leaks in sustained operation. These boundaries define the next test runs.

The articles below apply this methodology to concrete cases: tool comparison and test planning for mail load tests, a fully executed JMeter run with an HTML report, layered SMTP diagnosis with on-board tools, a test plan for cloud storage mounts, and the scientific method behind controlled experiments in mail flow.
