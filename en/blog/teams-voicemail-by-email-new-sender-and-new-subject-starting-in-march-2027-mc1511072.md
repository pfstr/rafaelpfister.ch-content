---
title: "Teams voicemail by email: new sender and new subject starting in March 2027 (MC1511072)"
navTitle: "Voicemail sender"
description: "Microsoft is changing how Teams voicemails are delivered by email: the sender will now be “Microsoft Teams” from the organization’s own tenant domain, while caller details will move to the subject line. This article explains who is affected, how to review transport rules, allow lists, and automations, and what to adjust before March 2027."
date: "2026-11-02"
kategorie: "Microsoft Teams"
timeToRead: "4 min read"
themen:
  - microsoft-teams
  - microsoft-365-exchange
produkte:
  - "teams"
  - "exchange-online"
protokolle:
  - "smtp"
  - "mail-auth"
  - "powershell"
hauptthema: "microsoft-teams"
slug: "teams-voicemail-by-email-new-sender-and-new-subject-starting-in-march-2027-mc1511072"
translationId: "article-b75c975f086dd0d5"
translationOf: teams-voicemail-mail-absender
url: https://rafaelpfister.ch/en/blog/teams-voicemail-by-email-new-sender-and-new-subject-starting-in-march-2027-mc1511072
translationSourceHash: 32089440ce20d39865ec421317bed6588c48eace59eb8f114015c626da412eba
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:36:23.789Z
translationReview: automatic
---

With Message Center notification MC1511072 dated October 16, 2026, Microsoft announced a new delivery method for Teams voicemail emails. According to the current plan, rollout will begin in early March 2027 and is expected to be completed by the end of June 2027; Microsoft reserves the right to make changes and intends to provide binding dates and any required customer actions in a further notification. The purpose is to support custom sender domains (Bring Your Own Domain), so voicemail emails are covered by the organization’s SPF, DKIM, and DMARC policies.

## What is changing

The “From” field will no longer contain the caller’s name, phone number, or email address. This information will instead appear in the subject line and continue to appear in the message body. The display in Teams itself will remain unchanged.

| Attribute | Today | After rollout |
|---|---|---|
| Sender (personal voicemail) | Caller’s name or number | `"Microsoft Teams" no-reply@<Standard-Tenant-Domain>` or custom domain, if configured |
| Sender domain | `skype.voicemail.microsoft.com` for PSTN calls, tenant domain for Teams-to-Teams calls | Default tenant domain or custom domain |
| Subject (personal voicemail) | `Voice mail (22 seconds)` | `Identified: John Smith (+19371817273) - Voicemail (00:22)` or `Unidentified: +19371817273 - Voicemail (00:22)` |
| Call queue (shared voicemail) | `Shared Voicemail (<CQ-Name>)` | `John Smith (+37251223123) - Shared voicemail in <CQ-Name> (00:05)` |
| Auto attendant (shared voicemail) | `Shared Voicemail (<AA-Name>)` | `John Smith - Shared voicemail in <AA-Name> (00:11)` |

For users, this has two visible consequences: all voicemails will appear in the inbox with the same sender, “Microsoft Teams,” and because the address is a no-reply address, the “Reply” feature will no longer reach the caller.

## Who is affected

All organizations using Teams Calling with voicemail are affected, including Teams Phone with Calling Plan, Operator Connect, or Direct Routing, as well as call queues and auto attendants with shared voicemail. The notification does not mention any licensing or cloud restrictions; the rollout has been announced worldwide.

Action is required wherever something relies on the current sender or subject:

- Exchange transport rules that check for `skype.voicemail.microsoft.com`, the caller as the sender, or the subjects `Voice mail` and `Shared Voicemail`
- Entries in allow lists: Tenant Allow/Block List, spoof allow entries, `AllowedSenderDomains` in anti-spam policies
- Inbox rules for users and call queue shared mailboxes that sort or forward voicemails based on sender or subject
- Power Automate flows, ticketing systems, and CRM integrations that read the caller’s phone number from the sender
- Upstream or downstream gateways (for example, with centralized mail flow in hybrid environments) that contain rules or exceptions for the voicemail domain

## How to check whether you are affected

