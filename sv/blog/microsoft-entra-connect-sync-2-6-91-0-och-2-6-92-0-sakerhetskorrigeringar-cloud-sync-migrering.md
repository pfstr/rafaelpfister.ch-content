---
title: "Microsoft Entra Connect Sync 2.6.91.0 och 2.6.92.0: säkerhetskorrigeringar, Cloud Sync-migrering och obligatorisk uppgradering senast i april 2027"
navTitle: "Microsoft Entra Connect Sync 2.6.92.0"
description: "Version 2.6.91.0 innehåller säkerhetskorrigeringar, en guidad övergång till Cloud Sync och åtgärdar de kända problemen i 2.6.84.0. Hotfixen 2.6.92.0 korrigerar konfigurationen av Pass-through Authentication. Senast den 7 april 2027 måste alla servrar köra minst 2.6.84.0 med Application-Based Authentication."
date: "2026-10-26"
kategorie: "Microsoft Entra"
timeToRead: "8 min läsning"
themen:
  - microsoft-entra
  - active-directory-entra
produkte:
  - "entra"
  - "entra-connect-releases"
protokolle:
  - "releases"
slug: "microsoft-entra-connect-sync-2-6-91-0-och-2-6-92-0-sakerhetskorrigeringar-cloud-sync-migrering"
translationId: article-59a51ef69f2e3538
translationOf: entra-connect-2-6-92-0
url: https://rafaelpfister.ch/sv/blog/microsoft-entra-connect-sync-2-6-91-0-och-2-6-92-0-sakerhetskorrigeringar-cloud-sync-migrering
translationSourceHash: ff045b1bab72cea3839167d59c0a71010e80a3a20b1644acef89c79921f6f219
translationModel: gpt-5.6-terra
translatedAt: 2026-10-26T08:25:52.471Z
translationReview: automatic
---

# Microsoft Entra Connect Sync 2.6.91.0 och 2.6.92.0: säkerhetskorrigeringar, Cloud Sync-migrering och obligatorisk uppgradering senast i april 2027

Microsoft släppte Entra Connect Sync 2.6.91.0 den 16 september 2026 och hotfixen 2.6.92.0 den 23 september 2026, båda endast som nedladdning via Microsoft Entra Admin Center. Version 2.6.91.0 åtgärdar säkerhetsbrister i medföljande tredjepartskomponenter, gör phishingresistent inloggning i installationsguiden allmänt tillgänglig, erbjuder ett guidat migreringsflöde till Microsoft Entra Cloud Sync och korrigerar de tre kända problemen i 2.6.84.0. Version 2.6.92.0 åtgärdar ett fel i 2.6.91.0 vid konfiguration av Pass-through Authentication. Samtidigt anger Microsoft en fast tidsfrist: Senast den 7 april 2027 måste alla servrar köra minst 2.6.84.0 och använda Application-Based Authentication, annars stoppas synkroniseringen. Artikeln riktar sig till administratörer som driver Entra Connect Sync. Föregående version beskrivs i artikeln om [Entra Connect Sync 2.6.84.0](/blog/entra-connect-2-6-84-0).

## Versionerna i korthet

| Version | Datum | Typ | Huvudpunkter |
|---|---|---|---|
| 2.6.91.0 | 16.09.2026 | Funktions- och säkerhetsversion (nedladdning) | Säkerhetskorrigeringar i tredjepartskomponenter, migreringsassistent för Cloud Sync, Passkey-inloggning i guiden är GA, nya Microsoft Graph-behörigheter, korrigeringar för de kända problemen i 2.6.84.0 |
| 2.6.92.0 | 23.09.2026 | Hotfix med säkerhetskorrigeringar (nedladdning) | Åtgärdar misslyckad registrering av Authentication Agent vid aktivering av Pass-through Authentication |

Microsoft klassar 2.6.92.0 i versionsinformationen som en version med säkerhetskorrigeringar och rekommenderar uppdatering ”så snart som möjligt”. Ingen automatisk uppgraderingsutrullning har annonserats för någon av de två versionerna (status 26 oktober 2026). Även 2.6.84.0 är fortsatt endast tillgänglig som nedladdning.

## Obligatorisk uppgradering senast den 7 april 2027

Versionshistoriken och uppgraderingsanvisningarna innehåller nu samma meddelande: Entra Connect Sync måste uppdateras till version 2.6.84.0 eller senare och byta till Application-Based Authentication (ABA) senast den 7 april 2027. Microsoft avvecklar den äldre autentiseringen via Directory Synchronization Account; servrar som inte uppfyller dessa krav synkroniserar inte längre efter detta datum. Om synkroniseringen stoppas återställer enligt Microsoft en uppgradering till den aktuella versionen följd av ABA-konfiguration driften.

För planeringen är en beteendeförändring i 2.6.84.0 avgörande: Sedan denna version växlar Entra Connect inte längre automatiskt befintliga servrar till ABA. Endast nyinstallationer konfigurerar ABA automatiskt. Enbart en uppgradering uppfyller alltså inte tidsfristen. Befintliga servrar måste växlas i guiden via **Additional tasks** > **Configure application-based authentication to Microsoft Entra ID**.

