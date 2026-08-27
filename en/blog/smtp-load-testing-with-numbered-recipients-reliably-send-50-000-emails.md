---
title: "SMTP Load Testing with Numbered Recipients: Reliably Send 50,000 Emails"
navTitle: "Numbered Load Tests"
description: "A load test is only as good as its evaluation. With the -N option, smtp-source numbers each email through the recipient address without sacrificing throughput. Learn how to set up a run with 50,000 emails, how many sessions make sense, and how to find missing numbers automatically."
date: "2026-08-27"
kategorie: "SMTP and Mail Flow"
timeToRead: "8 min read"
themen:
  - smtp-lasttests
  - smtp-mailflow
produkte:
  - "uebergreifend"
protokolle:
  - "smtp"
  - "testing"
slug: "smtp-load-testing-with-numbered-recipients-reliably-send-50-000-emails"
translationId: "article-57f09c758baf6e1e"
translationOf: smtp-lasttest-nummerierte-empfaenger
url: https://rafaelpfister.ch/en/blog/smtp-load-testing-with-numbered-recipients-reliably-send-50-000-emails
translationSourceHash: a2ec75884c06a6d736ea9b5895211ddc4cbba252c7ddf491752e1bec5ab1a24d
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:19:16.303Z
translationReview: automatic
---

# SMTP Load Testing with Numbered Recipients: Reliably Send 50,000 Emails

Anyone running a load test with 50,000 emails wants to be able to answer two questions afterward: Did they all arrive, and if not, which ones are missing? With identical test emails, you can only count, and a difference of 49,987 versus 50,000 says nothing about when and where the 13 missing messages were lost. If every email carries a sequential number, however, counting becomes reconciliation: Every number can be found individually in the target system's logs, gaps reveal when the loss occurred, and the delivery order can be verified.

The common reflex is a script that increments the subject line. That works, but it costs throughput because the load generator `smtp-source` from the Postfix package sets the subject once per invocation, and a loop with one invocation per email forces a separate connection setup for every message. The better message identifier is already built in: The `-N` option numbers the recipient address for each message, all within a single invocation with parallel sessions. For evaluation, the recipient address is just as useful as the subject because it appears in every tracking log.

