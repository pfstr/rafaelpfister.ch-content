---
title: "Essential Controls for TotemoMail Admins: Stop the Server, Check Queues, and Clean Up in a Controlled Manner"
navTitle: "TotemoMail Controls"
description: "The key controls for operating a totemomail gateway: stop the service via systemd and the Tanuki control script, count queue contents per repository, inspect individual messages, clean up in a controlled manner, and restart the service."
date: "2026-08-28"
kategorie: "TotemoMail"
timeToRead: "9 min read"
themen:
  - totemomail
produkte:
  - "totemomail"
  - "apache-james"
protokolle:
  - "smtp"
  - "troubleshooting"
slug: "essential-controls-for-totemomail-admins-stop-the-server-check-queues-and-clean-up-in-a"
translationId: "article-3a0a526ab6e38a06"
translationOf: totemomail-server-stoppen-queues-bereinigen
url: https://rafaelpfister.ch/en/blog/essential-controls-for-totemomail-admins-stop-the-server-check-queues-and-clean-up-in-a
translationSourceHash: bc887dcd4aa82db7e020247f75b86528f0fa331e1643c28a215a1638587197a6
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:39:16.919Z
translationReview: automatic
---

# Essential Controls for TotemoMail Admins: Stop the Server, Check Queues, and Clean Up in a Controlled Manner

When operating a totemomail gateway (now Kiteworks Email Protection Gateway), four tasks are essential: stopping the service cleanly, recording the queue contents, inspecting individual messages, and cleaning up the queues in a controlled manner before restarting the service.

These steps are needed for planned maintenance as well as incidents, for example when a faulty rule, an unreachable destination, or a load test has filled the queues. This article shows each step with the specific commands, including how to stop the service cleanly in the first place. The processing model behind it (processors, repositories, file formats) is described in the article [Understanding Mail Routing Between totemomail and Exchange Online](/blog/totemomail-m365).

All paths refer to an installation under `/opt/totemomail` with the service user `totemo`. Adjust the paths to your environment.

## How totemomail is started and stopped

Before stopping a service, you should know how it runs. Totemomail involves three layers:

- A **systemd unit** `totemomail.service` as the outermost control layer.
- The **control script** `/opt/totemomail/bin/totemomail`, which invokes the unit when starting and stopping.
- The **Tanuki Java Service Wrapper**: a native `wrapper` process that starts and monitors the actual Java process and can restart it after a crash.

You can inspect this setup on your system without needing permission to read the unit file. `systemctl show` queries the properties directly from systemd and also works when the file under `/etc/systemd/system/` is readable only by root:

