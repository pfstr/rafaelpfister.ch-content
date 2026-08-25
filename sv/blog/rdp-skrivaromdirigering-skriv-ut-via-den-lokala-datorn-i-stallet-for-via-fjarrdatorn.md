---
title: "RDP-skrivaromdirigering: Skriv ut via den lokala datorn i stället för via fjärrdatorn"
navTitle: "RDP-skrivaromdirigering"
description: "Utskriftsjobb från RDP-sessionen ska hamna på skrivaren bredvid användaren, inte på fjärrdatorn. Inställningen finns på tre ställen: i RDP-klienten, i .rdp-filen och på målsystemet. Dessutom hantering av varningen «Okänd utgivare» och en checklista för felsökning."
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
url: https://rafaelpfister.ch/sv/blog/rdp-skrivaromdirigering-skriv-ut-via-den-lokala-datorn-i-stallet-for-via-fjarrdatorn
translationSourceHash: a4f12f591e9dcb86f8ebdd3ff8af1008a130c3ec65424abe789ad4d6446eb4c2
translationModel: gpt-5.6-terra
translatedAt: 2026-08-25T04:14:57.993Z
translationReview: automatic
---

# RDP-skrivaromdirigering: Skriv ut via den lokala datorn i stället för via fjärrdatorn

En användare arbetar via Remote Desktop på en fjärrdator och vill skriva ut på skrivaren som står bredvid honom eller henne. Det är precis vad skrivaromdirigering är till för: RDP-klienten registrerar de lokala skrivarna i sessionen, utskriftsjobbet går tillbaka till klienten via RDP-kanalen och skrivs ut där. På målsystemet visas skrivaren med tillägget **(omdirigerad, session n)**. Drivrutiner på fjärrdatorn behövs vanligtvis inte för detta: Windows använder den universella drivrutinen **Remote Desktop Easy Print**; rätt skrivardrivrutin behöver bara vara installerad på den lokala klienten.

Omdirigeringen tillämpas bara när anslutningen upprättas. Efter varje ändring av inställningarna måste sessionen kopplas från helt och anslutas igen; det räcker inte att bara minimera RDP-fönstret.

## Klientsidan: aktivera omdirigering

Det snabbaste sättet går via det grafiska gränssnittet: starta `mstsc`, välj **Visa alternativ**, fliken **Lokala resurser**, markera **Skrivare** och spara anslutningen på fliken **Allmänt**. Den som i stället arbetar med .rdp-filer anger motsvarande rad direkt i filen; .rdp-filer är enkla textfiler och kan redigeras med valfri textredigerare:

```text
redirectprinters:i:1
```

En fallgrop med genvägar utan .rdp-fil: Om anslutningen startas med `mstsc /v:hostname` gäller inställningarna från den dolda filen `Default.rdp` i användarens Dokument-mapp. Saknas raden `redirectprinters:i:1` där, visas inte skrivaren trots att allt verkar vara korrekt konfigurerat. Detta kodstycke lägger till raden idempotent (befintlig `0` blir `1`, saknad rad läggs till) och visar resultatet för kontroll:

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

Ytterligare två fallgropar på klientsidan: För det första kommer Windows per måldator ihåg under `HKCU\Software\Microsoft\Terminal Server Client\LocalDevices` vilka omdirigeringar användaren senast tillät i säkerhetsdialogrutan; detta sparade val åsidosätter inställningen i .rdp-filen. Om nyckeln tas bort återställs läget. För det andra inaktiverar registervärdet `DisablePrinterRedirection` (DWORD, värde 1) under `HKLM\Software\Microsoft\Terminal Server Client` skrivaromdirigering helt på klienten; på hanterade enheter är det värt att kontrollera detta innan felsökningen börjar i sessionen.

## Serversidan: tillåt omdirigering

På målsystemet avgör principen **Tillåt inte omdirigering av klientskrivare** (Datorkonfiguration → Administrativa mallar → Windows-komponenter → Fjärrskrivbordstjänster → Fjärrskrivbordssessionsvärd → Skrivaromdirigering). Om den är satt till *Aktiverad* skapas inga klientskrivare, oavsett vad klienten begär. Principen om den mest restriktiva inställningen gäller: Om någon av de två sidorna blockerar omdirigeringen sker den inte.

