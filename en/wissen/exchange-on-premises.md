---
title: "Exchange On-Premises: Server Architecture and Operations"
blatt: "exchange-on-premises"
description: "Exchange Server in an organization's own data center: mailbox and Edge roles, transport pipeline, Active Directory, ESE databases, DAG, client access, security, monitoring, backup, and recovery."
fakten:
  - label: Product role
    wert: Self-hosted messaging and groupware platform
    href: https://learn.microsoft.com/en-us/exchange/exchange-server
  - label: Server roles
    wert: Mailbox and optional Edge Transport
    href: https://learn.microsoft.com/en-us/exchange/architecture/architecture
  - label: Operating system
    wert: Windows Server according to Exchange system requirements
    href: https://learn.microsoft.com/en-us/exchange/plan-and-deploy/system-requirements
  - label: Directory
    wert: Active Directory Domain Services
    href: https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory
  - label: Mailbox storage
    wert: ESE database, transaction logs, and checkpoint
    href: https://learn.microsoft.com/en-us/exchange/client-developer/backup-restore/transaction-logs-and-checkpoint-files-for-backup-and-restore-in-exchange
  - label: High availability
    wert: Database Availability Group and database copies
    href: https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups
  - label: Transport
    wert: Frontend Transport, Transport Service, and Mailbox Transport
    href: https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow
  - label: Transport resilience
    wert: Shadow Redundancy and Safety Net
    href: https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/transport-high-availability
  - label: Client access
    wert: HTTPS, MAPI/HTTP, Outlook on the web, EWS, and ActiveSync
    href: https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access
  - label: Administration
    wert: Exchange Admin Center and Exchange Management Shell
    href: https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/admin-interface
  - label: Monitoring
    wert: Managed Availability, Health Sets, Event Logs, and performance counters
    href: https://learn.microsoft.com/en-us/exchange/high-availability/managed-availability/managed-availability
  - label: Recovery
    wert: Server Recovery, database restore, and Recovery Database
    href: https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery
werbung:
  - tools
  - newsletter
ctaThemen:
  - exchange-onprem-hybrid
  - smtp-mailflow
translationSourceHash: 8d30441ccb794fc2e8228dfe1fa084ee38a525dcb5e59222e182819a74596bbc
translationModel: gpt-5.6-terra
translatedAt: 2026-10-05T11:22:00.089Z
translationReview: automatic
---

# Exchange On-Premises: Server Architecture and Operations

**Exchange On-Premises** means that the organization operates Exchange Server in its own infrastructure. It controls Windows hosts, Active Directory, certificates, transport services, queues, mailbox databases, and recovery. Microsoft provides product code, documentation, and updates; availability and secure maintenance remain the operator's responsibility ([Exchange Server documentation](https://learn.microsoft.com/en-us/exchange/exchange-server), [Exchange Server architecture](https://learn.microsoft.com/en-us/exchange/architecture/architecture)).

The practical difference from Exchange Online becomes immediately apparent during an outage. An on-premises admin can investigate a transport queue on a specific server, check the status of a database copy, and perform a controlled switchover to another copy. However, they must also understand how SMTP, Active Directory, ESE, Windows Failover Clustering, IIS, and Exchange services interact.

## The Mailbox server is the central building block

