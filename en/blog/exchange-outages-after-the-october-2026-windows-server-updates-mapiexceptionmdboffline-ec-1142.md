---
title: "Exchange outages after the October 2026 Windows Server updates: MapiExceptionMdbOffline (ec=1142)"
navTitle: "Exchange outage 10/2026"
description: "After the October 13, 2026 Windows Server updates, Exchange servers report MapiExceptionMdbOffline (ec=1142) after around 24 hours: OWA fails, queues grow, and DAG copies stop replicating. The cause is a new limit in the RPC Endpoint Mapper; since October 17, 2026, Microsoft has recommended the EpmapSavedContextCap registry value, but a fix is still unavailable. Symptoms, affected KBs, verification commands, and procedure."
date: "2026-10-16"
kategorie: "Exchange On-Prem / Hybrid"
timeToRead: "10 min read"
themen:
  - exchange-updates
  - exchange-onprem-hybrid
produkte:
  - "exchange-updates"
  - "exchange-on-premises"
protokolle:
  - "troubleshooting"
  - "releases"
  - "powershell"
slug: "exchange-outages-after-the-october-2026-windows-server-updates-mapiexceptionmdboffline-ec-1142"
featured: "2026-10-16"
warnung: true
translationId: "article-8fdf2b2744bdbd41"
aiPrompt: |
  Du bist mein Assistent für Exchange Server On-Premises. Seit den Windows-Server-Updates vom Oktober 2026 (KB5126646, KB5126655, KB5126651, KB5126868) melden Exchange-Server MapiExceptionMdbOffline (hr=0x80004005, ec=1142), Event ID 1006, wachsende Warteschlangen und nicht replizierende DAG-Kopien. Hilf mir Schritt für Schritt: ermitteln, welche Oktober-Updates (Windows, .NET Framework, Exchange SU) auf jedem Exchange-Server installiert sind, den Zustand von Datenbanken, Kopien und Warteschlangen prüfen, den Betrieb mit kontrollierten Neustarts überbrücken und die Windows-Updates auf noch nicht betroffenen Servern zurückstellen. Frage zuerst nach Exchange-Version, Windows-Server-Version, Anzahl Server, ob eine DAG im Einsatz ist und wie Updates verteilt werden (WSUS, Intune, manuell).
translationOf: exchange-windows-updates-oktober-2026
translationSourceHash: 47c39c05826230354e6b431330725587440880dac2b6a33c00f5d34da2afe2d7
translationModel: gpt-5.6-terra
translatedAt: 2026-10-19T07:49:05.502Z
translationReview: automatic
url: https://rafaelpfister.ch/en/blog/exchange-outages-after-the-october-2026-windows-server-updates-mapiexceptionmdboffline-ec-1142
---

# Exchange outages after the October 2026 Windows Server updates: MapiExceptionMdbOffline (ec=1142)

Since the October 13, 2026 Patch Tuesday, Exchange servers have been failing after around 24 hours of uptime: clients lose their connection to the mailbox, OWA reports `MapiExceptionMdbOffline: Unable to make connection to the server. (hr=0x80004005, ec=1142)`, emails remain in queues, and database copies in DAGs stop replicating. Microsoft added a note to the October Exchange SU announcement on October 15, 2026: the October updates for Windows Server are the trigger, and installing the Exchange SU is not required for the issue to occur. Since October 17, 2026, the cause and an official registry workaround have been known; a fix is not yet available. Details follow in the next section.

## Update October 19, 2026: Cause and registry workaround

The Exchange team published a dedicated post about the issue on October 17, 2026. The Windows update release notes (for example, KB5126646) now list the issue as a Known Issue with the status “unresolved.”

### Cause

The October updates limit the memory that the RPC Endpoint Mapper uses for endpoint lookup calls. By default, only 512 saved contexts are now allowed; according to Microsoft, that is insufficient for workloads such as Exchange. Exchange accumulates these contexts during operation until the limit is reached. After that, RPC connections to the mailbox database fail, and OWA reports `MapiExceptionMdbOffline`. A reader at Born verified the new configuration value `EpmapSavedContextCap` in the `RpcEpMap.dll` of the October updates. Frankys Web attributes the limit to the Denial of Service vulnerability CVE-2026-96300 in the RPC Endpoint Mapper, which was fixed on the same Patch Tuesday.

