---
title: "Hybrid Mail Flow: Routing Between Exchange Online and On-Premises"
blatt: "hybrid-mailfluss"
description: "Hybrid mail flow step by step: direct and centralized delivery, connectors, TLS certificates, remote domains, shared SMTP domains, mail gateways, Message Trace, and Message Tracking."
fakten:
  - label: Purpose
    wert: SMTP routing between Exchange Online and the on-premises Exchange organization
    href: https://learn.microsoft.com/en-us/exchange/transport-routing
  - label: Setup
    wert: Hybrid Configuration Wizard creates and maintains the transport configuration
    href: https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard
  - label: Transport
    wert: SMTP over TCP 25 with TLS
    href: https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites
  - label: Peer validation
    wert: Certificate name and connector conditions
    href: https://learn.microsoft.com/en-us/exchange/hybrid-deployment/hybrid-mail-flow
  - label: Cloud connectors
    wert: Inbound and outbound connectors
    href: https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail
  - label: Local connectors
    wert: Receive and Send Connectors
    href: https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors
  - label: Recipient routing
    wert: Remote Mailbox, Target Address, and coexistence domain
    href: https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox
  - label: Standard routing
    wert: Cloud and on-premises organizations can each send Internet mail directly
    href: https://learn.microsoft.com/en-us/exchange/transport-routing
  - label: Centralized Mail Transport
    wert: Internet mail from Exchange Online flows through the on-premises organization
    href: https://learn.microsoft.com/en-us/exchange/transport-routing
  - label: External gateways
    wert: Additional connector and filtering chain before or after Microsoft 365
    href: https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud
  - label: Cloud diagnostics
    wert: Message Trace and connector validation
    href: https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message
  - label: Local diagnostics
    wert: Message Tracking, queues, and protocol logs
    href: https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking
werbung:
  - tools
  - newsletter
ctaThemen:
  - exchange-onprem-hybrid
  - smtp-mailflow
translationSourceHash: 442ec532c64e034403d3449ffe1b46d4b6de05a2337d3c3210da2d83cb5f618f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-07T11:01:21.753Z
translationReview: automatic
---

# Hybrid Mail Flow: Routing Between Exchange Online and On-Premises

**Hybrid mail flow** is the SMTP path between an on-premises Exchange organization and Exchange Online. It enables mailboxes on both sides to use the same SMTP domain while ensuring that messages still arrive at the correct location. The Hybrid Configuration Wizard configures connectors and TLS parameters for this purpose ([Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing), [Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)).

Mail flow is only one part of Exchange Hybrid. Directory synchronization, free/busy, OAuth, and mailbox moves use other paths. This article therefore deliberately focuses on one question: **Which SMTP hops does a specific message traverse, and what decision is made at each hop?**

The **protocol stack** is straightforward: DNS identifies publicly reachable destinations, SMTP over TCP 25 transports the message, TLS protects and identifies the connection, and Exchange connectors determine which peer is used for which domain. Recipient objects provide the routing address; Message Trace and local tracking logs subsequently show what each organization did with the message ([Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing), [Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)).

## The basic model: two Exchange organizations, one address space

A hybrid environment has at least two transport organizations. The on-premises Exchange organization knows on-premises mailboxes and remote mailbox objects. Exchange Online knows cloud mailboxes and synchronized representations of on-premises recipients. Both sides can use the same primary domain, such as `example.com` ([Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)).

To prevent a message from ending up in the wrong location, each side needs an indication of the recipient's actual location. For a cloud mailbox, the on-premises remote mailbox object contains a remote routing address in the coexistence domain, typically `tenant.mail.onmicrosoft.com`. Conversely, Exchange Online knows synchronized on-premises recipients as mail-enabled objects ([Enable-RemoteMailbox](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox)).

The normal process is therefore simple:

1. The first Exchange organization accepts the message.
2. It resolves the recipient in its directory.
3. The recipient object indicates whether the mailbox is on-premises or on the other side.
4. The appropriate hybrid connector sends it to the other organization via SMTP/TLS.
5. There, the recipient is resolved again and the message is delivered.

Experts additionally check whether transport rules alter the route, whether a gateway is inserted, and which domain or connector priority explains the selected next hop.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-hybrid-mailfluss.svg?v=20260813" title="Interaktive Infografik: direkter und zentraler Hybrid-Mailfluss zwischen Internet, Exchange Online, Exchange On-Premises und Mail-Gateway" loading="lazy">
  <a href="/images/kb-interaktiv-hybrid-mailfluss.svg?v=20260813">Open the interactive hybrid mail flow diagram directly</a>.
</iframe>

## The standard route without centralized Internet transport

In the usual decentralized model, each side sends its own Internet mail. An on-premises mailbox uses the on-premises Exchange transport organization for outbound Internet mail. A cloud mailbox sends through Exchange Online Protection. Only messages between on-premises and cloud mailboxes traverse the hybrid connectors ([Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing)).

