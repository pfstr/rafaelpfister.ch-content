---
title: "New Outlook reset after Windows update: accounts gone, cause in the log"
navTitle: "Outlook reset"
description: "After restarting for an update, the new Outlook for Windows starts as if freshly installed, with all accounts and settings missing. The cause can be traced in the Olk log: a server-side enabled check (Flight nh-userDataSentinel) cannot find a marker file and deletes local data without prompting. This article shows the log entries, how to verify the case on your own devices and what to do after the reset."
date: "2026-11-11"
kategorie: "Outlook"
timeToRead: "6 min read"
themen:
  - outlook
  - windows-client
produkte:
  - "outlook"
  - "windows-client"
protokolle:
  - "troubleshooting"
slug: "new-outlook-reset-after-windows-update-accounts-gone-cause-in-the-log"
translationId: "article-c1340c0bc1862647"
translationOf: outlook-new-reset-nach-update
url: https://rafaelpfister.ch/sv/blog/new-outlook-reset-after-windows-update-accounts-gone-cause-in-the-log
translationSourceHash: 405d93eb98233f046e7ecb2823969a601db0041bb6ddaef8eeaa4c7d858e393c
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T03:28:21.366Z
translationReview: automatic
---

# New Outlook reset after Windows update: accounts gone, cause in the log

After restarting for the Windows updates in November 2026, the new Outlook for Windows started on one of my devices as if it had just been installed. The business account configured as the primary account was missing, as were the other accounts, signatures and personal settings. There was no error message; Outlook immediately displayed the wizard for adding an account.

The cause is recorded in Outlook's local log: the app deleted its local data itself because a newly enabled check assessed the state as "reset required". There is no indication of intervention by Windows Update, a user or an attacker.

## Sequence

| Time (local time) | Event | Source |
|---|---|---|
| 18:26 | Last entry of the running Outlook session, no orderly shutdown | `Olk\logs\olk_*.log` |
| 18:27 and 18:28 | Two restarts by Windows Update (`MoNotificationUx.exe`, then `TrustedInstaller.exe`) | System log, event 1074 |
| 18:41:55 | Outlook starts, check `nh-userDataSentinel` is active and reports `ResetRequired` | new `olk_*.log` |
| 18:41:56 | Outlook deletes its local data and starts as a new installation | new `olk_*.log` |

The timestamps in the Olk log and in the log file names are in UTC. In the table, they have been converted to local time (CET).

## The log entries

The first lines of the new session show the check and its result:

```text
Checking flight enabled status  flightName nh-userDataSentinel  isEnabled true
Reset sentinel check completed.  installState ResetRequired
  settingsFound true  resetSentinelFlagFound true
Reset sentinel check triggered user data reset.
Prompted for clear local state  clearLocalStateName Silent
```

This is followed by the deletion operations, each with `succeeded true`:

| Path under `%LOCALAPPDATA%\Microsoft\Olk` | Contents |
|---|---|
| `EBWebView` | WebView2 profile with sign-in status, account list and offline data |
| `UserSettings.json` | App settings |
| `Attachments` | Locally cached attachments |
| `pst_index_v2` | Index of attached PST files |
| `cache\Notification`, `cache\*.db` | Notification and background cache |
| `updated.txt`, `xpdApi.log`, `Feedback` | Status and auxiliary files |

A few lines later, `Last update date file missing, creating it. Barring clearing local state, this is likely a fresh install` appears. From this point on, Outlook behaves like a new installation.

## Why the check was triggered

`nh-userDataSentinel` is a Flight, meaning a feature that Microsoft enables server-side for individual installations. In all older logs on the device, this Flight is listed as `Flight status not set; using default (false)`, most recently at 13:23 UTC on the same day. When it started at 17:41 UTC, it was active for the first time.

The check looks for a marker file in the app's package folder:

```text
%LOCALAPPDATA%\Packages\Microsoft.OutlookForWindows_8wekyb3d8bbwe\LocalState\OlkSentinel
```

