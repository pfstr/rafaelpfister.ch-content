---
title: "Microsoft Entra Connect Sync 2.6.91.0 and 2.6.92.0: Security fixes, Cloud Sync migration and mandatory upgrade by April 2027"
navTitle: "Microsoft Entra Connect Sync 2.6.92.0"
description: "Version 2.6.91.0 brings security fixes, a guided switch to Cloud Sync and fixes the Known Issues from 2.6.84.0; hotfix 2.6.92.0 fixes Pass-through Authentication setup. By 7 April 2027, all servers must run at least 2.6.84.0 with Application-Based Authentication."
date: "2026-10-26"
kategorie: "Microsoft Entra"
timeToRead: "8 min. lesetid"
themen:
  - microsoft-entra
  - active-directory-entra
produkte:
  - "entra"
  - "entra-connect-releases"
protokolle:
  - "releases"
slug: "microsoft-entra-connect-sync-2-6-91-0-and-2-6-92-0-security-fixes-cloud-sync-migration-and"
translationId: article-59a51ef69f2e3538
translationOf: entra-connect-2-6-92-0
url: https://rafaelpfister.ch/no/blog/microsoft-entra-connect-sync-2-6-91-0-and-2-6-92-0-security-fixes-cloud-sync-migration-and
translationSourceHash: ff045b1bab72cea3839167d59c0a71010e80a3a20b1644acef89c79921f6f219
translationModel: gpt-5.6-terra
translatedAt: 2026-10-26T08:26:20.675Z
translationReview: automatic
---

# Microsoft Entra Connect Sync 2.6.91.0 og 2.6.92.0: Sikkerhetsrettinger, Cloud Sync-migrering og obligatorisk oppgradering innen april 2027

Microsoft lanserte Entra Connect Sync 2.6.91.0 16. september 2026 og hurtigreparasjonen 2.6.92.0 23. september 2026, begge kun som nedlasting via Microsoft Entra Admin Center. Versjon 2.6.91.0 retter sikkerhetssårbarheter i medfølgende tredjepartskomponenter, gjør phishing-resistent pålogging i installasjonsveiviseren allment tilgjengelig, gir en veiledet migreringsprosess til Microsoft Entra Cloud Sync og retter de tre Known Issues fra 2.6.84.0. Versjon 2.6.92.0 retter en feil i 2.6.91.0 ved oppsett av Pass-through Authentication. Samtidig oppgir Microsoft en fast frist: Innen 7. april 2027 må alle servere kjøre minst 2.6.84.0 og bruke Application-Based Authentication, ellers stopper synkroniseringen. Artikkelen er rettet mot administratorer som drifter Entra Connect Sync. Forrige status er beskrevet i artikkelen om [Entra Connect Sync 2.6.84.0](/blog/entra-connect-2-6-84-0).

## Versjonene i oversikt

| Versjon | Dato | Type | Hovedpunkter |
|---|---|---|---|
| 2.6.91.0 | 16.09.2026 | Funksjons- og sikkerhetsutgivelse (nedlasting) | Sikkerhetsrettinger i tredjepartskomponenter, Cloud Sync-migreringsassistent, Passkey-pålogging i veiviseren GA, nye Microsoft Graph-tillatelser, rettinger for Known Issues i 2.6.84.0 |
| 2.6.92.0 | 23.09.2026 | Hurtigreparasjon med sikkerhetsrettinger (nedlasting) | Retter mislykket registrering av Authentication Agent ved aktivering av Pass-through Authentication |

Microsoft klassifiserer 2.6.92.0 i Release Notes som en utgivelse med sikkerhetsrettinger og anbefaler å oppdatere «så snart som mulig». Det er ikke varslet noen automatisk oppgraderingsutrulling for noen av de to versjonene (per 26. oktober 2026). Også 2.6.84.0 er fortsatt bare tilgjengelig som nedlasting.

## Obligatorisk oppgradering innen 7. april 2027

Versjonshistorikken og oppgraderingsveiledningen har nå en identisk merknad: Entra Connect Sync må innen 7. april 2027 være oppdatert til versjon 2.6.84.0 eller nyere og være byttet til Application-Based Authentication (ABA). Microsoft avvikler eldre autentisering via Directory Synchronization Account; servere som ikke oppfyller disse kravene, vil ikke lenger synkronisere etter denne datoen. Hvis synkroniseringen stopper, gjenoppretter ifølge Microsoft en oppgradering til gjeldende versjon med påfølgende ABA-konfigurasjon driften.

For planleggingen er en atferdsendring fra 2.6.84.0 avgjørende: Fra og med denne versjonen bytter Entra Connect ikke lenger eksisterende servere til ABA automatisk. Bare nye installasjoner konfigurerer ABA automatisk. En oppgradering alene oppfyller derfor ikke fristen. Eksisterende servere må byttes i veiviseren via **Additional tasks** > **Configure application-based authentication to Microsoft Entra ID**.

Hvilken metode en server bruker for øyeblikket, vises av veiviseren under **View or export current configuration**. Microsoft oppgir i tillegg denne cmdleten:

```powershell
Get-ADSyncEntraConnectorCredential
```

