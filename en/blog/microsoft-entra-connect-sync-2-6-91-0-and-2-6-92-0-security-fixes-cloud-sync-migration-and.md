---
title: "Microsoft Entra Connect Sync 2.6.91.0 and 2.6.92.0: Security fixes, Cloud Sync migration, and mandatory upgrade by April 2027"
navTitle: "Microsoft Entra Connect Sync 2.6.92.0"
description: "Version 2.6.91.0 delivers security fixes, a guided move to Cloud Sync, and resolves the known issues in 2.6.84.0; hotfix 2.6.92.0 fixes Pass-through Authentication setup. By April 7, 2027, all servers must run at least 2.6.84.0 with Application-Based Authentication."
date: "2026-10-26"
kategorie: "Microsoft Entra"
timeToRead: "8 min read"
themen:
  - microsoft-entra
  - active-directory-entra
produkte:
  - "entra"
  - "entra-connect-releases"
protokolle:
  - "releases"
slug: "microsoft-entra-connect-sync-2-6-91-0-and-2-6-92-0-security-fixes-cloud-sync-migration-and"
translationId: article-59a51ef69f2e3538
translationOf: entra-connect-2-6-92-0
url: https://rafaelpfister.ch/en/blog/microsoft-entra-connect-sync-2-6-91-0-and-2-6-92-0-security-fixes-cloud-sync-migration-and
translationSourceHash: ff045b1bab72cea3839167d59c0a71010e80a3a20b1644acef89c79921f6f219
translationModel: gpt-5.6-terra
translatedAt: 2026-10-26T08:24:06.564Z
translationReview: required
---

# Microsoft Entra Connect Sync 2.6.91.0 and 2.6.92.0: Security fixes, Cloud Sync migration, and mandatory upgrade by April 2027

Microsoft released Entra Connect Sync 2.6.91.0 on September 16, 2026, and hotfix 2.6.92.0 on September 23, 2026, both available only as downloads through the Microsoft Entra admin center. Version 2.6.91.0 fixes security vulnerabilities in bundled third-party components, makes phishing-resistant sign-in in the setup wizard generally available, adds a guided migration workflow to Microsoft Entra Cloud Sync, and resolves the three known issues in 2.6.84.0. Version 2.6.92.0 fixes a 2.6.91.0 issue when setting up Pass-through Authentication. At the same time, Microsoft specifies a firm deadline: by April 7, 2027, all servers must run at least 2.6.84.0 and use Application-Based Authentication, or synchronization will stop. This article is intended for administrators running Entra Connect Sync. The previous status is described in the article on [Entra Connect Sync 2.6.84.0](/blog/entra-connect-2-6-84-0).

## Versions at a glance

| Version | Date | Type | Key points |
|---|---|---|---|
| 2.6.91.0 | September 16, 2026 | Feature and security release (download) | Security fixes in third-party components, Cloud Sync migration assistant, passkey sign-in in the wizard GA, new Microsoft Graph permissions, fixes for the known issues in 2.6.84.0 |
| 2.6.92.0 | September 23, 2026 | Hotfix with security fixes (download) | Fixes failed registration of the Authentication Agent when enabling Pass-through Authentication |

Microsoft classifies 2.6.92.0 in the release notes as a release with security fixes and recommends updating “as soon as possible.” No auto-upgrade rollout has been announced for either version (as of October 26, 2026). Version 2.6.84.0 also remains available only as a download.

## Mandatory upgrade by April 7, 2027

The version history and upgrade guidance now carry the same notice: Entra Connect Sync must be updated to version 2.6.84.0 or later and switched to Application-Based Authentication (ABA) by April 7, 2027. Microsoft is retiring legacy authentication through the Directory Synchronization Account; servers that do not meet these requirements will no longer synchronize after that date. If synchronization stops, Microsoft states that upgrading to the current version and then configuring ABA restores operation.

For planning, a behavior change from 2.6.84.0 is crucial: since that version, Entra Connect no longer automatically switches existing servers to ABA. Only new installations configure ABA automatically. An upgrade alone therefore does not meet the deadline. You must switch existing servers in the wizard through **Additional tasks** > **Configure application-based authentication to Microsoft Entra ID**.

