---
title: "F5 BIG-IP as an Outbound Proxy for Bulk Email Delivery: Persistence, SNAT, Timeouts, and DNS Resolution"
navTitle: "F5 Bulk Delivery"
description: "A bulk delivery of 1,000 emails per minute runs through a BIG-IP as an outbound proxy to the provider relay. This article explains why sticky sessions provide no benefit here, how to properly resolve the provider hostname using an FQDN node, and which SNAT, timeout, and connection-limit settings actually determine throughput."
date: "2026-08-26"
kategorie: "Load Balancer"
timeToRead: "9 min read"
themen:
  - loadbalancer
  - smtp-mailflow
produkte:
  - "loadbalancer"
protokolle:
  - "smtp"
  - "tcp"
  - "dns"
hauptthema: "loadbalancer"
related:
  - massenmailing-provider-wechsel-checkliste
  - mailserver-lastprofil-ermitteln
slug: "f5-big-ip-as-an-outbound-proxy-for-bulk-email-delivery-persistence-snat-timeouts-and-dns"
featured: true
translationId: "article-ee5e63e82ffd2604"
aiPrompt: |
  Du bist mein Netzwerk- und Mailflow-Assistent. Wir versenden Massenmails über eine F5 BIG-IP als ausgehenden Proxy zu einem Provider-Relay. Hilf mir, die BIG-IP-Konfiguration nach diesem Artikel zu prüfen: 1. Frage mich nach Versandrate, Anzahl paralleler Verbindungen und Nachrichten pro Verbindung. 2. Frage nach Virtual-Server-Typ, Persistenzprofil, Idle-Timeout und SNAT-Konfiguration. 3. Prüfe, ob der Provider-Hostname als FQDN-Node mit Autopopulate hinterlegt ist und ob DNS-Server auf der BIG-IP konfiguriert sind. 4. Nenne mir konkrete Abweichungen von den Empfehlungen aus dem Artikel und begründe jede Änderung.
translationOf: f5-big-ip-outbound-smtp-massenversand
url: https://rafaelpfister.ch/en/blog/f5-big-ip-as-an-outbound-proxy-for-bulk-email-delivery-persistence-snat-timeouts-and-dns
translationSourceHash: 218c4d189dd18000d6db2ead4b2106f8be858169c9d7b234e4f9320ac802fd46
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:27:46.844Z
translationReview: required
---

An invoice run or newsletter delivery of around 1,000 emails per minute leaves the corporate network, with an F5 BIG-IP acting as an outbound proxy to the provider's submission endpoint. The BIG-IP does not distribute traffic across multiple destinations; it passes it through. This exact setup determines which settings make sense and which supposed optimizations lead nowhere.

## The architecture in one sentence

The sending systems use an internal virtual server address on the BIG-IP as their smarthost, the BIG-IP translates the source addresses to a fixed public IP using SNAT, and forwards each connection to the provider hostname. There is no load balancing in the true sense on the BIG-IP because the pool has only one member. That may sound like a trivial configuration, but the detailed decisions—persistence, timeouts, SNAT type, and DNS resolution—determine whether delivery runs reliably or exhibits unexplained disconnects under load.

## Are sticky sessions better? No, for two reasons

The question of session persistence comes from the HTTP world, where a user with a shopping cart or login session must always land on the same backend. Applied to SMTP, the concept makes no sense.

First, SMTP is statelessly completed per connection: each connection handles one or more complete transactions (MAIL FROM, RCPT TO, DATA) and ends with QUIT. There is no state that would need to reside on the same target system across connections. Which system on the provider side accepts the next connection is irrelevant to delivery.

Second, there is simply nothing to persist on this BIG-IP: the pool contains exactly one member, the provider's single IP address. A persistence profile would only consume memory for a persistence table and incur a lookup on every connection that always returns the same result. The correct setting is therefore: Default Persistence Profile set to None. Even if the provider later publishes multiple IP addresses behind the hostname, persistence would be counterproductive because it would prevent distribution across those addresses and place an uneven load on individual targets.

What matters for bulk-delivery throughput is the sender's connection profile: a few long-lived connections carrying many messages per connection rather than a new connection for every email; more on that below.

## Virtual Server: FastL4 instead of Full Proxy

