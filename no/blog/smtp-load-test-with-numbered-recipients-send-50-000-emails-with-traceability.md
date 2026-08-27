---
title: "SMTP-load test with numbered recipients: send 50,000 emails with traceability"
navTitle: "Numbered load tests"
description: "A load test is only as good as its evaluation. With the -N option, smtp-source numbers every email via the recipient address without sacrificing throughput. Learn how to set up a run with 50,000 emails, how many sessions make sense and how to find missing numbers automatically."
date: "2026-08-27"
kategorie: "SMTP and mail flow"
timeToRead: "8 min read"
themen:
  - smtp-lasttests
  - smtp-mailflow
produkte:
  - "uebergreifend"
protokolle:
  - "smtp"
  - "testing"
slug: "smtp-load-test-with-numbered-recipients-send-50-000-emails-with-traceability"
translationId: "article-57f09c758baf6e1e"
translationOf: smtp-lasttest-nummerierte-empfaenger
url: https://rafaelpfister.ch/no/blog/smtp-load-test-with-numbered-recipients-send-50-000-emails-with-traceability
translationSourceHash: a2ec75884c06a6d736ea9b5895211ddc4cbba252c7ddf491752e1bec5ab1a24d
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:22:58.410Z
translationReview: automatic
---

# SMTP-load test with numbered recipients: send 50,000 emails with traceability

Anyone running a load test with 50,000 emails wants to be able to answer two questions afterwards: Did they all arrive, and if not, which ones are missing? With identical test emails, you can only count, and a difference between 49,987 and 50,000 says nothing about when and where the 13 missing messages were lost. If each email instead carries a sequential number, counting becomes reconciliation: each number can be found individually in the target system's logs, gaps reveal when the loss occurred, and the delivery order can be checked.

The common reflex is a script that increments the subject line. That works, but it costs throughput because the load generator `smtp-source` from the Postfix package sets the subject once per invocation, and a loop with one invocation per email forces a separate connection setup for every message. The better message identifier is already built in: the `-N` option numbers the recipient address for each message, within a single invocation with parallel sessions. For evaluation, the recipient address is just as useful as the subject line because it appears in every tracking log.

