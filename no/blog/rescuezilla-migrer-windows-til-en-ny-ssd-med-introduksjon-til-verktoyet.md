---
title: "Rescuezilla: Migrer Windows til en ny SSD, med introduksjon til verktøyet"
navTitle: "Rescuezilla-migrering"
description: "Rescuezilla er en gratis, Clonezilla-kompatibel imaging-løsning med grafisk grensesnitt. Artikkelen presenterer verktøyet og viser hele flyttingen av en Windows-installasjon til en ny SSD: forberedelser i Windows, oppstartsminnepinne, sikkerhetskopi, kontroll, gjenoppretting og etterarbeid."
date: "2026-09-29"
kategorie: "PC og maskinvare"
timeToRead: "10 min lesetid"
themen:
  - pc-hardware
produkte:
  - "windows-client"
protokolle:
  - "backup-dr"
  - "migration"
slug: "rescuezilla-migrer-windows-til-en-ny-ssd-med-introduksjon-til-verktoyet"
translationId: "article-d9438c5774d0167c"
translationOf: rescuezilla-windows-migration
url: https://rafaelpfister.ch/no/blog/rescuezilla-migrer-windows-til-en-ny-ssd-med-introduksjon-til-verktoyet
translationSourceHash: 33320c51c0a77d69bc5804e2c7e69d14c63731a23df6ef55969b37b043a49cd5
translationModel: gpt-5.6-terra
translatedAt: 2026-09-30T09:53:03.130Z
translationReview: automatic
---

Hvis du vil flytte en Windows-installasjon til en ny SSD eller en ny datamaskin uten å installere Windows på nytt, trenger du et verktøy som sikkerhetskopierer og gjenoppretter hele disken med alle partisjoner. Rescuezilla er et slikt verktøy: gratis, åpen kildekode og med et grafisk grensesnitt som kan brukes også uten Linux-kunnskaper. Denne artikkelen presenterer verktøyet og beskriver migreringen til en SSD som er like stor eller større. For en mindre mål-SSD finnes det en egen veiledning: [Flytt Windows med Rescuezilla til en mindre SSD](/blog/windows-kleinere-ssd-rescuezilla).

## Hva Rescuezilla er

Rescuezilla er et live-system basert på Ubuntu som startes fra en USB-minnepinne. Det kjører uavhengig av det installerte operativsystemet og sikkerhetskopierer derfor også volumer som Windows låser under drift. Prosjektet oppstod i 2019 som en fork av Redo Backup and Recovery, som på det tidspunktet ikke hadde blitt vedlikeholdt på sju år. Siden versjon 2.0 (2020) skriver Rescuezilla images i Clonezilla-format: En sikkerhetskopi opprettet med Rescuezilla kan gjenopprettes med Clonezilla og omvendt. Lisensen er GPL-3.0. Den nyeste versjonen er 2.6.2 fra mai 2026, basert på Ubuntu 26.04 LTS med Partclone 0.3.47.

Selve arbeidet utføres av Partclone. Det kjenner de vanlige filsystemene (NTFS, FAT, ext4 og flere) og leser bare de opptatte blokkene. En partisjon på 1 TB med 350 GB data gir derfor et image på rundt 215 GB (gzip-komprimert), ikke 1 TB. Partisjoner uten gjenkjent filsystem, som den Microsoft-reserverte partisjonen, sikkerhetskopierer Rescuezilla blokk for blokk med `dd`; disse filene har endelsen `.dd-ptcl-img` i imaget.

<details class="options-details">
<summary>Funksjoner i oversikt</summary>

| Funksjon | Formål |
|---|---|
| Backup | sikkerhetskopierer valgte partisjoner på en disk, inkludert partisjonstabellen, som et image til en lokal stasjon eller en nettverksressurs (SMB, SSH) |
| Restore | gjenoppretter et image til en disk, valgfritt med overskriving av partisjonstabellen |
| Verify Image | kontrollerer om et eksisterende image er komplett og lesbart |
| Clone | kopierer en disk direkte til en annen, uten mellomlagring |
| Image Explorer (beta) | monterer et image skrivebeskyttet for å hente ut enkeltfiler |
| VM-images | leser, i tillegg til Clonezilla-images, også VDI-, VMDK-, VHDX-, QCOW2- og Raw-images |
| Tilleggsverktøy | GParted, filbehandler, nettleser og verktøy for gjenoppretting av slettede filer på live-skrivebordet |
| CLI | eksperimentell kommandolinje (siden 2.5) for Backup, Verify, Restore og Clone |