The following queries run in an Exchange Online PowerShell session (`Connect-ExchangeOnline`).

### Search transport rules

```powershell
$muster = 'skype\.voicemail|voice ?mail|shared voicemail'
Get-TransportRule |
  Where-Object { ($_ | Format-List * | Out-String) -match $muster } |
  Select-Object Name, State, Priority
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `$muster` | Regular expression for the current sender domain and current subject texts |
| `Format-List *` | Outputs all rule properties so that conditions and exceptions are included in the search |
| `Out-String` | Converts the output to text to which `-match` is applied |
| `Select-Object Name, State, Priority` | Displays only the name, status, and priority of matching rules |

</details>

Review every matching rule for conditions such as `FromAddressContainsWords`, `SenderDomainIs` or `SubjectContainsWords`.

### Allow lists and anti-spam policies

```powershell
Get-TenantAllowBlockListItems -ListType Sender |
  Where-Object Value -match 'voicemail'
Get-TenantAllowBlockListSpoofItems |
  Where-Object SpoofedUser -match 'voicemail'
Get-HostedContentFilterPolicy |
  Select-Object Name, AllowedSenderDomains, AllowedSenders
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-ListType Sender` | Reads sender entries from the Tenant Allow/Block List |
| `Where-Object Value -match 'voicemail'` | Filters entries whose value contains the voicemail domain |
| `Get-TenantAllowBlockListSpoofItems` | Lists spoof allow and block entries |
| `AllowedSenderDomains, AllowedSenders` | Allow entries directly in anti-spam policies |

</details>

### Determine actual mail volume

Message tracing shows whether and where voicemails are currently being delivered:

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddDays(-10) -EndDate (Get-Date) -ResultSize 5000 |
  Where-Object { $_.SenderAddress -like '*@skype.voicemail.microsoft.com' -or $_.Subject -match 'voice ?mail' } |
  Group-Object RecipientAddress | Sort-Object Count -Descending |
  Select-Object Count, Name
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-StartDate` / `-EndDate` | Search period; Exchange Online returns a maximum of the last 90 days and no more than 10 days per query |
| `-ResultSize 5000` | Maximum number of results per query |
| `Where-Object …` | Keeps emails from the PSTN voicemail domain or with a voicemail subject |
| `Group-Object RecipientAddress` | Counts voicemails per recipient |

</details>

The recipient list shows which mailboxes and shared mailboxes receive voicemails. There, it is worth reviewing inbox rules (`Get-InboxRule -Mailbox <Adresse>`) and connected flows.

## What to do

1.  Change rules to use the subject instead of the sender: Base new conditions on the future subject patterns `- Voicemail (` and `- Shared voicemail in `. Until rollout reaches your tenant, keep the old and new conditions in parallel.

2.  Remove allow entries for `skype.voicemail.microsoft.com` after rollout. Emails from your own tenant domain do not need an exception; allowing your own domain would be a security risk because it makes spoofing easier.

3.  Adjust automations that read the caller’s number from the sender. The number will instead appear in parentheses in the subject or, for unknown callers, immediately after `Unidentified:`.

4.  Prepare your custom sender domain: Microsoft plans to provide configuration instructions for sending from a custom domain in the follow-up notification. If DMARC for this domain is set to `p=reject`, SPF and DKIM must cover sending from Exchange Online; otherwise, voicemails will end up in quarantine.

5.  Inform users and the service desk: sender “Microsoft Teams,” caller details in the subject, and no more ability to reply to the caller by email.

## Sources

1.  [MC1511072: Microsoft Teams: Upcoming change to voicemail message delivery by email](https://mc.merill.net/message/MC1511072): archived text of the Message Center notification with the timeline and a comparison of the current and future sender and subject.

2.  [Microsoft Learn: Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2): parameters and limits for message tracing in Exchange Online.

3.  [Microsoft Learn: Get-TenantAllowBlockListItems](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-tenantallowblocklistitems): retrieving the Tenant Allow/Block List.

4.  [Microsoft Learn: Set up voicemail for Teams](https://learn.microsoft.com/en-us/microsoftteams/set-up-phone-system-voicemail): Teams voicemail fundamentals and delivery by email.
