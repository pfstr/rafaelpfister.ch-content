---
title: "Exchange Online throttles and blocks outdated Exchange 2016 and 2019 servers starting in September 2026: How transport enforcement works"
navTitle: "EXO Enforcement 09/2026"
description: "Starting in the second week of September 2026, Exchange Online requires hybrid servers to have at least the October 2025 SU; otherwise, mail flow is throttled and later blocked. Background on transport enforcement since 2023, escalation stages with SMTP codes, the report in the Admin Center, the 90-day pause via PowerShell, and why the next increase will only allow ESU customers and Exchange SE."
date: "2026-09-07"
kategorie: "Exchange On-Premises / Hybrid"
timeToRead: "9 min read"
themen:
  - exchange-onprem-hybrid
  - exchange-updates
produkte:
  - "exchange-hybrid"
  - "exchange-online"
  - "hybrid-mailfluss"
  - "exchange-updates"
protokolle:
  - "smtp"
  - "migration"
  - "releases"
slug: "exchange-online-throttles-and-blocks-outdated-exchange-2016-and-2019-starting-in-september-2026"
translationId: "article-fff0c5efce59ef76"
draft: false
translationOf: exchange-online-transport-enforcement-hybrid-server
translationSourceHash: b2c2e61a4e97d49b1046134bf215a91e0d46ed817201023d8364945f3549d539
translationModel: gpt-5.6-terra
translatedAt: 2026-09-08T08:11:39.744Z
translationReview: automatic
url: https://rafaelpfister.ch/en/blog/exchange-online-throttles-and-blocks-outdated-exchange-2016-and-2019-starting-in-september-2026
---

# Exchange Online throttles and blocks outdated Exchange 2016 and 2019 servers starting in September 2026: How transport enforcement works

On September 2, 2026, the Exchange team announced an increase to the minimum version for Exchange 2016 and Exchange 2019 in hybrid mail flow. Starting in the second week of September 2026, Exchange Online will require servers that submit mail through an inbound connector of type `OnPremises` to be at least at the level of the last public security update from October 2025. Anything below that will be throttled and later blocked. In short: If you have not patched your hybrid servers since October 2025, you will gradually lose mail delivery to Exchange Online in the coming weeks. And the next increase, which Microsoft has indicated for the coming months, will be above every publicly available update: Then, only customers in the paid ESU program or environments with Exchange Server Subscription Edition (SE) will meet the requirement.

The announcement itself is brief. Its practical implications result from the enforcement system Microsoft has built up incrementally since 2023: which SMTP responses your server will receive, how to check the status in the Admin Center and through PowerShell, and which options remain during the transition until the ESU program ends in October 2026.

## What applies starting in the second week of September 2026

The new baseline corresponds to the security updates of October 14, 2025. That was the last Patch Tuesday on which Microsoft publicly provided updates for Exchange 2016 and 2019; all SUs since December 2025 have only been available through the ESU program.

| Version | Minimum level | KB | Build |
|---|---|---|---|
| Exchange 2019 CU15 | October 2025 SU (CU15 SU5) | KB5066367 | 15.2.1748.39 |
| Exchange 2016 CU23 | October 2025 SU (CU23 SU19) | KB5066369 | 15.1.2507.61 |

An October 2025 SU also exists for Exchange 2019 CU14 (KB5066368, build 15.2.1544.36). However, the Microsoft post explicitly names CU15 SU5 as the minimum version; CU14 has no longer been a recommended version since CU15 was released in February 2025. Include the move to CU15 in your plans if you are on CU14.

Three limitations are important:

- **Only hybrid mail flow is affected.** Exchange Online checks the version of submitting servers for messages that arrive through an inbound connector of type `OnPremises`. This is the classic hybrid configuration created by the Hybrid Configuration Wizard. Mail arriving through a third-party gateway or a connector of type `Partner` does not go through this enforcement.
- **The version is read from message headers.** An Exchange server writes its build to the `Received` line of every message it relays (`… with Microsoft SMTP Server … id 15.1.2507.61`). Exchange Online evaluates this information. Therefore, the relevant version is that of the server that actually hands the message over to Exchange Online—meaning, in many environments, the Edge Transport Server or the Mailbox server with the Send Connector to `*.mail.protection.outlook.com`.
- **Exchange SE is not affected.** The enforcement applies to Exchange 2016 and 2019; Exchange Server SE is above every baseline as long as it is patched regularly.

