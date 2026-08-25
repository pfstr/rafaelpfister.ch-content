---
title: "RDP-skriveromdirigering: Skriv ut via den lokale PC-en i stedet for den eksterne datamaskinen"
navTitle: "RDP-skriveromdirigering"
description: "Utskriftsjobber fra RDP-økten skal havne på skriveren ved siden av brukeren, ikke på den eksterne datamaskinen. Innstillingen finnes på tre steder: i RDP-klienten, i .rdp-filen og på målsystemet. I tillegg: hvordan håndtere advarselen «Ukjent utgiver» og en sjekkliste for feilsøking."
date: "2026-08-24"
kategorie: "Windows-klient"
timeToRead: "5 min lesetid"
themen:
  - windows-client
produkte:
  - "windows-client"
protokolle:
  - "troubleshooting"
slug: "rdp-skriveromdirigering-skriv-ut-via-den-lokale-pc-en-i-stedet-for-den-eksterne-datamaskinen"
translationId: "article-12521248666e9809"
draft: false
translationOf: rdp-druckerumleitung-lokale-drucker
url: https://rafaelpfister.ch/no/blog/rdp-skriveromdirigering-skriv-ut-via-den-lokale-pc-en-i-stedet-for-den-eksterne-datamaskinen
translationSourceHash: a4f12f591e9dcb86f8ebdd3ff8af1008a130c3ec65424abe789ad4d6446eb4c2
translationModel: gpt-5.6-terra
translatedAt: 2026-08-25T04:15:17.111Z
translationReview: automatic
---

# RDP-skriveromdirigering: Skriv ut via den lokale PC-en i stedet for den eksterne datamaskinen

En bruker arbeider via Remote Desktop på en ekstern datamaskin og vil skrive ut på skriveren som står ved siden av ham. Det er nettopp dette skriveromdirigering er til for: RDP-klienten registrerer de lokale skriverne i økten, utskriftsjobben går tilbake til klienten via RDP-kanalen og skrives ut der. På målsystemet vises skriveren med tillegget **(omdirigert, økt n)**. Drivere på den eksterne datamaskinen trengs vanligvis ikke: Windows bruker den universelle driveren **Remote Desktop Easy Print**; den riktige skriverdriveren må bare være installert på den lokale klienten.

Omdirigeringen gjelder bare når forbindelsen opprettes. Etter hver endring av innstillingene må økten kobles helt fra og kobles til på nytt; det holder ikke å bare minimere RDP-vinduet.

## Klientsiden: aktiver omdirigering

Den raskeste veien går via det grafiske grensesnittet: start `mstsc`, velg **Vis alternativer**, fanen **Lokale ressurser**, huk av for **Skrivere** og lagre forbindelsen i fanen **Generelt**. De som i stedet arbeider med .rdp-filer, legger inn den aktuelle linjen direkte i filen; .rdp-filer er enkle tekstfiler og kan redigeres med et hvilket som helst redigeringsprogram:

```text
redirectprinters:i:1
```

En fallgruve ved snarveier uten .rdp-fil: Hvis forbindelsen startes med `mstsc /v:hostname`, gjelder innstillingene fra den skjulte filen `Default.rdp` i brukerens Dokumenter-mappe. Hvis linjen `redirectprinters:i:1` mangler der, vises ikke skriveren, selv om alt tilsynelatende er konfigurert riktig. Dette utdraget legger til linjen idempotent (eksisterende `0` blir til `1`, manglende linje legges til) og viser resultatet for kontroll:

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

To andre fallgruver på klientsiden: For det første husker Windows per måldatamaskin under `HKCU\Software\Microsoft\Terminal Server Client\LocalDevices` hvilke omdirigeringer brukeren sist tillot i sikkerhetsdialogen; dette lagrede valget overstyrer innstillingen i .rdp-filen. Sletting av nøkkelen tilbakestiller tilstanden. For det andre deaktiverer registerverdien `DisablePrinterRedirection` (DWORD, verdi 1) under `HKLM\Software\Microsoft\Terminal Server Client` skriveromdirigering fullstendig på klienten; på administrerte enheter bør du kontrollere dette før feilsøkingen begynner i økten.

## Serversiden: tillat omdirigering

