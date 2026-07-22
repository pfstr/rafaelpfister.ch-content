---
title: "Ghost Sender or Ghost Admin? An MX Record Is Not a Firewall!"
navTitle: "Ghost Sender"
description: "Why Ghost Sender is not a new vulnerability in Exchange Online, but the predictable consequence of an open SMTP side entrance."
date: "2026-07-15"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min to read"
themen:
  - "microsoft-365-exchange"
slug: "ghost-sender-exchange-online-side-entrance"
translationOf: "ghost-sender-exchange-online-nebeneingang"
url: "https://rafaelpfister.ch/en/blog/ghost-sender-exchange-online-side-entrance"
image: "../images/ghost-admin.png"
---

# Ghost Sender or Ghost Admin? An MX Record Is Not a Firewall!

![A ghost admin holds the door next to the security gate open in a data center while emails slip past the filter straight into the mailbox.](../images/ghost-admin.png)

**InfoGuard Labs stirred up so much concern with "Ghost-Sender – Universal Email Spoofing against Exchange Online" that several inquiries about the "vulnerability" landed on my desk as well. Short verdict: the risk described is real. Classifying it as a universal vulnerability in Exchange Online is not. Anyone who places a third-party mail filter in front of Exchange Online but does not restrict the still-reachable EXO endpoint to that filter has not discovered a new hole. They simply have not finished configuring their mail flow.**

It looks as if, for a moment, InfoGuard forgot how a Mail Transfer Agent (MTA) works. A mail server that hosts mailboxes for a domain fundamentally accepts SMTP connections from the internet. That is exactly what it is there for. An MX record merely tells regular senders which path to take for delivery. It is neither a firewall rule nor an access control list.

## What "Ghost Sender" Actually Shows

