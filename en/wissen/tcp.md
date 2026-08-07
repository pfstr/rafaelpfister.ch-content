---
title: "TCP and Networking Basics: What Mail Admins Really Need"
blatt: "tcp"
description: "TCP as the foundation of SMTP, LDAP, and HTTPS: connection setup with the three-way handshake, ports and firewalls, typical failure patterns from timeout to connection refused, and the steps for a quick diagnosis."
fakten:
  - { label: "Full name", wert: "Transmission Control Protocol" }
  - { label: "Purpose", wert: "Reliable, connection-oriented transmission over IP" }
  - { label: "Introduced", wert: "1981 (RFC 793) · currently RFC 9293" }
  - { label: "OSI layer", wert: "Transport (layer 4)" }
  - { label: "Standard", wert: "RFC 9293", href: "https://datatracker.ietf.org/doc/html/rfc9293" }
  - { label: "Port range", wert: "0 to 65535, registry maintained by IANA", href: "https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml" }
  - { label: "Mail ports", wert: "25/587/465 (SMTP) · 389/636 (LDAP) · 443 (HTTPS/Graph)" }
  - { label: "Tools", wert: "nc, telnet, ss, tcpdump, Test-NetConnection" }
werbung: ["newsletter"]
ctaThemen: ["smtp-mailflow"]
---

# TCP and Networking Basics: What Mail Admins Really Need

Whether SMTP, LDAP, HTTPS, or the management interface of an appliance: practically everything a mail admin operates runs on TCP. The good news is that day-to-day operations require no study of window sizes and congestion algorithms. What they do require is a solid grasp of connection setup, ports, and the three failure patterns that explain nearly every incident.

## The three-way handshake

A TCP connection begins with three packets: the client sends **SYN**, the server answers with **SYN-ACK**, and the client confirms with **ACK**. Only then does data flow, such as the server's SMTP banner. This detail is diagnostic gold, because it separates network problems from application problems: if the handshake completes but no banner arrives, the problem lies with the service. If the handshake does not complete at all, the cause is the network, a firewall, or a service that is not listening.

## Ports are conventions

A port is nothing more than a number on which a service listens. The well-known assignments (25 for SMTP, 636 for LDAPS, 443 for HTTPS) are conventions from the IANA registry, not laws of nature; any appliance can be configured differently. For firewalls this means that rules describe a destination IP and a destination port. When a vendor such as HIN or Microsoft introduces new endpoints, the "migration" is often just a new firewall rule from a network point of view, one that has to be in place in time.

## The three failure patterns

- **Connection refused**: the destination actively answers with a rejection (RST). The host is reachable, but nothing is listening on that port. Typical causes: the service is stopped, the port is wrong, or a NAT rule points nowhere.
- **Timeout**: no answer arrives at all. Typical causes: a firewall drops the traffic silently, a route is missing, or the IP is wrong. Timeouts feel "slow", but they are almost always a hard connectivity problem.
- **Connection established, application fails**: the handshake succeeds, then TLS or the protocol breaks off. Typical causes: certificate errors, protocol versions, or an intermediate device that interferes with the connection.

Anyone who keeps these three patterns apart has usually narrowed the search for the cause down to a single team before the first ticket escalates.

## Diagnosis in one minute

```bash
nc -vz mail.example.ch 25          # Does a TCP connection come up?
nc -vz dc1.example.ch 636          # Is LDAPS reachable?
ss -tlnp | grep :25                # Is the service listening locally at all?
```

On Windows, `Test-NetConnection -ComputerName mail.example.ch -Port 25` does the same job. For the cases in between, `tcpdump` (or Wireshark) shows whether SYN packets reach the destination and what comes back. More is rarely needed: most "mail is not working" cases die on one of these three levels.
