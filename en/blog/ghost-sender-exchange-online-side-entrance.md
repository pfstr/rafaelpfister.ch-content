---
title: "Ghost Sender in Exchange Online: An MX Record Is Not a Firewall"
navTitle: "Ghost Sender"
description: "Direct delivery to Exchange Online bypasses an upstream gateway if the tenant does not explicitly block it. The risk is real, and the cause is an incomplete mail flow configuration."
date: "2026-07-15"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min read"
themen:
  - "microsoft-365-exchange"
slug: "ghost-sender-exchange-online-side-entrance"
translationOf: "ghost-sender-exchange-online-nebeneingang"
image: "../images/ghost-admin.png"
url: "https://rafaelpfister.ch/en/blog/ghost-sender-exchange-online-side-entrance"
---

# Ghost Sender in Exchange Online: An MX Record Is Not a Firewall

![A ghost admin holds open the door beside the security gate in the data centre while emails bypass the filter and go straight to the mailbox.](../images/ghost-admin.png)

The attack vector described by InfoGuard Labs as ‘Ghost Sender’ is real: an attacker can bypass an upstream email gateway and deliver directly to Exchange Online. However, this requires the tenant to continue accepting this direct route. This is not a universal vulnerability in Exchange Online, but an incompletely secured mail flow topology.

A Mail Transfer Agent that serves mailboxes for a domain generally accepts SMTP connections from the internet. The MX record tells legitimate senders the intended delivery route. It is neither a firewall rule nor an access list, and it does not prevent anyone from addressing a known Exchange Online endpoint directly.

## What ‘Ghost Sender’ actually shows