## Background: transport enforcement since 2023

The September announcement is not a new measure, but the next stage of a system Microsoft introduced in March 2023 under the title “Throttling and Blocking Email from Persistently Vulnerable Exchange Servers to Exchange Online.” Microsoft defines “persistently vulnerable” as any Exchange server that has either reached end of support or remains unpatched for known vulnerabilities. The goal is to protect Exchange Online recipients from messages originating from compromisable servers while also pressuring operators to patch or shut down those servers.

The system was enabled by version:

| Date | Affected version |
|---|---|
| August 2023 | Exchange 2007 |
| September 2023 | Exchange 2010 |
| December 2023 | Exchange 2013 |
| March 2024 | Exchange 2016 and 2019 (significantly outdated SU levels) |
| September 2026 | Exchange 2016 and 2019: baseline = October 2025 SU |
| “in a few months” | Exchange 2016 and 2019: baseline above the last public update |

For Exchange 2016 and 2019, the baseline had previously been versions that were “significantly behind on security updates.” What is new is that Microsoft is setting the limit at the last public update, thereby affecting for the first time servers that were still fully patched less than a year ago.

## The escalation stages

The enforcement operates in three functions Microsoft calls “reporting,” “throttling,” and “blocking.” As soon as a server falls below the baseline, a 90-day cycle begins. The stages from the 2023 overview post are:

| Time period | Measure | SMTP response |
|---|---|---|
| Day 0 through 30 | Report only in the Exchange Admin Center | none |
| Day 30 through 40 | Throttling for 5 minutes per hour | `450 4.7.230` |
| Day 40 through 50 | Throttling for 10 minutes per hour | `450 4.7.230` |
| Day 50 through 60 | Throttling for 20 minutes per hour | `450 4.7.230` |
| Day 60 through 70 | Throttling for 30 minutes per hour, plus blocking for 5 minutes per hour | `450 4.7.230` and `550 5.7.230` |
| Day 70 through 80 | Blocking for 10 minutes per hour | `550 5.7.230` |
| Day 80 through 90 | Blocking for 20 minutes per hour | `550 5.7.230` |
| Starting on day 90 | Complete blocking | `550 5.7.230` |

The two responses read as follows:

```text
450 4.7.230 Connecting Exchange server version is out-of-date;
connection to Exchange Online throttled for n mins/hr.

550 5.7.230 Connecting Exchange server version is out-of-date;
connection to Exchange Online blocked for n mins/hr.
```

The difference is critical operationally. With `450`, Exchange Online temporarily rejects the connection; the on-premises server retains the message in its queue and tries again. At first, users notice only delays. In Queue Viewer or in `Get-Queue`, the queue for the Send Connector to Exchange Online grows with the status `Retry` and the 4.7.230 message as `LastError`. With `550`, the rejection is final: the sender receives an NDR with code 5.7.230, and the message is lost unless it is sent again. Because blocking is initially active only for a few minutes per hour, the issue initially appears sporadic: some messages arrive, while others fail with an NDR. If you see this pattern in message tracking, check the version status first before looking for network or certificate issues.

The announcement does not say whether Microsoft will start the full 90-day cycle for the new baseline in the second week of September or begin at a later stage. The overview post states that, after a pause, the system continues at the stage previously reached. Do not rely on a 30-day grace period.

## Report in the Exchange Admin Center and through PowerShell

Exchange Online lists detected on-premises servers and their versions in a dedicated report: in the Exchange Admin Center under *Reports*, *Mail flow*, the report for out-of-date connecting on-premises Exchange servers. For each server, the report shows the detected build, whether it is below the baseline, and which enforcement stage applies.

Exchange Online PowerShell provides the same information:

```powershell
Connect-ExchangeOnline
Get-OnPremServerReportInfo
```

<details class="options-details">
<summary>Explaining the options</summary>

| Command | Effect |
|---|---|
| `Connect-ExchangeOnline` | Opens the session to Exchange Online (module `ExchangeOnlineManagement`). |
| `Get-OnPremServerReportInfo` | Lists on-premises servers detected by Exchange Online with build, enforcement status, and stage. |

</details>

The report only knows servers that actually hand off mail to Exchange Online. A management server without mail flow or a machine with only management tools does not appear in it. That is irrelevant to enforcement, but not to security: These systems also need the SUs.

