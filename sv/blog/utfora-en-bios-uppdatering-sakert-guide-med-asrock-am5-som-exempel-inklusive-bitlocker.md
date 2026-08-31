---
title: "Uppdatera BIOS säkert: guide med ASRock AM5 som exempel, inklusive förberedelser för BitLocker"
navTitle: "BIOS-uppdatering"
description: "Det kompletta förfarandet för en BIOS-uppdatering med ett ASRock AM5-kort som exempel: ta reda på versionen, verifiera nedladdningen med hash, pausa BitLocker korrekt, starta i UEFI (även när F2 inte fungerar), uppdatera med Instant Flash och konfigurera inställningarna efter uppdateringen på ett lämpligt sätt."
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
translationSourceHash: 555b16e753b2ac5dec357741b071ed6aa33de367a2197a8dbb10fef7c9f6a946
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:15:23.392Z
translationReview: automatic
url: https://rafaelpfister.ch/sv/blog/utfora-en-bios-uppdatering-sakert-guide-med-asrock-am5-som-exempel-inklusive-bitlocker
---

En BIOS-uppdatering hör till de underhållsåtgärder som sällan behövs och därför väcker frågor varje gång: Vilken version är rätt, hur får man den säkert till moderkortet och vad behöver man tänka på före och efter? Den här guiden dokumenterar hela förfarandet med ett ASRock A620I Lightning WiFi (sockel AM5) och tillverkarens egen metod Instant Flash som exempel. Stegen kan överföras till alla moderna moderkort, och de kritiska punkterna (BitLocker, Fast Boot, återställning av inställningar) är oberoende av tillverkare.

## När en BIOS-uppdatering är motiverad

Tre skäl motiverar ingreppet. För det första säkerhetskorrigeringar: sårbarheter i firmware kan endast stängas genom en BIOS-uppdatering, och tillverkarnas ändringsloggar beskriver dem oftast bara kortfattat. För det andra kompatibilitet: stöd för nya CPU-generationer och förbättrad minneskompatibilitet kommer uteslutande genom nya firmwareversioner, för AM5 via AMD:s AGESA-referensfirmware som moderkortstillverkarna integrerar i sina BIOS-versioner. För det tredje stabilitet: om ett system startar om spontant och händelseloggen bara registrerar Kernel-Power 41 med `BugcheckCode=0`, har kraschen skett på hårdvaru- eller firmwarenivå utan medverkan av Windows; typiska orsaker är instabila spänningar och minnesträning, och det är just detta lager som AGESA-utgåvorna underhåller. Poster som "Improve memory compatibility and system stability" eller reviderad EXPO-hantering i ändringsloggarna visar att en uppdatering hanterar sådana problem. Om ett system däremot kör stabilt och inte påverkas av de åtgärdade sårbarheterna är det legitimt att avvakta; en BIOS-uppdatering utan anledning är en risk utan motprestation.

## Steg 1: Fastställ aktuellt läge

Innan du laddar ner något behöver du två uppgifter: den exakta moderkortsmodellen och den installerade BIOS-versionen. Båda kan hämtas med PowerShell utan omstart:

```powershell
Get-CimInstance Win32_ComputerSystem | Select-Object Manufacturer, Model
Get-CimInstance Win32_BIOS | Select-Object SMBIOSBIOSVersion, ReleaseDate
```

<details class="options-details">
<summary>Förklaringar av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `Win32_ComputerSystem` | Positionsargumentet ClassName: CIM-klass med systemets tillverkare och modell |
| `Win32_BIOS` | CIM-klass med uppgifter om firmware, inklusive version och datum |
| `Select-Object <eigenschaften>` | begränsar utdata till de angivna egenskaperna |

</details>

Anteckna versionen. Du behöver den senare för att kontrollera att uppdateringen lyckades, och när du läser ändringsloggarna vill du veta vilka versioner du hoppar över.

## Steg 2: Ladda ner BIOS och verifiera kontrollsumman

Ladda endast ner BIOS från tillverkarens produktsida, aldrig från tredjepartsportaler. ASRock publicerar SHA256-kontrollsumman för varje version; efter nedladdningen jämför du den innan filen ens kommer i närheten av ett USB-minne:

```powershell
Get-FileHash .\A620I_Lightning_WiFi_4.43.zip -Algorithm SHA256
```

