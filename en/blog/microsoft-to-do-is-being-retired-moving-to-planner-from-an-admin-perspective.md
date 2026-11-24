---
title: "Microsoft To Do Is Being Retired: Moving to Planner from an Admin Perspective"
navTitle: "To Do Is Being Retired"
description: "Microsoft is retiring the To Do apps on June 30, 2027. Business accounts will move to Planner, existing tasks will not be migrated, and Planner will store new personal tasks outside the Exchange mailbox. For Swiss tenants, this changes data location and compliance coverage; affected users can be identified through Microsoft Graph."
date: "2026-11-24"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min read"
themen:
  - microsoft-365-exchange
produkte:
  - "planner"
  - "exchange-online"
protokolle:
  - "migration"
  - "powershell"
hauptthema: "microsoft-365-exchange"
slug: "microsoft-to-do-is-being-retired-moving-to-planner-from-an-admin-perspective"
translationId: "article-7691ba2aae63c942"
translationOf: microsoft-to-do-planner-umstieg
url: https://rafaelpfister.ch/en/blog/microsoft-to-do-is-being-retired-moving-to-planner-from-an-admin-perspective
translationSourceHash: ae78db5df03b2677ad2bda294efdd3b1f31ee233d2de5ea0238c2de865396fe8
translationModel: gpt-5.6-terra
translatedAt: 2026-11-24T12:57:39.678Z
translationReview: automatic
---

# Microsoft To Do Is Being Retired: Moving to Planner from an Admin Perspective

Microsoft is retiring Microsoft To Do. The apps for web, Windows, Mac, iOS, and Android will be discontinued on June 30, 2027, for business and school accounts as well as personal Microsoft accounts. For business and school accounts, Microsoft Planner will take over personal tasks; the redesigned “My tasks” view will roll out starting in December 2026. Microsoft published the change in a support article in November 2026, including a timeline, differences, and guidance for administrators.

In brief: There will be no automatic migration. Existing To Do tasks will remain visible in Planner until June 30, 2027, and will continue to reside in the Exchange Online mailbox. Starting with the rollout, Planner will store new personal tasks in a private Planner plan, no longer in Exchange. This changes data location and Purview coverage for Swiss tenants. Review licenses, shared lists, integrations, and retention requirements by mid-2027. Personal Microsoft accounts will not receive a replacement.

## Timeline

| Date | Change |
|---|---|
| December 2026 | Rollout of the redesigned “My tasks” view in Planner for business and school accounts begins. From then on, Planner will store newly created personal tasks in a private Planner plan. |
| before June 30, 2027 | Microsoft will provide an export feature for To Do data. The format, scope, and prerequisites have not yet been published. |
| June 30, 2027 | The To Do apps for web, Windows, Mac, iOS, and Android will be retired. Existing To Do tasks will no longer be visible in Planner afterward. |
| later, date not yet announced | The To Do service, APIs, and synchronization will be retired separately. Microsoft will announce the dates separately. |

Planner is already available to business accounts on the web at `planner.cloud.microsoft` and as an app in Teams. Microsoft has announced that it will update the existing To Do entry points in Microsoft 365. Microsoft clarified to Thurrott that existing integrations, including Outlook and Teams, will be switched to Planner.

## Existing Tasks Will Not Be Migrated

Microsoft will not transfer existing To Do tasks and lists to Planner. They can still be viewed and edited in Planner until June 30, 2027, but not afterward. They will not be transferred to the new private plan.

The data will not be deleted when the apps are retired. According to Microsoft, retiring the apps does not include data deletion; Microsoft addresses retention and access separately. To Do stores tasks as Exchange tasks in the user’s Exchange Online mailbox, the same mailbox that contains mail, appointments, contacts, and notes. The data will therefore remain in the mailbox for the time being after June 30, 2027.

Classic Outlook is not affected by the retirement. Microsoft states that its task feature is separate from To Do, will continue to work, and syncs with Exchange. To Do lists appear in classic Outlook as task folders under “My Tasks”; users with classic Outlook can therefore continue to read the existing data there. Microsoft has not yet described what will happen to the data when it later shuts down the To Do service itself.

