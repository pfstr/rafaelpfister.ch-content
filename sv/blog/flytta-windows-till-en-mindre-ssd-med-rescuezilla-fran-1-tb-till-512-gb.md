---
title: "Flytta Windows till en mindre SSD med Rescuezilla: från 1 TB till 512 GB"
navTitle: "Flytta till mindre SSD"
description: "Rescuezilla återställer endast en avbildning till en disk som räcker minst till slutet av den sista partitionen. Guiden visar, med en NVMe på 1 TB som exempel, hur du krymper C: med GParted, flyttar återställningspartitionen, tar bort Dirty-flaggan och återställer den nya avbildningen till en SSD på 512 GB."
date: "2026-09-29"
kategorie: "PC & hårdvara"
timeToRead: "9 min läsning"
themen:
  - pc-hardware
produkte:
  - "windows-client"
protokolle:
  - "backup-dr"
  - "migration"
slug: "flytta-windows-till-en-mindre-ssd-med-rescuezilla-fran-1-tb-till-512-gb"
translationId: "article-6947213279383f27"
translationOf: windows-kleinere-ssd-rescuezilla
url: https://rafaelpfister.ch/sv/blog/flytta-windows-till-en-mindre-ssd-med-rescuezilla-fran-1-tb-till-512-gb
translationSourceHash: ad4485736511552671c4e469ddaa672e9439cffb3e74d43bd8dd7333eaf662d3
translationModel: gpt-5.6-terra
translatedAt: 2026-09-30T09:56:17.202Z
translationReview: automatic
---

Rescuezilla är ett kostnadsfritt livesystem för säkerhetskopiering och återställning av hela diskar, kompatibelt med Clonezillas avbildningsformat (presentation av verktyget och grundläggande arbetsflöde: [Rescuezilla: Migrera Windows till en ny SSD](/blog/rescuezilla-windows-migration)). Så länge måldisken är lika stor eller större räcker säkerhetskopiering och återställning. Är den mindre avbryts återställningen, även om de använda data skulle få plats. Den här artikeln dokumenterar flytten av ett Windows 11-system från en NVMe på 1 TB (C: med 930 GB, varav omkring 350 GB används) till en NVMe på 512 GB, inklusive felmeddelandena som uppstår på vägen.

## Varför återställningen till den mindre disken misslyckas

Rescuezilla säkerhetskopierar varje partition separat med Partclone och sparar dessutom partitionstabellen. Vid återställningen skriver programmet denna tabell oförändrad till måldisken. Rescuezilla kan inte krympa en partition. Innan start kontrollerar programmet därför om den sista partitionen ryms helt på måldisken och avbryter annars med följande meddelande:

```text
The source partition table's final partition (/dev/nvme0n1p4:
1000203091968 bytes) must refer to a region completely within
the destination disk (512110190592 bytes).
```

En typisk Windows-installation har fyra partitioner: EFI-systempartitionen (200 MB), den Microsoft-reserverade partitionen (16 MB), C: och återställningspartitionen med Windows RE (här 904 MiB). Windows placerar återställningspartitionen i slutet av disken. Det räcker alltså inte att krympa C:: även återställningspartitionen måste flyttas fram, direkt efter C:.

Arbetsflödet som Rescuezilla-wikin beskriver för detta fall är: krymp källdisken med GParted, skapa en ny avbildning och återställ denna avbildning. En tidigare skapad avbildning av den oförändrade disken behålls som säkerhetskopia tills det nya systemet fungerar.

## Steg 1: Beräkna målstorleken

En SSD som säljs som 512 GB har 512'110'190'592 byte, vilket motsvarar 476,9 GiB (Rescuezilla visar också detta värde). EFI, MSR och återställningspartitionen tar tillsammans drygt 1,1 GiB. För C: återstår därmed knappt 475 GiB. Med lite marginal är **470 GiB = 481'280 MiB** ett lämpligt målvärde. De cirka 6 GB som lämnas lediga i slutet av den nya SSD:n spelar ingen roll.

