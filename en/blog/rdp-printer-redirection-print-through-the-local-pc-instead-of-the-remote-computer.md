---
title: "RDP Printer Redirection: Print Through the Local PC Instead of the Remote Computer"
navTitle: "RDP Printer Redirection"
description: "Print jobs from the RDP session should go to the printer next to the user, not to the remote computer. The relevant setting is in three places: the RDP client, the .rdp file, and the target system. This also covers handling the “Unknown Publisher” warning and a troubleshooting checklist."
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
url: https://rafaelpfister.ch/en/blog/rdp-printer-redirection-print-through-the-local-pc-instead-of-the-remote-computer
translationSourceHash: a4f12f591e9dcb86f8ebdd3ff8af1008a130c3ec65424abe789ad4d6446eb4c2
translationModel: gpt-5.6-terra
translatedAt: 2026-08-25T04:13:47.090Z
translationReview: automatic
---

# RDP Printer Redirection: Print Through the Local PC Instead of the Remote Computer

A user works on a remote computer via Remote Desktop and wants to print to the printer next to them. That is exactly what printer redirection is for: The RDP client registers the local printers in the session, the print job travels back to the client over the RDP channel, and is printed there. On the target system, the printer appears with the suffix **(redirected, session n)**. Drivers on the remote computer are generally not required: Windows uses the universal **Remote Desktop Easy Print** driver; the appropriate printer driver only needs to be installed on the local client.

Redirection only takes effect when the connection is established. After every change to the settings, the session must be fully disconnected and reconnected; simply minimizing the RDP window is not enough.

## Client side: enable redirection

The quickest way is through the graphical interface: start `mstsc`, select **Show Options**, open the **Local Resources** tab, check **Printers**, and save the connection on the **General** tab. If you work with .rdp files instead, add the corresponding line directly to the file; .rdp files are plain text files and can be edited with any editor:

```text
redirectprinters:i:1
```

One pitfall with shortcuts without an .rdp file: If the connection is started with `mstsc /v:hostname`, the settings from the hidden file `Default.rdp` in the user's Documents folder apply. If the line `redirectprinters:i:1` is missing, the printer will not appear even though everything seems to be configured correctly. This snippet adds the line idempotently (an existing `0` becomes `1`, and a missing line is added) and displays the result for verification:

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

There are two more client-side traps: First, Windows remembers for each target computer under `HKCU\Software\Microsoft\Terminal Server Client\LocalDevices` which redirections the user last allowed in the security dialog; this saved selection overrides the setting in the .rdp file. Deleting the key resets the state. Second, the registry value `DisablePrinterRedirection` (DWORD, value 1) under `HKLM\Software\Microsoft\Terminal Server Client` disables printer redirection completely on the client; on managed devices, it is worth checking this before troubleshooting within the session.

## Server side: allow redirection

On the target system, the **Do not allow client printer redirection** policy decides the outcome (Computer Configuration → Administrative Templates → Windows Components → Remote Desktop Services → Remote Desktop Session Host → Printer Redirection). If it is set to *Enabled*, no client printers are created, regardless of what the client requests. The most restrictive setting wins: If either side blocks redirection, it does not take place.

Without Group Policy, the same mechanism is controlled through the registry: `fDisableCpm` under `HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp` (0 = redirection allowed, 1 = blocked). In addition, the **Print Spooler** service must be running on the target system; without the spooler, redirected printers are not created either.

The same GPO category contains two useful neighboring settings: **Use Remote Desktop Easy Print printer driver first** (the default and usually the right choice) and **Set default client printer to be default printer in a session**.

## The “Unknown Publisher” warning

When opening an unsigned .rdp file that requests device redirection, the client displays a security warning with checkboxes for the individual resources. Checks added or removed there apply only to that connection launch, but are stored in the `LocalDevices` key mentioned above and therefore silently affect future connections. Anyone wondering why the printer checkbox keeps disappearing despite a correct .rdp file will almost always find the cause there.

There are three ways to handle the warning, in ascending order of effort. First: start the connection with `mstsc /v:hostname` instead of through the .rdp file; without a file, there is no publisher check, and the settings come from `Default.rdp`. Second: approve redirections for the target computer in advance through the registry, which removes the resource portion of the dialog:

```powershell
$key = "HKCU:\Software\Microsoft\Terminal Server Client\LocalDevices"
New-Item -Path $key -Force | Out-Null
Set-ItemProperty -Path $key -Name "hostname-oder-ip" -Value 0xFFFFFFFF -Type DWord
```

Third, the clean approach for .rdp files distributed throughout the organization: sign the file with `rdpsign.exe` and a certificate, then store the certificate thumbprint as a trusted publisher via GPO. The effort is rarely worthwhile for individual workstations, but it is the right solution for centrally distributed connection files.

## Troubleshooting checklist

If the printer does not appear in the session, check the following in this order:

1. **Reconnected?** Redirection only takes effect when the connection is established, not in an existing session.
2. **Correct file?** For shortcuts, check which .rdp file is actually opened; with `mstsc /v:`, the `Default.rdp` file matters.
3. **Saved selection?** Check or delete the `LocalDevices` key on the client.
4. **Client block?** `DisablePrinterRedirection` under `HKLM\Software\Microsoft\Terminal Server Client` must not be set to 1.
5. **Server block?** Check the “Do not allow client printer redirection” GPO or `fDisableCpm` on the target system.
6. **Spooler?** The Print Spooler service must be running on the target system.
7. **Verify in the session:** `Get-Printer | Where-Object DriverName -eq "Remote Desktop Easy Print"` lists redirected printers along with their session ID.

## Sources

1.  [Supported RDP properties](https://learn.microsoft.com/en-us/azure/virtual-desktop/rdp-properties): Reference for all .rdp properties, including redirectprinters with values and default.

2.  [Configure printer redirection over the Remote Desktop Protocol](https://learn.microsoft.com/en-us/azure/virtual-desktop/redirection-configure-printers): Easy Print, GPO and Intune configuration, DisablePrinterRedirection, and testing with Get-Printer.

3.  [rdpsign](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/rdpsign): Command reference for signing .rdp files using a certificate thumbprint.
