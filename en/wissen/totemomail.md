---
title: "Totemomail: Email Encryption on the Kiteworks Platform"
blatt: "totemomail"
description: "The Totemomail encryption gateway in operation: origins and the takeover by Kiteworks, gateway encryption with S/MIME and PGP, WebMail delivery, LDAP integration, the licensing model based on licensed users, and the M365 integration."
fakten:
  - { label: "Product", wert: "Totemomail (email encryption platform)" }
  - { label: "Vendor", wert: "totemo ag, Switzerland; part of Kiteworks since 2022", href: "https://www.kiteworks.com/" }
  - { label: "Purpose", wert: "Central signing and encryption in the mail flow" }
  - { label: "Methods", wert: "S/MIME, OpenPGP, TLS, WebMail portal" }
  - { label: "Integration", wert: "SMTP in the mail path · LDAP(S) to the directory · Graph/EWS to M365" }
  - { label: "Licensing model", wert: "licensed (internal) users, external users unlimited" }
  - { label: "Platform", wert: "Java-based, virtual or as an appliance" }
werbung: ["newsletter"]
ctaThemen: ["totemomail"]
---

# Totemomail: Email Encryption on the Kiteworks Platform

Like SEPPmail, Totemomail belongs to the family of secure mail gateways: encryption, decryption, and signing run centrally in the mail flow instead of on the clients. Developed by the Swiss company totemo ag, the product has belonged to Kiteworks since the takeover in 2022 and is continued there as part of a broader platform for protected content exchange. Existing installations are operated further under the Kiteworks umbrella.

## How it works in the mail flow

The gateway sits as an SMTP hop between the mail system and the internet. Policies decide per message whether it is signed, encrypted, or delivered in cleartext; S/MIME and OpenPGP are supported equally, including automatic key exchange with peers. For recipients without keys of their own, **WebMail delivery** is available: the message stays on the gateway, and the recipient reads and answers it through a password-protected portal. Totemomail thus covers the same use cases that carry the gateway model in general: professional secrecy obligations, sensitive personal data, and regulated communication.

## Directory and licensing logic

The **LDAP(S) integration** with Active Directory supplies the gateway with users and groups for policies and certificate assignment. One characteristic deserves particular attention: the **licensing model based on licensed users**. Counted are internal users who use encryption functions; external communication partners are free. If the directory grows uncontrolled (orphaned accounts, shared mailboxes, service accounts), the license counter fills up until the gateway rejects new users. Regular cleanup via LDAP queries, ideally automated, is therefore part of standard operations.

## Integration with Microsoft 365

In M365 environments, the gateway is attached to the Exchange Online mail flow through connectors: outbound traffic is forced through Totemomail, and inbound traffic is accepted only from its addresses. For additional functions that require a mailbox, the modern path leads through an **Entra app registration with Graph permissions** and certificate authentication rather than through service accounts with passwords; application access policies limit access to exactly the mailboxes required.

## Operational topics

Operations revolve around four recurring items: **certificate lifetimes** for transport, signing and the web interface, **license and user counts** with their mapping in the directory, **version changes** including a review of the release notes beforehand, and the **connections** to directory and mail system. When something breaks, the mail flow logs are the first place to look: was the message accepted, which policy applied, and does the recipient exist in the directory in the form the policy expects.