</details>

En begrensning er viktig ved migreringer: Rescuezilla krymper ikke partisjoner. Måldisken må minst rekke til slutten av den siste sikkerhetskopierte partisjonen. Hvis den er mindre, kreves det forarbeid med GParted, beskrevet i veiledningen det er lenket til ovenfor.

## Image eller klone

Rescuezilla tilbyr to fremgangsmåter for en migrering. Ved **kloning** er kilde- og måldisken tilkoblet samtidig, og Rescuezilla kopierer direkte. Det sparer tid og en tredje datadisk, men forutsetter at begge diskene er koblet til datamaskinen eller en adapter samtidig. Ved **imaging** opprettes først et image på en ekstern stasjon, som deretter gjenopprettes til den nye disken. Dette tar lengre tid, men har en fordel: Imaget beholdes som en fullstendig sikkerhetskopi av den gamle tilstanden, også hvis noe går galt under gjenopprettingen. For en migrering er imaget derfor det tryggere valget, og det er denne varianten artikkelen beskriver.

## Forutsetninger

1.  **USB-minnepinne** for Rescuezilla. Innholdet slettes når oppstartsavbildningen skrives.

2.  **Ekstern stasjon** for imaget, med ledig plass omtrent tilsvarende størrelsen på de opptatte dataene. Både exFAT og NTFS fungerer.

3.  **Måldisk**, minst like stor som kildedisken. Hvis den er mindre, må du først gå gjennom veiledningen for mindre SSD-er.

4.  **BitLocker-gjenopprettingsnøkkel**, dersom C: er kryptert. Den kan hentes på aka.ms/myrecoverykey eller i Entra-portalen for administrerte enheter.

## Trinn 1: Klargjør Windows

Kontroller tre punkter før sikkerhetskopieringen i en PowerShell med administratorrettigheter.

**BitLocker.** Partclone kan ikke lese et BitLocker-volum som NTFS. Rescuezilla sikkerhetskopierer det da blokk for blokk, som enhver partisjon uten gjenkjent filsystem, og imaget blir like stort som hele partisjonen. Ved migrering anbefales det å slå av BitLocker på forhånd og slå det på igjen etter flyttingen:

```powershell
manage-bde -status C:
manage-bde -off C:
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-status` | viser krypteringsgrad og beskyttelsesstatus for volumet |
| `-off` | dekrypterer volumet fullstendig; fortsetter å kjøre i bakgrunnen |
| `C:` | posisjonsargument: det berørte volumet |

</details>

Vent til `manage-bde -status C:` viser verdien `Fully Decrypted`. På enheter som administreres via Intune, kan en policy slå på BitLocker igjen. Kontroller derfor statusen en gang til rett før omstart.

**Dvalemodus og hurtigstart.** Med hurtigstart aktivert slår ikke Windows seg helt av, og NTFS anses fortsatt som i bruk. Da avbryter Partclone. Én kommando deaktiverer begge:

```powershell
powercfg /h off
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `/h off` | kortform av `/hibernate off`: deaktiverer dvalemodus og hurtigstart, `hiberfil.sys` fjernes |

</details>

**Filsystemtilstand.** Hvis dirty-biten er satt, avbryter Partclone med meldingen om at volumet er «scheduled for a check or it was shutdown uncleanly». Kontroller på forhånd:

```powershell
fsutil dirty query C:
```

Hvis kommandoen rapporterer `is Dirty`, planlegger du en kontroll ved neste oppstart med `chkdsk C: /f`, starter på nytt og kontrollerer igjen.

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `dirty query C:` | fsutil: spør om volumets dirty-bit er satt |
| `/f` | chkdsk: retter feil; på systemvolumet planlegges kontrollen til neste oppstart |

</details>

## Trinn 2: Opprett og start oppstartsminnepinnen

Last ned ISO-imaget fra prosjektets GitHub-utgivelsesside. Standardvarianten har Ubuntu-kodenavnet i filnavnet, i 2.6.2 `rescuezilla-2.6.2-64bit.resolute.iso`. Skriv det til USB-minnepinnen med en image-skriver som balenaEtcher; prosjektsiden anbefaler dette programmet for Windows, macOS og Linux.

Windows kan selv styre oppstarten fra minnepinnen:

```powershell
shutdown /r /o /t 0
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `/r` | omstart i stedet for avslåing |
| `/o` | starter i de avanserte oppstartsalternativene; kun sammen med `/r` |
| `/t 0` | ventetid i sekunder før utførelse |