De använda data måste understiga detta värde. Följande kommando i PowerShell med administratörsbehörighet visar hur långt Windows skulle kunna krympa en volym:

```powershell
$s = Get-PartitionSupportedSize -DriveLetter C
"{0:N1} GB" -f ($s.SizeMin / 1GB)
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `-DriveLetter C` | Volym vars möjliga storlekar efterfrågas |
| `SizeMin` | Minsta storlek som Windows självt skulle kunna krympa volymen till |
| `"{0:N1} GB" -f` | Formaterar bytevärdet som gigabyte med en decimal |

</details>

Om `SizeMin` ligger betydligt över mängden använda data blockerar orubbliga filer (växlingsfil, återställningspunkter, MFT) krympningen med Windows inbyggda verktyg. GParted flyttar med dessa data vid krympningen, så den gränsen gäller inte där.

## Steg 2: Stäng av BitLocker och viloläge

GParted kan varken läsa eller krympa en BitLocker-krypterad volym, och Partclone kan inte säkerhetskopiera den som NTFS. Kontrollera statusen i PowerShell med administratörsbehörighet:

```powershell
manage-bde -status C:
```

Om det inte står `Fully Decrypted`, stänger du av BitLocker och väntar tills dekrypteringen är klar:

```powershell
manage-bde -off C:
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `-status` | Visar volymens krypteringsgrad, metod och skyddsstatus |
| `-off` | Dekrypterar volymen helt och tar bort nyckelskydden |
| `C:` | Positionsargument: den berörda volymen |

</details>

Dekrypteringen körs i bakgrunden och kan, beroende på datamängd, ta en timme eller längre. På enheter som hanteras via Intune kan en princip slå på BitLocker igen efter kort tid. Kontrollera därför statusen ytterligare en gång direkt innan du startar GParted.

Dessutom måste viloläget vara avstängt. Med snabbstart aktiverad stängs Windows inte av helt, utan sparar kärnans tillstånd i `hiberfil.sys`. NTFS betraktas då fortfarande som i bruk, och GParted vägrar göra ändringar. Ett kommando stänger av både viloläge och snabbstart samt raderar `hiberfil.sys`:

```powershell
powercfg /h off
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `/h off` | Kortform av `/hibernate off`: inaktiverar viloläge och snabbstart, `hiberfil.sys` tas bort |

</details>

## Steg 3: Starta Rescuezilla

Windows kan styra nästa start direkt till startmenyn med avancerade alternativ. Där väljer du ”Använd en enhet” och USB-minnet med Rescuezilla:

```powershell
shutdown /r /o /t 0
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `/r` | Startar om i stället för att stänga av |
| `/o` | Startar i avancerade startalternativ (Windows RE); endast tillsammans med `/r` |
| `/t 0` | Väntetid i sekunder före körning |

</details>

Alternativt öppnas firmwarets startmeny när datorn slås på, på ASRock-kort med F11. Om ”Använd en enhet” saknas leder `shutdown /r /fw /t 0` direkt till UEFI-inställningarna, där startenheten för nästa start kan väljas.

## Steg 4: Krymp C: och flytta återställningspartitionen

Starta **Partition Editor** (GParted) på Rescuezilla-skrivbordet, inte Rescuezilla självt.

1.  Välj källdisken uppe till höger. Kontrollera storleken (här 931.51 GiB), så att du inte av misstag redigerar den externa säkerhetskopieringsdisken eller en andra intern disk.

2.  Högerklicka på NTFS-partitionen med C: → **Resize/Move**. Ange målvärdet i fältet **New size (MiB)**, här `481280`, och bekräfta med tabbtangenten. **Free space preceding** förblir oförändrat. Bekräfta med **Resize/Move**.

3.  Under C: visas nu en rad `unallocated`, under den återställningspartitionen (NTFS, omkring 900 MiB, flaggor `hidden, diag`). Högerklicka på återställningspartitionen → **Resize/Move**.

