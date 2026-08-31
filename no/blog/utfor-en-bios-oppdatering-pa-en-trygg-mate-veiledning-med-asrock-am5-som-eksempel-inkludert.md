---
title: "Utfør BIOS-oppdatering på en sikker måte: Veiledning med ASRock AM5 som eksempel, inkludert BitLocker-forberedelser"
navTitle: "BIOS-oppdatering"
description: "Den komplette prosessen for en BIOS-oppdatering med et ASRock AM5-kort som eksempel: Finn versjonen, verifiser nedlastingen med hash, sett BitLocker på pause riktig, start i UEFI (også når F2 ikke fungerer), oppdater med Instant Flash og angi fornuftige innstillinger etter oppdateringen."
date: "2026-08-26"
kategorie: "PC og maskinvare"
timeToRead: "8 min lesetid"
themen:
  - pc-hardware
produkte:
  - "windows-client"
protokolle:
  - "troubleshooting"
  - "releases"
slug: "utfor-en-bios-oppdatering-pa-en-trygg-mate-veiledning-med-asrock-am5-som-eksempel-inkludert"
translationId: "article-82840b2d159b9367"
translationOf: bios-update-asrock-am5-sicher-durchfuehren
translationSourceHash: 555b16e753b2ac5dec357741b071ed6aa33de367a2197a8dbb10fef7c9f6a946
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:15:54.361Z
translationReview: automatic
url: https://rafaelpfister.ch/no/blog/utfor-en-bios-oppdatering-pa-en-trygg-mate-veiledning-med-asrock-am5-som-eksempel-inkludert
---

En BIOS-oppdatering er blant vedlikeholdsoppgavene som sjelden er aktuelle, og derfor vekker spørsmål hver gang: Hvilken versjon er riktig, hvordan får man den trygt over på kortet, og hva må tas hensyn til før og etterpå? Denne veiledningen dokumenterer hele prosessen med et ASRock A620I Lightning WiFi (AM5-sokkel) og produsentens egen Instant Flash-metode som eksempel. Trinnene kan overføres til ethvert moderne hovedkort, og de kritiske punktene (BitLocker, Fast Boot, tilbakestilling av innstillinger) er uavhengige av produsent.

## Når en BIOS-oppdatering er på sin plass

Tre forhold rettferdiggjør inngrepet. For det første sikkerhetsrettinger: Sårbarheter i fastvaren kan bare lukkes via en BIOS-oppdatering, og produsentenes endringslogger omtaler dem vanligvis bare kort. For det andre kompatibilitet: Støtte for nye CPU-generasjoner og bedre minnekompatibilitet kommer utelukkende gjennom nye fastvareversjoner, på AM5 gjennom AGESA-referansefastvaren fra AMD, som hovedkortprodusentene bygger inn i BIOS-versjonene sine. For det tredje stabilitet: Hvis et system starter spontant på nytt og hendelsesloggen bare registrerer Kernel-Power 41 med `BugcheckCode=0`, har krasjet skjedd på maskinvare- eller fastvarenivå, uten medvirkning fra Windows; typiske årsaker er ustabile spenninger og minnetrening, og det er nettopp dette nivået AGESA-utgivelsene vedlikeholder. Oppføringer som "Improve memory compatibility and system stability" eller revidert EXPO-håndtering i endringsloggene indikerer at en oppdatering håndterer slike problemer. Hvis systemet derimot kjører stabilt og ikke er berørt av de rettede sårbarhetene, er det legitimt å vente; en BIOS-oppdatering uten grunn er en risiko uten motytelse.

## Trinn 1: Fastslå nåværende status

Før du laster ned noe, trenger du to opplysninger: nøyaktig hovedkortmodell og installert BIOS-versjon. PowerShell gir deg begge uten omstart:

```powershell
Get-CimInstance Win32_ComputerSystem | Select-Object Manufacturer, Model
Get-CimInstance Win32_BIOS | Select-Object SMBIOSBIOSVersion, ReleaseDate
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `Win32_ComputerSystem` | Posisjonsargumentet ClassName: CIM-klasse med produsent og systemmodell |
| `Win32_BIOS` | CIM-klasse med fastvareopplysninger, inkludert versjon og dato |
| `Select-Object <eigenschaften>` | reduserer utdataene til de angitte egenskapene |

</details>

Noter deg versjonen. Du trenger den senere for å kontrollere at oppdateringen lykkes, og når du leser endringsloggene, må du vite hvilke versjoner du hopper over.

## Trinn 2: Last ned BIOS og verifiser kontrollsummen

Last ned BIOS utelukkende fra produsentens produktside, aldri fra tredjepartsportaler. ASRock publiserer SHA256-kontrollsummen for hver versjon; sammenlign den etter nedlastingen før filen i det hele tatt kommer i nærheten av en USB-pinne:

```powershell
Get-FileHash .\A620I_Lightning_WiFi_4.43.zip -Algorithm SHA256
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `.\A620I_…_4.43.zip` | Posisjonsargumentet Path: filen som skal kontrolleres |
| `-Algorithm SHA256` | Hash-metode; må samsvare med kontrollsumtypen publisert av produsenten |