If Outlook finds user settings in the Olk folder (`settingsFound true`) but no matching marker in the package folder, it treats this as an indication that the app has been reset or reinstalled and deletes the user data. On the affected device, `OlkSentinel` was newly created at 18:41:56, while the other files in `LocalState` date from September 2026. The package folder itself was therefore not reset; the marker file simply did not exist before because the check was new. On the first start with the Flight enabled, the result was therefore a reset even though none was needed.

According to the logs, the restart by Windows Update was only the timing, not the cause: the Flight is evaluated at startup, and the first start after the update was the first one with the Flight active. The app version (1.2026.1026.100) was the same before and after the restart.

## Verifying the case on your own devices

The following query searches all Olk logs for the entries from the check:

```powershell
$logs = "$env:LOCALAPPDATA\Microsoft\Olk\logs"
Get-ChildItem $logs -Filter 'olk_*.log' |
  Select-String -Pattern 'Reset sentinel|Delete directory tree' |
  Select-Object Filename, LineNumber, Line
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-Filter 'olk_*.log'` | Only the app's session logs, without ZIP archives and push logs |
| `-Pattern` | Regular expression; matches the check and every deletion operation |
| `Select-Object Filename, LineNumber, Line` | Shows the log file, line number and text for each match |

</details>

Outlook packages older sessions into ZIP archives in the same folder. If the reset happened several days ago, the entry may only be in the archive and must be extracted first.

Creation times are suitable as a second indication: if `EBWebView` and `UserSettings.json` in the Olk folder, as well as `OlkSentinel` in the package folder, have the same timestamp, the reset occurred at that time.

```powershell
$olk = "$env:LOCALAPPDATA\Microsoft\Olk"
$pkg = "$env:LOCALAPPDATA\Packages\Microsoft.OutlookForWindows_8wekyb3d8bbwe"
Get-Item "$olk\EBWebView", "$olk\UserSettings.json", "$pkg\LocalState\OlkSentinel" |
  Select-Object FullName, CreationTime
```

## What was lost and what was not

Only local app data was deleted. Mailboxes, calendars and contacts are stored on the server at Microsoft 365 and Outlook.com and will reappear after signing in again. The Windows sign-in (Web Account Manager) remains in place; Outlook directly offers Microsoft accounts saved in Windows when adding an account.

You need to set up again:

- all accounts, in the desired order (the first account added is the primary account; see below)
- locally stored signatures, unless they return through the account's cloud synchronisation
- attached PST files
- app settings such as view, notifications and startup behaviour

For the primary account, the order is crucial because the new Outlook offers some features only in the primary account, such as S/MIME (see [S/MIME signature in the secondary account](/blog/outlook-smime-sekundaeres-konto-anhaenge-fehlen)). Therefore, add the account that should be the primary account first. If Outlook has already automatically adopted another account after the reset, remove it, add the desired primary account and then add the other accounts again.

## For administrators

The reset cannot be undone after it has occurred; backing up the folder `%LOCALAPPDATA%\Microsoft\Olk` before startup is the only way to restore the old state. In environments with profile containers (FSLogix) and non-persistent VDI sessions, you should check whether both the Olk folder and the package folder `Microsoft.OutlookForWindows_8wekyb3d8bbwe` are retained in the profile. If only one of the two folders is retained, this corresponds exactly to the state that the check interprets as a new installation.

Reports from users that the new Outlook has "forgotten all accounts" can be classified within a few minutes using the log query above. If `Reset sentinel check triggered user data reset` appears there, it is not an account issue or a security incident.

## Källor

1.  [Microsoft Learn: Deploy the new Outlook for Windows on VDI](https://learn.microsoft.com/en-us/microsoft-365-apps/outlook/get-started/virtualized-desktop-infrastructure): locations of the new Outlook's local data and guidance on profile containers.

2.  [Microsoft Learn: Control installing and using the new Outlook](https://learn.microsoft.com/en-us/microsoft-365-apps/outlook/get-started/control-install): deployment and management of the app in managed environments.

3.  [Microsoft Q&A: Outlook (new) won't open, makes me attempt to sign in](https://learn.microsoft.com/en-us/answers/questions/5863503/outlook-new-wont-open-makes-me-attempt-to-sign-in): user reports of lost accounts after local data was reset.
