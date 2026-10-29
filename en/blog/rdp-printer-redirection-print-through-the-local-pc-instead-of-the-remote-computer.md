---
title: "RDP Printer Redirection: Print Through the Local PC Instead of the Remote Computer"
navTitle: "RDP Printer Redirection"
description: "Print jobs from the RDP session should go to the printer next to the user, not to the remote computer. The setting is configured in three places: the RDP client, the .rdp file, and the target system. This also covers handling the “Unknown publisher” warning and a troubleshooting checklist."
date: "2026-08-24"
kategorie: "Windows Client"
timeToRead: "5 min read"
themen:
  - windows-client
produkte:
  - "windows-client"
protokolle:
  - "troubleshooting"
slug: "rdp-printer-redirection-print-through-the-local-pc-instead-of-the-remote-computer"
translationId: "article-12521248666e9809"
draft: false
translationOf: rdp-druckerumleitung-lokale-drucker
translationSourceHash: 1ca6c78df61e05f90d1aefc5b20b611e5c6a4bf03908d0466da114c2823e31cd
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:32:26.860Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/en/blog/rdp-printer-redirection-print-through-the-local-pc-instead-of-the-remote-computer
---

# RDP Printer Redirection: Print Through the Local PC Instead of the Remote Computer

A user works on a remote computer via Remote Desktop and wants to print to the printer next to them. That is exactly what printer redirection is for: The RDP client registers the local printers in the session, the print job is sent back to the client over the RDP channel, and printed there. On the target system, the printer appears with the suffix **(redirected, session n)**. Drivers on the remote computer are generally not required: Windows uses the universal **Remote Desktop Easy Print** driver; the appropriate printer driver only needs to be installed on the local client.

Redirection takes effect only when the connection is established. After every change to the settings, the session must be fully disconnected and reconnected; simply minimizing the RDP window is not enough.

## Client side: enable redirection

The easiest way to enable printer redirection is through the graphical interface: Start `mstsc`, select **Show Options**, open the **Local Resources** tab, select **Printers**, and save the connection on the **General** tab. If you work with .rdp files, you can adjust the line directly in the file; .rdp files are plain text files and can be edited with any editor:

```text
redirectprinters:i:1
```

There is one special case for shortcuts without an .rdp file: If the connection is started with `mstsc /v:hostname`, the settings from the hidden file `Default.rdp` in the user's Documents folder apply. If the line `redirectprinters:i:1` is missing there, the printer will not appear even though everything seems to be configured correctly. This snippet adds the line idempotently (an existing `0` is changed to `1`, and a missing line is added) and displays the result for verification:

```powershell
$f = "$env:USERPROFILE\Documents\Default.rdp"
if (Test-Path $f) {
    $c = Get-Content $f
    if ($c -match 'redirectprinters') {
        $c -replace 'redirectprinters:i:0', 'redirectprinters:i:1' | Set-Content $f
    } else {
        Add-Content $f 'redirectprinters:i:1'
    }
} else {
    Set-Content $f 'redirectprinters:i:1'
}
Select-String -Path $f -Pattern 'redirectprinters'
```

There are two more sources of errors on the client side: First, under `HKCU\Software\Microsoft\Terminal Server Client\LocalDevices`, Windows stores for each target computer which redirections the user last allowed in the security dialog; this saved selection overrides the setting in the .rdp file. Deleting the key resets the state. Second, the registry value `DisablePrinterRedirection` (DWORD, value 1) under `HKLM\Software\Microsoft\Terminal Server Client` disables printer redirection completely on the client; on managed devices, it is worth checking before troubleshooting the session.

## Server side: allow redirection

On the target system, the **Do not allow client printer redirection** policy decides the matter (Computer Configuration → Administrative Templates → Windows Components → Remote Desktop Services → Remote Desktop Session Host → Printer Redirection). If it is set to *Enabled*, no client printers are created, regardless of what the client requests. The most restrictive setting wins: If either side blocks redirection, it will not occur.

Without Group Policy, the same mechanism is controlled through the registry: `fDisableCpm` under `HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp` (0 = redirection allowed, 1 = blocked). In addition, the **Print Spooler** service must be running on the target system; without the spooler, redirected printers are not created either.

