---
title: "Testing SMTP on Linux: From the TCP Connection to Delivered Email"
navTitle: "Test SMTP"
description: "When an appliance stops delivering email, a manual SMTP test is more useful than any log. Learn how to check layer by layer using built-in tools, what different error patterns mean, and why a load balancer can distort diagnostics."
date: "2026-07-31"
kategorie: "SMTP and Mail Flow"
timeToRead: "10 min read"
themen:
  - smtp-mailflow
  - e-mail-verschluesselung
slug: "testing-smtp-on-linux-from-the-tcp-connection-to-delivered-email"
translationId: "article-cb44a92c03a47bc0"
translationOf: smtp-verbindung-testen-linux
url: https://rafaelpfister.ch/en/blog/testing-smtp-on-linux-from-the-tcp-connection-to-delivered-email
translationSourceHash: 650b4717ca00ffd3d02cebae8f1484027cf0f9b47de1b607caa951cd7264454a
translationModel: gpt-5.6-terra
translatedAt: 2026-08-01T06:11:53.422Z
translationReview: automatic
---

# Testing SMTP on Linux: From the TCP Connection to Delivered Email

When a mail gateway suddenly stops delivering anything, the appliance logs often provide only the final stage of the story: a delivery fails, the queue grows, and an error message mentions a timeout. A manual command-line test is the only way to reveal the actual cause. SMTP is a plain-text protocol that can be spoken entirely by hand, which makes it one of the most useful diagnostic tools in mail operations.

The second reason for manual testing: You usually cannot install anything on appliances. No package manager, no root privileges, no `swaks`. That is why all the following steps work with what is already available on virtually every Linux system.

## Separate the layers

A failed email delivery can fail at five different levels, each producing a different error pattern:

1. **Name resolution:** The target host cannot be translated into an IP address.
2. **TCP connection:** The connection to the port cannot be established or is reset.
3. **SMTP dialog:** The connection is established, but the server rejects the sender, recipient, or content.
4. **Transport encryption:** STARTTLS is missing, the certificate is invalid, or the TLS version does not match.
5. **Sender verification:** The email is accepted but rejected by the recipient because of SPF, DKIM, or DMARC.

Diagnostics improve dramatically when you test these layers individually and in sequence instead of sending a complete test email right away. A failed overall attempt only tells you that something does not work. Layer-by-layer testing tells you what.

## Step 1: Name resolution

```bash
getent hosts relay.example.com
```

If the output remains empty, no name server is reachable from this host, or it does not answer external names. This is more common than you might think: appliances in isolated zones often receive only an internal resolver that knows exclusively its own zones.

```bash
cat /etc/resolv.conf; grep hosts: /etc/nsswitch.conf
```

If resolution is unavailable, test directly against the IP address in the following steps. That is entirely sufficient for diagnostics and cleanly separates the DNS issue from the transport issue. In production, missing resolution is naturally still a separate finding that needs to be fixed.

## Step 2: Port reachability

Bash is sufficient for a basic TCP check. The pseudo-device `/dev/tcp` opens a connection without requiring `nc` or `telnet` to be installed:

```bash
timeout 10 bash -c 'exec 3<>/dev/tcp/192.0.2.25/25' ; echo "exit=$?"
```

The exit code is the actual information here:

| exit | Meaning |
|---|---|
| `0` | Connection established, port is open |
| `124` | Timeout: packets are being dropped, typical of a firewall with a DROP rule |
| `1` | Immediate rejection (RST) or no route |

In practice, the difference between 124 and 1 is the most important clue of all. A timeout means that something along the path is silently dropping traffic, and that is almost always a firewall rule. An immediate RST, on the other hand, comes from a system that responds but does not offer the service.

Check both relevant ports right away, plus any other target to determine whether the host is permitted to establish outbound connections at all:

```bash
for t in "192.0.2.25 25" "192.0.2.25 587" "1.1.1.1 443"; do set -- $t; timeout 8 bash -c "exec 3<>/dev/tcp/$1/$2" 2>/dev/null; echo "$1:$2 -> exit=$?"; done
```

If the comparison test also goes nowhere, the system generally has no direct outbound access and traffic must go through an internal relay or proxy. More on why this case is especially tricky below.

If `/dev/tcp` is unavailable, the shell is not bash. The feature does not exist in `sh`, `ash`, or `ksh`, which is often misinterpreted as an apparent network issue:

```bash
ps -p $$ -o comm= ; echo "BASH_VERSION=${BASH_VERSION:-keine bash}"
```

## Step 3: Listen first, do not send

An SMTP server greets you on its own with a `220` banner. The most informative individual test is therefore to open a connection and do nothing:

```bash
{ exec 3<>/dev/tcp/192.0.2.25/25 && timeout 15 cat <&3; echo "[ende exit=$?]"; }
```

These few characters distinguish two completely different situations. If you receive a `220 mail.example.com ESMTP`, the remote end is speaking and all subsequent errors are in the dialog. If nothing arrives, it is not due to a wrongly formulated command on your end, because you have not sent one.

The file descriptor remains open in the shell afterward. Close it before starting the next test; otherwise, you may continue working with an old, half-dead connection:

```bash
exec 3<&- 3>&-
```

## Step 4: The SMTP dialog by hand

Once the banner is available, run the full dialog. A concurrent read process is important so you can see every response the moment it arrives. A script that sends everything first and reads afterward shows you nothing if the dialog breaks off midway:

```bash
{
exec 3<>/dev/tcp/192.0.2.25/25
cat <&3 & R=$!
sleep 1; printf 'EHLO host.example.com\r\n' >&3
sleep 2; printf 'MAIL FROM:<absender@example.com>\r\n' >&3
sleep 2; printf 'RCPT TO:<empfaenger@example.net>\r\n' >&3
sleep 2; printf 'DATA\r\n' >&3
sleep 2; printf 'From: absender@example.com\r\nTo: empfaenger@example.net\r\nSubject: Relay-Test\r\nDate: %s\r\nMessage-ID: <%s@example.com>\r\n\r\nTestnachricht\r\n.\r\n' "$(date -R)" "$(date +%s).$$" >&3
sleep 3; printf 'QUIT\r\n' >&3
sleep 2; kill $R 2>/dev/null
}
```

Two details determine success or frustration. SMTP requires CRLF as the line ending, so use `printf` with `\r\n`, not `echo`. And the period on a line by itself ends the message body; it must be sent as `\r\n.\r\n`.

The expected sequence: `220` when establishing the connection, `250` in response to EHLO, `250 2.1.0` for MAIL FROM, `250 2.1.5` for RCPT TO, `354` for DATA, and finally `250 2.0.0 Ok: queued as <id>`. Note the queue ID. The operating provider can use it to trace the message if it never arrives at the recipient.

The EHLO name deserves attention: Some relays check it against forward and reverse DNS and otherwise respond with `501` or `504`. Use the actual FQDN of the sending system, not the short name.

## Step 5: STARTTLS and the certificate

For an encrypted connection, `openssl s_client` performs the STARTTLS negotiation itself and then passes the channel to standard input:

```bash
openssl s_client -connect 192.0.2.25:25 -starttls smtp -tls1_2 -brief </dev/null
```

If you connect via the IP address because DNS is unavailable, hostname validation cannot work. The certificate name then does not match the numeric address. SNI and the verification name can be specified explicitly, without any DNS lookup:

```bash
openssl s_client -connect 192.0.2.25:25 -servername mail.example.com -verify_hostname mail.example.com -starttls smtp -tls1_2 -brief </dev/null
```

Two error patterns occur regularly here and are often misinterpreted.

**“Didn't find STARTTLS in server response, trying anyway”** means the server did not offer STARTTLS in its EHLO response. `openssl` still sends a TLS ClientHello, the server sees protocol garbage, and the connection ends with `wrong version number` or `write:errno=32` (EPIPE). Both messages are follow-on errors. The actual information is: no STARTTLS. Use the plain-text dialog from Step 4 to check which capabilities the server actually reports.

**No STARTTLS on an internal hop** is often entirely correct. When a load balancer forwards the connection at Layer 4, it does not negotiate TLS; the system behind it does, with the actual destination. Testing in plain text on the internal segment is then not a security issue but simply the architecture.

## Step 6: Python as an alternative

If Python is available, you can avoid timing workarounds with `sleep`. The standard library is sufficient; nothing needs to be installed:

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

`set_debuglevel(1)` logs the complete dialog including all response codes, and `smtplib` reads every response synchronously. An interruption appears as `SMTPServerDisconnected` along with the last received line, instead of as a silent broken pipe.

Two pitfalls: `server_hostname` is mandatory when connecting via an IP address; otherwise, Python verifies the certificate against the numeric address. And if you deliberately disable verification, `check_hostname = False` must come before `verify_mode = ssl.CERT_NONE`, or Python raises a `ValueError`.

## Sender address, SPF, and alignment

A test surprisingly often fails not at the transport layer but because of the selected sender address. Three points should be checked in advance.

The sender domain must be an FQDN. An address such as `test@meine-testdomain` without a top-level domain is rejected by many MTAs during MAIL FROM with `501` or `553`.