</details>

Hvis verdien ikke samsvarer med produsentens opplysninger, er nedlastingen skadet eller manipulert: ikke flash. Etter utpakking gjenstår én ROM-fil, i eksempelet `A62IRW_4.43.ROM` på 32 MB.

## Trinn 3: Klargjør USB-pinnen

Flash-mekanismen i UEFI (hos ASRock "Instant Flash", hos andre produsenter Q-Flash, EZ Flash eller M-Flash) leser USB-pinnen direkte fra fastvaren. Det betyr at bare FAT32 gjenkjennes pålitelig, ikke NTFS og exFAT. Nesten alle ferdigkjøpte USB-pinner er allerede FAT32; du kan kontrollere det slik:

```powershell
Get-CimInstance Win32_LogicalDisk -Filter "DriveType=2" |
  Select-Object DeviceID, FileSystem, VolumeName
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `Win32_LogicalDisk` | CIM-klasse for logiske stasjoner |
| `-Filter "DriveType=2"` | WQL-filter for flyttbare medier; skjuler harddisker og CD-stasjoner |
| `Select-Object DeviceID, FileSystem, VolumeName` | viser stasjonsbokstav, filsystem og volumnavn |

</details>

Kopier ROM-filen til rotkatalogen på USB-pinnen. Ny formatering er bare nødvendig dersom filsystemet ikke passer. Størrelsen på pinnen spiller ingen rolle; filen er mindre enn enhver vanlig kapasitet i dag.

En merknad om valg av metode: Mange hovedkort tilbyr også en BIOS Flashback-knapp, som flasher uten CPU og uten et fungerende system. Dette er redningsveien for et hovedkort som ikke lenger starter. For et system som kjører, er Instant Flash i UEFI den riktige og enklere metoden. Windows-baserte flash-verktøy er verken nødvendige eller anbefalte på moderne plattformer.

## Trinn 4: Sett BitLocker på pause, ellers risikerer du nøkkelforespørselen

Dette er punktet som mangler i mange veiledninger. Hvis systemdisken er kryptert med BitLocker (ofte automatisk aktivert i Windows 11 med Microsoft-konto), knytter BitLocker nøkkelen til måleverdiene fra TPM. En BIOS-oppdatering endrer disse måleverdiene, og ved neste oppstart ber Windows om den 48-sifrede gjenopprettingsnøkkelen. Den som ikke har den tilgjengelig, står med et utilgjengelig system.

BitLocker har en egen mekanisme for dette scenariet. I PowerShell med administratorrettigheter:

```powershell
Suspend-BitLocker -MountPoint C: -RebootCount 2
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-MountPoint C:` | det berørte volumet, her systemdisken |
| `-RebootCount 2` | antall omstarter beskyttelsen forblir satt på pause (0 til 15; 0 = til manuell reaktivering) |

</details>

Verdien 2 dekker begge de forestående omstartene (én gang til UEFI, én gang etter flashing); deretter aktiveres beskyttelsen igjen automatisk og forsegler nøkkelen mot de nye måleverdiene. Kontroller likevel på forhånd at gjenopprettingsnøkkelen er mulig å finne, for eksempel i Microsoft-kontoen under aka.ms/myrecoverykey eller via `manage-bde -protectors -get C:`.

## Trinn 5: Kom inn i UEFI, også når F2 ikke reagerer

Den klassiske metoden med F2 eller Delete ved oppstart mislykkes ofte på moderne systemer: Når Fast Boot er aktivert, initialiserer fastvaren USB-tastaturet først etter POST, slik at tastetrykket ikke registreres. Men du er ikke avhengig av tasten: Windows kan styre neste omstart direkte til UEFI-oppsettet. I PowerShell med administratorrettigheter:

```powershell
shutdown /r /fw /t 5
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `/r` | omstart i stedet for avslutning |
| `/fw` | angir fastvarevariabelen som styrer neste oppstart direkte til UEFI-oppsettet; bare sammen med et avslutningsalternativ som `/r`, krever administratorrettigheter |
| `/t 5` | ventetid i sekunder før utførelse |