<details class="options-details">
<summary>Förklaringar av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `.\A620I_…_4.43.zip` | Positionsargumentet Path: filen som ska kontrolleras |
| `-Algorithm SHA256` | Hash-algoritm; måste motsvara den kontrollsumstyp som tillverkaren har publicerat |

</details>

Om värdet inte stämmer överens med tillverkarens uppgift är nedladdningen skadad eller manipulerad: flasha inte. Efter uppackningen återstår en enskild ROM-fil, i exemplet `A62IRW_4.43.ROM` med 32 MB.

## Steg 3: Förbered USB-minnet

Flashmekanismen i UEFI (hos ASRock "Instant Flash", hos andra tillverkare Q-Flash, EZ Flash eller M-Flash) läser USB-minnet direkt från firmware. Det innebär att endast FAT32 identifieras tillförlitligt, inte NTFS eller exFAT. Nästan alla färdigköpta USB-minnen är redan FAT32; du kan kontrollera det så här:

```powershell
Get-CimInstance Win32_LogicalDisk -Filter "DriveType=2" |
  Select-Object DeviceID, FileSystem, VolumeName
```

<details class="options-details">
<summary>Förklaringar av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `Win32_LogicalDisk` | CIM-klass för logiska enheter |
| `-Filter "DriveType=2"` | WQL-filter för flyttbara lagringsmedia; döljer hårddiskar och CD-enheter |
| `Select-Object DeviceID, FileSystem, VolumeName` | visar enhetsbeteckning, filsystem och volymnamn |

</details>

Kopiera ROM-filen till USB-minnets rotkatalog. Omformatering behövs bara om filsystemet inte passar. USB-minnets storlek är oviktig; filen är mindre än all kapacitet som är vanlig i dag.

En anmärkning om val av metod: Många moderkort erbjuder även en BIOS Flashback-knapp som flashar utan CPU och utan ett fungerande system. Det är räddningsvägen för ett moderkort som inte längre startar. För ett fungerande system är Instant Flash i UEFI rätt och enklare väg. Windows-baserade flashverktyg är varken nödvändiga eller rekommenderade på moderna plattformar.

## Steg 4: Pausa BitLocker, annars riskerar du att behöva återställningsnyckeln

Detta är punkten som saknas i många guider. Om systemdisken är krypterad med BitLocker (i Windows 11 ofta automatiskt aktiverat med ett Microsoft-konto) binder BitLocker nyckeln till TPM:s mätvärden. En BIOS-uppdatering ändrar dessa mätvärden, och vid nästa start kräver Windows den 48-siffriga återställningsnyckeln. Den som inte har den till hands står inför ett otillgängligt system.

BitLocker har en egen mekanism för detta scenario. I PowerShell med administratörsbehörighet:

```powershell
Suspend-BitLocker -MountPoint C: -RebootCount 2
```

