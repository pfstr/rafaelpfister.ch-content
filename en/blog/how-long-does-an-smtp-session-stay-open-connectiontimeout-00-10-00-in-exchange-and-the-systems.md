---
title: "How Long Does an SMTP Session Stay Open? ConnectionTimeout 00:10:00 in Exchange and the Systems for Which That Is Too Short"
navTitle: "SMTP Session Duration"
description: "Exchange terminates every incoming SMTP session after ten minutes, even if it is currently transferring data. Which senders stay on a connection that long, how to determine the actual session duration from the protocol log, and when ConnectionTimeout and ConnectionInactivityTimeout should be adjusted on a relay connector."
date: "2026-09-03"
kategorie: "SMTP and Mail Flow"
timeToRead: "10 min read"
themen:
  - smtp-mailflow
  - exchange-onprem-hybrid
produkte:
  - "exchange-on-premises"
  - "exchange"
protokolle:
  - "smtp"
  - "troubleshooting"
slug: "how-long-does-an-smtp-session-stay-open-connectiontimeout-00-10-00-in-exchange-and-the-systems"
translationId: "article-b40497933bbe0a88"
aiPrompt: |
  Du bist mein Exchange- und Mailflow-Assistent. Hilf mir, die SMTP-Session-Dauer auf einem Exchange-Receive-Connector zu beurteilen: 1. Frage mich, welche Systeme (Relays, Gateways, Applikationen, Scanner) über den Connector einliefern und ob sie Verbindungen über mehrere Nachrichten hinweg offen halten. 2. Lass dir die Ausgabe der Session-Auswertung aus dem Protokoll-Log geben (IP, Mails, Dauer, Timeout-Kennzeichen) und erkläre mir, welche Sessions am ConnectionTimeout abgebrochen wurden. 3. Empfiehl pro Connector konkrete Werte für ConnectionTimeout und ConnectionInactivityTimeout und begründe, warum der internetseitige Connector unverändert bleibt. 4. Nenne mir, was ich stattdessen auf der Client-Seite ändern kann, damit die Verbindung nach einer festen Anzahl Nachrichten neu aufgebaut wird.
translationOf: smtp-session-dauer-exchange-connectiontimeout
url: https://rafaelpfister.ch/en/blog/how-long-does-an-smtp-session-stay-open-connectiontimeout-00-10-00-in-exchange-and-the-systems
translationSourceHash: a107c4edd960dabb30ba1b6f263a693808a5edf6815747d81f5d446c103a7e79
translationModel: gpt-5.6-terra
translatedAt: 2026-09-04T08:19:17.264Z
translationReview: required
---

# How Long Does an SMTP Session Stay Open? ConnectionTimeout 00:10:00 in Exchange and the Systems for Which That Is Too Short

In short: An SMTP session has no natural end. RFC 5321 only limits the time spent waiting for the next step, and a client may continue delivering messages over an open connection for as long as the server keeps the connection open. By default, Exchange keeps it open for ten minutes on Receive Connectors, then the server terminates the connection regardless of whether data is currently flowing. This is irrelevant for Exchange-to-Exchange traffic and most MTAs because these senders reconnect on their own after seconds. For applications, gateways, and load generators that use a single connection for an entire sending run, however, this value causes interruptions that appear as connection errors on the client and as `421 4.4.1 Connection timed out` in the Exchange protocol log.

## Two timeouts with different meanings

A Receive Connector has two time limits that are often confused:

| Parameter | Meaning | Default Mailbox server | Default Edge Transport |
|---|---|---|---|
| `ConnectionInactivityTimeout` | maximum idle time without client activity; the connection is then closed | 00:05:00 | 00:01:00 |
| `ConnectionTimeout` | maximum total duration of the connection, even while it is actively transferring data | 00:10:00 | 00:05:00 |

