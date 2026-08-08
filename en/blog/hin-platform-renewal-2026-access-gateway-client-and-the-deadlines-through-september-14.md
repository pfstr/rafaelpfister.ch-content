---
title: "HIN Platform Renewal 2026: Access Gateway, Client, and the Deadlines Through September 14"
navTitle: "Access Gateway 2026"
description: "Firewall approval by August 14, Access Gateway version 4 from August 17, SAML endpoints, hardware tokens, and HIN Client by September 14. The mail gateway is not affected and will be replaced separately."
date: "2026-08-01"
kategorie: "HIN Gateway"
timeToRead: "5 min read"
themen:
  - hin-gateway
  - active-directory-entra
produkte:
  - "hin"
protokolle:
  - "tcp"
  - "migration"
related:
  - hin-mailgateway-backup-disaster-recovery
  - hin-update-issue-version-15.0.5
slug: "hin-platform-renewal-2026-access-gateway-client-and-the-deadlines-through-september-14"
translationId: "article-106aa61d54408397"
translationOf: hin-plattformerneuerung-2026
url: https://rafaelpfister.ch/en/blog/hin-platform-renewal-2026-access-gateway-client-and-the-deadlines-through-september-14
translationSourceHash: 1a174bd131b8bb29f9b1e1e793d4cf19b3f732e3c6fd779a25193f151ec8c109
translationModel: gpt-5.6-terra
translatedAt: 2026-08-08T14:07:17.505Z
translationReview: required
---

# HIN Platform Renewal 2026: Access Gateway, Client, and the Deadlines Through September 14

In 2026, HIN is renewing its identity and access platform. The first deadline is August 14, 2026, followed by the major transition on September 14, 2026.

**The HIN Access Gateway (AGW), HIN Client, and authentication methods are affected. The HIN Mailgateway is not affected.** It will also be replaced, but under a separate initiative with its own timeline.

<div class="choice-row">
  <a class="choice" href="#die-fristen">
    <span class="choice__label">Your situation</span>
    <span class="choice__title">Only AGW in operation</span>
    <span class="choice__hint">The deadlines below cover everything you need to do. →</span>
  </a>
  <a class="choice" href="/stargate">
    <span class="choice__label">Your situation</span>
    <span class="choice__title">Additional mail gateway migration required</span>
    <span class="choice__hint">The replacement with “Stargate” is also coming up, with a broader rollout starting in Q3 2026. Free assessment of your environment. →</span>
  </a>
</div>

## The deadlines

| Date | Action | Affects |
|---|---|---|
| 08/14/2026 | Firewall approval for `idp.id.hin.ch` (`185.154.38.46`, `193.168.215.45`) | AGW operators |
| 08/17/2026 | Automatic installation of AGW version 4 | AGW operators |
| from mid-August | Manual installation of HIN Client 4.0 recommended | All client users |
| 09/14/2026 | SAML endpoints migrated | Federations, EPR connections |
| 09/14/2026 | Hardware tokens and test identities expire | Token users, integrations |
| 09/14/2026 | Reconfigure the Authenticator app | App users |
| 09/14/2026 | Forced update to HIN Client 4.0 | All client users |

## Access Gateway is not Mailgateway

Both have Gateway in their names and are regularly confused. The Access Gateway controls access to HIN-protected applications and does not affect email traffic. The Mailgateway sits in the mail flow and encrypts messages.

## Access Gateway: Firewall and Version 4

By August 14, the AGW must be able to reach `idp.id.hin.ch`. This is a firewall change, not a setting in the gateway, and is therefore often the responsibility of the network administrator rather than the gateway administrator.

Starting August 17, version 4 will be installed automatically. Requirements: AGW version 3.1.50 or later, and Kerberos enabled as the authentication method. Connecting to Active Directory requires an LDAP account with read permissions.

Those who do not meet the requirements will not be updated, and experience shows that this often only becomes apparent when no one can log in anymore. It is therefore better to check the version now rather than in September.

## SAML: New Endpoints, Fewer Attributes