</details>

I de avanserte oppstartsalternativene velger du «Bruk en enhet» og USB-minnepinnen. Alternativt åpnes fastvarens oppstartsmeny når maskinen slås på (avhengig av produsent F8, F11 eller F12). Rescuezilla starter med Secure Boot aktivert; det trenger ikke å deaktiveres. Hvis skjermen forblir svart etter språkvalget, kan valget «Graphical Fallback Mode» i minnepinnens oppstartsmeny hjelpe.

## Trinn 3: Opprett sikkerhetskopi

Rescuezilla starter automatisk på skrivebordet. Veiviseren leder deg gjennom sikkerhetskopieringen i nummererte trinn:

1.  Velg **Backup**.

2.  *Step 1: Select Drive To Backup*: velg systemdisken. Modell og størrelse hjelper med å skille dem fra hverandre; enhetsnavnene (`nvme0n1`, `sda`) kan variere mellom to oppstarter.

3.  *Step 2: Select Partitions to Save*: la alle partisjonene være avkrysset. En oppstartbar gjenoppretting krever EFI-systempartisjonen, den Microsoft-reserverte partisjonen, C: og gjenopprettingspartisjonen.

4.  *Step 3: Select Destination Drive*: den eksterne stasjonen (Local) eller en nettverksressurs (Network).

5.  *Step 4: Select Destination Folder*: målmappen på stasjonen. Rescuezilla oppretter en undermappe med tidsstempel der.

6.  *Step 5: Name Your Backup*: valgfri beskrivelse; den vises senere i imagelisten.

7.  *Step 6: Customize Compression Settings*: Standardinnstillingen gzip passer i de fleste tilfeller.

8.  *Step 7: Confirm Backup Configuration*: kontroller opplysningene og start.

For 350 GB data på en NVMe-SSD tok sikkerhetskopieringen til en ekstern USB-SSD rundt 20 minutter. Til slutt rapporterer Rescuezilla for hver partisjon om den ble sikkerhetskopiert. Hvis en partisjon viser en feil, er imaget ufullstendig, selv om mappen finnes.

## Trinn 4: Kontroller imaget

Kontroller imaget før gjenopprettingen. **Verify Image** i hovedmenyen leser alle deler og kontrollerer at de er komplette og lesbare. I imagelisten viser kolonnen **Partitions** de sikkerhetskopierte partisjonene med størrelse; en gul varseltrekant markerer images med manglende partisjoner.

Imagemappen kan også vises i Windows. De viktigste filene:

| Fil | Innhold |
|---|---|
| `nvme0n1-pt.parted` | partisjonstabell i tekstformat, med start og slutt for hver partisjon i sektorer |
| `nvme0n1-gpt-1st`, `nvme0n1-gpt-2nd` | binære kopier av primær- og reserve-GPT-en |
| `nvme0n1p3.ntfs-ptcl-img.gz.aa`, `.ab`, … | Partclone-image av C:, gzip-komprimert, delt opp i deler på 4 GB |
| `nvme0n1p2.dd-ptcl-img.gz.aa` | blokkvis kopi av en partisjon uten gjenkjent filsystem |
| `clonezilla-img` | logg over sikkerhetskopieringen med resultat for hver partisjon |
| `Info-*.txt` | maskinvare- og SMART-opplysninger om kildesystemet på tidspunktet for sikkerhetskopieringen |

Fra `nvme0n1-pt.parted` kan du se om imaget passer på måldisken: Sluttsektoren til den siste partisjonen pluss én, ganget med 512 byte, må være mindre enn størrelsen på måldisken i byte.

## Trinn 5: Gjenopprett til den nye SSD-en

Monter den nye SSD-en og start Rescuezilla igjen.

1.  Velg **Restore**.

2.  *Step 1: Select Image Location*: den eksterne stasjonen med imaget.

3.  *Step 2: Select Backup Image*: velg imaget ut fra dato og partisjonsstørrelser.

4.  *Step 3: Select Drive To Restore*: den nye SSD-en. Kontroller modell og størrelse to ganger: Alle data på denne stasjonen overskrives.

5.  *Step 4: Select Partitions to Restore*: la alle partisjonene være avkrysset, samt **Overwrite partition table**.

6.  *Step 5: Confirm Restore Configuration*: kontroller og start.

