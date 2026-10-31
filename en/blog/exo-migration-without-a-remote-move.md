---
title: "EXO Migration Without a Remote Move"
navTitle: "EXO Migration Without a Remote Move"
description: "How to provision on-premises Exchange mailboxes in a controlled manner as new, empty Exchange Online mailboxes: PST backup, CSV approval, RemoteMailbox, synchronization, validation, and rollback."
date: "2026-08-07"
kategorie: "Exchange On-Premises / Hybrid"
timeToRead: "12 min read"
themen:
  - exchange-onprem-hybrid
  - microsoft-365-exchange
  - active-directory-entra
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange"
protokolle:
  - "powershell"
  - "migration"
related:
  - typische-ursachen-fuer-mail-loops-und-deren-behebung
  - totemomail-m365
  - ghost-sender-exchange-online-nebeneingang
slug: "exo-migration-without-a-remote-move"
translationId: "article-8f3c1b7a62d94e50"
draft: false
translationOf: exchange-mailboxen-ohne-remote-move-cloud-neu-erstellen
translationSourceHash: 8ef7108bef25cbfdabbdd58d6ed17a1cc34ba2da65eb7e685180b0976f99d62c
translationModel: gpt-5.6-terra
translatedAt: 2026-10-30T03:31:55.024Z
translationReview: required
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/en/blog/exo-migration-without-a-remote-move
---

# EXO Migration Without a Remote Move

A Hybrid Remote Move is the standard way to move an on-premises Exchange mailbox, including its contents, to Exchange Online. Not every organization permits this migration path. If a security policy excludes Remote Moves, a deliberately different approach may be justifiable: The on-premises mailbox is backed up as a PST, disconnected from the synchronized AD user, and a new, empty mailbox is provisioned in Exchange Online for that same user.

This approach is **not a mailbox migration**. It transfers neither messages nor calendars, rules, or permissions to the cloud. The PST serves solely as a backup and is not imported in this scenario. This procedure is therefore suitable only if an empty target mailbox is acceptable from a business perspective and the loss of the active mailbox configuration has been explicitly approved.

## Target State and Strict Prerequisites

After cutover, the same AD user remains in place. On-premises, however, it is no longer managed as `UserMailbox`, `SharedMailbox`, `RoomMailbox`, or `EquipmentMailbox`, but as `RemoteMailbox`. After synchronization, this object represents the new mailbox in Exchange Online.

The desired state looks like this:

1. The on-premises mailbox is fully backed up as a PST.
2. The on-premises mailbox is disconnected but has not yet been permanently deleted within the configured retention period.
3. The existing AD user is enabled as a RemoteMailbox.
4. The primary address, aliases, and the old `LegacyExchangeDN` are retained.
5. Entra Connect has synchronized the changes.
6. An Exchange Online service plan is assigned for user mailboxes.
7. Exchange Online shows a real cloud mailbox, and mail flow ends there.

The following must also be clarified before starting:

- The PST share is accessible via UNC. The `Exchange Trusted Subsystem` group has read and write permissions there.
- The executing account has the `Mailbox Import Export` management role.
- The PST is only the agreed backup; no later import is planned.
- Retention, Litigation Hold, eDiscovery, and regulatory requirements have been reviewed separately.
- Delegations, Send As, Send on Behalf, forwarding, inbox rules, mobile devices, and application access have been inventoried.
- During export and cutover, incoming messages are held in a controlled manner at the upstream gateway. Users and applications must no longer write to the source mailbox.
- The retention period of the on-premises mailbox database covers the rollback window.

## Why a CSV Approval List Is Essential

A direct pipeline such as `Get-Mailbox | Disable-Mailbox` is too risky for this process. It could also include system, discovery, or otherwise unapproved mailboxes. The following procedure therefore uses two explicit approvals:

- `Action=CUTOVER` determines which row may actually be switched.
- `PstVerified=YES` confirms that the export file has been reviewed technically and organizationally.