Inbound Internet mail follows the published MX record. If the MX record points to Exchange Online, EOP accepts the message first. For a cloud mailbox, Exchange Online delivers locally; for a synchronized on-premises recipient, it uses the hybrid outbound connector. If the MX record instead points to the on-premises environment or an upstream gateway, the first recipient decision takes place there.

This model keeps Internet paths short, but results in multiple possible egress IPs and filtering locations. SPF, DKIM, DMARC, allowlisting, and partner rules must account for both outbound paths. This is not a flaw in the hybrid model, but a consequence of distributed delivery.

## Deliberately classifying Centralized Mail Transport

**Centralized Mail Transport**, CMT, changes precisely this outbound path. Messages from Exchange Online mailboxes to the Internet are first sent to the on-premises Exchange organization. They leave the organization only there. This allows the on-premises side to continue using centralized transport rules, appliances, or fixed egress IPs ([Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing)).

The advantage is shared outbound control. The cost is additional hops and dependencies. If on-premises transport or its Internet connectivity fails, outbound cloud mail is now affected as well. Latency, queue location, egress IP, and the location of final filtering change.

For advanced admins, the decision is therefore not “CMT on or off,” but: Which specific policy requires the on-premises hop, what capacity must it support, and how is routing handled during an outage? Experts also document loop protection, TLS requirements, connector priority, and proof that every intended message actually takes the centralized path.

This concludes the CMT topic. Outlook client authentication or Hybrid Modern Authentication do not belong here because they do not select an SMTP hop.

## How connectors identify the peer

Once the route is selected, each side must be able to trust the peer. The Hybrid Configuration Wizard creates on-premises Send and Receive Connectors and matching inbound and outbound connectors in Exchange Online. Transport uses SMTP on TCP 25 and TLS ([Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites), [Hybrid mail flow](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/hybrid-mail-flow)).

The on-premises Send Connector determines the destination and TLS requirements for the coexistence domain. The cloud outbound connector defines the on-premises organization as the destination. In the opposite direction, the Receive or inbound connector accepts traffic based on documented conditions, including certificate identity and source.

The certificate has a specific role: it identifies the SMTP endpoint in the TLS handshake. The subject or Subject Alternative Name, connector parameters, presented certificate chain, and actual hostname must match. A valid certificate in the certificate store is insufficient if the transport service presents a different one.

Experts therefore check both directions separately. Direction A can work while direction B fails because of a different connector, DNS destination, or certificate name.

## Recipient routing and shared domains

A functioning connector does not yet determine which messages use it. This decision starts with the recipient object. An on-premises remote mailbox points to the cloud. A synchronized on-premises mailbox object in Exchange Online points back to the on-premises organization.

Accepted Domains also determine whether an organization is responsible for all recipients in a domain or may relay unknown recipients. In shared domains, an Internal Relay configuration is safe only if the next hop correctly recognizes or rejects unknown recipients ([Accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains), [Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)).

An outdated remote mailbox object can therefore cause misrouting despite healthy TLS. Conversely, a correct Target Address does not help if the cloud connector is disabled. Diagnosis connects the object and transport instead of considering only one side.

For experts, forwarding, mail contacts, distribution group expansion, and transport rules become important. They can change the original recipient address or create additional recipients. Each resulting message receives its own routing decision.

## Mail gateways before or after Exchange Online

Many organizations supplement hybrid with a secure email gateway or cloud filtering platform. This adds at least one more SMTP hop. The path must be diagrammed separately for inbound and outbound messages ([Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud)).

If the gateway is before Exchange Online, the MX record points to the gateway. EOP then initially sees its source IP. Enhanced Filtering for Connectors can include the original sender information in Microsoft filtering evaluation when the connector and skipped IPs are configured correctly ([Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)).

For outbound mail, it must be clear whether Exchange Online sends directly, through the gateway, or with CMT first on-premises and then through the gateway. Multiple permitted paths can bypass policies and create different DKIM signatures, egress IPs, and logs.

The expert control is an allowed path graph: Each arrow identifies the initiator, destination, port, TLS verification, permitted domains, filtering function, queue owner, and log source. A gateway name without this information is not yet an architecture.

## Tracing a message end to end

Troubleshooting starts with a test message whose sender, recipient, and time are known. First, the public path is checked, then the events on each involved Exchange side.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für DNS- und SMTP-Test">
    <button type="button" role="tab" data-os-tab="windows" aria-selected="true">Windows</button>
    <button type="button" role="tab" data-os-tab="unix" aria-selected="false">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName -Type MX example.com
