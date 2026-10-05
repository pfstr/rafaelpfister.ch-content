---
title: "Exchange Hybrid: Identity, Coexistence, and Operations"
blatt: "exchange-hybrid"
description: "A clear overview of Exchange Hybrid: prerequisites, directory synchronization, recipient authority, Hybrid Configuration Wizard, OAuth, organization relationships, Autodiscover, mailbox moves, and operations."
fakten:
  - label: Purpose
    wert: Coexistence of Exchange Server and Exchange Online
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: Shared namespace
    wert: Mailboxes on both sides can use the same SMTP domains
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: Directory sync
    wert: Microsoft Entra Connect Sync or Cloud Sync according to the supported model
    href: https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites
  - label: Configuration tool
    wert: Hybrid Configuration Wizard
    href: https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard
  - label: On-premises configuration
    wert: HybridConfiguration object in Active Directory
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: Cloud configuration
    wert: Connectors, organization relationships, and OAuth trust
    href: https://learn.microsoft.com/en-us/exchange/hybrid-deployment/deploy-hybrid
  - label: Recipient model
    wert: Remote Mailbox on-premises, Exchange Online mailbox in the cloud
    href: https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox
  - label: Coexistence features
    wert: Free/Busy, MailTips, archive, search, and mailbox moves depending on configuration
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: Mailbox migration
    wert: Mailbox Replication Service and migration endpoints
    href: https://learn.microsoft.com/en-us/exchange/mailbox-migration/migrate-mailboxes-across-tenants
  - label: Mail transport
    wert: Certificate-based SMTP/TLS between the two organizations
    href: https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites
  - label: Recipient management
    wert: Exchange Management Tools or supported cloud SoA transfer
    href: https://learn.microsoft.com/en-us/exchange/manage-hybrid-exchange-recipients-with-management-tools
  - label: Diagnostics
    wert: HCW log, Entra sync status, OAuth, organization, and transport configuration
    href: https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard
werbung:
  - tools
  - newsletter
ctaThemen:
  - exchange-onprem-hybrid
  - microsoft-365-exchange
translationSourceHash: c35b1646509133dd8975f96d030b2990225af14469d14a61e4fd16a4e2737f08
translationModel: gpt-5.6-terra
translatedAt: 2026-10-05T11:18:10.430Z
translationReview: automatic
---

# Exchange Hybrid: Identity, Coexistence, and Operations

**Exchange Hybrid** connects an on-premises Exchange organization with Exchange Online. Users can have mailboxes on either side while still using the same SMTP domains, a shared address book, and selected cross-organization features. Hybrid is therefore more than a pair of connectors: it connects directory data, recipients, authentication, Autodiscover, calendar features, migration, and mail transport ([Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)).

First, clarify three questions: **Where is the mailbox located?** **Where is its recipient object managed?** And **which service performs the requested operation?** Once these three answers are clear, the many hybrid components become a traceable chain.

## What Hybrid Brings Together for Users

Without Hybrid, the on-premises Exchange organization and Exchange Online are two separate systems. Hybrid adds a shared user experience on top. Mailboxes can use the same primary SMTP domain. Address book information is synchronized. Free/Busy queries and MailTips can work across organizations. Mailboxes can be moved with supported remote moves ([Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)).

However, these features do not share a single common data store. An on-premises mailbox remains in an on-premises ESE database; a cloud mailbox remains in Exchange Online. Active Directory and Entra ID each maintain directory objects. Organization relationships and OAuth allow selected queries across the boundary. SMTP connectors transport messages. The visible “single Exchange” is created by coordinated connections.

For administrators, this leads to an important rule: successful mail flow does not prove that Free/Busy works, and a successful Free/Busy query does not prove that a remote move is possible. Every function has its own path and evidence.

## The Components in a Logical Order

A hybrid deployment starts with its prerequisites, not with the wizard. The on-premises Exchange organization must be on a supported version. Public names, certificates, DNS, HTTPS and SMTP reachability must be correct. A Microsoft 365 tenant with Exchange Online and supported directory synchronization then connect the identities ([Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)).

