---
title: "Exchange Online: Architecture, Mail Flow, and Operations"
blatt: "exchange-online"
description: "Exchange Online for messaging admins: tenant and recipient model, EOP transport, connectors, client access, PowerShell and Graph, Message Trace, retention, security, and recovery."
fakten:
  - label: Product role
    wert: Cloud-based email, calendar, and directory service
    href: https://learn.microsoft.com/en-us/exchange/exchange-online
  - label: Platform
    wert: Microsoft 365
    href: https://learn.microsoft.com/en-us/office365/servicedescriptions/exchange-online-service-description/exchange-online-service-description
  - label: Mail acceptance
    wert: Exchange Online Protection and SMTP
    href: https://learn.microsoft.com/en-us/defender-office-365/eop-about
  - label: Recipients
    wert: Mailboxes, groups, contacts, mail users, and resources
    href: https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/recipients-in-exchange-online
  - label: Domains
    wert: Authoritative or Internal Relay
    href: https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains
  - label: Routing
    wert: Inbound and Outbound Connectors, rules, and MX
    href: https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail
  - label: Client access
    wert: Outlook, Outlook on the web, ActiveSync, and IMAP/POP special cases
    href: https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/clients-and-mobile-in-exchange-online
  - label: Identity
    wert: Microsoft Entra ID and modern authentication
    href: https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/enable-or-disable-modern-auth-in-exchange-online
  - label: Administration
    wert: Exchange Admin Center and Exchange Online PowerShell
    href: https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell
  - label: API
    wert: Microsoft Graph for mail, calendar, and administration functions
    href: https://learn.microsoft.com/en-us/graph/api/resources/mail-api-overview
  - label: Diagnostics
    wert: Message Trace, reports, and Service Health
    href: https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message
  - label: Retention
    wert: Recoverable Items, retention, and holds
    href: https://learn.microsoft.com/en-us/purview/retention-policies-exchange
werbung:
  - tools
  - newsletter
ctaThemen:
  - microsoft-365-exchange
  - smtp-mailflow
translationSourceHash: 5965bf4a9447505ffbe8b9a5d00c6f1abf629dfbde070d9c7700d9fdc9e4b603
translationModel: gpt-5.6-terra
translatedAt: 2026-10-05T11:27:01.759Z
translationReview: automatic
---

# Exchange Online: Architecture, Mail Flow, and Operations

