---
title: "NDR, DSN, Bounce: How to distinguish non-delivery notifications correctly"
navTitle: "NDR & Bounces"
description: "NDR, DSN, bounce, reject, backscatter: The terms surrounding failed delivery are often used interchangeably, but refer to different things. What the RFCs define, who generates which notification, how a DSN is structured, and why the difference between reject and bounce determines backscatter."
date: "2026-08-28"
kategorie: "SMTP and mail flow"
timeToRead: "10 min read"
themen:
  - smtp-mailflow
protokolle:
  - "smtp"
  - "troubleshooting"
slug: "ndr-dsn-bounce-how-to-correctly-distinguish-non-delivery-notifications"
translationId: "article-5c5164049a129fa4"
translationOf: ndr-dsn-bounce-unterschiede
translationSourceHash: 1500e27283dc4c53456810f133d12d81e1f60d7e874b6f3450b0a50fb47d4cf2
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:19:00.151Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/en/blog/ndr-dsn-bounce-how-to-correctly-distinguish-non-delivery-notifications
---

# NDR, DSN, Bounce: How to distinguish non-delivery notifications correctly

An email does not arrive, and the ticket may say “bounce,” “NDR,” “Mailer-Daemon,” or “server error message.” In day-to-day administration, these terms are used interchangeably, even though they refer to different things: A reject during the SMTP session is not a bounce email, a delay notification is not a delivery failure, and a read receipt has nothing to do with non-delivery. Those who distinguish the terms correctly find the cause faster, because each type of notification says something different about where the problem lies in the transport path and who can resolve it.

## DSN: the umbrella term from the RFCs

The formal umbrella term is Delivery Status Notification (DSN), defined in RFCs 3461 through 3464. A DSN is a machine-generated email that informs the sender about the delivery status of their message. Crucially, a DSN does not report only failures. The `Action` field in the machine-readable part has five values:

| Action | Meaning |
|---|---|
| `failed` | Delivery permanently failed; the email will not be retried |
| `delayed` | Delivery delayed; the server continues trying |
| `delivered` | Successfully delivered (delivery confirmation, only upon explicit request) |
| `relayed` | Relayed to a server that does not itself generate DSNs |
| `expanded` | Handed to a distribution list and expanded |

A non-delivery notification is therefore only a special case: a DSN with `Action: failed`. Microsoft calls precisely this special case a Non-Delivery Report (NDR). The term NDR originated in the Exchange world, but has since become widely used across vendors. To be precise: Every NDR is a DSN, but not every DSN is an NDR.

The delay notification (`Action: delayed`) deserves special attention because it is regularly misunderstood as a delivery failure in support cases. A typical subject is “Delivery delayed.” The email is still in the sending server's queue, which continues trying, usually for one to two days. Only when the queue lifetime expires does the final NDR follow. A user who resends the email after receiving a delay notification creates duplicates as soon as the target system becomes reachable again.

## Reject or bounce: the most important distinction

Before covering the other terms, the key technical distinction needs explaining, because it determines which server generates a notification.

**Reject (rejection during the session):** The receiving server rejects the email during the SMTP session, with a 5xx response code to `RCPT TO` or after `DATA`. It never accepts the email and does not itself generate a notification email. The responsibility for informing the sender lies with the submitting server: The sending MTA sees the 5xx response and then generates the NDR for its local user. In this case, the NDR the user reads comes from their own server, but quotes the error message from the remote system.

**Bounce (acceptance followed by a later return):** The receiving server accepts the email with `250 OK` and only afterward discovers that it cannot deliver it, for example because the mailbox does not exist, the quota is full, or a downstream server rejects it. It now bears responsibility for the message and must itself send a DSN to the sender. This subsequent returned email is a bounce in the narrower sense.

For troubleshooting, the distinction is immediately useful: If the NDR lists your own server as the generating system, the email was rejected during the session or never left at all. If a foreign server is the sender of the notification, the other side initially accepted the email, and the problem is behind its acceptance point, invisible to the sender.

Two additional bounce terms come from the marketing world and do not appear in any RFC: hard bounce for permanent errors (5xx, `Action: failed`) and soft bounce for temporary ones (4xx, `Action: delayed`). For mailing platforms, the distinction is crucial because hard bounces should lead to immediate list cleanup. Technically, they are the same mechanisms described above.

## Terms at a glance