The **Hybrid Configuration Wizard**, HCW, builds on this. It reads the desired configuration, writes a `HybridConfiguration` object to on-premises Active Directory, and configures appropriate settings on-premises and in Exchange Online. These can include organization relationships, OAuth, Intra-Organization Connectors, and transport connectors ([Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard), [Create a hybrid deployment](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/deploy-hybrid)).

| Component | Primary purpose | What to check first when there is an issue |
|---|---|---|
| Active Directory | On-premises user and Exchange attributes | Object, recipient type, proxy addresses, and time of change |
| Entra synchronization | Transfers supported identity and recipient attributes | Export errors, synchronization status, and cloud object |
| Exchange Online | Cloud mailbox and cloud configuration | Recipient type, license, mailbox status, and RBAC |
| HCW configuration | Aligns the two Exchange organizations | HCW log, selected parameters, and objects changed later |
| Organization relationship and OAuth | Cross-organization features | Target URI, Autodiscover, certificates, and token flow |
| SMTP connectors | Messages between both sides | Certificate name, source/target host, TLS, and message trace |

The table also shows why “run HCW again” is not a universal repair. The wizard can realign documented hybrid objects. It does not fix an incorrect DNS zone, a blocked firewall path, or an incorrectly maintained recipient object.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-exchange-hybrid.svg?v=20260813" title="Interaktive Infografik: Exchange-Hybrid-Verbindungen für Verzeichnissync, Empfänger, HCW, OAuth, Frei-Gebucht, Mailboxverschiebung und SMTP" loading="lazy">
  <a href="/images/kb-interaktiv-exchange-hybrid.svg?v=20260813">Open the interactive Exchange Hybrid diagram directly</a>.
</iframe>

## Technology Stack: Protocols and Management Tools

Hybrid is not an additional Exchange server process, but a connection between existing systems. Active Directory and Entra ID maintain identities and recipient attributes. Entra synchronization transfers supported values. HTTPS carries Autodiscover, Free/Busy, OAuth-protected service calls, and mailbox moves. SMTP with TLS transports messages. PowerShell, Exchange Admin Center, and HCW manage the involved objects ([Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites), [Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)).

This separation also determines the troubleshooting sequence. Search for an object issue in the directory and synchronization, a calendar issue in the HTTPS/OAuth path, and a mail issue in SMTP and connectors. This keeps the toolset tied to the affected function.

## Directory Synchronization and Recipient Authority

Once the platforms are connected, the source of recipient data becomes the most important operational question. In classic hybrid environments, a user is created in on-premises Active Directory. Exchange tools write the mail-related attributes. Entra Connect synchronizes the object to the cloud, where Exchange Online provisions the corresponding cloud object and, if applicable, a mailbox ([Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)).

A **Remote Mailbox** is an on-premises mail-enabled object that points to an Exchange Online mailbox. Attributes such as `remoteRoutingAddress`, `proxyAddresses`, and the recipient type help the on-premises organization route messages and management to the cloud side. [`Enable-RemoteMailbox`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox) creates or enables this on-premises representation; the cloud mailbox is created only through synchronization and licensing.

The usual admin question is therefore: Where do I need to change this value? The expert question is: Which system is authoritative for **this individual attribute**, and which synchronization run transfers it? A cloud portal can display a synchronized value without allowing it to be edited permanently.