Test-NetConnection mail.example.com -Port 25
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig MX example.com
nc -vz mail.example.com 25
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) and [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) show the published MX destination. [`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) and [`nc`](https://man.openbsd.org/nc) check whether TCP 25 is reachable from the measurement point. This does not yet prove successful TLS or connector validation.

Next comes the SMTP handshake. OpenSSL can be used on both administrative platforms.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-STARTTLS-Test">
    <button type="button" role="tab" data-os-tab="windows" aria-selected="true">Windows</button>
    <button type="button" role="tab" data-os-tab="unix" aria-selected="false">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
openssl s_client -starttls smtp -connect mail.example.com:25 `
  -servername mail.example.com -showcerts
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
openssl s_client -starttls smtp -connect mail.example.com:25 \
  -servername mail.example.com -showcerts
```

  </div>
</div>

[`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) shows the certificate chain, names, and TLS negotiation. For a complete authorized test dialog, [`swaks`](https://jetmore.org/john/code/swaks/) is suitable. Production messages are not tested using fabricated senders; the test identity and expected route are defined in advance.

In Exchange Online, [`Get-MessageTraceV2`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2) provides the cloud events. On-premises, [`Get-MessageTrackingLog`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog) shows processing on Exchange servers, and [`Get-Queue`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue) shows pending next hops. Timestamps are converted to a common time zone; the Internet Message ID and Network Message ID help link the sections ([Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq), [Message tracking](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking)).

## Common failure patterns without changing topics

A **TLS error** is initially treated as a transport issue: Which host was connected, which certificate did it present, and which connector condition did the peer expect? Recipient synchronization becomes relevant only if the message is misrouted after successful acceptance.

An **NDR for an unknown recipient**, on the other hand, leads first to the recipient object and the Accepted Domain type. Only when the object is correct is it checked whether the selected route transports it to the correct side.

A **pending queue** requires the next hop, retry time, and `LastError`. The open port of the destination host is useful only as the next test. A **loop** appears in repeated `Received` headers, hops, or tracking events and usually occurs when both sides relay unknown recipients to each other ([RFC 5321: Trace information and loop detection](https://datatracker.ietf.org/doc/html/rfc5321#section-6.3)).

A **connection that works in only one direction** is not a contradiction. The reverse directions use different senders, connectors, and certificate validations. They are captured and tested separately.

## Security, operations, and changes

Hybrid SMTP opens a deliberately permitted transport path. This path should be limited to documented source and destination systems, certificates, and domains. Open relays, overly broad IP ranges, or connectors that classify every message as trusted conflict with this model.

Certificate changes are planned like routing changes. Before expiration, the new certificate, service assignment, presented chain, and connector expectation are checked on both sides. This is followed by test messages in both directions and a controlled rollback plan.

For ongoing operations, at minimum monitor hybrid connectors, certificate expiration, queue growth, Message Trace errors, DNS destinations, and gateway health. With Centralized Mail Transport, add the capacity of the on-premises outbound path.

## Backup and rebuilding the mail path

During an outage, SMTP messages reside in queues of the systems responsible at the time. A configuration backup does not restore these pending messages. Configuration and transport state are therefore considered separately.

Rebuilding includes connector parameters on both sides, Accepted and Remote Domains, transport rules, public DNS entries, certificates with private keys, gateway configuration, and HCW selections. Secrets are stored securely; readable exports document structure and dependencies.

After rebuilding, do not test only a port. A marked message follows the expected path in each direction. Message Trace, local tracking logs, gateway logs, and the destination mailbox confirm every hop. Only this end-to-end acceptance shows that routing and filtering are correct again.

## Technical evolution and trade-offs

Over multiple Exchange generations, the Hybrid Configuration Wizard automated the connection to Exchange Online. Transport remained SMTP/TLS while cloud connectors, supported certificates, and routing options continued to evolve ([Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)).

Direct Internet transport keeps paths short and uses each platform where the mailbox resides. Centralized Mail Transport centralizes control but makes cloud mail dependent on the on-premises outbound path. A third-party gateway adds specialized filtering or encryption, but increases the number of hops and log sources. The right choice follows from a verifiable requirement, not from a desire for everything in the diagram to pass through the same box.

## Sources

- [Microsoft Learn – Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing)
- [Microsoft Learn – Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)
- [Microsoft Learn – Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)
- [Microsoft Learn – Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)
- [Microsoft Learn – Hybrid mail flow](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/hybrid-mail-flow)
- [Microsoft Learn – Enable-RemoteMailbox](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox)
- [Microsoft Learn – Set up connectors to route mail](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail)
- [Microsoft Learn – Connectors on Exchange servers](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors)
- [Microsoft Learn – Accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains)
- [Microsoft Learn – Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)
- [Microsoft Learn – Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud)
- [Microsoft Learn – Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)
- [Microsoft Learn – Trace an email message](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message)
- [Microsoft Learn – Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq)
- [Microsoft Learn – Message tracking](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking)
- [Microsoft Learn – Queues in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues)
- [Microsoft Learn – Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2)
- [Microsoft Learn – Get-MessageTrackingLog](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog)
- [Microsoft Learn – Get-Queue](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [BIND 9 – dig manual](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc(1)](https://man.openbsd.org/nc)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Swaks – SMTP test tool](https://jetmore.org/john/code/swaks/)
- [RFC 5321 – Trace information and loop detection](https://datatracker.ietf.org/doc/html/rfc5321#section-6.3)