First, only the inventory is created:

```powershell
$CsvPath = "C:\Migration\mailboxes.csv"
$RemoteRoutingDomain = "contoso.mail.onmicrosoft.com"

Get-Mailbox -ResultSize Unlimited -RecipientTypeDetails `
    UserMailbox,SharedMailbox,RoomMailbox,EquipmentMailbox |
    Sort-Object PrimarySmtpAddress |
    ForEach-Object {
        [pscustomobject]@{
            Identity             = $_.Identity
            DisplayName          = $_.DisplayName
            PrimarySmtpAddress   = $_.PrimarySmtpAddress.ToString()
            Alias                = $_.Alias
            SourceType           = $_.RecipientTypeDetails.ToString()
            ArchiveStatus        = $_.ArchiveStatus.ToString()
            ServerName           = $_.ServerName
            Database             = $_.Database.ToString()
            RemoteRoutingAddress = "$($_.Alias)@$RemoteRoutingDomain"
            Action               = "REVIEW"
            PstVerified          = "NO"
        }
    } |
    Export-Csv -Path $CsvPath -NoTypeInformation -Encoding UTF8
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Get-Mailbox -ResultSize Unlimited` | Removes the default limit of 1,000 results; without this parameter, mailboxes are missing from the inventory in large environments |
| `-RecipientTypeDetails UserMailbox,SharedMailbox,RoomMailbox,EquipmentMailbox` | Limits the query to the four mailbox types to be switched; system and discovery mailboxes are excluded |
| `Sort-Object PrimarySmtpAddress` | Sorts the output by primary SMTP address so that the CSV file remains consistently ordered during business review |
| `Export-Csv -Path` | Target path of the CSV file |
| `-NoTypeInformation` | Suppresses the type header `#TYPE ...`, which older PowerShell versions would otherwise write as the first line |
| `-Encoding UTF8` | Writes the file using UTF-8 encoding so that umlauts in display names are retained correctly |

</details>

The file is then reviewed and cleaned up from a business perspective. Only mailboxes that have actually been approved receive `Action=CUTOVER`. System mailboxes and special objects do not belong on this list.

## Phase 1: Back Up the Primary Mailbox and Archive as PST Files

`New-MailboxExportRequest` writes only to a UNC path. A unique file name is generated for each mailbox. An active on-premises Exchange online archive is exported separately:

```powershell
$CsvPath = "C:\Migration\mailboxes.csv"
$PstShare = "\\fileserver\exchange-pst$"
$BatchName = "EXO-NewMailbox-20260807"

$targets = Import-Csv $CsvPath | Where-Object Action -eq "CUTOVER"

foreach ($row in $targets) {
    $safeName = ($row.PrimarySmtpAddress -replace '[^a-zA-Z0-9@._-]', '_')
    $primaryPath = "$PstShare\$safeName.pst"

    New-MailboxExportRequest `
        -Mailbox $row.Identity `
        -FilePath $primaryPath `
        -Name "Primary-$safeName" `
        -BatchName $BatchName

    if ($row.ArchiveStatus -eq "Active") {
        New-MailboxExportRequest `
            -Mailbox $row.Identity `
            -IsArchive `
            -FilePath "$PstShare\$safeName-archive.pst" `
            -Name "Archive-$safeName" `
            -BatchName $BatchName
    }
}
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Import-Csv $CsvPath` | Reads the approval list; each row becomes an object with the CSV columns as properties |
| `Where-Object Action -eq "CUTOVER"` | Processes only explicitly approved rows |
| `New-MailboxExportRequest -Mailbox` | Source mailbox of the export, here the identity from the CSV row |
| `-FilePath` | Target path of the PST file; it must be a UNC path, as the cmdlet rejects local paths |
| `-Name` | Unique request name; allows the primary and archive exports to be assigned specifically later |
| `-BatchName` | Groups all requests from one run under a batch name; the basis for status queries and cleanup |
| `-IsArchive` | Exports the online archive instead of the primary mailbox; hence the second request for each mailbox with an active archive |

</details>

The export is not approved until **every** request has the status `Completed`:

```powershell
Get-MailboxExportRequest -BatchName $BatchName |
    Get-MailboxExportRequestStatistics -IncludeReport |
    Format-Table DisplayName,Status,PercentComplete,FailureCode,Message -AutoSize