For simply passing SMTP through, a Performance (Layer 4) virtual server with a FastL4 profile is the right choice. The BIG-IP then processes the connection largely in hardware or along the accelerated path, without fully terminating the TCP connection. A standard virtual server in Full Proxy mode only adds value if you actually want to intervene in the data stream on the BIG-IP, for example with an SMTP security profile or protocol-level iRules. For an outbound proxy to your own contracted provider, this is unnecessary and only creates additional sources of failure.

Important in both cases: do not enable a profile that writes into the connection. The sending systems negotiate STARTTLS directly with the provider relay; any instance that modifies or filters bytes puts the TLS handshake at risk.

## DNS resolution: the provider hostname belongs in the pool as an FQDN node

The provider supplied a hostname, not an IP address. The obvious instinct—resolving the IP once and entering it as a static node—is the worst option: if the provider changes the address (maintenance, migration, DR event), delivery stops until someone adjusts the BIG-IP configuration. That is exactly what FQDN nodes are for.

An FQDN node stores the hostname instead of the address. The BIG-IP resolves the name itself, creates a so-called ephemeral node for every returned address, and automatically updates these when the DNS response changes. By default, it queries the name again after the DNS TTL expires; alternatively, a fixed query interval can be set. With Autopopulate enabled, the pool also automatically adopts multiple A records as members: if the provider later expands its submission service to multiple addresses, the BIG-IP follows without a configuration change.

Two prerequisites are often forgotten. First, the BIG-IP needs working DNS servers in the system configuration (System, Configuration, Device, DNS); FQDN nodes use the system resolvers, not a DNS cache from a listener profile. Second, these resolvers must actually be reachable from the management or TMM context; otherwise, the node remains in unresolved status and the pool stays empty.

The tmsh configuration looks like this (addresses and names are examples):

```bash
tmsh create ltm node relay-provider fqdn { \
  name mail-relay.provider.example autopopulate enabled }

tmsh create ltm pool pool_provider_smtp \
  members add { relay-provider:25 } monitor tcp

tmsh create ltm snatpool snat_mailout \
  members add { 198.51.100.10 }

tmsh create ltm virtual vs_mailout_smtp \
  destination 10.0.5.10:25 ip-protocol tcp \
  profiles add { fastL4 } pool pool_provider_smtp \
  source-address-translation { type snat pool snat_mailout }
```

The sending systems then configure 10.0.5.10 as their smarthost. Whether you use port 25 or 587 is determined by the provider; the BIG-IP configuration is identical in both cases, only the port changes.

## SNAT: a fixed address instead of Automap

For outbound mail traffic, the source address must be under control. SNAT Automap uses the floating self IP of the outbound VLAN, and it can change unnoticed during network changes or failover rebuilds. However, providers often tie message submission to IP allowlisting, and even without formal allowlisting, reputation is tied to the source address. A dedicated SNAT pool with a fixed assigned address makes the source IP a documented, stable configuration object.

Regarding capacity: a single SNAT address provides around 64,000 simultaneous translations to a single destination (one IP, one port), because each connection receives its own ephemeral source port. For the load profile described here, with a few dozen concurrent connections, that is sufficient by orders of magnitude. Port exhaustion only becomes an issue if a misconfigured sender opens a new connection for every email and does not close it cleanly; translations then accumulate in a TIME-WAIT-like state. Fix that behavior at the sender, not with a second SNAT address.

## Timeouts: the most common cause of disconnects under load

A bulk sender keeps connections open and pushes message after message through them. Pauses can occur between two messages: the sender generates the next batch, or the relay delays acceptance (tarpitting, remnants of greylisting, internal queues). The FastL4 profile's idle timeout defaults to 300 seconds. If a pause exceeds that, the BIG-IP clears the connection, and the sender writes to a connection that no longer exists.

Two settings mitigate this. First, set the idle timeout to a value above realistic pauses; for bulk delivery, 600 seconds is a reasonable starting value. The value should not be arbitrarily high, however, or orphaned connections will accumulate in the connection table. Second, leave Reset on Timeout enabled in the profile: the BIG-IP then acknowledges the cleanup with a TCP reset, and the sending MTA immediately recognizes that the connection is gone instead of timing out and rescheduling the message only after several minutes.

