---
title: "Email Encryption: Transport Protection, S/MIME, PGP, and the Gateway Model"
blatt: "verschluesselung"
description: "The layers of email encryption cleanly separated: opportunistic and enforced TLS on the transport, S/MIME and PGP for content, central gateways instead of client-side chaos, and when each level is sufficient."
fakten:
  - { label: "Transport layer", wert: "StartTLS, MTA-STS, DANE (server to server)" }
  - { label: "Content layer", wert: "S/MIME and OpenPGP (end to end or gateway)" }
  - { label: "S/MIME", wert: "RFC 8551, certificate-based", href: "https://datatracker.ietf.org/doc/html/rfc8551" }
  - { label: "OpenPGP", wert: "RFC 9580, web of trust", href: "https://datatracker.ietf.org/doc/html/rfc9580" }
  - { label: "Gateway products", wert: "SEPPmail, Totemomail, HIN Mail Gateway" }
  - { label: "Alternative paths", wert: "GINA and portal delivery for recipients without keys" }
  - { label: "Typical obligation", wert: "healthcare (HIN), public authorities, finance" }
werbung: ["newsletter"]
ctaThemen: ["e-mail-verschluesselung", "seppmail"]
---

# Email Encryption: Transport Protection, S/MIME, PGP, and the Gateway Model

"Is the email encrypted?" is a poor question, because it mixes two layers that have to be considered separately: the path and the content. Keeping the layers apart makes it possible to map requirements from data protection, regulation, and common sense cleanly onto technology without losing the users.

## Level 1: the transport

Between mail servers, **TLS** protects the connection. By default this is opportunistic: the sending server attempts StartTLS but falls back to cleartext if there are problems. Transport protection becomes binding with **MTA-STS** (a policy published over HTTPS and DNS) or **DANE** (certificate binding via DNSSEC); both are DNS topics and therefore pure infrastructure work without user contact. Transport encryption protects against interception along the way, but not on the servers themselves: at the provider, at the gateway, and at the recipient, the message sits in cleartext.

## Level 2: the content

Content encryption makes the message itself unreadable for anyone without a key. Two standards share the field: **S/MIME** works with X.509 certificates and fits enterprise and government environments in which certificates are managed anyway. **OpenPGP** works with key pairs and the web of trust and dominates in technical communities. In practice, both fail at the same point: the recipient needs a key before the first confidential message can be sent, and precisely this key management reliably overwhelms end users.

## The gateway model: encryption as infrastructure

Secure mail gateways such as **SEPPmail**, **Totemomail**, or the **HIN Mail Gateway** solve the chicken-and-egg problem centrally: encryption happens at the gateway instead of at the client. Internal senders write ordinary messages; the gateway signs and encrypts according to policy, manages certificates and keys domain-wide, and automatically negotiates the best method with other gateways. For recipients without any cryptographic equipment, **portal or GINA delivery** remains: the message sits encrypted at the gateway, and the recipient retrieves it through a protected web page. That is the reason gateways are standard in industries with an encryption obligation, in Swiss healthcare in the form of the HIN platform.

The price of the model: the gateway is a central component in the mail flow and therefore business-critical. LDAP integration, certificate lifetimes, releases, and backups of the gateway belong on the same operational checklist as the mail server itself.

## Which level for what

As a rule of thumb: **transport encryption** is the baseline and protects the path, not the message. **Content encryption** (S/MIME or PGP) belongs where the message itself needs protection and both sides can manage keys. The **gateway model** fits when many senders without their own key material regularly send confidential content to changing recipients. Sector platforms, in turn, are a given wherever an industry requires them. The levels do not exclude one another: in practice transport encryption sits underneath everything, and the content level is added where it is needed.
