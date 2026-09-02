---
title: "Testing SMTP on Linux: From the TCP Connection to Delivered Email"
navTitle: "Test SMTP"
description: "When an appliance stops delivering email, a manual SMTP test is more useful than any log. Learn how to check each layer using built-in tools, what different error patterns mean, and why a load balancer can distort diagnosis."
date: "2026-07-31"
kategorie: "SMTP and Mail Flow"
timeToRead: "10 min read"
themen:
  - smtp-mailflow
  - testing
  - e-mail-verschluesselung
slug: "testing-smtp-on-linux-from-the-tcp-connection-to-delivered-email"
translationId: "article-cb44a92c03a47bc0"
translationOf: smtp-verbindung-testen-linux
translationSourceHash: af2a802f67ec6d294b1507eaf26e25704b938e8760ac6751104ce7258cc2a4b3
translationModel: gpt-5.6-terra
translatedAt: 2026-09-02T08:14:05.453Z
translationReview: automatic
url: https://rafaelpfister.ch/en/blog/testing-smtp-on-linux-from-the-tcp-connection-to-delivered-email
---

# Testing SMTP on Linux: From the TCP Connection to Delivered Email

When a mail gateway suddenly stops delivering anything, the appliance logs often show only the final result: delivery fails, the queue grows, and an error message reports a timeout. A manual test from the command line is the only way to determine the actual cause. SMTP is a plain-text protocol that can be spoken entirely by hand, which makes it a diagnostic tool available everywhere without additional installation.

There is another reason for manual testing: you usually cannot install anything on appliances. No package manager, no root access, no `swaks`. All the following steps therefore work with what is already available on virtually every Linux system.

## Separate the layers

A failed email delivery can fail at five different levels, each producing a different error pattern:

1. **Name resolution:** The target host cannot be translated into an IP address.
2. **TCP connection:** The connection to the port cannot be established or is reset.
3. **SMTP dialog:** The connection is established, but the server rejects the sender, recipient, or content.
4. **Transport encryption:** STARTTLS is unavailable, the certificate is invalid, or the TLS version does not match.
5. **Sender verification:** The email is accepted but discarded by the recipient because of SPF, DKIM, or DMARC.

Diagnosis improves enormously when you test these levels one at a time and in sequence instead of immediately sending a complete test email. A failed end-to-end attempt only tells you that something does not work. Layered testing tells you what.

## Step 1: Name resolution

```bash
getent hosts relay.example.com
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `hosts` | NSS database to query; uses the same sources and order as the system itself, according to `nsswitch.conf` |
| `relay.example.com` | Hostname to resolve |

</details>

If the output remains empty, no name server is reachable on this host, or it does not answer external names. This happens regularly in practice: appliances in isolated zones often receive only an internal resolver that knows exclusively its own zones.

```bash
cat /etc/resolv.conf; grep hosts: /etc/nsswitch.conf
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `/etc/resolv.conf` | File output by `cat` containing the configured name servers |
| `hosts:` | Search pattern for `grep`: the line defining the order of resolution sources (files, DNS) |
| `/etc/nsswitch.conf` | File searched by `grep` containing the NSS configuration |

</details>

If resolution is unavailable, test directly against the IP address in the following steps. This is fully sufficient for diagnosis and cleanly separates the DNS issue from the transport issue. In production, the missing resolution remains a separate finding that needs to be fixed.

## Step 2: Port reachability

For a pure TCP check, bash is sufficient. The pseudo-device `/dev/tcp` opens a connection without requiring `nc` or `telnet` to be installed:

```bash
timeout 10 bash -c 'exec 3<>/dev/tcp/192.0.2.25/25' ; echo "exit=$?"
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `timeout 10` | Stops the following command after 10 seconds and then returns exit code 124 |
| `bash -c '…'` | Runs the command string in bash; required because `/dev/tcp` is a bash feature |
| `exec 3<>/dev/tcp/192.0.2.25/25` | Opens file descriptor 3 for reading and writing as a TCP connection to 192.0.2.25, port 25 |
| `echo "exit=$?"` | Outputs the exit code of the preceding command |

</details>

The exit code is the actual information here:

| exit | Meaning |
|---|---|
| `0` | Connection established; the port is open |
| `124` | Timeout: packets are being dropped, typical of a firewall with a DROP rule |
| `1` | Immediate rejection (RST) or no route |

In practice, the difference between 124 and 1 is the most important indicator of all. A timeout means something along the route is silently dropping traffic, and that is almost always a firewall rule. An immediate RST, by contrast, comes from a system that responds but does not offer the service.

Check both relevant ports right away, as well as an arbitrary other destination, to see whether the host is allowed to establish outbound connections at all:

```bash
for t in "192.0.2.25 25" "192.0.2.25 587" "1.1.1.1 443"; do
  set -- $t
  timeout 8 bash -c "exec 3<>/dev/tcp/$1/$2" 2>/dev/null
  echo "$1:$2 -> exit=$?"