Utan grupprincip styrs samma mekanism via registret: `fDisableCpm` under `HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp` (0 = omdirigering tillåten, 1 = blockerad). Dessutom måste tjänsten **Utskriftshanterare** köras på målsystemet; utan spoolern skapas inte heller omdirigerade skrivare.

I samma GPO-kategori finns två användbara grannar: **Använd först skrivardrivrutinen Remote Desktop Easy Print** (standard och oftast rätt val) och **Ange klientens standardskrivare som standardskrivare för sessionen**.

## Varningen «Okänd utgivare»

När en osignerad .rdp-fil som begär enhetsomdirigeringar öppnas visar klienten en säkerhetsvarning med kryssrutor för de enskilda resurserna. Markeringar som sätts eller tas bort där gäller bara för just den anslutningsstarten, men sparas i den tidigare nämnda nyckeln `LocalDevices` och påverkar därmed framtida anslutningar i det tysta. Den som undrar varför kryssrutan för skrivare saknas gång på gång trots en korrekt .rdp-fil hittar nästan alltid orsaken där.

Det finns tre sätt att hantera varningen, i stigande arbetsinsats. För det första: starta anslutningen med `mstsc /v:hostname` i stället för via .rdp-filen; utan fil finns ingen utgivarkontroll, och inställningarna kommer från `Default.rdp`. För det andra: godkänn omdirigeringarna för måldatorn i förväg via registret, så försvinner resursdelen av dialogrutan:

```powershell
$key = "HKCU:\Software\Microsoft\Terminal Server Client\LocalDevices"
New-Item -Path $key -Force | Out-Null
Set-ItemProperty -Path $key -Name "hostname-oder-ip" -Value 0xFFFFFFFF -Type DWord
```

För det tredje, det rena sättet för distribuerade .rdp-filer i företaget: signera filen med `rdpsign.exe` och ett certifikat och lagra certifikatets tumavtryck som en betrodd utgivare via GPO. För enskilda arbetsstationer lönar sig sällan arbetet, men för centralt distribuerade anslutningsfiler är det rätt lösning.

## Checklista för felsökning

Om skrivaren inte visas i sessionen, kontrollera följande i denna ordning:

1. **Anslutit igen?** Omdirigeringen tillämpas bara när anslutningen upprättas, inte i en befintlig session.
2. **Rätt fil?** För genvägar, kontrollera vilken .rdp-fil som faktiskt öppnas; vid `mstsc /v:` räknas `Default.rdp`.
3. **Sparat val?** Kontrollera eller ta bort nyckeln `LocalDevices` på klienten.
4. **Klientblockering?** `DisablePrinterRedirection` under `HKLM\Software\Microsoft\Terminal Server Client` får inte vara satt till 1.
5. **Serverblockering?** Kontrollera GPO:n «Tillåt inte omdirigering av klientskrivare» respektive `fDisableCpm` på målsystemet.
6. **Spooler?** Tjänsten Utskriftshanterare måste köras på målsystemet.
7. **Kontroll i sessionen:** `Get-Printer | Where-Object DriverName -eq "Remote Desktop Easy Print"` listar de omdirigerade skrivarna med sessions-ID.

## Källor

1.  [Supported RDP properties](https://learn.microsoft.com/en-us/azure/virtual-desktop/rdp-properties): Referens över alla .rdp-egenskaper, däribland redirectprinters med värden och standardvärde.

2.  [Configure printer redirection over the Remote Desktop Protocol](https://learn.microsoft.com/en-us/azure/virtual-desktop/redirection-configure-printers): Easy Print, GPO- och Intune-konfiguration, DisablePrinterRedirection och testet med Get-Printer.

3.  [rdpsign](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/rdpsign): Kommandoreferens för signering av .rdp-filer med certifikatets tumavtryck.
