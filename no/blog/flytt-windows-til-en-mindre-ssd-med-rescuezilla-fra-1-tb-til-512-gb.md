---
title: "Flytt Windows til en mindre SSD med Rescuezilla: fra 1 TB til 512 GB"
navTitle: "Flytting til mindre SSD"
description: "Rescuezilla gjenoppretter bare et image til en disk som rekker minst til slutten av den siste partisjonen. Veiledningen viser, med en 1 TB NVMe som eksempel, hvordan du krymper C: med GParted, flytter gjenopprettingspartisjonen, fjerner Dirty-flagget og gjenoppretter det nye imaget til en 512 GB SSD."
date: "2026-09-29"
kategorie: "PC og maskinvare"
timeToRead: "9 min lesetid"
themen:
  - pc-hardware
produkte:
  - "windows-client"
protokolle:
  - "backup-dr"
  - "migration"
slug: "flytt-windows-til-en-mindre-ssd-med-rescuezilla-fra-1-tb-til-512-gb"
translationId: "article-6947213279383f27"
translationOf: windows-kleinere-ssd-rescuezilla
url: https://rafaelpfister.ch/no/blog/flytt-windows-til-en-mindre-ssd-med-rescuezilla-fra-1-tb-til-512-gb
translationSourceHash: ad4485736511552671c4e469ddaa672e9439cffb3e74d43bd8dd7333eaf662d3
translationModel: gpt-5.6-terra
translatedAt: 2026-09-30T09:57:02.440Z
translationReview: automatic
---

Rescuezilla er et gratis live-system for sikkerhetskopiering og gjenoppretting av hele datalagringsenheter, kompatibelt med imageformatet til Clonezilla (presentasjon av verktøyet og grunnleggende fremgangsmåte: [Rescuezilla: Migrer Windows til en ny SSD](/blog/rescuezilla-windows-migration)). Så lenge måldisken er like stor eller større, er sikkerhetskopiering og gjenoppretting tilstrekkelig. Er den mindre, avbrytes gjenopprettingen selv om de brukte dataene ville fått plass på den. Denne artikkelen dokumenterer flyttingen av et Windows 11-system fra en 1 TB NVMe (C: med 930 GB, hvorav rundt 350 GB er brukt) til en 512 GB NVMe, inkludert feilmeldingene som oppstår underveis.

## Hvorfor gjenopprettingen til den mindre disken mislykkes

Rescuezilla sikkerhetskopierer hver partisjon separat med Partclone og lagrer i tillegg partisjonstabellen. Ved gjenoppretting skriver det denne tabellen uendret til måldisken. Rescuezilla kan ikke krympe en partisjon. Før oppstart kontrollerer det derfor om den siste partisjonen ligger helt på måldisken, og avbryter ellers med denne meldingen:

```text
The source partition table's final partition (/dev/nvme0n1p4:
1000203091968 bytes) must refer to a region completely within
the destination disk (512110190592 bytes).
```

En typisk Windows-installasjon har fire partisjoner: EFI-systempartisjonen (200 MB), den Microsoft-reserverte partisjonen (16 MB), C: og gjenopprettingspartisjonen med Windows RE (her 904 MiB). Windows plasserer gjenopprettingspartisjonen på slutten av disken. Det er derfor ikke nok å krympe C:: Gjenopprettingspartisjonen må også flyttes fremover, rett bak C:.

Fremgangsmåten som Rescuezilla-wikien beskriver for dette tilfellet, er: Krymp kildedisken med GParted, opprett et nytt image, og gjenopprett dette imaget. Et tidligere opprettet image av den uendrede disken beholdes som sikkerhetskopi til det nye systemet kjører.

## Trinn 1: Beregn målstørrelsen

En SSD som selges som 512 GB, har 512'110'190'592 byte, som er 476,9 GiB (Rescuezilla viser også denne verdien). EFI, MSR og gjenopprettingspartisjonen trekker fra til sammen godt 1,1 GiB. For C: gjenstår dermed knapt 475 GiB. Med litt reserve er **470 GiB = 481'280 MiB** en fornuftig målverdi. De rundt 6 GB som blir ledige på slutten av den nye SSD-en, har ingen betydning.