Vilken metod en server för närvarande använder visas av guiden under **View or export current configuration**. Microsoft anger dessutom denna cmdlet:

```powershell
Get-ADSyncEntraConnectorCredential
```

Returvärdet `ConnectorIdentityType` är `ServiceAccount` (äldre konto) eller `Application` (ABA). Om `ServiceAccount` visas krävs omställningen före den 7 april 2027. Efter en lyckad omställning rekommenderar Microsoft att det gamla Directory Synchronization Account tas bort från Entra ID.

## Version 2.6.91.0

### Säkerhetskorrigeringar

Versionsinformationen nämner åtgärdade säkerhetsbrister i medföljande tredjepartskomponenter, härdade behörigheter för lokala temporära filer och säkrade nedladdningar av profilbilder i SharePoint-anslutningen. Microsoft anger inte CVE-nummer, CVSS-värden eller berörda komponenter. Därför går det inte att bedöma allvarlighetsgraden utifrån de offentliga källorna.

### Nya Microsoft Graph-behörigheter och Conditional Access

Entra Connect får ytterligare Microsoft Graph-behörigheter. Microsoft påpekar uttryckligen att den som använder appbaserade Conditional Access-principer bör granska principer som riktar sig till `Microsoft.Azure.SyncFabric` eller Microsoft 365 Reporting Service. Granska dessa principer före uppgraderingen så att de nya åtkomsterna inte blockeras.

### Guidad migrering till Cloud Sync

Nytt är ett guidat flöde för övergång från Entra Connect Sync till Microsoft Entra Cloud Sync. Det omfattar en utvärdering av den befintliga konfigurationen, installation av Provisioning Agent, en stegvis aktivering och validering. Funktionen är endast tillgänglig i Azure Public Cloud. Microsofts uppgraderingsanvisningar rekommenderar ändå att före en uppgradering kontrollera om Cloud Sync passar den egna miljön.

### Phishingresistent inloggning i guiden är allmänt tillgänglig

Inloggningen via Windows Web Account Manager, som introducerades som förhandsversion i 2.6.84.0, är nu generally available och aktiverad som standard. Dialogrutan stöder passkeys, FIDO2-säkerhetsnycklar och lösenord, återanvänder en befintlig inloggningssession för ytterligare Entra-tjänster och behåller Kerberos-nyckelrotationen för Seamless Single Sign-On.

### Ändrade PowerShell-cmdlets

2.6.84.0 gjorde parametern `-AADUsername` obligatorisk för `Set-ADSyncAADCompanyFeature` och `Set-ADSyncAADPasswordSyncState`. Med 2.6.91.0 är parametern åter valfri: Om den saknas härleder Entra Connect en inloggningsledtråd från anslutningskonfigurationen och öppnar en interaktiv inloggning. Detta gäller för `Set-ADSyncAADCompanyFeature`, `Set-ADSyncAADPasswordSyncState`, `Enable-ADSyncExportDeletionThreshold`, `Set-ADSyncScheduler` och `Set-ADSyncDirSyncConfiguration`. Den interaktiva inloggningen förblir därmed en del av dessa cmdlets; skript som anropar dem obevakat bör testas efter uppgraderingen.

Den som konfigurerar Seamless Single Sign-On med den fristående PowerShell-modulen måste importera `ADSync.psd1` före `AzureADSSO.psd1`.

### Ytterligare ändringar

- Dialogrutan **Select Containers** i Synchronization Service Manager är nu skrivskyddad. Ändringar av containerurvalet görs via **Customize synchronization options** i guiden.
- Stöd för ytterligare en Sovereign Cloud-miljö, inklusive Pass-through Authentication, Seamless Single Sign-On, Password Writeback och övervakning med Health Agent. Versionsinformationen anger inte vilken miljö som avses.
- Den medföljande SQL Server 2022 LocalDB har uppdaterats från 16.0.4250.1 till 16.0.4262.2.

### Felkorrigeringar

- Installation eller uppgradering med en befintlig ADSync-databas kunde misslyckas med felet `0xE0474352` (känt problem i 2.6.84.0).
- En uppgradering uppdaterade inte en installerad Visual C++ 2015–2022 Redistributable äldre än 14.41.
- Guiden kunde stängas vid start om principen för Windows PowerShell Transcription innehöll en ogiltig utdatakatalog.
- Certifikatrotationen för Application-Based Authentication kunde vid replikeringsfördröjningar i katalogen ta bort den sista användbara Application Key.
- Synchronization Service Manager kunde stängas vid listning av katalogpartitionerna i **Connector Properties** (känt problem i 2.6.84.0).
- **Change Credentials** öppnades inte för anslutningsrader utanför det synliga området när många Active Directory-anslutningar är konfigurerade.
- När en tidigare avmarkerad domän expanderades på sidan **Domain and OU filtering** kunde guiden välja domänen igen och efter slutförandet aktivera synkronisering för hela domänen (känt problem i 2.6.84.0).