Unlike a pure loopback functional test, this test setup sends to another system over the network. If Postfix is not installed on the source system, the article [smtp-source without a Postfix installation](https://rafaelpfister.ch/blog/smtp-source-ohne-postfix-installation) shows how to extract the tools from the RPM.

## The most important smtp-source options

For reference, here are the options used in this article, translated in essence from the man page:

| Option | Meaning |
|---|---|
| `-s n` | Number of parallel SMTP sessions (default: 1) |
| `-m n` | Total number of messages to send (default: 1) |
| `-l n` | Message body size in bytes, excluding headers |
| `-f adresse` | Sender address |
| `-t adresse` | Recipient address (default: `foo@hostname`) |
| `-S text` | Subject line, fixed for all messages in the invocation |
| `-F datei` | Sends headers and body unchanged from a file; overrides `-l` and `-S` |
| `-N` | Numbers the recipient address for each message (per-process counter; position and starting value depend on the version; see below) |
| `-r n` | Number of recipients per message (default: 1); address construction as with `-N` |
| `-d` | Do not disconnect after a message; send the next one over the same connection |
| `-c` | Display a running counter that increments with each completed `DATA` |
| `-w n` | Fixed wait time of n seconds between messages (per session) |
| `-v` | Verbose output for troubleshooting |
| `host:port` | TCP delivery target; without a port, the default smtp port is used |

The complete list, including TLS, LMTP, and timing options, is in the `smtp-source(1)` man page; its counterpart on the receiving side is `smtp-sink(1)` and is used for the evaluation below.

## How -N numbers recipients

`-N` activates a per-process counter that is embedded in the recipient address. Three properties determine the test setup, and all three can be read in the `smtp-source.c` source code:

First, the exact address format depends on the Postfix version. Postfix 3.5, as provided by RHEL 8, prepends the number to the entire address (`RCPT TO:<%d%s>`): `-t test@example.com` becomes `1test@example.com`, `2test@example.com`, and so on, with the counter starting at 1. Current Postfix versions instead append the number to the end of the local part and start at 0 (`test0@` through `test49999@`); for this variant, the man page recommends plus addressing (`-t 'test+@example.com'` produces `test+0@` and subsequent addresses) so that a target system with subaddressing assigns all of them to the same mailbox. Before the large run, verify the format with a handful of emails against a `smtp-sink` or in the target log; the expected set and evaluation search pattern depend on it.

Second, the counter is process-wide and shared by all parallel sessions. With `-s 8`, the eight sessions assign the numbers collectively, and each number occurs exactly once. The order across sessions is not deterministic, but completeness of the set of numbers is guaranteed.

Third, the starting value cannot be configured: 1 in Postfix 3.5 and 0 in current versions. The 50,000 emails therefore carry numbers 1 through 50,000 or 0 through 49,999, respectively, and the expected set for reconciliation must match.

## The test run in one invocation

```bash
smtp-source -c -d -N -s 8 -m 50000 -l 5120 \
  -f lasttest@example.com \
  -t test@example.com \
  gateway.example.com:25
```

| Option | Effect |
|---|---|
| `-c` | Running counter of completed deliveries as a single-line progress indicator |
| `-d` | Connections remain open across all messages; without `-d`, a new connection is created per message |
| `-N` | Recipient numbering: appends the per-process counter to the local part |
| `-s 8` | Eight parallel SMTP sessions |
| `-m 50000` | Total number of messages, distributed across sessions |
| `-l 5120` | Message size in bytes (excluding headers), 5 KB here |
| `-f` | Sender address |
| `-t` | Base recipient address; `-N` turns it into `1test@` through `50000test@` (Postfix 3.5), or `test0@` through `test49999@` (current versions) |
| `gateway.example.com:25` | Target host and port |

`-d` is crucial for the load profile: Without this option, `smtp-source` disconnects after every message and establishes a new connection for the next one; with `-d`, the eight connections remain open and deliver all messages sequentially, as a bulk sender does.

The `-v` familiar from functional tests is deliberately omitted: It logs every individual SMTP dialog from `HELO` through `QUIT` and produces hundreds of thousands of log lines for 50,000 emails without adding value to the evaluation. `-c` instead provides the summary from which the progress of the run can be read live. A prefixed `time` provides the total duration for calculating the rate.

The prerequisite for this entire approach is that the target system accepts the generated addresses. A `smtp-sink`, a catch-all domain, a provider's discard domain, or a gateway that resolves recipients only after acceptance meet this requirement. If the target checks every recipient against a directory, however, it rejects the numbered addresses, leaving only the subject-line variant.

## Setting custom headers

Some load tests need a custom header, for example as a marker by which the gateway identifies test emails or a rule applies. `smtp-source` has no option for this, but `-F` reads a fully preformatted message from a file, where any desired header can be included. The file consists of header lines, a blank line, and the body, with every line ending in `\r\n`:

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
| `-F datei` | Sends headers and body unchanged from the file; replaces the generated message content |

There are two consequences: `-F` overrides `-l` and `-S` because size and subject now come from the file (which is why both must be included there). `-N` remains effective, and recipients continue to be numbered; the header is identical in all messages because it comes from the fixed file.

## How many sessions?

The most reliable way to determine the appropriate number of sessions is by measurement, using exactly the same options as the planned main run: the same message source (the same `-F` file or the same `-l`), the same sender, and the same target. Only the quantity is reduced to 2,000 per stage, and `-s` varies. A short calibration run with increasing session counts shows when additional sessions no longer help:

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

Two details about the invocation: `-c` is intentionally omitted here so there are no running counter outputs between measurement lines; the loop produces exactly one result line per stage. And the empty local part in `-t` works well with numbering for a discard domain: With Postfix 3.5's prepended counter, it creates purely numeric recipient addresses (`1@blackhole.example.com`, `2@…`), which keeps log evaluation clear.

Specifically, the outer loop iterates through session counts from 1 to 32 in doubling steps. Before and after each run, `date +%s%N` records the current time as a large number: Unix seconds immediately followed by the nanosecond portion. Between them, `smtp-source` sends 2,000 messages (content, headers, and size come from the `-F` file) over the respective number of parallel connections that remain open thanks to `-d`; the loop waits until the invocation has fully completed. The `echo` line converts the time difference into a rate: 2,000 emails divided by the runtime in seconds, with runtime expressed in nanoseconds. This turns 2,000 times 10⁹ into the constant `2000000000000`. The Bash arithmetic `$(( ))` uses integer calculations and truncates decimal places, which is accurate enough for this measurement.

Three practical notes: `%N` provides nanoseconds only in GNU date (as is the case on RHEL and most Linux systems; BusyBox and macOS do not support it). The complete run sends 6 × 2,000 = 12,000 emails, so those also need a controlled recipient address, and `-N` numbering starts again at the starting value in every invocation. If an `smtp-source` invocation aborts with an error message, the rate for that line is meaningless; fix the cause first, then measure again.

The expected output is one line per stage. With fictional but typical example values, it looks like this:

```text
1 Sessions: 11 Mails/s
2 Sessions: 21 Mails/s
4 Sessions: 40 Mails/s
8 Sessions: 71 Mails/s
16 Sessions: 79 Mails/s
32 Sessions: 80 Mails/s
```

The interpretation: As long as the rate roughly doubles with the session count, parallel sessions are masking the wait time for responses from the target; the bottleneck is then path latency, not capacity. Once the curve levels off (between 8 and 16 sessions in the example), either the target system is saturated or the source has reached its limit. Use the smallest value at which the rate no longer increases significantly—8 to 16 in the example; additional sessions then increase only the load from parallelism, not throughput. For the main run with 50,000 emails, the measured rate can also immediately estimate the expected duration: At 71 emails/s, about 12 minutes.

## Evaluation on the receiving side

If a dedicated test receiver is available on the target system, `smtp-sink` handles the logging as well:

```bash
smtp-sink -c -d "mails/%Y%m%d-%H%M%S." 0.0.0.0:2525 200
```

| Option | Effect |
|---|---|
| `-c` | Running counters instead of the full SMTP dialog |
| `-d "mails/…"` | For the sink: dump, not connection persistence. Writes every accepted message to a separate file (filename pattern via strftime), including an `X-Rcpt-Args` header with the recipient address |
| `0.0.0.0:2525` | Listens on all interfaces on port 2525 |
| `200` | Backlog: maximum length of the queue of pending connections according to listen(2) |

After the run, extract the received numbers and compare them with the expected set. Since the numbers have no leading zeros, both lists are padded to a fixed width before comparison so that the alphabetical sort order of `comm` matches the numeric order. The search pattern matches the Postfix 3.5 address format (number before the address); for current versions, use `test[0-9]+@` and `seq 0 49999` accordingly:

```bash
grep -rhoE '[0-9]+test@example\.com' mails/ | \
  grep -oE '^[0-9]+' | sort -u | \
  awk '{printf "%08d\n", $1}' | sort > empfangen.txt
```

```bash
seq 1 50000 | awk '{printf "%08d\n", $1}' | \
  comm -23 - empfangen.txt
```

`comm -23` outputs exactly the numbers that appear in the expected set but not in the received list: the missing emails. Empty output means complete delivery. If numbers appear twice (visible from the difference between `sort` and `sort -u`), a system along the way duplicated the message, which is also a finding.

If the target is a production-like system rather than an smtp-sink, its logging takes the place of the dump files. On an Exchange server, for example, `Get-MessageTrackingLog -Recipients` or a filter on the recipient address provides the numbers that arrived; on a Postfix system, use `grep` on `to=` and the base address in the mail log. That is precisely the advantage of the number in the address: The recipient appears in every message tracking record, whereas the subject may be absent depending on the system or must first be enabled.

## When the number must be in the subject

Some evaluations depend on the subject, for example when the target system rewrites recipient addresses or logs show the recipient only in masked form. In that case, the loop variant remains: one `smtp-source` invocation per email with `-m 1` and a subject incremented by the shell, distributed among multiple parallel workers with contiguous number ranges.

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

The cost is a full connection setup per email: TCP handshake, banner, `HELO`, sending, `QUIT`. This run therefore does not measure the target system's maximum throughput, but a deliberately connection-intensive scenario. Determine the number of workers analogously to the calibration run above, only with the worker loop instead of `-s`. The leading zeros in the subject save the reformatting needed for reconciliation by the `-N` variant.

## Rules for tests against other systems

As soon as the test leaves your own system, three conditions apply. First: The operator of the target system knows about it and has agreed to the time window; 50,000 emails look like an attack or a spam wave to any monitoring system. Second: The recipient address terminates in a controlled location—a dedicated test mailbox, a discard rule on the target, or a provider-designated discard domain; production addresses do not belong in a load test. Third: An abort criterion is defined before starting, such as a growing queue on the target or an error rate above a threshold, and someone monitors those values during the run.

With these three points and the numbering, the run ultimately provides not only a throughput number but a verifiable statement: which of the 50,000 emails arrived, which are missing, and where they were last seen along the path.

## Sources

1.  [Postfix: smtp-source(1)](https://www.postfix.org/smtp-source.1.html): Load generator man page; describes the `-N` behavior of the current version (counter in the local part, plus addressing).

2.  [Postfix source code 3.5.8: smtp-source.c](https://github.com/vdukhovni/postfix/blob/v3.5.8/postfix/src/smtpstone/smtp-source.c): Documents prepending the number (`RCPT TO:<%d%s>`) with a starting value of 1 for the RHEL 8 version; in the [current version](https://github.com/vdukhovni/postfix/blob/master/postfix/src/smtpstone/smtp-source.c), the number is appended to the local part instead, starting at 0.

3.  [Postfix: smtp-sink(1)](https://www.postfix.org/smtp-sink.1.html): Test receiver man page with the dump options and recorded X headers.

4.  [GNU Coreutils: comm](https://www.gnu.org/software/coreutils/manual/html_node/comm-invocation.html): Comparison of two sorted lists, used here to reconcile expected and received numbers.
