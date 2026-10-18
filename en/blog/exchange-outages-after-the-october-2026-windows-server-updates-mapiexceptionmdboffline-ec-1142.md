---
title: "Exchange Outages After the October 2026 Windows Server Updates: MapiExceptionMdbOffline (ec=1142)"
navTitle: "Exchange Outage 10/2026"
description: "After the Windows Server updates of October 13, 2026, Exchange servers report MapiExceptionMdbOffline (ec=1142) after around 24 hours: OWA fails, queues grow, and DAG copies stop replicating. Microsoft confirms the link to the Windows updates, but a fix is not yet available. Symptoms, affected KBs, verification commands, and recommended steps."
date: "2026-10-16"
kategorie: "Exchange On-Premises / Hybrid"
timeToRead: "7 min read"
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
url: https://rafaelpfister.ch/en/blog/exchange-outages-after-the-october-2026-windows-server-updates-mapiexceptionmdboffline-ec-1142
translationSourceHash: 562adaae54d08dd594a344f91f79a05160af956ab320978cef34097a075cab52
translationModel: gpt-5.6-terra
translatedAt: 2026-10-16T08:18:01.506Z
translationReview: automatic
---

# Exchange Outages After the October 2026 Windows Server Updates: MapiExceptionMdbOffline (ec=1142)

Since Patch Tuesday on October 13, 2026, Exchange servers have been failing after around 24 hours of uptime: clients lose their mailbox connection, OWA reports `MapiExceptionMdbOffline: Unable to make connection to the server. (hr=0x80004005, ec=1142)`, emails remain in queues, and database copies in DAGs no longer replicate. Microsoft added a note to the announcement of the October Exchange SUs on October 15, 2026: the October updates for Windows Server are the cause, and installing the Exchange SU is not required for the issue to occur. No fix or official workaround is available yet (as of October 16, 2026).

## Symptoms

The symptoms reported so far:

- OWA fails with `MapiExceptionMdbOffline: Unable to make connection to the server. (hr=0x80004005, ec=1142)`; Event ID 1006 is recorded in the Application Log.
- Outlook clients lose their mailbox connection.
- Emails are delivered neither internally nor externally, and transport queues grow.
- In DAGs, database copies no longer replicate; according to reader reports at Born, switchovers when entering maintenance mode fail.
- The Exchange Admin Center still shows the databases as mounted (Frankys Web).

The error does not occur immediately after installation, but according to reports after approximately 22 to 30 hours of uptime. Restarting the server restores operation, but the outage recurs after approximately the same period. Comments at Born note that the mailbox replication service stops during or after database maintenance tasks, after which the MdbOffline errors begin; this has not been confirmed.

## Affected updates

In the addendum to the Exchange announcement, Microsoft lists these cumulative Windows updates:

| Windows Server | Update |
| --- | --- |
| Windows Server 2025 | KB5126646 |
| Windows Server 2022 | KB5126655 |
| Windows Server 2019 | KB5126651 |
| Windows Server 2016 | KB5126868 |

The same Patch Tuesday also saw the release of the Exchange SUs (Exchange SE: KB5128895, “SU11”) and cumulative updates for the .NET Framework. The following three packages were installed on an affected Exchange SE server running Windows Server 2019:

- Security Update For Exchange Server Subscription Edition (KB5128895)
- 2026-10 Cumulative Update for .NET Framework 3.5, 4.7.2 and 4.8 for Windows Server 2019 for x64 (KB5128923)
- 2026-10 Cumulative Update for Windows Server 2019 for x64-based Systems (KB5126651)

Microsoft does not mention the .NET update in the addendum. However, readers at Born report outages with the combination of the Windows and .NET updates (KB5128928 there on Windows Server 2022), as well as a server with KB5126651 and the Exchange SU but without the .NET update that was still running after more than 24 hours. Whether the .NET update is involved therefore remains unclear.

Regarding the Exchange SU: forums and comments state that the situation improves when Exchange SU11 is installed. This cannot be confirmed; outages also occurred on a server with SU11. Conversely, several administrators report outages without an installed Exchange SU, which is consistent with Microsoft’s statement. The Exchange SU closes six vulnerabilities, including a critical remote code execution vulnerability through a crafted email; details are in the [article on the October SU](/blog/exchange-security-updates-oktober-2026). Withholding it does not resolve the issue.

## Check the status

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
| `$kbs`, `-Id` | List of KB numbers to look for: the four Windows CUs from the Microsoft addendum and the .NET update for Windows Server 2019. |
| `-ErrorAction SilentlyContinue` | Suppresses the error message for KB numbers that are not installed on this server. |
| `Select-Object PSComputerName, HotFixID, InstalledOn` | Displays server, KB number, and installation date; the date can be used to estimate when the next outage is expected. |

