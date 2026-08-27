---
title: "Utfør en BIOS-oppdatering på en trygg måte: Veiledning med ASRock AM5 som eksempel, inkludert BitLocker-forberedelse"
navTitle: "BIOS-oppdatering"
description: "Hele prosessen for en BIOS-oppdatering, med et ASRock AM5-kort som eksempel: Finn versjonen, verifiser nedlastingen med hash, sett BitLocker korrekt på pause, start opp i UEFI (også når F2 ikke virker), oppdater med Instant Flash og konfigurer innstillingene fornuftig etter oppdateringen."
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
url: https://rafaelpfister.ch/no/blog/utfor-en-bios-oppdatering-pa-en-trygg-mate-veiledning-med-asrock-am5-som-eksempel-inkludert
translationSourceHash: 60fff28a10b0f91f3d59996b00afe614f2230b9831514dcf01a1f496b99f4fbd
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:27:26.221Z
translationReview: automatic
---

En BIOS-oppdatering er blant vedlikeholdsoppgavene som sjelden er aktuelle, og som derfor reiser spørsmål hver gang: Hvilken versjon er riktig, hvordan får man den trygt over på hovedkortet, og hva må man ta hensyn til før og etter? Denne veiledningen dokumenterer hele prosessen med et ASRock A620I Lightning WiFi (AM5-sokkel) og produsentens egen metode, Instant Flash, som eksempel. Trinnene kan overføres til ethvert moderne hovedkort; de kritiske punktene (BitLocker, Fast Boot, tilbakestilling av innstillinger) er uavhengige av produsent.

## Når en BIOS-oppdatering er nødvendig

Tre forhold rettferdiggjør inngrepet. For det første sikkerhetsrettinger: Sårbarheter i fastvaren kan bare lukkes gjennom en BIOS-oppdatering, og produsentenes endringslogger omtaler dem vanligvis bare kort. For det andre kompatibilitet: Støtte for nye CPU-generasjoner og forbedret minnekompatibilitet kommer utelukkende gjennom nye fastvareversjoner, på AM5 via AMDs AGESA-referansefastvare, som hovedkortprodusentene bygger inn i BIOS-versjonene sine. For det tredje stabilitet: Hvis et system starter på nytt spontant og hendelsesloggen bare registrerer Kernel-Power 41 med `BugcheckCode=0`, har krasjet skjedd på maskinvare- eller fastvarenivå uten medvirkning fra Windows; typiske årsaker er ustabile spenninger og minnetrening, og nettopp dette nivået vedlikeholdes av AGESA-utgivelsene. Oppføringer som "Improve memory compatibility and system stability" eller revidert EXPO-håndtering i endringsloggene indikerer at en oppdatering adresserer slike problemer. Hvis systemet derimot kjører stabilt og ikke er berørt av de rettede sårbarhetene, er det legitimt å vente; en BIOS-oppdatering uten grunn er en risiko uten gevinst.

## Trinn 1: Fastslå nåværende tilstand

Før du laster ned noe, trenger du to opplysninger: nøyaktig hovedkortmodell og installert BIOS-versjon. PowerShell gir deg begge uten omstart:

```powershell
Get-CimInstance Win32_ComputerSystem | Select-Object Manufacturer, Model
Get-CimInstance Win32_BIOS | Select-Object SMBIOSBIOSVersion, ReleaseDate
```

Noter deg versjonen. Du trenger den senere for å kontrollere resultatet, og når du leser endringsloggene, vil du vite hvilke versjoner du hopper over.

## Trinn 2: Last ned BIOS og verifiser kontrollsummen

Last ned BIOS utelukkende fra produsentens produktside, aldri fra tredjepartsportaler. ASRock publiserer SHA256-kontrollsummen for hver versjon; etter nedlastingen sammenligner du den før filen i det hele tatt kommer i nærheten av en USB-pinne:

```powershell
Get-FileHash .\A620I_Lightning_WiFi_4.43.zip -Algorithm SHA256
```

Hvis verdien ikke stemmer overens med produsentens opplysning, er nedlastingen skadet eller manipulert: Ikke flash. Etter utpakking gjenstår én enkelt ROM-fil, i eksemplet `A62IRW_4.43.ROM` på 32 MB.

## Trinn 3: Klargjør USB-pinnen

Flash-mekanismen i UEFI (hos ASRock "Instant Flash", hos andre produsenter Q-Flash, EZ Flash eller M-Flash) leser pinnen direkte fra fastvaren. Det betyr at bare FAT32 gjenkjennes pålitelig, ikke NTFS og exFAT. Nesten enhver ferdigkjøpt pinne er allerede FAT32; du kan kontrollere det slik:

```powershell
Get-CimInstance Win32_LogicalDisk -Filter "DriveType=2" |
  Select-Object DeviceID, FileSystem, VolumeName
```

Kopier ROM-filen til rotkatalogen på pinnen. Omformatering er bare nødvendig hvis filsystemet ikke passer. Størrelsen på pinnen er uvesentlig; filen er mindre enn enhver vanlig kapasitet i dag.

En merknad om valg av metode: Mange hovedkort tilbyr i tillegg en BIOS Flashback-knapp som flasher uten CPU og uten et fungerende system. Dette er redningsveien for et hovedkort som ikke lenger starter. For et system som fungerer, er Instant Flash i UEFI den riktige og enklere veien. Windows-baserte flash-verktøy er verken nødvendige eller anbefalte på dagens plattformer.

## Trinn 4: Sett BitLocker på pause, ellers risikerer du forespørsel om nøkkel