## Pausing enforcement: 90 days per year

For environments that cannot meet the baseline in the short term, Microsoft offers a pause. It can be activated for a total of 90 days per year, either continuously or in multiple periods:

```powershell
Get-TenantExemptionInfo -BlockingScenario UnpatchedOnPremServer

New-TenantExemptionInfo -BlockingScenario UnpatchedOnPremServer `
  -NumberOfDays 30
```

<details class="options-details">
<summary>Explaining the options</summary>

| Option | Effect |
|---|---|
| `Get-TenantExemptionInfo` | Shows whether a pause is active for the tenant and how long it will last. |
| `New-TenantExemptionInfo` | Creates a new pause. |
| `-BlockingScenario UnpatchedOnPremServer` | Selects the “out-of-date on-premises server” scenario; there are currently no other scenarios for this cmdlet. |
| `-NumberOfDays 30` | Duration of the pause in days. The allowance is 90 days per year, and the specified number is deducted from it. |

</details>

Two properties of the pause are important in practice. First, when it expires, enforcement continues at the stage at which it was paused; the pause does not reset the 90-day cycle. Second, there is no cmdlet to end an active pause early: If you create a 90-day pause and finish patching after two weeks, you have used up the annual allowance. Therefore, make the pause as short as possible and extend it if necessary.

The pause is also only a solution for the current baseline. If Microsoft raises the limit above the last public update in a few months, an exhausted allowance will no longer help.

## Why the next increase is the real deadline

Exchange 2016 and 2019 have been out of support since October 14, 2025. Microsoft then introduced two paid ESU periods: Period 1 through April 2026 and Period 2 from May through October 2026. With the announcement of Period 2 on April 15, 2026, the Exchange team clarified that there will be no further extension. The SUs from December 2025 through August 2026 (most recently build 15.2.1748.49 for 2019 CU15 and 15.1.2507.72 for 2016 CU23) are available exclusively to ESU customers and are not offered for public download.

This results in the following situation:

- **Today**, a server with the October 2025 SU meets the baseline, with or without ESU.
- **With the next increase**, Microsoft says the baseline will be above the October 2025 level. Without an ESU contract, there is no legal way to reach that level. Hybrid mail flow from these servers will then be throttled and blocked, regardless of how well the rest of the environment is operated.
- **On October 31, 2026**, Period 2 also ends. After that, there will be no more SUs for Exchange 2016 and 2019, for anyone. At the latest, the increase after next will therefore also affect ESU customers.

At best, the ESU program buys only a few months. The only permanent version that enforcement allows is Exchange Server SE. Microsoft has also announced that Exchange SE CU2 (planned for the second half of 2026) will end coexistence with Exchange 2016 and 2019: installation will stop if older servers are found in the organization. Therefore, migration is due not only because of mail flow, but also to enable continued installation of updates for SE at all.

For environments that retain Exchange On-Premises only to manage attributes in a hybrid configuration, the alternative is to remove the last server: Since Exchange 2019 CU12, recipient attributes can be managed using the management tools without a running Exchange server. There is then no on-premises mail flow, and enforcement becomes irrelevant.

## Determining the version level

`Get-ExchangeServer` displays only the CU, not the SU, in `AdminDisplayVersion`. The file version of `ExSetup.exe` or the [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) is reliable; it also reports missing manual steps. For a quick overview of all servers:

```powershell
Get-ExchangeServer | ForEach-Object {
  $path = "\\$($_.Name)\C$\Program Files\Microsoft\Exchange Server\V15\bin\ExSetup.exe"
  [pscustomobject]@{
    Server  = $_.Name
    Version = (Get-Item $path).VersionInfo.ProductVersion
  }
}
```

<details class="options-details">
<summary>Explaining the options</summary>

| Element | Effect |
|---|---|
| `Get-ExchangeServer` | Lists all Exchange servers in the organization. |
| `\\<Server>\C$\...\ExSetup.exe` | Administrative share path to the setup file; adjust it if the installation path differs. |
| `VersionInfo.ProductVersion` | File version corresponding to the installed SU build (for example, `15.1.2507.61`). |

</details>

If the version is below `15.2.1748.39` (2019 CU15) or `15.1.2507.61` (2016 CU23), the server will fall below the baseline starting in the second week of September.

## Recommended approach

1. **Inventory the version levels** as described above, including Edge Transport Servers and management servers.

2. **Check the report in Exchange Online.** `Get-OnPremServerReportInfo` shows which servers Exchange Online actually sees and whether an enforcement stage is already active. Compare the list with the inventory: Servers missing from it do not submit through the `OnPremises` connector.

3. **Install at least the October 2025 SU.** KB5066367 (2019 CU15) and KB5066369 (2016 CU23) are still publicly available from the Microsoft Download Center. SUs are cumulative; a server at the August 2025 level can be updated directly to October 2025. For CU14, install CU15 first. After installation, restart, check service status, and run the Health Checker again.

4. **Use the pause only as a bridge.** If the update cannot be completed in the first half of September, create `New-TenantExemptionInfo` with a short duration and do not treat the pause as planning reserve for the next increase.

5. **Schedule the migration to Exchange SE.** Without an ESU contract, the next increase is the hard deadline; with ESU, it is October 31, 2026. Exchange 2019 CU15 can be upgraded in place to SE; Exchange 2016 requires the detour of a fresh SE installation and mailbox or role moves. If you operate Exchange only for attribute management, remove the last server and continue using the management tools.

## Sources

1.  [Exchange 2016/2019: Throttling and Blocking up to the Final Public Update Baseline – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/exchange-20162019-throttling-and-blocking-up-to-the-final-public-update-baseline/4552717): The September 2, 2026 announcement with the new baseline (October 2025 SU), the start date in the second week of September, and the note about the upcoming increase beyond the last public update.

2.  [Throttling and Blocking Email from Persistently Vulnerable Exchange Servers to Exchange Online – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/throttling-and-blocking-email-from-persistently-vulnerable-exchange-servers-to-e/3815328): The 2023 overview post with the definition of “persistently vulnerable,” the reporting, throttling, and blocking stages, the 90-day cycle, SMTP responses 4.7.230 and 5.7.230, and the phased rollout plan by version.

3.  [How to pause throttling and blocking of out-of-date on-premises Exchange Servers – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/how-to-pause-throttling-and-blocking-of-out-of-date-on-premises-exchange-servers/4007169): The cmdlets `Get-OnPremServerReportInfo`, `Get-TenantExemptionInfo`, and `New-TenantExemptionInfo`, along with the report in the Exchange Admin Center and the annual 90-day allowance.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Build numbers for the October 2025 SUs and subsequent ESU updates through August 2026; it also notes that only ESU customers receive SUs from December 2025 onward.

5.  [Description of the security update for Microsoft Exchange Server 2019 CU15: October 14, 2025 (KB5066367) – Microsoft Support](https://support.microsoft.com/en-us/topic/description-of-the-security-update-for-microsoft-exchange-server-2019-cu15-october-14-2025-kb5066367-19a1091c-e0b3-4078-be6b-312463063d05): The KB article for the minimum level for Exchange 2019 CU15.

6.  [Description of the security update for Microsoft Exchange Server 2016 CU23: October 14, 2025 (KB5066369) – Microsoft Support](https://support.microsoft.com/en-us/topic/description-of-the-security-update-for-microsoft-exchange-server-2016-cu23-october-14-2025-kb5066369-8ca2ab99-dfde-4329-896a-3faa677d2603): The KB article for the minimum level for Exchange 2016 CU23.

7.  [Support for Exchange Server 2016 and Exchange Server 2019 ends today – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/support-for-exchange-server-2016-and-exchange-server-2019-ends-today/4461192): End of support on October 14, 2025.

8.  [Announcing Exchange 2016 / 2019 Extended Security Update program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-exchange-2016--2019-extended-security-update-program/4433495): Terms of the first ESU period.

9.  [Announcing Period 2 Exchange 2016/2019 Extended Security Update (ESU) program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-period-2-exchange-20162019-extended-security-update-esu-program/4511603): Duration from May through October 2026 and the statement that no further extension will follow.

10. [Exchange Online transport enforcement system explained – CodeTwo Admin's Blog](https://www.codetwo.com/admins-blog/persistently-vulnerable-exchange-server/): Tabular breakdown of the eight enforcement stages and rollout dates for each Exchange version; third-party source.

11. [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventory of CU/SU levels and outstanding manual steps.
