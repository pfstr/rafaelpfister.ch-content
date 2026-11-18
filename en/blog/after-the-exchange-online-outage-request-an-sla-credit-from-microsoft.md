---
title: "After the Exchange Online outage: Request an SLA credit from Microsoft"
navTitle: "SLA credit"
description: "The Exchange Online outage of November 12 and 13, 2026, lasted around 22 hours, putting many tenants below the guaranteed 99.9 percent availability. Microsoft does not automatically issue the credit; it must be requested by December 31, 2026. Learn how to calculate availability, which evidence Microsoft requires, and why small companies should proceed through their partner while large companies should contact Microsoft directly."
date: "2026-11-18"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min read"
themen:
  - microsoft-365-exchange
produkte:
  - "exchange-online"
protokolle:
  - "lizenzierung"
  - "powershell"
slug: "after-the-exchange-online-outage-request-an-sla-credit-from-microsoft"
translationId: "article-0e96d35615b0d8d9"
translationOf: microsoft-sla-gutschrift-einfordern
translationSourceHash: e2a540fa8b0a15956e2a9f4af6b604f517d4f077ec2d566bd2dd187a1d423216
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T11:20:19.074Z
translationReview: required
url: https://rafaelpfister.ch/en/blog/after-the-exchange-online-outage-request-an-sla-credit-from-microsoft
---

# After the Exchange Online outage: Request an SLA credit from Microsoft

<div class="notfall-hinweis notfall-hinweis--gross">
<p class="notfall-hinweis__titel">Free assistance with your SLA request</p>
<p>We will help you request the credit for the Exchange Online outage of November 12 and 13, 2026, from Microsoft or your partner free of charge: calculate availability, compile evidence, and submit the request. The deadline is December 31, 2026. Contact us using the <a href="https://rafaelpfister.ch/kontakt">contact form</a>.</p>
</div>