Unlike a pure loopback functional test, this test setup sends over the network to another system. If Postfix is not installed on the source system, the article [smtp-source without a Postfix installation](https://rafaelpfister.ch/blog/smtp-source-ohne-postfix-installation) shows you how to extract the tools from the RPM.

## The most important smtp-source options

For orientation, here are the options used in this article, translated in essence from the man page:

| Option | Meaning |
|---|---|
| `-s n` | Number of parallel SMTP sessions (default: 1) |
| `-m n` | Total number of messages to send (default: 1) |
| `-l n` | Message body size in bytes, excluding headers |
| `-f adresse` | Sender address |
| `-t adresse` | Recipient address (default: `foo@hostname`) |
| `-S text` | Subject line, fixed for all messages in the invocation |
| `-F datei` | Sends headers and body unchanged from a file; overrides `-l` and `-S` |
| `-N` | Numbers the recipient address for each message (per-process counter; position and starting value depend on the version, see below) |
| `-r n` | Number of recipients per message (default: 1), address formation as with `-N` |
| `-d` | Do not disconnect after a message; send the next one over the same connection |
| `-c` | Display a running counter that increments with every completed `DATA` |
| `-w n` | Fixed wait time of n seconds between messages (per session) |
| `-v` | Verbose output for troubleshooting |
| `host:port` | TCP delivery target; without a port specification, the default smtp port is used |

The full list, including TLS, LMTP and timing options, is in the `smtp-source(1)` man page; its counterpart for the receiving side is `smtp-sink(1)` and is used later in the evaluation.

## How -N numbers recipients

`-N` enables a per-process counter that is built into the recipient address. Three properties determine the test setup, all three can be read in the source code of `smtp-source.c`:

First, the exact address form depends on the Postfix version. Postfix 3.5, as provided by RHEL 8, prefixes the number to the full address (`RCPT TO:<%d%s>`): `-t test@example.com` becomes `1test@example.com`, `2test@example.com` and so on, and the counter starts at 1. Current Postfix versions instead append the number to the end of the local part and start at 0 (`test0@` through `test49999@`); for this variant, the man page recommends plus addressing (`-t 'test+@example.com'` becomes `test+0@` and subsequent addresses) so that a target system with subaddressing assigns everything to the same mailbox. Before the large run, check the form with a handful of emails against a `smtp-sink` or in the target's log; the expected set and the search pattern used for evaluation depend on it.

Second, the counter is process-wide and shared by all parallel sessions. With `-s 8`, the eight sessions assign the numbers together; each number occurs exactly once. The order across sessions is not deterministic, but completeness of the number set is guaranteed.

Third, the starting value cannot be configured: 1 in Postfix 3.5 and 0 in current versions. The 50,000 emails therefore carry numbers 1 through 50,000 or 0 through 49,999, respectively, and the expected set used for reconciliation must match.

## The test run in one invocation

```bash
smtp-source -c -d -N -s 8 -m 50000 -l 5120 \
  -f lasttest@example.com \
  -t test@example.com \
  gateway.example.com:25
```

| Option | Effect |
|---|---|
| `-c` | Running counter of completed deliveries as a single-line progress display |
| `-d` | Connections remain open across all messages; without `-d`, a new connection is created per message |
| `-N` | Recipient numbering: appends the per-process counter to the local part |
| `-s 8` | Eight parallel SMTP sessions |
| `-m 50000` | Total number of messages, distributed across sessions |
| `-l 5120` | Message size in bytes (excluding headers), 5 KB here |
| `-f` | Sender address |
| `-t` | Base recipient address; `-N` turns it into `1test@` through `50000test@` (Postfix 3.5) or `test0@` through `test49999@` (current versions) |
| `gateway.example.com:25` | Target host and port |

`-d` is crucial for the load profile: without this option, `smtp-source` disconnects after every message and establishes a new connection for the next one; with `-d`, the eight connections remain open and deliver all messages in sequence, as a bulk sender does.

The `-v` familiar from functional tests is deliberately omitted: it logs every individual SMTP dialog from `HELO` to `QUIT` and produces hundreds of thousands of log lines for 50,000 emails without adding value to the evaluation. `-c` instead provides the summary from which the run's progress can be monitored live. A preceding `time` provides the total duration for calculating the rate.

A prerequisite for the entire approach is that the target system accepts the generated addresses. A `smtp-sink`, a catch-all domain, a provider discard domain or a gateway that resolves recipients only after acceptance meets this requirement. If the target checks every recipient against a directory, however, it rejects the numbered addresses and only the subject-line variant remains.

## Setting custom headers

Some load tests need a custom header, for example as a marker by which the gateway recognises the test emails or to trigger a rule. `smtp-source` has no option for this, but `-F` reads a fully preformatted message from a file, where any desired header can be included. The file consists of header lines, a blank line and the body, with all lines terminated by `\r\n`:

```bash
{ printf 'X-Lasttest: aktiv\r\n'
  printf 'Subject: Lasttest\r\n'
  printf '\r\n'
  head -c 5120 /dev/zero | tr '\0' 'x'
  printf '\r\n'; } > lasttest.eml
```

```bash
smtp-source -c -d -N -s 8 -m 50000 -F lasttest.eml \
  -f lasttest@example.com \
  -t test@example.com \
  gateway.example.com:25
```

| Option | Effect |
|---|---|
| `-F datei` | Sends headers and body unchanged from the file; replaces generated message content |

This has two consequences: `-F` overrides `-l` and `-S` because size and subject now come from the file (which must therefore contain both). `-N` remains effective, however, and recipients continue to be numbered; the header is identical in all messages because it comes from the fixed file.

## How many sessions?

The most reliable way to determine the appropriate number of sessions is through measurement, using exactly the same options as for the planned main run: the same message source (the same `-F` file or the same `-l`), the same sender and the same target. Only the quantity is reduced to 2,000 per stage, and `-s` varies. A short calibration run with increasing session counts shows when additional sessions no longer help:

```bash
for s in 1 2 4 8 16 32; do
  t0=$(date +%s%N)
  smtp-source -d -N -s "$s" -m 2000 -F lasttest.eml \
    -f lasttest@example.com -t '@blackhole.example.com' \
    gateway.example.com:25
  t1=$(date +%s%N)
  echo "$s Sessions: $(( 2000000000000 / (t1 - t0) )) Mails/s"
done
```

Two details about the invocation: `-c` is deliberately omitted here so that no running counter output appears between the measurement lines; the loop produces exactly one result line per stage. And the empty local part in `-t` works well with numbering on a discard domain: with the prefixed counter in Postfix 3.5, it produces purely numerical recipient addresses (`1@blackhole.example.com`, `2@…`), keeping evaluation in the logs clear.

Specifically, the following happens: the outer loop goes through session counts 1 to 32 in doubling increments. Before and after each run, `date +%s%N` records the current time as a large number: Unix seconds immediately followed by the nanosecond component. In between, `smtp-source` sends 2,000 messages (content, headers and size come from the `-F` file) through the respective number of parallel connections, which remain open thanks to `-d`; the loop waits until the invocation is fully complete. The `echo` line converts the time difference to a rate: 2,000 emails divided by the runtime in seconds, with runtime available in nanoseconds. Thus, 2,000 times 10⁹ produces the constant `2000000000000`. The Bash arithmetic `$(( ))` uses integer calculations and truncates decimal places, which is sufficiently accurate for this measurement.

Three practical notes: `%N` provides nanoseconds only in GNU date (as is the case on RHEL and most Linux systems; BusyBox and macOS do not support it). The complete run sends 6 × 2,000 = 12,000 emails; they too need a controlled recipient address, and `-N` numbering starts again at the initial value in every invocation. If a `smtp-source` invocation aborts with an error message, the rate for that line is meaningless; fix the cause first, then measure again.

The expected output is one line per stage. With invented but typical example values, it looks like this:

```text
1 Sessions: 11 Mails/s
2 Sessions: 21 Mails/s
4 Sessions: 40 Mails/s
8 Sessions: 71 Mails/s
16 Sessions: 79 Mails/s
32 Sessions: 80 Mails/s
```

The interpretation: as long as the rate roughly doubles with the number of sessions, the parallel sessions are masking the wait time for responses from the target; the bottleneck is then path latency, not capacity. From the point where the curve flattens (between 8 and 16 sessions in the example), either the target system is saturated or the source has reached its limit. Choose the smallest value at which the rate no longer increases significantly—8 to 16 in the example; more sessions then only increase the load through parallelism, not throughput. The measured rate can also be used to estimate the expected duration of the main run with 50,000 emails: at 71 emails/s, approximately 12 minutes.

## Evaluation on the receiving side

If a dedicated test receiver is available on the target system, `smtp-sink` also handles the logging:

```bash
smtp-sink -c -d "mails/%Y%m%d-%H%M%S." 0.0.0.0:2525 200
```

| Option | Effect |
|---|---|
| `-c` | Running counters instead of the full SMTP dialog |
| `-d "mails/…"` | For the sink: dump, not connection persistence. Writes each accepted message to its own file (filename pattern via strftime), including an `X-Rcpt-Args` header containing the recipient address |
| `0.0.0.0:2525` | Listens on all interfaces on port 2525 |
| `200` | Backlog: maximum length of the queue of pending connections according to listen(2) |

After the run, extract the received numbers and compare them with the expected set. Because the numbers have no leading zeros, both lists are padded to a fixed width before comparison so that alphabetical sorting by `comm` matches numerical sorting. The search pattern matches the Postfix 3.5 address form (number before the address); for current versions, use `test[0-9]+@` and `seq 0 49999` accordingly:

```bash
grep -rhoE '[0-9]+test@example\.com' mails/ | \
  grep -oE '^[0-9]+' | sort -u | \
  awk '{printf "%08d\n", $1}' | sort > empfangen.txt
```

```bash
seq 1 50000 | awk '{printf "%08d\n", $1}' | \
  comm -23 - empfangen.txt
```

`comm -23` outputs exactly the numbers present in the expected set but absent from the received list: the missing emails. Empty output means complete delivery. If numbers appear twice (detectable from the difference between `sort` and `sort -u`), a system along the way duplicated the message, which is also a finding.

If the target is a production-like system rather than an smtp-sink, its logging takes over the role of the dump files. On an Exchange server, for example, `Get-MessageTrackingLog -Recipients` or a filter on the recipient address provides the numbers that arrived; on a Postfix system, use `grep` on `to=` and the base address over the mail log. This is precisely the advantage of the number in the address: the recipient appears in every message tracking record, while the subject may be absent there depending on the system or may need to be enabled first.

## When the number must be in the subject line

Some evaluations depend on the subject line, for example when the target system rewrites recipient addresses or the logs show the recipient only in masked form. The loop variant then remains: one `smtp-source` invocation per email with `-m 1` and a subject that the shell increments, distributed across multiple parallel workers with contiguous number ranges.

```bash
worker() {
  local i
  for ((i = $1; i <= $2; i++)); do
    smtp-source -s 1 -m 1 -l 5120 \
      -S "$(printf 'Lasttest %05d' "$i")" \
      -f lasttest@example.com -t test@example.com \
      gateway.example.com:25 || echo "$i" >> fehlend.log
  done
}
for w in 0 1 2 3; do
  worker $(( w * 12500 + 1 )) $(( (w + 1) * 12500 )) &
done
wait
```

The cost is a complete connection setup per email: TCP handshake, banner, `HELO`, sending, `QUIT`. This run therefore does not measure the target system's maximum throughput, but an intentionally connection-intensive scenario. Determine the number of workers analogously to the calibration run above, only using the worker loop instead of `-s`. The leading zeros in the subject line eliminate the reformatting required for reconciliation by the `-N` variant.

## Rules for tests against other systems

As soon as the test leaves your own system, three conditions apply. First: the operator of the target system knows about it and has approved the time window; 50,000 emails look like an attack or a spam wave to any monitoring system. Second: the recipient address terminates in a controlled location—in a dedicated test mailbox, a discard rule on the target or a discard domain provided by the provider for this purpose; production addresses do not belong in a load test. Third: an abort criterion is defined before starting, such as a growing queue on the target or an error rate above a threshold, and someone monitors these values during the run.

With these three points and the numbering, the run ultimately provides more than just a throughput number: a verifiable statement of which of the 50,000 emails arrived, which are missing and where they were last seen along the route.

## Kilder

1.  [Postfix: smtp-source(1)](https://www.postfix.org/smtp-source.1.html): Load generator man page; describes the `-N` behaviour of the current version (counter in the local part, plus addressing).

2.  [Postfix source code 3.5.8: smtp-source.c](https://github.com/vdukhovni/postfix/blob/v3.5.8/postfix/src/smtpstone/smtp-source.c): Documents the RHEL 8 version's prefixing of the number (`RCPT TO:<%d%s>`) with a starting value of 1; in the [current version](https://github.com/vdukhovni/postfix/blob/master/postfix/src/smtpstone/smtp-source.c), the number is instead appended to the local part, starting at 0.

3.  [Postfix: smtp-sink(1)](https://www.postfix.org/smtp-sink.1.html): Test receiver man page with the dump options and recorded X headers.

4.  [GNU Coreutils: comm](https://www.gnu.org/software/coreutils/manual/html_node/comm-invocation.html): Comparison of two sorted lists, used here to reconcile expected and received numbers.