Get-MailboxExportRequest -BatchName $BatchName |
    Where-Object Status -ne "Completed"
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Get-MailboxExportRequest -BatchName` | Lists all export requests in the specified batch |
| `Get-MailboxExportRequestStatistics -IncludeReport` | Adds the detailed history report to the statistics, containing error causes for individual requests |
| `Format-Table ... -AutoSize` | Displays the specified properties in tabular form; `-AutoSize` adjusts column widths to the content |
| `Where-Object Status -ne "Completed"` | Filters for all requests that are not yet complete or have failed; the output must be empty before proceeding |

</details>

In addition, the files' existence, size, readability, backup transfer, and access protection must be checked. Only then is `PstVerified=YES` set for the respective CSV row.

## Phase 2: Back Up Mailbox Data and Exchange Attributes

Before the first change, a machine-readable snapshot is created for each mailbox. It is more important than a screenshot because aliases, GUIDs, and the `LegacyExchangeDN` can later be reconstructed exactly:

```powershell
$SnapshotPath = "C:\Migration\Snapshots"
New-Item -ItemType Directory -Path $SnapshotPath -Force | Out-Null

Import-Csv "C:\Migration\mailboxes.csv" |
    Where-Object { $_.Action -eq "CUTOVER" -and $_.PstVerified -eq "YES" } |
    ForEach-Object {
        $mailbox = Get-Mailbox -Identity $_.Identity
        $safeName = ($_.PrimarySmtpAddress -replace '[^a-zA-Z0-9@._-]', '_')

        $mailbox |
            Select-Object Identity,DistinguishedName,ExchangeGuid,ArchiveGuid,
                RecipientTypeDetails,PrimarySmtpAddress,EmailAddresses,
                LegacyExchangeDN,Alias,Database,ServerName |
            Export-Clixml "$SnapshotPath\$safeName.xml"
    }
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `New-Item -ItemType Directory -Path ... -Force` | Creates the snapshot directory; `-Force` suppresses the error if it already exists |
| `Get-Mailbox -Identity` | Retrieves the current mailbox object for the respective CSV row |
| `Select-Object Identity,...,ServerName` | Reduces the object to the attributes required for later reconstruction, including GUIDs, addresses, `LegacyExchangeDN`, and database |
| `Export-Clixml` | Serializes the object as type-preserving CLIXML; unlike CSV, multivalued properties such as `EmailAddresses` are fully retained and can be read again with `Import-Clixml` |

</details>

Delegations and forwarding require separate exports. At a minimum, the following information should be backed up separately:

```powershell
$mailbox = Get-Mailbox -Identity user01@contoso.com

$mailbox | Format-List ForwardingAddress,ForwardingSmtpAddress,DeliverToMailboxAndForward
Get-MailboxPermission -Identity $mailbox.Identity
Get-ADPermission -Identity $mailbox.DistinguishedName
Get-InboxRule -Mailbox $mailbox.Identity
Get-CalendarProcessing -Identity $mailbox.Identity -ErrorAction SilentlyContinue
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Format-List ForwardingAddress,ForwardingSmtpAddress,DeliverToMailboxAndForward` | Displays the mailbox's three forwarding attributes in list format |
| `Get-MailboxPermission -Identity` | Lists mailbox permissions such as Full Access |
| `Get-ADPermission -Identity` | Lists AD permissions on the user object, including Send As; expects the Distinguished Name here |
| `Get-InboxRule -Mailbox` | Lists the mailbox's server-side inbox rules |
| `Get-CalendarProcessing -Identity` | Displays booking configuration; relevant for room and equipment mailboxes |
| `-ErrorAction SilentlyContinue` | Suppresses the error for mailbox types without booking configuration so that the backup does not stop |