Microsoft supports scenarios where only the Exchange Management Tools remain for on-premises recipient attributes. Certain environments also have a process for transferring Exchange attribute management to the cloud. These are different operating models with prerequisites; turning off the last server alone does not transfer data authority ([Manage recipients with Exchange Management Tools](https://learn.microsoft.com/en-us/exchange/manage-hybrid-exchange-recipients-with-management-tools), [Decommission after Source of Authority transfer](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/decommission-last-exchange-server)).

## Autodiscover and the Client Path

When recipients are correct, a client must find the location of the mailbox. Autodiscover answers this question. On-premises Exchange endpoints can redirect a client with a cloud mailbox to Exchange Online; cloud endpoints provide the settings for the online mailbox ([Autodiscover in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)).

A hybrid Autodiscover issue therefore often appears as an incorrect location: the user can sign in in principle, but reaches the on-premises endpoint, receives an unexpected redirect, or gets settings for a mailbox that no longer exists. DNS, SCPs, virtual directories, certificates, and recipient attributes are checked in that order.

Only after the client has reached the correct mailbox service do protocol and permission questions make sense. This keeps diagnosis understandable: first locate, then sign in, then authorize.

## Free/Busy and Other Cross-Organization Features

A shared address book is not enough for calendar queries. Free/Busy requires organization relationships, reachable Autodiscover, and a working trust or OAuth configuration. Exchange queries information from the other side rather than fully copying calendar data into its own system ([Sharing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/sharing/sharing)).

The same basic pattern applies to other hybrid features: an on-premises component makes a request, the other side authenticates it, authorizes the operation, and returns a limited result. During troubleshooting, document the source mailbox, target mailbox, direction, and endpoint. “Free/Busy does not work” is too broad without this information.

For experts, tokens and target URIs become relevant. HCW configures cross-organization relationships, but certificate changes, manual modifications, or outdated endpoints can disrupt later operations. Always export the configuration from both sides together.

## OAuth Between Exchange Organizations

Once it is clear which cross-organization queries take place, their authentication can be categorized. Exchange can use OAuth so that one organization presents a service call to the other organization. This affects hybrid features such as cross-organization availability and selected archive, search, or migration operations; the exact use depends on the version and configuration ([Configure OAuth authentication](https://learn.microsoft.com/en-us/exchange/configure-oauth-authentication-between-exchange-and-exchange-online-organizations-exchange-2013-help)).

The token flow is not a replacement for SMTP TLS. OAuth protects application calls, while hybrid mail transport uses its own connectors and certificate validation. This separation prevents the leap that is confusing in many explanations: first determine the function, then its protocol, and only then its authentication.

For experts, AuthConfig, AuthServer, PartnerApplication, Intra-Organization Connector, and Organization Relationship belong in a shared assessment. An individual object can be syntactically present while the certificate, realm, or target URI no longer matches the other side.

## Hybrid Modern Authentication Is a Separate Client Topic

**Hybrid Modern Authentication**, HMA, comes only now because it does not explain mail transport or recipient synchronization. HMA enables supported on-premises Exchange and Skype for Business resources to use Microsoft Entra ID for modern client authentication. The client receives an Entra token and uses it with the on-premises service ([Hybrid modern authentication overview](https://learn.microsoft.com/en-us/microsoft-365/enterprise/hybrid-modern-auth-overview)).

HMA therefore adds a cloud dependency to client access. Entra reachability, published URLs, registered Service Principal Names, and the on-premises Exchange configuration must align. Working hybrid mail flow says nothing about this token path.

Experts therefore handle HMA in a separate runbook with supported versions, exclusions, rollout groups, and a rollback plan. The feature is not casually attached to a routing option.

## Mailbox Moves

Coexistence is often set up to move mailboxes gradually. A remote move copies mailbox data through the Mailbox Replication Service, catches up with changes, and switches the mailbox to the target side in a controlled manner. Recipient attributes and routing are carried over or adjusted in the process ([Move mailboxes between on-premises and Exchange Online](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/move-mailboxes)).

For advanced administrators, the process consists of preparation, start, synchronization, completion, and follow-up verification. Before completion, check data volume, failed items, delegations, archives, client access, and mail flow. After completion, Autodiscover, licensing, target address, and the on-premises remote mailbox object must align.

Experts plan batch sizes, network throughput, MRS throttling, bad item limits, large items, and reverse migration. The technical progress value alone is not acceptance; user access, delegation, mobile clients, and cross-organization features are part of it.

## Hybrid Mail Flow Remains a Separate Path

Hybrid requires SMTP between the on-premises organization and Exchange Online. This mail path uses connectors, TLS, and certificates. It is important enough for a separate article because Internet mail, Centralized Mail Transport, mail gateways, and shared domains create several variants ([Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)).

The article [Hybrid mail flow](/kb/hybrid-mailfluss) starts with a specific message and traces every hop. Only there are Centralized Mail Transport, egress IP, filtering location, and additional queues compared. This article focuses on identity and coexistence.

## Security and Operations

Hybrid expands the reachable systems. Public HTTPS and SMTP endpoints, certificates, Entra synchronization, privileged accounts, and cross-organization trust objects must be inventoried together. HCW requires extensive permissions on both sides; its use and logs must be protected and stored in a traceable manner ([Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)).

In day-to-day operations, every hybrid feature should have an owner and a test: recipient synchronization, Free/Busy in both directions, remote move, Autodiscover, and SMTP in both directions. A regular synthetic test detects expired certificates or silently changed endpoints earlier than a migration project.

When issues occur, a shared timeline helps. Entra synchronization events, the HCW log, Exchange event logs, OAuth tests, message tracking, and message trace are not collected indiscriminately, but assigned to the affected function. This shortens diagnosis and prevents a successful test of another function from being misunderstood as proof.

## Backup, Rebuild, and Decommissioning

Mailbox data is protected on the side where it resides: on-premises databases with on-premises recovery, cloud mailboxes with Exchange Online and Purview features. The connection must also be recoverable. This includes the on-premises HybridConfiguration object, certificates and private keys, connector and organization configuration, Entra synchronization rules, and documented HCW decisions.

A rebuild starts with identity and name resolution, followed by HTTPS and SMTP reachability, HCW configuration, and functional tests. The wizard can recreate configuration, but without matching certificates, DNS, and recipient objects, no working overall system is created.

For decommissioning, first determine which hybrid features are still being used. Microsoft distinguishes between a remaining server, management tools only, and transferring Exchange attribute management to the cloud. Only after this decision are connectors, organization relationships, endpoints, and servers removed in a controlled manner ([Manage recipients with Exchange Management Tools](https://learn.microsoft.com/en-us/exchange/manage-hybrid-exchange-recipients-with-management-tools), [Decommission after Source of Authority transfer](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/decommission-last-exchange-server)).

## Technical Development and Limitations

Hybrid emerged with Exchange Online as a way to extend on-premises organizations into the cloud service in a controlled manner. Earlier generations relied more heavily on Federation Trusts; newer Exchange versions and HCW processes use OAuth and Intra-Organization Connectors for many cross-organization features ([Create a hybrid deployment](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/deploy-hybrid)).

The model is powerful because it allows migration and lasting coexistence. It is demanding because both Exchange organizations and their connection must be operated. Organizations with no remaining on-premises mailboxes after a migration should therefore consciously decide which management or coexistence function still justifies Hybrid.

## Sources

- [Microsoft Learn – Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)
- [Microsoft Learn – Active Directory in Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)
- [Microsoft Learn – Recipients in Exchange Online](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/recipients-in-exchange-online)
- [Microsoft Learn – Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)
- [Microsoft Learn – Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)
- [Microsoft Learn – Create a hybrid deployment](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/deploy-hybrid)
- [Microsoft Learn – Enable-RemoteMailbox](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox)
- [Microsoft Learn – Manage recipients with Exchange Management Tools](https://learn.microsoft.com/en-us/exchange/manage-hybrid-exchange-recipients-with-management-tools)
- [Microsoft Learn – Decommission after Source of Authority transfer](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/decommission-last-exchange-server)
- [Microsoft Learn – Autodiscover service](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)
- [Microsoft Learn – Sharing in Exchange](https://learn.microsoft.com/en-us/exchange/sharing/sharing)
- [Microsoft Learn – Configure OAuth authentication](https://learn.microsoft.com/en-us/exchange/configure-oauth-authentication-between-exchange-and-exchange-online-organizations-exchange-2013-help)
- [Microsoft Learn – Hybrid modern authentication overview](https://learn.microsoft.com/en-us/microsoft-365/enterprise/hybrid-modern-auth-overview)
- [Microsoft Learn – Move mailboxes between on-premises and Exchange Online](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/move-mailboxes)
- [Microsoft Learn – Cross-tenant mailbox migration](https://learn.microsoft.com/en-us/exchange/mailbox-migration/migrate-mailboxes-across-tenants)