**Exchange Online** is the Exchange service operated by Microsoft in Microsoft 365. It provides mailboxes, calendars, contacts, groups, SMTP transport, and administrative functions. The tenant admin decides on recipients, domains, connectors, rules, permissions, and retention. Microsoft, on the other hand, operates the mailbox servers, database copies, internal queues, patches, and failover operations ([Exchange Online service description](https://learn.microsoft.com/en-us/office365/servicedescriptions/exchange-online-service-description/exchange-online-service-description), [Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency)).

This makes Exchange Online functionally similar to an organization's own Exchange system, but not operationally. A local admin can examine a queue file or activate a database copy. In Exchange Online, they instead see the events, states, and configuration objects provided by the service. The most important skill is therefore mapping a user complaint to a clear path: identity, client access, recipient object, transport, filtering, delivery, or retention.

## From tenant to mailbox

The tenant forms the organizational framework. Within it, Exchange Online manages mail-enabled recipients: user and shared mailboxes, room and equipment mailboxes, distribution groups, Microsoft 365 Groups, contacts, and mail users. The recipient type determines whether data is stored, how delivery occurs, and which permissions are available ([Recipients in Exchange Online](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/recipients-in-exchange-online)).

A user account in Microsoft Entra ID and an Exchange mailbox are related, but they are not the same object. Licensing can trigger mailbox provisioning. Exchange then adds mail-related attributes and services. If an admin removes a license or deletes an account, different retention and deletion periods apply. Identity lifecycle, mailbox lifecycle, and compliance retention must therefore be planned together for operations and offboarding ([Delete or restore user mailboxes](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/delete-or-restore-mailboxes), [Retention policies for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange)).

For experts, the origin of attributes becomes important. In a cloud-only tenant, Exchange properties are managed online. With synchronized identities, the on-premises environment may still be the authoritative source for certain recipient attributes. A value may then appear in Exchange Online but must be changed on-premises and synchronized again. This model belongs in the article [Exchange Hybrid](/kb/exchange-hybrid), because it does not exist without directory synchronization.

## How an incoming message reaches the mailbox

Once the recipient is understood, the mail path can be traced. A domain's public MX normally points to Exchange Online Protection, EOP. EOP accepts the SMTP connection, evaluates the sender and message, applies protection and transport rules, and hands an allowed message to Exchange Online. Delivery to the mailbox then follows for local recipients ([Exchange Online Protection overview](https://learn.microsoft.com/en-us/defender-office-365/eop-about), [Mail flow in EOP](https://learn.microsoft.com/en-us/defender-office-365/eop-mail-flow)).

The **Accepted Domain** specifies how Exchange Online handles the recipient domain. With `Authoritative`, the service expects all valid recipients to exist in its own organization and rejects unknown addresses. `Internal Relay` allows unknown recipients to be forwarded to another system. This setting is only useful if the next hop and recipient resolution are reliably planned; otherwise, nondelivery reports or loops result ([Manage accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains)).

An internal message does not automatically remain “on the same server.” Exchange Online resolves the sender and recipient, checks rules and protection policies, and writes transport events. This event chain is crucial for the admin: `Delivered` means that the service delivered to its destination; `Filtered`, `Failed`, `Pending` or `Expanded` describe other steps. Message Trace makes these steps visible, but does not replace checking the destination mailbox or a downstream rule ([Trace an email message](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message), [Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq)).

## Outgoing messages and connectors

For outgoing messages, Exchange Online first determines whether to send directly to the destination system or use a configured Outbound Connector. A connector can route messages to an organization's own infrastructure, a partner, or a mail gateway. Selection is based, among other things, on the recipient domain, connector conditions, and transport rules ([Set up connectors to route mail](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail)).

Conversely, Inbound Connectors describe the conditions under which Exchange Online trusts a sending system. Typical criteria are the source IP or a TLS certificate. This information is security-relevant: an overly broad IP range or an imprecisely validated certificate can make external traffic appear to be internal partner traffic.

If an external mail gateway sits in front of EOP, Microsoft initially sees the gateway's IP address. **Enhanced Filtering for Connectors** can include information about the original hop in filtering evaluation. The feature is not a general “spam filter switch,” but must fit the actual path, the connectors, and the skipped IPs ([Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)).

The expert question here is: Which endpoint actually accepted a message, which identity was checked for the connector, and at which hop did the last content-based filtering occur? These three answers belong in every mail flow diagram.

## Technical structure from an admin perspective

Exchange Online does not publish a server list that a tenant admin manages like an on-premises farm. Nevertheless, the service has clearly identifiable technical components. They are visible through protocols and management interfaces.

| Component | Function | What the tenant admin sees |
|---|---|---|
| Exchange Online Protection | SMTP acceptance, anti-malware, anti-spam, and transport processing | Quarantine, policies, reports, and Message Trace |
| Exchange transport | Recipient resolution, rules, routing, and delivery | Connectors, Accepted Domains, rules, and events |
| Mailbox service | Email, calendar, contact, and folder storage | Mailbox objects, quotas, permissions, and client access |
| Microsoft Entra ID | User, group, application, and sign-in identities | Accounts, roles, Conditional Access, and app registrations |
| Exchange Online PowerShell | Exchange-specific administration | Cmdlets, RBAC, and auditable changes |
| Microsoft Graph | REST API for applications and automation | OAuth permissions, resources, and throttling |

The technology stack at the edge therefore mainly consists of SMTP and TLS for mail transport, plus HTTPS, OAuth, PowerShell, and REST for client and administrative access. Internal implementation details are relevant to the customer only insofar as Microsoft documents them as service behavior, a limit, or a diagnostic interface ([About the Exchange Online PowerShell module](https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell-v2), [Microsoft Graph mail API](https://learn.microsoft.com/en-us/graph/api/resources/mail-api-overview)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1056" src="/images/kb-interaktiv-exchange-online.svg?v=20260813" title="Interaktive Infografik: Exchange-Online-Pfad von DNS und EOP über Transport und Postfach bis Entra, PowerShell, Graph und Betriebsnachweis" loading="lazy">
  <a href="/images/kb-interaktiv-exchange-online.svg?v=20260813">Open the interactive Exchange Online graphic directly</a>.
</iframe>

## Client access and modern authentication

Mail transport ends at the mailbox; users then access it through client protocols. Outlook, Outlook on the web, mobile clients, and applications use HTTPS-based endpoints. Autodiscover helps clients find the appropriate service. Sign-in occurs through Microsoft Entra ID, while Exchange checks authorization on the mailbox ([Clients and mobile in Exchange Online](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/clients-and-mobile-in-exchange-online), [Modern authentication in Exchange Online](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/enable-or-disable-modern-auth-in-exchange-online)).

This separates two commonly conflated errors. If sign-in fails in Entra, the client often never reaches Exchange. If the token is valid, Exchange can still deny access because of a missing role, mailbox permission, client policy, or incorrect target mailbox. Sign-in logs and Exchange diagnostics must therefore be considered together in time.

Applications should preferably access through Microsoft Graph or supported Exchange interfaces. A Graph application permission can apply broadly; Exchange RBAC for Applications can define a narrower accessible mailbox scope. A valid OAuth token is therefore only the first step. The resource service then checks which action is allowed on which mailbox ([Role Based Access Control for Applications](https://learn.microsoft.com/en-us/exchange/permissions-exo/application-rbac)).

## Tracking permissions and changes

Exchange Online has its own administrative roles. Entra roles can enable entry to Exchange administration, but the actual Exchange cmdlets and their scope are determined by Exchange RBAC ([Permissions in Exchange Online](https://learn.microsoft.com/en-us/exchange/permissions-exo/permissions-exo)).

There are also mailbox permissions such as Full Access, Send As, and Send on Behalf. They control different actions and should not be inventoried as a single “delegation right.” OAuth and Exchange application roles are added for applications ([Manage permissions for recipients](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/manage-permissions-for-recipients)).

For experts, the origin of a change is just as important as the end state. Audit logs, Entra sign-in logs, and configuration exports answer who changed a rule, connector, or permission. A nightly export of central mail flow objects makes comparisons easier, but does not replace a protected audit source.

## Diagnostics: DNS first, then transport events

A mail flow analysis begins outside the tenant. The MX indicates which system accepts Internet mail. Message Trace is then used to check whether Exchange Online saw the specific message and how it processed it.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für MX- und Autodiscover-Abfrage">
    <button type="button" role="tab" data-os-tab="windows" aria-selected="true">Windows</button>
    <button type="button" role="tab" data-os-tab="unix" aria-selected="false">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName -Type MX example.com
Resolve-DnsName autodiscover.example.com
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig MX example.com
dig autodiscover.example.com
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) and [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) show publication and resolution. They do not yet say whether EOP accepted the message or whether a mailbox received it.

For the next step, select a narrow time range with sender and recipient. The same Exchange Online PowerShell runs on Windows and with `pwsh` on supported Unix systems.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Exchange Online PowerShell">
    <button type="button" role="tab" data-os-tab="windows" aria-selected="true">Windows</button>
    <button type="button" role="tab" data-os-tab="unix" aria-selected="false">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Connect-ExchangeOnline
Get-MessageTraceV2 -SenderAddress sender@example.net `
  -StartDate (Get-Date).AddHours(-2) -EndDate (Get-Date)
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```powershell
pwsh
Connect-ExchangeOnline
Get-MessageTraceV2 -SenderAddress sender@example.net `
  -StartDate (Get-Date).AddHours(-2) -EndDate (Get-Date)
```

  </div>
</div>

[`Connect-ExchangeOnline`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/connect-exchangeonline) establishes the authenticated administration session. [`Get-MessageTraceV2`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2) searches transport events; [`Get-Date`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-date) limits the time window. Reports are added for trends, and Service Health for Microsoft outages. A single green signal does not answer all three questions ([Exchange Online monitoring](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-exchange-monitoring?view=o365-worldwide)).

## Retention, deletion, and recovery

Microsoft protects the running service with multiple database copies, Shadow Redundancy, and Safety Net. These mechanisms serve the availability and data integrity of the service. They are not the user interface for recovering an accidentally deleted message ([Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency)).

Other functions apply to user and compliance cases: Deleted Item Retention, Recoverable Items, Single Item Recovery, Retention Policies, and Holds. Their effects overlap, but they serve different purposes. A retention rule can protect content from permanent deletion; it does not automatically provide a separate backup independent of the tenant with a freely selectable recovery point ([Recoverable Items folder](https://learn.microsoft.com/en-us/exchange/security-and-compliance/recoverable-items-folder/recoverable-items-folder), [Retention policies for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange)).

A robust recovery concept therefore documents which events are covered by Microsoft's service resiliency, which content can be recovered through Exchange or Purview retention, and which requirements need an independent copy. Restore tests should use specific cases: an individual message, a folder, a mailbox after user deletion, a legally retained item, and a tenant-wide outage.

## Security and typical limitations

Exchange Online combines several security areas: Internet mail, EOP, tenant configuration, Entra sign-in, mailbox rights, and applications. The protective effect depends on the actual message and sign-in path matching the configuration.

For mail flow, this means MX, connector identity, Enhanced Filtering, SPF/DKIM/DMARC, and transport rules must be checked as a chain. For client access, modern authentication, Conditional Access, Exchange RBAC, and mailbox permissions are separate controls. OAuth consent and the permitted mailbox scope are added for applications.

The deeper admin question is always the same: Which system made the decision, which input data did it see, and where is the result logged? Without these three details, even a formally correct policy remains difficult to verify.

## Technical evolution and deliberate trade-offs

Exchange Online evolved from Microsoft's hosted Exchange offerings and adopted many concepts from the server product: recipients, mailbox databases, transport, DAGs, Shadow Redundancy, and Safety Net. The service automates operation of this infrastructure and provides tenant admins with a higher administrative layer ([Exchange Team: 20 years ago](https://techcommunity.microsoft.com/blog/exchange/20-years-ago-in-a-galaxy-far-away8230/604456), [Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency)).

The benefit is outsourced platform operations, global service integration, and standardized management interfaces. The cost is less access to individual servers, queues, and database copies, as well as greater dependence on published diagnostic, export, and recovery functions. The task for experts is therefore not to guess the invisible internal topology, but to fully use the documented tenant controls and service signals.

## Sources

- [Microsoft Learn – Exchange Online](https://learn.microsoft.com/en-us/exchange/exchange-online)
- [Microsoft – Exchange Online service description](https://learn.microsoft.com/en-us/office365/servicedescriptions/exchange-online-service-description/exchange-online-service-description)
- [Microsoft Defender – Exchange Online Protection overview](https://learn.microsoft.com/en-us/defender-office-365/eop-about)
- [Microsoft Defender – Mail flow in EOP](https://learn.microsoft.com/en-us/defender-office-365/eop-mail-flow)
- [Microsoft Learn – Recipients in Exchange Online](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/recipients-in-exchange-online)
- [Microsoft Learn – Delete or restore user mailboxes](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/delete-or-restore-mailboxes)
- [Microsoft Learn – Accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains)
- [Microsoft Learn – Set up connectors to route mail](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail)
- [Microsoft Learn – Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)
- [Microsoft Learn – Trace an email message](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message)
- [Microsoft Learn – Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq)
- [Microsoft Learn – Clients and mobile in Exchange Online](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/clients-and-mobile-in-exchange-online)
- [Microsoft Learn – Modern authentication in Exchange Online](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/enable-or-disable-modern-auth-in-exchange-online)
- [Microsoft Learn – Exchange Online PowerShell](https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell)
- [Microsoft Learn – About the Exchange Online PowerShell module](https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell-v2)
- [Microsoft Graph – Mail API overview](https://learn.microsoft.com/en-us/graph/api/resources/mail-api-overview)
- [Microsoft Learn – RBAC for Applications](https://learn.microsoft.com/en-us/exchange/permissions-exo/application-rbac)
- [Microsoft Learn – Permissions in Exchange Online](https://learn.microsoft.com/en-us/exchange/permissions-exo/permissions-exo)
- [Microsoft Learn – Manage permissions for recipients](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/manage-permissions-for-recipients)
- [Microsoft Service Assurance – Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency)
- [Microsoft Learn – Recoverable Items folder](https://learn.microsoft.com/en-us/exchange/security-and-compliance/recoverable-items-folder/recoverable-items-folder)
- [Microsoft Purview – Retention policies for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange)
- [Microsoft Learn – Exchange Online monitoring](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-exchange-monitoring?view=o365-worldwide)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [BIND 9 – dig manual](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility)
- [Microsoft Learn – Connect-ExchangeOnline](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/connect-exchangeonline)
- [Microsoft Learn – Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2)
- [Microsoft Learn – Get-Date](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-date)
- [Exchange Team – 20 years ago](https://techcommunity.microsoft.com/blog/exchange/20-years-ago-in-a-galaxy-far-away8230/604456)