</details>

Hvis kommandoen rapporterer feil 203 ("The system could not find the environment option that was entered"), mangler det nesten alltid administratorrettigheter: Uten forhøyede rettigheter kan prosessen ikke sette den nødvendige fastvarevariabelen, og feilmeldingen oppgir ikke denne årsaken. En annen mulig vei uten fastvarevariabel går via gjenopprettingsmiljøet: `shutdown /r /o`, deretter Feilsøking, Avanserte alternativer, UEFI-fastvareinnstillinger.

## Trinn 6: Flash med Instant Flash

I UEFI finner du Instant Flash i Tool-menyen. Verktøyet lister opp alle ROM-filene på USB-pinnen; etter at du har valgt filen, kontrollerer det den, flasher og starter på nytt automatisk. I løpet av de få minuttene gjelder den eneste absolutte regelen i hele prosessen: Ikke avbryt strømforsyningen og ikke slå av datamaskinen. En avbrutt flashing er det eneste trinnet i denne veiledningen som faktisk kan gjøre hovedkortet ute av stand til å starte (og da krever den nevnte Flashback-redningsmetoden).

## Trinn 7: Etterarbeid, for oppdateringen tilbakestiller alt

Etter flashingen er alle BIOS-innstillinger satt til fabrikkinnstillinger. Dette er tilsiktet og gir en diagnostisk mulighet: RAM kjører nå uten EXPO-profil på JEDEC-grunnhastigheten. Hvis du flashet på grunn av stabilitetsproblemer, bør du bevisst la det være slik i én til to uker. Hvis krasjene uteblir, var minneprofilen involvert, og du kan teste EXPO på nytt målrettet med den nye fastvaren. Forskjellen i daglig bruk mellom 4800 og 6000 MT/s er knapt merkbar utenfor referansetester; en stabil datamaskin er verdt hvert benchmark-poeng.

To innstillinger er uansett verdt et besøk i UEFI: Hvis du har hatt omstarter i hvilemodus, kan du under Advanced, AMD CBS sette alternativet "Power Supply Idle Control" til "Typical Current Idle"; dette avhjelper en kjent inkompatibilitet mellom enkelte strømforsyninger og Ryzen-CPU-enes dype hviletilstander. Og hvis du senere igjen vil åpne oppsettet med F2, kan du deaktivere Fast Boot.

Suksesskontrollen tilbake i Windows:

```powershell
Get-CimInstance Win32_BIOS | Select-Object SMBIOSBIOSVersion
Get-CimInstance Win32_PhysicalMemory |
  Select-Object PartNumber, ConfiguredClockSpeed
manage-bde -status C:
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `Win32_BIOS` | CIM-klasse med fastvareversjonen; `SMBIOSBIOSVersion` må nå vise den nye versjonen |
| `Win32_PhysicalMemory` | CIM-klasse for minnemodulene; `ConfiguredClockSpeed` viser den faktiske hastigheten i MT/s |
| `-status` | manage-bde: viser krypterings- og beskyttelsesstatus for volumet |
| `C:` | Posisjonsargument: volumet som skal kontrolleres |

</details>

Den første linjen må vise den nye versjonen, den andre forventet minnehastighet, og BitLocker må igjen rapportere "Beskyttelse aktivert". Dermed er oppdateringen fullført og dokumentert. Hvis flashingen ble utført på grunn av stabilitetsproblemer, vil først observasjon de påfølgende ukene vise om de er løst, enklest ved å se etter nye Kernel-Power-41-oppføringer i systemhendelsesloggen.

## Kilder

1.  [ASRock A620I Lightning WiFi, BIOS-nedlastinger](https://pg.asrock.com/mb/AMD/A620I%20Lightning%20WiFi/index.asp#BIOS): Versjonsliste med endringslogger, SHA256-kontrollsummer og de støttede oppdateringsmetodene for eksempelkortet.

2.  [Microsoft Learn: Suspend-BitLocker](https://learn.microsoft.com/en-us/powershell/module/bitlocker/suspend-bitlocker): Referanse for å sette BitLocker-beskyttelsen på pause, inkludert parameteren RebootCount.

3.  [Microsoft Learn: Advanced troubleshooting for Event ID 41](https://learn.microsoft.com/en-us/troubleshoot/windows-client/performance/event-id-41-restart): Forklaring av Kernel-Power 41 og betydningen av BugcheckCode 0.

4.  [Microsoft Learn: shutdown-kommando](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/shutdown): Dokumentasjon av parameterne /fw og /o for omstart til henholdsvis UEFI og gjenopprettingsmiljøet.
