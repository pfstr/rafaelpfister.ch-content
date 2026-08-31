---
title: "AuthMechanism 10 and AuthAs Internal: How Exchange Classifies Message Submission in the Header"
navTitle: "AuthMechanism 10"
description: "The X-MS-Exchange-Organization-AuthMechanism header documents how a submitting server authenticated. Value 10 indicates a Receive Connector with Externally Secured and classifies external emails as internal, with consequences for spam filters, mail flow rules, and spoofing protection."
date: "2026-08-26"
featured: "2026-08-27"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "8 min read"
themen:
  - exchange-onprem-hybrid
  - microsoft-365-exchange
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange"
  - "exchange-hybrid"
protokolle:
  - "smtp"
  - "troubleshooting"
related:
  - exchange-hybrid-header-intern-extern
  - exchange-message-tracking-und-receive-connectoren-analysieren
  - typische-ursachen-fuer-mail-loops-und-deren-behebung
slug: "authmechanism-10-and-authas-internal-how-exchange-classifies-message-submission-in-the-header"
translationId: "article-0df383d5c49016da"
translationOf: exchange-authmechanism-10-authas-internal
url: https://rafaelpfister.ch/en/blog/authmechanism-10-and-authas-internal-how-exchange-classifies-message-submission-in-the-header
translationSourceHash: 5a9335a90afc9bf7df78b908f71b679f64c29f3b9e96bd7f25bcc916123b82df
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:17:25.101Z
translationReview: automatic
---

# AuthMechanism 10 and AuthAs Internal: How Exchange Classifies Message Submission in the Header

When analyzing spam, spoofing, and mail flow cases in Exchange environments, three headers stamped by Exchange upon receipt are crucial:

```text
X-MS-Exchange-Organization-AuthAs: Internal
X-MS-Exchange-Organization-AuthSource: EX01.example.com
X-MS-Exchange-Organization-AuthMechanism: 10
```

`AuthAs` records how the sender presented itself to transport. `AuthSource` identifies the server that performed the evaluation. `AuthMechanism` documents the mechanism through which authentication was established. Together, they determine whether Exchange treats a message as internal or external, and this classification has significant consequences.

## Why the classification matters

`AuthAs` has two values in practice: `Internal` and `Anonymous`. A message classified as `Internal` is handled differently from external mail:

- Mail flow rules with the condition “sender is outside the organization” do not apply.
- The message can be delivered to distribution groups and mailboxes that require authenticated senders (`RequireSenderAuthenticationEnabled`).
- Anti-spam and anti-spoofing checks are less strict or omitted; in hybrid environments, the external disclaimer is not appended and Outlook does not display an “External” indicator.
- The display name is resolved from the address book, making the email appear to recipients as internal mail.

That is precisely why the question “AuthAs Internal or Anonymous?” should be at the start of every header analysis: It helps clarify why an obvious spoofed email passed the spam filter or why a mail flow rule never triggered.

## The AuthMechanism values

Microsoft does not fully document the encoding of `AuthMechanism` publicly. Two values are relevant for troubleshooting and well documented:

| Value | Meaning |
|---|---|
| `04` | Authenticated Exchange traffic: mailbox-to-mailbox within the organization, as well as hybrid traffic through connectors configured by the Hybrid Configuration Wizard. |
| `10` | Receive Connector with the authentication option `ExternalAuthoritative` (“externally secured”): The connection is considered secured outside Exchange, and everything submitted through it is treated as internal. |

Other values appear in headers but have no official reference. In practice, the distinction is sufficient: `04` means genuine Exchange authentication; `10` means trust based on connector configuration.

## What Externally Secured really means

The `ExternalAuthoritative` option on a Receive Connector tells Exchange: Someone else handles securing this connection, such as a firewall, a dedicated network segment, or IPsec. Exchange then performs no further checks and treats every submission through this connector as authenticated and internal, including the right to use any internal sender addresses.

This is intended for only a few scenarios, such as a fully trusted application server in your own data center. It becomes problematic when the connector points to an upstream mail gateway or spam filter in the DMZ through which Internet mail also arrives. Every external email then carries the following after submission:

```text
X-MS-Exchange-Organization-AuthAs: Internal
X-MS-Exchange-Organization-AuthMechanism: 10
```

The consequences: External emails are considered internal, mail flow rules for external senders do not apply, spoofing protection for the organization's own domain is ineffective, and anyone who can reach the gateway can deliver using internal sender addresses to recipients that actually require authenticated senders.

## Finding affected connectors

The Exchange Management Shell shows which Receive Connectors are configured with `ExternalAuthoritative`:

```powershell
Get-ReceiveConnector | Where-Object {
  $_.AuthMechanism -match "ExternalAuthoritative"
} | Format-Table Identity, RemoteIPRanges, AuthMechanism, PermissionGroups
```

For each result, check which `RemoteIPRanges` are configured and whether the systems behind them actually need this trust. A gateway that only needs to relay mail does not.

## The alternative for relay scenarios

If a system only needs to relay anonymously through Exchange (printers, applications, monitoring), an anonymous relay connector is the cleaner solution: anonymous submission plus the right to deliver to any recipients, but without the Internal classification.

```powershell
New-ReceiveConnector -Name "Anonymous Relay" -TransportRole FrontendTransport `
  -RemoteIPRanges 192.0.2.10 -Bindings 0.0.0.0:25 -Usage Custom -PermissionGroups AnonymousUsers

Get-ReceiveConnector "EX01\Anonymous Relay" | Add-ADPermission `
  -User "NT AUTHORITY\ANONYMOUS LOGON" -ExtendedRights "ms-Exch-SMTP-Accept-Any-Recipient"
```

Emails through this connector remain `AuthAs: Anonymous`, undergo the usual checks, and cannot spoof internal senders. `ExternalAuthoritative` should be reserved for systems to which you deliberately want to grant the right to use internal sender addresses.

## Reading headers in context

The fastest way to determine whether a specific message was classified as internal or external and how it arrived is to examine the complete header: `AuthAs`, `AuthMechanism`, and `AuthSource` together with the `Received` chain. The [Mail Header Analyzer](/tools/header-analyzer) on this website evaluates these fields directly in the browser and highlights the hybrid classification in the delivery path; the header does not leave the browser.

The article [Internal or external? Classifying Exchange hybrid emails in the header](/blog/exchange-hybrid-header-intern-extern) explains how the classification is retained between Exchange Online and OnPrem in hybrid environments and how to identify an incorrect assignment.

## Sources

1.  [Microsoft Q&A: Exchange 2016 mail flow rule, which header is checked for "outside the organization"?](https://learn.microsoft.com/en-us/answers/questions/54418/exchange-2016-mail-flow-rule-which-header-is-check): Mapping of AuthAs and AuthMechanism 10 to the Externally Secured configuration and its effect on mail flow rules.

2.  [Microsoft Tech Community: Demystifying and troubleshooting hybrid mail flow: when is a message internal?](https://techcommunity.microsoft.com/blog/exchange/demystifying-and-troubleshooting-hybrid-mail-flow-when-is-a-message-internal/1420838): Official description of the Internal classification and its consequences in hybrid mail flow.

3.  [msxfaq: X-MS-Exchange-Organization-AuthAs](https://www.msxfaq.de/cloud/exchangeonline/transport/x-ms-exchange-organization-authas.htm): Observed AuthAs, AuthSource, and AuthMechanism values in different submission scenarios.

4.  [Microsoft Learn: Allow anonymous relay on Exchange servers](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/allow-anonymous-relay): Configuring the anonymous relay connector as an alternative to Externally Secured.
