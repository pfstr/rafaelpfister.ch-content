---
title: "RDP-skrivaromdirigering: Skriv ut via den lokala datorn i stället för via fjärrdatorn"
navTitle: "RDP-skrivaromdirigering"
description: "Utskriftsjobb från RDP-sessionen ska hamna på skrivaren bredvid användaren, inte på fjärrdatorn. Inställningen finns på tre ställen: i RDP-klienten, i .rdp-filen och på målsystemet. Dessutom beskrivs hanteringen av varningen ”Okänd utgivare” och en felsökningschecklista."
date: "2026-08-24"
kategorie: "Windows-klient"
timeToRead: "5 min lästid"
themen:
  - windows-client
produkte:
  - "windows-client"
protokolle:
  - "troubleshooting"
slug: "rdp-skrivaromdirigering-skriv-ut-via-den-lokala-datorn-i-stallet-for-via-fjarrdatorn"
translationId: "article-12521248666e9809"
draft: false
translationOf: rdp-druckerumleitung-lokale-drucker
translationSourceHash: 1ca6c78df61e05f90d1aefc5b20b611e5c6a4bf03908d0466da114c2823e31cd
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:33:42.832Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/rdp-skrivaromdirigering-skriv-ut-via-den-lokala-datorn-i-stallet-for-via-fjarrdatorn
---

# RDP-skrivaromdirigering: Skriv ut via den lokala datorn i stället för via fjärrdatorn

En användare arbetar via Remote Desktop på en fjärrdator och vill skriva ut på skrivaren som står bredvid honom eller henne. Det är precis vad skrivaromdirigering är till för: RDP-klienten registrerar de lokala skrivarna i sessionen, utskriftsjobbet går tillbaka till klienten via RDP-kanalen och skrivs ut där. På målsystemet visas skrivaren med tillägget **(omdirigerad, session n)**. Drivrutiner på fjärrdatorn behövs normalt inte för detta: Windows använder den universella drivrutinen **Remote Desktop Easy Print**; rätt skrivardrivrutin behöver bara vara installerad på den lokala klienten.

Omstyrningen tillämpas endast när anslutningen upprättas. Efter varje ändring av inställningarna måste sessionen kopplas från helt och anslutas på nytt; det räcker inte att bara minimera RDP-fönstret.

## Klientsidan: aktivera omdirigering

Det enklaste sättet att aktivera skrivaromdirigering är via det grafiska gränssnittet: starta `mstsc`, välj **Visa alternativ**, fliken **Lokala resurser**, markera **Skrivare** och spara anslutningen på fliken **Allmänt**. Den som arbetar med .rdp-filer kan anpassa raden direkt i filen; .rdp-filer är vanliga textfiler och kan redigeras med valfri redigerare:

```text
redirectprinters:i:1
```

En särskildhet för genvägar utan .rdp-fil: Om anslutningen startas med `mstsc /v:hostname` gäller inställningarna från den dolda filen `Default.rdp` i användarens Dokument-mapp. Om raden `redirectprinters:i:1` saknas där visas ingen skrivare, även om allt verkar vara korrekt konfigurerat. Det här kodavsnittet lägger till raden idempotent (befintlig `0` ändras till `1`, saknad rad läggs till) och visar resultatet för kontroll:

```powershell
$f = "$env:USERPROFILE\Documents\Default.rdp"
if (Test-Path $f) {
    $c = Get-Content $f
    if ($c -match 'redirectprinters') {
        $c -replace 'redirectprinters:i:0', 'redirectprinters:i:1' | Set-Content $f
    } else {
        Add-Content $f 'redirectprinters:i:1'
    }
} else {
    Set-Content $f 'redirectprinters:i:1'
}
Select-String -Path $f -Pattern 'redirectprinters'
```

Två ytterligare felkällor på klientsidan: För det första sparar Windows per måldator under `HKCU\Software\Microsoft\Terminal Server Client\LocalDevices` vilka omdirigeringar användaren senast tillät i säkerhetsdialogrutan; detta sparade val åsidosätter inställningen i .rdp-filen. Om nyckeln tas bort återställs tillståndet. För det andra inaktiverar registervärdet `DisablePrinterRedirection` (DWORD, värde 1) under `HKLM\Software\Microsoft\Terminal Server Client` skrivaromdirigering helt på klienten; på hanterade enheter är det värt att kontrollera detta innan felsökningen börjar i sessionen.

## Serversidan: tillåt omdirigering

På målsystemet avgör principen **Tillåt inte omdirigering av klientskrivare** (Datorkonfiguration → Administrativa mallar → Windows-komponenter → Fjärrskrivbordstjänster → Värd för fjärrskrivbordssessioner → Skrivaromdirigering). Om den är *Aktiverad* skapas inga klientskrivare, oavsett vad klienten begär. Principen om den mest restriktiva inställningen gäller: Om någon av de två sidorna blockerar omdirigering sker den inte.

Utan grupprincip styrs samma mekanism via registret: `fDisableCpm` under `HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp` (0 = omdirigering tillåten, 1 = blockerad). Dessutom måste tjänsten **Utskriftskö** köras på målsystemet; utan spoolern skapas inte heller omdirigerade skrivare.

I samma GPO-kategori finns två ytterligare användbara inställningar: **Använd först skrivardrivrutinen Remote Desktop Easy Print** (standard och oftast rätt val) och **Ange klientens standardskrivare som standardskrivare för sessionen**.