Modern Exchange servers use the **Mailbox server** as a shared building block. It contains Client Access services that accept and proxy connections, transport services for mail flow, and the Information Store with mailbox databases. An installation can start small; multiple servers and database copies expand the same basic model for high availability ([Exchange Server architecture](https://learn.microsoft.com/en-us/exchange/architecture/architecture)).

This consolidation does not mean that all functions share the same state. An HTTPS frontend can be reachable even though the requested database is not mounted. SMTP can accept connections while a message later waits in a queue. Diagnostics must therefore follow the actual path rather than only the server's overall status.

The optional **Edge Transport role** is typically located in the perimeter network and processes SMTP traffic only. EdgeSync transfers selected recipient and configuration information to a local AD LDS instance. Edge does not hold a mailbox database and is not a replacement for internal Mailbox servers ([Edge Transport servers](https://learn.microsoft.com/en-us/exchange/architecture/edge-transport-servers/edge-transport-servers)).

## Technology stack and dependencies

The server building block determines the technology stack. Exchange runs on supported Windows Server versions and uses Active Directory for organization, server, and recipient configuration. IIS provides HTTP endpoints. PowerShell provides the management interface. ESE stores mailbox and queue data in separate databases ([Exchange Server system requirements](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/system-requirements), [Active Directory in Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)).

| Technology | Function in Exchange operations | Key admin question |
|---|---|---|
| Windows Server | Processes, services, networking, certificate store, and event logs | Is the host healthy and correctly patched? |
| Active Directory | Exchange organization, servers, recipients, RBAC, and routing information | Is the correct change visible on the domain controllers in use? |
| IIS and HTTPS | Outlook on the web, EAC, EWS, ActiveSync, Autodiscover, and MAPI/HTTP frontends | Do the name, certificate, authentication, and backend route match? |
| SMTP and TLS | Message acceptance and forwarding | Which connector accepted the message, and which next hop was selected? |
| ESE | Mailbox databases, transport queue, and transaction logs | Which database and log sequence belong together? |
| PowerShell | Management through cmdlets and RBAC | Which role, scope, and server context apply? |

For experts, the Active Directory dependency is especially important. Exchange Setup extends the schema and writes organization configuration to the Configuration partition. Recipient attributes reside in the domain partition. Replication latency or an unavailable domain controller can therefore affect different functions differently.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-exchange-onprem.svg?v=20260813" title="Interaktive Infografik: Exchange-On-Premises-Pfad von Client und SMTP über Mailboxserver, Transport, Active Directory, ESE und DAG" loading="lazy">
  <a href="/images/kb-interaktiv-exchange-onprem.svg?v=20260813">Open the interactive Exchange On-Premises diagram directly</a>.
</iframe>

## The transport pipeline step by step

With the technical foundation in place, the message path can be examined more closely. An incoming SMTP connection first reaches the Front End Transport service. It accepts the dialog and proxies it to the Transport service; it does not deliver the message to a mailbox itself ([Mail flow and the transport pipeline](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow)).

The **Transport Service** stores the message in its queue database. It then categorizes it: recipients are resolved, rules and transport agents run, and routing determines the next hop. For a local mailbox, Mailbox Transport Delivery hands the message to the Store. A message sent from a mailbox returns to Transport through Mailbox Transport Submission.

This sequence explains common observations. A successful SMTP test proves only acceptance at the frontend. A `RECEIVE` event in message tracking does not yet prove delivery. Only subsequent events, the queue, and, where applicable, the Store state show where processing ended ([Message tracking](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking)).

Transport agents and mail flow rules can reject, redirect, copy, or modify messages. Because multiple transport instances can arise in the process, searches should not be based on the subject alone. Network Message ID, Internet Message ID, sender, recipient, time, and server together provide the more reliable trail.

## Routing, domains, and connectors

After acceptance, Exchange must determine whether a recipient is local or whether the message is forwarded. **Accepted Domains** describe this relationship. An authoritative domain expects all valid recipients in its own organization. An internal relay domain permits forwarding unknown recipients. External Relay hands the domain entirely to another mail server ([Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)).

Receive Connectors classify incoming sessions based on local binding, remote IP range, authentication, and permissions. Send Connectors select an outbound path based on address space, cost, source servers, and DNS or smart host routing. Multiple matching connectors are evaluated according to documented routing rules; a connector's name does not control selection ([Connectors on Exchange servers](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors), [Mail routing in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing)).

For normal operations, a simple model is sufficient: a Receive Connector explains **how a message enters**; an Accepted Domain and recipient resolution explain **whether Exchange is responsible**; a Send Connector and routing explain **where it goes next**. Experts add AD sites, delivery groups, DAG membership, connector scoping, and transport rules.

## Mailbox databases, logs, and checkpoints

When Transport delivers to the Store, another part of the system begins. Exchange stores mailboxes in ESE mailbox databases. Changes are first written to transaction logs and later committed to the `.edb` file. The checkpoint file records up to which log position the database pages have been written ([Transaction logs and checkpoint files](https://learn.microsoft.com/en-us/exchange/client-developer/backup-restore/transaction-logs-and-checkpoint-files-for-backup-and-restore-in-exchange)).

This sequence enables crash recovery but requires associated files. A copied `.edb` without matching logs and a known shutdown state is not automatically recoverable. Similarly, a backup must not indiscriminately delete log files still required for recovery or replication.

The transport queue also uses ESE, but it is a separate database with its own logs. Mailbox databases and queues must therefore be monitored and recovered separately. A healthy mailbox database does not eliminate a blocked SMTP next hop; an empty queue does not repair a corrupted mailbox copy ([Queues and the queue database](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues)).

## Database Availability Group and Active Manager

A single Mailbox server explains normal operations. For high availability, multiple servers are joined into a **Database Availability Group**, or DAG. Each mailbox database has exactly one active copy and can have passive copies on other DAG members. Changes are transferred through log and block replication and replayed on passive copies ([Database availability groups](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups), [Mailbox database copies](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-copies)).

The **Active Manager** in the Microsoft Exchange Replication Service determines which copy is active. Best Copy and Server Selection evaluates, among other things, copy and replay status, activation blocks, and server health. A copy queue of zero is therefore useful, but not complete proof that a copy can be activated immediately ([Active Manager](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/active-manager)).

Transport high availability protects a different portion of the path. Shadow Redundancy retains an additional copy while the message is in transit. Safety Net preserves already processed messages for possible redelivery after database activation. DAG, Shadow Redundancy, and Safety Net complement one another; none of these three features replaces a backup against accidental deletion or long-undetected corruption ([Transport high availability](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/transport-high-availability)).

## Client access and Autodiscover

The database can be healthy while a user still cannot open Outlook. Client Access Services accept HTTPS connections and proxy them to the backend on the server with the active database. A load balancer therefore needs more than an open TCP port: name, certificate, protocol endpoint, and backend health must align ([Client Access protocol architecture](https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access)).

Autodiscover provides the client with the appropriate settings. Internal domain clients can use Service Connection Points in Active Directory; external and other clients follow DNS and HTTPS procedures. Errors often result from outdated SCPs, conflicting DNS responses, incorrect certificate names, or a frontend that proxies to the wrong backend ([Autodiscover service](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)).

MAPI over HTTP is the typical Outlook transport. Outlook on the web, EWS, and ActiveSync also use HTTPS, but have their own virtual directories, authentication, and application characteristics. A successful OWA test therefore does not automatically prove a healthy MAPI/HTTP session ([MAPI over HTTP](https://learn.microsoft.com/en-us/exchange/clients/mapi-over-http/mapi-over-http)).

## Active Directory and recipients

After transport and client access, the directory remains as the common foundation. Exchange stores organization and server configuration as well as recipient attributes in Active Directory. Cmdlets do not write this data to a private Exchange database, but to AD through Exchange logic ([Active Directory in Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)).

A recipient issue is therefore investigated using three questions: Does the correct object exist? Are the type, primary address, proxy addresses, and target attributes correct? Has the change reached the domain controller used by the affected Exchange service? Only then is it worth investigating Transport.

For experts, global catalogs, AD sites, Recipient Update, Address Book Policies, and hybrid attributes are added. Direct changes using generic AD tools bypass Exchange validation and can create configurations that are syntactically present but functionally inconsistent.

## Security and administrative control

Exchange publishes SMTP and HTTPS services and processes highly privileged directory and mailbox data. The foundation consists of timely Security Updates, minimally exposed endpoints, appropriate certificates, secured administrative accounts, and traceable changes ([Exchange Server Security Updates](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates), [TLS certificates in Exchange Server](https://learn.microsoft.com/en-us/exchange/architecture/client-access/certificates)).

RBAC separates tasks through roles, role groups, and scopes. Mailbox permissions such as Full Access or Send As remain separate from this. Administrator Audit Logging records cmdlet changes, but does not replace operating system, Active Directory, and security logs ([Permissions in Exchange Server](https://learn.microsoft.com/en-us/exchange/permissions/permissions), [Administrator audit logging](https://learn.microsoft.com/en-us/exchange/policy-and-compliance/admin-audit-logging/admin-audit-logging)).

For experts, the management interface itself is part of the protection model. EAC, Exchange Management Shell, Remote PowerShell, WinRM, RDP, and hypervisor access have different permissions and protocols. A compromised server administrator can take actions outside Exchange RBAC; tiering and separate privileged accounts therefore remain important.

## Operations: from symptom to specific server

Managed Availability runs probes, monitors, and responders. Health Sets summarize these results by function and can trigger automatic recovery actions. They are a good starting point, but not a complete end-to-end check ([Managed Availability](https://learn.microsoft.com/en-us/exchange/high-availability/managed-availability/managed-availability)).

For mail flow, local diagnostics begin with [`Get-Queue`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue) and [`Get-MessageTrackingLog`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog). Queue count, next hop, retry time, and `LastError` must be considered together. For databases, follow with [`Get-MailboxDatabaseCopyStatus`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-mailboxdatabasecopystatus) and [`Test-ReplicationHealth`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/test-replicationhealth). [`Get-ServerHealth`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-serverhealth) displays Health Sets and monitors.

These cmdlets run in the Exchange Management Shell on supported Windows Server systems. Network and DNS tests, by contrast, can be performed from either admin platform. [`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) checks a TCP endpoint on Windows; [`nc`](https://man.openbsd.org/nc) performs the same port test on Unix. [`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) and [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) check DNS. [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) is suitable for SMTP with STARTTLS, while [`swaks`](https://jetmore.org/john/code/swaks/) is suitable for a controlled SMTP dialog.

The diagnostic sequence is: resolve the public or internal name, verify the connection to the correct frontend, confirm acceptance in the protocol log, follow tracking events, check the queue and next hop, and investigate the Store and database only for local delivery.

## Backup and recovery

High availability keeps the service available during individual failures; recovery restores a desired earlier or lost state. Exchange documents server recovery, database restore, and Recovery Database as distinct procedures ([Backup, restore, and disaster recovery](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery)).

A recoverable inventory includes at least Active Directory, Exchange organization and server configuration, certificates and private keys, mailbox databases with logs, connector and rule configuration, and documented installation and recovery parameters. The Recovery Database makes it possible to mount a restored database in isolation and transfer content to active mailboxes ([Restore data using a recovery database](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/restore-data-using-recovery-dbs)).

Experts do not only test whether a backup job succeeded. They measure how long it actually takes to restore Active Directory, a failed server, a database, and individual mailbox content. This includes checking which log sequences are required, which DNS and certificate dependencies exist, and whether client and SMTP paths work again after restoration.

## Technical evolution and limitations

Exchange 4.0 was released in 1996. Early versions used their own directory, MAPI, and ESE; SMTP and Active Directory became central platform components with Exchange 2000. Exchange 2007 introduced server roles and the Exchange Management Shell. Exchange 2010 replaced older clustering models with the Database Availability Group ([Exchange Team: A brief history of time](https://techcommunity.microsoft.com/blog/exchange/a-brief-history-of-time---exchange-server-way/589388), [Exchange Server 2007 transport redesign](https://techcommunity.microsoft.com/blog/exchange/motivations-for-the-exchange-server-2007-transport-redesign/600465)).

Later versions consolidated Client Access and Mailbox functions again into a shared server building block. Exchange Server Subscription Edition continued the on-premises product line in the Modern Lifecycle in 2025. Build versions, supported upgrade paths, and Security Updates are checked in Microsoft's current documentation before every change ([Exchange Server SE release notes](https://learn.microsoft.com/en-us/exchange/release-notes), [Exchange Server build numbers and release dates](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates)).

Exchange On-Premises is suitable when an organization needs control over database operations, network paths, and local integration, and can provide the required 24/7 operations. The trade-off is complex dependencies, continuous security maintenance, and recovery responsibility. A single server may look simple; a resilient Exchange service is always also an Active Directory, network, certificate, storage, and operations project.

## Sources

- [Microsoft Learn – Exchange Server documentation](https://learn.microsoft.com/en-us/exchange/exchange-server)
- [Microsoft Learn – Exchange Server architecture](https://learn.microsoft.com/en-us/exchange/architecture/architecture)
- [Microsoft Learn – Exchange Server system requirements](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/system-requirements)
- [Microsoft Learn – Active Directory in Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)
- [Microsoft Learn – Edge Transport servers](https://learn.microsoft.com/en-us/exchange/architecture/edge-transport-servers/edge-transport-servers)
- [Microsoft Learn – Mail flow and the transport pipeline](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow)
- [Microsoft Learn – Message tracking](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking)
- [Microsoft Learn – Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)
- [Microsoft Learn – Connectors on Exchange servers](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors)
- [Microsoft Learn – Mail routing in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing)
- [Microsoft Learn – Transaction logs and checkpoint files](https://learn.microsoft.com/en-us/exchange/client-developer/backup-restore/transaction-logs-and-checkpoint-files-for-backup-and-restore-in-exchange)
- [Microsoft Learn – Queues and the queue database](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues)
- [Microsoft Learn – Database availability groups](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups)
- [Microsoft Learn – Monitor database availability groups](https://learn.microsoft.com/en-us/exchange/high-availability/manage-ha/monitor-dags)
- [Microsoft Learn – Mailbox database copies](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-copies)
- [Microsoft Learn – Active Manager](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/active-manager)
- [Microsoft Learn – Transport high availability](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/transport-high-availability)
- [Microsoft Learn – Client Access protocol architecture](https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access)
- [Microsoft Learn – Autodiscover service](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)
- [Microsoft Learn – MAPI over HTTP](https://learn.microsoft.com/en-us/exchange/clients/mapi-over-http/mapi-over-http)
- [Microsoft Learn – Exchange admin interfaces](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/admin-interface)
- [Microsoft Learn – TLS certificates in Exchange Server](https://learn.microsoft.com/en-us/exchange/architecture/client-access/certificates)
- [Microsoft Learn – Permissions in Exchange Server](https://learn.microsoft.com/en-us/exchange/permissions/permissions)
- [Microsoft Learn – Administrator audit logging](https://learn.microsoft.com/en-us/exchange/policy-and-compliance/admin-audit-logging/admin-audit-logging)
- [Microsoft Learn – Managed Availability](https://learn.microsoft.com/en-us/exchange/high-availability/managed-availability/managed-availability)
- [Microsoft Learn – Get-Queue](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue)
- [Microsoft Learn – Get-MessageTrackingLog](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog)
- [Microsoft Learn – Get-MailboxDatabaseCopyStatus](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-mailboxdatabasecopystatus)
- [Microsoft Learn – Test-ReplicationHealth](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/test-replicationhealth)
- [Microsoft Learn – Get-ServerHealth](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-serverhealth)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc(1)](https://man.openbsd.org/nc)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [BIND 9 – dig manual](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Swaks – SMTP test tool](https://jetmore.org/john/code/swaks/)
- [Microsoft Learn – Backup, restore, and disaster recovery](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery)
- [Microsoft Learn – Restore data using a recovery database](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/restore-data-using-recovery-dbs)
- [Exchange Team – A brief history of time](https://techcommunity.microsoft.com/blog/exchange/a-brief-history-of-time---exchange-server-way/589388)
- [Exchange Team – Exchange Server 2007 transport redesign](https://techcommunity.microsoft.com/blog/exchange/motivations-for-the-exchange-server-2007-transport-redesign/600465)
- [Microsoft Learn – Exchange Server SE release notes](https://learn.microsoft.com/en-us/exchange/release-notes)
- [Microsoft Learn – Exchange Server build numbers and release dates](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates)