Both values accept 1 second through 1 day (`1.00:00:00`), and `ConnectionTimeout` must be greater than `ConnectionInactivityTimeout`. The values apply per connector, separately for the internet-facing `Default Frontend <Server>`, the Transport service connector `Default <Server>` on port 2525, and every custom relay connector.

The idle timeout is unproblematic: Five minutes exactly matches the minimum time RFC 5321 requires a server to wait for the next command, and a client that sends nothing for five minutes has generally forgotten the connection itself. The total timeout is Exchange-specific: It starts counting when the connection is established and continues while the client submits message after message. After ten minutes, Exchange closes the connection wherever the dialog happens to be, if necessary in the middle of a `DATA` block.

There is no equivalent on the sending side: A Send Connector has only `ConnectionInactivityTimeOut` (ten minutes by default) and instead limits sessions through `SmtpMaxMessagesPerConnection`, 20 messages by default. As a client, Exchange therefore ends every connection after no more than 20 messages and creates a new one. This is why the total timeout is never noticeable between Exchange servers: Sessions last seconds.

## What RFC 5321 requires

Section 4.5.3.2 of the standard defines minimum wait times a client should observe for each protocol step before giving up the connection:

| Step | Minimum client-side timeout |
|---|---|
| Waiting for the `220` banner | 5 minutes |
| Response to `MAIL` | 5 minutes |
| Response to `RCPT` | 5 minutes |
| Response to `DATA` (the `354`) | 2 minutes |
| Sending a data block | 3 minutes |
| Response to the final period | 10 minutes |
| Server: waiting for the next command | at least 5 minutes |

The RFC does not specify an upper limit for the total duration of a session. A client that submits messages over the same connection for 30 minutes without remaining silent for more than a few seconds is standards-compliant. The final client value is notable: Ten minutes of waiting for the response after the final period, because the server accepts and takes over the message during this phase. If the client aborts too soon here, the message has already been delivered and will be delivered a second time on the next attempt. The same situation occurs in reverse if the server closes the connection at this point because of the total timeout.

If a server closes the connection with `421`, the client should treat the transaction in progress under section 3.8 as though it had received a `451`, meaning a temporary error requiring a retry. An MTA with a queue does exactly that. An application without a queue instead reports an exception and leaves the rest to its caller.

## How long senders actually keep their sessions open

The client determines session duration, and the differences between sender types are substantial:

| Sender | Typical session duration | Limited by |
|---|---|---|
| Exchange Send Connector | Seconds | `SmtpMaxMessagesPerConnection` = 20 |
| Postfix with connection cache | maximum 5 minutes | `smtp_connection_reuse_time_limit` = 300s |
| Postfix without connection cache | one message per connection | Default behavior of the `smtp` client |
| Application using `.NET SmtpClient`, `JavaMail Transport`, Python `smtplib` | as long as the object lives: the entire run for a batch job | only program code |
| Quarantine notifications from mail gateways | one session per notification run | Product behavior, partly with `NOOP` keepalive |
| Multifunction devices, scan-to-mail | one message per connection; several minutes for large scans over slow links | File size and bandwidth |
| Load generators such as `smtp-source -d` | until the end of the run | Invocation parameters |

The first two rows explain why nobody notices this value for years in traditional environments: MTAs keep connections short on their own. For example, Postfix uses a cached connection for no more than five minutes before opening a new one, and Exchange disconnects after 20 messages. Both therefore stay below every Exchange default value.

The application row is the most common problem case. A batch job that sends invoices, pay stubs, or system notifications typically creates a client object, calls the send method on it in a loop, and closes it at the end. `System.Net.Mail.SmtpClient` has used the same connection for consecutive `Send` calls since .NET Framework 4 and sends `QUIT` only during `Dispose`; JavaMail behaves the same way with a `Transport` opened once. If the job runs longer than ten minutes, the `421` occurs somewhere in the middle, and the job terminates with an exception, in .NET for example with the text `Service not available, closing transmission channel. The server response was: 4.4.1 Connection timed out`. Which message is affected depends on runtime, making the error seem random: Sometimes it fails after 800 messages, sometimes after 1,200, depending on message size and server load.

