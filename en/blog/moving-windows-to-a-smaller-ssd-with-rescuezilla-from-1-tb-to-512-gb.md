---
title: "Moving Windows to a Smaller SSD with Rescuezilla: From 1 TB to 512 GB"
navTitle: "Move to smaller SSD"
description: "Rescuezilla restores an image only to a disk that extends at least to the end of the final partition. Using a 1 TB NVMe as an example, this guide shows how to shrink C: with GParted, move the recovery partition, remove the dirty flag, and restore the new image to a 512 GB SSD."
date: "2026-09-29"
kategorie: "PC & Hardware"
timeToRead: "9 min to read"
themen:
  - pc-hardware
produkte:
  - "windows-client"
protokolle:
  - "backup-dr"
  - "migration"
slug: "moving-windows-to-a-smaller-ssd-with-rescuezilla-from-1-tb-to-512-gb"
translationId: "article-6947213279383f27"
translationOf: windows-kleinere-ssd-rescuezilla
url: https://rafaelpfister.ch/en/blog/moving-windows-to-a-smaller-ssd-with-rescuezilla-from-1-tb-to-512-gb
translationSourceHash: ad4485736511552671c4e469ddaa672e9439cffb3e74d43bd8dd7333eaf662d3
translationModel: gpt-5.6-terra
translatedAt: 2026-09-30T09:53:35.141Z
translationReview: required
---

Rescuezilla is a free live system for backing up and restoring entire drives, compatible with Clonezilla's image format (tool overview and basic process: [Rescuezilla: Migrate Windows to a New SSD](/blog/rescuezilla-windows-migration)). As long as the target disk is the same size or larger, a backup and restore are sufficient. If it is smaller, the restore fails even though the used data would fit. This article documents moving a Windows 11 system from a 1 TB NVMe (C: with 930 GB, of which around 350 GB is used) to a 512 GB NVMe, including the error messages that occur along the way.

## Why the restore to the smaller disk fails

Rescuezilla backs up each partition individually with Partclone and also saves the partition table. During restoration, it writes this table unchanged to the target disk. Rescuezilla cannot shrink a partition. Before starting, it therefore checks whether the final partition fits entirely on the target disk and otherwise stops with this message:

```text
The source partition table's final partition (/dev/nvme0n1p4:
1000203091968 bytes) must refer to a region completely within
the destination disk (512110190592 bytes).
```

A typical Windows installation has four partitions: the EFI system partition (200 MB), the Microsoft reserved partition (16 MB), C:, and the recovery partition with Windows RE (904 MiB here). Windows places the recovery partition at the end of the disk. Therefore, shrinking C: is not enough: the recovery partition must also be moved forward, directly behind C:.

The process described by the Rescuezilla wiki for this case is: shrink the source disk with GParted, create a new image, and restore that image. An image created earlier from the unchanged disk remains as a backup until the new system is running.

## Step 1: Calculate the target size

An SSD sold as 512 GB has 512'110'190'592 bytes, which is 476.9 GiB (Rescuezilla displays this value as well). EFI, MSR, and the recovery partition take up a good 1.1 GiB in total. That leaves just under 475 GiB for C:. With some reserve, **470 GiB = 481'280 MiB** is a sensible target value. The approximately 6 GB left free at the end of the new SSD are insignificant.

The used data must be below this value. This command in an elevated PowerShell shows how far Windows could shrink a volume:

```powershell
$s = Get-PartitionSupportedSize -DriveLetter C
"{0:N1} GB" -f ($s.SizeMin / 1GB)
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-DriveLetter C` | Volume whose possible sizes are queried |
| `SizeMin` | Smallest size to which Windows itself could shrink the volume |
| `"{0:N1} GB" -f` | Formats the byte value as gigabytes with one decimal place |

</details>

If `SizeMin` is significantly above the amount of used data, unmovable files (page file, restore points, MFT) are preventing shrinking with Windows built-in tools. GParted moves this data when shrinking, so that limit does not apply there.

## Step 2: Disable BitLocker and hibernation

GParted can neither read nor shrink a BitLocker-encrypted volume, and Partclone cannot back it up as NTFS. Check the status in an elevated PowerShell:

```powershell
manage-bde -status C:
```

If it does not say `Fully Decrypted`, disable BitLocker and wait for decryption to finish:

```powershell
manage-bde -off C:
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-status` | Shows the volume's encryption percentage, method, and protection status |
| `-off` | Fully decrypts the volume and removes key protectors |
| `C:` | Positional argument: the affected volume |

</details>

Decryption runs in the background and may take an hour or longer depending on the amount of data. On devices managed through Intune, a policy may re-enable BitLocker after a short time. Therefore, check the status again immediately before starting GParted.

Hibernation must also be disabled. With Fast Startup enabled, Windows does not fully shut down but saves the kernel state in `hiberfil.sys`. NTFS is then considered still in use, and GParted refuses any changes. One command disables both hibernation and Fast Startup and deletes `hiberfil.sys`:

```powershell
powercfg /h off
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `/h off` | Short form of `/hibernate off`: disables hibernation and Fast Startup; `hiberfil.sys` is removed |

</details>

## Step 3: Boot Rescuezilla

Windows can direct the next startup straight to the startup menu with advanced options. There, select “Use a device” and the USB drive with Rescuezilla:

```powershell
shutdown /r /o /t 0
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `/r` | Restarts instead of shutting down |
| `/o` | Starts in advanced startup options (Windows RE); only together with `/r` |
| `/t 0` | Wait time in seconds before execution |

</details>

Alternatively, the firmware boot menu opens during power-on; on ASRock boards, use F11. If “Use a device” is missing, `shutdown /r /fw /t 0` takes you directly to UEFI Setup, where you can select the boot drive for the next startup.

## Step 4: Shrink C: and move the recovery partition

On the Rescuezilla desktop, start **Partition Editor** (GParted), not Rescuezilla itself.

1.  Select the source disk in the upper-right corner. Pay attention to its size (931.51 GiB here) so you do not accidentally edit the external backup disk or a second internal disk.

2.  Right-click the NTFS partition with C: → **Resize/Move**. Enter the target value in the **New size (MiB)** field, here `481280`, and confirm with the Tab key. Leave **Free space preceding** unchanged. Apply with **Resize/Move**.

3.  Below C:, there is now a line `unallocated`, followed by the recovery partition (NTFS, around 900 MiB, flags `hidden, diag`). Right-click the recovery partition → **Resize/Move**.

4.  Delete the value in **Free space preceding (MiB)**, enter `0`, and confirm with Tab. **New size** stays the same; **Free space following** changes to the entire free area. If `1` appears after pressing Tab instead of `0`, that is alignment to whole MiB and is fine. Apply with **Resize/Move**.

5.  Confirm the warning that moving may prevent booting with OK. It applies to partitions with bootloaders; Windows boots from the EFI system partition, which remains unchanged.

6.  The order is now: EFI, MSR, C:, recovery partition, `unallocated`. At this point, changes are only queued. Only clicking the green check mark (**Apply All Operations**) performs them.

Windows RE finds its partition again after it is moved because it retains its partition number. `reagentc /info` still shows `Enabled` afterward, with the path `harddisk0\partition4\Recovery\WindowsRE`.

The Rescuezilla guide shrinks only the last partition. That is sufficient if C: is the last partition. In a standard Windows 10 or 11 installation, the recovery partition is behind it, making the move mandatory.

## Step 5: Remove the dirty flag

The next backup fails at C: after shrinking with this message:

```text
ntfsclone-ng.c: NTFS Volume '/dev/nvme0n1p3' is scheduled for a check
or it was shutdown uncleanly. Please boot Windows or fix it by fsck.
```

The cause is intentional: `ntfsresize`, which GParted uses for NTFS, marks the file system for checking before every size change and leaves that mark in place. According to the man page, Windows should run `chkdsk` at the next startup. In the case described, however, the flag remained set after a normal Windows startup, and `Get-Volume C` reported `Full Repair Needed`. Check the status in an elevated PowerShell:

```powershell
fsutil dirty query C:
```

If the command reports `Volume - C: is Dirty`, have the check run at the next startup:

