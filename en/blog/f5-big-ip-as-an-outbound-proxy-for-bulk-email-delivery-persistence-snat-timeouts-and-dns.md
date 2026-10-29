---
title: "F5 BIG-IP as an Outbound Proxy for Bulk Email Delivery: Persistence, SNAT, Timeouts, and DNS Resolution"
navTitle: "F5 Bulk Delivery"
description: "A bulk mailing of 1,000 emails per minute passes through a BIG-IP as an outbound proxy to the provider relay. This article explains why sticky sessions do not help here, how to properly resolve the provider hostname using an FQDN node, and which SNAT, timeout, and connection-limit settings actually determine throughput."
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
translationOf: f5-big-ip-outbound-smtp-massenversand
translationSourceHash: 5c62d1467d3a000a8c4a3255d6f925e294d76d1b65054e91b863a5166e1a22f8
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:22:18.964Z
translationReview: required
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/en/blog/f5-big-ip-as-an-outbound-proxy-for-bulk-email-delivery-persistence-snat-timeouts-and-dns
---

A billing run or newsletter mailing with around 1,000 emails per minute leaves the corporate network, with an F5 BIG-IP acting as an outbound proxy to the provider's submission endpoint. The BIG-IP does not distribute traffic across multiple destinations; it passes it through. This exact setup determines which settings make sense and which supposed optimizations accomplish nothing.

## The architecture in one sentence

The sending systems use an internal virtual server address on the BIG-IP as their smarthost; the BIG-IP translates sender addresses through SNAT to a fixed public IP and forwards every connection to the provider hostname. Load balancing in the true sense does not take place on the BIG-IP because the pool has only one member. This may sound like a trivial configuration, but the detailed decisions—persistence, timeouts, SNAT type, and DNS resolution—determine whether delivery runs reliably or suffers unexplained disconnects under load.

## Are sticky sessions better? No, for two reasons

The question of session persistence comes from the HTTP world, where a user with a shopping cart or login session must always land on the same backend. Applied to SMTP, the concept makes no sense.

First, SMTP is statelessly completed per connection: Each connection handles one or more complete transactions (MAIL FROM, RCPT TO, DATA) and ends with QUIT. There is no state that must reside on the same target system across connections. Which system on the provider side accepts the next connection is irrelevant to delivery.

Second, there is simply nothing to persist on this BIG-IP: The pool contains exactly one member, the provider's single IP address. A persistence profile would only consume memory for a persistence table and incur a lookup on every connection that always returns the same result. The correct setting is therefore: Default Persistence Profile set to None. Even if the provider later publishes multiple IP addresses behind the hostname, persistence would be counterproductive because it would prevent distribution across those addresses and place an uneven load on individual targets.

What matters for bulk-delivery throughput is the sender's connection profile: a few long-lived connections with many messages per connection rather than a new connection for every email; more on that below.

## Virtual Server: FastL4 instead of Full Proxy

For simply passing SMTP through, a Performance (Layer 4) virtual server with a FastL4 profile is the right choice. The BIG-IP then processes the connection largely in hardware or on the accelerated path, without fully terminating the TCP connection. A standard virtual server in Full Proxy mode adds value only if you truly want to intervene in the data stream on the BIG-IP, for example with an SMTP security profile or protocol-level iRules. For an outbound proxy to your own contracted provider, this is unnecessary and only creates additional sources of error.

Important in both cases: do not enable a profile that writes into the connection. The sending systems negotiate STARTTLS directly with the provider relay; any instance that modifies or filters bytes jeopardizes TLS establishment.

## DNS resolution: the provider hostname belongs in the pool as an FQDN node

The provider supplied a hostname, not an IP address. The obvious impulse to resolve the IP once and enter it as a static node is the worst option: If the provider changes the address—for maintenance, a migration, or a DR event—delivery stops until someone adjusts the BIG-IP configuration. That is exactly what FQDN nodes are for.

An FQDN node stores the hostname instead of the address. The BIG-IP resolves the name itself, creates a so-called ephemeral node for every returned address, and updates these automatically when the DNS response changes. By default, it queries the name again every 3,600 seconds (one hour); with `interval ttl` it follows the TTL of the DNS response instead, or you can set another fixed query interval. With Autopopulate enabled, the pool also automatically adopts multiple A records as members: If the provider later expands its submission service to multiple addresses, the BIG-IP follows without a configuration change.

Two prerequisites are often overlooked. First, the BIG-IP needs working DNS servers in the system configuration (System, Configuration, Device, DNS); FQDN nodes use the system resolvers, not a DNS cache from a listener profile. Second, these resolvers must actually be reachable from the management or TMM context; otherwise, the node remains in unresolved status and the pool stays empty.

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

The sending systems then configure 10.0.5.10 as their smarthost. Whether you use port 25 or 587 is specified by the provider; the BIG-IP configuration is identical in both cases, with only the port changing.

## SNAT: a fixed address instead of Automap

For outbound mail traffic, the source address must be under control. SNAT Automap uses the floating self IP of the outbound VLAN, and it may change unnoticed during network changes or failover modifications. Providers often tie mail submission to IP allowlisting, and even without formal allowlisting, reputation is associated with the source address. A dedicated SNAT pool with a fixed assigned address makes the source IP a documented, stable configuration object.

In terms of capacity, a single SNAT address offers around 64,000 simultaneous translations to a single destination (one IP, one port), because every connection receives its own ephemeral source port. For the load profile described here, with a few dozen simultaneous connections, that is sufficient by orders of magnitude. Port exhaustion becomes an issue only when a misconfigured sender opens a new connection for every email and does not close it cleanly; translations then accumulate in a TIME-WAIT-like state. Fix such behavior at the sender, not by adding a second SNAT address.