De brukte dataene må være under denne verdien. Denne kommandoen i PowerShell med administratorrettigheter viser hvor langt Windows selv kan krympe et volum:

```powershell
$s = Get-PartitionSupportedSize -DriveLetter C
"{0:N1} GB" -f ($s.SizeMin / 1GB)
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-DriveLetter C` | Volumet det skal hentes informasjon om mulige størrelser for |
| `SizeMin` | Minste størrelse Windows selv kan krympe volumet til |
| `"{0:N1} GB" -f` | Formaterer byteverdien som gigabyte med én desimal |

</details>

Ligger `SizeMin` betydelig over mengden brukte data, blokkerer uflyttbare filer (sidevekslingsfil, gjenopprettingspunkter, MFT) krympingen med Windows' innebygde verktøy. GParted flytter disse dataene ved krymping, så grensen gjelder ikke der.

## Trinn 2: Slå av BitLocker og dvalemodus

GParted kan verken lese eller krympe et BitLocker-kryptert volum, og Partclone kan ikke sikkerhetskopiere det som NTFS. Kontroller statusen i PowerShell med administratorrettigheter:

```powershell
manage-bde -status C:
```

Hvis det ikke står `Fully Decrypted`, slår du av BitLocker og venter til dekrypteringen er fullført:

```powershell
manage-bde -off C:
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-status` | Viser krypteringsgrad, metode og beskyttelsesstatus for volumet |
| `-off` | Dekrypterer volumet fullstendig og fjerner nøkkelbeskyttelsen |
| `C:` | Posisjonsargument: det berørte volumet |

</details>

Dekrypteringen kjører i bakgrunnen og kan ta en time eller mer, avhengig av datamengden. På enheter som administreres via Intune, kan en policy slå på BitLocker igjen etter kort tid. Kontroller derfor statusen en gang til umiddelbart før du starter GParted.

I tillegg må dvalemodus være slått av. Med hurtigstart aktivert slår ikke Windows seg helt av, men lagrer kjernetilstanden i `hiberfil.sys`. NTFS anses da fortsatt som i bruk, og GParted nekter alle endringer. Én kommando slår av både dvalemodus og hurtigstart og sletter `hiberfil.sys`:

```powershell
powercfg /h off
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `/h off` | Kortform av `/hibernate off`: deaktiverer dvalemodus og hurtigstart, `hiberfil.sys` fjernes |

</details>

## Trinn 3: Start Rescuezilla

Windows kan styre neste oppstart direkte til oppstartsmenyen med avanserte alternativer. Der velger du «Bruk en enhet» og USB-pinnen med Rescuezilla:

```powershell
shutdown /r /o /t 0
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `/r` | Omstart i stedet for avslutning |
| `/o` | Starter i avanserte oppstartsalternativer (Windows RE); bare sammen med `/r` |
| `/t 0` | Ventetid i sekunder før utførelse |

</details>

Alternativt åpnes fastvarens oppstartsmeny når maskinen slås på, på ASRock-hovedkort med F11. Hvis «Bruk en enhet» mangler, går `shutdown /r /fw /t 0` direkte til UEFI-oppsettet, der oppstartsstasjonen kan velges for neste oppstart.

## Trinn 4: Krymp C: og flytt gjenopprettingspartisjonen

Start **Partition Editor** (GParted) på Rescuezilla-skrivebordet, ikke Rescuezilla selv.

1.  Velg kildedisken øverst til høyre. Se nøye på størrelsen (her 931.51 GiB), slik at du ikke ved en feil redigerer den eksterne sikkerhetskopidisken eller en annen intern disk.

2.  Høyreklikk på NTFS-partisjonen med C: → **Resize/Move**. Angi målverdien i feltet **New size (MiB)**, her `481280`, og bekreft med Tab-tasten. **Free space preceding** forblir uendret. Bekreft med **Resize/Move**.