</details>

`Get-HotFix` does not reliably list .NET updates on all systems; when in doubt, the update history in Settings shows the complete status. Check the Exchange build in the Exchange Management Shell with `Get-ExchangeServer | Format-Table Name, AdminDisplayVersion` and compare it with the [Exchange build numbers](/tools/exchange-builds).

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
| `CopyQueueLength`, `ReplayQueueLength` | Number of log files not yet copied or replayed; increasing values indicate that replication has stopped. |
| `ContentIndexState` | Status of the copy’s search index. |
| `Get-Queue \| Where-Object MessageCount -gt 0` | Shows only queues that contain emails. |
| `LastError` | Last queue error; for this issue, typically connection errors to the mailbox database. |
| `-FilterHashtable @{LogName='Application'; Id=1006; …}` | Searches the Application Log for Event ID 1006, as identified by Microsoft. |
| `StartTime=(Get-Date).AddDays(-2)` | Limits the search to the last two days, meaning the period since Patch Tuesday. |

</details>

Several Exchange components log Event ID 1006; the decisive entry is the one containing the MdbOffline message in its text.

## What to do until a fix is available

Microsoft says it is investigating the issue together with the Windows team and has not yet provided either a workaround or rollback. In the meantime:

1. **Exchange servers not yet updated**: Defer the October Windows updates (and the .NET update from the same day) on Exchange servers in WSUS, Intune, or your own update management system. This requires weighing the risks: the Windows updates also close security vulnerabilities. Exchange servers accessible from the Internet should therefore be monitored particularly closely.

2. **Install the Exchange SU anyway**: According to Microsoft, Exchange SU KB5128895 (or the ESU packages for 2016/2019) is not the cause and closes a critical vulnerability that can be exploited by email without an account.

3. **Servers already affected**: A restart restores operation for around one day. Schedule controlled restarts at intervals well below 22 hours, for example every night, rather than waiting for the next outage. According to readers at Born, restarting only the Exchange services also helps temporarily; how long it lasts is unconfirmed.

4. **DAG**: Restart members one at a time and move active databases to a freshly restarted server first. After each restart, use `Get-MailboxDatabaseCopyStatus` to verify that copies are `Healthy` again and queues are draining before restarting the next server.

5. **Monitor queues**: Keep an eye on `Get-Queue` and Event ID 1006, ideally with alerting so that an outage is not discovered only through user reports.

6. **Check backups**: A reader at Born reported a failed VSS writer in the backup after the outage. Check backups from the last few nights.

For uninstalling the Windows updates: since 2021, Microsoft has delivered the cumulative update together with the Servicing Stack Update as a combined package. `wusa /uninstall` does not remove such a package; `DISM /Online /Remove-Package` with the package name from `DISM /Online /Get-Packages` is required. Microsoft has not approved uninstallation as a workaround, and no reports of a successful rollback followed by stable operation are available so far. If you test this at all, test it on a single server first.

According to Frankys Web, Microsoft does not yet list the issue as a Known Issue in the Windows updates (as of October 16, 2026). Keep an eye on the Microsoft announcement; Microsoft posts additions there at the end under “Updates to this blog post.” This article will be updated as soon as a fix or official workaround is available.

## Sources

1.  [Released: October 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-october-2026-exchange-server-security-updates/4563014): Addendum of October 15, 2026, with the affected Windows updates, error message, Event ID 1006, and the statement that the Exchange SU is not required for the issue.

2.  [Exchange Server SE: Problems with the October updates for Windows Server – Frankys Web](https://www.frankysweb.de/en/exchange-server-se-problems-with-the-october-update-kb5128895/): Outage after approximately 24 hours, databases still shown as mounted in the EAC, restart as a temporary remedy, and comments about DAG restarts.

3.  [Warning: Exchange Server SE October 2026 SUs cause outages – Born’s IT and Windows Blog](https://borncity.com/blog/2026/10/15/achtung-exchange-server-okt-2026-su-fuehren-zu-ausfaellen/): Reader reports on timing, involved KBs (including .NET), the replication service, maintenance mode, and VSS errors.

4.  [October updates put Exchange servers out of service – IT Magazine](https://www.itmagazine.ch/artikel/88263/Oktober-Updates_setzen_Exchange-Server_ausser_Betrieb.html): Swiss summary of Microsoft’s confirmation.

5.  [Exchange Security Updates for October 2026 – rafaelpfister.ch](/blog/exchange-security-updates-oktober-2026): Contents of Exchange SU KB5128895 and the ESU packages, vulnerabilities, and installation notes.
