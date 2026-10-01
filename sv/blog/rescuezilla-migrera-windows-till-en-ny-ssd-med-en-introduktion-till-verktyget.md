---
title: "Rescuezilla: migrera Windows till en ny SSD – med en introduktion till verktyget"
navTitle: "Rescuezilla-migrering"
description: "Rescuezilla är en kostnadsfri, Clonezilla-kompatibel imaginglösning med grafiskt gränssnitt. Artikeln presenterar verktyget och visar hela flytten av en Windows-installation till en ny SSD: förberedelser i Windows, startbart USB-minne, säkerhetskopiering, kontroll, återställning och efterarbete."
date: "2026-09-29"
kategorie: "PC och hårdvara"
timeToRead: "10 min läsning"
themen:
  - pc-hardware
produkte:
  - "windows-client"
protokolle:
  - "backup-dr"
  - "migration"
slug: "rescuezilla-migrera-windows-till-en-ny-ssd-med-en-introduktion-till-verktyget"
translationId: "article-d9438c5774d0167c"
translationOf: rescuezilla-windows-migration
url: https://rafaelpfister.ch/sv/blog/rescuezilla-migrera-windows-till-en-ny-ssd-med-en-introduktion-till-verktyget
translationSourceHash: 33320c51c0a77d69bc5804e2c7e69d14c63731a23df6ef55969b37b043a49cd5
translationModel: gpt-5.6-terra
translatedAt: 2026-09-30T09:52:15.911Z
translationReview: automatic
---

Den som vill flytta en Windows-installation till en ny SSD eller en ny dator utan att installera om Windows behöver ett verktyg som säkerhetskopierar och återställer hela disken med alla partitioner. Rescuezilla är ett sådant verktyg: kostnadsfritt, med öppen källkod och med ett grafiskt gränssnitt som går att använda även utan Linux-kunskaper. Den här artikeln presenterar verktyget och beskriver migreringen till en SSD som är lika stor eller större. För en mindre mål-SSD finns en separat guide: [Flytta Windows till en mindre SSD med Rescuezilla](/blog/windows-kleinere-ssd-rescuezilla).

## Vad Rescuezilla är

Rescuezilla är ett livesystem baserat på Ubuntu som startas från ett USB-minne. Det körs oberoende av det installerade operativsystemet och säkerhetskopierar därför även volymer som Windows låser under drift. Projektet uppstod 2019 som en fork av Redo Backup and Recovery, som då inte hade underhållits på sju år. Sedan version 2.0 (2020) skriver Rescuezilla avbildningar i Clonezilla-format: En säkerhetskopia skapad med Rescuezilla kan återställas med Clonezilla och vice versa. Licensen är GPL-3.0. Den aktuella versionen är 2.6.2 från maj 2026, baserad på Ubuntu 26.04 LTS med Partclone 0.3.47.

Partclone utför själva arbetet. Det känner till de vanliga filsystemen (NTFS, FAT, ext4 och fler) och läser bara använda block. En partition på 1 TB med 350 GB data ger därför en avbildning på omkring 215 GB (gzip-komprimerad), inte 1 TB. Partitioner utan igenkänt filsystem, exempelvis den Microsoft-reserverade partitionen, säkerhetskopieras blockvis av Rescuezilla med `dd`; dessa filer har filändelsen `.dd-ptcl-img` i avbildningen.

<details class="options-details">
<summary>Funktioner i korthet</summary>

| Funktion | Syfte |
|---|---|
| Backup | säkerhetskopierar valda partitioner på en disk, inklusive partitionstabellen, som en avbildning till en lokal enhet eller nätverksresurs (SMB, SSH) |
| Restore | återställer en avbildning till en disk, valfritt med överskrivning av partitionstabellen |
| Verify Image | kontrollerar om en befintlig avbildning är komplett och läsbar |
| Clone | kopierar en disk direkt till en annan, utan mellanlagring |
| Image Explorer (beta) | monterar en avbildning skrivskyddat för att hämta ut enskilda filer |
| VM-Images | läser, utöver Clonezilla-avbildningar, även VDI, VMDK, VHDX, QCOW2 och Raw-avbildningar |
| Extra verktyg | GParted, filhanterare, webbläsare och verktyg för att återställa raderade filer på live-skrivbordet |
| CLI | experimentell kommandorad (sedan 2.5) för Backup, Verify, Restore och Clone |

