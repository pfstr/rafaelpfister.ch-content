---
title: "smtp-source without a Postfix installation: extracting load-testing tools from the RPM"
navTitle: "Extract smtp-source"
description: "smtp-source and smtp-sink are part of Postfix, but also run without an installed mail server. Learn how to extract the two tools from the package on RHEL, why running them from /tmp can fail because of the noexec mount option, and which libraries must be included."
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
translationSourceHash: fd4ad6beb5036817db9b7758653a2b7d015a6adb15d7b4a0b47f94161e34b4e6
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:12:25.001Z
translationReview: automatic
url: https://rafaelpfister.ch/sv/blog/smtp-source-without-installing-postfix-extract-load-testing-tools-from-the-rpm
---

# smtp-source without a Postfix installation: extracting load-testing tools from the RPM

For SMTP load tests, `smtp-source` is a good choice: the tool opens parallel sessions, keeps them open across multiple messages and thus models the connection behaviour of a bulk sender far more realistically than test tools that establish a new connection for each email. Its counterpart, `smtp-sink`, accepts emails and discards them without delivering anything. Both are included with Postfix.

That is precisely where the problem lies: Postfix is often not installed on the system from which you want to test. Installation is also undesirable on a mail gateway appliance, because an additional Postfix installation brings its own configuration under `/etc/postfix` and a system service which, in the worst case, occupies port 25 and blocks the actual mail system. There is also the question of what vendor support thinks of subsequently installed packages on its appliance.

However, both tools can also be used without installation: download the RPM, extract the binaries along with the libraries, done. This approach has two specifics, which this article demonstrates on an RHEL 8 system. You do not need root privileges, only access to the package repositories.

## Is smtp-source already available?

First, check whether the tool might already be present on the system. Depending on the distribution, `smtp-source` is located outside the normal PATH:

```bash
command -v smtp-source || \
  ls /usr/sbin/smtp-source /usr/lib/postfix/sbin/smtp-source 2>/dev/null
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `command -v smtp-source` | Outputs the path if the program is in PATH; otherwise nothing |
| `/usr/sbin/... /usr/lib/postfix/sbin/...` | The usual locations outside PATH (RHEL and Debian/Ubuntu respectively) |
| `2>/dev/null` | Suppresses `ls` error messages for paths that do not exist |

</details>

If the output remains empty, the associated package is also missing. On RPM systems, confirm this and check at the same time whether the repositories offer Postfix:

```bash
rpm -qa | grep -i postfix
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-q` | rpm query mode |
| `-a` | Lists all installed packages |
| `grep -i postfix` | Filters the list without considering case |

</details>

```bash
yum list available postfix
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `list available` | Shows only packages available in the repositories but not installed |
| `postfix` | Limits the output to the package being sought |

</details>

No Postfix was installed on the test system, but the BaseOS repository offered `postfix-3.5.8-8.el8_10`. This clears the way: the package can be downloaded without installing it.

## Download only the RPM

`yum download` (from the plugin package `dnf-plugins-core`, usually available on RHEL 8) downloads a package to the current directory without installing it. This works without root privileges as long as the target directory is writable:

```bash
cd /tmp && yum download postfix
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `cd /tmp` | Changes to a writable directory; `yum download` stores the RPM in the current directory |
| `download` | Subcommand from `dnf-plugins-core`: downloads the package without installing it |
| `postfix` | Name of the package to download |

</details>

If yum reports `No such command: download`, the plugin is missing. With root privileges, you can achieve the same using the installation command with `--downloadonly`:

```bash
sudo yum install --downloadonly --downloaddir=/tmp postfix
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `--downloadonly` | Stops after downloading; nothing is installed |
| `--downloaddir=/tmp` | Target directory for the downloaded RPM |
| `postfix` | Package name |

</details>

If neither is available, the remaining workaround is to use a second system with the same RHEL version: download the RPM there and copy it to the target system using `scp`.

## Extract binaries and libraries

`rpm2cpio` converts the RPM into a cpio archive stream, from which `cpio` selectively extracts individual paths. Besides the two binaries, you also need the Postfix libraries, because on RHEL the tools are dynamically linked against `libpostfix-*.so`:

```bash
cd /tmp && rpm2cpio postfix-*.rpm | \
  cpio -idmv ./usr/sbin/smtp-source ./usr/sbin/smtp-sink \
  './usr/lib64/postfix/*'
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `rpm2cpio postfix-*.rpm` | Converts the RPM into a cpio archive stream on stdout |
| `-i` | cpio extraction mode (copy-in) |
| `-d` | Creates missing directories while extracting |
| `-m` | Preserves file modification times |
| `-v` | Lists every extracted file |
| `./usr/sbin/smtp-source ./usr/sbin/smtp-sink` | The two binaries, with paths exactly as in the archive (including the leading `./`) |
| `'./usr/lib64/postfix/*'` | The Postfix libraries; the pattern is quoted so that cpio evaluates it rather than the shell |

</details>

The files are then located under `/tmp/usr/`.

## Problem 1: /tmp is mounted with noexec

The obvious attempt to run them directly from /tmp fails on hardened systems:

```text
bash: /tmp/usr/sbin/smtp-sink: Permission denied
[1]+  Exit 126
```

Exit code 126 despite a correctly set execute bit is typical for a file system with the `noexec` mount option. The kernel then denies all program execution from that file system, regardless of file permissions. You can check this directly:

```bash
mount | grep ' /tmp '
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `mount` | Without arguments, lists all mounted file systems along with their mount options |
| `' /tmp '` | Search pattern with a space before and after, so that it matches only the `/tmp` mount point and not, for example, `/var/tmp` |