Returverdien `ConnectorIdentityType` er `ServiceAccount` (eldre konto) eller `Application` (ABA). Hvis den viser `ServiceAccount`, må overgangen gjennomføres før 7. april 2027. Etter en vellykket overgang anbefaler Microsoft å fjerne den gamle Directory Synchronization Account fra Entra ID.

## Versjon 2.6.91.0

### Sikkerhetsrettinger

Release Notes oppgir rettede sikkerhetssårbarheter i medfølgende tredjepartskomponenter, styrkede tillatelser på lokale midlertidige filer og sikrede nedlastinger av profilbilder i SharePoint-koblingen. Microsoft oppgir ikke CVE-numre, CVSS-verdier eller berørte komponenter. Det er derfor ikke mulig å vurdere alvorlighetsgraden basert på offentlige kilder.

### Nye Microsoft Graph-tillatelser og Conditional Access

Entra Connect får ytterligere Microsoft Graph-tillatelser. Microsoft påpeker uttrykkelig: De som bruker app-baserte Conditional Access-policyer, bør kontrollere policyer som er rettet mot `Microsoft.Azure.SyncFabric` eller Microsoft 365 Reporting Service. Kontroller disse policyene før oppgraderingen, slik at den nye tilgangen ikke blokkeres.

### Veiledet migrering til Cloud Sync

Nytt er en veiledet prosess for overgang fra Entra Connect Sync til Microsoft Entra Cloud Sync. Den omfatter en vurdering av eksisterende konfigurasjon, oppsett av Provisioning Agent, trinnvis aktivering og validering. Funksjonen er bare tilgjengelig i Azure Public Cloud. Microsofts oppgraderingsveiledning anbefaler uansett å vurdere før en oppgradering om Cloud Sync er aktuelt for eget miljø.

### Phishing-resistent pålogging i veiviseren er allment tilgjengelig

Påloggingen via Windows Web Account Manager, som ble innført som Preview i 2.6.84.0, er nå generally available og aktiv som standard. Dialogen støtter passkeys, FIDO2-sikkerhetsnøkler og passord, gjenbruker en eksisterende påloggingsøkt for flere Entra-tjenester og opprettholder Kerberos-nøkkelrotasjon for Seamless Single Sign-On.

### Endrede PowerShell-cmdleter

2.6.84.0 hadde gjort parameteren `-AADUsername` obligatorisk for `Set-ADSyncAADCompanyFeature` og `Set-ADSyncAADPasswordSyncState`. Med 2.6.91.0 er parameteren valgfri igjen: Hvis den mangler, utleder Entra Connect et påloggingshint fra koblingskonfigurasjonen og åpner en interaktiv pålogging. Dette gjelder for `Set-ADSyncAADCompanyFeature`, `Set-ADSyncAADPasswordSyncState`, `Enable-ADSyncExportDeletionThreshold`, `Set-ADSyncScheduler` og `Set-ADSyncDirSyncConfiguration`. Interaktiv pålogging er dermed fortsatt en del av disse cmdletene; skript som kjører dem uten tilsyn, bør testes etter oppgraderingen.

De som konfigurerer Seamless Single Sign-On med det frittstående PowerShell-modulet, må importere `ADSync.psd1` før `AzureADSSO.psd1`.

### Andre endringer

- Dialogen **Select Containers** i Synchronization Service Manager er nå skrivebeskyttet. Endringer i utvalget av containere gjøres via **Customize synchronization options** i veiviseren.
- Støtte for enda et Sovereign Cloud-miljø, inkludert Pass-through Authentication, Seamless Single Sign-On, Password Writeback og Health Agent-overvåking. Release Notes oppgir ikke hvilket miljø det er snakk om.
- Den medfølgende SQL Server 2022 LocalDB er oppdatert fra 16.0.4250.1 til 16.0.4262.2.

### Feilrettinger

- Installasjon eller oppgradering med en eksisterende ADSync-database kunne mislykkes med feil `0xE0474352` (Known Issue fra 2.6.84.0).
- En oppgradering oppdaterte ikke en installert Visual C++ 2015–2022 Redistributable eldre enn 14.41.
- Veiviseren kunne lukkes ved oppstart hvis policyen for Windows PowerShell Transcription inneholdt en ugyldig utdatamappe.
- Sertifikatrotasjonen for Application-Based Authentication kunne, ved replikeringsforsinkelser i katalogen, fjerne den siste brukbare Application Key.
- Synchronization Service Manager kunne lukkes ved opplisting av katalogpartisjonene i **Connector Properties** (Known Issue fra 2.6.84.0).
- **Change Credentials** åpnet seg ikke for koblingsrader utenfor det synlige området når mange Active Directory-koblinger er konfigurert.
- Når et tidligere fravalgt domene ble utvidet på siden **Domain and OU filtering**, kunne veiviseren velge domenet på nytt og aktivere synkronisering for hele domenet etter fullføring (Known Issue fra 2.6.84.0).

Det siste punktet fortjener oppmerksomhet: De som kjører 2.6.84.0 og har gått gjennom veiviseren på nytt etter oppgraderingen, bør kontrollere domene- og OU-filtreringen. Et utilsiktet aktivert domene fører til flere objekter i Entra ID.