The wizard shows the method currently used by a server under **View or export current configuration**. Microsoft also provides this cmdlet:

```powershell
Get-ADSyncEntraConnectorCredential
```

The return value `ConnectorIdentityType` is `ServiceAccount` (legacy account) or `Application` (ABA). If it shows `ServiceAccount`, the switch is required before April 7, 2027. After a successful switch, Microsoft recommends removing the old Directory Synchronization Account from Entra ID.

## Version 2.6.91.0

### Security fixes

The release notes list fixed security vulnerabilities in bundled third-party components, hardened permissions for local temporary files, and secured downloads of profile pictures in the SharePoint connector. Microsoft does not provide CVE numbers, CVSS scores, or affected components. It is therefore not possible to assess the severity based on public sources.

### New Microsoft Graph permissions and Conditional Access

Entra Connect receives additional Microsoft Graph permissions. Microsoft explicitly notes that anyone using app-based Conditional Access policies should review policies targeting `Microsoft.Azure.SyncFabric` or the Microsoft 365 Reporting Service. Review these policies before upgrading so that the new access is not blocked.

### Guided migration to Cloud Sync

A guided workflow for moving from Entra Connect Sync to Microsoft Entra Cloud Sync is new. It includes an assessment of the existing configuration, setup of the Provisioning Agent, staged activation, and validation. The feature is available only in the Azure Public Cloud. Microsoft's upgrade guidance recommends checking before an upgrade whether Cloud Sync is suitable for your environment anyway.

### Phishing-resistant sign-in in the wizard generally available

Sign-in through the Windows Web Account Manager, introduced as a preview in 2.6.84.0, is now generally available and enabled by default. The dialog supports passkeys, FIDO2 security keys, and passwords; reuses an existing sign-in session for additional Entra services; and maintains Kerberos key rotation for Seamless Single Sign-On.

### Changed PowerShell cmdlets

Version 2.6.84.0 had made the `Set-ADSyncAADCompanyFeature` and `Set-ADSyncAADPasswordSyncState` parameter `-AADUsername` mandatory. With 2.6.91.0, the parameter is optional again: if it is missing, Entra Connect derives a sign-in hint from the connector configuration and opens an interactive sign-in. This applies to `Set-ADSyncAADCompanyFeature`, `Set-ADSyncAADPasswordSyncState`, `Enable-ADSyncExportDeletionThreshold`, `Set-ADSyncScheduler`, and `Set-ADSyncDirSyncConfiguration`. Interactive sign-in therefore remains part of these cmdlets; you should test scripts that call them unattended after the upgrade.

Anyone configuring Seamless Single Sign-On with the standalone PowerShell module must import `ADSync.psd1` before `AzureADSSO.psd1`.

### Additional changes

- The **Select Containers** dialog in Synchronization Service Manager is now read-only. Changes to container selection are made through **Customize synchronization options** in the wizard.
- Support for an additional Sovereign Cloud environment, including Pass-through Authentication, Seamless Single Sign-On, Password Writeback, and Health Agent monitoring. The release notes do not specify which environment is meant.
- The bundled SQL Server 2022 LocalDB was updated from 16.0.4250.1 to 16.0.4262.2.

### Bug fixes

- Installation or upgrade using an existing ADSync database could fail with error `0xE0474352` (a known issue in 2.6.84.0).
- An upgrade did not update an installed Visual C++ 2015–2022 Redistributable older than 14.41.
- The wizard could close at startup if the Windows PowerShell Transcription policy contained an invalid output directory.
- Application-Based Authentication certificate rotation could remove the last usable Application Key when directory replication was delayed.
- Synchronization Service Manager could close when listing directory partitions in **Connector Properties** (a known issue in 2.6.84.0).
- **Change Credentials** did not open for connector rows outside the visible area when many Active Directory connectors were configured.
- When expanding a previously deselected domain on the **Domain and OU filtering** page, the wizard could select the domain again and enable synchronization for the entire domain upon completion (a known issue in 2.6.84.0).