</details>

En begränsning är viktig vid migreringar: Rescuezilla förminskar inte partitioner. Måldisken måste minst räcka till slutet av den sista säkerhetskopierade partitionen. Om den är mindre krävs förarbete med GParted, vilket beskrivs i den länkade guiden ovan.

## Avbildning eller klon

Rescuezilla erbjuder två sätt att migrera. Vid **kloning** är käll- och måldisken anslutna samtidigt och Rescuezilla kopierar direkt. Det sparar tid och en tredje datalagringsenhet, men förutsätter att båda diskarna är anslutna samtidigt till datorn eller en adapter. Vid **imaging** skapas först en avbildning på en extern enhet, som sedan återställs till den nya disken. Det tar längre tid men har en fördel: avbildningen finns kvar som en fullständig säkerhetskopia av det tidigare läget, även om något går fel vid återställningen. För en migrering är avbildningen därför det säkrare valet, och det är denna variant som artikeln beskriver.

## Förutsättningar

1.  **USB-minne** för Rescuezilla. Dess innehåll raderas när startavbildningen skrivs.

2.  **Extern enhet** för avbildningen, med ledigt utrymme ungefär motsvarande mängden använda data. Både exFAT och NTFS fungerar.

3.  **Måldisk**, minst lika stor som källdisken. Om den är mindre ska du först gå igenom guiden för mindre SSD-enheter.

4.  **BitLocker-återställningsnyckel**, om C: är krypterad. Den kan hämtas på aka.ms/myrecoverykey eller i Entra-portalen för hanterade enheter.

## Steg 1: Förbered Windows

Kontrollera tre punkter före säkerhetskopieringen i en PowerShell med administratörsbehörighet.

**BitLocker.** Partclone kan inte läsa en BitLocker-volym som NTFS. Rescuezilla säkerhetskopierar då den blockvis, som alla partitioner utan igenkänt filsystem, och avbildningen blir lika stor som hela partitionen. Vid en migrering rekommenderas att du stänger av BitLocker först och aktiverar det igen efter flytten:

```powershell
manage-bde -status C:
manage-bde -off C:
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-status` | visar volymens krypteringsgrad och skyddsstatus |
| `-off` | dekrypterar volymen helt; fortsätter i bakgrunden |
| `C:` | positionsargument: den berörda volymen |

</details>

Vänta tills `manage-bde -status C:` visar värdet `Fully Decrypted`. På enheter som hanteras via Intune kan en princip aktivera BitLocker på nytt; kontrollera därför statusen igen precis före omstarten.

**Viloläge och snabbstart.** När snabbstart är aktiverad stängs Windows inte av helt, och NTFS betraktas fortfarande som i användning. Partclone avbryter då. Ett kommando inaktiverar båda:

```powershell
powercfg /h off
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `/h off` | kortform av `/hibernate off`: inaktiverar viloläge och snabbstart, `hiberfil.sys` tas bort |

</details>

**Filsystemets tillstånd.** Om Dirty Bit är satt avbryter Partclone med meddelandet att volymen är ”scheduled for a check or it was shutdown uncleanly”. Kontrollera i förväg:

```powershell
fsutil dirty query C:
```

Om kommandot rapporterar `is Dirty`, schemalägger du en kontroll vid nästa start med `chkdsk C: /f`, startar om och kontrollerar igen.

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `dirty query C:` | fsutil: frågar om volymens Dirty Bit är satt |
| `/f` | chkdsk: åtgärdar fel; på systemvolymen schemaläggs kontrollen till nästa start |

</details>

## Steg 2: Skapa och starta USB-minnet

Hämta ISO-avbildningen från projektets GitHub-releasesida. Standardvarianten har Ubuntu-kodnamnet i filnamnet, vid 2.6.2 `rescuezilla-2.6.2-64bit.resolute.iso`. Skriv den till USB-minnet med ett avbildningsskrivprogram som balenaEtcher; projektsidan rekommenderar detta program för Windows, macOS och Linux.

Windows kan själv initiera start från minnet:

```powershell
shutdown /r /o /t 0
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `/r` | startar om i stället för att stänga av |
| `/o` | startar i de avancerade startalternativen; endast tillsammans med `/r` |
| `/t 0` | väntetid i sekunder före körning |