done
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `set -- $t` | Splits the value pair at the space into positional parameters `$1` (IP address) and `$2` (port) |
| `timeout 8` | Stops the connection attempt after 8 seconds (exit code 124) |
| `bash -c "…"` | Runs the `/dev/tcp` connection setup in bash |
| `2>/dev/null` | Suppresses error messages so exactly one result line appears for each destination |

</details>

If the control test also fails, the system has no direct outbound access in general, and traffic must go through an internal relay or proxy. More on why this case is particularly tricky below.

If `/dev/tcp` is unavailable, the shell is not bash. The feature does not exist under `sh`, `ash`, or `ksh`, which is often misinterpreted as an apparent network issue:

```bash
ps -p $$ -o comm= ; echo "BASH_VERSION=${BASH_VERSION:-keine bash}"
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-p $$` | Limits output to the process with the PID of the current shell (`$$`) |
| `-o comm=` | Outputs only the command name; the empty label after `=` suppresses the header line |
| `${BASH_VERSION:-keine bash}` | Outputs the bash version or replacement text if the variable is not set |

</details>

## Step 3: Listen first, do not send

An SMTP server proactively greets you with a `220` banner. The most informative individual test is therefore to open a connection and do nothing:

```bash
{ exec 3<>/dev/tcp/192.0.2.25/25 && timeout 15 cat <&3; echo "[ende exit=$?]"; }
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `exec 3<>/dev/tcp/192.0.2.25/25` | Opens file descriptor 3 as a TCP connection to the target |
| `timeout 15 cat <&3` | Reads and outputs everything the server sends on its own for 15 seconds |
| `echo "[ende exit=$?]"` | Displays the exit code afterward; 124 means nothing else arrived for 15 seconds |

</details>

These few characters distinguish two entirely different situations. If a `220 mail.example.com ESMTP` arrives, the remote side is speaking and any further errors are in the dialog. If nothing arrives, it is not due to a wrongly formulated command on your end, because you did not send one.

The file descriptor remains open in the shell afterward. Close it before starting the next test; otherwise, you may continue working with an old connection that is no longer intact:

```bash
exec 3<&- 3>&-
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `3<&-` | Closes the read side of file descriptor 3 |
| `3>&-` | Closes the write side of file descriptor 3 |

</details>

## Step 4: The SMTP dialog by hand

Once the banner is present, perform the complete dialog. It is important to have a reader running alongside it so you see every response the moment it arrives. A script that sends everything first and reads afterward shows you nothing if the connection breaks in the middle of the dialog:

```bash
{
exec 3<>/dev/tcp/192.0.2.25/25
cat <&3 & R=$!
sleep 1; printf 'EHLO host.example.com\r\n' >&3
sleep 2; printf 'MAIL FROM:<absender@example.com>\r\n' >&3
sleep 2; printf 'RCPT TO:<empfaenger@example.net>\r\n' >&3
sleep 2; printf 'DATA\r\n' >&3
sleep 2; printf 'From: absender@example.com\r\nTo: empfaenger@example.net\r\nSubject: Relay-Test\r\n' >&3
printf 'Date: %s\r\nMessage-ID: <%s@example.com>\r\n\r\nTestnachricht\r\n.\r\n' "$(date -R)" "$(date +%s).$" >&3
sleep 3; printf 'QUIT\r\n' >&3
sleep 2; kill $R 2>/dev/null
}
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `exec 3<>/dev/tcp/192.0.2.25/25` | Opens file descriptor 3 as a TCP connection to the target |
| `cat <&3 & R=$!` | Starts a background reader for file descriptor 3 and stores its PID in `R` |
| `printf '…\r\n' >&3` | Sends an SMTP command with the required CRLF line ending over the connection |
| `sleep n` | Waits the specified number of seconds for the server response before the next command |
| `date -R` | Returns the date in the RFC-compliant format for the `Date:` header |
| `date +%s` | Returns Unix time as a simple unique basis for the Message-ID |
| `kill $R 2>/dev/null` | Terminates the background reader; no error message is shown if it has already ended |

</details>

Two details determine success or failure. SMTP requires CRLF as the line ending, so use `printf` with `\r\n` rather than `echo`. And the period on a line by itself ends the message section; it must be sent as `\r\n.\r\n`.

The expected sequence: `220` when establishing the connection, `250` in response to EHLO, `250 2.1.0` in response to MAIL FROM, `250 2.1.5` in response to RCPT TO, `354` in response to DATA, and finally `250 2.0.0 Ok: queued as <id>`. Note the queue ID. It can be used to trace the message with the operating provider if it never reaches the recipient.

The EHLO name deserves attention: some relays check it against forward and reverse DNS and otherwise respond with `501` or `504`. Use the actual FQDN of the sending system, not its short name.

## Step 5: STARTTLS and the certificate

For an encrypted connection, `openssl s_client` handles the STARTTLS negotiation itself and then passes the channel to standard input:

```bash
openssl s_client -connect 192.0.2.25:25 -starttls smtp -tls1_2 -brief </dev/null
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-connect 192.0.2.25:25` | Target host and port for the connection |
| `-starttls smtp` | Performs the SMTP plain-text dialog first, then switches to TLS via STARTTLS |
| `-tls1_2` | Negotiates TLS 1.2 exclusively |
| `-brief` | Reduces output to a short summary of the negotiated connection |
| `</dev/null` | Closes standard input immediately so `s_client` does not wait interactively after the handshake |

</details>

If you connect by IP address because DNS is unavailable, hostname validation cannot work. The certificate name will not match the numeric address. SNI and the verification name can be set explicitly, without any DNS lookup:

```bash
openssl s_client -connect 192.0.2.25:25 \
  -servername mail.example.com -verify_hostname mail.example.com \
  -starttls smtp -tls1_2 -brief </dev/null
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-servername mail.example.com` | Sets the SNI name in ClientHello, independently of the connection address |
| `-verify_hostname mail.example.com` | Validates the server certificate against this name rather than the numeric address |

</details>

Two error patterns appear regularly here and are often misinterpreted.

**“Didn't find STARTTLS in server response, trying anyway”** means the server did not offer STARTTLS in its EHLO response. `openssl` still sends a TLS ClientHello, the server sees invalid protocol data, and the connection ends with `wrong version number` or `write:errno=32` (EPIPE). Both messages are follow-on errors. The actual information is: no STARTTLS. Use the plain-text dialog from step 4 to check which capabilities the server actually reports.

**No STARTTLS on an internal hop** is often entirely correct. If a load balancer passes the connection through at Layer 4, it does not negotiate TLS; the system behind it does so with the actual destination. Testing in plain text on the internal segment is therefore not a security issue, but simply the architecture.

## Step 6: Python as an alternative

If Python is available, it saves you from manually controlling timing with `sleep`. The standard library is enough; nothing needs to be installed:

```python
#!/usr/bin/env python3
import smtplib, ssl
from email.message import EmailMessage
from email.utils import formatdate, make_msgid

msg = EmailMessage()
msg["From"] = "absender@example.com"
msg["To"] = "empfaenger@example.net"
msg["Subject"] = "Relay-Test"
msg["Date"] = formatdate(localtime=True)
msg["Message-ID"] = make_msgid(domain="example.com")
msg.set_content("Testnachricht\n")

ctx = ssl.create_default_context()
ctx.minimum_version = ssl.TLSVersion.TLSv1_2

s = smtplib.SMTP("192.0.2.25", 25, timeout=30, local_hostname="host.example.com")
s.set_debuglevel(1)
s.ehlo()
if s.has_extn("starttls"):
    s.starttls(context=ctx, server_hostname="mail.example.com")
    s.ehlo()
    print("TLS:", s.sock.version(), s.sock.cipher()[0])
s.send_message(msg)
s.quit()
```

`set_debuglevel(1)` logs the complete dialog, including all response codes, and `smtplib` reads every response synchronously. An interruption appears as `SMTPServerDisconnected` along with the last received line instead of as a silent broken pipe.

Two points often go wrong here: `server_hostname` is mandatory when connecting by IP address, otherwise Python validates the certificate against the numeric address. And if you deliberately disable validation, `check_hostname = False` must come before `verify_mode = ssl.CERT_NONE`, otherwise Python raises a `ValueError`.

## Sender address, SPF, and alignment

A test surprisingly often fails not during transport, but because of the selected sender address. Three points should be checked first.

The sender domain must be an FQDN. An address such as `test@meine-testdomain` without a top-level domain is rejected by many MTAs as early as MAIL FROM with `501` or `553`.

The domain must authorize the sending path in use. A look at the SPF record shows whether the outbound address is covered:

```bash
dig +short TXT example.com | grep spf1
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `+short` | Outputs only record values, without headers and metadata |
| `TXT` | Record type queried |
| `example.com` | Name queried |
| `grep spf1` | Filters the SPF line from multiple TXT records |

</details>

