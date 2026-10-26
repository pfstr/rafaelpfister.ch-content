---
title: "September 2026 Exchange security updates: nine vulnerabilities, Wrapper issue fixed, v2 released"
navTitle: "Exchange SU 09/2026"
description: "The September SU closes nine vulnerabilities in Exchange SE and 2019 (eight in Exchange 2016), including a spoofing vulnerability with CVSS 9.3, and fixes the Wrapper issue in hybrid environments. On October 2, v2 followed with an additional CVE; there are also three Known Issues with workarounds and a SettingOverride that should now be removed."
date: "2026-10-07"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "11 min read"
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
translationSourceHash: 4b2c59794f87e999c5b89865e0005faa7d5cbb36478f6b8475d5e3f121186a7d
translationModel: gpt-5.6-terra
translatedAt: 2026-10-26T08:29:01.071Z
translationReview: automatic
url: https://rafaelpfister.ch/en/blog/september-2026-exchange-security-updates-nine-vulnerabilities-wrapper-issue-fixed-v2-released
---

# September 2026 Exchange security updates: nine vulnerabilities, Wrapper issue fixed, v2 released

On September 8, 2026, Microsoft released security updates (SUs) for Exchange Server. They close nine vulnerabilities in Exchange SE and Exchange 2019, and eight in Exchange 2016. None of them was publicly known in advance, none is being actively exploited according to the Security Update Guide, and Microsoft rates all of them as *Important* with “Exploitation Less Likely.” However, the highest CVSS score, 9.3, is significantly higher than last month’s. This month is notable for three reasons: the SU fixes the *Wrapper messages* issue in shared mailboxes that has been open since June, it introduces three new or ongoing Known Issues, and on October 2 Microsoft released a **Version 2** that closes an additional vulnerability.

## Which Exchange versions the update is available for

The September 8, 2026 SUs are available for the following versions:

- **Exchange Server Subscription Edition (SE) RTM**: KB5121608, build 15.2.2562.49; publicly available.
- **Exchange Server 2019 CU15**: KB5121609, build 15.2.1748.51; only through the **Period 2 ESU program**.
- **Exchange Server 2019 CU14**: KB5121610, build 15.2.1544.46; only through Period 2 ESU.
- **Exchange Server 2016 CU23**: KB5121611, build 15.1.2507.73; only through Period 2 ESU.

Exchange 2016 and 2019 are out of support. According to Microsoft, only organizations enrolled in the Period 2 ESU program receive SUs from May through October 2026. According to the KB articles, this eligibility extends through October 2026. There is also pressure from Exchange Online: since the second week of September, Exchange Online has throttled and blocked hybrid mail flow from servers below the October 2025 level; see the [article on transport enforcement](/blog/exchange-online-transport-enforcement-hybrid-server). According to the announcement, Exchange Online itself is already protected; in hybrid environments, however, every Exchange server still needs the SU, as do machines with the Exchange Management Tools.

You can compare your current level with the overview of [Exchange build numbers](/tools/exchange-builds).

## Overview of the vulnerabilities

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

CVE-2026-55007 does not affect Exchange 2016; the Security Update Guide lists only Exchange SE and 2019 CU14/CU15 for it. One documentation detail: CVE-2026-69380 is missing from the CVE list in the KB articles for the four September SUs. However, the Security Update Guide identifies exactly the four September builds as the fix for this CVE (as of October 7, 2026).

**[CVE-2026-69356](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69356)** has the highest score, CVSS 9.3. According to Microsoft, an unauthenticated attacker can send a crafted calendar invitation containing a malicious meeting link; if the recipient opens the meeting and selects the link to join, Cross-Site Scripting is triggered. This therefore requires user interaction, but no account in the organization.

**[CVE-2026-69380](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69380)** (Elevation of Privilege, CVSS 8.1) requires only a low-privileged account and an assigned mailbox. According to the FAQ in the Security Update Guide, an attacker can exploit weaknesses in the validation of requests and identity tokens to impersonate another user and take over the mailboxes of all Exchange users: reading and sending mail and downloading attachments. A single compromised user account is sufficient as the starting point.

**[CVE-2026-69641](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69641)** (Elevation of Privilege, CVSS 9.1) leads to the same outcome, takeover of all mailboxes, but requires membership in a highly privileged role group.