```text
Föderationsdienst
  broker.hin.ch/realms/HINBroker/protocol/saml/descriptor

EPD-Zugang
  idp.id.hin.ch/auth/realms/hinid/protocol/saml/descriptor
```

The transition changes attribute formats and bindings. The set of attributes is reduced to GLN, name, date of birth, and gender.

This is where integrations break. Any application that uses additional attributes for roles or tenant separation will no longer receive them after September 14. The issue will not appear as a login error, but as missing authorization in the target system.

Test identities expire on the same date, so anyone wishing to test the transition in an integration environment should do so beforehand.

Organizations that operate a federation almost always also run their own mail infrastructure. For these organizations, the platform renewal coincides with the [replacement of the Mailgateway with “Stargate”](/stargate) in the same year: technically independent, but competing for the same people and maintenance windows.

## Tokens, App, and HIN Client 4.0

Hardware tokens will no longer be issued and expire on September 14. Alternatives include HIN Client, SMS code, or the Authenticator app. The app itself remains valid until September 14 and must then be reconfigured via the self-service portal.

HIN Client will be automatically updated to version 4.0 by September 14 at the latest; manual installation is available from mid-August via `download.hin.ch`. Logins will now take place through the browser.

The critical point is the system requirements: **Version 4.0 requires Windows 11 or macOS 14.** Older devices must be updated or replaced beforehand. For some practices, the deadline is therefore not a software task but a procurement task. Anyone who realizes this only in September will face delivery times and reinstallation of practice software.

## Five Questions to Assess Your Situation

1. Which AGW version is running, and is Kerberos enabled?
2. Does the firewall allow outbound access to `idp.id.hin.ch`?
3. How many workstations are still running Windows 10 or macOS 13 and earlier?
4. How many hardware tokens are in use, and what will affected users switch to?
5. Does any application use HIN attributes that will no longer be available?

The answers to 3 and 5 determine the effort required. The rest can be completed in a few hours and is documented by HIN.

## The Second Initiative: “Stargate”

Independently of this, HIN is replacing the Mailgateway with the new HIN Gateway, internally known as project “Stargate,” technically a data mesh approach with end-to-end encryption and decentralized key management. This is not an appliance replacement, but an architectural change.

The effort is therefore on an entirely different level. The platform renewal primarily requires meeting deadlines for a firewall rule, a version status, and device replacement, whereas Stargate puts the production mail flow itself up for review: the established rule set, key material, handling recipients without a HIN identity, and the question of what to fall back on if something does not go as expected. Since the migration takes place in booked four-hour windows and HIN recommends one month of preparation, such an appointment leaves no room for open issues.

<aside class="offer-box">
  <span class="offer-box__tag">Free assessment</span>
  <p><strong>You do not need to know where you stand. That is exactly what the assessment is for.</strong> I will review your existing gateway environment and tell you what needs to be done before the migration window, regardless of whether you migrate yourself afterward or choose to receive support.</p>
  <a class="offer-box__cta" href="/stargate">Register now</a>
</aside>

## Sources

1.  [HIN Platform Renewal: These Technical Adjustments Are Required for HIN Members](https://www.hin.ch/de/blog/2026/technische-anpassungen.cfm): deadlines in August and September, SAML endpoints, reduced attribute set, firewall approvals.

2.  [The New HIN Client Is Here: What Changes for HIN Members](https://www.hin.ch/de/blog/2026/neuer-hin-client.cfm): version 4.0, operating system requirements, browser-based login.

3.  [HIN Gateway: Secure Communication Within the HIN Community](https://www.hin.ch/de/services/hin-mail/hin-gateway.cfm): replacement of the Mailgateway, architecture, operating models, migration in booked time windows.

4.  [Configuring the HIN Access Gateway](https://cdn.hin.ch/agw/manual/DE/4-konfiguration-des-hin-access-gateway.html): role of the AGW in access management.

5.  [Connecting Active Directory](https://cdn.hin.ch/agw/manual/DE/5-anbindung-active-directory.html): Kerberos and the LDAP account with read permissions.

6.  [HIN AG: “From Mailgateway to Data Mesh”](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm): background on “Stargate,” decentralized nodes, timeline.
