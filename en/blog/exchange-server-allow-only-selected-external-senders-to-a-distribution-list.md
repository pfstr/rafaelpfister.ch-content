---
title: "Exchange Server: Allow only selected external senders to a distribution list"
navTitle: "Distribution list: external senders"
description: "A distribution list in Exchange Server can be configured so that only specific external senders can reach it. Two approaches using PowerShell: delivery restrictions with mail contacts or a transport rule. Also covers the limits of sender verification and how DMARC closes them."
date: "2026-11-05"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "7 min read"
themen:
  - exchange-onprem-hybrid
  - microsoft-365-exchange
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange"
  - "exchange-on-premises"
protokolle:
  - "smtp"
  - "mail-auth"
  - "powershell"
related:
  - exchange-authmechanism-10-authas-internal
  - exchange-message-tracking-und-receive-connectoren-analysieren
slug: "exchange-server-allow-only-selected-external-senders-to-a-distribution-list"
translationId: "article-b3dc70052f5e8729"
translationOf: exchange-verteiler-externe-absender
url: https://rafaelpfister.ch/en/blog/exchange-server-allow-only-selected-external-senders-to-a-distribution-list
translationSourceHash: cff4b579945841f2469f23b5e44d834f6be7528cf9817f9d7ade0fd613f1c582
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:21:41.831Z
translationReview: automatic
---

# Exchange Server: Allow only selected external senders to a distribution list

By default, a distribution list in Exchange Server accepts messages only from authenticated, meaning internal, senders. If a partner, vendor, or customer should also be able to reach the list without opening it to the entire internet, there are two options: a delivery restriction on the group itself or a transport rule. Both work in Exchange Server 2016, 2019, and the Subscription Edition.

## Starting point: RequireSenderAuthenticationEnabled

The `RequireSenderAuthenticationEnabled` property controls whether a group accepts external messages at all. If it is set to `$true` (the default for new distribution groups), Exchange rejects every message from an unauthenticated sender. Both approaches below therefore require this check to be disabled:

```powershell
Set-DistributionGroup -Identity "Projektteam" -RequireSenderAuthenticationEnabled $false
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-Identity` | The distribution group (name, alias, or email address). |
| `-RequireSenderAuthenticationEnabled $false` | The group also accepts messages from unauthenticated, external senders. |

</details>

Without further restrictions, after this step any external sender can write to the list. Restricting it to individual senders follows in the next step, ideally during the same maintenance window.

## Option 1: Delivery restriction with mail contacts

The list of allowed senders (`AcceptMessagesOnlyFromSendersOrMembers`) accepts only objects from your own directory: mailboxes, mail users, mail contacts, and groups. A plain SMTP address cannot be entered there. You therefore create a mail contact for each external sender. To prevent the contact from appearing in the global address list, hide it:

```powershell
$kontakt = @{
    Name                 = "Extern Max Muster"
    ExternalEmailAddress = "max.muster@partner.example"
    OrganizationalUnit   = "example.local/Kontakte/Extern"
}
New-MailContact @kontakt
Set-MailContact -Identity "Extern Max Muster" -HiddenFromAddressListsEnabled $true
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Name` | Display name of the contact in Active Directory. A consistent prefix such as “External” makes later maintenance easier. |
| `ExternalEmailAddress` | The external address used by the sender. |
| `OrganizationalUnit` | Target OU in Active Directory. If not specified, the contact is placed in the default `Users` container. |
| `-HiddenFromAddressListsEnabled $true` | Hides the contact from the global address list and all address lists. |

</details>

Then add the contact as an allowed sender. The hashtable with `Add` adds to the existing list instead of overwriting it:

```powershell
$erlaubt = @{ Add = "Extern Max Muster", "Extern Anna Beispiel" }
Set-DistributionGroup -Identity "Projektteam" -AcceptMessagesOnlyFromSendersOrMembers $erlaubt
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-AcceptMessagesOnlyFromSendersOrMembers` | List of senders allowed to reach the group. Individual senders and groups are possible; for groups, all members can send. |
| `@{ Add = ... }` | Adds entries without removing existing ones. `@{ Remove = ... }` removes entries; a value without a hashtable replaces the entire list. |

</details>

As soon as this list contains at least one entry, Exchange rejects **all other senders**, including internal ones. If employees should continue to be able to write to the list, add an appropriate group as well, such as the distribution group itself (then all members can send) or a group containing all employees. Rejected senders receive a non-delivery report with status `5.7.1`.

Check the current configuration as follows:

```powershell
Get-DistributionGroup -Identity "Projektteam" |
    Format-List Name, RequireSenderAuthenticationEnabled,
        AcceptMessagesOnlyFrom, AcceptMessagesOnlyFromDLMembers
```

`AcceptMessagesOnlyFromSendersOrMembers` is a composite parameter: Exchange distributes the entries across the `AcceptMessagesOnlyFrom` (individual senders) and `AcceptMessagesOnlyFromDLMembers` (groups) properties. This is why the query displays those two properties. You can find the same setting in the Exchange Admin Center, in the group properties under Delivery management.

