---
title: "Apache James: Modular Mail Server and Mailet Platform"
blatt: "apache-james"
description: "Apache James in technical context: protocols and mail roles, component-based architecture, queue and Mailet pipeline, mailbox and storage model, operating variants from PostgreSQL to Cassandra, and the evolution from a Java Apache project to a JVM mail platform."
fakten:
  - label: Full name
    wert: Java Apache Mail Enterprise Server
    href: https://james.apache.org/
  - label: Category
    wert: MTA, MDA, mailbox server, and mail application platform
    href: https://james.apache.org/documentation.html
  - label: Project
    wert: Apache Software Foundation
    href: https://projects.apache.org/committee.html?james
  - label: Runtime
    wert: JVM · Java 21 starting with version 3.9
    href: https://james.apache.org/james/update/2025/09/25/james-3.9.0.html
  - label: Languages
    wert: primarily Java, with some modules in Scala
    href: https://github.com/apache/james-project
  - label: Protocols
    wert: SMTP, LMTP, IMAP, POP3, ManageSieve, JMAP
    href: https://james.apache.org/server/feature-protocols.html
  - label: Architecture style
    wert: modular, component-based, Inversion of Control, event-driven
    href: https://james.apache.org/
  - label: Backends
    wert: PostgreSQL/JPA or Cassandra · OpenSearch · RabbitMQ · S3
    href: https://james.apache.org/download.cgi
  - label: Configuration
    wert: conf/*.xml and *.properties · environment variables
    href: https://james.apache.org/server/config.html
  - label: Packaging
    wert: ZIP distributions and official Docker images
    href: https://james.apache.org/download.cgi
  - label: Administration
    wert: WebAdmin REST API, CLI, and metrics
    href: https://james.apache.org/server/manage-webadmin.html
  - label: Monitoring
    wert: Health Checks, Prometheus, JMX, logs, and Grafana
    href: https://james.apache.org/server/metrics.html
  - label: License
    wert: Apache License 2.0
    href: https://www.apache.org/licenses/LICENSE-2.0
werbung:
  - newsletter
ctaThemen:
  - smtp-mailflow
translationSourceHash: adbfb005b83b16086ba55e53dd469f3aff1e5642364da5ab8b2da5d265a1ce51
translationModel: gpt-5.6-terra
translatedAt: 2026-10-03T09:21:40.355Z
translationReview: automatic
---

# Apache James: Modular Mail Server and Mailet Platform

Apache James is an open-source mail server and also a toolkit for applications whose business logic is based on email. The name stands for **Java Apache Mail Enterprise Server**. James can accept and forward messages via SMTP, manage local mailboxes, expose them through IMAP, POP3, or JMAP, and control the entire message flow through freely combinable processing components. The project therefore describes itself not merely as a server, but as a modular **inversion-of-control platform on the JVM** ([Apache James – Project Overview](https://james.apache.org/)).

This dual role distinguishes James from traditional Mail Transfer Agents such as Postfix and from turnkey security appliances. An administrator can run James as a pure SMTP relay, a complete mailbox server, or an embedded mail engine within a product. Spam scanning, encryption, archiving, or domain-specific routing do not come from a rigid feature block, but from a pipeline of **Matchers** and **Mailets**. This makes James exceptionally adaptable, but shifts part of the product responsibility from the vendor to the operating organization.

This explanation follows a message through James: from the protocol servers through the queue and Mailet pipeline to mailbox storage. The operating variants, diagnostics, and finally the project's technical evolution build on that foundation.

## Classification: MTA, MDA, and Application Platform

Not every component in an email system has the same role. A **Mail User Agent** (MUA) is the user's client, such as Thunderbird. A **Mail Transfer Agent** (MTA) transports messages between systems. A **Mail Delivery Agent** (MDA) places a message into the destination mailbox. James can serve as both MTA and MDA; through its protocol and mailbox modules, it also provides server-side services for MUAs. The official component overview lists separate projects for servers, protocols, Mailets, mailboxes, and tests ([Apache James – Software Components](https://james.apache.org/documentation.html)).

| Role | Implementation in James | Handoff point |
|---|---|---|
| Message transport | SMTP and LMTP servers, queue, Remote Delivery Mailet | other MTAs, relays, and gateways |
| Local delivery | Mailet pipeline and Mailbox API | users, domains, and quotas |
| Mailbox access | IMAP, POP3, and JMAP | mail clients and web applications |
| Filter logic | Matchers, Mailets, Processors, and Sieve | internal rules and external scanning services |
| Administration | WebAdmin REST API, CLI, Health Checks, and metrics | automation and monitoring |

James is therefore **not a mail client**, nor is it a preconfigured secure mail gateway. It provides building blocks for transport, delivery, storage, and processing. Whether this becomes a simple relay, a multi-tenant mail service, or a product-specific gateway is determined by the selected distribution and configuration.

## Protocols, TLS, and Ports

James provides SMTP, LMTP, IMAP, POP3, and ManageSieve as TCP-based services; JMAP and WebAdmin use HTTP ([Apache James – Protocol Servers](https://james.apache.org/server/feature-protocols.html)). Depending on the listener, TLS protects a connection encrypted from the start or is inserted into an existing session through StartTLS. DNS is not part of the James process, but it is essential for a public MTA: MX records determine the destination, A and AAAA records its addresses, and PTR records affect the reputation of outgoing connections.

The port number alone does not describe the security semantics. Port 25 is intended for server-to-server transport; authenticated client submission belongs on port 587 according to [RFC 6409](https://datatracker.ietf.org/doc/html/rfc6409). Since [RFC 8314](https://datatracker.ietf.org/doc/html/rfc8314), port 465 has again been registered for implicitly encrypted Message Submission. The same two patterns apply to IMAP and POP3: a plaintext connection with possible StartTLS, or TLS established immediately.

| Service | Typical ports | Standard | Meaning in James |
|---|---:|---|---|
| SMTP | 25, 587, 465 | [RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321) | acceptance, relay, and submission |
| LMTP | configurable, registered 24 | [RFC 2033](https://datatracker.ietf.org/doc/html/rfc2033) | local handoff with status per recipient |
| IMAP4rev2 | 143, 993 | [RFC 9051](https://datatracker.ietf.org/doc/html/rfc9051) | synchronous mailbox access |
| POP3 | 110, 995 | [RFC 1939](https://datatracker.ietf.org/doc/html/rfc1939) | simple message retrieval |
| ManageSieve | 4190 | [RFC 5804](https://datatracker.ietf.org/doc/html/rfc5804) | management of user-specific Sieve rules |
| JMAP Mail | usually 443 | [RFC 8621](https://datatracker.ietf.org/doc/html/rfc8621) | HTTP-based mailbox access for modern clients |

The ports are configurable; the binding factor is the combination of listener, protocol, TLS mode, and authentication. The [IANA Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) remains the reference for registered assignments.

## Architectural Approach

James follows a **component-based architecture**. Protocol servers, queue, processing logic, mailbox, user management, search index, and administration are separated from each other through APIs and assembled through dependency injection. The distributions documented for James 3.9 use Google Guice for this; the Spring setup belongs to an older generation. The decoupling is not merely code organization: it allows the same [Mailbox API](https://james.apache.org/mailbox/mailbox-api.html) to be used with different persistence layers and the same Mailet logic to be used in very different server profiles.

The central data path is asynchronous. An SMTP listener does not have to deliver an accepted message completely before responding to the connection. It places a mail object into a queue; a **Spooler** removes it later and sends it through the Mailet container. The queue thus separates reception load, processing time, and the availability of downstream systems. The distributed operations documentation accordingly describes it as a mandatory part of an SMTP server ([Apache James – Distributed Server Operations](https://james.apache.org/server/manage-guice-distributed-james.html)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 976" src="/images/apache-james-architektur.svg?v=20260813" title="Interaktive Infografik: technische Architektur und Nachrichtenfluss von Apache James" loading="lazy">
  <a href="/images/apache-james-architektur.svg?v=20260813">Open the technical architecture infographic</a>
</iframe>

### The Processing Path of a Message

1. **Protocol acceptance:** SMTP or LMTP checks the session, authentication, envelope sender, and recipients. After the end of `DATA`, an internal `Mail` object is created with the envelope, MIME content, and attributes.
2. **Queue:** The object is queued persistently or transiently. Only from this point onward are acceptance and processing decoupled.
3. **Spooler:** Workers remove queue entries and hand them to the Mailet container.
4. **Processor:** A named Processor contains an ordered list of Matcher/Mailet pairs. The required `root` Processor is the entry point.
5. **Matcher:** A Matcher does not modify the message, but returns the subset of recipients for which a condition applies.
6. **Mailet:** The associated Mailet modifies the message or envelope, triggers a side effect, delivers locally or remotely, or branches into another Processor.
7. **Result:** The message ends up in a user mailbox, in outgoing delivery, in a Mail Repository for later handling, or is completed after a successful action.

An important detail is the **recipient-specific split**. If a Matcher applies only to some recipients, the container splits processing into matching and nonmatching recipient sets. Rules therefore do not necessarily apply to an entire MIME message. A Mailet can also jump directly to another Processor via `ToProcessor`; the pipeline is therefore more of a directed processing graph than a single linear list. The official [Mailet Container documentation](https://james.apache.org/server/feature-mailetcontainer.html) describes exactly this model.

A minimal, simplified pattern looks like this:

```xml
<processor state="root" enableJmx="true">
  <mailet match="RelayLimit=30" class="ToRepository">
    <repositoryPath>cassandra://var/mail/relay-denied/</repositoryPath>
  </mailet>
  <mailet match="RecipientIsLocal" class="LocalDelivery" />
  <mailet match="All" class="RemoteDelivery" />
</processor>
```

The order is part of the semantics. A broadly matching rule at the beginning can make subsequent rules unreachable; an infinite loop between Processors can tie up the Spooler. James therefore provides configurable error handling for each Matcher and Mailet, as well as dedicated error Processors ([Mailet Container Configuration](https://james.apache.org/server/config-mailetcontainer.html)).

The component architecture becomes concrete as soon as a message reaches the queue. The Processor, Matcher, and Mailet then determine which processing steps follow and where the result goes.

## Technical Structure

The architecture describes the message path; installation and operation must now translate it into a concrete component model. The decisive factor is which runtime, storage, and supporting services the selected James profile actually requires.

### Technology Stack and Administration Overview

For an initial product assessment, operational boundaries matter more than class names. The following overview condenses the stack into the questions that should be clarified before installation, integration, or taking over an existing environment:

| Area | Technology or artifact | What the administrator needs to know |
|---|---|---|
| Runtime | Java 21, JVM; source code primarily Java, with some Scala modules | heap, garbage collection, threading, and JVM patches are part of server operations |
| Build and package | Maven multi-module project; ZIPs and Docker images | custom Mailets must match the James, Java, and Jakarta generation |
| Wiring | Guice in the 3.9 generation; Spring in older installations | the selected distribution determines the available modules and configuration files |
| Configuration | `conf/*.xml`, `conf/*.properties`, environment variables | especially important: `smtpserver.xml`, `mailetcontainer.xml`, `webadmin.properties`, JMAP, and backend files |
| Processing | MailQueue, Spooler, Processor, Matcher, Mailet | acceptance, processing, and final delivery are separate states |
| Data | PostgreSQL/JPA or Cassandra; optional S3, OpenSearch, RabbitMQ | source, projection, queue, and blob content require separate recovery plans |
| Administration | WebAdmin REST API and `james-cli` | REST is more powerful; the CLI is included with every wiring variant |
| Observability | Health Checks, Dropwizard Metrics, Prometheus, JMX, logs, Grafana | queues, Mailets, Matchers, protocols, and backends have their own metrics |
| Security | TLS keystores, SMTP AUTH, JWT for WebAdmin, network segmentation | WebAdmin without enabled JWT is not protected by default |

According to the project, all configuration files reside in `conf` or `conf/META-INF`; which ones actually apply depends on wiring and backend. Values can be obtained from the environment with `${env:VARIABLE}` ([Apache James – Configuration](https://james.apache.org/server/config.html)). This is practical for containers, but it does not replace secret management: certificates, private keys, JWT keys, and database passwords should be provided as mounted secrets or through the orchestration platform.

### Protocol Layer

The Protocols project provides extensible server implementations for SMTP, LMTP, IMAP, POP3, ManageSieve, and JMAP ([James Protocols](https://james.apache.org/server/feature-protocols.html)). The listeners are not hardwired to a particular storage implementation. IMAP and JMAP access data through the Mailbox API; SMTP hands accepted messages to the queue and Mailet container. This allows protocols to be scaled or disabled independently of the backend topology.

### Mailbox, Mail Repository, and Blob Store

James distinguishes among three storage terms that should not be conflated in operations:

| Storage | Content | Visibility | Typical recovery |
|---|---|---|---|
| **Mailbox** | folders, messages, flags, UIDs, ACLs, and quotas for a user | IMAP/JMAP/POP3 | restore or replication of the mailbox backend |
| **Mail Repository** | messages from processing paths such as `error`, `relay-denied`, or quarantine | administration only | fix the cause and reprocess the message |
| **Blob Store** | binary MIME content or large objects | indirectly referenced through metadata | consistent backup with metadata and references |

The [persistence documentation](https://james.apache.org/server/feature-persistence.html) emphasizes that a Mail Repository is **not** the user mailbox. This separation is valuable for incident response: a faulty message can be isolated, investigated, and returned to the pipeline after a correction without bypassing the mailbox model.

### Event Bus, Search, and Projections

Mailbox operations generate events, such as `MailboxAdded`, `MessageMoveEvent`, `FlagsUpdated`, or quota changes. Listeners use these to update quotas, search indexes, and other projections. In the distributed profile, RabbitMQ handles communication, OpenSearch handles search, and Cassandra handles metadata; binary content is stored in an S3-compatible object store. This decomposition enables horizontal scaling, but creates **eventual consistency** between the source and projections. Failed listener events end up in an Event Dead Letter and must be monitored and, if necessary, redelivered ([Distributed James – Mailbox Event Bus](https://james.apache.org/server/manage-guice-distributed-james.html#Mailbox_Event_Bus)).

### MIME, Sieve, and Sender Authentication

The James project encompasses more than the server. **Apache Mime4J** parses MIME structures in a streaming manner or as an object model; **jSieve** implements the Sieve filtering language; **jSPF** and **jDKIM** provide Java libraries for sender verification and DKIM signing and verification, respectively. These modules are independent projects and can also be used outside a complete James server ([Apache James – Components](https://james.apache.org/documentation.html)).

Which of these components run on one node or in a distributed setup is not merely a performance question. The choice also determines consistency, restart behavior, and the number of backends to monitor.

## Operating Variants and Scaling

For James 3.9.0, Apache documents several profiles. They are not simply different installers, but different consistency, scaling, and operational models. In this version, the JPA variant is explicitly designated as **legacy**; it is joined by a PostgreSQL distribution and a distributed distribution ([Apache James – Downloads](https://james.apache.org/download.cgi)). The items labeled **operational inference** in the infographic are derived recommendations, not literal vendor statements.

<iframe class="kb-infographic" style="aspect-ratio: 1280 / 956" src="/images/apache-james-betriebsmodelle.svg?v=20260813" title="Interaktive Infografik: Apache-James-Betriebsmodelle und Technologiestacks" loading="lazy">
  <a href="/images/apache-james-betriebsmodelle.svg?v=20260813">Open the operating model comparison infographic</a>
</iframe>

| Profile | Persistence and services | Suitable for | Operational consequence |
|---|---|---|---|
| JPA/Guice (legacy) | embedded H2 database or external SQL database; traditional single-server model | labs, migration of older installations, small specialized solutions | few components, but a limited strategic path and vertical scaling |
| PostgreSQL | PostgreSQL as the core; optional OpenSearch, RabbitMQ, and S3-compatible storage | new single-node or multi-node installations with a relational basis | backup and HA are well understood; introduce additional services only for required scaling |
| Distributed/Guice | Cassandra, RabbitMQ, OpenSearch, and an S3-compatible object store | large, horizontally scalable services | multiple failure domains, projections, Dead Letters, and more complex consistency checks |
| Memory | transient in-memory components | testing and development | no persistent production data |

The 3.9 release highlights the high-performance PostgreSQL implementation as a major addition and describes it as capable of running standalone as well as scaling with RabbitMQ, OpenSearch, and S3 ([Apache James 3.9.0](https://james.apache.org/james/update/2025/09/25/james-3.9.0.html)). For new installations, this is usually the easiest starting point to understand: start with relational consistency and familiar backup procedures, then add services only for specifically measured requirements.

## Security Model

James provides TLS, SMTP authentication, protocol controls, and cryptographic Mailets. However, this does not automatically result in secure production operation. Transport encryption protects one hop; it replaces neither end-to-end encryption nor mandatory recipient verification. The [TLS configuration](https://james.apache.org/server/config-ssl-tls.html) separates keystore, enabled cipher suites, StartTLS, and implicit TLS for each listener. A certificate change must therefore be tracked separately for SMTP, IMAP, POP3, and HTTP.

**WebAdmin** deserves special attention. The REST API can modify domains, users, mailboxes, queues, repositories, quotas, and maintenance tasks. According to the [WebAdmin documentation](https://james.apache.org/server/manage-webadmin.html), JWT authentication is disabled by default; without additional protection, the API must therefore never be reachable from an uncontrolled network. Health endpoints and API documentation may also intentionally remain outside authentication.

Minimum production hardening includes:

- binding WebAdmin to a management network, enabling JWT, and further restricting access through a firewall or reverse proxy;
- preventing open relays through explicit relay, authentication, and recipient rules;
- operating submission and server-to-server SMTP on separate listeners with different policies;
- removing demo domains, sample users, and default passwords from container images before the first external start;
- managing private keys outside the container layer and monitoring expiration dates;
- treating custom Mailets as application code: review dependencies, run tests, and limit runtime permissions;
- deliberately designing spam and malware scanning. James is a platform; external scanners and reputation services are integrated through Mailets or protocol handoffs.

For troubleshooting, the message path is checked again in the same order: listener, queue, Mailet pipeline, repository, mailbox, and outgoing delivery.

## Operations and Troubleshooting

With a modular mail server, “the service is running” is not a sufficient statement of state. WebAdmin Health Checks distinguish `healthy`, `degraded`, and `unhealthy`; in strict mode, even a degraded component results in HTTP 503. Depending on the profile, checks include JPA or Cassandra, OpenSearch, RabbitMQ, the Guice lifecycle, Event Dead Letters, and a complete test delivery ([WebAdmin Health Checks](https://james.apache.org/server/manage-webadmin.html#HealthCheck)).

For diagnostics, a layered approach is more efficient than a global log search:

1. **Connection:** Does the client reach the correct listener, and does TLS succeed with the expected certificate and hostname?
2. **SMTP transaction:** Which response code was returned for `MAIL FROM`, `RCPT TO`, and `DATA`? A `250` after `DATA` means acceptance, not necessarily final delivery.
3. **Queue:** Is the number of queued entries growing, is their age increasing, or is the same remote error recurring?
4. **Mailet pipeline:** Which Processor and which Matcher/Mailet pair handled the message? The mail ID serves as the correlation key.
5. **Repository:** Is the message in `error`, `address-error`, `relay-denied`, or a custom repository? Fix the cause before reprocessing.
6. **Mailbox and events:** Is the message present in the authoritative mailbox store but missing from the search index or JMAP? Then listeners, Dead Letters, and reindexing are more relevant than SMTP.
7. **Remote Delivery:** For outgoing delivery, check DNS, route, TLS, peer response code, retry plan, and bounce generation separately.

A compact synthetic check can connect the administration and data planes:

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für den Health Check">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$headers = @{ Authorization = "Bearer $env:JAMES_ADMIN_JWT" }
Invoke-RestMethod `
  -Uri "https://james-admin.example.net/healthcheck?strict" `
  -Headers $headers</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">curl --fail --silent \
  -H "Authorization: Bearer $JAMES_ADMIN_JWT" \
  "https://james-admin.example.net/healthcheck?strict"</code></pre>
  </div>
</div>

On Windows, [`Invoke-RestMethod`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod) calls the REST endpoint; on Linux and Unix, [`curl`](https://curl.se/docs/manpage.html) performs the same HTTP check. Both commands test only the documented WebAdmin Health Check here and do not replace a synthetic SMTP or mailbox transaction.

In addition, at minimum, alert on queue depth and age, error repositories, Event Dead Letters, OpenSearch indexing lag, backend latencies, SMTP response classes, JVM memory, and certificate expiration periods. In the distributed variant, a healthy James process while RabbitMQ or OpenSearch is impaired is only a partial success.

### Tools for the Administrator Workstation

James includes a command-line client for domains, users, mailboxes, mappings, quotas, and reindexing; in Guice containers, it is available as `james-cli` ([James CLI](https://james.apache.org/server/manage-cli.html)). A reliable diagnostic setup should also include several protocol-neutral tools on the administrator workstation:

| Tool | Use with James |
|---|---|
| [`swaks`](https://www.jetmore.org/john/code/swaks/) | complete SMTP and submission transaction with AUTH, TLS, envelope, and freely set headers |
| [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) | check certificate chain, SNI, cipher, and StartTLS on SMTP, IMAP, or POP3 |
| [`curl`](https://curl.se/docs/manpage.html) and [`jq`](https://jqlang.org/manual/) | query WebAdmin, Health Checks, tasks, and metrics automatically |
| [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) or [`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) | check MX, A/AAAA, PTR, SPF, DKIM, and DMARC |
| [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) or [Wireshark](https://www.wireshark.org/docs/wsug_html_chunked/) | distinguish handshakes, retransmits, connection drops, and protocol dialogs |
| [Prometheus](https://prometheus.io/docs/introduction/overview/) and [Grafana](https://grafana.com/docs/grafana/latest/) | monitor queue and protocol metrics, latency percentiles, Mailet/Matcher runtimes, and backend states |
| [JMX](https://docs.oracle.com/en/java/javase/21/management/java-management-extensions-jmx-user-guide.html), [VisualVM](https://visualvm.github.io/documentation.html), and [`jcmd`](https://docs.oracle.com/en/java/javase/21/docs/specs/man/jcmd.html) | investigate heap, threads, garbage collection, and JVM-internal metrics |

The native [metrics documentation](https://james.apache.org/server/metrics.html) lists active SMTP, IMAP, and LMTP connections, queue entries, sent and delivered messages, response times per protocol, and runtimes of individual Mailets and Matchers, among other metrics. These metrics are more meaningful than a single process uptime because they reflect the path of a message through the architecture.

## Technical History

James did not originate as a port of an existing Unix MTA. The oldest surviving project pages from **1997/1998** initially describe a planned Java server that was not yet usable, based on shared packages from the Java Apache Project. It envisioned a shared protocol interface, JDBC storage, and a **MailServlet** interface modeled after Servlets; the Apache JServ environment provided technical groundwork ([James 1.0 Archive](https://svn.apache.org/repos/asf/james/server/tags/james_1_0/docs/index.html?p=1400000)). The later Mailet API preserved the core idea of small, deployable processing components without becoming part of the Java Servlet specification.

| Period | Technical development step |
|---|---|
| 1997–1998 | Design in the Java Apache Project: pure Java server, shared protocol and resource interfaces, MailServlet concept |
| February 2001 | Migration from the Java Apache Project to the Jakarta project ([Jakarta News 2001](https://jakarta.apache.org/site/news/news-2001.html#20010311.1)) |
| James 1.x/2.x | stable SMTP/POP3 server, temporarily NNTP; Mailet engine, file and RDBMS storage; Avalon/Phoenix component container ([Document Archive](https://james.apache.org/server/archive/document_archive.html)) |
| early 2000s | rise from a Jakarta subproject to an independent Apache Software Foundation top-level project ([James 2.1.3 – archived project page](https://svn.apache.org/repos/asf/james/server/tags/deprecated/build_2_2_0_RC1/www/index.html?p=1400000)) |
| 2010 | James 3.0 M1 with full IMAP support, SMTP/LMTP, revised Mailet API, and Maildir, JPA, and JCR storage ([Release Announcement](https://james.apache.org/james/update/2010/11/05/james-3.0-M1.html)) |
| James 3.x | replacement of Avalon/Phoenix by Spring and later a strategic move toward Guice; expansion of IMAP, JMAP, REST administration, and distributed backends |
| September 2025 | James 3.9.0: transition from `javax` to `jakarta`, Java 21, and new PostgreSQL implementation ([Release Announcement](https://james.apache.org/james/update/2025/09/25/james-3.9.0.html)) |

The source code resides in the official [apache/james-project repository](https://github.com/apache/james-project). The 3.9 generation considered here consists primarily of Java; some modules use Scala. It is built as a large Maven multi-module project. Its long history explains why multiple generations remain visible in documentation and installations: Phoenix and Spring terms in older texts, Guice in the 3.x documentation, JPA as a legacy path, and PostgreSQL or Cassandra profiles for distributed deployments.

## Suitability and Limitations

James is especially suitable when email is **part of an application** rather than merely infrastructure: rule-based processing, custom Mailets, open protocols, JMAP, controllable data storage, or horizontal scaling without a proprietary server core. The public APIs allow transport, mailbox, and business logic to evolve independently.

James is less suitable for organizations expecting a turnkey appliance with a complete GUI, preconfigured spam and malware protection, vendor SLAs, and a single backup object. Modular freedom creates integration work. The distributed profile in particular requires operational experience with multiple data systems and a clear definition of source, projection, rebuild, and recovery point.

The key architectural question is therefore: **Should email be operated as a configurable protocol system or as a finished product?** For the former, James provides an unusually deep and open toolkit. For the latter, a more heavily preconfigured product is often more economical.

## Sources

- [Apache Projects – James Committee](https://projects.apache.org/committee.html?james)
- [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0)
- [Microsoft Learn – Invoke-RestMethod](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod)
- [curl – Manpage](https://curl.se/docs/manpage.html)
- [SWAKS – Swiss Army Knife for SMTP](https://www.jetmore.org/john/code/swaks/)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [jq – Manual](https://jqlang.org/manual/)
- [BIND 9 – dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)
- [tcpdump – Manpage](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [Wireshark – User’s Guide](https://www.wireshark.org/docs/wsug_html_chunked/)
- [Prometheus – Overview](https://prometheus.io/docs/introduction/overview/)
- [Grafana – Documentation](https://grafana.com/docs/grafana/latest/)
- [Oracle – JMX User Guide](https://docs.oracle.com/en/java/javase/21/management/java-management-extensions-jmx-user-guide.html)
- [VisualVM – Documentation](https://visualvm.github.io/documentation.html)
- [Oracle – jcmd](https://docs.oracle.com/en/java/javase/21/docs/specs/man/jcmd.html)
- [Apache James – Project Overview](https://james.apache.org/) – self-description, JVM, protocols, modules, and architectural goals.
- [Apache James – Software Components](https://james.apache.org/documentation.html) – server, Mailet, mailbox, Protocols, and subprojects.
- [Apache James – Protocol Servers](https://james.apache.org/server/feature-protocols.html) – supported protocol services.
- [RFC 6409](https://datatracker.ietf.org/doc/html/rfc6409)
- [RFC 8314](https://datatracker.ietf.org/doc/html/rfc8314)
- [IETF: SMTP](https://datatracker.ietf.org/doc/html/rfc5321), [LMTP](https://datatracker.ietf.org/doc/html/rfc2033), [Message Submission](https://datatracker.ietf.org/doc/html/rfc6409), [IMAP4rev2](https://datatracker.ietf.org/doc/html/rfc9051), [POP3](https://datatracker.ietf.org/doc/html/rfc1939), [ManageSieve](https://datatracker.ietf.org/doc/html/rfc5804), and [JMAP Mail](https://datatracker.ietf.org/doc/html/rfc8621) – normative protocol standards.
- [RFC 2033](https://datatracker.ietf.org/doc/html/rfc2033)
- [RFC 9051](https://datatracker.ietf.org/doc/html/rfc9051)
- [RFC 1939](https://datatracker.ietf.org/doc/html/rfc1939)
- [RFC 5804](https://datatracker.ietf.org/doc/html/rfc5804)
- [RFC 8621](https://datatracker.ietf.org/doc/html/rfc8621)
- [IANA Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) – registered ports.
- [Mailbox API](https://james.apache.org/mailbox/mailbox-api.html)
- [Apache James – Managing Distributed James](https://james.apache.org/server/manage-guice-distributed-james.html) – Cassandra, S3, OpenSearch, RabbitMQ, event bus, and operations.
- [Apache James – Mailet Container](https://james.apache.org/server/feature-mailetcontainer.html) – Matchers, Mailets, Processors, Spooler, and recipient splitting.
- [Apache James – Mailet Container Configuration](https://james.apache.org/server/config-mailetcontainer.html) – pipeline configuration and error handling.
- [Apache James – Configuration](https://james.apache.org/server/config.html) – configuration directory, files, and environment variables.
- [Apache James – Persistence](https://james.apache.org/server/feature-persistence.html) – distinction between Mailbox and Mail Repository.
- [Apache James – Downloads](https://james.apache.org/download.cgi) – official server profiles and downloads.
- [Apache James Server 3.9.0](https://james.apache.org/james/update/2025/09/25/james-3.9.0.html) – Java 21, Jakarta transition, and PostgreSQL implementation.
- [Apache James – SSL/TLS Configuration](https://james.apache.org/server/config-ssl-tls.html) – TLS modes and listener configuration.
- [Apache James – WebAdmin](https://james.apache.org/server/manage-webadmin.html) – REST administration, JWT note, and Health Checks.
- [Apache James – Command Line](https://james.apache.org/server/manage-cli.html) – CLI for domains, users, mailboxes, mappings, quotas, and reindexing.
- [Apache James – Metrics](https://james.apache.org/server/metrics.html) – Prometheus, JMX, and available operational metrics.
- [James 1.0 Archive of the Java Apache Project](https://svn.apache.org/repos/asf/james/server/tags/james_1_0/docs/index.html?p=1400000) – early architecture and MailServlet planning.
- [Jakarta Project News 2001](https://jakarta.apache.org/site/news/news-2001.html) – migration of the James project to Jakarta.
- [Apache James Document Archive](https://james.apache.org/server/archive/document_archive.html) – documentation for versions 1.x and 2.x.
- [James 2.1.3 – archived project page](https://svn.apache.org/repos/asf/james/server/tags/deprecated/build_2_2_0_RC1/www/index.html?p=1400000)
- [Apache James 3.0 M1](https://james.apache.org/james/update/2010/11/05/james-3.0-M1.html) – IMAP, storage profiles, and the Mailet API of the 3.x generation.
- [Apache James – GitHub Repository](https://github.com/apache/james-project) – source code, build, and module structure.