The same GPO category contains two additional useful settings: **Use Remote Desktop Easy Print printer driver first** (the default and usually the right choice) and **Set the client default printer to be the default printer in a session**.

## The “Unknown publisher” warning

When opening an unsigned .rdp file that requests device redirections, the client displays a security warning with checkboxes for the individual resources. Selections made or cleared there apply only to that connection attempt, but are stored in the `LocalDevices` key mentioned above and therefore silently affect future connections. If you wonder why the printer checkbox keeps disappearing despite a correct .rdp file, you will almost always find the cause there.

Since the April 2026 security update (CVE-2026-26151), Remote Desktop Connection displays a new security dialog whenever an .rdp file is opened, before the connection is established. All redirections requested by the file, including printers, are disabled by default and must be enabled individually. Unsigned files also receive the notice “Caution: Unknown remote connection.” According to Microsoft, connections started directly with a computer name in Remote Desktop Connection are not affected. As a temporary workaround, Microsoft documents the registry value `RedirectionWarningDialogVersion` (DWORD, value 1) under `HKLM\Software\Policies\Microsoft\Windows NT\Terminal Services\Client`, which restores the previous dialog; support for it may be removed in a future update.

There are three ways to deal with the warning, in increasing order of effort. First: Start the connection with `mstsc /v:hostname` instead of through the .rdp file; without a file, there is no publisher check, and the settings come from `Default.rdp`. The security dialog introduced in April 2026 does not appear either. Second: Pre-approve the redirections for the target computer through the registry, which removes the resource portion of the previous dialog. Microsoft does not document whether this pre-approval also applies to the new security dialog introduced in April 2026:

```powershell
$key = "HKCU:\Software\Microsoft\Terminal Server Client\LocalDevices"
New-Item -Path $key -Force | Out-Null
Set-ItemProperty -Path $key -Name "hostname-oder-ip" -Value 0xFFFFFFFF -Type DWord
```

Third, the proper approach for .rdp files distributed within an organization: Sign the file with `rdpsign.exe` and a certificate, then use GPO to store the certificate thumbprint as a trusted publisher. The effort is rarely worthwhile for individual workstations, but it is the right solution for centrally distributed connection files.

## Troubleshooting checklist

If the printer does not appear in the session, check the following in this order:

1. **Reconnected?** Redirection takes effect only when the connection is established, not in an existing session.
2. **Correct file?** For shortcuts, check which .rdp file is actually opened; with `mstsc /v:`, the `Default.rdp` file is what matters.
3. **Saved selection?** Check or delete the `LocalDevices` key on the client. When starting through an .rdp file, also check whether the printer was enabled in the security dialog; since April 2026, it is disabled there by default.
4. **Client block?** `DisablePrinterRedirection` under `HKLM\Software\Microsoft\Terminal Server Client` must not be set to 1.
5. **Server block?** Check the “Do not allow client printer redirection” GPO or `fDisableCpm` on the target system.
6. **Spooler?** The Print Spooler service must be running on the target system.
7. **Check in the session:** `Get-Printer | Where-Object DriverName -eq "Remote Desktop Easy Print"` lists redirected printers along with their session ID.

## Change log

- **October 25, 2026:** Added: Since the April 2026 security update (CVE-2026-26151), a new dialog when opening .rdp files shows all redirections, including printers, disabled by default; also added the RedirectionWarningDialogVersion registry value and an additional verification step.

## Sources

1.  [Supported RDP properties](https://learn.microsoft.com/en-us/azure/virtual-desktop/rdp-properties): Reference for all .rdp properties, including redirectprinters with values and default.

2.  [Configure printer redirection over the Remote Desktop Protocol](https://learn.microsoft.com/en-us/azure/virtual-desktop/redirection-configure-printers): Easy Print, GPO and Intune configuration, DisablePrinterRedirection, and testing with Get-Printer.

3.  [rdpsign](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/rdpsign): Command reference for signing .rdp files using a certificate thumbprint.

4.  [Understanding security warnings when opening Remote Desktop (RDP) files](https://learn.microsoft.com/en-us/windows-server/remote/remote-desktop-services/remotepc/understanding-security-warnings): New security dialog since the April 2026 update, redirections disabled by default, and the RedirectionWarningDialogVersion registry value.