The gateway row describes a documented case: Symantec (now Broadcom) Messaging Gateway sends spam quarantine notifications over a single connection and sends `NOOP` as a keepalive between messages. Exchange responds to `NOOP` with the five-second tarpit delay, so at most about 120 notifications can get through in ten minutes before the session ends with `421 4.4.1` and the gateway must reconnect.

The scanner row is a size problem rather than a volume problem: A 60 MB scan over a 2 Mbps connection requires around four minutes of transfer time alone; at 100 MB, it takes almost seven minutes. On an Edge Transport server with a five-minute total timeout, that is already enough to cause an interruption; on a Mailbox server, there is some margin left, but not much.

## What happens when the connection is terminated

When the total timeout expires, Exchange writes the response `421 4.4.1 Connection timed out` to the protocol log, sends it to the client, and closes the connection. For the transaction currently in progress: If the final period has not yet been sent, the message has not been accepted and must be repeated in full. If the period was sent and the connection is closed before the `250` response, the client has no information about whether Exchange took over the message; a properly implemented client retries it, and the recipient may receive it twice. The likelihood is small, but not zero with thousands of messages per run.

The proxy path also needs to be considered: The Front End Transport service accepts the connection on port 25 and forwards it as its own SMTP session to the Transport service on port 2525, where connector `Default <Server>` uses the same default values. A long session therefore appears in both logs, and an adjustment must include both connectors.

## Determining the actual session duration from the protocol log

Before changing a value, it is worth looking at real sessions. The prerequisite is verbose protocol logging on the affected connector; it is already enabled on `Default Frontend <Server>`, but not on all other connectors:

```powershell
Set-ReceiveConnector -Identity 'EX01\Relay Applikationen' -ProtocolLoggingLevel Verbose
```

The logs are located under `%ExchangeInstallPath%TransportRoles\Logs\FrontEnd\ProtocolLog\SmtpReceive` (Front End) and `%ExchangeInstallPath%TransportRoles\Logs\Hub\ProtocolLog\SmtpReceive` (Transport service), named by UTC hour as `RECVyyyyMMddhh-nnnn.log`. Each line is a protocol event with the fields `date-time`, `connector-id`, `session-id`, `sequence-number`, `local-endpoint`, `remote-endpoint`, `event`, `data`, and `context`. All lines in a session have the same `session-id`, so session duration is the difference between the first and last timestamp for that ID.

The following script evaluates the most recent log file of the day for a connector, summarizes lines per session, and displays the 15 longest sessions with the number of messages, duration, and whether Exchange ended them with `421 4.4.1`:

```powershell
$logPfad = Join-Path $env:ExchangeInstallPath 'TransportRoles\Logs\FrontEnd\ProtocolLog\SmtpReceive'
$connector = 'Relay Applikationen'
$tag = (Get-Date).ToUniversalTime().ToString('yyyyMMdd')
$datei = Get-ChildItem $logPfad -Filter "RECV$tag*.log" |
    Sort-Object Name -Descending |
    Select-Object -First 1

$sessions = @{}
Get-Content $datei.FullName |
    Where-Object { $_ -notlike '#*' -and $_ -like "*$connector*" } |
    ForEach-Object {
        $c = $_ -split ','
        $s = $c[2]
        if (-not $sessions[$s]) {
            $sessions[$s] = [pscustomobject]@{
                IP = ($c[5] -split ':')[0]; Start = $c[0]; Ende = $c[0]
                Zeilen = 0; Mails = 0; Timeout = $false
            }
        }
        $sessions[$s].Ende = $c[0]
        $sessions[$s].Zeilen++
        if ($c[7] -like 'MAIL FROM*') { $sessions[$s].Mails++ }
        if ($c[7] -like '421 4.4.1*') { $sessions[$s].Timeout = $true }
    }

$sessions.Values |
    Sort-Object Zeilen -Descending |
    Select-Object -First 15 IP, Mails, Zeilen, Timeout,
        @{ n = 'Dauer_s'
           e = { [math]::Round(([datetime]$_.Ende - [datetime]$_.Start).TotalSeconds, 1) } } |
    Format-Table -AutoSize
```

