---
title: "October 2026 Exchange security updates: six vulnerabilities, final update for Exchange 2016 and 2019"
navTitle: "Exchange SU 10/2026"
description: "The October SU closes six vulnerabilities, including a critical remote code execution through a crafted email, and fixes the .ics issue and the WordBreaker deadlock. It is the final update for Exchange 2016 and 2019, including under the ESU program."
date: "2026-10-14"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "7 min read"
themen:
  - exchange-updates
  - exchange-onprem-hybrid
produkte:
  - "exchange-updates"
protokolle:
  - "releases"
  - "powershell"
slug: "october-2026-exchange-security-updates-six-vulnerabilities-final-update-for-exchange-2016-and"
translationId: article-173db9431239a95f
translationOf: exchange-security-updates-oktober-2026
url: https://rafaelpfister.ch/en/blog/october-2026-exchange-security-updates-six-vulnerabilities-final-update-for-exchange-2016-and
translationSourceHash: ba4b8563ea0b25f5badf03f09e69f9b8329bb75bf4c2422cb5c3fbb97f55d137
translationModel: gpt-5.6-terra
translatedAt: 2026-10-14T08:15:25.503Z
translationReview: automatic
---

# October 2026 Exchange security updates: six vulnerabilities, final update for Exchange 2016 and 2019

Microsoft released security updates (SUs) for Exchange Server SE, Exchange 2019, and Exchange 2016 on October 13, 2026. According to the KB articles, they address six vulnerabilities. None was publicly known in advance, and according to the Security Update Guide, none is being actively exploited. However, Microsoft rates two as “Exploitation More Likely,” including a *Critical* remote code execution that an unauthenticated attacker can trigger with a crafted email. This month is notable for two reasons: the SU resolves the two open Known Issues from the September updates (HTTP 500 for published calendars and the ContentEngine deadlock in Exchange SE), and according to Microsoft, it is the final security update ever for Exchange 2016 and 2019, including for customers in the Period 2 ESU program.

## Which Exchange versions the update is available for

The SUs from October 13, 2026, are available for the following versions:

- **Exchange Server Subscription Edition (SE) RTM**: KB5128895, build 15.2.2562.56; publicly available.
- **Exchange Server 2019 CU15**: KB5128896, build 15.2.1748.55; only through the **Period 2 ESU program**.
- **Exchange Server 2019 CU14**: KB5128897, build 15.2.1544.50; only through Period 2 ESU.
- **Exchange Server 2016 CU23**: KB5128898, build 15.1.2507.77; only through Period 2 ESU.

You can compare your own version against the [Exchange build numbers](/tools/exchange-builds).

