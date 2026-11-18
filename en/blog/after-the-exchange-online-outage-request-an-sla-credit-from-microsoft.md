---
title: "After the Exchange Online outage: Request an SLA credit from Microsoft"
navTitle: "SLA credit"
description: "The Exchange Online outage on November 12 and 13, 2026, lasted around 22 hours, putting many tenants below the guaranteed availability of 99.9 percent. Microsoft does not automatically issue the credit; it must be requested by December 31, 2026. Learn how to calculate availability, what evidence Microsoft requires, and why small companies should go through their partner while large companies should contact Microsoft directly."
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
url: https://rafaelpfister.ch/en/blog/after-the-exchange-online-outage-request-an-sla-credit-from-microsoft
translationSourceHash: 537caf78f05fd364041d0a3155f020be11b80f4c936d732e61f0a01f443853c1
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T09:20:30.781Z
translationReview: required
---

# After the Exchange Online outage: Request an SLA credit from Microsoft

From November 12, 2026, 6:54 PM, to November 13, 2026, 5:10 PM (Swiss time), many organizations had no or only intermittent access to their Exchange Online mailboxes. Microsoft listed the disruption in the Admin Center under EX1550543 (Exchange Online) and MO1550821 (other Microsoft 365 services). I documented the timeline in the [live ticker on the outage](https://rafaelpfister.ch/liveticker/exchange-online-ausfall-2026-11). Microsoft guarantees monthly availability of 99.9 percent for Exchange Online. An outage of more than 22 hours is well below that. However, you will only receive a credit if you request it, and the process depends on how your organization purchased Microsoft 365.

## What the SLA guarantees

The relevant document is the “Service Level Agreement for Microsoft Online Services,” which Microsoft republishes monthly. This article is based on the version dated November 17, 2026. For Exchange Online, the SLA defines downtime as any period during which users cannot send or receive email using Outlook on the web. There are no scheduled maintenance windows for Exchange Online.

Availability is calculated in user minutes:

```text
Verfügbarkeit (%) = (Benutzerminuten - Ausfallminuten) / Benutzerminuten x 100
```

User minutes are the number of minutes in the month multiplied by the number of users. Downtime minutes are the duration of each incident multiplied by the number of affected users. If availability falls below a threshold, a credit is issued against the monthly fee for the affected service:

| Monthly availability | Credit |
|---|---|
| below 99.9% | 25% |
| below 99% | 50% |
| below 95% | 100% |

The same thresholds apply to Microsoft Teams, SharePoint Online, and OneDrive, each with its own definition of downtime. For Teams, it only counts if users cannot chat or start meetings. During EX1550543, chats and meetings largely continued to work according to Microsoft; calendars and invitations were affected. A Teams claim is therefore unlikely to succeed.

In addition to mailbox availability, Exchange Online has other service levels, such as delivery time within Microsoft 365 and acceptance of incoming email. If an incident affects more than one of these, the SLA requires you to choose one. For the November outage, mailbox availability through Outlook on the web is the obvious metric.

## What the outage means in percentage terms

November 2026 has 30 days, or 43,200 minutes per user. This results in the following limits per user:

| Threshold | Permitted downtime in November |
|---|---|
| 99.9% | 43 minutes |
| 99% | 7 hours 12 minutes |
| 95% | 36 hours |

The disruption lasted 1,336 minutes from the initial report until resolution. Two examples for an organization with 200 mailboxes:

| Scenario | Downtime minutes | Availability | Credit |
|---|---|---|---|
| All 200 users had no access throughout the entire disruption | 200 x 1,336 = 267,200 | 96.91% | 50% |
| 120 users had no access for 10 hours | 120 x 600 = 72,000 | 99.17% | 25% |

The total number of user minutes is 200 x 43,200 = 8,640,000 in both cases. What matters is how many users were actually affected and for how long. Microsoft checks this using its own telemetry, but requires you to provide this information in the request. Many tenants in Europe were affected only from late in the evening on November 12 or only intermittently. Therefore, calculate using the times you can document, not the full duration of the Microsoft notice.

## What the credit is—and is not

The SLA addresses several points that should be clear before submitting a request:

1.  **Request required.** Microsoft does not review or grant credits on its own initiative. According to the SLA, a Microsoft status notice explicitly does not constitute confirmation that a service level was missed.

2.  **Credit, not refund.** The credit is applied against a future invoice. It is limited to the monthly fee for the affected service.

3.  **Exclusive remedy.** According to the SLA, credits are the exclusive remedy for availability issues. Microsoft does not reimburse lost revenue, overtime, or other consequential costs. You may not reduce invoices on your own.

4.  **Affected service only.** If you have a suite such as Microsoft 365 E3 or Business Premium, the credit applies to the Exchange Online portion, not the full license price. Microsoft determines that portion during its review; the SLA provides no formula for it.

5.  **No credit for free and trial offers.** Trials, previews, and free services are excluded, as are donated licenses for nonprofit organizations. Consumers with Microsoft 365 Personal or Family are not covered by this SLA.

6.  **Open agreements in days.** Those licensed through Open, Open Value, or Open Value Subscription receive any credit as additional usage days rather than a monetary amount.

7.  **Deadline.** For Microsoft 365 services, the request with all information must be received by the end of the month following the incident month. For the November outage, this is **December 31, 2026**. Azure has its own 60-day deadline.

## What information Microsoft requires

The SLA lists five required pieces of information. If any are missing, Microsoft will reject the request:

1.  a precise description of the incident,
2.  the time and duration of the downtime,
3.  the affected resources (for Exchange Online: tenant and service),
4.  the number and locations of affected users,
5.  a description of the errors that occurred.

The incident ID (EX1550543) and tenant ID should always be included. Both can be queried with Microsoft Graph PowerShell:

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
| `-Scopes "ServiceHealth.Read.All"` | Permission to read service health notifications |
| `-Scopes "Organization.Read.All"` | Permission to read organization data, including the tenant ID |
| `(Get-MgOrganization).Id` | outputs the tenant ID |
| `-ServiceHealthIssueId "EX1550543"` | retrieves this exact incident |
| `Select-Object …` | displays the incident ID, title, start, end, and classification |

</details>

The classification (`Classification`) shows whether Microsoft lists the incident as an `incident` or an `advisory`. According to Microsoft’s partner documentation, notices classified as “Advisory” are generally not eligible for credits. EX1550543 was classified as an Incident.

For the number and location of users, an analysis of licensed mailboxes by usage location is sufficient:

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
| `$abfrage = @{ … }` | collects the parameters for `Get-MgUser` in a hashtable (splatting), keeping the line short |
| `All` | retrieves all users, not just the first page |
| `Filter 'assignedLicenses/$count ne 0'` | only users with at least one assigned license |
| `ConsistencyLevel eventual` | required for the `$count` filter (advanced query) |
| `CountVariable anzahl` | stores the total count in the variable `$anzahl` |
| `Property UsageLocation` | also retrieves the usage location |
| `Group-Object UsageLocation` | counts users by country |

</details>

This number is the upper limit. Document how many users actually had no access using your own data: timestamped help desk tickets, error messages from Outlook and Outlook on the web (screenshots), results from availability tests in your monitoring system, and sign-in and access logs. The more precise this information is, the less the outcome depends solely on Microsoft’s telemetry.

## Small companies: usually through the partner

Small and medium-sized companies in Switzerland typically purchase Microsoft 365 through a partner in the Cloud Solution Provider Program (CSP) or directly by credit card in the Admin Center. You can see which route applies in the Microsoft 365 Admin Center under **Billing > Your products**: if a partner is listed as the source, the agreement is a CSP agreement.

### Purchased through a CSP partner

In this case, Microsoft does not accept the request directly from you. According to Partner Center, only CSP direct-bill partners and indirect providers (distributors) can request credits, not indirect resellers. If your IT service provider is an indirect reseller, the request therefore goes through them to their distributor and then to Microsoft. Each level has its own, often earlier deadlines. Contact them now rather than waiting until the end of December.

Your partner needs from you:

1.  the tenant ID,
2.  the incident ID (EX1550543),
3.  the period, number and locations of affected users, as well as the observed errors,
4.  an email requesting the SLA credit. It must come from an address in the affected domain; Microsoft does not accept email from a private address.

Microsoft already knows about the incident itself. The request must document that your organization was affected: according to Microsoft, the name, tenant ID, and ticket number alone are not sufficient. If the credit is granted, Microsoft credits it to the partner, who passes it on to you. Whether and how it does so is governed by your agreement with the partner. Ask if nothing appears on the next invoice.

### Purchased directly from Microsoft

If you paid Microsoft directly for Microsoft 365, submit the request yourself in the Admin Center. An administrator role is required:

1.  At `admin.cloud.microsoft` open **Help & support** in the lower-right corner.
2.  Describe the issue, for example, “Request an SLA service credit for a Microsoft 365 outage,” and select **Contact Support**.
3.  Choose email as the contact method. The subject should include “SLA service credit request” and the ID EX1550543.
4.  List the five required items from the previous section in the description and attach the evidence.

A separate request is required for each service and incident. According to the SLA, Microsoft generally processes requests within 45 days.

### Is it worthwhile for small companies?

For ten users, the credit is small: for a suite, only the Exchange portion counts, with 25 or 50 percent of that for one month. The effort is also small if you compile the information now. The request has a second benefit: it documents for your partner and management how severely your organization was affected. This helps later when considering whether an email fallback is necessary.

## Large companies: directly with Microsoft, using their own measurement data

Large organizations generally license Microsoft 365 through an Enterprise Agreement (EA) or the Microsoft Customer Agreement for Enterprise (MCA-E). The request goes directly to Microsoft, either through the Admin Center as described above or through the Unified Support agreement. If you have a Customer Success Account Manager (CSAM), inform them about the request; they can track its processing. For an EA through a Licensing Solution Partner (LSP), also clarify which invoice will show the credit.

With several thousand users, the amounts are substantial, and Microsoft reviews the information accordingly closely. The difference from a small tenant is in the evidence:

1.  **Determine affected users precisely.** Large tenants are distributed across multiple databases and regions in Microsoft data centers, and not all mailboxes were disrupted for the same length of time. Reporting all users as affected across the board is vulnerable to challenge in large tenants. Measurement data is more robust: synthetic tests against Outlook on the web by location, availability data from your own monitoring, and analysis of help desk tickets by location and time.

2.  **Specify locations.** The SLA requires the locations of affected users. For international organizations, it is worthwhile to break this down by country because the disruption affected North and South America first and other regions only later.

3.  **Clearly separate hybrid issues.** The SLA excludes delays in your own data center, such as email in the queue of local Exchange servers to the tenant. Document that the error occurred within Microsoft 365. The queues of local servers are good evidence because they show that Exchange Online did not accept email or accepted it only with delays.

4.  **Choose the service level.** If delivery times were primarily affected rather than mailbox access, the service level for delivery time may be the better basis. Only one request is permitted per incident and service, so the choice should be made before submitting the request.

5.  **Involve procurement and license management.** The credit appears on the invoice, not in the IT cost center. Coordinate with procurement on who submits the request, who verifies the credit, and how it is recorded.

Large organizations should also prepare the process for the next outage: a template containing the five required items, monitoring that measures the availability of Outlook on the web by location, and clear responsibility for SLA requests.

## Request template

The following template contains the information required by the SLA. Microsoft processes requests in English most quickly:

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

1.  [Service Level Agreement for Microsoft Online Services (version overview)](https://www.microsoft.com/licensing/docs/view/Service-Level-Agreements-SLA-for-Online-Services): version dated November 17, 2026, definition of downtime for Exchange Online, credit tiers, required information, deadlines, and exclusions.

2.  [Microsoft Learn: How and when to request a credit from Microsoft](https://learn.microsoft.com/en-us/partner-center/billing/request-credit): request process for CSP partners, exclusion of indirect resellers, required evidence, classification of Advisories.

3.  [Microsoft Learn: Get-MgServiceAnnouncementIssue](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.devices.serviceannouncement/get-mgserviceannouncementissue): cmdlet for retrieving a service health incident through Microsoft Graph.

4.  [Microsoft 365 Status (status.cloud.microsoft)](https://status.cloud.microsoft/): public status messages for EX1550543 and MO1550821.

5.  [Windows Latest: Microsoft 365 outage, you may get money off your bill](https://www.windowslatest.com/2026/11/13/microsoft-365-outage-you-may-get-money-off-your-bill-if-you-use-it-for-work-but-youll-have-to-ask-microsoft-for-it-as-a-server-failure-takes-down-outlook-teams-and-exchange/): request process in the Admin Center for the November outage, note regarding consumers.

6.  [Office Watch: Microsoft 365 went down? How to claim your SLA credit](https://office-watch.com/?p=78178): exclusion of consumer, free, and nonprofit licenses; one request per service and incident.

7.  [Live ticker: Exchange Online, worldwide mailbox access outage](https://rafaelpfister.ch/liveticker/exchange-online-ausfall-2026-11): timeline of the disruption on November 12 and 13, 2026, including Microsoft’s chronology.