<details class="options-details">
<summary>Options explained</summary>

| Element | Effect |
|---|---|
| `$logPfad` | Front End Transport service log directory; use `Hub` instead of `FrontEnd` for the Transport service |
| `$connector` | Connector name component; filters on field `connector-id`, logged as `Server\Name` |
| `$tag` | UTC date, because log files are named by UTC hour |
| `-Filter "RECV$tag*.log"` | Only Receive logs from the current day |
| `Sort-Object Name -Descending`, `Select-Object -First 1` | The most recent file (highest hour, highest instance number) |
| `$_ -notlike '#*'` | Skips the header lines `#Software`, `#Version`, `#Log-Type`, `#Date`, `#Fields` |
| `$_ -split ','` | Splits the CSV line; fields 0, 2, 5, and 7 used here precede the first free-text field and are therefore stable |
| `$c[2]` | `session-id`, the grouping key |
| `($c[5] -split ':')[0]` | IPv4 address from `remote-endpoint` (adjust the parsing for IPv6 endpoints) |
| `$c[0]` as `Start` and `Ende` | First and last timestamp of the session; `Ende` is overwritten with every line |
| `$c[7] -like 'MAIL FROM*'` | Counts messages based on the received `MAIL FROM` command |
| `$c[7] -like '421 4.4.1*'` | Marks sessions Exchange terminated because of the total timeout |
| `Sort-Object Zeilen -Descending` | Most active sessions first; alternatively, sort by `Dauer_s` |
| `Dauer_s` | Difference between ISO 8601 timestamps in seconds, rounded to one decimal place |

</details>

In the output, you can identify affected systems by `Timeout` being set to `True` and `Dauer_s` being close to 600: The session lived exactly as long as the connector permits. Sessions with many messages and a duration well below 600 seconds are unproblematic, even if they are currently the longest ones. To see which sources are affected, grouping the marked sessions is sufficient:

```powershell
$sessions.Values |
    Where-Object { $_.Timeout } |
    Group-Object IP |
    Select-Object Name, Count |
    Sort-Object Count -Descending
```

Two limitations of this approach: A session that crosses an hour boundary is split across two log files and appears shortened in an individual file; for a daily analysis, read all files for the day. Also, `Mails` counts `MAIL FROM` commands, meaning attempts rather than accepted messages.

## Adjusting values: on which connector and by how much

The default values protect the internet-facing connector, where arbitrary remote systems can occupy connections. They remain unchanged there; a legitimate external MTA reconnects anyway. Adjust the dedicated connector through which the identified internal systems submit messages. If no such connector exists, create one restricted to the sender IPs with `RemoteIPRanges`; that is better than increasing the value on `Default Frontend`. To view the current state of all connectors:

```powershell
Get-ReceiveConnector |
    Format-Table Name, TransportRole, ConnectionTimeout, ConnectionInactivityTimeout, TarpitInterval -AutoSize
```

The adjustment itself, here with a total duration of one hour and an unchanged idle timeout:

```powershell
$werte = @{
    ConnectionTimeout           = '01:00:00'
    ConnectionInactivityTimeout = '00:05:00'
}
Set-ReceiveConnector -Identity 'EX01\Relay Applikationen' @werte
Set-ReceiveConnector -Identity 'EX01\Default EX01' @werte
```

<details class="options-details">
<summary>Options explained</summary>