## New Tasks Are Stored Outside Exchange

The new view changes the storage location. According to the FAQ for Planner administrators, tasks from To Do and Outlook are stored in Exchange, whereas plans and their tasks are stored in Azure. Planner stores attachments to plan tasks in the group’s SharePoint site, and Premium plans in Dataverse. New personal tasks from “My tasks” will therefore no longer reside in the user’s mailbox. This has two implications for administrators.

### Data Location

Exchange Online is one of the services with a long-term data location commitment. If a tenant is provisioned in Switzerland, Microsoft states in the Product Terms that it stores mailbox content (email body text, calendar entries, and attachments) at rest only in Switzerland. The commitment does not explicitly mention tasks, but they reside in the same mailbox.

Microsoft lists Planner among the other Microsoft 365 services without a long-term commitment. For tenants with Switzerland as their country, the table lists “EUR1,” the Europe macro-region, as the storage location. Planner also does not participate in Advanced Data Residency. Personal tasks that To Do previously stored in the mailbox in Switzerland will be stored by Planner in the Europe macro-region after the transition. The storage locations for each service in your tenant are listed on the Data Location Card in the Microsoft 365 admin center. Organizations that have contractually committed to a Switzerland data location or specified it in internal policies must assess this change.

### Retention and eDiscovery

Exchange retention policies in Microsoft Purview cover tasks in the mailbox only if they have an end date; tasks without an end date are not covered. This limitation already applies to To Do tasks today.

For Planner, Microsoft states that Purview eDiscovery search and Legal Hold support only tasks in plans shared with a group. Personal plans (roster plans), plans in Loop components, Premium plans, and plans in shared and private Teams channels are explicitly not supported. Microsoft has been rolling out retention and deletion policies for Planner content since mid-July 2026, with completion announced for early August (MC1384412); according to the announcement, they can be applied to user- and group-based plans. The announcement does not specify which plan type the new private plan technically corresponds to. After the rollout, verify whether your retention policies and holds cover the new personal tasks.

## Differences Between To Do and Planner

Microsoft lists the following differences:

| Area | Difference according to Microsoft |
|---|---|
| Organization | Planner combines personal tasks and assigned work under “My tasks,” currently with grid and board views. A list view will follow with the redesign. |
| Reminders | Due-date notifications in Planner differ from the reminders scheduled per task in To Do. |
| Sharing | Planner works with plans and assignments. Memberships, assignments, and permissions of shared To Do lists will not be transferred. |
| Desktop and offline | Planner is available on the web and in Teams. Native apps for Windows and Mac are not planned; web and Teams do not offer comparable offline functionality. |
| Categories | Planner labels work differently from Outlook categories and do not synchronize with them. |
| Attachments | File attachments to tasks are not currently supported in the new private Planner plan. |
| Star and priority | The personal star is used for individual planning and arrives with the new view. A task’s priority applies to everyone involved in a shared plan. |
| List groups | No direct equivalent in Planner. |
| Printing, emailing a list | Not currently planned for the replacement solution. |

Users who work offline with the To Do app on Windows or Mac have no equivalent replacement in Planner.

## Licenses and Access to Planner

Planner is included in Microsoft 365 Business Basic, Business Standard, and Business Premium, as well as Office 365 E1, E3, and E5, among others. If Planner is not included in a license, Microsoft offers Planner Plan 1 or Planner and Project Plan 3 as separate subscriptions; these are also required for Premium features. According to Microsoft, no new configuration is required, and there is no separate switch administrators can use to control or postpone the transition.

If Planner is disabled in the license assignment (service plan `PROJECTWORKMANAGEMENT`), affected users need access again if Planner is to replace To Do. According to the Planner FAQ, removing the Planner license only hides the tile anyway: users without a license can still create and edit plans through `planner.cloud.microsoft`. If you have previously blocked Planner through licensing, review that decision in light of the transition.

## Assessing the Environment with Microsoft Graph