På målsystemet er det policyen **Ikke tillat omdirigering av klientskrivere** (Datamaskinkonfigurasjon → Administrative maler → Windows-komponenter → Eksterne skrivebordstjenester → Vert for økt for eksternt skrivebord → Skriveromdirigering) som avgjør. Hvis den er satt til *Aktivert*, opprettes ingen klientskrivere, uansett hva klienten ber om. Det mest restriktive alternativet gjelder: Hvis én av sidene blokkerer omdirigeringen, finner den ikke sted.

Uten gruppepolicy styres den samme mekanismen via registret: `fDisableCpm` under `HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp` (0 = omdirigering tillatt, 1 = blokkert). I tillegg må tjenesten **Utskriftskø** kjøre på målsystemet; uten spooleren opprettes heller ikke omdirigerte skrivere.

I samme GPO-kategori finnes to nyttige naboinnstillinger: **Bruk først skriverdriveren Remote Desktop Easy Print** (standard og som regel riktig valg) og **Angi klientens standardskriver som øktens standardskriver**.

## Advarselen «Ukjent utgiver»

Når du åpner en usignert .rdp-fil som ber om enhetsomdirigeringer, viser klienten en sikkerhetsadvarsel med avkrysningsbokser for de enkelte ressursene. Avkrysninger som settes eller fjernes der, gjelder bare for denne ene oppstarten av forbindelsen, men lagres i den nevnte `LocalDevices`-nøkkelen og påvirker dermed fremtidige forbindelser i det stille. De som lurer på hvorfor avkrysningen for skriveren stadig mangler til tross for korrekt .rdp-fil, finner nesten alltid årsaken der.

Det finnes tre måter å håndtere advarselen på, i stigende rekkefølge av innsats. For det første: Start forbindelsen med `mstsc /v:hostname` i stedet for via .rdp-filen; uten fil blir det ingen utgiverkontroll, og innstillingene kommer fra `Default.rdp`. For det andre: Godkjenn omdirigeringene for måldatamaskinen på forhånd via registret, så faller ressursdelen av dialogen bort:

```powershell
$key = "HKCU:\Software\Microsoft\Terminal Server Client\LocalDevices"
New-Item -Path $key -Force | Out-Null
Set-ItemProperty -Path $key -Name "hostname-oder-ip" -Value 0xFFFFFFFF -Type DWord
```

For det tredje, den ryddige løsningen for distribuerte .rdp-filer i virksomheten: signer filen med `rdpsign.exe` og et sertifikat, og lagre sertifikatets fingeravtrykk som en klarert utgiver via GPO. For enkeltstående arbeidsstasjoner er innsatsen sjelden verdt det, men for sentralt distribuerte forbindelsesfiler er dette den riktige løsningen.

## Sjekkliste for feilsøking

Hvis skriveren ikke vises i økten, kontroller i denne rekkefølgen:

1. **Koblet til på nytt?** Omdirigeringen gjelder bare når forbindelsen opprettes, ikke i en eksisterende økt.
2. **Riktig fil?** Ved snarveier må du kontrollere hvilken .rdp-fil som faktisk åpnes; ved `mstsc /v:` er det `Default.rdp` som gjelder.
3. **Lagret valg?** Kontroller eller slett `LocalDevices`-nøkkelen på klienten.
4. **Klientblokkering?** `DisablePrinterRedirection` under `HKLM\Software\Microsoft\Terminal Server Client` må ikke være satt til 1.
5. **Serverblokkering?** Kontroller GPO-en «Ikke tillat omdirigering av klientskrivere» eller `fDisableCpm` på målsystemet.
6. **Spooler?** Tjenesten Utskriftskø på målsystemet må kjøre.
7. **Kontroll i økten:** `Get-Printer | Where-Object DriverName -eq "Remote Desktop Easy Print"` viser de omdirigerte skriverne med økt-ID.

## Kilder

1.  [Supported RDP properties](https://learn.microsoft.com/en-us/azure/virtual-desktop/rdp-properties): Referanse over alle .rdp-egenskaper, inkludert redirectprinters med verdier og standardverdi.

2.  [Configure printer redirection over the Remote Desktop Protocol](https://learn.microsoft.com/en-us/azure/virtual-desktop/redirection-configure-printers): Easy Print, GPO- og Intune-konfigurasjon, DisablePrinterRedirection og testen med Get-Printer.

3.  [rdpsign](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/rdpsign): Kommandoreferanse for signering av .rdp-filer via sertifikatets fingeravtrykk.