The two Remote Code Execution vulnerabilities have different starting conditions: [CVE-2026-69355](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69355) (CVSS 8.8) requires a low-privileged authenticated account, while [CVE-2026-55007](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-55007) (CVSS 8.1) can be triggered without sign-in through a crafted Visio attachment but, according to Microsoft, requires persistently low available memory on the target system. The remaining four vulnerabilities are: CVE-2026-69378 (DoS through uncontrolled recursion, without sign-in), CVE-2026-69361 (SSRF, where the server sends HTTP requests to internal or loopback systems), CVE-2026-69375 (an authenticated attacker can replace file contents), and CVE-2026-69382 (disclosure of credentials through a weak cryptographic algorithm, requiring an authentication cookie that has already been obtained).

## Version 2 from October 2: CVE-2026-96940 added

On October 2, 2026, Microsoft released “Version 2” of the September SUs. According to the announcement, the only difference from the first release is the additional fix for **[CVE-2026-96940](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-96940). This Elevation of Privilege vulnerability has CVSS 8.8, is neither publicly known nor exploited, but Microsoft rates it as the only one of the ten CVEs with **“Exploitation More Likely.”** An authenticated attacker can use it to access other mailboxes in the same organization and read mail including attachments. Exchange Online has already been corrected server-side.

| Version | KB | Build v2 |
| --- | --- | --- |
| Exchange SE RTM | KB5129955 | 15.2.2562.53 |
| Exchange 2019 CU15 | KB5129956 | 15.2.1748.53 |
| Exchange 2019 CU14 | KB5129957 | 15.2.1544.48 |
| Exchange 2016 CU23 | KB5129958 | 15.1.2507.75 |

In practice, this means the fix for CVE-2026-96940 is included only in the v2 builds. Servers already running the September 8 SU additionally need v2. If you are patching only now, you can install v2 directly because SUs are cumulative. The KB articles do not explicitly say whether servers with the first September SU must install v2; since the new CVE is closed only by v2, that is the obvious interpretation.

## Wrapper issue fixed: remove the SettingOverride now

The issue known since the June SU, where *Wrapper messages* appear in the Inbox of shared mailboxes in hybrid environments, is fixed with the September SU on all four versions. The [August article](/blog/exchange-security-updates-august-2026) still stated that the SettingOverride documented as a workaround could remain in place. After installing the September SU, the opposite applies: in the related support article, Microsoft recommends checking and removing the override.

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

If the first command reports that the object `DisableBlockSharedAndUserMailboxHeaders` was not found, Microsoft says no further action is required.

For Exchange SE, the SU also fixes an error in hybrid free/busy queries through Microsoft Graph: on-premises users saw the busy times of Exchange Online mailboxes shifted by their own UTC offset, without an error message.

## Known issues

**Published calendars (.ics) return HTTP 500 to calendar apps.** The issue has existed since the August SU (Exchange SE from build 15.2.2562.46, as well as Exchange 2019 and 2016) and is not fixed in either the September SU or v2. Subscriptions to anonymously published calendars no longer update; the same URL works in a browser. According to Microsoft, the cause is that Exchange identifies clients by User-Agent; calendar apps without a browser identifier end up in a code path disabled by the August SU. As a workaround, Microsoft describes an URL Rewrite rule on the “Exchange Back End” site in IIS that appends the parameter `layout=premium` to .ics requests under `/owa/calendar/`; this requires the IIS URL Rewrite module to be installed. The exact steps (through IIS Manager or directly in `applicationHost.config`) are available in the [KB5126672 support article](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672). Microsoft does not specify a date for the fix (as of October 7, 2026).

**Free/busy for delegated mailboxes in hybrid environments (Exchange SE only).** If availability lookup is configured exclusively through the Graph API, queries for Exchange Online mailboxes through delegated on-premises access fail. Outlook reports “Your server location could not be determined,” OWA displays “No information,” and `(403) Forbidden` appears in the EWS logs. The documented workaround routes the queries through EWS again instead of Graph:

```powershell
Set-SettingOverride -Identity EnableRouteThroughMSGraphFeature -Parameters "Enabled=False"
Get-ExchangeDiagnosticInfo -Process Microsoft.Exchange.Directory.TopologyService -Component VariantConfiguration -Argument Refresh
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-Identity EnableRouteThroughMSGraphFeature` | The override that controls routing availability queries through Microsoft Graph. |
| `-Parameters "Enabled=False"` | Disables the Graph path; queries run through EWS again. |
| `-Process Microsoft.Exchange.Directory.TopologyService` | Directs the diagnostic call to the topology service. |
| `-Component VariantConfiguration -Argument Refresh` | Reloads Variant Configuration so the override takes effect without waiting. |

</details>

In the v2 announcement, Microsoft lists this issue among the fixed issues. If you set the workaround, after installing v2 you should check the [KB5127092 support article](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092) to see whether it should be reverted; as of October 7, 2026, instructions for doing so had not yet been published there.

