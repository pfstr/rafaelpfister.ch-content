---
title: "Store and retrieve shared mailbox owners in Active Directory"
navTitle: "Shared mailbox owners"
description: "When a message from a shared mailbox is blocked, DLP solutions need to know whom to notify. Which AD attributes are suitable, which merely represent access permissions, why Exchange Online permissions do not reach on-premises AD, and how a vendor can retrieve the mapping automatically through LDAP or Microsoft Graph."
date: "2026-11-16"
kategorie: "Active Directory / Entra"
timeToRead: "8 min read"
themen:
  - active-directory-entra
  - microsoft-365-exchange
produkte:
  - "entra"
  - "exchange-online"
  - "exchange"
protokolle:
  - "ldap"
  - "powershell"
slug: "store-and-retrieve-shared-mailbox-owners-in-active-directory"
translationId: "article-ed41bd7fefc08361"
translationOf: shared-mailbox-verantwortliche-ad
url: https://rafaelpfister.ch/en/blog/store-and-retrieve-shared-mailbox-owners-in-active-directory
translationSourceHash: 54b0d22f6d1fd74974494b96e5566af25ea846062f8f9ce6a255607e40f937e6
translationModel: gpt-5.6-terra
translatedAt: 2026-11-16T08:51:20.776Z
translationReview: required
---

# Store and retrieve shared mailbox owners in Active Directory

The following request came across my desk during a data loss prevention project: Whenever a message from a shared mailbox is blocked, the people responsible should be notified so they can respond. The vendor of the DLP solution should retrieve this mapping automatically from Active Directory. The question is: Where in AD does it say who is responsible for a shared mailbox?

In short: AD knows access permissions, not responsibility. Some attributes show who can access a mailbox or send from it, and even those are incomplete once the mailbox is in Exchange Online. For reliable automation, the responsible person must be explicitly maintained in an attribute. The standard `manager` attribute is suitable for this, is synchronized by Entra Connect, and can be queried through LDAP and Microsoft Graph.

## Access is not responsibility

In AD, a shared mailbox is a disabled user object with Exchange attributes. The people who use it are defined by permissions: Full Access, Send As, and Send on Behalf. These lists often contain ten or twenty people, as well as delegates, trainees, and service accounts. A DLP alert sent to everyone with permissions reaches many people, none of whom feels responsible.

The question of responsibility is organizational: One person (or a small group) decides whether a blocked message was legitimate and maintains the permissions. This information exists in AD only if someone enters it.

## Which AD attributes are candidates

| Attribute | Content | Suitability for responsible persons |
|---|---|---|
| `manager` | DN of one person, single-valued, part of the person classes and therefore available on the mailbox user object | Suitable. Synchronized by Entra Connect to Entra ID and readable through Graph |
| `managedBy` | DN of one manager, single-valued | Not available on the user object. According to the schema, defined for groups, contacts, computers, and OUs, among others, but not for the User class |
| `msExchDelegateListLink` | DNs of people who have Full Access with automapping, multi-valued; back link `msExchDelegateListLinkBL` on the user | Access only, and incomplete: Full Access without automapping, permissions through groups, and permissions assigned in Exchange Online are missing |
| `publicDelegates` | DNs of people with Send on Behalf permissions, multi-valued | Access only. Written back by Entra Connect in hybrid environments |
| Permission list (`nTSecurityDescriptor`) | Extended “Send As” right on the user object | Only Send As assigned locally. Permissions assigned in Exchange Online are not stored there |
| `msExchMailboxSecurityDescriptor` | Mailbox permissions for on-premises mailboxes, binary | Access only, only for mailboxes on on-premises Exchange servers, difficult for vendors to evaluate |
| `extensionAttribute1` through `extensionAttribute15` | Free-text fields | Possible, but text only; no reference to an object and no automatic cleanup when a person is renamed or leaves |

In practice, the claim that the mapping is stored “directly in AD” usually refers to `msExchDelegateListLink`: ADUC or ADSI Edit does indeed show people there. However, the attribute is created only as a side effect of Outlook automapping and is not suitable as a data source for responsibilities.

## Where the mailbox is located determines the data source

If mailboxes are in Exchange Online and users are synchronized from on-premises AD (hybrid), the following applies: Permissions assigned in Exchange Online remain in Exchange Online. For Exchange Hybrid, Entra Connect writes back only a small list of attributes (including `publicDelegates` and `proxyAddresses`), but neither `msExchDelegateListLink` nor Send As permissions. In this case, on-premises AD shows the state before migration plus anything maintained locally since then.

For Exchange Online, only Exchange Online itself provides the actual state of permissions (`Get-EXOMailboxPermission`, `Get-EXORecipientPermission`). The vendor would need access to Exchange Online PowerShell, which is disproportionate for a simple mapping. A maintained attribute on the user object is simpler and works regardless of where the mailbox is located.

