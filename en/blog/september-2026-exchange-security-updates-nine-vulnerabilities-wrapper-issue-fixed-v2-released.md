---
title: "September 2026 Exchange security updates: nine vulnerabilities, wrapper issue fixed, v2 released"
navTitle: "Exchange SU 09/2026"
description: "The September SU addresses nine vulnerabilities in Exchange SE and 2019 (eight in Exchange 2016), including a spoofing vulnerability with CVSS 9.3, and fixes the wrapper issue in hybrid environments. Version 2 followed on October 2 with an additional CVE; there are also three known issues with workarounds and a SettingOverride that should now be removed."
date: "2026-10-07"
kategorie: "Exchange On-Premises / Hybrid"
timeToRead: "7 min read"
themen:
  - exchange-updates
  - exchange-onprem-hybrid
produkte:
  - "exchange-updates"
protokolle:
  - "releases"
  - "powershell"
slug: "september-2026-exchange-security-updates-nine-vulnerabilities-wrapper-issue-fixed-v2-released"
translationId: article-53db0c02bc33f9bd
translationOf: exchange-security-updates-september-2026
url: https://rafaelpfister.ch/en/blog/september-2026-exchange-security-updates-nine-vulnerabilities-wrapper-issue-fixed-v2-released
translationSourceHash: d0738f61713a26973457a9e536720b9787af35b22867c648dbe113d2aeb3f4ea
translationModel: gpt-5.6-terra
translatedAt: 2026-10-08T10:43:52.590Z
translationReview: automatic
---

# September 2026 Exchange security updates: nine vulnerabilities, wrapper issue fixed, v2 released

Microsoft released security updates (SUs) for Exchange Server on September 8, 2026. They address nine vulnerabilities in Exchange SE and Exchange 2019, and eight in Exchange 2016. None was publicly known in advance, none is being actively exploited according to the Security Update Guide, and Microsoft rates all of them as *Important* with “Exploitation Less Likely.” However, the highest CVSS score, 9.3, is significantly higher than last month’s. This month stands out for three reasons: the SU fixes the *wrapper messages* issue in shared mailboxes that has been open since June, it introduces three new or ongoing known issues, and on October 2 Microsoft released **Version 2**, which addresses an additional vulnerability.

## Which Exchange versions the update is available for

The SUs from September 8, 2026 are available for the following versions:

- **Exchange Server Subscription Edition (SE) RTM**: KB5121608, build 15.2.2562.49; publicly available.
- **Exchange Server 2019 CU15**: KB5121609, build 15.2.1748.51; available only through the **Period 2 ESU program**.
- **Exchange Server 2019 CU14**: KB5121610, build 15.2.1544.46; available only through Period 2 ESU.
- **Exchange Server 2016 CU23**: KB5121611, build 15.1.2507.73; available only through Period 2 ESU.

Exchange 2016 and 2019 are out of support. According to Microsoft, only organizations enrolled in the Period 2 ESU program receive the SUs from May through October 2026. According to the KB articles, this entitlement extends through October 2026. There is also pressure from Exchange Online: since the second week of September, Exchange Online has been throttling and blocking hybrid mail flow from servers below the October 2025 level; details are in the [article on transport enforcement](/blog/exchange-online-transport-enforcement-hybrid-server). Exchange Online itself is already protected according to the announcement; nevertheless, every Exchange server in hybrid environments requires the SU, as do machines with the Exchange Management Tools.

You can compare your current level against the overview of [Exchange build numbers](/tools/exchange-builds).

## Vulnerabilities at a glance

| CVE | Type | CVSS |
| --- | --- | --- |
| CVE-2026-69356 | Spoofing (Cross-Site Scripting) | 9.3 |
| CVE-2026-69641 | Elevation of Privilege | 9.1 |
| CVE-2026-69355 | Remote Code Execution | 8.8 |
| CVE-2026-55007 | Remote Code Execution | 8.1 |
| CVE-2026-69380 | Elevation of Privilege | 8.1 |
| CVE-2026-69378 | Denial of Service | 7.5 |
| CVE-2026-69361 | Spoofing (Server-Side Request Forgery) | 6.5 |
| CVE-2026-69375 | Tampering | 6.5 |
| CVE-2026-69382 | Information Disclosure | 5.9 |