The last point deserves attention: if you run 2.6.84.0 and ran through the wizard again after the upgrade, you should check domain and OU filtering. An unintentionally enabled domain results in additional objects in Entra ID.

## Version 2.6.92.0

The hotfix resolves a 2.6.91.0 issue: enabling Pass-through Authentication through the wizard could fail while registering the locally installed Microsoft Entra Connect Authentication Agent. The release notes list no further changes. Microsoft marks 2.6.92.0 as a release with security fixes; individual vulnerabilities are not listed.

## Known issues

| Issue | Affected version | Status |
|---|---|---|
| Installation or upgrade with an existing ADSync database fails with `0xE0474352` | 2.6.84.0 | Fixed in 2.6.91.0 |
| Synchronization Service Manager closes in **Connector Properties** | 2.6.84.0 | Fixed in 2.6.91.0 |
| Wizard selects a deselected domain again | 2.6.84.0 | Fixed in 2.6.91.0 |
| PTA activation in the wizard fails during agent registration | 2.6.91.0 | Fixed in 2.6.92.0 |
| Synchronization fails after an upgrade (`System.IO.FileLoadException`) if `miiserver.exe.config` was modified previously | 2.5.190.0, 2.6.1.0 | Still listed in the version history; workaround via manual bindingRedirect, see the [article on 2.6.84.0](/blog/entra-connect-2-6-84-0) |

The withdrawn version 2.6.79.0 remains unavailable. Microsoft now directs affected users to 2.6.92.0: uninstall and install the current version.

## Support deadlines

Each 2.x version falls out of support twelve months after the release of its successor. The two new versions result in these deadlines:

| Version | Released | End of support |
|---|---|---|
| 2.5.79.0 | September 1, 2025 | October 23, 2026 (expired) |
| 2.5.190.0 | November 19, 2025 | February 2, 2027 |
| 2.6.1.0 | February 2, 2026 | March 10, 2027 |
| 2.6.3.0 | March 10, 2026 | July 7, 2027 |
| 2.6.84.0 | July 7, 2026 | September 16, 2027 |
| 2.6.91.0 | September 16, 2026 | September 23, 2027 |
| 2.6.92.0 | September 23, 2026 | pending |

Versions 2.5.190.0 through 2.6.3.0 are also subject to the April 7, 2027 deadline: they are below the required minimum version of 2.6.84.0 and will no longer synchronize after that date, even though regular support for 2.6.3.0 does not end until July 2027. Version 2.5.79.0 has been unsupported since October 23, 2026; according to Microsoft, unsupported versions can unexpectedly stop working.

## Recommended approach

The target version is 2.6.92.0. It includes the security fixes from both releases, resolves all known issues in 2.6.84.0 and 2.6.91.0, and meets the minimum version for the April 7, 2027 deadline. The installer is available only in the Microsoft Entra admin center under **Microsoft Entra Connect | Get started**, on the **Manage** tab; prerequisites are .NET Framework 4.7.2 and TLS 1.2. Before upgrading, check app-based Conditional Access policies for `Microsoft.Azure.SyncFabric` and the Microsoft 365 Reporting Service, as well as scripts that call the listed cloud configuration cmdlets. For low-risk upgrades, Microsoft recommends Swing Migration: first update the staging server, verify the configuration after Full Import and Full Synchronization, then switch roles and update the previously active server. According to Microsoft, anyone who has not updated for 12 to 18 months should use this approach. After the upgrade, switch existing servers that still report `ServiceAccount` to Application-Based Authentication, and check domain and OU filtering on servers that previously ran 2.6.84.0.

## Sources

1.  [Microsoft Entra Connect: Version release history](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-version-history): Release notes for 2.6.91.0 and 2.6.92.0, known issues in 2.6.84.0, retirement table, and the April 7, 2027 deadline.

2.  [Microsoft Entra Connect: Upgrade from a previous version](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-upgrade-previous-version): Upgrade methods, Swing Migration through a staging server, and the mandatory upgrade notice.

3.  [Authenticate to Microsoft Entra ID by using application identity](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/authenticate-application-id): Manual switch of existing servers to ABA, `Get-ADSyncEntraConnectorCredential`, and removal of the legacy account.