Exchange 2016 and 2019 are out of support. According to the KB articles, organizations in the Period 2 ESU program receive security updates through the end of October 2026; everyone else should migrate to Exchange SE. According to the announcement, Exchange Online is already protected; in hybrid environments, however, every Exchange server still needs the SU, including management-only servers and machines with the Exchange Management Tools. See the [Final update for Exchange 2016 and 2019](#letztes-update-für-exchange-2016-und-2019) section for what the end of ESU specifically means.

## Overview of the vulnerabilities

| CVE | Type | CVSS |
| --- | --- | --- |
| CVE-2026-70279 | Spoofing (Cross-Site Scripting) | 9.3 |
| CVE-2026-77910 | Elevation of Privilege | 8.8 |
| CVE-2026-70571 | Remote Code Execution | 8.1 |
| CVE-2026-87692 | Elevation of Privilege | 7.8 |
| CVE-2026-87695 | Elevation of Privilege | 7.8 |
| CVE-2026-87696 | Elevation of Privilege | 7.5 |

All four KB articles list the same six CVEs. However, Exchange 2019 CU15 is missing from the Security Update Guide for CVE-2026-87692; only Exchange SE, 2019 CU14, and 2016 CU23 are listed as affected there (as of October 14, 2026). Frankys Web interprets this as meaning that 2019 CU15 is not affected by this vulnerability. The difference does not matter for installation, since the SU on CU15 closes all the other vulnerabilities anyway.

**[CVE-2026-70571](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-70571)** is the month’s only vulnerability that Microsoft rates as *Critical* (Remote Code Execution, CVSS 8.1). The cause is a stack-based buffer overflow. According to the FAQ in the Security Update Guide, an unauthenticated attacker can send a crafted email to an affected Exchange server and thereby trigger an internal conversion process; no user interaction is required. According to the CVSS vector, attack complexity is high (AC:H); Microsoft states that successful exploitation is not guaranteed and requires an in-depth understanding of the target environment. Nevertheless, Microsoft rates exploitation as “Exploitation More Likely.” The attacker does not need an account in the organization; it is enough for the server to accept email.

**[CVE-2026-70279](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-70279) has the highest score at CVSS 9.3 and is also rated “Exploitation More Likely.” It is a cross-site scripting vulnerability: an unauthenticated attacker sends a crafted link; if an administrator opens it, the attacker can perform actions with that administrator’s permissions and thus access or modify organizational data. According to the FAQ, this requires user interaction, typically through social engineering. The severity rating is *Important*.

**[CVE-2026-77910](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-77910)** (Elevation of Privilege, CVSS 8.8) requires only a low-privileged account. According to the FAQ, an attacker can manipulate a shared cache of the Exchange Control Panel, causing a malicious mail add-in to be installed in another user’s mailbox without any action by that user. Microsoft describes the same attack path for [CVE-2026-87696](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-87696) (CVSS 7.5), where it results from insufficient validation of cryptographic signatures and has high attack complexity.

The remaining two vulnerabilities, [CVE-2026-87692](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-87692) (deserialization of untrusted data) and [CVE-2026-87695](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-87695), are local elevation-of-privilege vulnerabilities with CVSS 7.8: according to Microsoft, a logged-on low-privileged attacker can obtain SYSTEM privileges. Both are rated “Exploitation Less Likely,” as are CVE-2026-77910 and CVE-2026-87696.

Regarding **CVE-2026-96940** from September v2: the October announcement does not mention v2. According to the announcement FAQ, SUs are cumulative; if you are on a supported CU, install only the latest SU. Frankys Web confirms that the October SU includes the fix from v2. Servers still running the first September SU from September 8 can therefore install the October SU directly; details about v2 are in the [article on September v2](/blog/exchange-security-updates-september-2026-v2).

## Final update for Exchange 2016 and 2019

The announcement is clear on this point: October 2026 is the final month in which customers in the Period 2 ESU program receive new security updates for Exchange 2016 and 2019. Starting with the next release, updates will be issued only for Exchange SE. Future announcements and CVE reports will no longer mention 2016 and 2019, and Microsoft explicitly writes: “We will stop testing Exchange 2016 and 2019 for vulnerabilities.” Therefore, the fact that a CVE does not mention these versions does not mean they are unaffected; Microsoft emphasizes in the announcement that the older versions do not become secure as a result. Frankys Web additionally points out that Exchange SE and 2019 CU15 are technically very similar, so future SE vulnerabilities will very likely affect 2019 as well.

According to the announcement, Period 2 ESU can still be purchased for a limited time. However, this only brings an organization up to the October 2026 update level; no further updates will follow. Organizations that have already purchased an ESU and have questions about access can contact `ExchangeandSfBServerESUInquiry@service.microsoft.com` according to the KB articles.

For operations, this means:

1. **Exchange 2016 and 2019 with ESU**: Install the October SU. The version will then remain frozen, and every vulnerability disclosed later will remain unpatched.

2. **Exchange 2016 and 2019 without ESU**: These servers have received no security updates since May 2026. Microsoft recommends migrating to Exchange SE as soon as possible.

3. **Hybrid environments**: In addition to the missing patch, Exchange Online throttles and blocks mail flow from outdated on-premises servers; the rules are in the [article on transport enforcement](/blog/exchange-online-transport-enforcement-hybrid-server).

## Resolved issues and what you need to do afterward

**Published calendars (.ics) return HTTP 500 to calendar apps** (all versions). The issue that existed since the August SU is resolved with the October SU; [KB5126672](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672) now lists installing the four October updates or later as the solution. The support article no longer describes the previous URL rewrite workaround on the “Exchange Back End” site, and Microsoft does not provide instructions for removing the rule (as of October 14, 2026). If you configured the rule, document which servers it is active on and test a calendar subscription before removing it.

**ContentEngine deadlock due to missing Korean WordBreaker files** (Exchange SE only). According to [KB5130098](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098), the issue is resolved with KB5128895 or a later update, and “no additional action” is required after installing the October update. This also applies to servers on which you applied the manual workaround from the [September article](/blog/exchange-security-updates-september-2026#workaround-für-den-wordbreaker-deadlock): Microsoft does not require you to roll back the copied rule files. Servers without the workaround no longer need to add the files manually.

**Free/busy for delegated mailboxes in hybrid environments with a Graph API-only configuration** (Exchange SE only). This issue is not fixed by the October SU: [KB5127092](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092) now states, “The issue has been fixed in the service.” This resolves the contradiction from the [v2 article](/blog/exchange-security-updates-september-2026-v2). If you set the workaround, remove it according to KB5127092:

```powershell
Get-SettingOverride -Identity EnableRouteThroughMSGraphFeature
Set-SettingOverride -Identity EnableRouteThroughMSGraphFeature -Parameters "Enabled=True"
Get-ExchangeDiagnosticInfo -Process Microsoft.Exchange.Directory.TopologyService -Component VariantConfiguration -Argument Refresh
```

<details class="options-details">
<summary>Explaining the options</summary>

| Option | Effect |
|---|---|
| `Get-SettingOverride -Identity EnableRouteThroughMSGraphFeature` | Shows the current value of the override; the next command is needed only when `Enabled=False` is set. |
| `-Parameters "Enabled=True"` | Re-enables routing availability queries through Microsoft Graph. |
| `-Process Microsoft.Exchange.Directory.TopologyService` | Directs the diagnostic call to the topology service. |
| `-Component VariantConfiguration -Argument Refresh` | Reloads Variant Configuration so that the change takes effect immediately. |

</details>

Then check the free/busy query for a delegated Exchange Online mailbox from Outlook or OWA.

## Known issues

Neither the announcement nor the four KB articles list Known Issues for the October SU (as of October 14, 2026). The three open issues from the September updates are therefore resolved: two by the October SU and one by a fix in Exchange Online (see above). The wrapper issue in shared mailboxes had already been resolved with the September SU.

## Installation and follow-up

Microsoft describes the familiar process in the announcement: inventory with the [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), determine the path with the [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) if the CU version is outdated, install the SU, restart the server, and verify that all Exchange services are running. For errors, Microsoft refers to the [SetupAssist script](https://aka.ms/ExSetupAssist). According to KB5128895, after installation the Health Checker shows whether the SU was installed correctly and whether further steps are needed. The Security Update Guide does not list a required restart for the October builds; the announcement nevertheless recommends one after setup.

According to KB5128895, the installation file for Exchange SE is named `ExchangeSubscriptionEdition-KB5128895-x64-en.exe` (Download Center: “SU11”), for Exchange 2019 CU15 `Exchange2019-KB5128896-x64-en.exe`, for CU14 `Exchange2019-KB5128897-x64-en.exe`, and for Exchange 2016 `Exchange2016-KB5128898-x64-en.exe`. The packages are CU-specific.

For hybrid environments, the FAQ states: if the auth certificate is changed after installing an SU, run the Hybrid Configuration Wizard again. On Windows Server 2025, installed Exchange SUs do not appear in Control Panel; Microsoft advises against uninstalling SUs.

The following follow-up work from previous months remains outstanding if not already completed: remove the wrapper SettingOverride `DisableBlockSharedAndUserMailboxHeaders` (see the [September article](/blog/exchange-security-updates-september-2026)) and check whether the CVE-2026-42897 mitigation (M2.1.0) is still active (see the [article on the July SU](/blog/exchange-security-updates-juli-2026)).

## Recommended approach

Install the October SU promptly on all Exchange servers and machines with the Management Tools: the critical remote code execution CVE-2026-70571 can be triggered by email without an account, and Microsoft considers its exploitation as likely as that of the cross-site scripting vulnerability CVE-2026-70279. Servers with the first September SU also receive the fix for CVE-2026-96940. Then remove the free/busy override if set, and check published calendars. For Exchange 2016 and 2019, this is the final update; plan the migration to Exchange SE now, because these servers will receive no security updates after November.

## Sources

1.  [Released: October 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/released-october-2026-exchange-server-security-updates/ba-p/4563014): Announcement from October 13, 2026, with supported versions, resolved issues, the end of updates for Exchange 2016/2019, and FAQ (accessed through the RSS feed of the Exchange Team Blog; no comments from the Exchange team, as of October 14, 2026).

2.  [Description of the security update for Microsoft Exchange Server Subscription Edition RTM October 13, 2026 (KB5128895) – Microsoft Support](https://support.microsoft.com/help/5128895): CVE list, resolved issues, installation file, and Health Checker information.

3.  [Description of the security update for Microsoft Exchange Server 2019 CU15 October 13, 2026 (KB5128896) – Microsoft Support](https://support.microsoft.com/help/5128896): KB article for Exchange 2019 CU15 with ESU information.

4.  [Description of the security update for Microsoft Exchange Server 2019 CU14 October 13, 2026 (KB5128897) – Microsoft Support](https://support.microsoft.com/help/5128897): KB article for Exchange 2019 CU14 with the ESU contact address.

5.  [Description of the security update for Microsoft Exchange Server 2016 CU23 October 13, 2026 (KB5128898) – Microsoft Support](https://support.microsoft.com/help/5128898): KB article for Exchange 2016 CU23.

6.  [Security Update Guide – Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/): Type, CVSS, severity rating, exploitation assessment, FAQ, and affected builds for the six October CVEs and CVE-2026-96940.

7.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Build numbers for the October SUs and previous updates.

8.  [Published calendar (.ics) returns HTTP 500 for calendar applications – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672): Fix through the October updates.

9.  [ContentEngine deadlock because of missing Korean WordBreaker rule files – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098): Fix with KB5128895; no further action required.

10. [Availability (free/busy) fails for delegated mailboxes in Exchange hybrid deployments using Graph API only – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092): Service fix and commands to reset the SettingOverride.

11. [New security updates for Exchange Server (October 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-oktober-2026/): German-language breakdown of the six CVEs, information about the included v2 fix, and assessment of the end of ESU.

12. [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventory of CU/SU versions before and after installation.