CVE-2026-55007 does not affect Exchange 2016; the Security Update Guide lists only Exchange SE and Exchange 2019 CU14/CU15 for it. One documentation detail: CVE-2026-69380 is missing from the CVE lists in the KB articles for the four September SUs. However, the Security Update Guide lists precisely the four September builds as the fix for this CVE (as of October 7, 2026).

**[CVE-2026-69356](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69356)** has the highest score at CVSS 9.3. According to Microsoft, an unauthenticated attacker can send a crafted calendar invitation containing a malicious meeting link; when the recipient opens the meeting and selects the link to join, the cross-site scripting is triggered. This requires user interaction, but no account in the organization.

**[CVE-2026-69380](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69380)** (Elevation of Privilege, CVSS 8.1) requires only a low-privileged account and an assigned mailbox. According to the FAQ in the Security Update Guide, an attacker can exploit weaknesses in request and identity token validation to impersonate another user and take over the mailboxes of all Exchange users: reading and sending email and downloading attachments. A single compromised user account is enough as a starting point.

**[CVE-2026-69641](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69641)** (Elevation of Privilege, CVSS 9.1) leads to the same result—the takeover of all mailboxes—but requires membership in a highly privileged role group.

The two remote code execution vulnerabilities have different prerequisites: [CVE-2026-69355](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69355) (CVSS 8.8) requires an authenticated low-privileged account, while [CVE-2026-55007](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-55007) (CVSS 8.1) can be triggered without authentication through a crafted Visio attachment, but according to Microsoft requires persistently low available memory on the target system. The other four vulnerabilities are: CVE-2026-69378 (DoS through uncontrolled recursion, without authentication), CVE-2026-69361 (SSRF, with the server sending HTTP requests to internal or loopback systems), CVE-2026-69375 (an authenticated attacker can replace file contents), and CVE-2026-69382 (disclosure of credentials through a weak cryptographic algorithm, requiring a previously stolen authentication cookie).

## Version 2 from October 2: CVE-2026-96940 added

On October 2, 2026, Microsoft released “Version 2” of the September SUs. According to the announcement, the only difference from the first release is the additional fix for **[CVE-2026-96940](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-96940). This elevation-of-privilege vulnerability has a CVSS score of 8.8, is neither publicly known nor exploited, but is the only one of the ten CVEs that Microsoft rates as **“Exploitation More Likely.”** An authenticated attacker can use it to access other mailboxes in the same organization and read email along with attachments. Exchange Online has already been fixed server-side.

| Version | KB | Build v2 |
| --- | --- | --- |
| Exchange SE RTM | KB5129955 | 15.2.2562.53 |
| Exchange 2019 CU15 | KB5129956 | 15.2.1748.53 |
| Exchange 2019 CU14 | KB5129957 | 15.2.1544.48 |
| Exchange 2016 CU23 | KB5129958 | 15.1.2507.75 |

In practice, this means that the fix for CVE-2026-96940 is included only in the v2 builds. Servers already running the September 8 SU additionally require v2. Anyone patching now can install v2 directly, since SUs are cumulative. The KB articles do not explicitly state whether servers with the first September SU must install v2; because the new CVE is fixed only in v2, that is the obvious interpretation.

## Wrapper issue fixed: remove the SettingOverride now

The issue known since the June SU, where *wrapper messages* appear in the inbox of shared mailboxes in hybrid environments, is fixed in the September SU for all four versions. The [August article](/blog/exchange-security-updates-august-2026) still stated that the SettingOverride documented as a workaround could remain in place. After installing the September SU, the opposite applies: in the associated support article, Microsoft recommends checking and removing the override.

```powershell
Get-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"
Remove-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"
```