The domain must authorize the sending path being used. A look at the SPF record shows whether the outbound address is covered:

```bash
dig +short TXT example.com | grep spf1
```

And when DMARC is active, alignment is decisive. If the record contains `aspf=s`, the domain in the envelope (MAIL FROM) and the domain in the `From:` header must match exactly, not merely be related:

```bash
dig +short TXT _dmarc.example.com
```

With `p=reject`, a test email with unsuitable alignment disappears at the recipient without comment, even though your relay accepted it with `250 queued`. This is the most common cause of messages that are considered successful on the sending side but still never arrive.

## When a load balancer is in between

In larger environments, an appliance rarely sends directly to the internet. A virtual server on a load balancer commonly accepts the connection, rewrites it to a defined address using source NAT, and only then forwards it externally. This has an unpleasant consequence for diagnostics.

A virtual server operating at Layer 4 acknowledges the TCP handshake immediately, before it has established its own connection to the destination. If that second connection fails, the client sees a successfully established connection that is immediately reset afterward: `Connection reset by peer`, without any SMTP banner. The issue is then neither on your side nor at the destination, but in the pool behind the virtual server—for example, because a member is marked down or the configured FQDN cannot be resolved.

This also explains why a direct test against the internet destination must fail if the forwarding rule accepts only traffic from the already rewritten SNAT address. Connections with the original source address match no rule and are dropped. In such environments, always test against the intended virtual server, not the actual destination.

A single line tells you which source address your system uses for a specific destination. The value after `src` is exactly the information the network team needs for the allowlist:

```bash
ip route get 192.0.2.25
```

If the system is behind NAT, the remote end does not see this address but the perimeter's public address. It cannot be determined from inside as long as no traffic gets through; it is specified in the NAT rule.

## Error patterns at a glance

| Observation | Likely cause |
|---|---|
| `Name or service not known` | No name resolution on the host |
| Timeout, exit 124 | Firewall silently drops traffic (DROP) |
| `Connection refused` | No service on the port or a REJECT rule |
| Connection established, no banner, then RST | Load balancer accepts, backend unreachable |
| `Didn't find STARTTLS` | Server does not offer transport encryption |
| `wrong version number`, `errno=32` | Follow-on errors after forcing TLS without STARTTLS |
| `501` / `553` during MAIL FROM | Sender domain is not an FQDN or is not permitted |
| `554 relay access denied` | Source IP is not allowlisted on the relay |
| `250 queued`, but no delivery | SPF, DKIM, or DMARC alignment at the recipient |

## Load tests and rate limits

For volume tests, one rule that is often overlooked in daily operations applies: The number of connections, not the number of messages, is the issue. Typical relays allow several hundred connections per minute but tens of thousands of messages. Therefore, keep one session open and send many envelopes through it instead of reconnecting for every message.

In `smtplib`, this simply means using the same connection object multiple times and rebuilding the session in a controlled manner after a fixed number of messages. By contrast, anyone opening a new connection for every email hits the connection limit long before the message limit and triggers rejections that look like an issue on the remote side.

## Conclusion

The manual SMTP test is not a fallback for environments without tools; it is the most precise diagnostic available in mail operations. It cleanly separates name resolution, reachability, protocol dialog, and encryption, and provides an unambiguous result at every level. Those who first only listen, then conduct the dialog by hand, and take the response codes seriously can reach a conclusion within minutes that can substantiate a ticket with the network or provider team: source address, destination port, observed behavior, and exit code.

## Sources

1.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): Defines the SMTP dialog, command sequence, and the meaning of response codes.

2.  [RFC 3207: SMTP Service Extension for Secure SMTP over TLS](https://www.rfc-editor.org/rfc/rfc3207): Describes STARTTLS as an extension, including behavior when the server does not offer it.

3.  [RFC 7208: Sender Policy Framework (SPF)](https://www.rfc-editor.org/rfc/rfc7208): Structure and evaluation of the SPF record for authorizing sending systems.

4.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc7489): Governs alignment between envelope and header senders as well as policy evaluation.

5.  [OpenSSL: s_client](https://docs.openssl.org/master/man1/openssl-s_client/): Reference for the options used, including `-starttls`, `-servername` and `-verify_hostname`.

6.  [Python documentation: smtplib](https://docs.python.org/3/library/smtplib.html): Standard library for SMTP sessions, including STARTTLS and debug output.

7.  [Bash Reference Manual: Redirections](https://www.gnu.org/software/bash/manual/bash.html#Redirections): Documents `/dev/tcp` as a bash-specific pseudo-device for network connections.
