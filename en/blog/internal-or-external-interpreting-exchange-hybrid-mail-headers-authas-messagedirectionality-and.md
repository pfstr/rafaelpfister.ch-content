---
title: "Internal or External? Interpreting Exchange Hybrid Mail Headers: AuthAs, MessageDirectionality, and X-originatororg"
navTitle: "Reading Hybrid Headers"
description: "In Exchange hybrid environments, header classification determines whether an email is treated as internal. Which headers carry the classification, how tenant attribution works through certificates and connectors, and how to identify a misrouted message."
date: "2026-08-26"
kategorie: "Exchange On-Premises / Hybrid"
timeToRead: "10 min read"
themen:
  - exchange-onprem-hybrid
  - microsoft-365-exchange
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange-hybrid"
  - "hybrid-mailfluss"
  - "exchange-online"
protokolle:
  - "smtp"
  - "troubleshooting"
related:
  - exchange-authmechanism-10-authas-internal
  - microsoft-365-compauth-reason-codes
  - einliefernde-ip-adressen-aggregieren
slug: "internal-or-external-interpreting-exchange-hybrid-mail-headers-authas-messagedirectionality-and"
translationId: "article-c8d7859be8dbfe63"
translationOf: exchange-hybrid-header-intern-extern
url: https://rafaelpfister.ch/en/blog/internal-or-external-interpreting-exchange-hybrid-mail-headers-authas-messagedirectionality-and
translationSourceHash: 5a0eccedd4b1a194461602319f5f1a8f59de204c1710e261c2358591bb720dfb
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:18:46.825Z
translationReview: automatic
---

# Internal or External? Interpreting Exchange Hybrid Mail Headers: AuthAs, MessageDirectionality, and X-originatororg

In a hybrid environment, mail between Exchange On-Premises and Exchange Online should be treated as internal mail: no spam filter in between, no "External" label, delivery to protected distribution groups, resolved display names. Whether this works is determined not by the sender domain, but by a handful of headers that must be preserved on the path between the two worlds. Anyone who can read them can answer the most common hybrid questions directly from the headers: Did the message arrive through the hybrid connector? Why was it classified as external? And which tenant was it attributed to?

## The headers involved

```text
X-MS-Exchange-Organization-AuthAs: Internal
X-MS-Exchange-Organization-AuthSource: EX01.example.com
X-MS-Exchange-Organization-MessageDirectionality: Originating
X-OriginatorOrg: example.com
```

**`AuthAs`** carries the classification: `Internal` or `Anonymous`. It is the result of the other signals and the most direct indicator of how Exchange handled the message.

**`AuthSource`** specifies the FQDN of the server that performed the classification: an on-premises server, a mailbox server in Exchange Online, or an EOP frontend. This indicates on which side the evaluation took place.

**`MessageDirectionality`** distinguishes `Originating` (the message originated within the organization, in Exchange Online or through an authenticated inbound connector) from `Incoming` (the message came in from outside).

**`X-OriginatorOrg`** identifies the sender organization from Exchange Online's perspective: the default or matching accepted domain of the sending tenant. The header is set when sending from Exchange Online through the XOORG SMTP extension and is tied to the combination of EOP TLS certificate, connector configuration, and accepted domain. It therefore cannot be spoofed simply by including it: an `X-OriginatorOrg` submitted from outside without the associated trust relationship is not recognized as such.

In addition, there are the `X-MS-Exchange-CrossTenant-*` headers that Exchange Online stamps when crossing between tenants, including `X-MS-Exchange-CrossTenant-AuthAs`. They reflect the classification from the receiving tenant's perspective.

## How the trust relationship works technically

Internal classification across the organizational boundary is based on two components configured by the Hybrid Configuration Wizard:

First, the **inbound connector** in Exchange Online of the OnPremises type, which identifies the submitting source through the TLS certificate (`TlsSenderCertificateName`) or its IP address. Exchange Online also uses this mapping to determine which tenant a submission is attributed to.

Second, the **`CloudServicesMailEnabled`** flag on the connectors on both sides. It ensures that the `X-MS-Exchange-Organization-*` headers (cross-premises headers) are retained during the transition rather than removed as they are for external mail. If the flag is missing or the message takes a route without this configuration, the headers are lost and the message arrives as `Anonymous`.