<details class="options-details">
<summary>Options explained</summary>

| Command | Effect |
|---|---|
| `Get-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"` | Checks whether the workaround override is set in the organization. |
| `Remove-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"` | Removes the override once the September SU is installed. |

</details>

If the first command reports that the object `DisableBlockSharedAndUserMailboxHeaders` could not be found, Microsoft says no further action is necessary.

For Exchange SE, the SU also fixes an error in hybrid free/busy queries through Microsoft Graph: on-premises users saw busy times for Exchange Online mailboxes shifted by their own UTC offset, without an error message.

## Known issues

**Published calendars (.ics) return HTTP 500 to calendar apps.** The issue has existed since the August SU (Exchange SE from build 15.2.2562.46, as well as Exchange 2019 and 2016) and remains unfixed in the September SU and v2. Subscriptions to anonymously published calendars no longer update; the same URL works in a browser. According to Microsoft, the cause is that Exchange identifies clients by their user agent; calendar apps without a browser identifier end up in a code path disabled by the August SU. As a workaround, Microsoft describes an IIS URL Rewrite rule on the “Exchange Back End” site that appends the parameter `layout=premium` to .ics requests under `/owa/calendar/`; the IIS URL Rewrite module must be installed. The exact steps (using IIS Manager or directly in `applicationHost.config`) are in the [KB5126672 support article](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672). Microsoft does not provide a date for the fix (as of October 7, 2026).

**Free/busy for delegated mailboxes in hybrid environments (Exchange SE only).** If availability lookup is configured exclusively through the Graph API, queries for Exchange Online mailboxes using delegated on-premises access fail. Outlook reports “Your server location could not be determined,” OWA displays “No information,” and the EWS logs show `(403) Forbidden`. The documented workaround routes the queries through EWS again instead of Graph:

```powershell
Set-SettingOverride -Identity EnableRouteThroughMSGraphFeature -Parameters "Enabled=False"
Get-ExchangeDiagnosticInfo -Process Microsoft.Exchange.Directory.TopologyService -Component VariantConfiguration -Argument Refresh
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-Identity EnableRouteThroughMSGraphFeature` | The override that controls routing availability queries through Microsoft Graph. |
| `-Parameters "Enabled=False"` | Disables the Graph path, so queries run through EWS again. |
| `-Process Microsoft.Exchange.Directory.TopologyService` | Directs the diagnostic call to the topology service. |
| `-Component VariantConfiguration -Argument Refresh` | Reloads Variant Configuration so the override takes effect without waiting. |

</details>

In the announcement for v2, Microsoft lists this issue among the resolved issues. Anyone who set the workaround should check the [KB5127092 support article](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092) after installing v2 to see whether it should be reverted; as of October 7, 2026, it did not yet include instructions for doing so.

**ContentEngine deadlock due to missing Korean WordBreaker rule files (Exchange SE only).** With the September SU (build 15.2.2562.49 and v2 build 15.2.2562.53), the rule files for the updated Korean WordBreaker are not installed. The consequences are missing search results, delayed mail delivery, and Outlook or MAPI clients hanging or losing their connection. According to the announcement, this affects messages in Korean. The workaround in the [KB5130098 support article](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098) is to extract the two files `ko.token.rule.bin` and `ko.complex.rule.bin` from SQL Server 2025 Express RTM, verify their SHA256 hashes, copy them to the `Native` directory of the Exchange installation, and restart the Search Host Controller service. Microsoft is still investigating the issue.

## Installation and follow-up

Microsoft recommends the familiar process: inventory using the [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), determine the path with the [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) if the current version is outdated, install the SU, restart the server, and verify that all Exchange services are running. The Health Checker also shows afterward whether the SU was installed correctly. The Security Update Guide lists a restart as required for the updates.

Three follow-up tasks are required after installation:

1. Remove the wrapper SettingOverride `DisableBlockSharedAndUserMailboxHeaders` if it is set (see above).