**ContentEngine deadlock due to missing Korean WordBreaker files (Exchange SE only).** With the September SU (build 15.2.2562.49 and v2 build 15.2.2562.53), the rule files for the updated Korean WordBreaker are not installed. The consequences are missing search results, delayed mail delivery, and Outlook or MAPI clients that hang or lose their connection. [KB5130098](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098) identifies environments that process mail with Korean-language content as affected. This does not depend on the language of your own users: messages with Korean text also reach organizations without Korean correspondence, for example as spam or newsletters. Swiss environments are therefore not automatically excluded. Microsoft is still investigating the issue; verification and the workaround are in the next section.

## Workaround for the WordBreaker deadlock

The workaround from KB5130098 adds two rule files from the SQL Server 2025 Express RTM package. Microsoft ties it to conditions: it must not be applied based on symptoms alone, but only on servers that meet all verification criteria and are missing **both** files. The [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) does not check these files; version v26.10.02 from October 2, 2026 contains no corresponding check (as of October 9, 2026). The verification must therefore be performed manually.

### 1. Check the server

The following script does not change anything on the system. It reads the installation path from the environment variable `ExchangeInstallPath` and outputs the values that KB5130098 lists as prerequisites.

```powershell
$native  = Join-Path $env:ExchangeInstallPath 'Bin\Search\Ceres\Native'
$exsetup = Get-Item (Join-Path $env:ExchangeInstallPath 'Bin\ExSetup.exe')
$dll     = Get-Item (Join-Path $native 'korwbrkr.dll')
[pscustomobject]@{
    ExSetup         = $exsetup.VersionInfo.FileVersion
    KorwbrkrVersion = $dll.VersionInfo.FileVersion
    KorwbrkrBytes   = $dll.Length
    KorwbrkrSha256  = (Get-FileHash -LiteralPath $dll.FullName -Algorithm SHA256).Hash
    KoTokenRule     = Test-Path (Join-Path $native 'ko.token.rule.bin')
    KoComplexRule   = Test-Path (Join-Path $native 'ko.complex.rule.bin')
}
```

<details class="options-details">
<summary>Options explained</summary>

| Line | Effect |
|---|---|
| `Join-Path $env:ExchangeInstallPath 'Bin\Search\Ceres\Native'` | Builds the path to the directory where the rule files belong. |
| `Get-Item ...\ExSetup.exe` | Reads the file whose version indicates the installed Exchange build. |
| `VersionInfo.FileVersion` | File version from the EXE or DLL resources. |
| `Get-FileHash -LiteralPath ... -Algorithm SHA256` | Calculates the SHA256 hash of the installed `korwbrkr.dll`. |
| `Test-Path ...` | Returns `True` if the respective rule file is already present. |

</details>

The workaround is permitted only if all values match:

| Value | Expected |
| --- | --- |
| `ExSetup` | 15.02.2562.049 or 15.02.2562.053 |
| `KorwbrkrVersion` | 16.0.5194.1000 |
| `KorwbrkrBytes` | 326544 |
| `KorwbrkrSha256` | 1C6BD8E144BA677EBCC83323AE59DB3881918170F9B3A5189B44611558B92C61 |
| `KoTokenRule`, `KoComplexRule` | both `False` |

If the build, DLL version, or hash differs, Microsoft refers you to support. If either of the two rule files is already present, stop and do not overwrite anything.

### 2. Obtain the rule files

The files come from `SQLEXPR_x64_ENU.exe`, the English x64 package of SQL Server 2025 Express RTM (version 17.0.1000.7, 748772024 bytes). Microsoft recommends performing this step on a management workstation, not on the Exchange server. SQL Server is not installed in the process: the package is only extracted, and the Full-Text MSI is extracted administratively only. First verify the download signature and hash:

```powershell
Get-AuthenticodeSignature .\SQLEXPR_x64_ENU.exe |
    Select-Object Status, SignerCertificate
Get-FileHash .\SQLEXPR_x64_ENU.exe -Algorithm SHA256
```

Expected are status `Valid` with a Microsoft certificate and SHA256 value `74AA90C11202A5524E769B9BC22531BAEF22D91E9B2D2E8C3CB99E89A65C5297`. Then unpack and extract:

```powershell
$work = 'C:\Temp\KoreanRules'
$p = Start-Process .\SQLEXPR_x64_ENU.exe -ArgumentList '/q', "/x:$work\Media" -Wait -PassThru
$p.ExitCode
$msi = "$work\Media\x64\Setup\SQL_FULLTEXT.MSI"
$msiArgs = '/a', $msi, "TARGETDIR=$work\Files", '/qn', '/L*V', "$work\extract.log"
$p = Start-Process msiexec.exe -ArgumentList $msiArgs -Wait -PassThru
$p.ExitCode
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `/q` | Extracts without dialogs. |
| `/x:<Pfad>` | SQL package extract mode: extracts the installation media to the folder without starting SQL Setup. |
| `-Wait -PassThru` | Waits for the process to finish and returns the process object so the exit code can be read. |
| `msiexec /a <MSI>` | Administrative installation: saves the MSI contents as files without installing or registering anything. |
| `TARGETDIR=<Pfad>` | Target folder for the extracted files. |
| `/qn` | Without a user interface. |
| `/L*V <Datei>` | Detailed logging that shows the cause in the event of an error. |

</details>

Both exit codes must be `0`. If extraction fails, preserve `extract.log` and stop. Then verify the size and hash of both files:

```powershell
Get-ChildItem "$work\Files" -Recurse -Include 'ko.token.rule.bin', 'ko.complex.rule.bin' |
    Select-Object FullName, Length,
        @{ n = 'SHA256'; e = { (Get-FileHash -LiteralPath $_.FullName -Algorithm SHA256).Hash } }