```powershell
chkdsk C: /f
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `dirty query C:` | fsutil: checks whether the volume's dirty bit is set |
| `C:` | chkdsk: the volume to check |
| `/f` | Fixes errors found; on the system volume, schedules the check for the next startup |

</details>

Answer the question asking whether the check should be performed at the next restart with Y and restart Windows normally. Afterward, `fsutil dirty query C:` must show the message `is NOT Dirty`, and `Get-Volume C` must again report `Healthy`. Only then start Rescuezilla again.

## Step 6: Create and check the new image

Now use Rescuezilla to create a new image of the shrunken disk. The image size remains virtually the same as the first backup (around 214 GB here), because Partclone backs up only used blocks. Rescuezilla places each backup in its own timestamped folder, such as `2026-09-29-1343-img-rescuezilla`.

All images then appear side by side in the image selection list. The **Partitions** column shows their sizes: the image from before shrinking contains `ntfs 930.4GB`, while the new one contains `ntfs 470GB`. A failed backup attempt appears with a yellow warning triangle; it contains the old partition table and triggers exactly the error message from the first section during restoration. Delete such folders so they are not selected accidentally.

You can also determine in advance whether an image fits the target disk by looking at the `<disk>-pt.parted` file in the image folder. It contains the backed-up partition table in 512-byte sectors:

```text
Number  Start       End         Size        File system  Name
 1      2048s       411647s     409600s     fat32        EFI system partition
 2      411648s     444415s     32768s                   Microsoft reserved partition
 3      444416s     986105855s  985661440s  ntfs         Basic data partition
 4      986105856s  987957247s  1851392s    ntfs
```

The end of the final partition (987'957'247 + 1 sectors × 512 bytes = 505.8 GB) is below the target disk's 512.1 GB. The image fits.

## Step 7: Restore to the new SSD

1.  In Rescuezilla, select **Restore**, the drive containing the images, and then the **new** image (without a warning triangle and with the shrunken C: partition).

2.  Select the new SSD as the target. Check the model and size twice: restoration completely overwrites the target disk.

3.  Keep all partitions selected, as well as **Overwrite partition table**, and start the restore.

4.  Then boot from the new SSD. The easiest option is to disconnect the old disk first; otherwise, select the new SSD in the firmware boot menu.

## Follow-up tasks

On the new drive, re-enable what was disabled for the migration. `powercfg /h on` enables hibernation again. Enable BitLocker through Settings → Privacy & security → Device encryption or with `manage-bde -on C:`; then verify with `manage-bde -protectors -get C:` that the recovery key has been backed up. If Secure Boot was disabled in UEFI to boot Rescuezilla, enable it again; while it is disabled, BitLocker logs event 810 at every startup.

You can delete the image of the unchanged 1 TB disk once Windows boots cleanly from the new SSD and all data is complete.

## Sources

1.  [Rescuezilla Wiki: Restoring to a smaller disk](https://github.com/rescuezilla/rescuezilla/wiki/HOWTO:-Restoring-to-a-smaller-disk.-Eg,-1000GB-HDD-to-500GB-SSD): official procedure with GParted and the reason for the restoration limitation.

2.  [Rescuezilla](https://rescuezilla.com/): project page with the live system download.

3.  [GParted Manual](https://gparted.org/display-doc.php?name=help-manual): using Resize/Move, MiB alignment, and performing queued operations.

4.  [ntfsresize(8), Ubuntu Manpage](https://manpages.ubuntu.com/manpages/noble/man8/ntfsresize.8.html): the tool behind NTFS shrinking in GParted; describes the deliberately set check marker.

5.  [Microsoft Learn: chkdsk](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/chkdsk): `/f` parameter and scheduling the check on the system volume.

6.  [Microsoft Learn: fsutil dirty](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/fsutil-dirty): querying and meaning of the dirty bit.

7.  [Microsoft Learn: manage-bde status](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-status): BitLocker status query.

8.  [Microsoft Learn: manage-bde off](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-off): decrypting a volume.

9.  [Microsoft Learn: Powercfg command-line options](https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/powercfg-command-line-options): `/hibernate` option and its effect on Fast Startup.

10.  [Microsoft Learn: Get-PartitionSupportedSize](https://learn.microsoft.com/en-us/powershell/module/storage/get-partitionsupportedsize): minimum and maximum partition sizes from Windows' perspective.

11.  [Microsoft Learn: REAgentC command-line options](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/reagentc-command-line-options): checking the status and location of Windows RE.

12.  [Microsoft Learn: shutdown](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/shutdown): `/o` and `/fw` parameters for booting into advanced options or UEFI, respectively.