The scenario [described by InfoGuard Labs](https://labs.infoguard.ch/posts/ghost-sender/) looks like this:

1. An organisation hosts its mailboxes in Exchange Online.
2. The public MX record points to an upstream Secure Email Gateway.
3. The Exchange Online endpoint at `*.mail.protection.outlook.com` remains directly reachable from the internet.
4. The administrator has not restricted Exchange Online so that only the upstream gateway may deliver there.
5. An attacker ignores the MX record and submits their message directly to Exchange Online.

The intended route is therefore:

```text
Internet -> Drittanbieter-Filter -> Exchange Online -> Postfach
```

However, this route remains open:

```text
Angreifer -> Exchange Online -> Postfach
```

This is a serious misconfiguration. The upstream filter can be bypassed through this route; spoofed senders, phishing and CEO fraud are made considerably easier as a result. InfoGuard deserves credit for making the issue visible, investigating its prevalence and publishing an easy-to-use test.

But where exactly is the product flaw here?

The media framing does little to aid classification either. [Heise headlines that Exchange Online lets forged emails ‘through without complaint’](https://www.heise.de/news/Ghost-Sender-Exchange-Online-laesst-gefaelschte-E-Mails-anstandslos-durch-11327666.html), even though only certain insufficiently hardened third-party and hybrid configurations are affected. [Crow in the Cloud](https://crowinthe.cloud/die-geister-die-ich-nicht-rief-effektiver-schutz-gegen-ghost-sender-in-exchange-online/) puts it far more accurately: not a security hole in the strict sense, but a design and configuration issue.

## ‘An MTA is doing MTA-Things’

Every Exchange Online tenant has a public SMTP endpoint. This endpoint is not secret, nor should it be. Microsoft itself explains that Exchange Online accepts messages addressed directly to mailboxes hosted there by default: [that is simply how email works](https://techcommunity.microsoft.com/blog/exchange/direct-send-vs-sending-directly-to-an-exchange-online-tenant/4439865).

[SMTP itself also describes the MX record as a mechanism for identifying the regular destination system](https://www.rfc-editor.org/rfc/rfc5321.html#section-5.1). This does not oblige the destination server to reject connections through every other reachable host. An attacker does not have to follow the signposted route. If another MTA is reachable, knows the recipient domain and accepts the message, it will be tried, much as spammers have attempted to contact poorly protected backup MX systems for decades.

Anyone placing a third-party filter upstream changes the standard topology. ‘Exchange Online is my internet mail gateway’ becomes ‘only my third-party gateway may pass internet mail to Exchange Online’. This new `Trust-Border` is not created by a DNS entry. It must be explicitly enforced on the receiving system.

Microsoft documents precisely this: where there is an external MX, an inbound connector of type `Partner` should be created which, for `SenderDomains *`, accepts only the certificate or source IP addresses of the upstream service. Messages delivered directly past the gateway are then rejected. This is stated verbatim in Microsoft’s guide [‘Manage mail flow using a third-party cloud service with Exchange Online’](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud#best-practices-for-using-a-third-party-cloud-filtering-service-with-microsoft-365-or-office-365).

Frank Carius also describes this ‘side entrance’ in detail in the [MSXFAQ](https://www.msxfaq.de/cloud/exchangeonline/transport/exo-nebeneingang.htm).

## SPF, DKIM and DMARC are not bouncers

InfoGuard shows messages for which SPF, DKIM and DMARC fail yet which still arrive in the mailbox. This looks spectacular, but it is not a cryptographic ‘bypass’ of these mechanisms. The emails do not succeed in passing them. They return `fail`. What matters is which local action the receiving system derives from this result.

SPF checks whether a system is authorised to send for the envelope sender. DKIM checks a signature. DMARC combines these results with the visible sender domain and publishes a requested treatment. Even the current [DMARC standard RFC 9989](https://www.rfc-editor.org/rfc/rfc9989.html#section-1) explicitly states that the recipient may take this requested treatment into account, but is not obliged to do so. DMARC is an important signal, but not a network access control.

With an upstream gateway, Exchange Online initially sees the IP address of that gateway rather than that of the original sender. This is what [Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors) is for: it reconstructs the original source and improves SPF, DKIM, DMARC, anti-spoofing and anti-phishing evaluation. However, Enhanced Filtering is not a door lock either. It does not replace the restrictive partner connector.

The misconfiguration becomes particularly obvious when an administrator weakens or completely bypasses EOP checks using an SCL bypass because the upstream product is supposedly already filtering, while at the same time leaving direct internet delivery open. In that case, they have not had a protection mechanism ‘bypassed’; they have deliberately provided no effective protection for one of two entrances.

Microsoft can certainly be criticised if a message with a clearly visible authentication failure lands in the inbox without a warning. The semantics of connector types, the documentation and missing warnings in the Configuration Analyzer can all be criticised. These are legitimate points. However, the existence of a publicly reachable SMTP endpoint is not a security vulnerability.

## ‘Direct Send’ is not the same as ‘direct delivery’

The discussion conflates two things:

- **Direct Send** refers, at Microsoft, to anonymous messages whose envelope sender (`5321.MailFrom`) uses one of the tenant’s own accepted domains.
- **Direct delivery to Exchange Online** generally refers to an SMTP message that ignores the published third-party MX and is submitted directly to the Exchange endpoint. The sender may also use any external domain.

The switch

```powershell
Set-OrganizationConfig -RejectDirectSend $true
```

is useful if Direct Send is not required. It prevents internal domain spoofing through this route. However, it does not close the entire side entrance for arbitrary external senders. Microsoft describes the exact scope in the [cmdlet documentation for `RejectDirectSend`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-organizationconfig?view=exchange-ps#-rejectdirectsend). Anyone wishing to prevent ‘Ghost Sender’ completely still needs access restrictions via a partner connector or a suitable mail flow rule.

## Does Microsoft really have to do everything for the administrator?

No. Anyone adding an extra mail filter to a production transport chain assumes responsibility for that transport chain.

The provider cannot reliably guess whether, in addition to the external MX, scanners, multifunction devices, SaaS services, hybrid servers, partner relays or other legitimate systems need to send directly to Exchange Online. An automatic ‘the MX points elsewhere, so block everything else’ would interrupt desired mail flows in many real-world environments. The administrator must therefore explicitly define the intended trust boundary.

Nevertheless, Microsoft should make this easier for those responsible. A good Configuration Analyzer should detect an external MX without a restrictive partner connector and issue a clear warning. The setup dialogue could explain that a connector of type ‘Your organisation’ identifies matching connections, but does not automatically reject non-matching connections. Secure-by-default switches and better operational reports would also be welcome.

That would be sensible product hardening. But it does not change the technical assessment: an insecure special topology remains an insecure configuration and does not become a zero-day merely because it is widespread.

## How to close the side entrance

For environments with an upstream filter, at least these points belong on the checklist:

1. **Document the mail flow completely.** Which systems are actually permitted to deliver to Exchange Online? This includes hybrid, application and emergency routes.
2. **Set up a restrictive partner connector.** Use `SenderDomains *` and restrict delivery to a certificate (preferred) or maintained source IP ranges. A connector of type `OnPremises` or ‘Your organisation’ does not enforce this default-deny effect (see, for example: [Mail routing between Apache James and Exchange Online](/blog/totemomail-m365)).
3. **Configure Enhanced Filtering correctly.** If EOP is to continue filtering, the original IP and sender information must be reconstructed correctly. Blanket SCL-`-1` bypasses should be reviewed critically.
4. **Disable Direct Send if unused.** First, use Message Trace or the available reports to check whether scanners or applications depend on it.
5. **Do not switch blindly.** Test and then monitor gateway IP ranges, certificate changes, hybrid mail flow, and `onmicrosoft.com`-, Teams and other special routes.

A simplified example of the IP-based variant is:

```powershell
New-InboundConnector `
  -Name "Only from upstream mail gateway" `
  -ConnectorType Partner `
  -SenderDomains * `
  -RestrictDomainsToIPAddresses $true `
  -SenderIpAddresses <Gateway-IP-ranges> `
  -RequireTls $true
```

Where possible, certificate binding should be preferred over an IP allowlist. Changes should first be made in a controlled test, because a faulty allowlist can very quickly turn the open side entrance into a complete mail outage.

## The simple self-test

The test shown by InfoGuard (and MSXFAQ) is useful:

```powershell
Send-MailMessage `
  -SmtpServer <tenant-name>.mail.protection.outlook.com `
  -To admin@<tenant-domain> `
  -From noreply@example.com `
  -Subject "EXO side entrance" `
  -Body "Test email directly to the tenant"
```

With a correctly restricted partner connector, an SMTP rejection such as `5.7.51 TenantInboundAttribution; Rejecting` is to be expected. An alternative transport rule may initially accept the message and then move it to quarantine; therefore, in addition to the SMTP response, Message Trace, quarantine and the mailbox must also be checked. `Send-MailMessage` (deprecated) is used here only as an easily understandable illustration. Any controlled SMTP testing tool serves the same purpose.

## A useful test with a misleading label

‘Ghost Sender’ is not a new SMTP exploit. It is a catchy name for an open side entrance whose protection Microsoft has documented for a long time and which the administrator has left open.

The irony is that InfoGuard itself describes the issue in its own article as a ‘widespread and systematic misconfiguration’ and concludes with the sentence ‘Ghost-Sender is a misconfiguration’. Microsoft’s Security Response Center also initially did not classify the report as a security vulnerability. The facts are therefore present in the article: unfortunately, only the title, test email and ‘Vulnerability’ branding tell a more dramatic story.

The useful part of the publication is the wake-up call: many companies apparently have not properly locked down their mail flow. The problematic part is the claim that Exchange Online has a universal security vulnerability as a result. No: Exchange Online initially behaves like an MTA here. It becomes insecure because the trust boundary has not been fully configured.

Does Microsoft really have to do everything for the administrator? No. But apparently it is necessary to keep reminding them that DNS routing does not replace access control.

## Sources

1.  [InfoGuard Labs: Ghost-Sender – Universal Email Spoofing against Exchange Online](https://labs.infoguard.ch/posts/ghost-sender/): The original investigation, including its prevalence analysis and its own conclusion that ‘Ghost-Sender is a misconfiguration’.

2.  [Ghost Sender: Exchange Online Mail Spoofing Tester](https://ghost-sender.com/): The online test published by InfoGuard to check your own tenant for the open side entrance.

3.  [MSXFAQ: Exchange Online as a side entrance for receiving mail](https://www.msxfaq.de/cloud/exchangeonline/transport/exo-nebeneingang.htm): Frank Carius’ assessment: not an error in Exchange Online, but an administrator misconfiguration.

4.  [Microsoft: Direct Send vs sending directly to an Exchange Online tenant](https://techcommunity.microsoft.com/blog/exchange/direct-send-vs-sending-directly-to-an-exchange-online-tenant/4439865): Microsoft explains that accepting mail directly for hosted mailboxes is how email works, and distinguishes it from Direct Send.

5.  [Microsoft Learn: Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud): The official guide, including the dedicated step for a restrictive partner connector when using an external MX.

6.  [Microsoft Learn: Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors): Reconstructs the original sender source behind a gateway; improves evaluation but does not replace the connector.

7.  [Heise: Ghost-Sender – Exchange Online lets forged emails through without complaint](https://www.heise.de/news/Ghost-Sender-Exchange-Online-laesst-gefaelschte-E-Mails-anstandslos-durch-11327666.html): An example of sensationalised reporting that generalises only certain misconfigurations.

8.  [Crow in the Cloud: The ghosts I did not summon](https://crowinthe.cloud/die-geister-die-ich-nicht-rief-effektiver-schutz-gegen-ghost-sender-in-exchange-online/): An apt assessment as a design and configuration issue, including protective measures.

9.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321.html): Describes the MX record as a mechanism for identifying the regular destination system, not as an access control.

10.  [RFC 9989: DMARC](https://www.rfc-editor.org/rfc/rfc9989.html): States that the recipient may take the published DMARC treatment into account, but does not have to.

---

## Is your mail flow secure?

Unsure whether your Exchange Online tenant also has an open side entrance? **adeptio** reviews your entire mail flow: from MX records, connectors and third-party gateways to EOP, SPF, DKIM, DMARC and Direct Send. Practical, independent and with concrete recommendations.

Anyone wishing to have their mail flow reviewed or properly secured is welcome to arrange a no-obligation consultation:

**[Book a consultation with adeptio](https://outlook.office.com/bookwithme/user/b4d64d6bdbca4b489074d459cd30b50c@adeptio.ch/meetingtype/3Wgk7rXJfk261852Hyovkg2?anonymous&ismsaljsauthenabled&ep=mlink)**  
[adeptio.ch](https://adeptio.ch/)