| Parameter | Effect |
|---|---|
| `ConnectionTimeout` | Total duration of a connection; permitted from 00:00:01 through 1.00:00:00, must be greater than `ConnectionInactivityTimeout` |
| `ConnectionInactivityTimeout` | Idle time before closing; five minutes matches the RFC minimum and can remain unchanged |
| `-Identity 'EX01\Relay Applikationen'` | The Front End connector for internal senders |
| `-Identity 'EX01\Default EX01'` | The Transport service connector on port 2525 to which the Front End forwards the session |
| `@werte` | Splatting: passes both parameters from the hashtable to the cmdlet |

</details>

The value should exceed the longest legitimate session shown by the analysis, with headroom for load spikes. One hour covers most batch runs; a two-hour overnight run accordingly requires more, up to the maximum of one day. However, the value should not be arbitrarily high even on an internal connector, because `MaxInboundConnectionPerSource` (default 20) and `MaxInboundConnection` (default 5000) also count: A client that repeatedly opens new connections in addition to a stuck connection reaches the per-source limit sooner the longer old connections remain open.

For senders that send `NOOP` between messages, set `TarpitInterval` on the same connector to `00:00:00`. The tarpit delay provides no benefit for authenticated or IP-restricted internal senders and artificially extends every session.

The Exchange-side change fixes the symptom. The more reliable solution is in the client: It reconnects after a fixed number of messages, as Exchange does after 20 and Postfix does after five minutes. With `.NET SmtpClient`, this means creating and discarding the object for blocks of, for example, 100 messages; with JavaMail, the `Transport` is closed and reopened accordingly. This allows sending to targets whose timeouts cannot be adjusted, especially Exchange Online, whose inbound connectors have no timeout parameters.

## Additional time limits on the path

The Exchange value is not the only limit. Firewalls and load balancers maintain their own idle timers for TCP connections: A FastL4 profile on an F5 BIG-IP defaults to 300 seconds, and Azure Load Balancer to four minutes. These timers measure idle time rather than total duration and therefore take effect during sending pauses, for example when a batch job reads data from the database between two blocks. The smallest value along the entire path always applies. The article [F5 BIG-IP as an outbound proxy for bulk email delivery](https://rafaelpfister.ch/blog/f5-big-ip-outbound-smtp-massenversand) explains how to size timeouts on a load balancer for persistent SMTP connections.

## Sources

1.  [Microsoft Learn: Set-ReceiveConnector](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-receiveconnector): Reference with the default values and valid ranges of `ConnectionTimeout`, `ConnectionInactivityTimeout`, `TarpitInterval`, `MaxInboundConnection`, and `MaxInboundConnectionPerSource` for Mailbox and Edge Transport servers.

2.  [Microsoft Learn: Set-SendConnector](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-sendconnector): `ConnectionInactivityTimeOut` and `SmtpMaxMessagesPerConnection` on the sending side.

3.  [Microsoft Learn: Protocol logging](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/protocol-logging): Locations, file names, and field layout of SMTP protocol logs for Front End and Transport service.

4.  [Microsoft Learn: Mail routing in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing): The Front End Transport service as a stateless proxy in front of the Transport service.

5.  [RFC 5321, section 4.5.3.2 Timeouts](https://www.rfc-editor.org/rfc/rfc5321.html#section-4.5.3.2): Minimum wait times for each protocol step, the rationale for ten minutes after the final period, and behavior for `421` in section 3.8.

6.  [Postfix: postconf(5)](https://www.postfix.org/postconf.5.html): `smtp_connection_reuse_time_limit` (300s) and `smtpd_timeout` as an example of an MTA that keeps sessions short on its own.

7.  [Broadcom Knowledge Base: Quarantine notification process appears to be failing, logs may show 421 4.4.1 Connection timed out](https://knowledge.broadcom.com/external/article/154389/quarantine-notification-process-appears.html): Documented case of a gateway reaching the Exchange total timeout with `NOOP` keepalive and tarpit.

8.  [Microsoft Learn: SmtpClient Class](https://learn.microsoft.com/en-us/dotnet/api/system.net.mail.smtpclient): Connection reuse across multiple `Send` calls and `QUIT` only during `Dispose`.
