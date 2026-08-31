---
title: "RDP-printeromdirigering: Skriv ut via den lokale PC-en i stedet for den eksterne maskinen"
navTitle: "RDP-printeromdirigering"
description: "Utskriftsjobber fra RDP-økten skal havne på skriveren ved siden av brukeren, ikke på den eksterne maskinen. Innstillingen finnes på tre steder: i RDP-klienten, i .rdp-filen og på målsystemet. I tillegg beskrives håndtering av advarselen «Ukjent utgiver» og en sjekkliste for feilsøking."
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
translationSourceHash: 2cb3845d308ebda202c6c33b20cbe791ddfbeeb584341876bdc340e0febf65b5
translationModel: gpt-5.6-terra
translatedAt: 2026-08-30T09:30:13.729Z
translationReview: automatic
url: https://rafaelpfister.ch/no/blog/rdp-skriveromdirigering-skriv-ut-via-den-lokale-pc-en-i-stedet-for-den-eksterne-datamaskinen
---

# RDP-printeromdirigering: Skriv ut via den lokale PC-en i stedet for den eksterne maskinen

En bruker arbeider via Remote Desktop på en ekstern maskin og vil skrive ut på skriveren som står ved siden av vedkommende. Det er nettopp dette printeromdirigering er til for: RDP-klienten registrerer de lokale skriverne i økten, utskriftsjobben går tilbake til klienten via RDP-kanalen og skrives ut der. På målsystemet vises skriveren med tillegget **(omdirigert, økt n)**. Det er vanligvis ikke nødvendig med drivere på den eksterne maskinen: Windows bruker universaldriveren **Remote Desktop Easy Print**; riktig skriverdriver trenger bare å være installert på den lokale klienten.

Om dirigeringen fungerer, avgjøres bare når forbindelsen opprettes. Etter hver endring av innstillingene må økten kobles helt fra og kobles til på nytt; det er ikke nok å bare minimere RDP-vinduet.

## Klientsiden: aktiver omdirigering

Den enkleste måten å aktivere printeromdirigering på er via det grafiske brukergrensesnittet: Start `mstsc`, velg **Vis alternativer**, fanen **Lokale ressurser**, kryss av for **Skrivere**, og lagre forbindelsen under fanen **Generelt**. De som arbeider med .rdp-filer, kan tilpasse linjen direkte i filen; .rdp-filer er enkle tekstfiler og kan redigeres med en hvilken som helst tekstredigerer:

```text
redirectprinters:i:1
```

En særegenhet ved snarveier uten .rdp-fil: Hvis forbindelsen startes med `mstsc /v:hostname`, gjelder innstillingene fra den skjulte filen `Default.rdp` i brukerens Dokumenter-mappe. Hvis linjen `redirectprinters:i:1` mangler der, vises ikke skriveren, selv om alt tilsynelatende er riktig konfigurert. Dette snuttet legger til linjen idempotent (eksisterende `0` endres til `1`, manglende linje legges til) og viser resultatet for kontroll:

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

To andre feilkilder på klientsiden: For det første lagrer Windows per målmaskin under `HKCU\Software\Microsoft\Terminal Server Client\LocalDevices` hvilke omdirigeringer brukeren sist tillot i sikkerhetsdialogen; dette lagrede valget overstyrer innstillingen i .rdp-filen. Hvis nøkkelen slettes, tilbakestilles tilstanden. For det andre deaktiverer registerverdien `DisablePrinterRedirection` (DWORD, verdi 1) under `HKLM\Software\Microsoft\Terminal Server Client` printeromdirigering fullstendig på klienten; på administrerte enheter bør dette kontrolleres før feilsøkingen begynner i økten.

## Serversiden: tillat omdirigering

På målsystemet er det policyen **Ikke tillat omdirigering av klientskrivere** (Datamaskinkonfigurasjon → Administrative maler → Windows-komponenter → Remote Desktop Services → Remote Desktop Session Host → Printer Redirection) som avgjør. Hvis den er satt til *Aktivert*, opprettes ingen klients skrivere, uansett hva klienten ber om. Prinsippet om den mest restriktive innstillingen gjelder: Hvis én av sidene blokkerer omdirigering, finner den ikke sted.

