---
title: "RDP-skriveromdirigering: Skriv ut via den lokale PC-en i stedet for via den eksterne datamaskinen"
navTitle: "RDP-skriveromdirigering"
description: "Utskriftsjobber fra RDP-økten skal havne på skriveren ved siden av brukeren, ikke på den eksterne datamaskinen. Innstillingen finnes på tre steder: i RDP-klienten, i .rdp-filen og på målsystemet. I tillegg behandles advarselen «Ukjent utgiver» og en sjekkliste for feilsøking."
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
translationSourceHash: 1ca6c78df61e05f90d1aefc5b20b611e5c6a4bf03908d0466da114c2823e31cd
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:34:03.602Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/no/blog/rdp-skriveromdirigering-skriv-ut-via-den-lokale-pc-en-i-stedet-for-den-eksterne-datamaskinen
---

# RDP-skriveromdirigering: Skriv ut via den lokale PC-en i stedet for via den eksterne datamaskinen

En bruker arbeider via Remote Desktop på en ekstern datamaskin og vil skrive ut på skriveren som står ved siden av ham. Det er nettopp dette skriveromdirigering er til for: RDP-klienten registrerer de lokale skriverne i økten, utskriftsjobben går tilbake til klienten via RDP-kanalen og skrives ut der. På målsystemet vises skriveren med tillegget **(omdirigert, økt n)**. Det trengs som regel ingen drivere på den eksterne datamaskinen: Windows bruker den universelle driveren **Remote Desktop Easy Print**; riktig skriverdriver må bare være installert på den lokale klienten.

Omdirigeringen trer bare i kraft når tilkoblingen opprettes. Etter hver endring i innstillingene må økten kobles helt fra og kobles til på nytt; det er ikke nok å bare minimere RDP-vinduet.

## Klientsiden: aktiver omdirigeringen

Skriveromdirigering aktiveres enklest via det grafiske grensesnittet: start `mstsc`, velg **Vis alternativer**, fanen **Lokale ressurser**, kryss av for **Skrivere** og lagre tilkoblingen på fanen **Generelt**. De som arbeider med .rdp-filer, kan tilpasse linjen direkte i filen; .rdp-filer er enkle tekstfiler og kan redigeres med en hvilken som helst teksteditor:

```text
redirectprinters:i:1
```

En særhet ved snarveier uten .rdp-fil: Hvis tilkoblingen startes med `mstsc /v:hostname`, gjelder innstillingene fra den skjulte filen `Default.rdp` i brukerens Dokumenter-mappe. Hvis linjen `redirectprinters:i:1` mangler der, vises ikke skriveren, selv om alt tilsynelatende er riktig konfigurert. Dette snutten legger til linjen idempotent (eksisterende `0` blir til `1`, manglende linje legges til) og viser resultatet for kontroll:

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

To andre feilkilder på klientsiden: For det første lagrer Windows per måldatamaskin under `HKCU\Software\Microsoft\Terminal Server Client\LocalDevices` hvilke omdirigeringer brukeren sist tillot i sikkerhetsdialogen; dette lagrede valget overstyrer innstillingen i .rdp-filen. Sletting av nøkkelen tilbakestiller tilstanden. For det andre deaktiverer registerverdien `DisablePrinterRedirection` (DWORD, verdi 1) under `HKLM\Software\Microsoft\Terminal Server Client` skriveromdirigering fullstendig på klienten; på administrerte enheter bør dette kontrolleres før feilsøkingen i økten begynner.

## Serversiden: tillat omdirigering

På målsystemet avgjør policyen **Ikke tillat omdirigering av klientskrivere** (Datamaskinkonfigurasjon → Administrative maler → Windows-komponenter → Remote Desktop Services → Remote Desktop Session Host → Skriveromdirigering). Hvis den er satt til *Aktivert*, opprettes ingen klientskrivere, uavhengig av hva klienten ber om. Prinsippet er at den mest restriktive innstillingen gjelder: Hvis en av sidene blokkerer omdirigeringen, skjer den ikke.

Uten gruppepolicy styres den samme mekanismen via registeret: `fDisableCpm` under `HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp` (0 = omdirigering tillatt, 1 = blokkert). I tillegg må tjenesten **Utskriftskø** kjøre på målsystemet; uten spooleren opprettes heller ikke omdirigerte skrivere.

I samme GPO-kategori finnes to andre nyttige innstillinger: **Bruk først skriverdriveren Remote Desktop Easy Print** (standard og vanligvis det riktige valget) og **Angi klientens standardskriver som standardskriver for økten**.

## Advarselen «Ukjent utgiver»

Når en usignert .rdp-fil som ber om enhetsomdirigeringer åpnes, viser klienten en sikkerhetsadvarsel med avkrysningsbokser for de enkelte ressursene. Avkrysninger som settes eller fjernes der, gjelder bare for denne ene tilkoblingsstarten, men lagres i den ovennevnte nøkkelen `LocalDevices` og påvirker dermed fremtidige tilkoblinger i det stille. De som lurer på hvorfor avkrysningen for skriveren stadig mangler til tross for en korrekt .rdp-fil, finner nesten alltid årsaken der.

