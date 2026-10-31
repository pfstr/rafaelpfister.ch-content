---
title: "HIN Platform Renewal 2026: Access Gateway, Client, and the Deadlines Through October 26"
navTitle: "Access Gateway 2026"
description: "Firewall approval by August 31, Access Gateway version 4 starting in September, SAML endpoints by October 12, hardware tokens and HIN Client 4.0 by October 26. The mail gateway is not affected and will be replaced separately."
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
translationSourceHash: cd34e6297afca9491bd965e40dbaf55b920296e62e57726754824474fcb27f0d
translationModel: gpt-5.6-terra
translatedAt: 2026-10-30T03:45:26.299Z
translationReview: required
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/en/blog/hin-platform-renewal-2026-access-gateway-client-and-the-deadlines-through-september-14
---

# HIN Platform Renewal 2026: Access Gateway, Client, and the Deadlines Through October 26

In 2026, HIN is renewing its identity and access platform. HIN reset the deadlines on August 27, 2026: the first deadline expired on August 31, 2026, the old SAML endpoints have been unavailable since October 12, 2026, and the major changeover will follow on October 26, 2026.

**The HIN Access Gateway (AGW), the HIN Client, and authentication methods are affected. The HIN Mailgateway is not affected.** It will also be replaced, but as part of a separate initiative with its own timeline.

<div class="choice-row">
  <a class="choice" href="#die-fristen">
    <span class="choice__label">Your situation</span>
    <span class="choice__title">Only AGW in operation</span>
    <span class="choice__hint">The deadlines below are all you need to do. →</span>
  </a>
  <a class="choice" href="/stargate">
    <span class="choice__label">Your situation</span>
    <span class="choice__title">Additional mail gateway migration required</span>
    <span class="choice__hint">Then replacement with “Stargate” is also coming up, with broad rollout starting in Q3 2026. Free assessment of your environment. →</span>
  </a>
</div>

## The deadlines

| Date | Action | Affects |
|---|---|---|
| 08/31/2026 | Firewall approval for `idp.id.hin.ch` and `cdn.hin.ch` (443/tcp) | AGW operators |
| starting September 2026 | Automatic installation of AGW version 4 (excluding cluster installations) | AGW operators |
| September through 10/26/2026 | Phased automatic update to HIN Client 4.0 | All Client users |
| 10/12/2026 | Old SAML endpoints no longer available | Federations, EPR connections |
| 10/26/2026 | Hardware tokens and test identities for EPR access expire | Token users, integrations |
| 10/26/2026 | Existing Authenticator App usable for the last time; setup of the new one starting 10/27/2026 | App users |
| 10/26/2026 | HIN Client 4.0 required | All Client users |
| starting early November 2026 | Browser login in the HIN Client | All Client users |

## Access Gateway is not Mailgateway

Both have Gateway in their name and are regularly confused. The Access Gateway controls access to HIN-protected applications and does not affect mail traffic. The Mailgateway is part of the mail flow and encrypts messages.

## Access Gateway: Firewall and version 4

By August 31, 2026, the AGW had to be able to reach `idp.id.hin.ch` and `cdn.hin.ch` via 443/tcp. HIN recommends setting the rule for hostnames rather than IP addresses because IP addresses may change. This is a firewall change, not a gateway setting, and therefore often falls to the network administrator rather than the gateway administrator.

Since September 2026, version 4 has been installed automatically; cluster installations are excluded, and HIN will contact these customers directly. Prerequisites: AGW version 3.1.50 or later, with Kerberos enabled as the authentication method. Connecting to Active Directory requires an LDAP account with read permissions.

Anyone who does not meet the requirements will not be updated, and experience shows that this is only noticed once no one can log in anymore. It is therefore better to check the version now rather than wait for an outage.

## SAML: new endpoints, fewer attributes

```text
Föderationsdienst
  broker.hin.ch/realms/HINBroker/protocol/saml/descriptor

EPD-Zugang
  idp.id.hin.ch/auth/realms/hinid/protocol/saml/descriptor
```

The changeover also changes attribute formats and bindings. The attribute set will be reduced to GLN, name, date of birth, and gender.

This is where integrations fail. Any application that uses additional attributes for roles or tenant separation will no longer receive them through the new endpoints; the old ones have been unavailable since October 12, 2026. The issue does not appear as a login error, but as a missing authorization in the target system.

