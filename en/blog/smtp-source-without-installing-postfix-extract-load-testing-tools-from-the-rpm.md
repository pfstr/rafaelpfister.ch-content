---
title: "smtp-source without installing Postfix: Extract load-testing tools from the RPM"
navTitle: "Extract smtp-source"
description: "smtp-source and smtp-sink are part of Postfix, but also run without an installed mail server. Learn how to extract the two tools from the package on RHEL, why running them from /tmp can fail due to the noexec mount option, and which libraries must be included."
date: "2026-08-27"
kategorie: "SMTP and mail flow"
timeToRead: "7 min read"
themen:
  - smtp-mailflow
  - smtp-lasttests
produkte:
  - "uebergreifend"
protokolle:
  - "smtp"
  - "testing"
  - "troubleshooting"
slug: "smtp-source-without-installing-postfix-extract-load-testing-tools-from-the-rpm"
translationId: "article-d0a27da11509d24b"
translationOf: smtp-source-ohne-postfix-installation
url: https://rafaelpfister.ch/en/blog/smtp-source-without-installing-postfix-extract-load-testing-tools-from-the-rpm
translationSourceHash: 2b4bda3ea22f49c9d5269ec15b0c1dbfd779ccc6d03ad5b234aba738e5bb119f
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:23:15.970Z
translationReview: automatic
---

# smtp-source without installing Postfix: Extract load-testing tools from the RPM

For SMTP load tests, `smtp-source` is a good choice: the tool opens parallel sessions, keeps them open across multiple messages, and therefore models the connection behavior of a bulk sender much more realistically than test tools that establish a new connection for each email. Its counterpart, `smtp-sink`, accepts emails and discards them without delivering anything. Both are included with Postfix.

That is exactly where the problem lies: Postfix is often not installed on the system from which you want to test. Installation is also undesirable on a mail gateway appliance, because an additional Postfix installation brings its own configuration under `/etc/postfix` and a system service that, in the worst case, occupies port 25 and blocks the actual mail system. There is also the question of what vendor support thinks of subsequently installed packages on its appliance.

However, both tools can also be used without installation: download the RPM, extract the binaries and libraries, and you are done. The path there has two particularities, which this article demonstrates using an RHEL 8 system. You do not need root privileges, only access to the package sources.

## Is smtp-source already available?

First, check whether the tool may already be on the system after all. Depending on the distribution, `smtp-source` is located outside the normal PATH:

```bash
command -v smtp-source || \
  ls /usr/sbin/smtp-source /usr/lib/postfix/sbin/smtp-source 2>/dev/null
```

If the output remains empty, the associated package is also missing. On RPM systems, confirm this and check whether the repositories offer Postfix at the same time:

```bash
rpm -qa | grep -i postfix
```

```bash
yum list available postfix
```

No Postfix was installed on the test system, but the BaseOS repository offered `postfix-3.5.8-8.el8_10` . This clears the way: the package can be downloaded without installing it.

## Download the RPM only

`yum download` (from the `dnf-plugins-core` plugin package, usually available on RHEL 8) downloads a package to the current directory without installing it. This works without root privileges as long as the target directory is writable:

```bash
cd /tmp && yum download postfix
```

If yum reports `No such command: download`, the plugin is missing. With root privileges, you can achieve the same result using the installation command with `--downloadonly`:

```bash
sudo yum install --downloadonly --downloaddir=/tmp postfix
```

If neither is available, the remaining workaround is a second system with the same RHEL version: download the RPM there and copy it to the target system with `scp`.

## Extract binaries and libraries

`rpm2cpio` converts the RPM into a cpio archive stream, from which `cpio` extracts selected paths. In addition to the two binaries, you also need the Postfix libraries, because on RHEL the tools are dynamically linked against `libpostfix-*.so`:

```bash
cd /tmp && rpm2cpio postfix-*.rpm | \
  cpio -idmv ./usr/sbin/smtp-source ./usr/sbin/smtp-sink \
  './usr/lib64/postfix/*'
```

The files are then located below `/tmp/usr/`. The path specifications begin with `./` because cpio expects paths exactly as they appear in the archive.

## Problem 1: /tmp is mounted with noexec

The obvious attempt to start directly from /tmp fails on hardened systems:

```text
bash: /tmp/usr/sbin/smtp-sink: Permission denied
[1]+  Exit 126
```