Dette er punktet som mangler i mange veiledninger. Hvis systemdisken er kryptert med BitLocker (ofte aktivert automatisk i Windows 11 med Microsoft-konto), binder BitLocker nøkkelen til måleverdiene fra TPM. En BIOS-oppdatering endrer disse måleverdiene, og ved neste oppstart ber Windows om den 48-sifrede gjenopprettingsnøkkelen. Den som ikke har den lett tilgjengelig, står igjen med et utilgjengelig system.

BitLocker har en egen mekanisme for dette scenariet. I PowerShell med administratorrettigheter:

```powershell
Suspend-BitLocker -MountPoint C: -RebootCount 2
```

Parameteren `RebootCount 2` dekker begge de forestående omstartene (én til UEFI, én etter flashingen); deretter aktiveres beskyttelsen automatisk igjen og forsegler nøkkelen mot de nye måleverdiene. Kontroller likevel på forhånd at gjenopprettingsnøkkelen er tilgjengelig, for eksempel i Microsoft-kontoen på aka.ms/myrecoverykey eller med `manage-bde -protectors -get C:`.

## Trinn 5: Kom inn i UEFI, også når F2 ikke reagerer

Den klassiske metoden via F2 eller Delete ved oppstart mislykkes ofte på moderne systemer: Når Fast Boot er aktivert, initialiserer fastvaren USB-tastaturet først etter POST, slik at tastetrykket ikke når fram. Du er imidlertid ikke avhengig av tasten: Windows kan sende neste omstart direkte til UEFI-oppsettet. I PowerShell med administratorrettigheter:

```powershell
shutdown /r /fw /t 5
```

Hvis kommandoen rapporterer feil 203 ("The system could not find the environment option that was entered"), mangler det nesten alltid administratorrettigheter: Uten hevede rettigheter kan ikke prosessen sette den nødvendige fastvarevariabelen, og feilmeldingen oppgir ikke denne årsaken. En annen mulig vei uten fastvarevariabel går via gjenopprettingsmiljøet: `shutdown /r /o`, deretter Feilsøking, Avanserte alternativer, UEFI-fastvareinnstillinger.

## Trinn 6: Flash med Instant Flash

I UEFI finner du Instant Flash i Tool-menyen. Verktøyet viser alle ROM-filer på pinnen; etter at du har valgt en, kontrollerer det filen, flasher og starter på nytt automatisk. I løpet av de få minuttene gjelder den eneste absolutte regelen i hele prosessen: Ikke avbryt strømforsyningen og ikke slå av datamaskinen. En avbrutt flashing er det eneste trinnet i denne veiledningen som faktisk kan gjøre hovedkortet ute av stand til å starte (og da krever den nevnte Flashback-redningsveien).

## Trinn 7: Etterarbeid, fordi oppdateringen tilbakestiller alt

Etter flashingen er alle BIOS-innstillinger satt til fabrikkinnstillinger. Dette er tilsiktet og gir en diagnostisk mulighet: RAM kjører nå uten EXPO-profil på JEDEC-grunnhastigheten. Hvis du flashet på grunn av stabilitetsproblemer, bør du bevisst la det være slik i én til to uker. Hvis krasjene uteblir, var minneprofilen involvert, og du kan teste EXPO målrettet på nytt med den nye fastvaren. Forskjellen i daglig bruk mellom 4800 og 6000 MT/s er knapt merkbar utenfor referansetester; en stabil datamaskin er verdt hvert eneste poeng i en referansetest.

To innstillinger er uansett verdt et besøk i UEFI: De som har hatt omstarter i tomgang, kan under Advanced, AMD CBS sette alternativet "Power Supply Idle Control" til "Typical Current Idle"; dette avhjelper en kjent inkompatibilitet mellom enkelte strømforsyninger og de dype tomgangstilstandene til Ryzen-CPU-er. Og de som igjen vil komme inn i oppsettet med F2, kan deaktivere Fast Boot.

Kontroll av resultatet tilbake i Windows:

```powershell
Get-CimInstance Win32_BIOS | Select-Object SMBIOSBIOSVersion
Get-CimInstance Win32_PhysicalMemory |
  Select-Object PartNumber, ConfiguredClockSpeed
manage-bde -status C:
```

Den første linjen må vise den nye versjonen, den andre den forventede minnehastigheten, og BitLocker må igjen rapportere "Beskyttelse aktivert". Dermed er oppdateringen fullført og dokumentert. Hvis det ble flashet på grunn av stabilitetsproblemer, er det først observasjon gjennom de påfølgende ukene som viser om de er løst, enklest ved å se etter nye Kernel-Power-41-oppføringer i systemhendelsesloggen.

## Kilder

1.  [ASRock A620I Lightning WiFi, BIOS-nedlastinger](https://pg.asrock.com/mb/AMD/A620I%20Lightning%20WiFi/index.asp#BIOS): Versjonsliste med endringslogger, SHA256-kontrollsummer og støttede oppdateringsmetoder for hovedkortet i eksemplet.

2.  [Microsoft Learn: Suspend-BitLocker](https://learn.microsoft.com/en-us/powershell/module/bitlocker/suspend-bitlocker): Referanse for å sette BitLocker-beskyttelsen på pause, inkludert parameteren RebootCount.

3.  [Microsoft Learn: Advanced troubleshooting for Event ID 41](https://learn.microsoft.com/en-us/troubleshoot/windows-client/performance/event-id-41-restart): Forklaring av Kernel-Power 41 og betydningen av BugcheckCode 0.

4.  [Microsoft Learn: shutdown-kommandoen](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/shutdown): Dokumentasjon av parameterne /fw og /o for omstart til henholdsvis UEFI og gjenopprettingsmiljøet.