Gjenopprettingen tar omtrent like lang tid som sikkerhetskopieringen. Slå deretter av maskinen og fjern minnepinnen.

## Trinn 6: Start fra den nye stasjonen

Det enkleste er å koble fra den gamle disken før første oppstart. Hvis begge diskene er tilkoblet, har de samme partisjons- og volum-ID-er, og fastvaren kan velge den gamle for oppstart. Hvis den gamle disken skal brukes videre, må du ikke formatere den før det nye systemet er kontrollert.

Hvis den nye SSD-en er større enn den gamle, ligger den ekstra lagringsplassen som et ikke-allokert område på slutten. C: kan imidlertid ikke utvides direkte i Diskbehandling, fordi gjenopprettingspartisjonen ligger mellom C: og det ledige området. Med GParted på Rescuezilla-minnepinnen flytter du først gjenopprettingspartisjonen til slutten (Resize/Move, **Free space following** til 0) og utvider deretter C: til hele det ledige området. Windows RE finner deretter partisjonen igjen fordi den beholder partisjonsnummeret; `reagentc /info` viser statusen.

Hvis datamaskinen også byttes ved migreringen, starter Windows som regel uten tilpasninger og installerer manglende drivere etterpå. Aktiveringen skjer via den digitale lisensen; ved bytte av hovedkort eventuelt først etter innlogging med Microsoft-kontoen gjennom aktiveringsfeilsøkingen.

## Etterarbeid

På den nye stasjonen aktiverer du igjen det som ble deaktivert for migreringen. Dvalemodus aktiveres med `powercfg /h on`. Du aktiverer BitLocker via Innstillinger → Personvern og sikkerhet → Enhetskryptering eller med `manage-bde -on C:`; kontroller deretter med `manage-bde -protectors -get C:` at en gjenopprettingsnøkkel finnes og er lagret.

Behold imaget på den eksterne stasjonen til Windows har kjørt uten problemer på den nye stasjonen i noen dager og alle data er kontrollert. Deretter kan det slettes eller arkiveres som en sikkerhetskopi av leveringstilstanden.

## Kilder

1.  [Rescuezilla](https://rescuezilla.com/): prosjektside med funksjonsoversikt, nedlasting og FAQ.

2.  [Rescuezilla på GitHub](https://github.com/rescuezilla/rescuezilla): kildekode, prosjekthistorikk og funksjonsliste i README.

3.  [Rescuezilla Releases](https://github.com/rescuezilla/rescuezilla/releases/latest): nyeste versjon med utgivelsesnotater, kompatibilitetsliste og ISO-varianter.

4.  [Rescuezilla Changelog](https://raw.githubusercontent.com/rescuezilla/rescuezilla/master/CHANGELOG.md): innføring av CLI, Verify Image og Clone samt oppdateringen av Secure Boot-shim-en.

5.  [Rescuezilla Wiki: Restoring to a smaller disk](https://github.com/rescuezilla/rescuezilla/wiki/HOWTO:-Restoring-to-a-smaller-disk.-Eg,-1000GB-HDD-to-500GB-SSD): offisiell fremgangsmåte for måldisker som er mindre enn originalen.

6.  [Partclone](https://partclone.org/): verktøyet som Rescuezilla og Clonezilla bruker til filsystembevisst sikkerhetskopiering.

7.  [GParted Manual](https://gparted.org/display-doc.php?name=help-manual): flytting og utvidelse av partisjoner.

8.  [Microsoft Learn: manage-bde status](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-status): statuskontroll av BitLocker.

9.  [Microsoft Learn: manage-bde off](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-off): dekryptering av et volum.

10.  [Microsoft Learn: Powercfg command-line options](https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/powercfg-command-line-options): alternativet `/hibernate` og påvirkningen på hurtigstart.

11.  [Microsoft Learn: fsutil dirty](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/fsutil-dirty): kontroll av dirty-biten.

12.  [Microsoft Learn: chkdsk](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/chkdsk): kontroll og reparasjon av NTFS-volumer.

13.  [Microsoft Learn: shutdown](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/shutdown): parameteren `/o` for oppstart i de avanserte oppstartsalternativene.

14.  [Microsoft Support: Reactivating Windows after a hardware change](https://support.microsoft.com/en-us/windows/reactivating-windows-after-a-hardware-change-2c0e962a-f04c-145b-6ead-fb3fc72b6665): aktivering via digital lisens etter bytte av hovedkort.