```

| File | Bytes | SHA256 |
| --- | --- | --- |
| `ko.token.rule.bin` | 56132 | 8F2BD853593913EB8F73DCD4FCAC4216F216A0FF76A4569DF071BE3C36773010 |
| `ko.complex.rule.bin` | 717792 | 0390D1E9A76EF33283025CF8F164430E311584B9535949C4EA1A74B6BB107B87 |

The files are located in the extracted tree under `...\MSSQL\Binn\ftcomponents\wordbreakers`. Transfer only these two files to a staging folder on the Exchange server, no SQL media and explicitly not the `korwbrkr.dll` from the SQL package.

### 3. Copy the files

```powershell
$native  = Join-Path $env:ExchangeInstallPath 'Bin\Search\Ceres\Native'
$staging = 'C:\Temp\KoStaging'
foreach ($f in 'ko.token.rule.bin', 'ko.complex.rule.bin') {
    Copy-Item -LiteralPath (Join-Path $staging $f) -Destination $native
    Get-FileHash -LiteralPath (Join-Path $native $f) -Algorithm SHA256
}
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Copy-Item -LiteralPath <Quelle>` | Copies the file from the staging folder; the path is evaluated without wildcards. |
| `-Destination $native` | Target directory next to the installed `korwbrkr.dll`. |
| `Get-FileHash ... -Algorithm SHA256` | Verifies the hash at the destination again against the table above. |

</details>

Use `Copy-Item`, not `Move-Item`. A copied file receives the inherited permissions of the target directory; a moved file retains the permissions of its source folder. The files require the normal read permissions of `Native`. Do not replace any other files in this directory.

### 4. Restart Search Host Controller

```powershell
Restart-Service -Name HostControllerService
```

The restart temporarily interrupts search and content processing on the server; Microsoft provides for a maintenance window for this. You do not need to restart the server itself. Do not forcibly terminate Exchange processes: if the service does not start cleanly, Microsoft says support is the next step.

### 5. Verification

The Host Controller starts the ContentEngine process `NodeRunner.exe` itself; do not start it manually. This query shows whether it is running:

```powershell
Get-CimInstance Win32_Process -Filter "Name = 'NodeRunner.exe'" |
    Where-Object CommandLine -like '*ContentEngineNode1*' |
    Select-Object ProcessId, CreationDate
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Get-CimInstance Win32_Process -Filter "Name = 'NodeRunner.exe'"` | Reads all NodeRunner processes including their command lines. |
| `Where-Object CommandLine -like '*ContentEngineNode1*'` | Filters for the ContentEngine process. |
| `Select-Object ProcessId, CreationDate` | Displays process ID and start time; the start time must be after the service restart. |

</details>

If no process appears or it repeatedly terminates, do not work on any additional servers. Otherwise, use Outlook on the web with a mailbox whose active database copy is on this server to check whether new mail is delivered and found, including mail with Korean text. Also check the service that was originally affected (delivery, Outlook connection). Monitor the backlog in the search index separately: the fact that new mail can be found does not prove that older items have already been processed. Work on additional servers one at a time and only after successfully checking the previous one.

## Installation and follow-up

Microsoft recommends the familiar process: inventory with the [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), determine the path with the [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) if the current level is outdated, install the SU, restart the server, and check that all Exchange services are running. The Health Checker also shows afterward whether the SU was installed correctly. The Security Update Guide lists a required restart for the updates.

After installation, there are three follow-up tasks:

1. Remove the Wrapper SettingOverride `DisableBlockSharedAndUserMailboxHeaders` if it is set (see above).

2. On Exchange SE, use the verification script from the [WordBreaker section](#workaround-für-den-wordbreaker-deadlock) to check whether the two rule files are missing, and check whether the free/busy issue occurs. Apply the workarounds only when their respective conditions are met.

3. For published calendars with external subscribers, configure the URL Rewrite rule from KB5126672 if this has not already been done since the August SU.

Also remaining from July is checking whether the CVE-2026-42897 mitigation (M2.1.0) is still active; how to remove it is described in the [article on the July SU](/blog/exchange-security-updates-juli-2026).

## Recommended approach

Install the October 2 v2 builds directly on all Exchange servers and Exchange Management Tools machines; servers with the September 8 SU also need v2 for CVE-2026-96940. The spoofing vulnerability with CVSS 9.3 and mailbox takeover through a low-privileged user account (CVE-2026-69380) are sufficient reasons not to wait for the next Patch Tuesday. Then remove the Wrapper override, check the three Known Issues, and run the Health Checker. The Health Checker does not report the missing WordBreaker files; use the verification script in the WordBreaker section for that. The ESU program for Exchange 2016 and 2019 ends in October 2026; migration to Exchange SE can no longer be postponed.

## Sources

1.  [Released: September 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-september-2026-exchange-server-security-updates/4554411): Official release announcement with supported versions, ESU information, Known Issues, fixed issues, and installation process (accessed through the RSS feed of the Exchange Team Blog).

2.  [Released: September 2026 V2 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/released-september-2026-v2-exchange-server-security-updates/ba-p/4561718): Announcement of v2 from October 2, 2026; the only difference is CVE-2026-96940.

3.  [Description of the security update for Microsoft Exchange Server Subscription Edition RTM: September 8, 2026 (KB5121608) – Microsoft Support](https://support.microsoft.com/help/5121608): CVE list, fixed issues, and the three Known Issues for Exchange SE.

4.  [Description of the security update for Microsoft Exchange Server 2019 CU15: September 8, 2026 (KB5121609) – Microsoft Support](https://support.microsoft.com/help/5121609): KB article for Exchange 2019 CU15.

5.  [Description of the security update for Microsoft Exchange Server 2019 CU14: September 8, 2026 (KB5121610) – Microsoft Support](https://support.microsoft.com/help/5121610): KB article for Exchange 2019 CU14.

6.  [Description of the security update for Microsoft Exchange Server 2016 CU23: September 8, 2026 (KB5121611) – Microsoft Support](https://support.microsoft.com/help/5121611): KB article for Exchange 2016 CU23, without CVE-2026-55007.

7.  [Security Update Guide – Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/): Type, CVSS, severity, exploitation assessment, and FAQ for all nine September CVEs and CVE-2026-96940, including the affected builds for each CVE.

8.  [Description of version 2 of the security update for Microsoft Exchange Server Subscription Edition RTM October 2, 2026 (KB5129955) – Microsoft Support](https://support.microsoft.com/help/5129955): KB article for v2 for Exchange SE; the v2 articles for 2019 and 2016 are KB5129956 through KB5129958.

9.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Build numbers for the September SUs and v2 from October 2, 2026.

10. [Wrapper messages appear in shared mailbox in hybrid environments after installing the June 2026 Security Update – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/hotfix/2026/5105719): Fix in the September SU and instructions for removing the SettingOverride.

11. [Published calendar (.ics) returns HTTP 500 for calendar applications – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672): Cause and URL Rewrite workaround.

12. [Availability (free/busy) fails for delegated mailboxes in Exchange hybrid deployments using Graph API only – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092): Symptoms and SettingOverride workaround for Exchange SE.

13. [ContentEngine deadlock because of missing Korean WordBreaker rule files – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098): Affected SE builds, verification criteria, expected file sizes and hashes, acquisition from SQL Server 2025 Express, service restart, and verification.

14. [Hybrid free/busy through Microsoft Graph incorrectly shifts busy times by requester timezone – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5125804): The time zone issue in Exchange SE fixed by the September SU.

15. [New security updates for Exchange Server (September 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-september-2026/): German-language breakdown of the nine CVEs with CVSS scores and builds.

16. [Exchange Server: security updates September 8, 2026 – Borns Tech and Windows World](https://borncity.com/blog/?p=329371): German-language summary with references to the later replacement update.

17. [microsoft/CSS-Exchange – GitHub](https://github.com/microsoft/CSS-Exchange): Source code for the Exchange Health Checker; release v26.10.02 from October 2, 2026 without a check for the WordBreaker files (as of October 9, 2026).