Test identities for EPR access expire on October 26, 2026, and must be requested again. For EPR integration, HIN requires identities verified by video identification; a test environment is not yet available. Starting October 26, 2026, assertions without a GLN may also arrive.

Anyone operating a federation almost always also operates their own mail infrastructure. For these organizations, the platform renewal falls in the same year as the [Mailgateway replacement with “Stargate”](/stargate): technically independent, but competing for the same people and maintenance windows.

## Tokens, App, and HIN Client 4.0

Hardware tokens are no longer issued and expire on October 26, 2026. Alternatives: HIN Client, SMS code, or Authenticator App. The existing Authenticator App can be used through October 26, 2026; the new app can be set up as a second factor in the self-service portal starting October 27, 2026. HIN recommends setting up login with a password and SMS code as emergency access beforehand.

The HIN Client will be automatically updated to version 4.0 in phases from September through October 26, 2026. If the Client does not show version 4.0 on October 26, it must be installed manually via `download.hin.ch`. Browser login will follow starting in early November 2026.

The critical point is the system requirements: **version 4.0 requires Windows 11 with TPM or macOS 14**, and Intel Macs additionally require a security chip; without TPM, login with an SMS code remains available. Older devices must be updated or replaced beforehand. For some practices, the deadline is therefore not a software task but a procurement task. Anyone who notices this only shortly before October 26 will face delivery times and reinstallation of their practice software.

## Five questions to assess your situation

1. Which AGW version is running, and is Kerberos enabled?
2. Does the firewall allow outbound access to `idp.id.hin.ch`?
3. How many workstations are still running Windows 10 or macOS 13 and older?
4. How many hardware tokens are in use, and what will affected users switch to?
5. Does any application use HIN attributes that will no longer be available?

The answers to 3 and 5 determine the effort involved. The rest can be completed in a few hours and is documented by HIN.

## The second initiative: “Stargate”

Independently of this, HIN is replacing the Mailgateway with the new HIN Gateway, internally known as project “Stargate,” technically a data mesh approach with end-to-end encryption and decentralized key management. This is not an appliance replacement, but an architectural change.

The effort is therefore on an entirely different level. The platform renewal primarily requires meeting deadlines for a firewall rule, a version level, and a device replacement, whereas Stargate puts the production mail flow itself up for consideration: the established ruleset, key material, handling of recipients without a HIN identity, and the question of what to fall back on if something does not work as expected. Because migration takes place in booked four-hour windows and HIN recommends one month of preparation, such an appointment leaves no room for unresolved issues.

<aside class="offer-box">
  <span class="offer-box__tag">Free assessment</span>
  <p><strong>You do not need to know where you stand. That is exactly what the assessment is for.</strong> I will review your existing gateway environment and tell you what needs to be done before the migration window, regardless of whether you migrate yourself afterward or get support.</p>
  <a class="offer-box__cta" href="/stargate">Register now</a>
</aside>

## Change log

- **October 25, 2026:** Updated deadlines according to HIN’s reset of August 27, 2026: firewall approval (now also for cdn.hin.ch) by August 31, old SAML endpoints by October 12, hardware tokens, the existing Authenticator App, and HIN Client 4.0 by October 26, 2026. Added: TPM requirement for HIN Client 4.0 and browser login starting in early November 2026.

## Sources

1.  [HIN platform renewal: These technical adjustments are required for HIN members](https://www.hin.ch/de/blog/2026/technische-anpassungen.cfm): deadlines from August through October 2026 (as of August 27, 2026), SAML endpoints, reduced attribute set, firewall approvals, Authenticator App, and hardware tokens.

2.  [The new HIN Client is here: What is changing for HIN members](https://www.hin.ch/de/blog/2026/neuer-hin-client.cfm): version 4.0, rollout through October 26, 2026, operating system and TPM requirements, browser-based login starting in November 2026.

3.  [HIN Gateway: Secure communication within the HIN Community](https://www.hin.ch/de/services/hin-mail/hin-gateway.cfm): replacement of the Mailgateway, architecture, operating models, migration in booked time windows.

4.  [Configuring the HIN Access Gateway](https://cdn.hin.ch/agw/manual/DE/4-konfiguration-des-hin-access-gateway.html): role of the AGW in access management.

5.  [Active Directory connection](https://cdn.hin.ch/agw/manual/DE/5-anbindung-active-directory.html): Kerberos and the LDAP account with read permissions.

6.  [HIN AG: “From Mailgateway to Data Mesh”](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm): background on “Stargate,” decentralized nodes, timeline.