This explains several observations from the first few days:

- The outage occurs only after several hours, and the timing depends on the workload. According to Microsoft, some servers are never affected.
- Restarting the server resets the counter and therefore helps only until the limit is reached again. According to a reader at Born, restarting only the Exchange services does not release the occupied memory.
- The limit is included in the cumulative Windows updates. The Exchange SU and .NET update are not involved.
- The suspected connection to database maintenance tasks and the replication service is therefore not the cause.

### Workaround

Microsoft recommends raising the limit through the registry and then restarting the server. In an elevated Command Prompt:

```cmd
reg add "HKLM\SOFTWARE\Microsoft\Rpc" /v EpmapSavedContextCap /t REG_DWORD /d 65536 /f
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `"HKLM\SOFTWARE\Microsoft\Rpc"` | Registry key for the RPC configuration. |
| `/v EpmapSavedContextCap` | Name of the value that sets the limit for saved Endpoint Mapper contexts (without an entry, 512 applies). |
| `/t REG_DWORD` | Data type: 32-bit number. |
| `/d 65536` | New limit, as recommended by Microsoft. |
| `/f` | Overwrites an existing value without prompting. |

</details>

The Endpoint Mapper reads the value only at startup; the change does not take effect without restarting the server. If the issue occurs again afterward, Microsoft recommends increasing the value further. According to Microsoft, the measure resolved the issue in all customer cases handled so far. According to Born, Nino Bilic (Microsoft) describes the entry as a temporary workaround that poses no security risk but is not the final solution. A reader at Born reports a server that has been running uninterrupted for more than 26 hours with 65536. Some administrators use higher values such as 16777215; so far, there has been no documented case in which 65536 was insufficient after a restart.

You can check whether the value is set with this query in PowerShell:

```powershell
Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Rpc" -Name EpmapSavedContextCap
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-Path "HKLM:\SOFTWARE\Microsoft\Rpc"` | Registry key containing the value. |
| `-Name EpmapSavedContextCap` | Reads only this value; if it is missing, PowerShell reports an error and the default limit of 512 applies. |

</details>

Set the value on all Exchange servers, including those that have not failed so far. On servers that do not yet have the October updates, you can set the value before installation; the restart after installation will then activate it. In a DAG, restart the members one at a time as described below. Frankys Web also provides a PowerShell script that sets the value, backs up the previous state, and can be reverted with `-Remove`.

According to Microsoft, the Windows updates should explicitly not be uninstalled. A reader at Frankys Web also reports RDP sign-in issues (CredSSP) after uninstalling KB5126868. If the registry workaround is insufficient, Microsoft offers an additional mitigation through Support for business. Once a fix is available, you should remove the registry value again because the higher limit weakens protection against CVE-2026-96300.

### Open questions

- **Fix**: Microsoft has not given a date. A reader at Born suspects that Microsoft could remove the limit through a Known Issue Rollback (Feature ID 63096240); this has not been confirmed.
- **Value**: It remains unclear whether 65536 will also suffice in large environments until the next Patch Tuesday. Continue monitoring Event ID 1006.
- **Other server roles**: A reader at Born also sees servers without Exchange affected. Microsoft comments only on Exchange.
- **Backup**: It is unclear whether the reported VSS Writer error is a consequence of the outage.

## Symptoms

Symptoms reported so far:

- OWA fails with `MapiExceptionMdbOffline: Unable to make connection to the server. (hr=0x80004005, ec=1142)`; Event ID 1006 appears in the Application Log.
- Outlook clients lose their connection to the mailbox.
- Emails are delivered neither internally nor externally, and transport queues grow.
- In DAGs, database copies stop replicating; according to reader reports at Born, switchovers fail when entering maintenance mode.
- The Exchange Admin Center still shows the databases as mounted (Frankys Web).

The error does not occur immediately after installation, but according to reports after about 22 to 30 hours of uptime. Restarting the server restores operation, but the outage repeats after approximately the same period. Comments at Born observed that the mailbox replication service stops during or after database maintenance tasks and MdbOffline errors begin afterward; based on the now-known cause, this is a side effect rather than the trigger.

## Affected updates

In its addendum to the Exchange announcement, Microsoft lists these cumulative Windows updates:

| Windows Server | Update |
| --- | --- |
| Windows Server 2025 | KB5126646 |
| Windows Server 2022 | KB5126655 |
| Windows Server 2019 | KB5126651 |
| Windows Server 2016 | KB5126868 |

The same Patch Tuesday also brought the Exchange SUs (Exchange SE: KB5128895, “SU11”) and cumulative updates for the .NET Framework. The following three packages were installed on an affected Exchange SE server running Windows Server 2019:

- Security Update For Exchange Server Subscription Edition (KB5128895)
- 2026-10 Cumulative Update for .NET Framework 3.5, 4.7.2 and 4.8 for Windows Server 2019 for x64 (KB5128923)
- 2026-10 Cumulative Update for Windows Server 2019 for x64-based Systems (KB5126651)

Microsoft does not mention the .NET update in the addendum. However, readers at Born report outages with the combination of Windows and .NET updates (KB5128928 there under Windows Server 2022), as well as a server with KB5126651 and the Exchange SU but no .NET update that was still running after more than 24 hours. It therefore remains unclear whether the .NET update is involved.

Regarding the Exchange SU: forums and comments say that the situation improves if Exchange SU11 is installed. This cannot be confirmed; outages also occurred on a server with SU11. Conversely, several administrators report outages without an installed Exchange SU, which is consistent with Microsoft’s statement. The Exchange SU closes six vulnerabilities, including a critical remote code execution issue through a specially crafted email; details in the [article on the October SU](/blog/exchange-security-updates-oktober-2026). Holding it back does not resolve the issue.

## Check status

First, determine which October updates are installed on each Exchange server:

```powershell
$kbs = "KB5126646", "KB5126655", "KB5126651", "KB5126868", "KB5128923"
Get-HotFix -Id $kbs -ErrorAction SilentlyContinue |
  Select-Object PSComputerName, HotFixID, InstalledOn
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `$kbs`, `-Id` | List of the KB numbers to look for: the four Windows CUs from the Microsoft addendum and the .NET update for Windows Server 2019. |
| `-ErrorAction SilentlyContinue` | Suppresses the error message for KB numbers that are not installed on this server. |
| `Select-Object PSComputerName, HotFixID, InstalledOn` | Displays server, KB number, and installation date; the date can be used to estimate when the next outage is likely. |