You can use the Microsoft Graph To Do API to identify which users use To Do and which lists are shared. The following script writes all lists per user, including sharing status and the number of open tasks, to a CSV file. It requires the modules `Microsoft.Graph.Authentication` and `Microsoft.Graph.Users`, as well as an app registration with the application permissions `Tasks.Read.All` and `User.Read.All` (with admin consent) and a certificate for sign-in. `Tasks.Read.All` grants read access to the tasks of all users; remove the permission or the app registration again after the assessment.

```powershell
$verbindung = @{
    ClientId              = '<App-ID>'
    TenantId              = '<Tenant-ID>'
    CertificateThumbprint = '<Zertifikat-Fingerabdruck>'
}
Connect-MgGraph @verbindung -NoWelcome

$benutzer = Get-MgUser -All -Filter "accountEnabled eq true" -Property Id, UserPrincipalName

$bestand = foreach ($b in $benutzer) {
    try {
        $listen = Get-MgUserTodoList -UserId $b.Id -All -ErrorAction Stop
    }
    catch {
        # No Exchange Online mailbox or no access: skip user
        continue
    }
    foreach ($l in $listen) {
        $aufgaben = @(Get-MgUserTodoTask -UserId $b.Id -TodoTaskListId $l.Id -All)
        [pscustomobject]@{
            Benutzer = $b.UserPrincipalName
            Liste    = $l.DisplayName
            Typ      = $l.WellknownListName
            Geteilt  = $l.IsShared
            Besitzer = $l.IsOwner
            Offen    = @($aufgaben | Where-Object Status -ne 'completed').Count
            Gesamt   = $aufgaben.Count
        }
    }
}

$bestand | Export-Csv -Path .\todo-bestand.csv -NoTypeInformation -Encoding UTF8
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `ClientId`, `TenantId` | App registration and tenant that the script signs in to |
| `CertificateThumbprint` | Thumbprint of the certificate in the current user’s certificate store; app-only sign-in without a user account |
| `-NoWelcome` | Suppresses the welcome message from `Connect-MgGraph` |
| `-All` | Reads all pages of a result instead of only the first |
| `-Filter "accountEnabled eq true"` | Only enabled accounts; disabled accounts such as shared mailboxes are excluded |
| `-Property Id, UserPrincipalName` | Loads only the two required properties |
| `-UserId` | User whose lists or tasks are queried |
| `-ErrorAction Stop` | Makes an error catchable so that `catch` can skip the user |
| `-TodoTaskListId` | List whose tasks are queried |
| `WellknownListName` | `defaultList` for the default “Tasks” list, `flaggedEmails` for flagged messages, `none` for custom lists |
| `IsShared` | `True` if the list is shared with other users |
| `IsOwner` | `False` if the list belongs to another user and was only shared |
| `Status -ne 'completed'` | Counts all incomplete tasks |
| `-NoTypeInformation` | No type line at the beginning of the CSV file |
| `-Encoding UTF8` | Preserves umlauts in list names |

</details>

Accounts without an Exchange Online mailbox return an error and are skipped. In large tenants, a test run with a subset is advisable, for example using `-Top 20` instead of `-All` with `Get-MgUser`.

Reviewing the CSV file:

- **Shared lists** (`Geteilt` = `True`) show where collaboration takes place through To Do. Microsoft will not transfer these memberships; the lists must be rebuilt as a Planner plan with the relevant people.
- **Open tasks** in custom lists (`Typ` = `none` or `defaultList`) must be manually transferred to Planner by users by June 30, 2027, or secured using the announced export feature.
- **Flagged messages** (`Typ` = `flaggedEmails`) remain flagged on the email in the mailbox.

## Personal Microsoft Accounts

The To Do apps will also be retired on June 30, 2027, for personal Microsoft accounts, including Microsoft 365 Personal and Family. Planner is not available for these accounts. Microsoft recommends choosing another task management solution and securing the data with the announced export feature. According to Microsoft, an export does not mean that the data can be imported into Planner or another application. Users with both business and personal accounts must review both accounts separately.

## Preparation for Administrators

1. **Access and licenses:** Ensure that all To Do users have a license that includes Planner and that service plan `PROJECTWORKMANAGEMENT` is enabled.

2. **Shared lists and integrations:** Assess the environment using the script above. Identify Power Automate flows and third-party applications that use the Microsoft Graph To Do API. Microsoft will retire the API separately and later; according to Microsoft, integrations may nevertheless require adjustments.

3. **Retention and data location:** Review whether existing retention requirements apply to To Do tasks and whether Purview policies and holds cover the new private plan. Assess whether storage in the Europe macro-region is compatible with your requirements. Involve compliance stakeholders before deleting or reorganizing task data.

4. **Inform users:** Tell users that no migration will take place, how long existing data will remain visible in Planner, and which features will change (offline use, reminders, attachments, list groups).

5. **Monitor the Message Center:** Microsoft will announce rollout details, the export feature, and the dates for the service and APIs there.

## Sources

1.  [Moving from Microsoft To Do to Microsoft Planner](https://support.microsoft.com/en-us/planner/to-do-to-planner): Announcement with timeline, differences between To Do and Planner, preparation steps, and FAQ.

2.  [Review data storage and compliance in Microsoft To Do](https://support.microsoft.com/en-us/todo/review-data-storage-and-compliance-in-microsoft-to-do): Storage of To Do tasks as tasks in the Exchange Online mailbox.

3.  [Frequently asked questions for admins about Microsoft Planner](https://learn.microsoft.com/en-us/planner/faq-for-planner-admins): Storage locations of tasks (Exchange, Azure, SharePoint, Dataverse), service plan `PROJECTWORKMANAGEMENT`, and access through `planner.cloud.microsoft` without a license.

4.  [Microsoft Planner for admins](https://learn.microsoft.com/en-us/planner/planner-for-admins): Subscriptions that include Planner, as well as Planner Plan 1 and Planner and Project Plan 3.

5.  [Other Microsoft 365 services data locations](https://learn.microsoft.com/en-us/microsoft-365/enterprise/m365-dr-service-other?view=o365-worldwide#planner): Planner without a long-term data location commitment, storage location “EUR1” for tenants with Switzerland as their country.

6.  [Data Residency for Exchange Online](https://learn.microsoft.com/en-us/microsoft-365/enterprise/m365-dr-service-exo?view=o365-worldwide): Product Terms commitment for mailbox content at rest in Switzerland.

7.  [Security, privacy, and compliance in Microsoft Planner](https://learn.microsoft.com/en-us/planner/planner-security-privacy-compliance): Storage of plan data in Azure; Planner does not participate in Advanced Data Residency. Current as of August 2025, before the introduction of retention policies for Planner.

8.  [Microsoft Planner support in Microsoft Purview](https://learn.microsoft.com/en-us/planner/planner-in-purview): eDiscovery and Legal Hold only for tasks in group-based plans, with a list of unsupported plan types.

9.  [MC1384412: Retention policies for Microsoft Planner content](https://mc.merill.net/message/MC1384412): Mirror of the Message Center announcement on retention and deletion policies for Planner, rollout from July to August 2026.

10.  [Learn about retention for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange): Supported mailbox items; tasks only with an end date.

11.  [Use the Microsoft To Do API](https://learn.microsoft.com/en-us/graph/api/resources/todo-overview?view=graph-rest-1.0): Microsoft Graph To Do API resources (lists, tasks, checklists, linked resources).

12.  [List lists](https://learn.microsoft.com/en-us/graph/api/todo-list-lists?view=graph-rest-1.0): Endpoint, permissions, and properties `isShared`, `isOwner`, and `wellknownListName`.

13.  [List Todo tasks](https://learn.microsoft.com/en-us/graph/api/todotasklist-list-tasks?view=graph-rest-1.0): Endpoint for a list’s tasks and application permission `Tasks.Read.All`.

14.  [Microsoft Will Retire its To Do App in June 2027](https://www.thurrott.com/cloud/344071/microsoft-will-retire-its-to-do-app-on-in-june-2027): Report including a subsequent statement from Microsoft on integrations in Outlook and Teams.