Uten gruppepolicy styres samme mekanisme via registeret: `fDisableCpm` under `HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp` (0 = omdirigering tillatt, 1 = blokkert). I tillegg må tjenesten **Utskriftskø** kjøre på målsystemet; uten spooleren opprettes heller ikke omdirigerte skrivere.

I samme GPO-kategori finnes to andre nyttige innstillinger: **Bruk først skriverdriveren for Remote Desktop Easy Print** (standard og som regel riktig valg) og **Gjør klientens standardskriver til standardskriver i økten**.

## Advarselen «Ukjent utgiver»

Når en usignert .rdp-fil som ber om enhetsomdirigeringer åpnes, viser klienten en sikkerhetsadvarsel med avkrysningsbokser for de enkelte ressursene. Avkryssinger som settes eller fjernes der, gjelder bare for denne ene tilkoblingen, men lagres i den ovennevnte nøkkelen `LocalDevices` og påvirker dermed stille fremtidige forbindelser. Den som lurer på hvorfor avkryssingen for skriver alltid mangler til tross for korrekt .rdp-fil, finner nesten alltid årsaken der.

Det finnes tre måter å håndtere advarselen på, med økende innsats. For det første: Start forbindelsen med `mstsc /v:hostname` i stedet for via .rdp-filen; uten fil utføres ingen utgiverkontroll, og innstillingene hentes fra `Default.rdp`. For det andre: Godkjenn omdirigeringene for målmaskinen på forhånd via registeret, så bortfaller ressursdelen av dialogen:

```powershell
$key = "HKCU:\Software\Microsoft\Terminal Server Client\LocalDevices"
New-Item -Path $key -Force | Out-Null
Set-ItemProperty -Path $key -Name "hostname-oder-ip" -Value 0xFFFFFFFF -Type DWord
```

For det tredje, den ryddige løsningen for distribuerte .rdp-filer i bedriften: Signer filen med `rdpsign.exe` og et sertifikat, og lagre sertifikatets fingeravtrykk som en klarert utgiver via GPO. For enkeltarbeidsplasser er innsatsen sjelden verdt det, men for sentralt distribuerte forbindelsesfiler er dette riktig løsning.

## Sjekkliste for feilsøking

Hvis skriveren ikke vises i økten, kontroller følgende i denne rekkefølgen:

1. **Koblet til på nytt?** Omdirigering fungerer bare når forbindelsen opprettes, ikke i en eksisterende økt.
2. **Riktig fil?** Ved snarveier må du kontrollere hvilken .rdp-fil som faktisk åpnes; ved `mstsc /v:` er det `Default.rdp` som gjelder.
3. **Lagret valg?** Kontroller eller slett nøkkelen `LocalDevices` på klienten.
4. **Klientblokkering?** `DisablePrinterRedirection` under `HKLM\Software\Microsoft\Terminal Server Client` må ikke være satt til 1.
5. **Serverblokkering?** Kontroller GPO-en «Ikke tillat omdirigering av klientskrivere» eller `fDisableCpm` på målsystemet.
6. **Spooler?** Tjenesten Utskriftskø må kjøre på målsystemet.
7. **Kontroll i økten:** `Get-Printer | Where-Object DriverName -eq "Remote Desktop Easy Print"` viser de omdirigerte skriverne med økt-ID.

## Kilder

1.  [Supported RDP properties](https://learn.microsoft.com/en-us/azure/virtual-desktop/rdp-properties): Referanse for alle .rdp-egenskaper, inkludert redirectprinters med verdier og standardverdi.

2.  [Configure printer redirection over the Remote Desktop Protocol](https://learn.microsoft.com/en-us/azure/virtual-desktop/redirection-configure-printers): Easy Print, GPO- og Intune-konfigurasjon, DisablePrinterRedirection og testen med Get-Printer.

3.  [rdpsign](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/rdpsign): Kommandoreferanse for signering av .rdp-filer med sertifikatets fingeravtrykk.