## Versjon 2.6.92.0

Hurtigreparasjonen retter en feil i 2.6.91.0: Aktivering av Pass-through Authentication via veiviseren kunne mislykkes ved registrering av den lokalt installerte Microsoft Entra Connect Authentication Agent. Release Notes nevner ingen andre endringer. Microsoft markerer 2.6.92.0 som en utgivelse med sikkerhetsrettinger; enkelte sårbarheter er ikke oppført.

## Kjente problemer

| Problem | Berørt versjon | Status |
|---|---|---|
| Installasjon eller oppgradering med eksisterende ADSync-database mislykkes med `0xE0474352` | 2.6.84.0 | Rettet i 2.6.91.0 |
| Synchronization Service Manager lukkes i **Connector Properties** | 2.6.84.0 | Rettet i 2.6.91.0 |
| Veiviseren velger et fravalgt domene på nytt | 2.6.84.0 | Rettet i 2.6.91.0 |
| PTA-aktivering i veiviseren mislykkes ved agentregistrering | 2.6.91.0 | Rettet i 2.6.92.0 |
| Synkronisering mislykkes etter oppgradering (`System.IO.FileLoadException`), hvis `miiserver.exe.config` tidligere ble endret | 2.5.190.0, 2.6.1.0 | Fortsatt oppført i versjonshistorikken; løsning via manuell bindingRedirect, se [artikkel om 2.6.84.0](/blog/entra-connect-2-6-84-0) |

Den tilbaketrukne versjonen 2.6.79.0 er fortsatt ikke tilgjengelig. Microsoft henviser nå berørte brukere til 2.6.92.0: avinstaller og installer den gjeldende versjonen.

## Supportfrister

Hver 2.x-versjon faller ut av støtte tolv måneder etter at etterfølgerversjonen ble lansert. Med de to nye versjonene blir fristene som følger:

| Versjon | Lansert | Slutt på støtte |
|---|---|---|
| 2.5.79.0 | 1. september 2025 | 23. oktober 2026 (utløpt) |
| 2.5.190.0 | 19. november 2025 | 2. februar 2027 |
| 2.6.1.0 | 2. februar 2026 | 10. mars 2027 |
| 2.6.3.0 | 10. mars 2026 | 7. juli 2027 |
| 2.6.84.0 | 7. juli 2026 | 16. september 2027 |
| 2.6.91.0 | 16. september 2026 | 23. september 2027 |
| 2.6.92.0 | 23. september 2026 | fortsatt åpen |

For versjonene 2.5.190.0 til 2.6.3.0 gjelder i tillegg fristen 7. april 2027: De ligger under den påkrevde minimumsversjonen 2.6.84.0 og vil ikke lenger synkronisere etter denne datoen, selv om ordinær støtte for 2.6.3.0 først opphører i juli 2027. Versjon 2.5.79.0 har vært uten støtte siden 23. oktober 2026; ifølge Microsoft kan versjoner som ikke lenger støttes, slutte å fungere uventet.

## Anbefalt fremgangsmåte

Målversjonen er 2.6.92.0. Den inneholder sikkerhetsrettingene fra begge utgivelsene, retter alle kjente problemer i 2.6.84.0 og 2.6.91.0 og oppfyller minimumsversjonen for fristen 7. april 2027. Installasjonsprogrammet finnes kun i Microsoft Entra Admin Center under **Microsoft Entra Connect | Get started**, fanen **Manage**; forutsetningene er .NET Framework 4.7.2 og TLS 1.2. Før oppgraderingen bør du kontrollere app-baserte Conditional Access-policyer for `Microsoft.Azure.SyncFabric` og Microsoft 365 Reporting Service, samt skript som kaller de nevnte cmdletene for skykonfigurasjon. Microsoft anbefaler Swing Migration for oppgraderinger med lav risiko: Oppdater først staging-serveren, verifiser konfigurasjonen etter Full Import og Full Synchronization, bytt deretter roller og oppdater den tidligere aktive serveren. De som ikke har oppdatert på 12 til 18 måneder, bør ifølge Microsoft velge denne metoden. Etter oppgraderingen må eksisterende servere som fortsatt rapporterer `ServiceAccount`, byttes til Application-Based Authentication, og på servere som tidligere kjørte 2.6.84.0, må domene- og OU-filtreringen kontrolleres.

## Kilder

1.  [Microsoft Entra Connect: Version release history](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-version-history): Release Notes for 2.6.91.0 og 2.6.92.0, Known Issues fra 2.6.84.0, avviklingstabell og fristen 7. april 2027.

2.  [Microsoft Entra Connect: Upgrade from a previous version](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-upgrade-previous-version): Oppgraderingsmetoder, Swing Migration via en staging-server og merknad om obligatorisk oppgradering.

3.  [Authenticate to Microsoft Entra ID by using application identity](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/authenticate-application-id): Manuell overgang av eksisterende servere til ABA, `Get-ADSyncEntraConnectorCredential` og fjerning av eldre konto.