This leads to the most important diagnostic rule: a hybrid message is internal only if it actually took the route configured by the HCW.

## Case 1: The message arrives as Anonymous even though it should be internal

This is the most common issue: messages from on-premises mailboxes appear as external in Exchange Online, with spam scanning, an "External" label, or rejection by protected distribution groups. The causes, in descending order of frequency:

- **Incorrect route:** The message did not go through the hybrid connector, but through MX instead (and thus through EOP as Internet mail) or through an upstream gateway that removes cross-premises headers or terminates the TLS connection. This is visible in the `Received` chain: instead of the direct handoff from on-premises to `*.mail.protection.outlook.com` through the connector, intermediate hops appear.
- **Certificate replacement:** The on-premises certificate was renewed, but `TlsSenderCertificateName` on the inbound connector was not updated. Certificate-based identification no longer works.
- **Connector configuration changed:** `CloudServicesMailEnabled` was disabled during troubleshooting, or a manually created connector replaced the HCW connector without the necessary settings.

Check this on the Exchange Online side:

```powershell
Get-InboundConnector | Format-List Name, ConnectorType,
  TlsSenderCertificateName, SenderIPAddresses, CloudServicesMailEnabled
```

In Message Trace, the `ConnectorName` field shows whether the message was actually submitted through the expected connector.

## Case 2: Attribution to the wrong tenant

Exchange Online attributes every incoming message to a tenant; the `X-EOPTenantAttributedMessage` header contains the GUID of the attributed tenant. If two tenants use the same `TlsSenderCertificateName` or the same `SenderIPAddresses` in their inbound connectors, for example with a shared gateway service provider or after a migration, a message can be attributed to the wrong tenant. It then does not appear in the Message Trace of the organization's own tenant and is subject to another tenant's transport rules.

`Get-OrganizationConfig | Select-Object GUID` returns the GUID of the organization's own tenant; if it does not match the header, the connector identifiers must be separated: a separate certificate or separate IP ranges per tenant.

## Case 3: Mail classified as external is still treated as internal

The reverse case occurs on-premises: a Receive Connector with the `ExternalAuthoritative` option ("Externally secured") classifies everything submitted through it as internal, identifiable by `AuthAs: Internal` in combination with `AuthMechanism: 10`. If such a connector points to a gateway through which Internet mail also passes, external mail is considered internal, with all the consequences for spam filtering and spoofing protection. Details and countermeasures are covered in the article [AuthMechanism 10 and AuthAs Internal](/blog/exchange-authmechanism-10-authas-internal).

## Reading the header as a whole

To classify a specific message, all signals are needed together: the `Received` chain for the actual route, `AuthAs`/`AuthSource`/`MessageDirectionality` for the classification, and `X-OriginatorOrg` and the CrossTenant headers for the originating organization. The [Mail Header Analyzer](/tools/header-analyzer) on this website evaluates these fields directly in the browser and highlights tenant transitions and hybrid classification in the delivery path; the header never leaves the browser.

## Sources

1.  [Microsoft Tech Community: Demystifying and troubleshooting hybrid mail flow: when is a message internal?](https://techcommunity.microsoft.com/blog/exchange/demystifying-and-troubleshooting-hybrid-mail-flow-when-is-a-message-internal/1420838): Official description of internal classification, the headers involved, and connector requirements.

2.  [Microsoft Tech Community: Advanced Office 365 Routing: Locking Down Exchange On-Premises when MX points to Office 365](https://techcommunity.microsoft.com/blog/exchange/advanced-office-365-routing-locking-down-exchange-on-premises-when-mx-points-to-/609238): How XOORG and X-OriginatorOrg work when routing between Exchange Online and on-premises.

3.  [Microsoft Learn (archive): Use headers to determine which Exchange Online tenant a message was attributed to](https://learn.microsoft.com/en-us/archive/blogs/eopfieldnotes/use-headers-to-determine-which-exchange-online-tenant-a-message-was-attributed-to): X-EOPTenantAttributedMessage and the procedure for incorrect tenant attribution.

4.  [Microsoft Learn: Configure mail flow using connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow): Reference for inbound connector types, TlsSenderCertificateName, and attribution.