```bash
systemctl show totemomail.service -p Type -p User -p ExecStart -p ExecStop \
  -p KillMode -p TimeoutStopUSec --no-pager
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `show totemomail.service` | Shows the runtime properties of the unit as loaded by systemd |
| `-p <Property>` | Limits output to the specified property; can be specified multiple times |
| `--no-pager` | Outputs directly to the console instead of opening a pager such as `less` |

</details>

A typical output looks like this:

```text
Type=oneshot
TimeoutStopUSec=1min 30s
ExecStart={ path=/opt/totemomail/bin/totemomail ; argv[]=/opt/totemomail/bin/totemomail start ; ... }
ExecStop={ path=/opt/totemomail/bin/totemomail ; argv[]=/opt/totemomail/bin/totemomail stop ; ... }
User=totemo
KillMode=control-group
```

This reveals the important properties: `systemctl stop totemomail` calls the control script with the argument `stop`, waits up to 90 seconds for a clean shutdown, and then uses `KillMode=control-group` to terminate all remaining processes in the unit. Stopping through systemd is therefore equivalent to calling the script directly, but it also performs cleanup if the script hangs.

The `active (exited)` status in `systemctl status totemomail` is normal in this setup and not an error: the unit is `Type=oneshot`, the start script exits after startup, and the wrapper continues running as an independent daemon managed by systemd only indirectly. Therefore, the unit status does not show whether the service is actually running; the process list does:

```bash
ps -ef | grep -E 'wrapper|TotemoBootStrapper' | grep -v grep
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-e` | Shows all processes, not only those from the current session |
| `-f` | Full output format with the complete command line |
| `grep -E 'wrapper\|TotemoBootStrapper'` | Filters for the wrapper process and the Java main class |
| `grep -v grep` | Removes the grep processes themselves from the result list |

</details>

Under normal operation, two processes appear: the native `wrapper` (started with `../conf/wrapper.conf` and the PID file `totemomail.pid`) and the Java process with the main class `ch.totemo.bootstrapper.TotemoBootStrapper`. If either one is missing, the service has not started completely.

## Step 1: Stop the service

Stop the service first before performing any work on the queues. As long as totemomail is running, it accepts messages, processes the queues, and delivers mail; only stopping it freezes the inventory for analysis.

```bash
sudo systemctl stop totemomail
```

Then verify that the wrapper and Java processes have stopped:

```bash
ps -ef | grep -E 'wrapper|TotemoBootStrapper' | grep -v grep
```

The output must be empty. The PID file `/opt/totemomail/bin/totemomail.pid` also disappears. If a process remains after the stop timeout expires, systemd terminates it through the control group; in that case, check `journalctl -u totemomail` before continuing.

Do not forget the upstream layer: while the service is stopped, newly arriving messages accumulate at the sending system, for example in the Exchange queue or at the upstream relay. This is intended. Reputable senders automatically retry delivery after the service restarts.

## Step 2: Record queue contents

Totemomail queues are file-based mail repositories of the underlying Apache James. They are located below the James application directory, here `/opt/totemomail/mailer/apps/james/var/mail/`. Each subdirectory is a repository, and each message consists of two files: `*.FileStreamStore` contains the complete MIME message, while `*.FileObjectStore` contains the serialized status object with metadata.

To get an overview of the contents, count the `FileObjectStore` files per directory:

```bash
for d in /opt/totemomail/mailer/apps/james/var/mail/*/; do \
  printf '%-22s %s\n' "$(basename "$d")" \
  "$(find "$d" -maxdepth 1 -name '*.FileObjectStore' | wc -l)"; \
done
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `for d in .../*/` | Iterates through all repository directories (the trailing `/` limits it to directories) |
| `printf '%-22s %s\n'` | Formats output in two columns; `%-22s` pads the name to 22 characters, left-aligned |
| `basename "$d"` | Reduces the full path to the directory name |
| `find "$d" -maxdepth 1` | Searches only directly in the directory, without subdirectories |
| `-name '*.FileObjectStore'` | Counts one file per message; the stream counterpart would double the number |
| `wc -l` | Counts the files found |

</details>

The result is one line per queue with the number of messages, for example:

```text
DBUnavailable          0
error                  12
incoming               121
outgoing               0
spool                  0
```

The standard repositories mean: `spool` contains accepted but not yet processed messages, `incoming` contains messages for internal delivery, `outgoing` contains outgoing messages, `error` contains failed messages, and `DBUnavailable` contains messages parked because a backend could not be reached. Depending on the configuration, additional repositories for special routes may exist; they follow the same file scheme.

If `find` runs from a directory that the service user cannot access (such as another user's home directory after `sudo -u totemo`), the warning `Failed to restore initial working directory` appears for each invocation. It is harmless and disappears after `cd ~`.

## Step 3: Inspect messages

Numbers alone are not enough to make a decision. Before deleting anything, you should know what is in the queues: unwanted messages from an incident, or legitimate email that should be delivered after the service restarts?

The `FileStreamStore` files are unmodified RFC 822 messages. You can therefore read the most important headers directly:

```bash
for f in /opt/totemomail/mailer/apps/james/var/mail/incoming/*.FileStreamStore; do \
  awk 'BEGIN{IGNORECASE=1} /^(From|To|Subject|Date):/{print} /^\r?$/{exit}' "$f"; \
  echo ---; \
done | less
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `BEGIN{IGNORECASE=1}` | Compares header names without regard to case (GNU awk) |
| `/^(From\|To\|Subject\|Date):/{print}` | Outputs only the four relevant header lines |
| `/^\r?$/{exit}` | Stops at the blank line between header and body; the message content is not read |
| `echo ---` | Separator line between messages |
| `less` | Pages through output instead of scrolling for many messages |

</details>

For large inventories, distribution is more informative than the individual view. To show the most frequent senders:

```bash
grep -him1 '^From:' /opt/totemomail/mailer/apps/james/var/mail/incoming/*.FileStreamStore \
  | sort | uniq -c | sort -rn | head
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-h` | Suppresses filenames in the output so identical senders are grouped together |
| `-i` | Ignores case |
| `-m1` | Only the first match per file (the header, not quoted `From:` lines in the body) |
| `sort \| uniq -c` | Groups identical sender lines and counts them |
| `sort -rn \| head` | Sorts by frequency in descending order and shows the ten most frequent |

</details>

If a single sender or subject dominates with hundreds of copies, this indicates a loop or misdirected bulk mailing; those messages are candidates for cleanup. Checking file timestamps (`ls -lt`) further narrows down the period and shows whether older, legitimate messages are mixed in.

## Step 4: Clean up in a controlled manner

Only now should you delete anything, and even now, use an intermediate step: first move the contents to a backup directory instead of deleting them directly. The result for mail operations is the same (the queue is empty), but the step is reversible, and individual legitimate messages can later be restored from the backup or reused as `.eml` files.

```bash
mkdir -p /opt/totemomail/queue-backup-$(date +%F)
mv /opt/totemomail/mailer/apps/james/var/mail/incoming/* \
   /opt/totemomail/queue-backup-$(date +%F)/
```

Important: leave the repository directories themselves in place; only move their contents. Also, a message's stream and object files belong together. Removing only one of them leaves orphaned files that generate log errors at the next startup.

Once the backup has been checked or the contents are unquestionably worthless (for example, load-test messages only), delete all queue contents across all repositories:

```bash
find /opt/totemomail/mailer/apps/james/var/mail/ -mindepth 2 -maxdepth 2 -type f \
  \( -name '*.FileStreamStore' -o -name '*.FileObjectStore' \) -delete
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-mindepth 2 -maxdepth 2` | Matches only files directly within the repository directories, not `var/mail` itself or deeper levels |
| `-type f` | Regular files only; the directories remain intact |
| `\( -name ... -o -name ... \)` | Both file types of a message: stream and status object |
| `-delete` | Deletes matches directly; run it without this option first to review the match list |

</details>

Then run the same count as in step 2: all repositories must show 0.

## Step 5: Restart the service

```bash
sudo systemctl start totemomail
```

Starting calls the control script with `start`, which daemonizes the wrapper; the wrapper then starts the Java process. Verify both through the process list from the first section and check the log files under `/opt/totemomail/bin/`: `wrapper.log` logs the startup of the wrapper and JVM, while `console.log` and `console.err` contain output from the application itself.

Finally, perform a functional test with a single test message through the gateway before releasing normal mail flow again. And if a rule or mail loop had filled the queues, fix the cause before allowing traffic again. Otherwise, recording the queue contents starts all over again.

## Summary

| Step | Command | Verification |
|---|---|---|
| Stop | `sudo systemctl stop totemomail` | `ps` filter empty, PID file gone |
| Count contents | `find` loop over `var/mail/*/` | Count per repository |
| Inspect | `awk` header extract, `grep` sender statistics | Separate unwanted from legitimate messages |
| Clean up | `mv` to backup, then `find ... -delete` | Count shows 0 everywhere |
| Start | `sudo systemctl start totemomail` | Processes, `wrapper.log`, test message |

## Sources

1.  [Apache James Server 2: Provided Mailets](https://james.apache.org/server/2/provided_mailets.html): Documentation of the mailets and repositories on which the totemomail queue structure is based.

2.  [Tanuki Software: Java Service Wrapper](https://wrapper.tanukisoftware.com/doc/english/introduction.html): How the wrapper works, including its startup and monitoring of the totemomail Java process, the PID file, and `wrapper.conf`.

3.  [systemd.service(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html): Meaning of `Type=oneshot`, `ExecStop` and `TimeoutStopSec` for units that call an external control script.

4.  [systemd.kill(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.kill.html): `KillMode=control-group` as a safeguard that terminates unit processes remaining after the stop script.

5.  [RFC 5322: Internet Message Format](https://datatracker.ietf.org/doc/html/rfc5322): Structure of the message headers read when inspecting `FileStreamStore` files.