</details>

I de avancerade startalternativen väljer du ”Använd en enhet” och USB-minnet. Alternativt öppnas firmwarets startmeny när datorn slås på (beroende på tillverkare F8, F11 eller F12). Rescuezilla startar med Secure Boot aktiverat; det behöver inte stängas av. Om skärmen förblir svart efter språkvalet hjälper alternativet ”Graphical Fallback Mode” i USB-minnets startmeny.

## Steg 3: Skapa en säkerhetskopia

Rescuezilla startar automatiskt på skrivbordet. Guiden leder genom säkerhetskopieringen i numrerade steg:

1.  Välj **Backup**.

2.  *Step 1: Select Drive To Backup*: välj systemdisken. Modell och storlek hjälper vid identifieringen; enhetsnamnen (`nvme0n1`, `sda`) kan skilja sig mellan två starter.

3.  *Step 2: Select Partitions to Save*: låt alla partitioner vara markerade. För en startbar återställning behövs EFI-systempartitionen, den Microsoft-reserverade partitionen, C: och återställningspartitionen.

4.  *Step 3: Select Destination Drive*: den externa enheten (Local) eller en nätverksresurs (Network).

5.  *Step 4: Select Destination Folder*: målmappen på enheten. Rescuezilla skapar en undermapp med tidsstämpel i den.

6.  *Step 5: Name Your Backup*: valfritt en beskrivning; den visas senare i avbildningslistan.

7.  *Step 6: Customize Compression Settings*: Standardinställningen gzip passar i de flesta fall.

8.  *Step 7: Confirm Backup Configuration*: kontrollera uppgifterna och starta.

För 350 GB data på en NVMe-SSD tog säkerhetskopieringen till en extern USB-SSD omkring 20 minuter. I slutet meddelar Rescuezilla för varje partition om den säkerhetskopierades korrekt. Om en partition visar ett fel är avbildningen ofullständig, även om mappen finns.

## Steg 4: Kontrollera avbildningen

Kontrollera avbildningen före återställningen. **Verify Image** i huvudmenyn läser alla delar och kontrollerar att de är kompletta och läsbara. I avbildningslistan visar kolumnen **Partitions** de säkerhetskopierade partitionerna med storlek; en gul varningstriangel markerar avbildningar med saknade partitioner.

Avbildningsmappen kan även granskas i Windows. De viktigaste filerna:

| Fil | Innehåll |
|---|---|
| `nvme0n1-pt.parted` | partitionstabell i textformat, med början och slut för varje partition i sektorer |
| `nvme0n1-gpt-1st`, `nvme0n1-gpt-2nd` | binära kopior av den primära GPT:n och backup-GPT:n |
| `nvme0n1p3.ntfs-ptcl-img.gz.aa`, `.ab`, … | Partclone-avbildning av C:, gzip-komprimerad och uppdelad i delar om 4 GB |
| `nvme0n1p2.dd-ptcl-img.gz.aa` | blockvis kopia av en partition utan igenkänt filsystem |
| `clonezilla-img` | logg över säkerhetskopieringen med resultat för varje partition |
| `Info-*.txt` | hårdvaru- och SMART-information om källsystemet vid tidpunkten för säkerhetskopieringen |

Av `nvme0n1-pt.parted` går det att utläsa om avbildningen ryms på måldisken: slutsektorn för den sista partitionen plus ett, multiplicerat med 512 byte, måste vara mindre än måldiskens storlek i byte.

## Steg 5: Återställ till den nya SSD:n

Installera den nya SSD:n och starta Rescuezilla igen.

1.  Välj **Restore**.

2.  *Step 1: Select Image Location*: den externa enheten med avbildningen.

3.  *Step 2: Select Backup Image*: välj avbildningen utifrån datum och partitionsstorlekar.

4.  *Step 3: Select Drive To Restore*: den nya SSD:n. Kontrollera modell och storlek två gånger: Alla data på denna enhet skrivs över.

5.  *Step 4: Select Partitions to Restore*: låt alla partitioner vara markerade, liksom **Overwrite partition table**.

