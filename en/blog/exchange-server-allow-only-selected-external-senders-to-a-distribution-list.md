---
title: "Exchange: Allow only selected external senders to a distribution list"
navTitle: "Distribution lists: external senders"
description: "An Exchange distribution list can be configured so that only specific external senders can reach it. Two approaches using PowerShell: delivery restrictions with mail contacts or a transport rule. Also covers the limits of sender verification and how DMARC closes them."
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
translationSourceHash: 14114b684164aa90e68aac9d9ca4f75ed59c1ce77cf6479a167f154e66788d45
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T15:42:08.397Z
translationReview: automatic
url: https://rafaelpfister.ch/en/blog/exchange-server-allow-only-selected-external-senders-to-a-distribution-list
---

# Exchange: Allow only selected external senders to a distribution list

By default, an Exchange distribution list accepts messages only from authenticated, meaning internal, senders. If a partner, vendor, or customer should also be able to reach the list without opening it to the entire internet, there are two options: a delivery restriction on the group itself or a transport rule. Both work in Exchange Server 2016, 2019, and the Subscription Edition.

## Starting point: RequireSenderAuthenticationEnabled

Whether a group accepts external messages at all is controlled by the `RequireSenderAuthenticationEnabled` property. If it is set to `$true` (the default for new distribution groups), Exchange rejects every message from an unauthenticated sender. Both of the following options therefore require this check to be disabled:

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

The list of allowed senders (`AcceptMessagesOnlyFromSendersOrMembers`) accepts only objects from your own directory: mailboxes, mail users, mail contacts, and groups. You cannot enter a plain SMTP address there. Therefore, create a mail contact for each external sender. To keep the contact from appearing in the global address book, hide it:

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
| `Name` | Display name of the contact in Active Directory. A consistent prefix such as “External” makes future maintenance easier. |
| `ExternalEmailAddress` | The external address from which the sender sends. |
| `OrganizationalUnit` | Target OU in Active Directory. Without specifying it, the contact is placed in the default `Users` container. |
| `-HiddenFromAddressListsEnabled $true` | Hides the contact from the global address book and all address lists. |

</details>

Then add the contact as an allowed sender. The hashtable with `Add` appends to the existing list instead of overwriting it:

```powershell
$erlaubt = @{ Add = "Extern Max Muster", "Extern Anna Beispiel" }
Set-DistributionGroup -Identity "Projektteam" -AcceptMessagesOnlyFromSendersOrMembers $erlaubt
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-AcceptMessagesOnlyFromSendersOrMembers` | List of senders allowed to reach the group. Individual senders and groups are possible; for groups, all members may send. |
| `@{ Add = ... }` | Adds entries without removing existing ones. `@{ Remove = ... }` removes entries; a value without a hashtable replaces the entire list. |

</details>

As soon as this list contains at least one entry, Exchange rejects **all other senders**, including internal ones. If employees should continue to be able to write to the list, also add a suitable group, for example the distribution group itself (then all members may send) or a group containing all employees. Rejected senders receive a non-delivery report with status `5.7.1`.

Check the current configuration as follows:

```powershell
Get-DistributionGroup -Identity "Projektteam" |
    Format-List Name, RequireSenderAuthenticationEnabled,
        AcceptMessagesOnlyFrom, AcceptMessagesOnlyFromDLMembers
```

`AcceptMessagesOnlyFromSendersOrMembers` is a composite parameter: Exchange distributes the entries across the `AcceptMessagesOnlyFrom` (individual senders) and `AcceptMessagesOnlyFromDLMembers` (groups) properties. This is why the query shows those two properties. You can find the same setting in the Exchange Admin Center in the group properties under Delivery management.

This option is suitable when only a few external senders need to be added and the list of allowed senders should be visible directly on the group object. The disadvantage is the maintenance effort: each new sender requires a contact in Active Directory.

## Option 2: Transport rule without contacts

If all internal users should still be able to write to the list and only external senders need to be restricted, a transport rule is simpler. It requires no contacts and can also apply to entire domains. `RequireSenderAuthenticationEnabled` must also be set to `$false` here.

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
| `ExceptIfFromAddressMatchesPatterns` | Exception: Sender addresses that match one of the regular expressions. `^…$` anchors an individual address, while `@domain$` allows an entire domain. Escape periods with `\.`. |
| `RejectMessageReasonText` | Text in the non-delivery report sent to the rejected sender. |
| `RejectMessageEnhancedStatusCode` | Enhanced status code in the non-delivery report; `5.7.1` indicates missing permission. |

</details>

Without anchoring with `^` and `$`, a pattern such as `max\.muster@partner\.example` would also match `max.muster@partner.example.net`. Add new senders with `Set-TransportRule` and the complete expanded list of patterns; the parameter replaces the existing list.

## Limitation of both options: The sender address can be spoofed

Both options check only the sender address. Exchange does not determine whether the message actually came from the approved partner. Anyone who knows an approved sender’s address can spoof it and thus reach the list. For a list with internal participants, this is an entry point for phishing that also appears to come from a familiar address.

SPF, DKIM, and DMARC close this gap. If the partner’s domain has a DMARC policy with `p=reject` or `p=quarantine` and your mail gateway enforces that policy, spoofed messages from that domain never reach Exchange in the first place. This protection is absent for domains without a DMARC policy or with `p=none`.

If your gateway writes the check result to the `Authentication-Results` header, a second transport rule can additionally reject approved senders when DMARC has not passed:

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
| `FromAddressMatchesPatterns` | Condition: The rule applies only to approved senders, using the same patterns as in the first rule. |
| `ExceptIfHeaderMatchesMessageHeader` | Name of the header checked by the exception. |
| `ExceptIfHeaderMatchesPatterns` | Exception: If the header contains `dmarc=pass`, the message is delivered. All other messages from approved senders are rejected. |

</details>

This rule works reliably only if the gateway removes or overwrites incoming `Authentication-Results` headers from the internet. Otherwise, a sender can supply the header with `dmarc=pass` themselves. Your gateway’s documentation describes which headers it sets and whether it removes foreign headers; Exchange itself does not evaluate DMARC.

## Which option fits

The delivery restriction with contacts is suitable for lists with a fixed, small group of external senders and can also restrict internal senders if desired. The transport rule is suitable when all internal users may send, entire partner domains should be allowed, or no contacts should be created in Active Directory. Regardless of the option, check the DMARC policy for every approved domain and document who maintains the list of allowed senders.

## Sources

1.  [Microsoft Learn: Set-DistributionGroup](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-distributiongroup?view=exchange-ps): Parameters AcceptMessagesOnlyFromSendersOrMembers and RequireSenderAuthenticationEnabled, permitted object types.

2.  [Microsoft Learn: Recipient resolution in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/recipient-resolution): How the Categorizer evaluates delivery restrictions and sender authentication.

3.  [Microsoft Learn: New-MailContact](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-mailcontact?view=exchange-ps): Creating mail contacts for external addresses.

4.  [Microsoft Learn: New-TransportRule](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-transportrule?view=exchange-ps): Conditions, exceptions, and actions for transport rules, including FromScope and HeaderMatchesPatterns.

5.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489): Specification of the DMARC policies p=none, quarantine, and reject.

6.  [RFC 8601: Message Header Field for Indicating Message Authentication Status](https://datatracker.ietf.org/doc/html/rfc8601): Structure of the Authentication-Results header and the recommendation that receiving systems remove foreign instances.