2. On Exchange SE, check whether the free/busy and WordBreaker issues occur and apply the workarounds if necessary.

3. For published calendars with external subscribers, configure the URL Rewrite rule from KB5126672 if this has not already been done since the August SU.

In addition, from July, verify whether the CVE-2026-42897 mitigation (M2.1.0) is still active; instructions for removing it are in the [article on the July SU](/blog/exchange-security-updates-juli-2026).

## Recommended approach

Install the v2 builds from October 2 directly on all Exchange servers and Exchange Management Tools machines; servers with the September 8 SU additionally require v2 for CVE-2026-96940. The spoofing vulnerability with CVSS 9.3 and mailbox takeover through a low-privileged user account (CVE-2026-69380) are sufficient reason not to wait for the next Patch Tuesday. Then remove the wrapper override, review the three known issues, and run the Health Checker. The ESU program for Exchange 2016 and 2019 ends in October 2026; migration to Exchange SE can no longer be postponed.

## Sources

1.  [Released: September 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-september-2026-exchange-server-security-updates/4554411): Official release announcement with supported versions, ESU notice, known issues, resolved issues, and installation procedure (accessed through the Exchange Team Blog RSS feed).

2.  [Released: September 2026 V2 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/released-september-2026-v2-exchange-server-security-updates/ba-p/4561718): Announcement of v2 from October 2, 2026; the only difference is CVE-2026-96940.

3.  [Description of the security update for Microsoft Exchange Server Subscription Edition RTM: September 8, 2026 (KB5121608) – Microsoft Support](https://support.microsoft.com/help/5121608): CVE list, resolved issues, and the three known issues for Exchange SE.

4.  [Description of the security update for Microsoft Exchange Server 2019 CU15: September 8, 2026 (KB5121609) – Microsoft Support](https://support.microsoft.com/help/5121609): KB article for Exchange 2019 CU15.

5.  [Description of the security update for Microsoft Exchange Server 2019 CU14: September 8, 2026 (KB5121610) – Microsoft Support](https://support.microsoft.com/help/5121610): KB article for Exchange 2019 CU14.

6.  [Description of the security update for Microsoft Exchange Server 2016 CU23: September 8, 2026 (KB5121611) – Microsoft Support](https://support.microsoft.com/help/5121611): KB article for Exchange 2016 CU23, without CVE-2026-55007.

7.  [Security Update Guide – Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/): Type, CVSS, severity, exploitation assessment, and FAQ for all nine September CVEs and CVE-2026-96940, including affected builds for each CVE.

8.  [Description of version 2 of the security update for Microsoft Exchange Server Subscription Edition RTM October 2, 2026 (KB5129955) – Microsoft Support](https://support.microsoft.com/help/5129955): KB article for v2 for Exchange SE; the v2 articles for 2019 and 2016 are KB5129956 through KB5129958.

9.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Build numbers for the September SUs and v2 from October 2, 2026.

10. [Wrapper messages appear in shared mailbox in hybrid environments after installing the June 2026 Security Update – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/hotfix/2026/5105719): September SU fix and instructions for removing the SettingOverride.

11. [Published calendar (.ics) returns HTTP 500 for calendar applications – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672): Cause and URL Rewrite workaround.

12. [Availability (free/busy) fails for delegated mailboxes in Exchange hybrid deployments using Graph API only – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092): Symptoms and SettingOverride workaround for Exchange SE.

13. [ContentEngine deadlock because of missing Korean WordBreaker rule files – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098): Affected SE builds and manual workaround.

14. [Hybrid free/busy through Microsoft Graph incorrectly shifts busy times by requester timezone – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5125804): The time zone issue in Exchange SE fixed by the September SU.

15. [New security updates for Exchange Server (September 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-september-2026/): German-language breakdown of the nine CVEs with CVSS scores and builds.

16. [Exchange Server: Security updates September 8, 2026 – Borns Tech and Windows World](https://borncity.com/blog/?p=329371): German-language summary with notes on the later replacement update.