## Recommendation: `manager` as the responsibility attribute

For every shared mailbox, enter the responsible person in the `manager` attribute. Reasons:

- The attribute is part of the standard schema, so no schema extension is needed.
- It contains a reference (DN) rather than text. If the person is renamed or moved, the reference remains valid.
- Entra Connect synchronizes `manager` to Entra ID. The vendor can read it either through LDAP in on-premises AD or through Microsoft Graph.
- Through the `directReports` back link, every responsible person (and every report) can see which mailboxes they are responsible for.

The limitation is that `manager` is single-valued. Where a backup is essential, there are two options. Option 1: A group per mailbox whose members are the responsible people; its name follows a convention (for example, `SM-Verantwortlich-buchhaltung`) or is stored in an `extensionAttribute`. Option 2: The backup is derived by the vendor not from AD, but from the `manager` of the responsible person (escalation to their manager). Option 2 requires no additional objects and is sufficient in most organizations.

The attribute is set only in the authoritative system: in on-premises AD for synchronized objects, and in Entra ID for cloud-only objects.

## Assessing the current state: Who has access today?

For the initial population, a list of the people with current permissions is helpful. The responsible person is usually among them; the business department confirms the selection. For mailboxes in Exchange Online:

```powershell
Connect-ExchangeOnline

$postfaecher = Get-EXOMailbox -RecipientTypeDetails SharedMailbox -ResultSize Unlimited

$bestand = foreach ($pf in $postfaecher) {
    $voll = Get-EXOMailboxPermission -Identity $pf.UserPrincipalName |
        Where-Object {
            $_.AccessRights -contains 'FullAccess' -and
            -not $_.IsInherited -and
            $_.User -notlike 'NT AUTHORITY\*'
        }
    $sendas = Get-EXORecipientPermission -Identity $pf.UserPrincipalName |
        Where-Object {
            $_.AccessRights -contains 'SendAs' -and
            $_.Trustee -notlike 'NT AUTHORITY\*'
        }
    [pscustomobject]@{
        Postfach  = $pf.PrimarySmtpAddress
        Vollzugriff = ($voll.User -join '; ')
        SendenAls = ($sendas.Trustee -join '; ')
    }
}

$bestand | Export-Csv -Path .\shared-mailbox-bestand.csv -NoTypeInformation -Encoding UTF8
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-RecipientTypeDetails SharedMailbox` | Shared mailboxes only; no user, room, or equipment mailboxes |
| `-ResultSize Unlimited` | All mailboxes instead of the default limit of 1,000 |
| `-Identity` | Mailbox whose permissions are queried |
| `AccessRights -contains 'FullAccess'` | Filters for Full Access; other mailbox permissions are excluded |
| `-not $_.IsInherited` | Removes inherited entries (administrators, system accounts) |
| `User -notlike 'NT AUTHORITY\*'` | Removes the mailbox’s own `NT AUTHORITY\SELF` entry |
| `-NoTypeInformation` | No type line at the beginning of the CSV file |
| `-Encoding UTF8` | Preserves umlauts in names |

</details>

If a group has permissions, its name appears in the list; members can be resolved with `Get-DistributionGroupMember` or `Get-ADGroupMember`.

## Setting responsible persons

Prepare the confirmed mapping as a CSV with the columns `Postfach` and `Verantwortlich` (email addresses in both cases), then set it in on-premises AD:

```powershell
Import-Module ActiveDirectory

$zuordnung = Import-Csv -Path .\shared-mailbox-verantwortliche.csv

foreach ($eintrag in $zuordnung) {
    $postfach = Get-ADUser -Filter "mail -eq '$($eintrag.Postfach)'"
    $person = Get-ADUser -Filter "mail -eq '$($eintrag.Verantwortlich)'"
    Set-ADUser -Identity $postfach -Manager $person
}
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-Filter "mail -eq '…'"` | Finds the object using its primary email address |
| `-Identity` | User object of the shared mailbox |
| `-Manager` | Writes the DN of the responsible person to the `manager` attribute |

</details>

For cloud-only objects without on-premises AD, `Set-MgUserManagerByRef` from the Microsoft Graph PowerShell SDK sets the same value in Entra ID.

## Retrieval by the vendor: LDAP

Shared mailboxes can be identified in AD through `msExchRecipientTypeDetails`: `4` represents a shared mailbox on an on-premises Exchange server, while `34359738368` represents a shared mailbox in Exchange Online (RemoteSharedMailbox). The query the vendor can incorporate into its solution is:

```text
Basis:     DC=example,DC=com (Unterbaum)
Filter:    (&(objectClass=user)(|(msExchRecipientTypeDetails=4)(msExchRecipientTypeDetails=34359738368)))
Attribute: mail, proxyAddresses, manager
```