You have no influence over the other side's timeouts, but they are part of the picture: if the provider relay closes connections after 120 seconds of inactivity, a generous BIG-IP timeout does no good. The smallest timeout value across the entire path is decisive; when in doubt, ask the provider and use that value as the basis for planning.

## Connection strategy: few connections, many messages

Without submission requirements from the provider, a brief calculation is worthwhile. 1,000 emails per minute equals about 17 per second. An SMTP transaction over an already-established connection takes well under half a second with normal latency. With 10 to 20 parallel connections and, for example, 100 messages per connection before the sender renews them, the target rate is easily reached. The provider side generally has significantly more connection capacity available, but it is shared with all other customers. A few long-lived connections with many transactions are therefore not only efficient—the TCP and TLS handshakes are avoided per message—but also the most considerate way to use third-party infrastructure.

The controls for this are in the sending system, not on the BIG-IP: maximum messages per connection, maximum parallel connections to the smarthost, and reuse of established connections. On the BIG-IP, the setup can be protected with a connection limit on the pool member, for example 200 simultaneous connections: under normal operation, that value is never reached, but a misconfigured sender that suddenly opens one connection per email will not flood the provider relay unchecked. The limit is a safety net, not a control mechanism.

Whether the configured connection profile is actually achieved in practice is shown by measurement: connections per minute and messages per connection can be evaluated from message tracking or connector logs, as described in the article [Determining a Mail Server's Load Profile](https://rafaelpfister.ch/blog/mailserver-lastprofil-ermitteln) described. For a load test with a realistic bulk-load profile (few sessions, many messages per session), smtp-source from the Postfix package is better suited than HTTP-oriented load-testing tools because it produces exactly this connection profile.

## Monitoring: do not burden the provider with health checks

A monitor on the pool member is useful so that the BIG-IP detects and cleanly reports an outage on the provider side. The rule is: every health check is a real connection to the provider and counts against the same limits as production traffic. A simple TCP monitor with a moderate interval (30 seconds or more) is entirely sufficient. A full SMTP monitor that checks through to the banner or EHLO provides little additional insight, but creates log entries on the provider side and, in the worst case, questions about why a connection without email arrives every 5 seconds.

## Checklist

| Setting | Recommendation |
|---|---|
| Persistence profile | None; sticky sessions provide no benefit for SMTP, especially not with a single-member pool |
| Virtual server type | Performance (Layer 4) with FastL4 profile; no intervention in the data stream |
| Target node | FQDN node with Autopopulate instead of a static IP; DNS servers configured on the BIG-IP |
| SNAT | Dedicated SNAT pool with a fixed address known to the provider; no Automap |
| Idle timeout | Above actual sending pauses; starting value 600 s; Reset on Timeout enabled |
| Connection limit | As a safety net on the pool member, e.g., 200 |
| Monitor | TCP, interval 30 s or more; no aggressive SMTP monitor |
| Sender configuration | Few parallel connections, many messages per connection; reuse enabled |

The short answer to the original question is therefore: no, sticky sessions are not better; they are ineffective to harmful in this setup. The quality of the solution depends on DNS resolution of the provider hostname, a stable SNAT address, timeouts suited to the load profile, and ensuring that the sending systems submit their 1,000 emails per minute over a few established connections rather than a thousand individual ones.

## Sources

1.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): Section 4.5.4 and the transaction model show that multiple mail transactions over a connection are the intended normal case.

2.  [K7820: Overview of SNAT features](https://my.f5.com/manage/s/article/K7820): F5 introductory article on SNAT, SNAT pools, and port translation per destination.

3.  [tmsh reference: ltm node](https://clouddocs.f5.com/cli/tmsh-reference/latest/modules/ltm/ltm_node.html): documents the FQDN options (name, autopopulate, interval) for nodes and therefore for pool members.

4.  [smtp-source(1), Postfix](https://www.postfix.org/smtp-source.1.html): Load generator that replicates the bulk sender connection profile (few sessions, many messages).

5.  [Determining a Mail Server's Load Profile](https://rafaelpfister.ch/blog/mailserver-lastprofil-ermitteln): Guide on evaluating connections per minute and messages per connection from message tracking.