<details class="options-details">
<summary>Förklaringar av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-MountPoint C:` | den berörda volymen, här systemdisken |
| `-RebootCount 2` | antal omstarter som skyddet förblir pausat (0 till 15; 0 = till manuell återaktivering) |

</details>

Värdet 2 täcker båda kommande omstarterna (en gång till UEFI, en gång efter flashningen); därefter återaktiveras skyddet automatiskt och förseglar nyckeln mot de nya mätvärdena. Kontrollera oavsett detta i förväg att återställningsnyckeln går att hitta, till exempel i Microsoft-kontot på aka.ms/myrecoverykey eller via `manage-bde -protectors -get C:`.

## Steg 5: Kom in i UEFI, även om F2 inte reagerar

Det klassiska sättet via F2 eller Del vid påslagning misslyckas ofta på moderna system: med Fast Boot aktiverat initierar firmware USB-tangentbordet först efter POST, så tangenttryckningen registreras inte. Men du är inte beroende av tangenten: Windows kan styra nästa omstart direkt till UEFI-inställningarna. I PowerShell med administratörsbehörighet:

```powershell
shutdown /r /fw /t 5
```

<details class="options-details">
<summary>Förklaringar av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `/r` | startar om i stället för att stänga av |
| `/fw` | ställer in firmwarevariabeln som styr nästa start direkt till UEFI-inställningarna; endast tillsammans med ett avstängningsalternativ som `/r`, kräver administratörsbehörighet |
| `/t 5` | väntetid i sekunder före körning |

</details>

Om kommandot rapporterar fel 203 ("The system could not find the environment option that was entered") saknas nästan alltid administratörsbehörighet: utan höjda behörigheter får processen inte ange den nödvändiga firmwarevariabeln, och felmeddelandet nämner inte orsaken. En annan möjlig väg utan firmwarevariabel går via återställningsmiljön: `shutdown /r /o`, sedan Felsökning, Avancerade alternativ, UEFI-firmwareinställningar.

## Steg 6: Flasha med Instant Flash

I UEFI hittar du Instant Flash i verktygsmenyn. Verktyget listar alla ROM-filer på USB-minnet; efter valet kontrollerar det filen, flashar och startar om automatiskt. Under de få minuterna gäller den enda hårda regeln i hela processen: avbryt inte strömförsörjningen och stäng inte av datorn. En avbruten flashning är det enda steget i den här guiden som faktiskt kan göra moderkortet obootbart (och då kräver den nämnda Flashback-räddningsvägen).

## Steg 7: Efterarbete, eftersom uppdateringen återställer allt

Efter flashningen återställs samtliga BIOS-inställningar till fabriksinställningarna. Det är avsiktligt och ger en diagnostisk möjlighet: RAM-minnet kör nu utan EXPO-profil i JEDEC:s grundhastighet. Om du flashade på grund av stabilitetsproblem bör du medvetet låta det vara så i en till två veckor. Om krascherna uteblir var minnesprofilen inblandad, och du kan sedan testa EXPO igen med den nya firmwaren. Skillnaden i vardagen mellan 4800 och 6000 MT/s är knappt märkbar utanför benchmarktester; en stabil dator är värd varje benchmarkpoäng.

Två inställningar är ändå värda ett besök i UEFI: Den som haft omstarter vid tomgång kan under Advanced, AMD CBS ställa alternativet "Power Supply Idle Control" till "Typical Current Idle"; det mildrar en känd inkompatibilitet mellan vissa nätaggregat och Ryzen-processornas djupa tomgångslägen. Den som i framtiden vill komma in i inställningarna med F2 igen kan stänga av Fast Boot.

Kontrollera resultatet tillbaka i Windows:

```powershell
Get-CimInstance Win32_BIOS | Select-Object SMBIOSBIOSVersion
Get-CimInstance Win32_PhysicalMemory |
  Select-Object PartNumber, ConfiguredClockSpeed
manage-bde -status C:
```

<details class="options-details">
<summary>Förklaringar av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `Win32_BIOS` | CIM-klass med firmwareversionen; `SMBIOSBIOSVersion` måste nu visa den nya versionen |
| `Win32_PhysicalMemory` | CIM-klass för minnesmodulerna; `ConfiguredClockSpeed` visar den faktiska hastigheten i MT/s |
| `-status` | manage-bde: visar volymens krypterings- och skyddsstatus |
| `C:` | positionsargument: volymen som ska kontrolleras |

</details>

Den första raden måste visa den nya versionen, den andra den förväntade minneshastigheten, och BitLocker måste åter rapportera "Skydd aktiverat". Därmed är uppdateringen slutförd och dokumenterad. Om flashningen gjordes på grund av stabilitetsproblem visar först observation under de följande veckorna om de är åtgärdade, enklast genom att kontrollera nya Kernel-Power 41-poster i systemets händelselogg.

## Källor

1.  [ASRock A620I Lightning WiFi, BIOS-nedladdningar](https://pg.asrock.com/mb/AMD/A620I%20Lightning%20WiFi/index.asp#BIOS): Versionslista med ändringsloggar, SHA256-kontrollsummor och de uppdateringsmetoder som stöds för exempelmoderkortet.

2.  [Microsoft Learn: Suspend-BitLocker](https://learn.microsoft.com/en-us/powershell/module/bitlocker/suspend-bitlocker): Referens för att pausa BitLocker-skyddet, inklusive parametern RebootCount.

3.  [Microsoft Learn: Advanced troubleshooting for Event ID 41](https://learn.microsoft.com/en-us/troubleshoot/windows-client/performance/event-id-41-restart): Klassificering av Kernel-Power 41 och innebörden av BugcheckCode 0.

4.  [Microsoft Learn: shutdown-kommando](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/shutdown): Dokumentation av parametrarna /fw och /o för omstart till UEFI respektive återställningsmiljön.
