---
title: "Performing a BIOS Update Safely: A Guide Using an ASRock AM5 Example, Including BitLocker Preparation"
navTitle: "BIOS Update"
description: "The complete BIOS update process using an ASRock AM5 board as an example: determine the version, verify the download using a hash, properly suspend BitLocker, boot into UEFI (even if F2 does not work), update using Instant Flash, and sensibly configure settings after the update."
date: "2026-08-26"
kategorie: "PC & Hardware"
timeToRead: "8 min read"
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
url: https://rafaelpfister.ch/en/blog/performing-a-bios-update-safely-a-guide-using-an-asrock-am5-example-including-bitlocker
translationSourceHash: 60fff28a10b0f91f3d59996b00afe614f2230b9831514dcf01a1f496b99f4fbd
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:25:16.381Z
translationReview: automatic
---

A BIOS update is one of those maintenance tasks that rarely comes up and therefore raises questions every time: Which version is the right one, how do you get it safely onto the board, and what needs to be considered before and afterward? This guide documents the complete process using an ASRock A620I Lightning WiFi (AM5 socket) with the manufacturer's Instant Flash method. The steps apply to any modern motherboard; the critical aspects (BitLocker, Fast Boot, resetting settings) are manufacturer-independent.

## When a BIOS Update Is Necessary

Three reasons justify the procedure. First, security fixes: firmware vulnerabilities can only be closed through a BIOS update, and manufacturers' changelogs usually mention them only briefly. Second, compatibility: support for new CPU generations and improved memory compatibility come exclusively through new firmware versions—on AM5, through AMD's AGESA reference firmware, which board manufacturers embed in their BIOS versions. Third, stability: if a system restarts spontaneously and the Event Viewer records only Kernel-Power 41 with `BugcheckCode=0`, the crash occurred at the hardware or firmware level, without Windows being involved; typical causes include unstable voltages and memory training, which is precisely the layer maintained by AGESA releases. Entries such as "Improve memory compatibility and system stability" or revised EXPO handling in changelogs indicate that an update addresses such issues. If a system is stable and unaffected by the vulnerabilities fixed, waiting is legitimate; a BIOS update without a reason is a risk with no benefit.

## Step 1: Determine the Current State

Before downloading anything, you need two pieces of information: the exact board model and the installed BIOS version. PowerShell provides both without a restart:

```powershell
Get-CimInstance Win32_ComputerSystem | Select-Object Manufacturer, Model
Get-CimInstance Win32_BIOS | Select-Object SMBIOSBIOSVersion, ReleaseDate
```

Make a note of the version. You will need it later to verify success, and when reading changelogs, you need to know which versions you are skipping.

## Step 2: Download the BIOS and Verify the Checksum

Download the BIOS exclusively from the manufacturer's product page, never from third-party portals. ASRock publishes the SHA256 checksum for every version; compare it after downloading, before the file gets anywhere near a USB drive:

```powershell
Get-FileHash .\A620I_Lightning_WiFi_4.43.zip -Algorithm SHA256
```

If the value does not match the manufacturer's information, the download is corrupted or tampered with: do not flash it. After extracting it, one ROM file remains, in this example `A62IRW_4.43.ROM` at 32 MB.

## Step 3: Prepare the USB Drive

The flash mechanism in UEFI ("Instant Flash" on ASRock, Q-Flash, EZ Flash, or M-Flash on other manufacturers) reads the drive directly from the firmware. This means only FAT32 is reliably recognized; NTFS and exFAT are not. Almost every off-the-shelf USB drive is already FAT32; you can check it like this:

```powershell
Get-CimInstance Win32_LogicalDisk -Filter "DriveType=2" |
  Select-Object DeviceID, FileSystem, VolumeName
```

Copy the ROM file to the root directory of the drive. Reformatting is necessary only if the file system is unsuitable. The drive size is irrelevant; the file is smaller than any commonly available capacity today.

A note on choosing the method: many boards also offer a BIOS Flashback button, which flashes without a CPU or a working system. This is the recovery option for a board that no longer starts. For a working system, Instant Flash in UEFI is the proper and simpler approach. Windows-based flash tools are neither necessary nor recommended on current platforms.

## Step 4: Suspend BitLocker, or You May Be Prompted for the Recovery Key