Siden sikkerhetsoppdateringen fra april 2026 (CVE-2026-26151) viser Remote Desktop Connection en ny sikkerhetsdialog hver gang en .rdp-fil åpnes, før tilkoblingen opprettes. Alle omdirigeringer som filen ber om, inkludert skrivere, er deaktivert som standard og må aktiveres enkeltvis. Usignerte filer får i tillegg meldingen «Caution: Unknown remote connection». Ifølge Microsoft er tilkoblinger som startes direkte med et datamaskinnavn i Remote Desktop Connection, ikke berørt. Som en midlertidig løsning dokumenterer Microsoft registerverdien `RedirectionWarningDialogVersion` (DWORD, verdi 1) under `HKLM\Software\Policies\Microsoft\Windows NT\Terminal Services\Client`, som gjenoppretter den tidligere dialogen; støtten for dette kan forsvinne med en fremtidig oppdatering.

Det finnes tre måter å håndtere advarselen på, i stigende grad av innsats. For det første: start tilkoblingen med `mstsc /v:hostname` i stedet for via .rdp-filen; uten fil utføres ingen utgiverkontroll, og innstillingene hentes fra `Default.rdp`. Den nye sikkerhetsdialogen som ble innført i april 2026, vises heller ikke da. For det andre: godkjenn omdirigeringene for måldatamaskinen på forhånd via registeret, så bortfaller ressursdelen i den tidligere dialogen. Microsoft dokumenterer ikke om denne forhåndsgodkjenningen også gjelder for den nye sikkerhetsdialogen siden april 2026:

```powershell
$key = "HKCU:\Software\Microsoft\Terminal Server Client\LocalDevices"
New-Item -Path $key -Force | Out-Null
Set-ItemProperty -Path $key -Name "hostname-oder-ip" -Value 0xFFFFFFFF -Type DWord
```

For det tredje, den ryddige løsningen for distribuerte .rdp-filer i virksomheten: signer filen med `rdpsign.exe` og et sertifikat, og lagre sertifikatets fingeravtrykk som en klarert utgiver via GPO. For enkeltstående arbeidsstasjoner er innsatsen sjelden verdt det, men for sentralt distribuerte tilkoblingsfiler er det riktig løsning.

## Sjekkliste for feilsøking

Hvis skriveren ikke vises i økten, kontroller følgende i denne rekkefølgen:

1. **Koblet til på nytt?** Omdirigeringen trer bare i kraft når tilkoblingen opprettes, ikke i en eksisterende økt.
2. **Riktig fil?** Ved snarveier må du kontrollere hvilken .rdp-fil som faktisk åpnes; ved `mstsc /v:` er det `Default.rdp` som gjelder.
3. **Lagret valg?** Kontroller eller slett nøkkelen `LocalDevices` på klienten. Ved oppstart via en .rdp-fil må du i tillegg kontrollere om skriveren ble aktivert i sikkerhetsdialogen; siden april 2026 er den deaktivert der som standard.
4. **Klientblokkering?** `DisablePrinterRedirection` under `HKLM\Software\Microsoft\Terminal Server Client` må ikke være satt til 1.
5. **Serverblokkering?** Kontroller GPO-en «Ikke tillat omdirigering av klientskrivere» eller `fDisableCpm` på målsystemet.
6. **Spooler?** Tjenesten Utskriftskø må kjøre på målsystemet.
7. **Kontroll i økten:** `Get-Printer | Where-Object DriverName -eq "Remote Desktop Easy Print"` viser de omdirigerte skriverne med økt-ID.

## Endringslogg

- **25. oktober 2026:** Lagt til: Siden sikkerhetsoppdateringen fra april 2026 (CVE-2026-26151) viser en ny dialog ved åpning av .rdp-filer alle omdirigeringer, også skrivere, deaktivert som standard; i tillegg registerverdien RedirectionWarningDialogVersion og et ekstra kontrolltrinn.

## Kilder

1.  [Supported RDP properties](https://learn.microsoft.com/en-us/azure/virtual-desktop/rdp-properties): Referanse for alle .rdp-egenskaper, inkludert redirectprinters med verdier og standardverdi.

2.  [Configure printer redirection over the Remote Desktop Protocol](https://learn.microsoft.com/en-us/azure/virtual-desktop/redirection-configure-printers): Easy Print, GPO- og Intune-konfigurasjon, DisablePrinterRedirection og testen med Get-Printer.

3.  [rdpsign](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/rdpsign): Kommandoreferanse for signering av .rdp-filer med sertifikatets fingeravtrykk.

4.  [Understanding security warnings when opening Remote Desktop (RDP) files](https://learn.microsoft.com/en-us/windows-server/remote/remote-desktop-services/remotepc/understanding-security-warnings): Ny sikkerhetsdialog siden oppdateringen fra april 2026, omdirigeringer deaktivert som standard og registerverdien RedirectionWarningDialogVersion.