4.  Radera värdet i fältet **Free space preceding (MiB)**, ange `0` och bekräfta med tabbtangenten. **New size** förblir samma, **Free space following** hoppar till hela det lediga området. Om `1` visas efter att du tryckt på Tab i stället för `0`, beror det på justeringen till hela MiB och är i sin ordning. Bekräfta med **Resize/Move**.

5.  Bekräfta en varning om att flytten kan förhindra uppstart med OK. Den gäller partitioner med startladdare; Windows startar från EFI-systempartitionen, som lämnas oförändrad.

6.  Ordningen är nu: EFI, MSR, C:, återställningspartition, `unallocated`. Hittills är ändringarna bara schemalagda. Först ett klick på den gröna bocken (**Apply All Operations**) genomför ändringarna.

Windows RE hittar sin partition igen efter flytten eftersom den behåller sitt partitionsnummer. `reagentc /info` visar därefter fortfarande `Enabled` med sökvägen `harddisk0\partition4\Recovery\WindowsRE`.

Rescuezilla-guiden krymper bara den sista partitionen. Det räcker om C: är den sista partitionen. I en standardinstallation av Windows 10 eller 11 ligger återställningspartitionen efter den, och då är flytten obligatorisk.

## Steg 5: Ta bort Dirty-flaggan

Nästa säkerhetskopiering avbryts efter krympningen vid C: med följande meddelande:

```text
ntfsclone-ng.c: NTFS Volume '/dev/nvme0n1p3' is scheduled for a check
or it was shutdown uncleanly. Please boot Windows or fix it by fsck.
```

Orsaken är avsiktlig: `ntfsresize`, som GParted använder för NTFS, markerar filsystemet för kontroll före varje storleksändring och lämnar kvar markeringen. Enligt manualsidan ska Windows köra `chkdsk` vid nästa start. I det beskrivna fallet var flaggan dock fortfarande satt efter en normal Windows-start, och `Get-Volume C` rapporterade `Full Repair Needed`. Du kontrollerar tillståndet i PowerShell med administratörsbehörighet:

```powershell
fsutil dirty query C:
```

Om kommandot rapporterar `Volume - C: is Dirty`, låter du kontrollen köras vid nästa start:

```powershell
chkdsk C: /f
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `dirty query C:` | fsutil: frågar om volymens Dirty-bit är satt |
| `C:` | chkdsk: volymen som ska kontrolleras |
| `/f` | Åtgärdar hittade fel; för systemvolymen schemaläggs kontrollen till nästa start |

</details>

Svara J på frågan om kontrollen ska utföras vid nästa omstart och starta om Windows normalt. Därefter måste `fsutil dirty query C:` visa meddelandet `is NOT Dirty`, och `Get-Volume C` rapporterar åter `Healthy`. Först då startar du Rescuezilla igen.

## Steg 6: Skapa och kontrollera en ny avbildning

Använd nu Rescuezilla för att skapa en ny avbildning av den krympta disken. Avbildningens storlek förblir i praktiken densamma som vid den första säkerhetskopieringen (här omkring 214 GB), eftersom Partclone bara säkerhetskopierar använda block. Rescuezilla placerar varje säkerhetskopia i en egen mapp med tidsstämpel, exempelvis `2026-09-29-1343-img-rescuezilla`.

I listan för val av avbildning visas sedan alla avbildningar bredvid varandra. Kolumnen **Partitions** visar storlekarna: avbildningen före krympningen innehåller `ntfs 930.4GB`, den nya `ntfs 470GB`. Ett avbrutet säkerhetskopieringsförsök visas med en gul varningstriangel; det innehåller den gamla partitionstabellen och utlöser vid återställning exakt felmeddelandet från första avsnittet. Radera sådana mappar så att de inte väljs av misstag.

Du kan också i förväg kontrollera om en avbildning ryms på måldisken i filen `<disk>-pt.parted` i avbildningsmappen. Den innehåller den säkerhetskopierade partitionstabellen i sektorer om 512 byte:

```text
Number  Start       End         Size        File system  Name
 1      2048s       411647s     409600s     fat32        EFI system partition
 2      411648s     444415s     32768s                   Microsoft reserved partition
 3      444416s     986105855s  985661440s  ntfs         Basic data partition
 4      986105856s  987957247s  1851392s    ntfs