3.  Under C: står det nå en linje med `unallocated`, og under den ligger gjenopprettingspartisjonen (NTFS, rundt 900 MiB, flagg `hidden, diag`). Høyreklikk på gjenopprettingspartisjonen → **Resize/Move**.

4.  Slett verdien i feltet **Free space preceding (MiB)**, skriv inn `0` og bekreft med Tab. **New size** forblir den samme, mens **Free space following** hopper til hele det ledige området. Hvis det står `1` i stedet for `0` etter Tab, skyldes det justering til hele MiB og er i orden. Bekreft med **Resize/Move**.

5.  Bekreft en advarsel om at flyttingen kan hindre oppstart, med OK. Den gjelder partisjoner med oppstartslaster; Windows starter fra EFI-systempartisjonen, som forblir uendret.

6.  Rekkefølgen er nå: EFI, MSR, C:, gjenopprettingspartisjon, `unallocated`. Så langt er endringene bare planlagt. Først et klikk på den grønne haken (**Apply All Operations**) utfører dem.

Windows RE finner partisjonen sin igjen etter flyttingen fordi den beholder partisjonsnummeret. `reagentc /info` viser fortsatt `Enabled` med banen `harddisk0\partition4\Recovery\WindowsRE` etterpå.

Rescuezilla-veiledningen krymper bare den siste partisjonen. Det er nok dersom C: er den siste partisjonen. I en standardinstallasjon av Windows 10 eller 11 ligger gjenopprettingspartisjonen etter den, og da er flytting obligatorisk.

## Trinn 5: Fjern Dirty-flagget

Neste sikkerhetskopiering avbrytes ved C: etter krympingen med denne meldingen:

```text
ntfsclone-ng.c: NTFS Volume '/dev/nvme0n1p3' is scheduled for a check
or it was shutdown uncleanly. Please boot Windows or fix it by fsck.
```

Årsaken er tilsiktet: `ntfsresize`, som GParted bruker for NTFS, markerer filsystemet for kontroll før hver størrelsesendring og lar denne markeringen stå. Ifølge man-siden skal Windows kjøre `chkdsk` ved neste oppstart. I det beskrevne tilfellet var flagget fortsatt satt etter en vanlig Windows-oppstart, og `Get-Volume C` rapporterte `Full Repair Needed`. Du kan kontrollere tilstanden i PowerShell med administratorrettigheter:

```powershell
fsutil dirty query C:
```

Hvis kommandoen rapporterer `Volume - C: is Dirty`, lar du kontrollen kjøre ved neste oppstart:

```powershell
chkdsk C: /f
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `dirty query C:` | fsutil: spør om volumets Dirty-bit er satt |
| `C:` | chkdsk: volumet som skal kontrolleres |
| `/f` | Retter funne feil; på systemvolumet planlegges kontrollen til neste oppstart |

</details>

Svar J på spørsmålet om kontrollen skal utføres ved neste omstart, og start Windows normalt på nytt. Etterpå må `fsutil dirty query C:` vise meldingen `is NOT Dirty`, og `Get-Volume C` rapporterer igjen `Healthy`. Først da starter du Rescuezilla på nytt.

## Trinn 6: Opprett og kontroller et nytt image

Bruk nå Rescuezilla til å opprette et nytt image av den krympede disken. Imagestørrelsen blir praktisk talt den samme som ved første sikkerhetskopiering (her rundt 214 GB), fordi Partclone bare sikkerhetskopierer brukte blokker. Rescuezilla lagrer hver sikkerhetskopi i sin egen mappe med tidsstempel, for eksempel `2026-09-29-1343-img-rescuezilla`.

I listen for imagevalg står deretter alle imagene ved siden av hverandre. Kolonnen **Partitions** viser størrelsene: Imagets før krympingen inneholder `ntfs 930.4GB`, det nye `ntfs 470GB`. Et avbrutt sikkerhetskopieringsforsøk vises med en gul varseltrekant; det inneholder den gamle partisjonstabellen og utløser ved gjenoppretting nøyaktig feilmeldingen fra første avsnitt. Slett slike mapper så de ikke velges ved en feil.

Du kan også på forhånd se om et image passer på måldisken i filen `<disk>-pt.parted` i imagemappen. Den inneholder den sikkerhetskopierte partisjonstabellen i sektorer på 512 byte:

```text
Number  Start       End         Size        File system  Name
 1      2048s       411647s     409600s     fat32        EFI system partition
 2      411648s     444415s     32768s                   Microsoft reserved partition
 3      444416s     986105855s  985661440s  ntfs         Basic data partition
 4      986105856s  987957247s  1851392s    ntfs