| Term | What it is | Who generates the notification | Standard |
|---|---|---|---|
| DSN | Umbrella term: delivery status notification (failed, delayed, delivered, relayed, expanded) | The MTA responsible for the email | RFC 3461 through 3464 |
| NDR | DSN with `Action: failed`; Microsoft term for a non-delivery notification | Sending MTA (after reject) or receiving MTA (after acceptance) | RFC 3464, Microsoft documentation |
| Reject | 5xx rejection during the active SMTP session; no separate email | Nobody; the sending MTA turns it into an NDR | RFC 5321 |
| Bounce | Returned email after prior acceptance | Receiving MTA | RFC 5321, RFC 3464 |
| Hard/Soft Bounce | Marketing classification: permanent (5xx) vs. temporary (4xx) | Same as bounce | No RFC |
| Delay notification | DSN with `Action: delayed`; email is still in the queue | Sending or relaying MTA | RFC 3464 |
| Backscatter | NDRs sent to forged sender addresses, usually triggered by spam | Misconfigured receiving MTAs | No RFC, anti-abuse term |
| MDN / read receipt | Notification that the recipient displayed or deleted the message | Recipient's email client | RFC 8098 |
| Out-of-office reply | Automatic response from a mailbox that was reached | Mailbox or groupware server | RFC 3834 |

## How a DSN is structured

Standards-compliant DSNs use the MIME type `multipart/report; report-type=delivery-status` with three parts: a human-readable explanation, a machine-readable part of type `message/delivery-status` and, optionally, the original message or its headers. The machine-readable part is the most valuable for diagnosis because its fields are standardized:

```text
Reporting-MTA: dns; mail01.example.net
Received-From-MTA: dns; client.example.org

Final-Recipient: rfc822; max.muster@example.com
Action: failed
Status: 5.1.1
Remote-MTA: dns; mx.example.com
Diagnostic-Code: smtp; 550 5.1.1 <max.muster@example.com>:
    Recipient address rejected: User unknown
```

| Field | Meaning |
|---|---|
| `Reporting-MTA` | The server that generated this DSN; the first indication of responsibility |
| `Final-Recipient` | The recipient address to which the status applies (one block per recipient) |
| `Action` | One of the five status values (failed, delayed, delivered, relayed, expanded) |
| `Status` | Enhanced Status Code according to RFC 3463, for example `5.1.1` |
| `Remote-MTA` | The remote system with which the reporting MTA communicated |
| `Diagnostic-Code` | The literal SMTP response from the remote system; often the most informative line |

A DSN is always sent with an empty envelope sender (`MAIL FROM:<>`). This is not an oversight, but a requirement of RFC 5321: The empty sender prevents another DSN from following an undeliverable DSN and two servers from sending error messages to each other indefinitely. This leads to a configuration rule: A mail system must not categorically reject emails with an empty envelope sender, or legitimate non-delivery notifications will never reach its users.

Exchange and Exchange Online follow the standard for the format, but package the content in their own presentation: The user sees a formatted page with a plain-language explanation, followed by “Generating server” (corresponding to `Reporting-MTA`) and the raw details. For diagnosis, it is always worth looking at this lower technical section.

## Reading Enhanced Status Codes

The `Status` field and usually also `Diagnostic-Code` contain a three-part code according to RFC 3463: class.subject.detail. The class indicates finality, while subject and detail indicate the cause:

| Code range | Meaning |
|---|---|
| `2.x.x` | Success (only in delivery confirmations) |
| `4.x.x` | Temporary failure; the server will try again |
| `5.x.x` | Permanent failure; no further attempts |
| `x.1.x` | Addressing problem, e.g. `5.1.1` unknown recipient, `5.1.10` domain without MX |
| `x.2.x` | Mailbox problem, e.g. `5.2.2` mailbox full, `5.2.3` message too large for mailbox |
| `x.3.x` | Target system problem, e.g. `4.3.2` system is not accepting anything at the moment |
| `x.4.x` | Network and routing, e.g. `4.4.1` no response, `4.4.7` queue lifetime expired |
| `x.5.x` | Protocol error in the SMTP dialog |
| `x.7.x` | Policy and security, e.g. `5.7.1` relay denied or policy rejection, `5.7.26` missing authentication (SPF/DKIM/DMARC) |

The classic three-digit SMTP response code (such as `550`) and the Enhanced Status Code often appear together on one line: `550 5.7.1 ...`. The three-digit code controls the sending server's protocol behavior; the enhanced code provides the diagnostic statement. If the code and free text conflict, the remote system's free text is often the more precise source, since many systems use generic codes and write the actual cause in the comment, including reference IDs for the other side's support.

Note that `5.7.x` rejections by reputation and content filters often intentionally provide little information. Looking only at the code sends you looking in the wrong place; the blocklist or filter vendor named in the free text will get you to the answer faster.

## Backscatter: the harmful kind of bounce