From November 12, 2026, at 6:54 PM until November 13, 2026, at 5:10 PM (Swiss time), many organizations had no or only intermittent access to their Exchange Online mailboxes. Microsoft listed the disruption in the Admin Center under EX1550543 (Exchange Online) and MO1550821 (other Microsoft 365 services). I documented the timeline in the [live ticker covering the outage](https://rafaelpfister.ch/liveticker/exchange-online-ausfall-2026-11). For Exchange Online, Microsoft guarantees monthly availability of 99.9 percent. An outage lasting more than 22 hours is well below that. However, you receive a credit only if you request one, and the process depends on how your organization purchased Microsoft 365.

## What the SLA guarantees

The relevant document is the “Service Level Agreement for Microsoft Online Services,” which Microsoft republishes every month. This article is based on the version dated November 17, 2026. For Exchange Online, the SLA defines downtime as any period during which users cannot send or receive email using Outlook on the web. There are no scheduled maintenance windows for Exchange Online.

Availability is calculated in user minutes:

```text
Verfügbarkeit (%) = (Benutzerminuten - Ausfallminuten) / Benutzerminuten x 100
```

User minutes are the number of minutes in the month multiplied by the number of users. Downtime minutes are the duration of each incident multiplied by the number of affected users. If availability falls below a threshold, you receive a credit against the monthly fee for the affected service:

| Monthly availability | Credit |
|---|---|
| below 99.9% | 25% |
| below 99% | 50% |
| below 95% | 100% |

The same thresholds apply to Microsoft Teams, SharePoint Online, and OneDrive, with their own definitions of downtime. For Teams, it counts only if users cannot chat or start meetings. During EX1550543, chats and meetings largely worked according to Microsoft; calendars and invitations were affected. A Teams request is therefore unlikely to succeed.

In addition to mailbox availability, Exchange Online has other service levels, such as delivery time within Microsoft 365 and acceptance of incoming email. If an incident affects more than one of them, the SLA says you must choose one. For the November outage, mailbox availability through Outlook on the web is the obvious metric.

## What the outage means in percentage terms

November 2026 has 30 days, or 43,200 minutes per user. This results in the following thresholds per user:

| Threshold | Permitted downtime in November |
|---|---|
| 99.9% | 43 minutes |
| 99% | 7 hours 12 minutes |
| 95% | 36 hours |

The disruption lasted 1,336 minutes from the first report until it was resolved. Two example calculations for an organization with 200 mailboxes:

| Scenario | Downtime minutes | Availability | Credit |
|---|---|---|---|
| All 200 users without access for the entire disruption | 200 x 1,336 = 267,200 | 96.91% | 50% |
| 120 users without access for 10 hours | 120 x 600 = 72,000 | 99.17% | 25% |

The total number of user minutes is 200 x 43,200 = 8,640,000 in both cases. What matters is how many users were actually affected and for how long. Microsoft verifies this based on its own telemetry, but requires you to provide this information in the request. Many tenants in Europe were affected only from late in the evening on November 12 or only intermittently. Therefore, calculate using the times you can document, not the total duration stated in Microsoft’s notice.

## What the credit is and is not

The SLA sets out several points that should be clear before submitting a request:

1.  **Only upon request.** Microsoft does not review or grant credits on its own initiative. According to the SLA, a Microsoft status notice explicitly does not constitute confirmation that a service level was missed.

2.  **Credit, not refund.** The credit is applied against a future invoice. It is limited to the monthly fee for the affected service.

3.  **Sole remedy.** According to the SLA, credits are the sole remedy for availability issues. Microsoft does not reimburse lost revenue, overtime, or other consequential costs. You may not reduce invoices on your own.

4.  **Only the affected service.** If you have a suite such as Microsoft 365 E3 or Business Premium, the credit applies to the Exchange Online portion, not the entire license price. Microsoft determines the amount of that portion during its review; the SLA provides no formula for it.

5.  **No credit for free and trial offers.** Trial versions, previews, and free services are excluded, as are donated licenses for nonprofit organizations. Individual customers with Microsoft 365 Personal or Family are not covered by this SLA.

6.  **Open agreements in days.** Those licensed through Open, Open Value, or Open Value Subscription receive any credit as additional usage days rather than a monetary amount.

7.  **Deadline.** For Microsoft 365 services, the request with all information must be received by the end of the month following the incident month. For the November outage, this is **December 31, 2026**. Azure has its own deadline of 60 days.

## What information Microsoft requires

The SLA lists five required details. If any are missing, Microsoft rejects the request:

1.  an exact description of the incident,
2.  the time and duration of the downtime,
3.  the affected resources (for Exchange Online: tenant and service),
4.  the number and locations of affected users,
5.  a description of the errors encountered.

The incident ID (EX1550543) and tenant ID should always be included. Both can be retrieved using Microsoft Graph PowerShell:

```powershell
Connect-MgGraph -Scopes "ServiceHealth.Read.All", "Organization.Read.All"
(Get-MgOrganization).Id
Get-MgServiceAnnouncementIssue -ServiceHealthIssueId "EX1550543" |
    Select-Object Id, Title, StartDateTime, EndDateTime, Classification
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-Scopes "ServiceHealth.Read.All"` | Permission to read service health notices |
| `-Scopes "Organization.Read.All"` | Permission to read organization data, including the tenant ID |
| `(Get-MgOrganization).Id` | outputs the tenant ID |
| `-ServiceHealthIssueId "EX1550543"` | retrieves this specific incident |
| `Select-Object …` | displays the incident ID, title, start, end, and classification |

</details>

The classification (`Classification`) indicates whether Microsoft lists the incident as an `incident` or an `advisory`. According to Microsoft’s partner documentation, notices classified as “Advisory” are generally not eligible for credits. EX1550543 was classified as an Incident.

For the number and location of users, an evaluation of licensed mailboxes by usage location is sufficient:

```powershell
Connect-MgGraph -Scopes "User.Read.All"
$abfrage = @{
    All              = $true
    Filter           = 'assignedLicenses/$count ne 0'
    ConsistencyLevel = 'eventual'
    CountVariable    = 'anzahl'
    Property         = 'UsageLocation'
}
Get-MgUser @abfrage |
    Group-Object UsageLocation |
    Select-Object Name, Count
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `$abfrage = @{ … }` | collects the parameters for `Get-MgUser` in a hashtable (splatting) to keep the line short |
| `All` | retrieves all users, not just the first page |
| `Filter 'assignedLicenses/$count ne 0'` | only users with at least one assigned license |
| `ConsistencyLevel eventual` | required for the `$count` filter (advanced query) |
| `CountVariable anzahl` | stores the total count in the variable `$anzahl` |
| `Property UsageLocation` | also retrieves the usage location |
| `Group-Object UsageLocation` | counts users by country |

</details>

This number is the upper limit. Document how many users actually had no access using your own data: help desk tickets with timestamps, error messages from Outlook and Outlook on the web (screenshots), availability test results from your monitoring, and sign-in and access logs. The more precise this information is, the less the outcome depends solely on Microsoft’s telemetry.

## Small companies: usually through the partner

Small and medium-sized companies in Switzerland usually purchase Microsoft 365 through a partner in the Cloud Solution Provider Program (CSP) or directly by credit card in the Admin Center. You can see which route applies in the Microsoft 365 Admin Center under **Billing > Your products**: if a partner is listed as the source, the agreement is a CSP agreement.

### Purchased through a CSP partner

In this case, Microsoft does not accept the request from you. According to Partner Center, only CSP direct-bill partners and indirect providers (distributors) can request credits, not indirect resellers. If your IT service provider is an indirect reseller, the request therefore goes through them to their distributor and then to Microsoft. Each level has its own, often earlier deadlines. Contact them now rather than waiting until the end of December.

Your partner needs from you:

1.  the tenant ID,
2.  the incident ID (EX1550543),
3.  the period, number, and locations of affected users, as well as the observed errors,
4.  an email requesting the SLA credit. It must come from an address in the affected domain; Microsoft does not accept email from a personal address.

Microsoft already knows about the incident. The request must document that your organization was affected: according to Microsoft, the name, tenant ID, and ticket number alone are not sufficient. If the credit is granted, Microsoft credits it to the partner, and the partner passes it on to you. Whether and how it does so is governed by your agreement with the partner. Ask if nothing appears on the next invoice.

### Purchased directly from Microsoft

If you paid Microsoft 365 directly to Microsoft, submit the request yourself in the Admin Center. This requires an administrator role:

1.  Open **Help & support** in the bottom-right corner at `admin.cloud.microsoft`.
2.  Describe the issue, for example, “Request an SLA service credit for a Microsoft 365 outage,” and select **Contact support**.
3.  Choose email as the contact method. The subject should include “SLA service credit request” and incident ID EX1550543.
4.  In the description, list the five required details from the previous section and attach the evidence.

A separate request is required for each service and incident. According to the SLA, Microsoft generally processes requests within 45 days.

### Is it worth it for small companies?

For ten users, the credit is small: with a suite, only the Exchange portion counts, of which 25 or 50 percent applies for one month. The effort is also small if you compile the information now. The request has a second benefit: it documents how severely your organization was affected for the partner and management. This helps later when considering whether an email fallback is necessary.

## Large companies: directly with Microsoft, using their own measurement data

Large organizations usually license Microsoft 365 through an Enterprise Agreement (EA) or the Microsoft Customer Agreement for Enterprise (MCA-E). Here, the request goes directly to Microsoft, either through the Admin Center as described above or through the Unified Support agreement. If you have a Customer Success Account Manager (CSAM), notify them of the request; they can track its processing. For an EA through a Licensing Solution Partner (LSP), also clarify which invoice will show the credit.

With several thousand users, significant amounts are involved, and Microsoft scrutinizes the information accordingly. The difference from a small tenant lies in the evidence:

1.  **Determine affected users precisely.** Large tenants are distributed across multiple databases and regions in Microsoft data centers; not all mailboxes were affected for the same length of time. Reporting all users as affected across the board is vulnerable to challenge for large tenants. Measurement data is more reliable: synthetic tests against Outlook on the web by location, availability data from your own monitoring, and analyses of help desk tickets by location and time.

2.  **Specify locations.** The SLA requires the locations of affected users. For international organizations, it is worth breaking these down by country because the disruption first affected North and South America and only later other regions.

3.  **Clearly separate hybrid issues.** Delays in your own data center, such as email queued on local Exchange servers for the tenant, are excluded by the SLA. Document that the error occurred within Microsoft 365. The queues on local servers are good evidence because they show that Exchange Online did not accept email, or accepted it only with delays.

4.  **Choose the service level.** If delivery times were primarily affected in your environment rather than mailbox access, the service level for delivery time may be a better basis. Only one request is permitted per incident and service, so this choice should be made before submitting the request.

5.  **Involve procurement and license management.** The credit appears on the invoice, not in the IT cost center. Coordinate with procurement on who submits the request, who verifies the credit, and how it is recorded.

Large organizations should also prepare the process for the next outage: a template containing the five required details, monitoring that measures Outlook on the web availability by location, and clear responsibility for SLA requests.

## Template for the request

The following template includes the details required by the SLA. Microsoft processes requests in English most quickly:

```text
Subject: SLA service credit request: EX1550543, Exchange Online

Tenant ID: <Tenant-ID>
Service: Exchange Online
Incident ID: EX1550543 (related: MO1550821)
Service level: Exchange Online availability (Outlook on the web)

Description of the incident:
Users were unable to send or receive email using Outlook on the web,
Outlook for Windows and mobile clients.

Time and duration of downtime:
From 2026-11-12 <hh:mm> UTC to 2026-11-13 <hh:mm> UTC (<n> minutes).

Affected users:
<n> of <total> licensed users, locations: <Land: Anzahl, ...>

Errors observed:
<Fehlermeldungen, z. B. Verbindungsfehler in Outlook, HTTP 5xx in Outlook on the web>

Uptime calculation (November 2026):
(<total> x 43,200 - <Ausfallminuten>) / (<total> x 43,200) x 100 = <xx.xx> %

Attachments: helpdesk ticket export, monitoring data, screenshots.
```

## Sources

1.  [Service Level Agreement for Microsoft Online Services (version overview)](https://www.microsoft.com/licensing/docs/view/Service-Level-Agreements-SLA-for-Online-Services): version dated November 17, 2026; definition of downtime for Exchange Online, credit tiers, required details, deadlines, and exclusions.

2.  [Microsoft Learn: How and when to request a credit from Microsoft](https://learn.microsoft.com/en-us/partner-center/billing/request-credit): request process for CSP partners, exclusion of indirect resellers, required evidence, and classification of Advisories.

3.  [Microsoft Learn: Get-MgServiceAnnouncementIssue](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.devices.serviceannouncement/get-mgserviceannouncementissue): cmdlet for retrieving a service health incident through Microsoft Graph.

4.  [Microsoft 365 Status (status.cloud.microsoft)](https://status.cloud.microsoft/): public status notices for EX1550543 and MO1550821.

5.  [Windows Latest: Microsoft 365 outage, you may get money off your bill](https://www.windowslatest.com/2026/11/13/microsoft-365-outage-you-may-get-money-off-your-bill-if-you-use-it-for-work-but-youll-have-to-ask-microsoft-for-it-as-a-server-failure-takes-down-outlook-teams-and-exchange/): request process in the Admin Center for the November outage and a note about individual customers.

6.  [Office Watch: Microsoft 365 went down? How to claim your SLA credit](https://office-watch.com/?p=78178): exclusion of individual, free, and nonprofit licenses; one request per service and incident.

7.  [Live ticker: Exchange Online, worldwide mailbox access outage](https://rafaelpfister.ch/liveticker/exchange-online-ausfall-2026-11): timeline of the disruption on November 12 and 13, 2026, with Microsoft chronology.
