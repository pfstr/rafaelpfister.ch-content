---
title: "Utföra en BIOS-uppdatering säkert: guide med ASRock AM5 som exempel, inklusive BitLocker-förberedelse"
navTitle: "BIOS-uppdatering"
description: "Det kompletta förloppet för en BIOS-uppdatering med ett ASRock AM5-kort som exempel: fastställ version, verifiera nedladdningen med hash, pausa BitLocker korrekt, starta i UEFI (även när F2 inte fungerar), uppdatera med Instant Flash och konfigurera inställningarna lämpligt efter uppdateringen."
date: "2026-08-26"
kategorie: "PC & hårdvara"
timeToRead: "8 min läsning"
themen:
  - pc-hardware
produkte:
  - "windows-client"
protokolle:
  - "troubleshooting"
  - "releases"
slug: "utfora-en-bios-uppdatering-sakert-guide-med-asrock-am5-som-exempel-inklusive-bitlocker"
translationId: "article-82840b2d159b9367"
translationOf: bios-update-asrock-am5-sicher-durchfuehren
url: https://rafaelpfister.ch/sv/blog/utfora-en-bios-uppdatering-sakert-guide-med-asrock-am5-som-exempel-inklusive-bitlocker
translationSourceHash: 60fff28a10b0f91f3d59996b00afe614f2230b9831514dcf01a1f496b99f4fbd
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:26:56.676Z
translationReview: automatic
---

En BIOS-uppdatering är ett av de underhållsarbeten som sällan behövs och därför väcker frågor varje gång: Vilken version är rätt, hur får man den säkert till moderkortet och vad behöver göras före och efter? Den här guiden dokumenterar hela processen med ett ASRock A620I Lightning WiFi (AM5-sockel) och tillverkarens egen metod Instant Flash som exempel. Stegen kan överföras till alla moderna moderkort, och de kritiska punkterna (BitLocker, Fast Boot, återställning av inställningar) är oberoende av tillverkare.

## När en BIOS-uppdatering är motiverad

Tre skäl motiverar ingreppet. För det första säkerhetskorrigeringar: sårbarheter i firmware kan endast stängas genom en BIOS-uppdatering, och tillverkarnas ändringsloggar nämner dem oftast bara kortfattat. För det andra kompatibilitet: stöd för nya CPU-generationer och förbättrad minneskompatibilitet kommer uteslutande genom nya firmware-versioner, för AM5 via AMD:s AGESA-referensfirmware som moderkortstillverkarna bäddar in i sina BIOS-versioner. För det tredje stabilitet: om ett system startar om spontant och händelseloggen endast registrerar Kernel-Power 41 med `BugcheckCode=0`, har kraschen inträffat på hårdvaru- eller firmwarenivå utan medverkan av Windows; typiska orsaker är instabila spänningar och minnesträning, och det är just denna nivå som AGESA-utgåvorna underhåller. Poster som "Improve memory compatibility and system stability" eller en omarbetad EXPO-hantering i ändringsloggarna är en indikation på att en uppdatering åtgärdar sådana problem. Om ett system däremot fungerar stabilt och inte påverkas av de åtgärdade sårbarheterna, är det legitimt att avvakta; en BIOS-uppdatering utan anledning är en risk utan motprestation.

## Steg 1: Fastställ aktuellt läge

Innan du laddar ned något behöver du två uppgifter: exakt moderkortsmodell och installerad BIOS-version. PowerShell ger båda utan omstart:

```powershell
Get-CimInstance Win32_ComputerSystem | Select-Object Manufacturer, Model
Get-CimInstance Win32_BIOS | Select-Object SMBIOSBIOSVersion, ReleaseDate
```

Anteckna versionen. Du behöver den senare för att kontrollera resultatet, och när du läser ändringsloggarna vill du veta vilka versioner du hoppar över.

## Steg 2: Ladda ned BIOS och verifiera kontrollsumman

Ladda endast ned BIOS från tillverkarens produktsida, aldrig från tredjepartsportaler. ASRock publicerar SHA256-kontrollsumman för varje version; jämför den efter nedladdningen innan filen ens kommer i närheten av ett USB-minne:

```powershell
Get-FileHash .\A620I_Lightning_WiFi_4.43.zip -Algorithm SHA256
```

Om värdet inte stämmer överens med tillverkarens uppgift är nedladdningen skadad eller manipulerad: flasha inte. Efter uppackning återstår en enda ROM-fil, i exemplet `A62IRW_4.43.ROM` på 32 MB.

## Steg 3: Förbered USB-minnet

Flashmekanismen i UEFI (hos ASRock "Instant Flash", hos andra tillverkare Q-Flash, EZ Flash eller M-Flash) läser USB-minnet direkt från firmware. Det innebär att endast FAT32 identifieras tillförlitligt, inte NTFS och exFAT. Nästan alla färdigköpta USB-minnen är redan FAT32; du kan kontrollera det så här:

```powershell
Get-CimInstance Win32_LogicalDisk -Filter "DriveType=2" |
  Select-Object DeviceID, FileSystem, VolumeName
```

Kopiera ROM-filen till USB-minnets rotkatalog. Omformatering behövs endast om filsystemet inte stämmer. USB-minnets storlek spelar ingen roll, filen är mindre än alla vanliga kapaciteter i dag.

En anmärkning om metodval: Många moderkort erbjuder även en BIOS Flashback-knapp, som flashar utan CPU och utan ett fungerande system. Det är räddningsvägen för ett moderkort som inte längre startar. För ett fungerande system är Instant Flash i UEFI den rätta och enklare vägen. Windows-baserade flashverktyg är varken nödvändiga eller rekommenderade på aktuella plattformar.