Backscatter occurs when a server first accepts spam with a forged sender and then sends an NDR to the forged address. The NDR thus reaches an uninvolved party whose address was abused by the spammer. During major spam waves, affected parties receive thousands of NDRs for emails they never sent, and servers that generate such NDRs in large volumes end up on blocklists themselves (such as UCEPROTECT's Backscatterer list).

The remedy follows directly from the reject-bounce distinction: Anything that can be rejected belongs in the SMTP session, not sent back after acceptance. Specifically, this means recipient validation at the outermost acceptance point (the edge gateway knows the valid addresses, through directory lookup or Recipient Callout, instead of accepting everything and failing internally), spam and malware rejection during the session rather than quarantine NDRs, and avoiding NDRs for messages classified as spam. A reject does not generate backscatter because, with a forged sender, the 5xx response reaches the spammer's server, which does not create an NDR to the victim from it.

## What is not a non-delivery notification

Three types of notifications regularly end up in the same bucket in tickets, but do not belong there:

**MDN (Message Disposition Notification, RFC 8098):** The read receipt. It is not generated by the transport system, but by the recipient's email client, and reports that the message was displayed or deleted, not that it was delivered. The MIME type is accordingly `multipart/report; report-type=disposition-notification`. A missing read receipt says nothing about delivery; most clients ask the user or suppress MDNs entirely.

**Out-of-office replies and autoresponders (RFC 3834):** An out-of-office reply proves the opposite of a delivery failure because it requires the email to have reached the mailbox. In ticket descriptions (“I receive an automatic response—did my email arrive?”), it is worth asking which notification is actually present.

**Quarantine notifications:** Notifications such as the quarantine digest from Microsoft 365 or a gateway inform the recipient about held emails. They go to the recipient, not the sender, and do not follow a DSN standard. In this scenario, the sender often receives nothing at all, which explains cases where an email “disappears without an error message.”

## Diagnostic checklist

When a notification is available, clarify the following in this order:

1. What type is it: NDR (`Action: failed`), delay (`Action: delayed`), MDN, autoresponder, or quarantine notice? For a delay notification: wait; do not resend.
2. Who generated the notification (`Reporting-MTA` or “Generating server”)? Your own server means a reject or internal error; a foreign server means acceptance followed by a later failure on the other side.
3. What do the status and diagnostic code say? Class 4 versus class 5 distinguishes temporary from permanent; the subject (`x.1` address, `x.2` mailbox, `x.4` network, `x.7` policy) narrows down the cause, and the remote system's free text provides the details.
4. If no notification is received even though the email does not arrive: Check message tracking on your own system and consider quarantine or silent filtering on the other side.

The articles on [Message Tracking and SMTP diagnostics in the Command Generator](https://rafaelpfister.ch/tools/command-builder) and the [Mail Header Analyzer](https://rafaelpfister.ch/tools/header-analyzer) show how to specifically reproduce individual delivery paths and evaluate the transport path of an email that arrived.

## Change history

- **October 25, 2026:** Corrected: Updated the Mail Header Analyzer link to the correct address. Updated the source link to the new Microsoft Learn address.

## Sources

1.  [RFC 3461: SMTP Service Extension for Delivery Status Notifications](https://www.rfc-editor.org/rfc/rfc3461): SMTP extension that lets senders request and control DSNs.

2.  [RFC 3463: Enhanced Mail System Status Codes](https://www.rfc-editor.org/rfc/rfc3463): Definition of the three-part status codes (class.subject.detail).

3.  [RFC 3464: An Extensible Message Format for Delivery Status Notifications](https://www.rfc-editor.org/rfc/rfc3464): Structure of the DSN as multipart/report, with fields such as Action, Status, and Diagnostic-Code.

4.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): Basic rules for response codes, transfer of responsibility upon acceptance, and an empty envelope sender for error messages.

5.  [RFC 8098: Message Disposition Notification](https://www.rfc-editor.org/rfc/rfc8098): Standard for read receipts, distinguishing them from DSNs.

6.  [RFC 3834: Recommendations for Automatic Responses to Electronic Mail](https://www.rfc-editor.org/rfc/rfc3834): Rules for autoresponders such as out-of-office replies.

7.  [Microsoft Learn: Email non-delivery reports and SMTP errors in Exchange Online](https://learn.microsoft.com/en-us/troubleshoot/exchange/email-delivery/ndr/non-delivery-reports-in-exchange-online): NDR structure and code list from the Exchange Online perspective.

8.  [UCEPROTECT Backscatterer](https://www.backscatterer.org/): Blocklist for systems that generate backscatter; explains the listing criteria.