6.  *Step 5: Confirm Restore Configuration*: kontrollera och starta.

Återställningen tar ungefär lika lång tid som säkerhetskopieringen. Stäng sedan av och ta bort USB-minnet.

## Steg 6: Starta från den nya enheten

Det enklaste är att koppla ur den gamla disken före första starten. Om båda diskarna är anslutna har de samma partitions- och volym-ID:n, och firmwaret kan välja den gamla för start. Om den gamla disken ska fortsätta användas ska du formatera den först efter att det nya systemet har kontrollerats.

Om den nya SSD:n är större än den gamla ligger det extra lagringsutrymmet som oallokerat utrymme i slutet. C: kan dock inte utökas direkt i Diskhantering, eftersom återställningspartitionen ligger mellan C: och det lediga utrymmet. Med GParted på Rescuezilla-minnet flyttar du först återställningspartitionen till slutet (Resize/Move, **Free space following** till 0) och utökar sedan C: till hela det lediga utrymmet. Windows RE hittar sedan sin partition igen eftersom den behåller sitt partitionsnummer; `reagentc /info` visar statusen.

Om datorn också byts vid migreringen startar Windows i regel utan anpassningar och installerar saknade drivrutiner efteråt. Aktiveringen sker via den digitala licensen, och vid byte av moderkort möjligen först efter inloggning med Microsoft-kontot via aktiveringsfelsökaren.

## Efterarbete

På den nya enheten aktiverar du åter det som inaktiverades för migreringen. Viloläget aktiveras med `powercfg /h on`. BitLocker aktiverar du via Inställningar → Sekretess och säkerhet → Enhetskryptering eller med `manage-bde -on C:`; kontrollera sedan med `manage-bde -protectors -get C:` att en återställningsnyckel finns och har sparats.

Behåll avbildningen på den externa enheten tills Windows har fungerat utan problem på den nya enheten i några dagar och alla data har kontrollerats. Därefter kan den raderas eller arkiveras som en säkerhetskopia av leveransskicket.

## Källor

1.  [Rescuezilla](https://rescuezilla.com/): projektsida med funktionsöversikt, nedladdning och FAQ.

2.  [Rescuezilla på GitHub](https://github.com/rescuezilla/rescuezilla): källkod, projekthistorik och funktionslista i README.

3.  [Rescuezilla Releases](https://github.com/rescuezilla/rescuezilla/releases/latest): aktuell version med versionsinformation, kompatibilitetslista och ISO-varianter.

4.  [Rescuezilla Changelog](https://raw.githubusercontent.com/rescuezilla/rescuezilla/master/CHANGELOG.md): införande av CLI, Verify Image och Clone samt uppdateringen av Secure Boot-shim.

5.  [Rescuezilla Wiki: Restoring to a smaller disk](https://github.com/rescuezilla/rescuezilla/wiki/HOWTO:-Restoring-to-a-smaller-disk.-Eg,-1000GB-HDD-to-500GB-SSD): officiellt förfarande för måldiskar som är mindre än originalet.

6.  [Partclone](https://partclone.org/): verktyget som Rescuezilla och Clonezilla använder för filsystemsanpassad säkerhetskopiering.

7.  [GParted Manual](https://gparted.org/display-doc.php?name=help-manual): flyttning och utökning av partitioner.

8.  [Microsoft Learn: manage-bde status](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-status): statuskontroll för BitLocker.

9.  [Microsoft Learn: manage-bde off](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-off): dekryptering av en volym.

10.  [Microsoft Learn: Powercfg command-line options](https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/powercfg-command-line-options): alternativet `/hibernate` och dess påverkan på snabbstart.

11.  [Microsoft Learn: fsutil dirty](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/fsutil-dirty): kontroll av Dirty Bit.

12.  [Microsoft Learn: chkdsk](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/chkdsk): kontroll och reparation av NTFS-volymer.

13.  [Microsoft Learn: shutdown](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/shutdown): parametern `/o` för start i de avancerade startalternativen.

14.  [Microsoft Support: Reactivating Windows after a hardware change](https://support.microsoft.com/en-us/windows/reactivating-windows-after-a-hardware-change-2c0e962a-f04c-145b-6ead-fb3fc72b6665): aktivering via den digitala licensen efter byte av moderkort.