</details>

The solution is to copy the binaries and libraries to a directory whose file system permits execution, such as your own home directory:

```bash
mkdir -p ~/bin && \
  cp /tmp/usr/sbin/smtp-source /tmp/usr/sbin/smtp-sink \
     /tmp/usr/lib64/postfix/libpostfix-*.so ~/bin/ && \
  chmod +x ~/bin/smtp-source ~/bin/smtp-sink
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `mkdir -p ~/bin` | Creates the target directory; no error if it already exists |
| `cp ... ~/bin/` | Copies the two binaries and the `libpostfix-*.so` libraries into the executable directory |
| `chmod +x` | Sets the execute bit on both binaries |

</details>

Note that `noexec` also affects the loading of shared libraries. It is therefore not enough to move only the binaries and leave the libraries in /tmp.

## Problem 2: the library path

Without further instructions, the dynamic linker searches for the Postfix libraries under `/usr/lib64/postfix`, where they are absent because there is no installation:

```text
smtp-sink: error while loading shared libraries: libpostfix-global.so:
cannot open shared object file: No such file or directory
```

`LD_LIBRARY_PATH` adds your own directory to the linker's search path. Prefix the variable to every invocation:

```bash
LD_LIBRARY_PATH=~/bin ~/bin/smtp-source ...
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `LD_LIBRARY_PATH=~/bin` | Adds `~/bin` to the dynamic linker's search path for this one invocation |
| `~/bin/smtp-source` | Invocation using the full path, as `~/bin` need not be in PATH |

</details>

With `ldd ~/bin/smtp-source`, you can check in advance whether all dependencies can be resolved. Apart from the Postfix libraries, the tools depend only on standard system libraries.

## Functional test via loopback

You can verify that everything works without sending a single real email: `smtp-sink` listens as a disposable receiver on a high port, while `smtp-source` delivers to it. All traffic remains on localhost.

```bash
LD_LIBRARY_PATH=~/bin ~/bin/smtp-sink -v 127.0.0.1:2525 100 &
```

```bash
LD_LIBRARY_PATH=~/bin ~/bin/smtp-source -s 2 -m 10 -l 5120 \
  -f test@example.com -t test@example.com 127.0.0.1:2525
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-v` (smtp-sink) | Logs every dialog step of accepted connections |
| `127.0.0.1:2525` | smtp-sink listens only on localhost, port 2525 |
| `100` | Backlog: maximum length of the queue of pending connections according to listen(2) |
| `-s 2` | Two parallel SMTP sessions |
| `-m 10` | Ten messages in total, distributed across the sessions |
| `-l 5120` | Message size in bytes (without headers), 5 KB here |
| `-f` / `-t` | Sender and recipient addresses |

</details>

On success, `smtp-source` produces no output, while smtp-sink outputs the complete SMTP dialog from `HELO` through to `QUIT` for every message. Then stop the background process and remove the remnants from /tmp:

```bash
kill %1
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `%1` | Shell job specification: terminates the first background job, smtp-sink in this case |

</details>

```bash
rm -rf /tmp/usr /tmp/postfix-*.rpm
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-r` | Removes the directory tree recursively |
| `-f` | No prompts and no error for paths that do not exist |
| `/tmp/usr /tmp/postfix-*.rpm` | The extracted tree and the downloaded RPM |

</details>

## Notes for the real load test

For reliable throughput measurements, the load generator belongs on a separate machine in the same network segment, not on the test target itself. If `smtp-source` runs on the gateway being examined, the generator and the mail system compete for CPU and I/O, and the measurement reflects that competition rather than actual capacity. On the target system locally, the extracted tool is primarily suitable for functional tests of the ruleset and initial plausibility checks.

As soon as the test targets the real port 25, these are real emails that pass through the gateway's ruleset and may be delivered depending on the configuration. Therefore, use recipient addresses with controlled endpoints: a dedicated test mailbox, a rule that discards the test senders, or a discard domain provided by the provider for this purpose. Production addresses do not belong in a load test.

The procedure described is suitable beyond the two SMTP tools for any command-line program supplied by a package whose installation on the target system is not an option. The combination of `yum download`, `rpm2cpio` and an executable directory in the home directory is the same on every RPM system.

## Källor

1.  [Postfix: smtp-source(1)](https://www.postfix.org/smtp-source.1.html): Manual page with all load generator parameters, including session and message control.

2.  [Postfix: smtp-sink(1)](https://www.postfix.org/smtp-sink.1.html): Manual page for the test receiver, including options for artificial delays and error responses.

3.  [Red Hat: How to download a package without installing it](https://access.redhat.com/solutions/10154): Documents `yum download` and the `--downloadonly` alternative.

4.  [man7.org: mount(8)](https://man7.org/linux/man-pages/man8/mount.8.html): Description of the `noexec` mount option and its effect on program execution.
