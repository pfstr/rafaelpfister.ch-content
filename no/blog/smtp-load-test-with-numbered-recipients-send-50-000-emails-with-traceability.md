---
title: "SMTP load testing with numbered recipients: send every email traceably"
navTitle: "Numbered load tests"
description: "A load test is only as good as its evaluation. With the -N option, smtp-source numbers every email through the recipient address without sacrificing throughput. How to structure the run, how many sessions make sense, and how to find missing numbers automatically."
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
translationSourceHash: 7145f2b49fb0b141d9c74d009d7c480ce4d119b4c97236e2ed7d92a39f65a1c5
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:50:31.250Z
translationReview: automatic
url: https://rafaelpfister.ch/no/blog/smtp-load-test-with-numbered-recipients-send-50-000-emails-with-traceability
---

# SMTP load testing with numbered recipients: send every email traceably

Anyone running a load test wants to be able to answer two questions afterwards: Did all emails arrive, and if not, which ones are missing? With identical test emails, you can only count, and a count showing 13 missing messages says nothing about when or where they were lost. If each email carries a sequential number, counting becomes reconciliation: Every number can be found individually in the target system's logs, gaps show when the loss occurred, and the delivery order can be checked.

The common reflex is a script that increments the subject line. That works, but costs throughput, because the load generator `smtp-source` from the Postfix package sets the subject fixed per invocation, and a loop with one invocation per email forces a separate connection setup for every message. The better message identifier is already built in: The option `-N` numbers the recipient address for each message, within a single invocation with parallel sessions. For evaluation, the recipient address is just as useful as the subject, because it appears in every tracking log.