</details>

`Get-HotFix` does not reliably list .NET updates on all systems; if in doubt, the update history in Settings shows the complete status. Check the Exchange build in the Exchange Management Shell with `Get-ExchangeServer | Format-Table Name, AdminDisplayVersion` and compare it with the [Exchange build numbers](/tools/exchange-builds).

Then check the status of databases, copies, and queues in the Exchange Management Shell:

```powershell
Get-MailboxDatabaseCopyStatus -Server $env:COMPUTERNAME |
  Format-Table Name, Status, CopyQueueLength, ReplayQueueLength, ContentIndexState
Get-Queue | Where-Object MessageCount -gt 0 |
  Format-Table Identity, DeliveryType, Status, MessageCount, LastError
Get-WinEvent -FilterHashtable @{LogName='Application'; Id=1006; StartTime=(Get-Date).AddDays(-2)} |
  Select-Object TimeCreated, ProviderName, Message
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Get-MailboxDatabaseCopyStatus -Server $env:COMPUTERNAME` | Status of all database copies on the local server; without a DAG, only the active copy appears. |
| `CopyQueueLength`, `ReplayQueueLength` | Number of log files not yet copied or not yet replayed; increasing values indicate that replication has stopped. |
| `ContentIndexState` | Status of the copy’s search index. |
| `Get-Queue \| Where-Object MessageCount -gt 0` | Displays only queues that contain emails. |
| `LastError` | Last queue error; for this issue, typically connection errors to the mailbox database. |
| `-FilterHashtable @{LogName='Application'; Id=1006; …}` | Searches the Application Log for Event ID 1006, as specified by Microsoft. |
| `StartTime=(Get-Date).AddDays(-2)` | Limits the search to the last two days, i.e., the period since Patch Tuesday. |

</details>

Several Exchange components log Event ID 1006; the decisive entry is the one containing the MdbOffline message in the text.

## Procedure until a fix is available

The following points are from October 16, 2026, when no official workaround was available yet. Since October 17, 2026, the registry workaround from the update above comes first; points 3 through 6 remain valid as a fallback.