## Steg 4: Pausa BitLocker, annars riskerar du att bli ombedd om nyckeln

Det här är punkten som saknas i många guider. Om systemdisken är krypterad med BitLocker (i Windows 11 ofta automatiskt aktiverat med Microsoft-konto) binder BitLocker nyckeln till TPM:s mätvärden. En BIOS-uppdatering ändrar dessa mätvärden, och vid nästa start kräver Windows den 48-siffriga återställningsnyckeln. Den som inte har den tillgänglig står inför ett oåtkomligt system.

BitLocker har en egen mekanism för detta scenario. I PowerShell med administratörsbehörighet:

```powershell
Suspend-BitLocker -MountPoint C: -RebootCount 2
```

Parametern `RebootCount 2` täcker båda kommande omstarterna (en gång till UEFI, en gång efter flashningen); därefter återaktiveras skyddet automatiskt och förseglar nyckeln mot de nya mätvärdena. Kontrollera oavsett detta i förväg att återställningsnyckeln går att hitta, till exempel i Microsoft-kontot under aka.ms/myrecoverykey eller via `manage-bde -protectors -get C:`.

## Steg 5: Kom in i UEFI, även om F2 inte svarar

Den klassiska vägen via F2 eller Delete vid uppstart misslyckas ofta på moderna system: med Fast Boot aktiverat initierar firmware USB-tangentbordet först efter POST, så tangenttryckningen registreras inte. Du är dock inte beroende av tangenten: Windows kan styra nästa omstart direkt till UEFI-inställningarna. I PowerShell med administratörsbehörighet:

```powershell
shutdown /r /fw /t 5
```

Om kommandot rapporterar fel 203 ("The system could not find the environment option that was entered") saknas nästan alltid administratörsbehörighet: utan höjda behörigheter får processen inte ange den nödvändiga firmware-variabeln, och felmeddelandet anger inte denna orsak. En andra möjlig väg utan firmware-variabel går via återställningsmiljön: `shutdown /r /o`, sedan Felsökning, Avancerade alternativ, UEFI-firmwareinställningar.

## Steg 6: Flasha med Instant Flash

I UEFI hittar du Instant Flash i Tool-menyn. Verktyget listar alla ROM-filer på USB-minnet; efter valet kontrollerar det filen, flashar och startar om automatiskt. Under de få minuterna gäller den enda hårda regeln i hela processen: avbryt inte strömförsörjningen och stäng inte av datorn. En avbruten flashning är det enda steget i denna guide som faktiskt kan göra moderkortet obootbart (och då kräver den nämnda räddningsvägen med Flashback).

## Steg 7: Efterarbete, eftersom uppdateringen återställer allt

Efter flashningen återställs samtliga BIOS-inställningar till fabriksinställningarna. Det är avsiktligt och ger en diagnostisk möjlighet: RAM-minnet körs nu utan EXPO-profil i JEDEC-grundhastigheten. Om du flashade på grund av stabilitetsproblem bör du medvetet låta det vara så i en till två veckor. Om krascherna uteblir var minnesprofilen inblandad, och du kan sedan testa EXPO igen med den nya firmwaren. Skillnaden i vardagen mellan 4800 och 6000 MT/s märks knappt utanför benchmarks; en stabil dator är värd varje benchmark-poäng.

Två inställningar är ändå värda ett besök i UEFI: Om omstarterna inträffade vid tomgång kan du under Advanced, AMD CBS ställa alternativet "Power Supply Idle Control" till "Typical Current Idle"; det mildrar en känd inkompatibilitet mellan vissa nätaggregat och Ryzen-CPU:ernas djupa tomgångslägen. Och om du framöver vill komma in i inställningarna med F2 igen kan du stänga av Fast Boot.

Resultatkontrollen tillbaka i Windows:

```powershell
Get-CimInstance Win32_BIOS | Select-Object SMBIOSBIOSVersion
Get-CimInstance Win32_PhysicalMemory |
  Select-Object PartNumber, ConfiguredClockSpeed
manage-bde -status C:
```

Den första raden måste visa den nya versionen, den andra den förväntade minneshastigheten, och BitLocker måste åter rapportera "Skydd aktiverat". Därmed är uppdateringen slutförd och dokumenterad. Om flashningen gjordes på grund av stabilitetsproblem visar först observation under de följande veckorna om de har lösts, enklast genom att titta efter nya Kernel-Power-41-poster i systemets händelselogg.

## Källor

1.  [ASRock A620I Lightning WiFi, BIOS-nedladdningar](https://pg.asrock.com/mb/AMD/A620I%20Lightning%20WiFi/index.asp#BIOS): Versionslista med ändringsloggar, SHA256-kontrollsummor och de uppdateringsmetoder som stöds av moderkortet i exemplet.

2.  [Microsoft Learn: Suspend-BitLocker](https://learn.microsoft.com/en-us/powershell/module/bitlocker/suspend-bitlocker): Referens för att pausa BitLocker-skyddet, inklusive parametern RebootCount.

3.  [Microsoft Learn: Advanced troubleshooting for Event ID 41](https://learn.microsoft.com/en-us/troubleshoot/windows-client/performance/event-id-41-restart): Klassificering av Kernel-Power 41 och betydelsen av BugcheckCode 0.

4.  [Microsoft Learn: shutdown-kommandot](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/shutdown): Dokumentation av parametrarna /fw och /o för omstart till UEFI respektive återställningsmiljön.