```

Slutten av den siste partisjonen (987'957'247 + 1 sektorer × 512 byte = 505,8 GB) ligger under måldiskens 512,1 GB. Imaget passer.

## Trinn 7: Gjenopprett til den nye SSD-en

1.  Velg **Restore** i Rescuezilla, stasjonen med imagene og deretter det **nye** imaget (uten varseltrekant, med den krympede C:-partisjonen).

2.  Velg den nye SSD-en som mål. Kontroller modell og størrelse to ganger: Gjenopprettingen overskriver måldisken fullstendig.

3.  La alle partisjoner være avkrysset, det samme gjelder **Overwrite partition table**, og start gjenopprettingen.

4.  Start deretter fra den nye SSD-en. Det enkleste er å koble fra den gamle disken på forhånd; ellers velger du den nye SSD-en i fastvarens oppstartsmeny.

## Etterarbeid

På den nye stasjonen slår du på igjen det som ble deaktivert for flyttingen. Dvalemodus aktiveres med `powercfg /h on`. BitLocker aktiverer du via Innstillinger → Personvern og sikkerhet → Enhetskryptering eller med `manage-bde -on C:`; kontroller deretter med `manage-bde -protectors -get C:` at gjenopprettingsnøkkelen er sikkerhetskopiert. Hvis Secure Boot ble deaktivert i UEFI for å starte Rescuezilla, slår du det på igjen; så lenge det er av, logger BitLocker hendelse 810 ved hver oppstart.

Du kan slette imaget av den uendrede 1 TB-disken så snart Windows starter problemfritt fra den nye SSD-en og dataene er fullstendige.

## Kilder

1.  [Rescuezilla Wiki: Restoring to a smaller disk](https://github.com/rescuezilla/rescuezilla/wiki/HOWTO:-Restoring-to-a-smaller-disk.-Eg,-1000GB-HDD-to-500GB-SSD): offisiell fremgangsmåte med GParted, årsaken til begrensningen ved gjenoppretting.

2.  [Rescuezilla](https://rescuezilla.com/): prosjektside med nedlasting av live-systemet.

3.  [GParted Manual](https://gparted.org/display-doc.php?name=help-manual): bruk av Resize/Move, justering til MiB og utføring av planlagte operasjoner.

4.  [ntfsresize(8), Ubuntu Manpage](https://manpages.ubuntu.com/manpages/noble/man8/ntfsresize.8.html): verktøyet bak NTFS-krymping i GParted; beskriver den bevisst satte kontrollmarkeringen.

5.  [Microsoft Learn: chkdsk](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/chkdsk): parameteren `/f` og planlegging av kontroll på systemvolumet.

6.  [Microsoft Learn: fsutil dirty](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/fsutil-dirty): spørring og betydning av Dirty-bit.

7.  [Microsoft Learn: manage-bde status](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-status): statuskontroll av BitLocker.

8.  [Microsoft Learn: manage-bde off](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-off): dekryptering av et volum.

9.  [Microsoft Learn: Powercfg command-line options](https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/powercfg-command-line-options): alternativet `/hibernate` og påvirkningen på hurtigstart.

10.  [Microsoft Learn: Get-PartitionSupportedSize](https://learn.microsoft.com/en-us/powershell/module/storage/get-partitionsupportedsize): minste og største partisjonsstørrelse fra Windows' perspektiv.

11.  [Microsoft Learn: REAgentC command-line options](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/reagentc-command-line-options): kontroller status og lagringssted for Windows RE.

12.  [Microsoft Learn: shutdown](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/shutdown): parameterne `/o` og `/fw` for oppstart i henholdsvis avanserte alternativer og UEFI.