```

Slutet på den sista partitionen (987'957'247 + 1 sektorer × 512 byte = 505,8 GB) ligger under måldiskens 512,1 GB. Avbildningen passar.

## Steg 7: Återställ till den nya SSD:n

1.  Välj **Restore** i Rescuezilla, sedan enheten med avbildningarna och därefter den **nya** avbildningen (utan varningstriangel, med den krympta C:-partitionen).

2.  Välj den nya SSD:n som mål. Kontrollera modell och storlek två gånger: återställningen skriver över måldisken helt.

3.  Låt alla partitioner vara markerade, liksom **Overwrite partition table**, och starta återställningen.

4.  Starta sedan från den nya SSD:n. Enklast är att koppla bort den gamla disken först; annars väljer du den nya SSD:n i firmwarets startmeny.

## Efterarbete

På den nya enheten aktiverar du igen det som inaktiverades för flytten. Viloläget aktiveras med `powercfg /h on`. Du aktiverar BitLocker via Inställningar → Sekretess och säkerhet → Enhetskryptering eller med `manage-bde -on C:`; kontrollera därefter med `manage-bde -protectors -get C:` att återställningsnyckeln är sparad. Om Secure Boot inaktiverades i UEFI för att starta Rescuezilla ska du aktivera det igen; så länge det är avstängt loggar BitLocker händelse 810 vid varje start.

Du kan radera avbildningen av den oförändrade disken på 1 TB så snart Windows startar korrekt från den nya SSD:n och alla data finns där.

## Källor

1.  [Rescuezilla Wiki: Återställa till en mindre disk](https://github.com/rescuezilla/rescuezilla/wiki/HOWTO:-Restoring-to-a-smaller-disk.-Eg,-1000GB-HDD-to-500GB-SSD): officiellt arbetsflöde med GParted och orsaken till begränsningen vid återställning.

2.  [Rescuezilla](https://rescuezilla.com/): projektsida med nedladdning av livesystemet.

3.  [GParted Manual](https://gparted.org/display-doc.php?name=help-manual): hantering av Resize/Move, justering till MiB och genomförande av schemalagda åtgärder.

4.  [ntfsresize(8), Ubuntu Manpage](https://manpages.ubuntu.com/manpages/noble/man8/ntfsresize.8.html): verktyget bakom NTFS-krympning i GParted; beskriver den avsiktligt satta kontrollmarkeringen.

5.  [Microsoft Learn: chkdsk](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/chkdsk): parametern `/f` och schemaläggning av kontrollen på systemvolymen.

6.  [Microsoft Learn: fsutil dirty](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/fsutil-dirty): kontroll och betydelse av Dirty-biten.

7.  [Microsoft Learn: manage-bde status](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-status): statuskontroll för BitLocker.

8.  [Microsoft Learn: manage-bde off](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-off): dekryptering av en volym.

9.  [Microsoft Learn: Powercfg command-line options](https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/powercfg-command-line-options): alternativet `/hibernate` och dess påverkan på snabbstart.

10.  [Microsoft Learn: Get-PartitionSupportedSize](https://learn.microsoft.com/en-us/powershell/module/storage/get-partitionsupportedsize): minsta och största partitionsstorlek ur Windows perspektiv.

11.  [Microsoft Learn: REAgentC command-line options](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/reagentc-command-line-options): kontrollera status och lagringsplats för Windows RE.

12.  [Microsoft Learn: shutdown](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/shutdown): parametrarna `/o` och `/fw` för start i avancerade alternativ respektive UEFI.