## Timeouts: the most common cause of disconnects under load

A bulk sender keeps connections open and sends message after message through them. Pauses can occur between two messages: The sender generates the next batch, or the relay delays acceptance (tarpitting, remnants of greylisting, internal queues). The FastL4 profile's idle timeout is 300 seconds by default. If a pause exceeds that, the BIG-IP removes the connection, and the sender writes to a connection that no longer exists.

Two settings mitigate this. First, set the idle timeout to a value above realistic pauses; for bulk delivery, 600 seconds is a reasonable starting point. The value should not be arbitrarily high, however, or orphaned connections will accumulate in the connection table. Second, leave Reset on Timeout enabled in the profile: The BIG-IP then acknowledges cleanup with a TCP reset, and the sending MTA immediately recognizes that the connection is gone instead of running into a timeout and rescheduling the message only after several minutes.

You have no influence over the other side's timeouts, but they are part of the picture: If the provider relay closes connections after 120 seconds of inactivity, a generous BIG-IP timeout is of no use. The smallest timeout value along the entire path is decisive; when in doubt, ask the provider and use that value as the basis for planning.

## Connection strategy: few connections, many messages

Without submission requirements from the provider, a quick calculation is useful. 1,000 emails per minute is around 17 per second. An SMTP transaction over an already established connection takes well under half a second at normal latency. With 10 to 20 parallel connections and, for example, 100 messages per connection before the sender renews them, the target rate is easily reached. The provider side generally has considerably more connection capacity available, but it is shared with all other customers. A few long-lived connections with many transactions are therefore not only efficient—the TCP and TLS setup is eliminated for each message—but also the most considerate way to use third-party infrastructure.

The controls for this are in the sending system, not on the BIG-IP: maximum messages per connection, maximum parallel connections to the smarthost, and reuse of established connections. On the BIG-IP, you can protect the setup with a connection limit on the pool member, for example 200 simultaneous connections: Under normal operation, that value is never reached, but a misconfigured sender that suddenly opens a connection for every email will not flood the provider relay unchecked. The limit is a safety net, not a control mechanism.

Whether the configured connection profile actually takes effect in practice is shown by measurement: Connections per minute and messages per connection can be evaluated from Message Tracking or connector logs, as described in the article [Determining a Mail Server's Load Profile](https://rafaelpfister.ch/blog/mailserver-lastprofil-ermitteln). For a load test with a realistic bulk-load profile—few sessions, many messages per session—smtp-source from the Postfix package is better suited than HTTP-oriented load-testing tools because it creates exactly this connection profile.

## Monitoring: do not burden the provider with health checks

A monitor on the pool member makes sense so that the BIG-IP can detect and clearly report an outage on the provider side. The following applies: Every health check is a real connection to the provider and counts against the same limits as production traffic. A simple TCP monitor with a moderate interval (30 seconds or more) is entirely sufficient. A complete SMTP monitor that checks through the banner or EHLO provides little additional insight, but generates log entries on the provider side and, in the worst case, questions about why a connection without mail arrives every 5 seconds.

## Checklist

| Setting | Recommendation |
|---|---|
| Persistence profile | None; sticky sessions provide no benefit for SMTP, especially not with a one-member pool |
| Virtual server type | Performance (Layer 4) with FastL4 profile; no intervention in the data stream |
| Target node | FQDN node with Autopopulate instead of a static IP; DNS servers configured on the BIG-IP |
| SNAT | Dedicated SNAT pool with a fixed address known to the provider; no Automap |
| Idle timeout | Above actual sending pauses, starting value 600 s; Reset on Timeout enabled |
| Connection limit | As a safety net on the pool member, e.g., 200 |
| Monitor | TCP, interval 30 s or more; no aggressive SMTP monitor |
| Sender configuration | Few parallel connections, many messages per connection; reuse enabled |

The short answer to the original question is therefore: No, sticky sessions are not better; in this setup, they are ineffective at best and harmful at worst. The quality of the solution depends on DNS resolution of the provider hostname, a stable SNAT address, timeouts that match the load profile, and sending systems delivering their 1,000 emails per minute over a few established connections rather than a thousand individual ones.

## Change log

- **October 25, 2026:** Corrected: The default name-resolution interval for FQDN nodes is 3,600 seconds, not the DNS TTL; the BIG-IP follows the TTL only with the interval ttl setting.

## Sources

1.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): Section 4.5.4 and the transaction model show that multiple mail transactions over one connection are the intended normal case.

2.  [K7820: Overview of SNAT features](https://my.f5.com/manage/s/article/K7820): F5 introductory article on SNAT, SNAT pools, and port translation per destination.

3.  [tmsh reference: ltm node](https://clouddocs.f5.com/cli/tmsh-reference/latest/modules/ltm/ltm_node.html): documents the FQDN options (name, autopopulate, interval) for nodes and therefore pool members, including the default interval of 3,600 seconds and the value `ttl`.

4.  [smtp-source(1), Postfix](https://www.postfix.org/smtp-source.1.html): Load generator that replicates the bulk sender connection profile (few sessions, many messages).

5.  [Determining a Mail Server's Load Profile](https://rafaelpfister.ch/blog/mailserver-lastprofil-ermitteln): In-house guide explaining how to evaluate connections per minute and messages per connection from Message Tracking.

6.  [F5 Bug ID 746657: tmsh help for FQDN node or pool member shows incorrect default for fqdn interval](https://cdn.f5.com/product/bugtracker/ID746657.html): Confirms 3,600 seconds as the actual default value of the FQDN query interval; the earlier help entry stating TTL was incorrect.
