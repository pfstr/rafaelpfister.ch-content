---
title: "Rescuezilla: Migrate Windows to a New SSD, with an Introduction to the Tool"
navTitle: "Rescuezilla Migration"
description: "Rescuezilla is a free, Clonezilla-compatible imaging solution with a graphical interface. This article introduces the tool and shows the complete migration of a Windows installation to a new SSD: preparation in Windows, bootable USB drive, backup, verification, restore, and follow-up steps."
date: "2026-09-29"
kategorie: "PC & Hardware"
timeToRead: "10 min to read"
themen:
  - pc-hardware
produkte:
  - "windows-client"
protokolle:
  - "backup-dr"
  - "migration"
slug: "rescuezilla-migrate-windows-to-a-new-ssd-with-an-introduction-to-the-tool"
translationId: "article-d9438c5774d0167c"
translationOf: rescuezilla-windows-migration
url: https://rafaelpfister.ch/en/blog/rescuezilla-migrate-windows-to-a-new-ssd-with-an-introduction-to-the-tool
translationSourceHash: 33320c51c0a77d69bc5804e2c7e69d14c63731a23df6ef55969b37b043a49cd5
translationModel: gpt-5.6-terra
translatedAt: 2026-09-30T09:49:25.030Z
translationReview: automatic
---

Anyone who wants to move a Windows installation to a new SSD or a new computer without reinstalling Windows needs a tool that backs up and restores the entire drive with all its partitions. Rescuezilla is such a tool: free, open source, and with a graphical interface that can be used without Linux knowledge. This article introduces the tool and describes migration to an SSD of equal or greater size. There is a separate guide for a smaller target SSD: [Move Windows to a smaller SSD with Rescuezilla](/blog/windows-kleinere-ssd-rescuezilla).

## What Rescuezilla is

Rescuezilla is an Ubuntu-based live system that starts from a USB drive. It runs independently of the installed operating system and can therefore back up volumes that Windows locks while it is running. The project was created in 2019 as a fork of Redo Backup and Recovery, which had not been maintained for seven years at that point. Since version 2.0 (2020), Rescuezilla has written images in Clonezilla format: a backup created with Rescuezilla can be restored with Clonezilla and vice versa. It is licensed under GPL-3.0. The current version is 2.6.2 from May 2026, based on Ubuntu 26.04 LTS with Partclone 0.3.47.

Partclone does the actual work. It supports common file systems (NTFS, FAT, ext4, and others) and reads only used blocks. A 1 TB partition with 350 GB of data therefore produces an image of around 215 GB (gzip-compressed), rather than 1 TB. Rescuezilla backs up partitions without a recognized file system, such as the Microsoft Reserved Partition, block by block with `dd`; these files have the `.dd-ptcl-img` extension in the image.

<details class="options-details">
<summary>Features at a glance</summary>

| Feature | Purpose |
|---|---|
| Backup | backs up selected partitions on a drive, including the partition table, as an image to a local drive or network share (SMB, SSH) |
| Restore | restores an image to a drive, optionally overwriting the partition table |
| Verify Image | checks whether an existing image is complete and readable |
| Clone | copies one drive directly to another, without intermediate storage |
| Image Explorer (beta) | mounts an image read-only to retrieve individual files |
| VM images | reads VDI, VMDK, VHDX, QCOW2, and raw images in addition to Clonezilla images |
| Additional tools | GParted, file manager, web browser, and tools for recovering deleted files on the live desktop |
| CLI | experimental command line (since 2.5) for Backup, Verify, Restore, and Clone |

</details>

One limitation is important for migrations: Rescuezilla does not shrink partitions. The target drive must extend at least to the end of the last backed-up partition. If it is smaller, preparation with GParted is required, as described in the guide linked above.

## Image or clone

Rescuezilla offers two ways to migrate. With **cloning**, the source and target drives are connected at the same time, and Rescuezilla copies directly. This saves time and a third drive, but requires both drives to be connected to the computer or an adapter at the same time. With **imaging**, an image is first created on an external drive and then restored to the new drive. This takes longer, but has one advantage: the image remains as a complete backup of the old state, even if something goes wrong during restoration. An image is therefore the safer choice for a migration, and this article describes that option.

## Requirements

1.  **USB drive** for Rescuezilla. Its contents will be erased when writing the boot image.

2.  **External drive** for the image, with free space approximately equal to the amount of used data. Both exFAT and NTFS work.

3.  **Target drive**, at least as large as the source drive. If it is smaller, first work through the guide for smaller SSDs.

