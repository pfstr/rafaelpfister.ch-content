---
title: "Understanding SMTP: How Email Is Actually Delivered"
blatt: "smtp"
description: "What SMTP is and how mail flow works: envelope and headers, MX delivery, relays and smarthosts, the ports 25, 465 and 587, StartTLS, and the reply codes with which servers justify acceptance and rejection."
fakten:
  - { label: "Full name", wert: "Simple Mail Transfer Protocol" }
  - { label: "Purpose", wert: "Delivery of email between servers and from client to server" }
  - { label: "Introduced", wert: "1982 (RFC 821) · currently RFC 5321" }
  - { label: "OSI layer", wert: "Application (layer 7)" }
  - { label: "Transport", wert: "TCP" }
  - { label: "Ports", wert: "25 (server to server) · 587 (submission) · 465 (submission over TLS)", href: "https://datatracker.ietf.org/doc/html/rfc8314" }
  - { label: "Standard", wert: "RFC 5321 (protocol) · RFC 5322 (message format)", href: "https://datatracker.ietf.org/doc/html/rfc5321" }
  - { label: "Sender validation", wert: "SPF, DKIM, DMARC (via DNS)" }
  - { label: "Tools", wert: "openssl s_client, swaks, telnet" }
werbung: ["tools", "newsletter"]
ctaThemen: ["smtp-mailflow"]
---

# Understanding SMTP: How Email Is Actually Delivered

Every email that leaves or reaches an organization travels over SMTP. The protocol is more than forty years old, text-based and so plain that it can be spoken by hand. That is exactly what makes it valuable for administrators: whoever understands the dialog can trace any delivery problem back to its cause.

## Envelope and headers are two different things

SMTP transports a message inside an envelope. The **envelope** consists of `MAIL FROM` (the sender for bounces, also called the return path) and `RCPT TO` (the actual recipients). The **headers** in the message text (`From:`, `To:`, `Subject:`) are independent of it and secondary for the server. Many phenomena follow from this separation: BCC recipients appear only in the envelope, bounces go to the envelope sender, and spoofing often begins with the envelope and header senders diverging. This is exactly where SPF (which checks the envelope) and DMARC (which checks alignment with the header From) come in.

## The path of a message

The sending server queries DNS for the **MX record** of the destination domain, connects on port 25 and runs the SMTP dialog: `EHLO`, optionally `STARTTLS`, then `MAIL FROM`, `RCPT TO`, `DATA`. The receiving server answers each step with a three-digit code. **2xx** means accepted, **4xx** means temporarily rejected (the sender retries later, typically with greylisting or full queues), **5xx** means permanently rejected. These codes are the most important diagnostic currency in mail flow: a `550 5.1.1` says "recipient does not exist", a `451 4.7.1` points to reputation or policy problems.

In organizations the path is rarely direct. **Relays and smarthosts** accept mail internally and forward it, secure mail gateways such as SEPPmail or Totemomail sit as an additional station in the path, and Exchange Online talks to the outside world through connectors. Every station writes a `Received:` header. The chain of these headers, read from bottom to top, is the record of the delivery path.

## Client submission: 587 and 465 instead of 25

Port 25 belongs to server-to-server delivery; many providers block it for consumer connections. Clients and applications submit over **submission**: port **587** with StartTLS or port **465** with immediate TLS (implicit TLS, officially sanctioned again since RFC 8314). Submission requires authentication (`AUTH`), and that is precisely where clean operations part ways with legacy baggage: SMTP basic auth against Exchange Online is deprecated, and multifunction devices and scripts need modern routes such as OAuth or a dedicated relay.

## Transport encryption is opportunistic

Between servers, **StartTLS** is the standard: the connection begins in cleartext and is upgraded to TLS by command. Without additional measures this remains opportunistic, because when TLS fails, many servers deliver in cleartext. Transport encryption becomes binding only with **MTA-STS** or **DANE**, both anchored in DNS. For confidential content, transport encryption is not sufficient in any case; that is where end-to-end or gateway encryption with S/MIME and PGP begins.

## Testing SMTP by hand

The quickest integrity test of a connection including its certificate:

```bash
openssl s_client -starttls smtp -connect mail.example.ch:25 -servername mail.example.ch
```

