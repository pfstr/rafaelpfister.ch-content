---
title: "SEPPmail: The Secure Mail Gateway with GINA Delivery"
blatt: "seppmail"
description: "How the SEPPmail Secure E-Mail Gateway works: S/MIME and domain encryption at the gateway, GINA for recipients without keys, its position in the mail flow, LDAP integration and the operational topics from firmware to cluster."
fakten:
  - { label: "Product", wert: "SEPPmail Secure E-Mail Gateway (appliance, virtual or cloud)" }
  - { label: "Vendor", wert: "SEPPmail AG, Switzerland", href: "https://www.seppmail.com/" }
  - { label: "Purpose", wert: "Signing and encryption centrally at the gateway" }
  - { label: "Methods", wert: "S/MIME, OpenPGP, domain encryption, TLS" }
  - { label: "Without recipient key", wert: "GINA (password-protected web delivery)" }
  - { label: "Integration", wert: "SMTP in the mail path · LDAP(S) to the directory" }
  - { label: "Administration", wert: "Web GUI, with LDAP login for admins since 15.0.6" }
werbung: ["newsletter"]
ctaThemen: ["seppmail"]
---

# SEPPmail: The Secure Mail Gateway with GINA Delivery

In German-speaking countries, SEPPmail is one of the most widely deployed representatives of the gateway model: encryption and signing happen not on the client but centrally on an appliance in the mail flow. Users write ordinary messages; the cryptography is handled by the gateway according to policy. Email encryption thereby turns from an end-user problem into an infrastructure task, with everything that entails: operations, updates, directory integration, monitoring.

## Position in the mail flow

The gateway sits as an SMTP station between the mail system (Exchange, M365 or another server) and the internet. Outbound, it signs messages with S/MIME, automatically encrypts to recipients whose key is known and applies corporate policies (for example: always encrypt to domain X). Inbound, it decrypts, verifies signatures and passes the cleaned messages on to the mail system. Integration follows the classic connector pattern: Exchange Online routes outbound traffic through the gateway, and inbound it accepts mail only from the gateway's addresses. Forgotten side entrances that bypass the gateway are the most common configuration error in this model, not the gateway itself.

## GINA: encryption without a counterpart

The distinguishing feature of many SEPPmail installations is **GINA**: if the recipient has neither an S/MIME certificate nor a PGP key, the message is delivered as an encrypted container that the recipient opens in a browser with a password set once. Replies travel back over the same protected channel. Confidential communication therefore also works with counterparts that have no cryptographic equipment at all, which is the deciding factor in sectors such as healthcare, fiduciary services and public administration.

## Directory and key management

In production, the gateway stands or falls with two integrations. **LDAP(S)** to Active Directory supplies users, groups and addresses for policies and certificate assignment; since firmware 15.0.6, administrators of the web GUI can also authenticate against the directory, including group mapping, which removes local admin accounts. **Key management** is handled by the gateway itself: user and domain certificates are generated, renewed and stored centrally. The gateway thus holds the organization's key material in one place; configuration and key backups are correspondingly part of the operating concept, because encrypted historical mail is unreadable without the keys.

## Operations

Recurring tasks are manageable and can be scheduled: **certificates** for the web GUI, SMTP TLS and signing along with their expiry dates, **firmware levels** including the release notes before installation, **backups** of the configuration and key material, and **watching the mail flow**. When something breaks, the order is stable: first check acceptance and relaying at the SMTP level, then the matching policy, then directory and keys. Most outages in practice trace back to expired certificates, changed directory structures and adjustments to connectors.