This option is suitable when only a few external senders need to be added and the list of allowed senders should be visible directly on the group object. The drawback is the maintenance effort: every new sender needs a contact in Active Directory.

## Option 2: Transport rule without contacts

If all internal users should still be able to write to the list while only external senders are restricted, a transport rule is simpler. It requires no contacts and can also be applied to entire domains. `RequireSenderAuthenticationEnabled` must also be set to `$false` here.

```powershell
$regel = @{
    Name                              = "Projektteam: nur freigegebene externe Absender"
    SentTo                            = "projektteam@example.com"
    FromScope                         = "NotInOrganization"
    ExceptIfFromAddressMatchesPatterns = '^max\.muster@partner\.example$',
                                         '@lieferant\.example$'
    RejectMessageReasonText           = "Sie sind nicht berechtigt, an diese Adresse zu senden."
    RejectMessageEnhancedStatusCode   = "5.7.1"
}
New-TransportRule @regel
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Name` | Name of the rule in the list of transport rules. |
| `SentTo` | Condition: The message is addressed to this group. |
| `FromScope NotInOrganization` | Condition: The sender is not part of your organization. Internal messages are therefore unaffected. |
| `ExceptIfFromAddressMatchesPatterns` | Exception: Sender addresses matching one of the regular expressions. `^…$` anchors an individual address, while `@domain$` allows an entire domain. Escape periods with `\.`. |
| `RejectMessageReasonText` | Text in the non-delivery report sent to the rejected sender. |
| `RejectMessageEnhancedStatusCode` | Enhanced status code in the non-delivery report; `5.7.1` indicates insufficient permission. |

</details>

Without anchoring with `^` and `$`, a pattern such as `max\.muster@partner\.example` would also match `max.muster@partner.example.net`. Add new senders with `Set-TransportRule` and the complete, expanded list of patterns; the parameter replaces the existing list.

## Limitation of both options: The sender address can be spoofed

Both options check only the sender address. Exchange does not determine whether the message actually came from the approved partner. Anyone who knows an approved sender's address can spoof it and reach the list. For a list with internal participants, this provides an entry point for phishing that also appears to come from a familiar address.

SPF, DKIM, and DMARC close this gap. If the partner's domain has a DMARC policy of `p=reject` or `p=quarantine` and your mail gateway enforces that policy, spoofed messages from that domain never reach the Exchange Server in the first place. Domains without a DMARC policy or with `p=none` do not have this protection.

If your gateway writes the check result to the `Authentication-Results` header, a second transport rule can additionally reject approved senders if DMARC did not pass:

```powershell
$regel = @{
    Name                            = "Projektteam: freigegebene Absender nur mit DMARC pass"
    SentTo                          = "projektteam@example.com"
    FromScope                       = "NotInOrganization"
    FromAddressMatchesPatterns      = '^max\.muster@partner\.example$',
                                      '@lieferant\.example$'
    ExceptIfHeaderMatchesMessageHeader = "Authentication-Results"
    ExceptIfHeaderMatchesPatterns   = 'dmarc=pass'
    RejectMessageReasonText         = "Absender konnte nicht verifiziert werden."
    RejectMessageEnhancedStatusCode = "5.7.1"
}
New-TransportRule @regel
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `FromAddressMatchesPatterns` | Condition: The rule applies only to approved senders, using the same patterns as the first rule. |
| `ExceptIfHeaderMatchesMessageHeader` | Name of the header checked by the exception. |
| `ExceptIfHeaderMatchesPatterns` | Exception: If the header contains `dmarc=pass`, the message is delivered. All other messages from approved senders are rejected. |

</details>

This rule works reliably only if the gateway removes or overwrites incoming `Authentication-Results` headers from the internet. Otherwise, a sender can supply the header with `dmarc=pass` themselves. Your gateway's documentation describes which headers it sets and whether it removes foreign headers; Exchange Server itself does not evaluate DMARC.

## Which option is right

The delivery restriction with contacts is suitable for lists with a fixed, small set of external senders and can also restrict internal senders if desired. The transport rule is suitable when everyone internally may send, entire partner domains should be allowed, or no contacts should be created in Active Directory. Regardless of the option, check the DMARC policy for every allowed domain and document who maintains the list of allowed senders.

## Sources

1.  [Microsoft Learn: Set-DistributionGroup](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-distributiongroup?view=exchange-ps): Parameters AcceptMessagesOnlyFromSendersOrMembers and RequireSenderAuthenticationEnabled, permitted object types.

2.  [Microsoft Learn: Recipient resolution in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/recipient-resolution): How the Categorizer evaluates delivery restrictions and sender authentication.

3.  [Microsoft Learn: New-MailContact](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-mailcontact?view=exchange-ps): Creating mail contacts for external addresses.

4.  [Microsoft Learn: New-TransportRule](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-transportrule?view=exchange-ps): Conditions, exceptions, and actions for transport rules, including FromScope and HeaderMatchesPatterns.

5.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489): Specification of the DMARC policies p=none, quarantine, and reject.

6.  [RFC 8601: Message Header Field for Indicating Message Authentication Status](https://datatracker.ietf.org/doc/html/rfc8601): Structure of the Authentication-Results header and the recommendation that receiving systems remove foreign instances.