</details>

These configurations do not automatically transfer to the new cloud mailbox.

## Phase 3: Disconnect the On-Premises Mailbox and Enable RemoteMailbox

The actual cutover is brief but consequential. `Disable-Mailbox` removes the Exchange attributes from the AD user and disconnects the on-premises mailbox. Mailbox data remains available as a disconnected mailbox until the database retention period expires. Immediately afterward, `Enable-RemoteMailbox` enables the same AD user for Exchange Online.

The following script processes only doubly approved rows. It preserves the primary SMTP address, all existing proxy addresses, and the old `LegacyExchangeDN` as an X500 address. The X500 entry prevents NDRs when replying to older messages or using old Outlook AutoComplete entries.

```powershell
$CsvPath = "C:\Migration\mailboxes.csv"
$PstShare = "\\fileserver\exchange-pst$"

$targets = Import-Csv $CsvPath |
    Where-Object { $_.Action -eq "CUTOVER" -and $_.PstVerified -eq "YES" }

foreach ($row in $targets) {
    $safeName = ($row.PrimarySmtpAddress -replace '[^a-zA-Z0-9@._-]', '_')
    $pstPath = "$PstShare\$safeName.pst"

    if (-not (Test-Path $pstPath)) {
        throw "PST fehlt: $pstPath"
    }

    if ((Get-Item $pstPath).Length -eq 0) {
        throw "PST ist leer: $pstPath"
    }

    $mailbox = Get-Mailbox -Identity $row.Identity -ErrorAction Stop
    $primary = $mailbox.PrimarySmtpAddress.ToString()
    $legacyDn = $mailbox.LegacyExchangeDN

    if ($primary -ine $row.PrimarySmtpAddress) {
        throw "Primaere SMTP-Adresse stimmt nicht mit der Freigabeliste ueberein: $($row.Identity)"
    }

    $preservedAddresses = @($mailbox.EmailAddresses | ForEach-Object ToString)

    Disable-Mailbox -Identity $row.Identity -Confirm:$false

    $enableParams = @{
        Identity             = $row.Identity
        Alias                = $row.Alias
        PrimarySmtpAddress   = $primary
        RemoteRoutingAddress = $row.RemoteRoutingAddress
        ACLableSyncedObjectEnabled = $true
    }

    switch ($row.SourceType) {
        "SharedMailbox"    { $enableParams.Shared = $true }
        "RoomMailbox"      { $enableParams.Room = $true }
        "EquipmentMailbox" { $enableParams.Equipment = $true }
        "UserMailbox"      { }
        default             { throw "Nicht unterstuetzter Mailbox-Typ: $($row.SourceType)" }
    }

    Enable-RemoteMailbox @enableParams

    $orderedAddresses = @(
        "SMTP:$primary"
        $preservedAddresses |
            Where-Object { $_ -notmatch '^SMTP:' -or $_.Substring(5) -ine $primary }
        "smtp:$($row.RemoteRoutingAddress)"
        "X500:$legacyDn"
    )

    $seen = [System.Collections.Generic.HashSet[string]]::new(
        [System.StringComparer]::OrdinalIgnoreCase
    )
    $finalAddresses = foreach ($address in $orderedAddresses) {
        if ($seen.Add($address)) { $address }
    }

    Set-RemoteMailbox -Identity $row.Identity -EmailAddressPolicyEnabled $false
    Set-RemoteMailbox -Identity $row.Identity -EmailAddresses $finalAddresses
    Set-RemoteMailbox -Identity $row.Identity -PrimarySmtpAddress $primary

    Get-RemoteMailbox -Identity $row.Identity |
        Format-List DisplayName,RecipientTypeDetails,PrimarySmtpAddress,
            RemoteRoutingAddress,EmailAddresses
}
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Get-Mailbox -Identity ... -ErrorAction Stop` | Retrieves the source mailbox; `-ErrorAction Stop` makes a lookup error terminating rather than silently continuing |
| `Disable-Mailbox -Identity` | Removes the Exchange attributes from the AD user and disconnects the on-premises mailbox; the data remains in the database as a disconnected mailbox |
| `-Confirm:$false` | Suppresses the interactive prompt; approval is provided here through the CSV list, not at the prompt |
| `Enable-RemoteMailbox -Identity` | Enables the same AD user as a RemoteMailbox for Exchange Online |
| `-Alias` | Restores the Exchange alias to the value from the approval list |
| `-PrimarySmtpAddress` | Retains the previous primary SMTP address |
| `-RemoteRoutingAddress` | Target address in the `mail.onmicrosoft.com` routing domain through which on-premises Exchange reaches the cloud mailbox |
| `-ACLableSyncedObjectEnabled` | Marks the object as ACL-capable so that permissions such as Full Access remain evaluable in Exchange Online after synchronization |
| `-Shared` / `-Room` / `-Equipment` | Creates the respective special type instead of a user mailbox; the script sets exactly one switch matching the source type |
| `Set-RemoteMailbox -EmailAddressPolicyEnabled $false` | Excludes the object from the email address policy so that it does not overwrite manually configured addresses |
| `-EmailAddresses` | Sets the complete deduplicated address list, including old proxy addresses, routing address, and X500 entry |
| `Get-RemoteMailbox -Identity` | Queries the result for verification immediately after the change |

