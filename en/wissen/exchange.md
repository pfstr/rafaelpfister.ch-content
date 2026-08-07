---
title: "Exchange and Microsoft 365: the mail system behind the mailboxes"
blatt: "exchange"
description: "An overview of Exchange Online, Exchange Server, and hybrid deployments: connectors and mail flow, transport rules, Autodiscover, the role of the last on-premises server, and what running the platform involves today."
fakten:
  - { label: "Product family", wert: "Exchange Online (M365), Exchange Server (on-premises), hybrid" }
  - { label: "Vendor", wert: "Microsoft", href: "https://learn.microsoft.com/en-us/exchange/" }
  - { label: "Current on-premises release", wert: "Exchange Server SE (Subscription Edition)", href: "https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates" }
  - { label: "Protocols", wert: "SMTP, HTTPS (Outlook, EWS, Graph), Autodiscover" }
  - { label: "Administration", wert: "Exchange Admin Center, Exchange Online PowerShell" }
  - { label: "Mail flow components", wert: "Connectors, transport rules, accepted domains" }
  - { label: "Directory", wert: "Entra ID or Active Directory (hybrid: Entra Connect)" }
werbung: ["tools", "newsletter"]
ctaThemen: ["microsoft-365-exchange", "exchange-onprem-hybrid"]
---

# Exchange and Microsoft 365: the mail system behind the mailboxes

In most organizations, Exchange is the center of mail traffic, whether as Exchange Online in Microsoft 365, as a self-operated Exchange Server, or as a hybrid of both. For admins, mailbox administration matters less than what lies beneath it: how mail flows in and out, where rules take effect, and which neighboring systems are attached.

## The three deployment models

**Exchange Online** is the standard case: Microsoft operates the platform, the organization manages configuration and policies. **Exchange Server** continues to run wherever data must remain in-house or line-of-business applications require local mailboxes; since the Subscription Edition, this has been a subscription model with continuous updates instead of large version jumps. **Hybrid** connects both worlds through shared namespaces and connectors. The best-known hybrid fixture: as long as identities are synchronized from the on-premises Active Directory, Microsoft requires a remaining Exchange server, or at least its management tools, for proper attribute maintenance, and this "last server" needs patching and upkeep like any other.

## Mail flow: connectors and transport rules

Mail flow into and out of Exchange Online runs through **connectors**: inbound connectors define which systems mail is accepted from (for example from an upstream secure mail gateway), outbound connectors control the path out (for example forced through a gateway or straight to the internet). Anyone operating an encryption or filtering gateway lives in this connector world, including the classic pitfalls: a forgotten direct delivery path that bypasses the gateway (a "side entrance"), or a connector that no longer matches after a partner has replaced its certificate. **Transport rules** add policy logic to the flow: disclaimers, redirects, header stamps for downstream systems, blocks. They apply centrally and are logged, which makes them the preferred tool for compliance requirements.

## Autodiscover and the client side

Clients locate their mailboxes via **Autodiscover**; the associated DNS records are part of a domain's basic mail configuration. Client access itself now runs over HTTPS (MAPI over HTTP, Exchange Web Services, increasingly Graph). Relevant for admins: legacy protocols such as Basic authentication have been disabled, IMAP/POP special cases require modern authentication, and multifunction devices that want "just SMTP" are a chapter of their own between submission, relay, and Graph.

## Operations today: policies and edges

Running a mail system has shifted from server maintenance to rule sets. What matters are retention and delegation policies, transport and compliance rules, outbound sender authentication (SPF, DKIM, DMARC) and the question of which systems may submit mail at all. Typical incidents arise at the edges: connectors to gateways and multifunction devices, expiring certificates and secrets, size and recipient limits, and vendor changes to default values that used to apply silently.