## Varningen ”Okänd utgivare”

När en osignerad .rdp-fil som begär enhetsomdirigeringar öppnas visar klienten en säkerhetsvarning med kryssrutor för de enskilda resurserna. Markeringar som läggs till eller tas bort där gäller endast för denna anslutningsstart, men sparas i den tidigare nämnda nyckeln `LocalDevices` och påverkar därmed framtida anslutningar i det tysta. Den som undrar varför kryssrutan för skrivaren gång på gång saknas trots korrekt .rdp-fil hittar nästan alltid orsaken där.

Sedan säkerhetsuppdateringen från april 2026 (CVE-2026-26151) visar Remote Desktop Connection en ny säkerhetsdialog varje gång en .rdp-fil öppnas, innan anslutningen upprättas. Alla omdirigeringar som filen begär, inklusive skrivare, är där avstängda som standard och måste aktiveras var för sig. Osignerade filer får dessutom meddelandet ”Caution: Unknown remote connection”. Anslutningar som startas direkt med ett datornamn i Remote Desktop Connection påverkas enligt Microsoft inte. Som en tillfällig reservlösning dokumenterar Microsoft registervärdet `RedirectionWarningDialogVersion` (DWORD, värde 1) under `HKLM\Software\Policies\Microsoft\Windows NT\Terminal Services\Client`, som återställer den tidigare dialogrutan; stödet för detta kan försvinna i en framtida uppdatering.

Det finns tre sätt att hantera varningen, i stigande arbetsinsats. För det första: starta anslutningen via `mstsc /v:hostname` i stället för via .rdp-filen; utan fil sker ingen kontroll av utgivaren och inställningarna kommer från `Default.rdp`. Inte heller säkerhetsdialogen som infördes i april 2026 visas då. För det andra: godkänn omdirigeringarna för måldatorn i förväg via registret, så försvinner resursdelen i den tidigare dialogrutan. Microsoft dokumenterar inte om detta förhandsgodkännande även gäller den nya säkerhetsdialogen sedan april 2026:

```powershell
$key = "HKCU:\Software\Microsoft\Terminal Server Client\LocalDevices"
New-Item -Path $key -Force | Out-Null
Set-ItemProperty -Path $key -Name "hostname-oder-ip" -Value 0xFFFFFFFF -Type DWord
```

För det tredje, den rena lösningen för distribuerade .rdp-filer i företaget: signera filen med `rdpsign.exe` och ett certifikat, och lagra certifikatets tumavtryck som betrodd utgivare via GPO. För enskilda arbetsstationer är insatsen sällan värd det, men för centralt distribuerade anslutningsfiler är det rätt lösning.

## Felsökningschecklista

Om skrivaren inte visas i sessionen, kontrollera i denna ordning:

1. **Ansluten på nytt?** Omdirigeringen tillämpas endast när anslutningen upprättas, inte i en befintlig session.
2. **Rätt fil?** För genvägar, kontrollera vilken .rdp-fil som faktiskt öppnas; vid `mstsc /v:` är `Default.rdp` avgörande.
3. **Sparat val?** Kontrollera eller ta bort nyckeln `LocalDevices` på klienten. Vid start via en .rdp-fil ska du dessutom kontrollera om skrivaren har aktiverats i säkerhetsdialogrutan; sedan april 2026 är den avstängd där som standard.
4. **Klientblockering?** `DisablePrinterRedirection` under `HKLM\Software\Microsoft\Terminal Server Client` får inte vara satt till 1.
5. **Serverblockering?** Kontrollera GPO:n ”Tillåt inte omdirigering av klientskrivare” respektive `fDisableCpm` på målsystemet.
6. **Spooler?** Tjänsten Utskriftskö måste köras på målsystemet.
7. **Kontroll i sessionen:** `Get-Printer | Where-Object DriverName -eq "Remote Desktop Easy Print"` listar de omdirigerade skrivarna inklusive sessions-ID.

## Ändringshistorik

- **25 oktober 2026:** Tillagt: Sedan säkerhetsuppdateringen från april 2026 (CVE-2026-26151) visar en ny dialog alla omdirigeringar, även skrivare, som avstängda som standard när .rdp-filer öppnas; dessutom registervärdet RedirectionWarningDialogVersion och ett ytterligare kontrollsteg.

## Källor

1.  [Supported RDP properties](https://learn.microsoft.com/en-us/azure/virtual-desktop/rdp-properties): Referens över alla .rdp-egenskaper, inklusive redirectprinters med värden och standardvärde.

2.  [Configure printer redirection over the Remote Desktop Protocol](https://learn.microsoft.com/en-us/azure/virtual-desktop/redirection-configure-printers): Easy Print, GPO- och Intune-konfiguration, DisablePrinterRedirection och testet med Get-Printer.

3.  [rdpsign](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/rdpsign): Kommandoreferens för signering av .rdp-filer med certifikatets tumavtryck.

4.  [Understanding security warnings when opening Remote Desktop (RDP) files](https://learn.microsoft.com/en-us/windows-server/remote/remote-desktop-services/remotepc/understanding-security-warnings): Ny säkerhetsdialog sedan uppdateringen från april 2026, omdirigeringar som är avstängda som standard och registervärdet RedirectionWarningDialogVersion.