</details>

The script is deliberately not a fully automated migration tool. It stops at the first discrepancy so that an administrator can assess the cause and state. Before a production batch, the code should be validated with a small number of test mailboxes and the Exchange versions in use.

## Phase 4: Synchronize, License, and Verify

After the on-premises change, a delta cycle is started on the Entra Connect server:

```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-PolicyType Delta` | Synchronizes only objects changed since the last cycle; the alternative `Initial` would perform a full, significantly longer run |

</details>

For user mailboxes, a valid Exchange Online service plan must then be assigned, for example through group-based licensing. Shared, room, and equipment mailboxes must be assessed according to current Microsoft licensing terms and the required features.

Provisioning is asynchronous. Microsoft generally states less than 30 minutes for normal changes, but in individual cases it can take up to 24 hours. During this time, upstream mail flow should hold messages in a controlled manner instead of delivering them to a target that is not yet ready.

The on-premises verification must now show a RemoteMailbox:

```powershell
Get-RemoteMailbox -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,RemoteRoutingAddress,EmailAddresses

Get-Mailbox -Identity user01@contoso.com -ErrorAction SilentlyContinue
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Get-RemoteMailbox -Identity` | Must return the switched object as a RemoteMailbox |
| `Format-List RecipientTypeDetails,...` | Displays type, addresses, and routing address in list format for verification |
| `Get-Mailbox -Identity ... -ErrorAction SilentlyContinue` | Cross-check: the command must no longer return anything, because no connected mailbox exists on-premises; `-ErrorAction SilentlyContinue` suppresses the expected error message |

</details>

In Exchange Online, verify whether the former MailUser has become a real mailbox:

```powershell
Get-EXORecipient -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,EmailAddresses

Get-EXOMailbox -Identity user01@contoso.com |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,ExchangeGuid
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Get-EXORecipient -Identity` | Displays the recipient type in Exchange Online; `UserMailbox` or the respective special type is expected, no longer `MailUser` |
| `Get-EXOMailbox -Identity` | Returns only real cloud mailboxes; a match proves provisioning has completed |
| `Format-List ...,ExchangeGuid` | Lists the verification attributes; `ExchangeGuid` uniquely identifies the new cloud mailbox |