And with DMARC enabled, alignment is decisive. If the record contains `aspf=s`, the domain in the envelope (MAIL FROM) and the domain in the `From:` header must match exactly, not merely be related:

```bash
dig +short TXT _dmarc.example.com
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `+short` | Outputs only record values, without headers and metadata |
| `TXT _dmarc.example.com` | Record type and the DMARC-defined name below the domain |

</details>

With `p=reject`, a test email with improper alignment disappears without comment at the recipient, even though your relay accepted it with `250 queued`. This is the most common cause of messages considered successful on the sending side that nevertheless never arrive.

## When a load balancer is in between

In larger environments, an appliance rarely sends directly to the internet. A virtual server on a load balancer typically accepts the connection, rewrites it to a defined address using source NAT, and only then forwards it externally. This has an unpleasant consequence for diagnosis.

A virtual server operating at Layer 4 acknowledges the TCP handshake immediately, before it has established a connection to the destination itself. If this second connection fails, the client sees a successfully established connection that is reset immediately afterward: `Connection reset by peer`, without any SMTP banner. The issue is then neither on your side nor at the destination, but in the pool behind the virtual server—for example, because a member is marked down or the configured FQDN cannot be resolved.

This also explains why a test directly against the internet destination must fail if the forwarding rule accepts only traffic from the already rewritten SNAT address. Connections with the original source address match no rule and are dropped. In such environments, always test against the intended virtual server, not the actual destination.

A single line tells you which source address your system uses for a particular destination. The value after `src` is exactly the information the network team needs for the allowlisting request:

```bash
ip route get 192.0.2.25
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `route get` | Asks the kernel which route it would choose for a specific destination |
| `192.0.2.25` | Destination address of the simulated connection |

</details>

If the system is behind NAT, the remote side does not see this address, but rather the perimeter's public address. It cannot be determined from inside as long as no traffic gets through; it is listed in the NAT rule.

## Error patterns at a glance

| Observation | Likely cause |
|---|---|
| `Name or service not known` | No name resolution on the host |
| Timeout, exit 124 | Firewall silently drops traffic (DROP) |
| `Connection refused` | No service on the port or a REJECT rule |
| Connection established, no banner, then RST | Load balancer accepts, backend unreachable |
| `Didn't find STARTTLS` | Server does not offer transport encryption |
| `wrong version number`, `errno=32` | Follow-on errors after forced TLS without STARTTLS |
| `501` / `553` at MAIL FROM | Sender domain is not an FQDN or is not permitted |
| `554 relay access denied` | Source IP is not allowlisted on the relay |
| `250 queued`, but no delivery | SPF, DKIM, or DMARC alignment at the recipient |

## Load tests and rate limits

For volume tests, one rule applies that is often overlooked in day-to-day work: the number of connections is the issue, not the number of messages. Typical relays permit a few hundred connections per minute but tens of thousands of messages. Therefore, keep a session open and send many envelopes over it instead of reconnecting for every message.

In `smtplib`, this simply means using the same connection object multiple times and deliberately reestablishing the session after a fixed number of messages. By contrast, anyone who opens a new connection for each email exceeds the connection limit long before the message limit and triggers rejections that look like an issue with the remote side.

## Conclusion

The manual SMTP test is not a workaround for environments without tools, but the most precise diagnostic available in email operations. It cleanly separates name resolution, reachability, protocol dialog, and encryption, and provides a clear result for every level. Those who first only listen, then perform the dialog by hand, and take the response codes seriously can reach a conclusion within minutes that supports a ticket with the network or provider team: source address, destination port, observed behavior, and exit code.

## Sources

1.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): Defines the SMTP dialog, command sequence, and meaning of response codes.

2.  [RFC 3207: SMTP Service Extension for Secure SMTP over TLS](https://www.rfc-editor.org/rfc/rfc3207): Describes STARTTLS as an extension, including behavior when the server does not offer it.

3.  [RFC 7208: Sender Policy Framework (SPF)](https://www.rfc-editor.org/rfc/rfc7208): Structure and evaluation of the SPF record for authorizing sending systems.

4.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc7489): Defines alignment between envelope and header senders and policy evaluation.

5.  [OpenSSL: s_client](https://docs.openssl.org/master/man1/openssl-s_client/): Reference for the options used, including `-starttls`, `-servername` and `-verify_hostname`.

6.  [Python documentation: smtplib](https://docs.python.org/3/library/smtplib.html): Standard library for SMTP sessions, including STARTTLS and debug output.

7.  [Bash Reference Manual: Redirections](https://www.gnu.org/software/bash/manual/bash.html#Redirections): Documents `/dev/tcp` as a bash-specific pseudo-device for network connections.