This is the point missing from many guides. If the system drive is encrypted with BitLocker (which is often automatically enabled in Windows 11 with a Microsoft account), BitLocker binds the key to TPM measurements. A BIOS update changes these measurements, and Windows will request the 48-digit recovery key at the next startup. Anyone who does not have it readily available is left with an inaccessible system.

BitLocker includes its own mechanism for this scenario. In an elevated PowerShell session:

```powershell
Suspend-BitLocker -MountPoint C: -RebootCount 2
```

The `RebootCount 2` parameter covers both upcoming restarts (once into UEFI and once after flashing); afterward, protection re-enables itself and seals the key against the new measurements. Regardless, verify beforehand that the recovery key can be found, for example in the Microsoft account at aka.ms/myrecoverykey or through `manage-bde -protectors -get C:`.

## Step 5: Get into UEFI, Even if F2 Does Not Respond

The traditional method of pressing F2 or Delete when powering on often fails on modern systems: with Fast Boot enabled, the firmware initializes the USB keyboard only after POST, so the keystroke does not register. But you do not need to rely on the key: Windows can direct the next restart straight into UEFI Setup. In an elevated PowerShell session:

```powershell
shutdown /r /fw /t 5
```

If the command reports error 203 ("The system could not find the environment option that was entered"), administrator rights are almost always missing: without elevation, the process cannot set the necessary firmware variable, and the error message does not name this cause. A second option without a firmware variable goes through the recovery environment: `shutdown /r /o`, then Troubleshoot, Advanced options, UEFI Firmware Settings.

## Step 6: Flash with Instant Flash

In UEFI, you will find Instant Flash in the Tool menu. The tool lists all ROM files on the drive; after selection, it verifies the file, flashes it, and restarts automatically. During the few minutes it takes, the sole hard rule of the entire process applies: do not interrupt the power supply or turn off the computer. An interrupted flash is the only step in this guide that can actually render the board unable to boot (and then requires the Flashback recovery option mentioned above).

## Step 7: Follow Up, Because the Update Resets Everything

After flashing, all BIOS settings are set to factory defaults. This is intentional and provides a diagnostic opportunity: the RAM now runs at its JEDEC base speed without an EXPO profile. If you flashed because of stability issues, deliberately leave it that way for one to two weeks. If the crashes stop, the memory profile was involved, and you can specifically test EXPO again with the new firmware. The everyday difference between 4800 and 6000 MT/s is barely noticeable outside benchmarks; a stable computer is worth every benchmark point.

Two settings are worth visiting in UEFI anyway: if you had idle restarts, under Advanced, AMD CBS you can set "Power Supply Idle Control" to "Typical Current Idle"; this mitigates a known incompatibility between some power supplies and the deep idle states of Ryzen CPUs. And if you want to return to entering Setup with F2 in the future, you can disable Fast Boot.

To verify success back in Windows:

```powershell
Get-CimInstance Win32_BIOS | Select-Object SMBIOSBIOSVersion
Get-CimInstance Win32_PhysicalMemory |
  Select-Object PartNumber, ConfiguredClockSpeed
manage-bde -status C:
```

The first line must show the new version, the second the expected memory speed, and BitLocker must again report "Protection On." This completes and documents the update. If you flashed due to stability problems, only observation over the following weeks will show whether they have been resolved, most easily by checking for new Kernel-Power 41 entries in the System event log.

## Sources

1.  [ASRock A620I Lightning WiFi, BIOS downloads](https://pg.asrock.com/mb/AMD/A620I%20Lightning%20WiFi/index.asp#BIOS): Version list with changelogs, SHA256 checksums, and the supported update methods for the example board.

2.  [Microsoft Learn: Suspend-BitLocker](https://learn.microsoft.com/en-us/powershell/module/bitlocker/suspend-bitlocker): Reference for suspending BitLocker protection, including the RebootCount parameter.

3.  [Microsoft Learn: Advanced troubleshooting for Event ID 41](https://learn.microsoft.com/en-us/troubleshoot/windows-client/performance/event-id-41-restart): Classification of Kernel-Power 41 and the meaning of BugcheckCode 0.

4.  [Microsoft Learn: shutdown command](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/shutdown): Documentation of the /fw and /o parameters for restarting into UEFI and the recovery environment, respectively.