1. **Exchange servers not yet updated**: The original recommendation was to postpone the October Windows updates. With the registry workaround, this is no longer necessary: set the value and install the updates, which also close security vulnerabilities.

2. **Install the Exchange SU anyway**: According to Microsoft, the Exchange SU KB5128895 (or the ESU packages for 2016/2019) is not the trigger and closes a critical vulnerability that can be exploited by email without an account.

3. **Servers already affected**: A restart restores operation for about one day. Schedule controlled restarts at intervals well below 22 hours, for example every night, rather than waiting for the next outage. According to readers at Born, restarting only the Exchange services also helps temporarily; for how long is not documented.

4. **DAG**: Restart members one at a time and move active databases to a freshly restarted server beforehand. After each restart, use `Get-MailboxDatabaseCopyStatus` to verify that copies are `Healthy` again and queues are being cleared before restarting the next server.

5. **Monitor queues**: Keep an eye on `Get-Queue` and Event ID 1006, ideally with an alert so that an outage is not noticed only through user reports.

6. **Check backups**: A reader at Born reports a failed VSS Writer in the backup after the outage. Check backups from recent nights.

Regarding uninstalling Windows updates: since 2021, Microsoft has delivered the cumulative update together with the Servicing Stack Update as a combined package. `wusa /uninstall` does not remove such a package; `DISM /Online /Remove-Package` with the package name from `DISM /Online /Get-Packages` is required instead. Since October 17, 2026, Microsoft has explicitly advised against uninstalling; with the registry workaround, there is no reason to do so anymore.

Microsoft now lists the issue as a Known Issue in the Windows update release notes (as of October 19, 2026, unresolved). This article will be updated as soon as a fix is available.

## Sources

1.  [Released: October 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-october-2026-exchange-server-security-updates/4563014): Addendum from October 15, 2026 with the affected Windows updates, error message, Event ID 1006, and the statement that the Exchange SU is not required for the issue.

2.  [Exchange Server SE: Problems with the October updates for Windows Server – Frankys Web](https://www.frankysweb.de/en/exchange-server-se-problems-with-the-october-update-kb5128895/): Outage after about 24 hours, databases still shown as mounted in EAC, restart as a temporary workaround, comments on DAG restarts.

3.  [Warning: Exchange Server SE October 2026 SUs cause outages – Born’s IT and Windows Blog](https://borncity.com/blog/2026/10/15/achtung-exchange-server-okt-2026-su-fuehren-zu-ausfaellen/): Reader reports on timing, involved KBs (including .NET), replication service, maintenance mode, and VSS errors.

4.  [October updates put Exchange servers out of service – IT Magazine](https://www.itmagazine.ch/artikel/88263/Oktober-Updates_setzen_Exchange-Server_ausser_Betrieb.html): Swiss summary of Microsoft’s confirmation.

5.  [Exchange servers may become unstable after installation of October 2026 Windows updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/exchange-servers-may-become-unstable-after-installation-of-october-2026-windows-/4565382): Exchange team post from October 17, 2026 with the EpmapSavedContextCap registry value, the recommendation not to uninstall the Windows updates, and a note about workload-dependent effects.

6.  [October 13, 2026: KB5126646 (OS Build 26100.33650) – Microsoft Support](https://support.microsoft.com/en-us/servicing/os/windows-server/2026/10/kb5126646-windows-server-2025-security-update): Known Issue with the limit of 512 endpoint lookup calls, status unresolved, and an additional mitigation through Microsoft Support for business.

7.  [Microsoft analyzes Exchange Server issues caused by Windows October 2026 updates – Born’s IT and Windows Blog](https://borncity.com/blog/2026/10/17/microsoft-analysiert-exchange-server-probleme-durch-windows-oktober-2026-updates/): Reader analysis of RpcEpMap.dll (EpmapSavedContextCap, Feature ID), statement by Nino Bilic, experience with 65536 and higher values.

8.  [CVE-2026-96300 – OpenCVE](https://app.opencve.io/cve/CVE-2026-96300): RPC Endpoint Mapper Service Denial of Service Vulnerability, CWE-770, published on October 13, 2026.

9.  [Exchange security updates from October 2026 – rafaelpfister.ch](/blog/exchange-security-updates-oktober-2026): Contents of Exchange SU KB5128895 and the ESU packages, vulnerabilities, and installation notes.