</details>

The batch is considered complete only when the following tests are also successful:

- Delivery from external and internal senders
- Sending to external and internal recipients
- Replying to an old message to verify the X500 address
- Signing in with Outlook and Outlook on the web
- Delegations and Send As
- Forwarding and transport rules
- Room and equipment bookings
- Applications, scanners, and SMTP relays
- Message trace showing delivery to the new cloud mailbox

## Rollback and Cleanup

The on-premises source mailbox must not be deleted with `Remove-StoreMailbox` during the validation phase. As long as it exists as a disconnected mailbox within mailbox retention, there is still a technical fallback option. However, a rollback requires a controlled reversal of the RemoteMailbox attributes and reconnection of the on-premises mailbox; at the same time, two active delivery targets must be prevented.

Before a rollback, mail flow, synchronization status, and messages already received in the cloud must therefore be backed up. Switching back is not a simple one-liner and should be included in the change as a tested runbook.

After successful acceptance, export requests are cleaned up, PST files are archived according to the protection and retention concept, and temporary permissions on the export share are removed:

```powershell
Get-MailboxExportRequest -BatchName $BatchName |
    Remove-MailboxExportRequest -Confirm:$false
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Get-MailboxExportRequest -BatchName` | Selects exactly the export requests of the completed batch |
| `Remove-MailboxExportRequest -Confirm:$false` | Removes the requests without prompting; the PST files themselves remain unaffected |

</details>

Disconnected mailboxes should only be permanently cleaned up after the agreed rollback window has expired and in accordance with the retention concept.

## Conclusion

If Hybrid Remote Moves are not permitted and no mailbox data needs to be transferred to Exchange Online, an existing synchronized AD user can be switched in a controlled manner from an on-premises mailbox to a new cloud mailbox. The critical part is not `Enable-RemoteMailbox`, but the process control around it: complete inventorying, verified PST backup, explicit approvals, preservation of proxy and X500 addresses, controlled mail flow, and a real rollback window.

## Change Log

- **October 25, 2026:** Updated the source link to the new Microsoft Learn address.

## Sources

1. [Microsoft Learn – Enable-RemoteMailbox](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox?view=exchange-ps): Enables an existing on-premises AD user for a mailbox in the cloud-based service and documents the switches for user, shared, room, and equipment mailboxes.

2. [Microsoft Learn – New-MailboxExportRequest](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-mailboxexportrequest?view=exchange-ps): Reference for exporting primary and archived on-premises mailboxes to PST files.

3. [Microsoft Learn – Mailbox import and export requests](https://learn.microsoft.com/en-us/exchange/mailbox-import-and-export-requests-exchange-2013-help): Prerequisites for the export share, permissions, and Exchange Trusted Subsystem.

4. [Microsoft Learn – Disable or delete a mailbox in Exchange Server](https://learn.microsoft.com/en-us/exchange/recipients/disconnected-mailboxes/disable-or-delete-mailboxes): Behavior of `Disable-Mailbox`, removal of Exchange attributes, and retention of the disconnected mailbox.

5. [Microsoft Learn – Disconnected mailboxes](https://learn.microsoft.com/en-us/exchange/recipients/disconnected-mailboxes/disconnected-mailboxes): Reconnecting, restoring, and permanently deleting disconnected mailboxes.

6. [Microsoft Learn – Delays in provisioning of a user or mailbox](https://learn.microsoft.com/en-us/troubleshoot/exchange/user-and-shared-mailboxes/delays-provision-mailbox-sync-changes): Typical duration and troubleshooting for provisioning in Exchange Online.

7. [Microsoft Learn – Move mailboxes between on-premises and Exchange Online organizations](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/move-mailboxes): The standard Hybrid Remote Move as a reference and distinction from the rebuild described here.