The scenario [described by InfoGuard Labs](https://labs.infoguard.ch/posts/ghost-sender/) looks like this:

1. An organization runs its mailboxes in Exchange Online.
2. The public MX record points to an upstream secure email gateway.
3. The Exchange Online endpoint at `*.mail.protection.outlook.com` remains directly reachable from the internet.
4. The administrator has not restricted Exchange Online so that only the upstream gateway may deliver there.
5. An attacker ignores the MX record and delivers the message directly to Exchange Online.

The intended path is therefore:

```text
Internet -> Drittanbieter-Filter -> Exchange Online -> Postfach
```

However, this path remained open:

```text
Angreifer -> Exchange Online -> Postfach
```

That is a misconfiguration worth taking seriously. The upstream filter can be bypassed this way; spoofed senders, phishing, and CEO fraud become significantly easier. InfoGuard deserves credit for making the problem visible, investigating how widespread it is, and publishing an easy-to-use test.

But where exactly is the product defect supposed to be here?

The media coverage doesn't help much with the assessment either. [Heise's headline claims Exchange Online lets forged emails through "without a hitch"](https://www.heise.de/news/Ghost-Sender-Exchange-Online-laesst-gefaelschte-E-Mails-anstandslos-durch-11327666.html), even though only certain, incompletely hardened third-party and hybrid configurations are affected. [Crow in the Cloud](https://crowinthe.cloud/die-geister-die-ich-nicht-rief-effektiver-schutz-gegen-ghost-sender-in-exchange-online/) puts it much more accurately: not a security hole in the narrow sense, but a design and configuration problem.

## "An MTA Is Doing MTA-Things"

Every Exchange Online tenant has a public SMTP endpoint. That endpoint is no secret, nor is it meant to be. Microsoft itself explains that Exchange Online accepts messages by default that are addressed directly to mailboxes hosted there: [that's simply how email works](https://techcommunity.microsoft.com/blog/exchange/direct-send-vs-sending-directly-to-an-exchange-online-tenant/4439865).

[SMTP itself also describes the MX record as the mechanism for determining the regular target system](https://www.rfc-editor.org/rfc/rfc5321.html#section-5.1). That does not create any obligation for the destination server to reject connections coming through any other reachable host. An attacker does not have to follow the signposted path. If another MTA is reachable, knows the recipient domain, and accepts the message, it will be tried, much the way spammers have been probing poorly protected backup MX systems for decades.

Anyone who places a third-party filter in front changes the default topology. "Exchange Online is my internet mail gateway" becomes "only my third-party gateway may hand internet mail to Exchange Online." This new trust border does not arise from a DNS entry. It must be explicitly enforced on the receiving system.

That is exactly what Microsoft documents: with an external MX, you should set up an inbound connector of type `Partner` that, for `SenderDomains *`, accepts only the certificate or the source IP addresses of the upstream service. Messages delivered straight past the gateway are then rejected. That is stated verbatim in Microsoft's guide ["Manage mail flow using a third-party cloud service with Exchange Online"](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud#best-practices-for-using-a-third-party-cloud-filtering-service-with-microsoft-365-or-office-365).

Frank Carius also describes this "side entrance" in detail in the [MSXFAQ](https://www.msxfaq.de/cloud/exchangeonline/transport/exo-nebeneingang.htm).

## SPF, DKIM, and DMARC Are Not a Bouncer

InfoGuard shows messages where SPF, DKIM, and DMARC fail and that still land in the mailbox. That looks spectacular, but it is not a cryptographic "bypass" of these mechanisms. The messages precisely do not pass successfully. They return `fail`. What matters is which local action the receiving system derives from that result.

SPF checks whether a system is allowed to send for the envelope sender. DKIM checks a signature. DMARC ties these results to the visible sender domain and publishes a desired treatment. Even the current [DMARC standard RFC 9989](https://www.rfc-editor.org/rfc/rfc9989.html#section-1) explicitly states that the recipient may take this desired treatment into account, but is not obligated to. DMARC is an important signal, but it is not network access control.

With an upstream gateway, there's the added factor that Exchange Online initially sees the IP address of that gateway rather than the original sender's. That's what [Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors) is for: it reconstructs the original source and improves SPF, DKIM, DMARC, anti-spoofing, and anti-phishing evaluation. Enhanced Filtering, however, is likewise not a door lock. It does not replace the restrictive partner connector.

The misconfiguration becomes especially obvious when an administrator weakens or entirely bypasses the EOP check via an SCL bypass (on the assumption that the upstream product is already filtering) while at the same time leaving direct internet delivery open. In that case, they have not had a protection mechanism "bypassed" on them; they have deliberately left one of two entrances without effective protection.

You can certainly criticize Microsoft for letting a message land in the inbox without a warning despite a clearly visible authentication failure. You can criticize the semantics of the connector types, the documentation, and the missing warnings in the Configuration Analyzer. All of that is legitimate. But the existence of a publicly reachable SMTP endpoint is not a vulnerability.

## "Direct Send" Is Not the Same as "Direct Delivery"

Two things are being conflated in the discussion:

- **Direct Send** at Microsoft refers to anonymous messages whose envelope sender (`5321.MailFrom`) uses one of the tenant's own accepted domains.
- **Direct delivery to Exchange Online** generally refers to an SMTP message that ignores the published third-party MX and is delivered straight to the Exchange endpoint. The sender can use any arbitrary external domain here too.

The switch

```powershell
Set-OrganizationConfig -RejectDirectSend $true
```

makes sense if Direct Send is not needed. It prevents internal domain spoofing via this path. It does not, however, close the entire side entrance for arbitrary external senders. Microsoft describes the exact scope in the [cmdlet documentation for `RejectDirectSend`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-organizationconfig?view=exchange-ps#-rejectdirectsend). Anyone who wants to fully prevent "Ghost Sender" still needs the access restriction via a partner connector or a matching mail flow rule.

## Does Microsoft Really Have to Take Everything Off the Administrator's Hands?

No. Anyone who builds an additional mail filter into a production transport chain takes on responsibility for that transport chain.

The vendor cannot reliably guess whether, besides the external MX, scanners, multifunction devices, SaaS services, hybrid servers, partner relays, or other legitimate systems still need to send directly to Exchange Online. An automatic "the MX points elsewhere, so block everything else" would break desired mail flows in numerous real-world environments. That's why the administrator has to explicitly define the desired trust boundary.

Still, Microsoft could make life easier for those responsible. A good Configuration Analyzer should detect an external MX without a restrictive partner connector and warn clearly. The setup dialog could explain that a connector of type "Your organization" identifies matching connections but does not automatically reject non-matching ones. Secure-by-default switches and better operational reporting would also be welcome.

That would be sensible product hardening. It does not, however, change the technical classification: an insecure special topology remains an insecure configuration, and it does not become a zero-day just because it is widespread.

## How to Close the Side Entrance

For environments with an upstream filter, the checklist should include at least these points:

1. **Fully document the mail flow.** Which systems are actually allowed to deliver to Exchange Online? This includes hybrid, application, and emergency paths.
2. **Set up a restrictive partner connector.** Use `SenderDomains *` and restrict delivery to a certificate (preferred) or to maintained source IP ranges. A connector of type `OnPremises`, i.e. "Your organization," does not enforce this default-deny behavior.
3. **Configure Enhanced Filtering correctly.** If EOP is still supposed to filter, the original IP and sender information must be reconstructed cleanly. Blanket SCL `-1` bypasses need critical review.
4. **Disable Direct Send if unused.** Check beforehand with message trace or the available reports whether scanners or applications depend on it.
5. **Don't switch blindly.** Test gateway IP ranges, certificate changes, hybrid mail flow, as well as `onmicrosoft.com`, Teams, and other special paths, and then monitor them.

A simplified example for the IP-based variant:

```powershell
New-InboundConnector `
  -Name "Only from upstream mail gateway" `
  -ConnectorType Partner `
  -SenderDomains * `
  -RestrictDomainsToIPAddresses $true `
  -SenderIpAddresses <IP-Bereiche-des-Gateways> `
  -RequireTls $true
```

Where possible, certificate binding is preferable to an IP allowlist. Changes should first go through a controlled test, because a faulty allowlist very quickly turns the open side entrance into a complete mail outage.

## The Simple Self-Test

The test shown by InfoGuard (and MSXFAQ) is useful:

```powershell
Send-MailMessage `
  -SmtpServer <tenantname>.mail.protection.outlook.com `
  -To admin@<tenantdomain> `
  -From noreply@example.com `
  -Subject "EXO Nebeneingang" `
  -Body "Testmail direkt zum Tenant"
```

With a correctly restricted partner connector, you should expect an SMTP rejection like `5.7.51 TenantInboundAttribution; Rejecting`. An alternative transport rule can accept the message initially and then move it to quarantine; that's why, besides the SMTP response, you should also check message trace, quarantine, and the mailbox. `Send-MailMessage` (deprecated) serves here only as an easily understandable illustration. Any controlled SMTP test tool serves the same purpose.

## Conclusion: Good Test, Wrong Label

"Ghost Sender" is not a new SMTP exploit. It is a catchy name for an open side entrance whose mitigation Microsoft has documented for a long time and which the administrator left open.

The irony: InfoGuard itself calls the problem a "widespread and systematic misconfiguration" in its own post and closes with the sentence "Ghost-Sender is a misconfiguration." Microsoft's Security Response Center also did not initially classify the report as a vulnerability. So the facts are all present in the article: only the title, the test email, and the "vulnerability" branding unfortunately tell a more dramatic story.

The useful part of the publication is the wake-up call: many companies apparently haven't locked down their mail flow properly. The problematic part is the claim that Exchange Online has a universal vulnerability because of it. No: Exchange Online initially just behaves like an MTA here. What makes it insecure is the trust boundary that was never finished being configured.

Does the administrator really need everything taken off their hands? No. But apparently it still needs to be repeated again and again that DNS routing is no substitute for access control.

## Sources

1.  [InfoGuard Labs: Ghost-Sender – Universal Email Spoofing against Exchange Online](https://labs.infoguard.ch/posts/ghost-sender/) — The original research, including the prevalence analysis and its own conclusion, "Ghost-Sender is a misconfiguration."

2.  [Ghost Sender: Exchange Online Mail Spoofing Tester](https://ghost-sender.com/) — The online test published by InfoGuard to check your own tenant for the open side entrance.

3.  [MSXFAQ: Exchange Online als Nebeneingang für Mailempfang](https://www.msxfaq.de/cloud/exchangeonline/transport/exo-nebeneingang.htm) — Frank Carius' assessment: not a flaw in Exchange Online, but an administrator misconfiguration.

4.  [Microsoft: Direct Send vs sending directly to an Exchange Online tenant](https://techcommunity.microsoft.com/blog/exchange/direct-send-vs-sending-directly-to-an-exchange-online-tenant/4439865) — Microsoft explains that direct acceptance of mail to hosted mailboxes is simply how email works, and distinguishes it from Direct Send.

5.  [Microsoft Learn: Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud) — The official guide, including the dedicated step for the restrictive partner connector with an external MX.

6.  [Microsoft Learn: Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors) — Reconstructs the original sender source behind a gateway; improves evaluation but does not replace the connector.

7.  [Heise: Ghost-Sender – Exchange Online lässt gefälschte E-Mails anstandslos durch](https://www.heise.de/news/Ghost-Sender-Exchange-Online-laesst-gefaelschte-E-Mails-anstandslos-durch-11327666.html) — Example of the sensationalized coverage that generalizes only certain misconfigurations.

8.  [Crow in the Cloud: Die Geister, die ich nicht rief](https://crowinthe.cloud/die-geister-die-ich-nicht-rief-effektiver-schutz-gegen-ghost-sender-in-exchange-online/) — An apt classification as a design and configuration problem, including protective measures.

9.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321.html) — Describes the MX record as the mechanism for determining the regular target system, not as access control.

10.  [RFC 9989: DMARC](https://www.rfc-editor.org/rfc/rfc9989.html) — States that the recipient may take the published DMARC treatment into account, but is not required to.

---

## Is Your Mail Flow Secure?

Not sure whether your Exchange Online tenant also has an open side entrance? **adeptio** reviews your entire mail flow: from MX records, connectors, and third-party gateways to EOP, SPF, DKIM, DMARC, and Direct Send. Practical, independent, and with concrete recommendations.

If you'd like to have your mail flow reviewed or properly secured, feel free to schedule a no-obligation consultation:

**[Book a consultation with adeptio](https://outlook.office.com/book/Erstgesprchadeptio%40adeptio.ch/s/Akxr6wxKAEGw3d5sEmi-AQ2?ismsaljsauthenabled)**  
[adeptio.ch](https://adeptio.ch/)
