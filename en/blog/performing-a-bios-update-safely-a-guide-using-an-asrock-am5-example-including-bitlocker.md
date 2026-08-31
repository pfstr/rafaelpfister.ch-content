---
title: "Performing a BIOS Update Safely: A Guide Using ASRock AM5 as an Example, Including BitLocker Preparation"
navTitle: "BIOS Update"
description: "The complete BIOS update process using an ASRock AM5 board as an example: determine the version, verify the download using a hash, properly suspend BitLocker, boot into UEFI (even if F2 does nothing), update with Instant Flash, and configure settings sensibly after the update."
date: "2026-08-26"
kategorie: "PC & Hardware"
timeToRead: "8 min to read"
themen:
  - pc-hardware
produkte:
  - "windows-client"
protokolle:
  - "troubleshooting"
  - "releases"
slug: "performing-a-bios-update-safely-a-guide-using-an-asrock-am5-example-including-bitlocker"
translationId: "article-82840b2d159b9367"
translationOf: bios-update-asrock-am5-sicher-durchfuehren
translationSourceHash: 555b16e753b2ac5dec357741b071ed6aa33de367a2197a8dbb10fef7c9f6a946
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:13:12.256Z
translationReview: automatic
url: https://rafaelpfister.ch/en/blog/performing-a-bios-update-safely-a-guide-using-an-asrock-am5-example-including-bitlocker
---

A BIOS update is one of those maintenance tasks that rarely comes up and therefore raises questions every time: Which version is the right one, how do you get it safely onto the board, and what needs to be considered before and afterward? This guide documents the complete process using an ASRock A620I Lightning WiFi (Socket AM5) with the manufacturer’s Instant Flash method. The steps apply to any modern motherboard; the critical points (BitLocker, Fast Boot, resetting settings) are manufacturer-independent.

## When a BIOS update is warranted

Three situations justify the intervention. First, security fixes: Firmware vulnerabilities can only be closed through a BIOS update, and manufacturers’ changelogs usually mention them only briefly. Second, compatibility: Support for new CPU generations and improved memory compatibility are available only through new firmware versions—on AM5, through AMD’s AGESA reference firmware, which board manufacturers embed in their BIOS versions. Third, stability: If a system restarts unexpectedly and the Event Log records only Kernel-Power 41 with `BugcheckCode=0`, the crash occurred at the hardware or firmware level, without Windows being involved; typical causes include unstable voltages and memory training, precisely the layer maintained by AGESA releases. Entries such as "Improve memory compatibility and system stability" or revised EXPO handling in changelogs indicate that an update addresses such issues. Conversely, if a system is stable and unaffected by the vulnerabilities fixed, waiting is legitimate; a BIOS update without a reason is risk with no benefit.

## Step 1: Determine the current state

Before downloading anything, you need two pieces of information: the exact board model and the installed BIOS version. PowerShell provides both without a restart:

```powershell
Get-CimInstance Win32_ComputerSystem | Select-Object Manufacturer, Model
Get-CimInstance Win32_BIOS | Select-Object SMBIOSBIOSVersion, ReleaseDate
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Win32_ComputerSystem` | ClassName positional argument: CIM class containing the system manufacturer and model |
| `Win32_BIOS` | CIM class containing firmware information, including version and date |
| `Select-Object <eigenschaften>` | limits the output to the specified properties |

</details>

Make a note of the version. You will need it later to verify success, and when reading the changelogs, you need to know which versions you are skipping.

## Step 2: Download the BIOS and verify the checksum

Download the BIOS exclusively from the manufacturer’s product page, never from third-party portals. ASRock publishes the SHA256 checksum for each version; after downloading, compare it before the file even gets near a USB drive:

```powershell
Get-FileHash .\A620I_Lightning_WiFi_4.43.zip -Algorithm SHA256
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `.\A620I_…_4.43.zip` | Path positional argument: the file to verify |
| `-Algorithm SHA256` | Hash algorithm; must match the checksum type published by the manufacturer |

</details>

If the value does not match the manufacturer’s information, the download is corrupted or tampered with: do not flash it. After extracting it, a single ROM file remains, `A62IRW_4.43.ROM` in this example, with a size of 32 MB.

## Step 3: Prepare the USB drive

The flash mechanism in UEFI ("Instant Flash" on ASRock, Q-Flash, EZ Flash, or M-Flash on other manufacturers) reads the drive directly from the firmware. This means only FAT32 is reliably recognized; NTFS and exFAT are not. Almost every ready-made USB drive is already FAT32; you can check it like this:

```powershell
Get-CimInstance Win32_LogicalDisk -Filter "DriveType=2" |
  Select-Object DeviceID, FileSystem, VolumeName
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Win32_LogicalDisk` | CIM class for logical drives |
| `-Filter "DriveType=2"` | WQL filter for removable media; hides hard drives and CD drives |
| `Select-Object DeviceID, FileSystem, VolumeName` | displays the drive letter, file system, and volume name |

</details>

Copy the ROM file to the root directory of the drive. Reformatting is necessary only if the file system is unsuitable. Drive size is irrelevant; the file is smaller than any capacity commonly available today.

A note on choosing a method: Many boards also offer a BIOS Flashback button that flashes without a CPU and without a working system. This is the recovery path for a board that no longer boots. For a running system, Instant Flash in UEFI is the appropriate and simpler method. Windows-based flashing tools are neither necessary nor recommended on current platforms.

## Step 4: Suspend BitLocker or risk being prompted for the recovery key

This is the point missing from many guides. If the system drive is encrypted with BitLocker (often automatically enabled on Windows 11 with a Microsoft account), BitLocker ties the key to TPM measurements. A BIOS update changes those measurements, and Windows requests the 48-digit recovery key on the next startup. Anyone who does not have it readily available is left with an inaccessible system.

BitLocker has its own mechanism for this scenario. In PowerShell with administrator rights:

```powershell
Suspend-BitLocker -MountPoint C: -RebootCount 2
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-MountPoint C:` | the affected volume, here the system drive |
| `-RebootCount 2` | number of restarts for which protection remains suspended (0 to 15; 0 = until manually resumed) |

</details>

The value 2 covers both upcoming restarts (once into UEFI, once after flashing); protection then reactivates automatically and seals the key against the new measurements. Regardless, verify beforehand that the recovery key can be found, for example in the Microsoft account at aka.ms/myrecoverykey or via `manage-bde -protectors -get C:`.

## Step 5: Enter UEFI, even if F2 does not respond

The traditional method of pressing F2 or Delete while turning on the system often fails on modern systems: With Fast Boot enabled, firmware initializes the USB keyboard only after POST, so the key press does not register. But you do not need to rely on the key: Windows can direct the next restart straight into UEFI Setup. In PowerShell with administrator rights:

```powershell
shutdown /r /fw /t 5
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `/r` | restart instead of shutting down |
| `/fw` | sets the firmware variable that directs the next startup straight into UEFI Setup; works only together with a shutdown option such as `/r`, and requires administrator rights |
| `/t 5` | delay in seconds before execution |

</details>

If the command reports error 203 ("The system could not find the environment option that was entered"), administrator rights are almost always missing: Without elevation, the process cannot set the necessary firmware variable, and the error message does not identify this cause. A second route without a firmware variable goes through the recovery environment: `shutdown /r /o`, then Troubleshoot, Advanced options, UEFI Firmware Settings.

## Step 6: Flash with Instant Flash

In UEFI, you will find Instant Flash in the Tools menu. The tool lists all ROM files on the USB drive; after you select one, it verifies the file, flashes it, and restarts automatically. During the few minutes it takes, the one hard rule for the entire process applies: Do not interrupt the power supply or turn off the computer. An interrupted flash is the only step in this guide that can actually make the board unbootable (in which case it needs the Flashback recovery path mentioned above).

## Step 7: Follow-up work, because the update resets everything

After flashing, all BIOS settings are restored to factory defaults. This is intentional and provides a diagnostic opportunity: The RAM now runs at the JEDEC base speed without an EXPO profile. If you flashed because of stability issues, deliberately leave it that way for one to two weeks. If the crashes stop, the memory profile was involved, and you can deliberately test EXPO again with the new firmware. The everyday difference between 4800 and 6000 MT/s is barely noticeable outside benchmarks; a stable computer is worth every benchmark point.

Two settings are worth visiting in UEFI anyway: If you had restarts while idle, under Advanced, AMD CBS you can set "Power Supply Idle Control" to "Typical Current Idle"; this mitigates a known incompatibility between some power supplies and the deep idle states of Ryzen CPUs. And if you want to use F2 to enter Setup again in the future, you can disable Fast Boot.

Verify success back in Windows:

```powershell
Get-CimInstance Win32_BIOS | Select-Object SMBIOSBIOSVersion
Get-CimInstance Win32_PhysicalMemory |
  Select-Object PartNumber, ConfiguredClockSpeed
manage-bde -status C:
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Win32_BIOS` | CIM class with the firmware version; `SMBIOSBIOSVersion` must now show the new version |
| `Win32_PhysicalMemory` | CIM class for the memory modules; `ConfiguredClockSpeed` shows the actual operating speed in MT/s |
| `-status` | manage-bde: displays the volume’s encryption and protection status |
| `C:` | positional argument: the volume to check |

</details>

The first line must show the new version, the second the expected memory speed, and BitLocker must again report "Protection On." This completes and documents the update. If you flashed because of stability issues, only observation over the following weeks will show whether they are resolved, most easily by checking for new Kernel-Power 41 entries in the System Event Log.

## Sources

1.  [ASRock A620I Lightning WiFi, BIOS downloads](https://pg.asrock.com/mb/AMD/A620I%20Lightning%20WiFi/index.asp#BIOS): List of versions with changelogs, SHA256 checksums, and supported update methods for the example board.

2.  [Microsoft Learn: Suspend BitLocker](https://learn.microsoft.com/en-us/powershell/module/bitlocker/suspend-bitlocker): Reference for suspending BitLocker protection, including the RebootCount parameter.

3.  [Microsoft Learn: Advanced troubleshooting for Event ID 41](https://learn.microsoft.com/en-us/troubleshoot/windows-client/performance/event-id-41-restart): Explanation of Kernel-Power 41 and the significance of BugcheckCode 0.

4.  [Microsoft Learn: shutdown command](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/shutdown): Documentation for the /fw and /o parameters for restarting into UEFI and the recovery environment, respectively.