Unlike a pure loopback functional test, this test setup sends to another system over the network. If Postfix is not installed on the source system, the article [smtp-source without a Postfix installation](https://rafaelpfister.ch/blog/smtp-source-ohne-postfix-installation) shows how to extract the tools from the RPM.

## The most important smtp-source options

For orientation, here are the options used in this article, translated in substance from the man page:

<details class="options-details">
<summary>Options at a glance</summary>

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
| `host:port` | Delivery target via TCP; without a port specification, the default smtp port |

</details>

The complete list, including TLS, LMTP and timing options, is in the `smtp-source(1)` man page; the counterpart for the receiving side is `smtp-sink(1)` and is used later in the evaluation.

## How -N numbers recipients

`-N` enables a per-process counter that is inserted into the recipient address. Three characteristics determine the test setup, and all three can be found in the source code of `smtp-source.c`:

First, the exact address format depends on the Postfix version. Postfix 3.5, as provided by RHEL 8, prefixes the number to the complete address (`RCPT TO:<%d%s>`): `-t test@example.com` becomes `1test@example.com`, `2test@example.com` and so on, and the counter starts at 1. Current Postfix versions instead append the number to the end of the local part and start at 0 (`test0@` through `test49999@`); for this variant, the man page recommends plus addressing (`-t 'test+@example.com'` becomes `test+0@` and subsequent addresses), so a target system with subaddressing assigns everything to the same mailbox. Check the format before the large run with a handful of emails against an `smtp-sink` or in the target's log; the expected quantity and the search pattern used for evaluation depend on it.

Second, the counter is process-wide and shared by all parallel sessions. With `-s 8`, the eight sessions assign the numbers jointly; every number occurs exactly once. The ordering across sessions is non-deterministic, but the completeness of the set of numbers is guaranteed.

Third, the starting value cannot be configured: 1 in Postfix 3.5, 0 in current versions. The emails therefore carry numbers from 1 up to the total count from `-m`, or from 0 up to total count minus 1, and the expected set for reconciliation must match.

## The test run in one invocation

How many emails the run contains does not matter to the approach; `-m` determines the total count, and the examples in this article use 50,000 as an arbitrary placeholder.

```bash
smtp-source -c -d -N -s 8 -m 50000 -l 5120 \
  -f lasttest@example.com \
  -t test@example.com \
  gateway.example.com:25
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-c` | Running counter of completed deliveries as a one-line progress display |
| `-d` | Connections remain open across all messages; without `-d`, a new connection per message |
| `-N` | Recipient numbering: appends the per-process counter to the local part |
| `-s 8` | Eight parallel SMTP sessions |
| `-m 50000` | Total number of messages, distributed across sessions |
| `-l 5120` | Message size in bytes (excluding headers), 5 KB here |
| `-f` | Sender address |
| `-t` | Base recipient address; `-N` turns it into `1test@`, `2test@` and so on (Postfix 3.5), or `test0@`, `test1@` and so on (current versions) |
| `gateway.example.com:25` | Target host and port |

</details>

`-d` is decisive for the load profile: Without this option, `smtp-source` disconnects after every message and establishes a new connection for the next one; with `-d`, the eight connections remain open and deliver all messages in sequence, as a bulk sender does.

The familiar `-v` from functional tests is deliberately omitted: It logs every individual SMTP dialogue from `HELO` to `QUIT` and produces hundreds of thousands of log lines in a large run, with no added value for evaluation. `-c` instead provides the summary that lets you follow the run's progress live. A preceding `time` provides the total duration for rate calculation.

A prerequisite for the entire approach: The target system accepts the generated addresses. An `smtp-sink`, a catch-all domain, a provider discard domain, or a gateway that resolves recipients only after acceptance meet this requirement. If the target checks every recipient against a directory, however, it rejects the numbered addresses, leaving only the subject variant.

## Setting custom headers

Some load tests need a custom header, for example as a marker by which the gateway recognizes the test emails or for a rule to take effect. `smtp-source` has no option for this, but `-F` reads a fully preformatted message from a file, where any desired header can be included. The file consists of header lines, a blank line and the body, with all lines terminated by `\r\n`:

```bash
{ printf 'X-Lasttest: aktiv\r\n'
  printf 'Subject: Lasttest\r\n'
  printf '\r\n'
  head -c 5120 /dev/zero | tr '\0' 'x'
  printf '\r\n'; } > lasttest.eml
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `head -c 5120` | Outputs the first 5120 bytes of input, here from `/dev/zero` |
| `tr '\0' 'x'` | Replaces every null byte with the character `x` and thus generates the 5 KB filler text |
| `> lasttest.eml` | Writes the assembled message to the file for `-F` |

</details>

```bash
smtp-source -c -d -N -s 8 -m 50000 -F lasttest.eml \
  -f lasttest@example.com \
  -t test@example.com \
  gateway.example.com:25
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-F datei` | Sends headers and body unchanged from the file; replaces the generated message content |

</details>

Two consequences: `-F` overrides `-l` and `-S`, because the size and subject now come from the file (which is why both must be included there). `-N` remains effective, however: Recipients continue to be numbered; the header is identical in all messages because it comes from the fixed file.

## How many sessions?

The most reliable way to determine the appropriate number of sessions is through measurement, using exactly the same options as for the planned main run: the same message source (the same `-F` file or the same `-l`), same sender, same target. Only the quantity is reduced to 2,000 per stage, while `-s` varies. A short calibration run with increasing session counts shows when additional sessions no longer help:

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

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `date +%s%N` | Outputs Unix seconds immediately followed by the nanosecond portion as one number |
| `-d` | Connections remain open across all messages in the stage |
| `-N` | Recipient numbering via the per-process counter |
| `-s "$s"` | Number of parallel sessions, 1 through 32 in each loop iteration |
| `-m 2000` | 2,000 messages per measurement stage |
| `-F lasttest.eml` | The same message file as in the planned main run |
| `-f` | Sender address |
| `-t '@blackhole.example.com'` | Base recipient address with an empty local part on a discard domain |
| `gateway.example.com:25` | Target host and port |

</details>

Two details of the invocation: `-c` is deliberately omitted here, so no running counter output appears between measurement lines; the loop produces exactly one result line per stage. The empty local part in `-t` also works well with numbering on a discard domain: With the prefixed counter in Postfix 3.5, it produces purely numeric recipient addresses (`1@blackhole.example.com`, `2@…`), which keeps log evaluation clear.

Specifically, the following happens: The outer loop runs through session counts from 1 to 32 in doubling steps. Before and after each run, `date +%s%N` records the current time as one large number, namely Unix seconds immediately followed by the nanosecond portion. In between, `smtp-source` sends 2,000 messages (content, headers and size come from the `-F` file) over the respective number of parallel connections, which remain open thanks to `-d`; the loop waits until the invocation is completely finished. The `echo` line converts the time difference into a rate: 2,000 emails divided by the runtime in seconds, where the runtime is in nanoseconds. This turns 2,000 times 10⁹ into the constant `2000000000000`. The Bash arithmetic `$(( ))` performs integer calculations and truncates decimal places, which is sufficiently accurate for this measurement.

Three practical notes: `%N` provides nanoseconds only with GNU date (the case on RHEL and most Linux systems; BusyBox and macOS do not support it). The complete run sends 6 × 2,000 = 12,000 emails; these also need a controlled recipient address, and `-N` numbering starts over at the initial value with every invocation. If an `smtp-source` invocation aborts with an error message, the rate on that line is meaningless; resolve the cause first, then measure again.

The expected output is one line per stage. With invented but typical example values, it looks like this:

```text
1 Sessions: 11 Mails/s
2 Sessions: 21 Mails/s
4 Sessions: 40 Mails/s
8 Sessions: 71 Mails/s
16 Sessions: 79 Mails/s
32 Sessions: 80 Mails/s
```

The interpretation: As long as the rate roughly doubles with the session count, the parallel sessions cover the wait time for responses from the target; the bottleneck is then path latency, not capacity. From the point at which the curve flattens (between 8 and 16 sessions in the example), either the target system is saturated or the source has reached its limit. Choose the smallest value at which the rate no longer increases significantly, 8 to 16 in the example; additional sessions then increase only the load through parallelism, not throughput. For the main run, the measured rate can also be used to estimate the expected duration: the total count from `-m` divided by the rate.

## Evaluation on the receiving side

If a dedicated test receiver is available on the target system, `smtp-sink` also handles the logging:

```bash
smtp-sink -c -d "mails/%Y%m%d-%H%M%S." 0.0.0.0:2525 200
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-c` | Running counters instead of the full SMTP dialogue |
| `-d "mails/…"` | For the sink: dump, not connection persistence. Writes each accepted message to a separate file (name pattern via strftime), including an `X-Rcpt-Args` header containing the recipient address |
| `0.0.0.0:2525` | Listens on all interfaces on port 2525 |
| `200` | Backlog: maximum queue length of pending connections according to listen(2) |

</details>

After the run, extract the received numbers and compare them with the expected set. Since the numbers have no leading zeros, both lists are padded to a fixed width before comparison so that the alphabetical sorting of `comm` matches numeric sorting. The search pattern matches the Postfix 3.5 address format (number before the address); for current versions, use `test[0-9]+@` and `seq` accordingly, starting at 0:

```bash
grep -rhoE '[0-9]+test@example\.com' mails/ | \
  grep -oE '^[0-9]+' | sort -u | \
  awk '{printf "%08d\n", $1}' | sort > empfangen.txt
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `grep -r` | Searches the `mails/` directory recursively |
| `grep -h` | Suppresses file names before matches |
| `grep -o` | Outputs only the matching address part, not the entire line |
| `grep -E` | Extended regular expressions, here for `[0-9]+` |
| `sort -u` | Sorts and removes duplicates (each number once) |
| `awk '{printf "%08d\n", $1}'` | Pads each number to eight digits with leading zeros |
| `sort` | Sorts the padded numbers for comparison with `comm` |

</details>

```bash
seq 1 50000 | awk '{printf "%08d\n", $1}' | \
  comm -23 - empfangen.txt
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `seq 1 50000` | Generates the expected set of numbers; the end value corresponds to the sent total from `-m` |
| `comm -23` | Suppresses column 2 (only in file 2) and column 3 (in both); what remains are lines that exist only in the expected set |
| `-` | Reads the first comparison list from the pipe rather than from a file |
| `empfangen.txt` | Second comparison list: the numbers actually received |

</details>

`comm -23` outputs exactly the numbers present in the expected set but not in the received list: the missing emails. Empty output means complete delivery. If numbers appear twice (recognizable from the difference between `sort` and `sort -u`), a system along the route duplicated the message, which is also a finding.

If the target is a production-like system rather than an smtp-sink, its logging takes the role of the dump files. On an Exchange server, for example, `Get-MessageTrackingLog -Recipients` or a filter on the recipient address provides the numbers that arrived; on a Postfix system, use `grep` on `to=` and the base address via the mail log. That is precisely the advantage of the number in the address: The recipient appears in every message tracking log, whereas the subject may be absent depending on the system or first need to be enabled.

## When the number must be in the subject

Some evaluations depend on the subject, for example when the target system rewrites recipient addresses or logs show the recipient only in masked form. Then the loop variant remains: one `smtp-source` invocation per email with `-m 1` and a subject incremented by the shell, distributed across several parallel workers with contiguous number ranges.

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

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-s 1` | One session per invocation; the four workers provide parallelism |
| `-m 1` | Exactly one message per invocation, so the subject can be set per email |
| `-l 5120` | Message size in bytes (excluding headers), 5 KB here |
| `-S "$(printf 'Lasttest %05d' "$i")"` | Subject with the sequential number padded to five digits |
| `-f` / `-t` | Sender and recipient address |
| `gateway.example.com:25` | Target host and port |

</details>

The cost is a complete connection setup per email: TCP handshake, banner, `HELO`, sending, `QUIT`. This run therefore does not measure the target system's maximum throughput, but a deliberately connection-intensive case. Determine the number of workers analogously to the calibration run above, only with the worker loop instead of `-s`. The leading zeros in the subject eliminate the reformatting needed for comparison with the `-N` variant.

## Rules for tests against other systems

As soon as the test leaves your own system, three conditions apply. First: The operator of the target system is informed and has agreed to the time window; a load test looks like an attack or a spam wave to any monitoring system. Second: The recipient address terminates in a controlled location, in a dedicated test mailbox, a discard rule on the target, or a discard domain provided for this purpose by the provider; production addresses do not belong in a load test. Third: An abort criterion is defined before the start, such as a growing queue on the target or an error rate above a threshold, and someone monitors these values during the run.

With these three points and numbering, the run ultimately provides not just a throughput figure but a verifiable statement: which emails arrived, which are missing, and where along the route they were last seen.

## Kilder

1.  [Postfix: smtp-source(1)](https://www.postfix.org/smtp-source.1.html): Man page for the load generator; describes the `-N` behavior of the current version (counter on the local part, plus addressing).

2.  [Postfix source code 3.5.8: smtp-source.c](https://github.com/vdukhovni/postfix/blob/v3.5.8/postfix/src/smtpstone/smtp-source.c): documents for the RHEL 8 version that the number is prefixed (`RCPT TO:<%d%s>`) with a starting value of 1; in the [current version](https://github.com/vdukhovni/postfix/blob/master/postfix/src/smtpstone/smtp-source.c), the number is instead appended to the local part, starting at 0.

3.  [Postfix: smtp-sink(1)](https://www.postfix.org/smtp-sink.1.html): Man page for the test receiver with dump options and recorded X headers.

4.  [GNU Coreutils: comm](https://www.gnu.org/software/coreutils/manual/html_node/comm-invocation.html): Comparison of two sorted lists, used here to reconcile expected and received numbers.