Den sista punkten förtjänar uppmärksamhet: Den som kör 2.6.84.0 och har genomgått guiden på nytt efter uppgraderingen bör kontrollera domän- och OU-filtreringen. En oavsiktligt aktiverad domän leder till ytterligare objekt i Entra ID.

## Version 2.6.92.0

Hotfixen åtgärdar ett fel i 2.6.91.0: Aktivering av Pass-through Authentication via guiden kunde misslyckas vid registrering av den lokalt installerade Microsoft Entra Connect Authentication Agent. Versionsinformationen nämner inga ytterligare ändringar. Microsoft markerar 2.6.92.0 som en version med säkerhetskorrigeringar; enskilda brister anges inte.

## Kända problem

| Problem | Berörd version | Status |
|---|---|---|
| Installation eller uppgradering med befintlig ADSync-databas misslyckas med `0xE0474352` | 2.6.84.0 | Åtgärdat i 2.6.91.0 |
| Synchronization Service Manager stängs vid **Connector Properties** | 2.6.84.0 | Åtgärdat i 2.6.91.0 |
| Guiden väljer en avmarkerad domän igen | 2.6.84.0 | Åtgärdat i 2.6.91.0 |
| PTA-aktivering i guiden misslyckas vid agentregistrering | 2.6.91.0 | Åtgärdat i 2.6.92.0 |
| Synkroniseringen misslyckas efter uppgradering (`System.IO.FileLoadException`) om `miiserver.exe.config` tidigare har ändrats | 2.5.190.0, 2.6.1.0 | Fortsatt listat i versionshistoriken; lösning via manuell bindingRedirect, se [artikeln om 2.6.84.0](/blog/entra-connect-2-6-84-0) |

Den tillbakadragna versionen 2.6.79.0 är fortfarande inte tillgänglig. Microsoft hänvisar nu berörda användare till 2.6.92.0: avinstallera och installera den aktuella versionen.

## Supporttider

Varje 2.x-version hamnar utanför support tolv månader efter att efterföljande version har släppts. Med de två nya versionerna gäller följande tidsfrister:

| Version | Släppt | Support upphör |
|---|---|---|
| 2.5.79.0 | 1 september 2025 | 23 oktober 2026 (utgånget) |
| 2.5.190.0 | 19 november 2025 | 2 februari 2027 |
| 2.6.1.0 | 2 februari 2026 | 10 mars 2027 |
| 2.6.3.0 | 10 mars 2026 | 7 juli 2027 |
| 2.6.84.0 | 7 juli 2026 | 16 september 2027 |
| 2.6.91.0 | 16 september 2026 | 23 september 2027 |
| 2.6.92.0 | 23 september 2026 | ännu inte fastställt |

För versionerna 2.5.190.0 till 2.6.3.0 gäller dessutom tidsfristen den 7 april 2027: De ligger under den obligatoriska minimiversionen 2.6.84.0 och synkroniserar inte längre efter detta datum, även om ordinarie support för 2.6.3.0 inte upphör förrän i juli 2027. Version 2.5.79.0 saknar support sedan den 23 oktober 2026; enligt Microsoft kan versioner som inte längre stöds oväntat sluta fungera.

## Rekommenderat tillvägagångssätt

Målversionen är 2.6.92.0. Den innehåller säkerhetskorrigeringarna från båda versionerna, åtgärdar alla kända problem i 2.6.84.0 och 2.6.91.0 samt uppfyller minimiversionen för tidsfristen den 7 april 2027. Installationsprogrammet finns endast i Microsoft Entra Admin Center under **Microsoft Entra Connect | Get started**, fliken **Manage**; förutsättningarna är .NET Framework 4.7.2 och TLS 1.2. Kontrollera före uppgraderingen appbaserade Conditional Access-principer för `Microsoft.Azure.SyncFabric` och Microsoft 365 Reporting Service samt skript som anropar de nämnda cmdlets för molnkonfiguration. Microsoft rekommenderar Swing Migration för uppgraderingar med låg risk: uppdatera först staging-servern, verifiera konfigurationen efter Full Import och Full Synchronization, byt sedan roller och uppdatera den tidigare aktiva servern. Den som inte har uppdaterat på 12 till 18 månader bör enligt Microsoft välja denna metod. Efter uppgraderingen ska befintliga servrar som fortfarande rapporterar `ServiceAccount` växlas till Application-Based Authentication, och domän- och OU-filtreringen bör kontrolleras på servrar som tidigare körde 2.6.84.0.

## Källor

1.  [Microsoft Entra Connect: Version release history](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-version-history): Versionsinformation för 2.6.91.0 och 2.6.92.0, kända problem i 2.6.84.0, avvecklingstabell och tidsfristen den 7 april 2027.

2.  [Microsoft Entra Connect: Upgrade from a previous version](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-upgrade-previous-version): Uppgraderingsmetoder, Swing Migration via en staging-server och information om obligatorisk uppgradering.

3.  [Authenticate to Microsoft Entra ID by using application identity](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/authenticate-application-id): Manuell omställning av befintliga servrar till ABA, `Get-ADSyncEntraConnectorCredential` och borttagning av det äldre kontot.