Exit code 126 despite the execute bit being set correctly is the typical symptom of a filesystem mounted with the `noexec` option. The kernel then denies any program execution from that filesystem, regardless of file permissions. You can check this directly:

```bash
mount | grep ' /tmp '
```

The solution: copy the binaries and libraries to a directory whose filesystem permits execution, for example your own home directory:

```bash
mkdir -p ~/bin && \
  cp /tmp/usr/sbin/smtp-source /tmp/usr/sbin/smtp-sink \
     /tmp/usr/lib64/postfix/libpostfix-*.so ~/bin/ && \
  chmod +x ~/bin/smtp-source ~/bin/smtp-sink
```

Note that `noexec` also affects loading shared libraries. Therefore, it is not enough to move only the binaries and leave the libraries in /tmp.

## Problem 2: the library path

Without further configuration, the dynamic linker looks for the Postfix libraries under `/usr/lib64/postfix`, where they are absent because Postfix is not installed:

```text
smtp-sink: error while loading shared libraries: libpostfix-global.so:
cannot open shared object file: No such file or directory
```

`LD_LIBRARY_PATH` adds your own directory to the linker's search path. Prefix the variable to every invocation:

```bash
LD_LIBRARY_PATH=~/bin ~/bin/smtp-source ...
```

With `ldd ~/bin/smtp-source`, you can check in advance whether all dependencies can be resolved. Apart from the Postfix libraries, the tools depend only on standard system libraries.

## Loopback functionality test

You can verify that everything works without sending a single real email: `smtp-sink` listens as a discard receiver on a high port, while `smtp-source` delivers to it. All traffic remains on localhost.

```bash
LD_LIBRARY_PATH=~/bin ~/bin/smtp-sink -v 127.0.0.1:2525 100 &
```

```bash
LD_LIBRARY_PATH=~/bin ~/bin/smtp-source -s 2 -m 10 -l 5120 \
  -f test@example.com -t test@example.com 127.0.0.1:2525
```

| Option | Effect |
|---|---|
| `-v` (smtp-sink) | Logs every dialog step of accepted connections |
| `127.0.0.1:2525` | smtp-sink listens only on localhost, port 2525 |
| `100` | Backlog: maximum length of the queue of pending connections according to listen(2) |
| `-s 2` | Two parallel SMTP sessions |
| `-m 10` | Ten messages in total, distributed across the sessions |
| `-l 5120` | Message size in bytes (without headers), 5 KB here |
| `-f` / `-t` | Sender and recipient address |

On success, `smtp-source` produces no output, while smtp-sink outputs the complete SMTP dialog for every message, from `HELO` to `QUIT`. Then stop the background process and remove the remnants from /tmp:

```bash
kill %1
```

```bash
rm -rf /tmp/usr /tmp/postfix-*.rpm
```

## Notes for the actual load test

For reliable throughput measurements, the load generator belongs on a separate machine in the same network segment, not on the test subject itself. If `smtp-source` runs on the gateway being tested, the generator and mail system compete for CPU and I/O, and the measurement reflects that competition rather than actual capacity. On the target system itself, the extracted tool is primarily suitable for functionality tests of the rule set and initial plausibility checks.

As soon as the test targets the real port 25, these are real emails that pass through the gateway's rule set and may be delivered depending on the configuration. Therefore, use recipient addresses with controlled destinations: a dedicated test mailbox, a rule that discards the test senders, or a discard domain provided by the provider for this purpose. Production addresses do not belong in a load test.

The procedure described is suitable beyond the two SMTP tools for any command-line program included in a package whose installation is not an option on the target system. The combination of `yum download`, `rpm2cpio`, and an executable directory in the home directory is the same on every RPM system.

## Sources

1.  [Postfix: smtp-source(1)](https://www.postfix.org/smtp-source.1.html): Man page with all parameters of the load generator, including session and message control.

2.  [Postfix: smtp-sink(1)](https://www.postfix.org/smtp-sink.1.html): Man page for the test receiver, including options for artificial delays and error responses.

3.  [Red Hat: How to download a package without installing it](https://access.redhat.com/solutions/10154): documents `yum download` and the `--downloadonly` alternative.

4.  [man7.org: mount(8)](https://man7.org/linux/man-pages/man8/mount.8.html): description of the `noexec` mount option and its effect on program execution.