The value in `manager` is a DN. The responsible person’s email address is obtained by a second query of that DN for the `mail` attribute. A read account without further permissions is sufficient because `manager` and `mail` are readable by authenticated users. The connection should use LDAPS (port 636).

To verify the mapping in PowerShell:

```powershell
$filter = '(|(msExchRecipientTypeDetails=4)(msExchRecipientTypeDetails=34359738368))'

$verantwortlich = @{
    Name       = 'Verantwortlich'
    Expression = {
        if ($_.manager) {
            (Get-ADUser -Identity $_.manager -Properties mail).mail
        }
    }
}

Get-ADUser -LDAPFilter $filter -Properties mail, manager |
    Select-Object mail, $verantwortlich |
    Sort-Object mail
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-LDAPFilter` | LDAP filter for the two recipient types of shared mailboxes |
| `-Properties mail, manager` | Loads the two attributes that `Get-ADUser` does not return by default |
| `Expression = { … }` | Resolves the DN in `manager` to the responsible person’s email address |

</details>

For message mapping, the vendor should consider all addresses in `proxyAddresses`, not just `mail`: A message may be sent using a secondary mailbox address.

## Retrieval by the vendor: Microsoft Graph

If the vendor has no LDAP access to the internal network, Microsoft Graph is a good option. Shared mailboxes exist in Entra ID as disabled user objects, and `manager` is available there through synchronization:

```http
GET https://graph.microsoft.com/v1.0/users/buchhaltung@example.com/manager?$select=displayName,mail
```

The vendor’s app registration requires the `User.Read.All` application permission. Graph does not provide a direct list of all shared mailboxes because Entra ID does not know the mailbox type. The vendor therefore queries the sender address of the blocked message specifically, which is sufficient for the DLP use case anyway.

## Integration into the DLP notification

How the notification is technically triggered depends on the DLP solution. With Microsoft Purview DLP, note that notifications can additionally be sent to fixed individuals, but only to individual recipients, not to groups or distribution lists. A dynamic “mailbox-to-responsible-person” mapping is generally not provided. For a small number of mailboxes, one rule per mailbox is conceivable; for many mailboxes, a downstream process is needed that retrieves incidents, resolves the sender address through LDAP or Graph, and notifies the responsible person.

The same logic applies to an external solution: Determine the sender address of the blocked message, find the mailbox through `mail` or `proxyAddresses`, resolve `manager`, and notify. If `manager` is missing, the solution should report to a defined shared address (service desk or data protection) rather than discard the alert.

## Ongoing maintenance

In practice, two rules are enough to keep the mapping current:

- New shared mailboxes are created only with a responsible person entered; the request includes the field.
- A monthly report lists mailboxes without `manager` and mailboxes whose responsible person is disabled.

```powershell
$filter = '(|(msExchRecipientTypeDetails=4)(msExchRecipientTypeDetails=34359738368))'

Get-ADUser -LDAPFilter $filter -Properties mail, manager |
    Where-Object {
        -not $_.manager -or
        -not (Get-ADUser -Identity $_.manager).Enabled
    } |
    Select-Object mail, manager
```

When a person leaves, `directReports` on their object shows which mailboxes need a new responsible person. This check can be included in the offboarding process.

## Sources

1.  [Manager attribute (Active Directory Schema)](https://learn.microsoft.com/en-us/windows/win32/adschema/a-manager): single-valued DN attribute of the person classes, back link `directReports`.

2.  [Managed-By attribute (Active Directory Schema)](https://learn.microsoft.com/en-us/windows/win32/adschema/a-managedby): list of classes for which `managedBy` is defined; the User class is not included.

3.  [Microsoft Entra Connect Sync: Attributes synchronized to Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-sync-attributes-synchronized): synchronized attributes including `manager`, as well as the list of Exchange Hybrid writeback attributes.

4.  [Get-EXOMailboxPermission](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-exomailboxpermission): query mailbox permissions in Exchange Online.

5.  [Get-EXORecipientPermission](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-exorecipientpermission): query Send As permissions in Exchange Online.

6.  [Adding or removing mailbox permission in EAC doesn't address the msExchDelegateListLink attribute](https://support.microsoft.com/en-us/help/4502159): Microsoft support article on the relationship between Full Access and `msExchDelegateListLink` in Exchange 2016 and 2019.

7.  [List manager (Microsoft Graph)](https://learn.microsoft.com/en-us/graph/api/user-list-manager): Graph endpoint and required permissions for retrieving the manager.

8.  [Send email notifications and show policy tips for DLP policies](https://learn.microsoft.com/en-us/purview/dlp-use-notifications-and-policy-tips): recipients of DLP notifications in Microsoft Purview; individual recipients only, no groups.