4.  **BitLocker recovery key**, if C: is encrypted. Available at aka.ms/myrecoverykey or in the Entra portal for managed devices.

## Step 1: Prepare Windows

Check three items before backing up in a PowerShell session with administrator rights.

**BitLocker.** Partclone cannot read a BitLocker volume as NTFS. Rescuezilla then backs it up block by block like any partition without a recognized file system, and the image becomes as large as the entire partition. For a migration, it is advisable to turn off BitLocker beforehand and turn it back on after the move:

```powershell
manage-bde -status C:
manage-bde -off C:
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-status` | shows the volume's encryption level and protection status |
| `-off` | fully decrypts the volume; continues running in the background |
| `C:` | positional argument: the affected volume |

</details>

Wait until `manage-bde -status C:` shows the value `Fully Decrypted`. On devices managed through Intune, a policy may turn BitLocker back on; therefore, check the status again immediately before restarting.

**Hibernation and Fast Startup.** With Fast Startup enabled, Windows does not shut down completely, and NTFS is considered still in use. Partclone then aborts. One command disables both:

```powershell
powercfg /h off
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `/h off` | short form of `/hibernate off`: disables hibernation and Fast Startup, and removes `hiberfil.sys` |

</details>

**File system state.** If the dirty bit is set, Partclone aborts with the message that the volume is “scheduled for a check or it was shutdown uncleanly.” Check it beforehand:

```powershell
fsutil dirty query C:
```

If the command reports `is Dirty`, schedule a check for the next startup with `chkdsk C: /f`, restart, and check again.

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `dirty query C:` | fsutil: checks whether the volume's dirty bit is set |
| `/f` | chkdsk: fixes errors; on the system volume, it schedules the check for the next startup |

</details>

## Step 2: Create and boot the USB drive

Download the ISO image from the project's GitHub release page. The standard variant includes the Ubuntu codename in its filename; for 2.6.2, it is `rescuezilla-2.6.2-64bit.resolute.iso`. Write it to the USB drive with an image writer such as balenaEtcher; the project page recommends this program for Windows, macOS, and Linux.

Windows itself can initiate booting from the USB drive:

```powershell
shutdown /r /o /t 0
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `/r` | restarts instead of shutting down |
| `/o` | starts into advanced startup options; only together with `/r` |
| `/t 0` | delay in seconds before execution |

</details>

In the advanced startup options, select “Use a device” and the USB drive. Alternatively, the firmware boot menu opens when powering on (F8, F11, or F12, depending on the vendor). Rescuezilla starts with Secure Boot enabled; it does not need to be disabled. If the screen remains black after selecting the language, use “Graphical Fallback Mode” in the USB drive's boot menu.

## Step 3: Create a backup

Rescuezilla starts automatically on the desktop. The wizard guides you through the backup in numbered steps:

1.  Select **Backup**.

2.  *Step 1: Select Drive To Backup*: select the system drive. The model and size help distinguish drives; device names (`nvme0n1`, `sda`) may differ between two boots.

3.  *Step 2: Select Partitions to Save*: leave all partitions checked. A bootable restore requires the EFI system partition, Microsoft Reserved Partition, C:, and the recovery partition.

4.  *Step 3: Select Destination Drive*: the external drive (Local) or a network share (Network).

5.  *Step 4: Select Destination Folder*: the destination folder on the drive. Rescuezilla creates a subfolder with a timestamp in it.

6.  *Step 5: Name Your Backup*: optionally enter a description; it will later appear in the image list.

7.  *Step 6: Customize Compression Settings*: the default gzip setting is suitable in most cases.

8.  *Step 7: Confirm Backup Configuration*: review the settings and start.

For 350 GB of data on an NVMe SSD, backing up to an external USB SSD took around 20 minutes. At the end, Rescuezilla reports whether each partition was backed up successfully. If a partition shows an error, the image is incomplete even if the folder exists.

## Step 4: Verify the image

Verify the image before restoring it. **Verify Image** in the main menu reads all parts and checks that they are complete and readable. In the image list, the **Partitions** column shows the backed-up partitions and their sizes; a yellow warning triangle marks images with missing partitions.

The image folder can also be viewed in Windows. The most important files are:

| File | Contents |
|---|---|
| `nvme0n1-pt.parted` | partition table in text form, with the start and end of each partition in sectors |
| `nvme0n1-gpt-1st`, `nvme0n1-gpt-2nd` | binary copies of the primary and backup GPT |
| `nvme0n1p3.ntfs-ptcl-img.gz.aa`, `.ab`, … | Partclone image of C:, gzip-compressed and split into 4 GB parts |
| `nvme0n1p2.dd-ptcl-img.gz.aa` | block-by-block copy of a partition without a recognized file system |
| `clonezilla-img` | backup log with the result for each partition |
| `Info-*.txt` | hardware and SMART information for the source system at the time of the backup |

From `nvme0n1-pt.parted`, you can determine whether the image fits on the target drive: the last sector of the final partition plus one, multiplied by 512 bytes, must be smaller than the target drive's size in bytes.

## Step 5: Restore to the new SSD

Install the new SSD and start Rescuezilla again.

1.  Select **Restore**.

2.  *Step 1: Select Image Location*: the external drive containing the image.

3.  *Step 2: Select Backup Image*: select the image based on its date and partition sizes.

4.  *Step 3: Select Drive To Restore*: the new SSD. Check the model and size twice: all data on this drive will be overwritten.

5.  *Step 4: Select Partitions to Restore*: leave all partitions checked, as well as **Overwrite partition table**.

6.  *Step 5: Confirm Restore Configuration*: review and start.

The restore takes about as long as the backup. Then shut down and remove the USB drive.

## Step 6: Boot from the new drive

The easiest option is to disconnect the old drive before the first boot. If both drives are connected, they have the same partition and volume IDs, and the firmware may select the old one to boot. If the old drive will continue to be used, format it only after verifying the new system.

If the new SSD is larger than the old one, the additional space appears as unallocated space at the end. However, C: cannot be extended directly in Disk Management because the recovery partition is between C: and the free space. With GParted on the Rescuezilla USB drive, first move the recovery partition to the end (Resize/Move, set **Free space following** to 0), then expand C: into all the free space. Windows RE finds its partition again afterward because it retains its partition number; `reagentc /info` shows the status.

If the computer also changes during the migration, Windows generally starts without adjustments and installs missing drivers afterward. Activation happens through the digital license; after a motherboard change, it may require signing in with the Microsoft account and using the activation troubleshooter.

## Follow-up steps

On the new drive, re-enable what was disabled for the migration. `powercfg /h on` enables hibernation again. Enable BitLocker through Settings → Privacy & security → Device encryption or with `manage-bde -on C:`; then check with `manage-bde -protectors -get C:` that a recovery key exists and is backed up.

Keep the image on the external drive until Windows has run on the new drive without issues for a few days and all data has been checked. It can then be deleted or archived as a backup of the original state.

## Sources

1.  [Rescuezilla](https://rescuezilla.com/): project page with feature overview, download, and FAQ.

2.  [Rescuezilla on GitHub](https://github.com/rescuezilla/rescuezilla): source code, project history, and feature list in the README.

3.  [Rescuezilla Releases](https://github.com/rescuezilla/rescuezilla/releases/latest): current version with release notes, compatibility list, and ISO variants.

4.  [Rescuezilla Changelog](https://raw.githubusercontent.com/rescuezilla/rescuezilla/master/CHANGELOG.md): introduction of CLI, Verify Image, and Clone, plus the update to the Secure Boot shim.

5.  [Rescuezilla Wiki: Restoring to a smaller disk](https://github.com/rescuezilla/rescuezilla/wiki/HOWTO:-Restoring-to-a-smaller-disk.-Eg,-1000GB-HDD-to-500GB-SSD): official process for target drives smaller than the original.

6.  [Partclone](https://partclone.org/): tool used by Rescuezilla and Clonezilla for file-system-aware backups.

7.  [GParted Manual](https://gparted.org/display-doc.php?name=help-manual): moving and enlarging partitions.

8.  [Microsoft Learn: manage-bde status](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-status): checking BitLocker status.

9.  [Microsoft Learn: manage-bde off](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-off): decrypting a volume.

10.  [Microsoft Learn: Powercfg command-line options](https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/powercfg-command-line-options): option `/hibernate` and its effect on Fast Startup.

11.  [Microsoft Learn: fsutil dirty](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/fsutil-dirty): checking the dirty bit.

12.  [Microsoft Learn: chkdsk](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/chkdsk): checking and repairing NTFS volumes.

13.  [Microsoft Learn: shutdown](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/shutdown): parameter `/o` for starting in advanced startup options.

14.  [Microsoft Support: Reactivating Windows after a hardware change](https://support.microsoft.com/en-us/windows/reactivating-windows-after-a-hardware-change-2c0e962a-f04c-145b-6ead-fb3fc72b6665): activation through the digital license after a motherboard change.
