---
slug: "entra-connect-sync-2-6-84-0-hva-som-endres-og-hvem-som-bor-oppdatere-na"
title: "Entra Connect Sync 2.6.84.0: Hva som endres og hvem som bør oppdatere nå"
navTitle: "Entra Connect 2.6.84"
description: "Sikkerhetsutgivelsen gir støtte for passkeys og endringer i appautentisering, PowerShell og Password Hash Sync. Forgjengerversjonen ble trukket tilbake; derfor krever oppdateringen en gradert vurdering."
date: "2026-07-17"
kategorie: "Microsoft Entra"
timeToRead: "11 min lesetid"
themen:
  - microsoft-entra
  - active-directory-entra
draft: false
translationOf: "entra-connect-2-6-84-0"
translationId: article-85bd27acb917e406
translationReview: required
translationSourceHash: da16eeec10c227af5ba6f33ae138e0148db5b34736874eeed6b2b60c0b469a81
translatedAt: 2026-09-05T07:47:08.278Z
url: https://rafaelpfister.ch/no/blog/entra-connect-sync-2-6-84-0-hva-som-endres-og-hvem-som-bor-oppdatere-na
translationModel: gpt-5.6-terra
---

# Entra Connect Sync 2.6.84.0: Hva som endres og hvem som bør oppdatere nå

Microsoft publiserte Entra Connect Sync 2.6.84.0 som en sikkerhetsutgivelse 7. juli 2026 og anbefaler en rask oppgradering. Samtidig ble den direkte forgjengeren 2.6.79.0 trukket tilbake på grunn av et installasjonsproblem som ble oppdaget i etterkant. Konsekvensen er verken «installer umiddelbart overalt» eller «vent og ignorer»: Berørte systemer og systemer som snart faller utenfor støtte, bør bytte raskt, mens alle andre først kan prøve oppdateringen kontrollert.

## Hvorfor denne utgivelsen fortjener særlig forsiktighet

2.6-linjen av Entra Connect Sync har hatt en humpete start. En kort gjennomgang, fordi den er relevant for oppdateringsbeslutningen:

- **2.6.1.0** (februar 2026) rettet blant annet en feil der redigering av Entra ID-connector-konfigurasjonen i Synchronization Service Manager slettet parameterne for Application-Based Authentication, med den følge at veiviseren og sertifikatrotasjon mislyktes. For alle 2.5-versjoner gjaldt derfor den bemerkelsesverdige anbefalingen om rett og slett ikke å bruke produktets administrasjonsgrensesnitt.
- **2.6.3.0** (mars 2026) var en hurtigreparasjon for et problem der Auto-Upgrade kunne stoppe Entra Connect-serveren uventet. Den daværende nødløsningen: Auto-Upgrade oppdager manuelt endrede konfigurasjonsfiler og hopper ganske enkelt over slike servere.
- **2.6.79.0** (juni 2026) ble trukket helt tilbake etter publisering. Installasjonsprogrammet er ikke lenger tilgjengelig; de som har installert versjonen, skal ifølge Microsoft avinstallere den og installere 2.6.84.0. Microsoft dokumenterer ikke hva problemet konkret var.

Per i dag er versjon 2.6.84.0 kun tilgjengelig som nedlasting via Microsoft Entra Admin Center («Released for download»). En utrulling via Auto-Upgrade er ennå ikke kunngjort. Også dette er et signal: Microsoft distribuerer ennå ikke selv versjonen bredt til eksisterende installasjoner.

## Nye funksjoner

### Phishing-resistent pålogging i oppsettsveiviseren (forhåndsversjon)

Oppsettsveiviseren støtter nå pålogging med passkeys og FIDO2-sikkerhetsnøkler via Windows Web Account Manager (WAM). Bakgrunnen er at Microsoft siden 2024/2025 gradvis har tvunget frem MFA for pålogginger til administrasjonsgrensesnitt for Azure og Entra, og mange organisasjoner har begrenset administratorkontoene sine til phishing-resistente metoder (FIDO2, passkeys, sertifikatbasert autentisering) gjennom Conditional Access. Nettopp disse godt sikrede kontoene kunne tidligere ikke logge på i Entra Connect-veiviseren, fordi den innebygde påloggingsdialogen ikke støttet metodene. I praksis førte dette til lite elegante omveier: for eksempel egne «oppsettskontoer» med svakere autentiseringskrav, bare for å få veiviseren gjennomført. Dette gapet lukkes nå, om enn foreløpig som en forhåndsversjon.

### Støtte for den franske Sovereign Cloud

2.6.84.0 gir støtte for det franske Sovereign Cloud-miljøet, inkludert Pass-through Authentication, Seamless Single Sign-On, Password Writeback og Health Agent-overvåking. I samme omgang er det rettet en feil der Application Proxy-skynavnet i France Cloud ikke ble løst riktig, og PTA-registreringen mislyktes med «EnvironmentName attribute is invalid».

## Atferdsendringer i detalj

Den mest interessante delen av utgivelsen er ikke de nye funksjonene, men den endrede virkemåten. Flere av disse retter designbeslutninger som i praksis har ført til overraskelser.

### Auto-Upgrade ødelegger ikke lenger tilpassede konfigurasjonsfiler

Dette er endringen med lengst forhistorie. Tidligere overskrev Auto-Upgrade filen `miiserver.exe.config` fullstendig ved oppdatering. Manuelle tilpasninger gikk tapt. Det høres ut som et særtilfelle, men var det ikke: Microsoft hadde selv instruert administratorer i FIPS-miljøer om å redigere nettopp denne filen, slik at Password Hash Synchronization fungerer med aktivert FIPS-modus. Den som fulgte den offisielle veiledningen, hadde dermed en «modifisert» konfigurasjonsfil.

Konsekvensene viste seg ved oppgradering til 2.5.190.0 og 2.6.1.0 som et kjent problem: Hvis installasjonsprogrammet oppdager en endret `miiserver.exe.config`, lar det filen være urørt; men da mangler den nye assembly-bindingen, og synkroniseringstjenesten mislykkes etter oppgraderingen med `System.IO.FileLoadException: Could not load file or assembly 'System.Diagnostics.DiagnosticSource, Version=6.0.0.1'`. Den dokumenterte løsningen: legg manuelt til en bindingRedirect i `assemblyBinding`-seksjonen i `miiserver.exe.config` (under `%programfiles%\Microsoft Azure AD Sync\Bin`):

```xml
<dependentAssembly>
  <assemblyIdentity name="System.Diagnostics.DiagnosticSource" publicKeyToken="cc7b13ffcd2ddd51" culture="neutral" />
  <bindingRedirect oldVersion="0.0.0.0-8.0.0.0" newVersion="8.0.0.0" />
</dependentAssembly>
```

Start deretter ADSync-tjenesten på nytt. Hurtigreparasjonen 2.6.3.0 avdempet problemet bare for Auto-Upgrade: berørte servere ble ganske enkelt hoppet over og ble værende på den gamle versjonen. Med 2.6.84.0 kommer den egentlige løsningen: Oppgraderingsprosessen slår sammen kundetilpasninger med den nye konfigurasjonen og validerer resultatet før det brukes. Den som oppgraderer manuelt fra en berørt versjon, bør likevel kontrollere tilstanden til `miiserver.exe.config` på forhånd og sikkerhetskopiere filen: Sammenslåingsmekanismen er ny og dermed heller ikke utprøvd i praksis ennå.

### Application-Based Authentication: Slutt på stille tilbakefall og stille overgang

Som en påminnelse: Siden 2.5.76.0 er Application-Based Authentication (ABA) generelt tilgjengelig og standard. I stedet for den gamle Directory Synchronization Accounts (en skykonto med lagret passord) autentiserer synkroniseringsserveren seg som en Entra ID-applikasjon med et sertifikat, ideelt TPM-beskyttet. Dette er en langt mer robust arkitektur: intet passord som kan lekke, og en legitimasjon som er bundet til maskinen.

2.6.84.0 rydder opp i to atferdsmønstre som har undergravd denne sikkerhetsgevinsten:

**Ikke lenger stille tilbakefall.** Hvis ABA-oppsettet i veiviseren mislyktes, falt oppsettet tidligere tilbake til Legacy-kontoen uten kommentar. Resultatet: Administratoren trodde vedkommende hadde sertifikatbasert pålogging, mens serveren i realiteten kjørte med den gamle passordkontoen. Et klassisk fail-open-mønster. Nå avbryter veiviseren med en tydelig feilmelding («Microsoft Entra Connect could not configure application-based authentication for this server. Setup cannot continue.»), slik at den egentlige årsaken rettes i stedet for å skjules.

**Ikke lenger automatisk overgang i bakgrunnen.** Tidligere byttet Entra Connect selv eksisterende servere fra Legacy-kontoen til ABA under pågående synkronisering. Godt ment fra et sikkerhetsperspektiv, men en betydelig driftsrisiko: En autentiseringsmetode byttes uten forespørsel, uten endringsvindu og uten at noen vet om det. Og hvis noe går galt (TPM-problemer, Conditional Access-konflikter, brannmur), stanser synkroniseringen. Nå gjelder følgende: Bare nyinstallasjoner konfigurerer ABA automatisk; eksisterende servere bytter først når en administrator starter veiviseren og eksplisitt velger **Configure application-based authentication to Microsoft Entra ID**. Byttet hører dermed igjen hjemme der det skal være: i en planlagt endring.

I tillegg er TPM-håndteringen forbedret: Oppsettet tester nå sertifikatets signeringsevne på forhånd og håndterer TPM-signaturkontrollen riktig. På servere med feilaktig TPM-fastvare, som ikke kan generere en gyldig signatur, faller oppsettet kontrollert tilbake til et programvarebasert sertifikat. Også dette har en forhistorie: TPM-relaterte ABA-feil strakk seg over flere tidligere utgivelser (2.5.79.0, 2.5.190.0), blant annet på grunn av inkompatibilitet mellom TPM-implementasjoner og standard signeringsmetode i MSAL-biblioteket.

### PowerShell-cmdleter krever nå eksplisitt administratorpålogging

En endring skriptoperatører må kjenne til: Cmdletene `Set-ADSyncAADCompanyFeature` og `Set-ADSyncAADPasswordSyncState`, som endrer sky-konfigurasjonen, krever nå parameteren `-AADUsername` for interaktiv administratorautentisering. Også veiviseren selv skriver ikke lenger skyendringer med lagrede tjenestelegitimasjoner, men via en interaktiv MSAL-pålogging. Og avinstalleringsveiviseren ber om administratorlegitimasjoner for opprydding i sky-konfigurasjonen; hopper man over dette, ryddes det bare lokalt.

Bakgrunnen er den samme røde tråden som ved ABA: Handlinger mot tenant-en skal kunne tilordnes en reell, sporbar administratoridentitet i stedet for en anonym tjenestekonto. Dette passer med en feilretting i samme utgivelse: Tidligere logget administratorrevisjonsloggingen identiteten til tjenestekontoen i stedet for den faktiske administratoren ved endringer i synkroniseringsregler: et revisjonsspor som ikke oppfyller sitt formål. Først begge deler sammen gir en brukbar revisjon. Den praktiske konsekvensen: Den som tidligere har kalt disse cmdletene uovervåket i skript, må bygge om disse prosessene: Interaktiv autentisering og automatisering går ikke sammen.

### PHS-selvreparasjon fjernet

Den minst iøynefallende, men konseptuelt interessante endringen: Password Hash Synchronization reaktiverer ikke lenger selv feature-flagget i skyen i bakgrunnen. Hvis flagget er deaktivert, må en administrator aktivere det eksplisitt igjen.

Tidligere gjaldt følgende: Hvis PHS ble deaktivert på tenant-nivå (bevisst eller ved et uhell), «reparerte» funksjonen seg selv og slo seg på igjen. For miljøer som bevisst hadde deaktivert PHS (for eksempel av compliance-grunner, fordi ingen passordhasher skal flyte til skyen, eller i en migrasjonsfase), var dette en funksjon som satte seg over en dokumentert administratorbeslutning. At nettopp en mekanisme som synkroniserer passordhasher, reaktiverer seg selv på eget initiativ, var vanskelig å forsvare.

Men baksiden bør ikke forties: Selvreparasjonen har også reddet miljøer der flagget ble deaktivert av en feil eller et mislykket skript, uten at noen oppdaget det. Denne sikringen faller nå bort. Den som bruker PHS i produksjon (selv om det bare er som reserve for nødinnlogging), bør fremover overvåke PHS-statusen aktivt, for eksempel via Entra Connect Health eller ved å se på synkroniseringens heartbeat-verdier.

### Oppdaterte komponenter: SQL LocalDB 2022, MSAL, VC++-runtime

Mindre spektakulært, men på overtid, er moderniseringen av de medfølgende komponentene:

- **SQL Server LocalDB 2019 → 2022.** Den interne databasen i Entra Connect var tidligere basert på SQL Server 2019 Express LocalDB (en versjon hvis ordinære støtte opphørte i februar 2025). Med SQL Server 2022 er installasjonen igjen på en versjon med aktiv støtte.
- **MSAL 4.64.1 → 4.83.3.** Microsoft Authentication Library er den sentrale komponenten for all tokeninnhenting (ABA, veiviserpålogging, PowerShell). Hoppet over rundt tjue mindre versjoner tar med seg de akkumulerte rettingene og forbedringene i biblioteket.
- **Visual C++ Redistributable 2013 → 2015–2022 (14.42).** Det bemerkelsesverdige her er mindre oppdateringen enn den gamle avhengigheten: Frem til denne utgivelsen krevde Entra Connect et runtime-miljø der støtten utløp i april 2024. VC++ 2013-avhengigheten er nå fjernet helt.

Dette passer med den generelle merknaden i utgivelsesnotatene om at «multiple security vulnerabilities in bundled third-party dependencies» er rettet. Det er sannsynligvis hovedgrunnen til klassifiseringen som sikkerhetsutgivelse: Utdaterte medfølgende komponenter er ikke et kosmetisk problem i et produkt som kjører med rettigheter nær Domain Admin i sentrum av identitetsinfrastrukturen.

## De øvrige feilrettingene

For fullstendighetens skyld, de øvrige rettingene:

- **Metaverse-søk i Synchronization Service Manager** er reparert. Etter advarselen om ikke å bruke grensesnittet i eldre versjoner i det hele tatt, blir det nå tydeligvis vedlikeholdt igjen.
- **PowerShell-diagnoserapporten (HTML)** rendrer igjen riktig; relevant for alle som bruker `Invoke-ADSyncDiagnostics` for supporttilfeller.
- **Generic SQL Connector:** Profilopprettelsen mislyktes fordi obligatoriske parametere ikke ble fylt ut under konfigurasjonen. Gjelder miljøer som kobler til ytterligere kataloger via GSQL-connectoren.
- **China Cloud:** Instansnavnet ble ikke løst riktig av Discovery Endpoint API, noe som kunne gjøre at registreringen av skyinstansen mislyktes.
- **Administratorrevisjonslogging** logger nå den faktiske administratoren i stedet for tjenestekontoen ved endringer i synkroniseringsregler (se ovenfor).

## Støttefrister: hvem som likevel må handle nå

Siden mars 2023 har Entra Connect Sync 2.x hatt en streng retirement-policy: Hver versjon faller utenfor støtte tolv måneder etter at etterfølgerversjonen er utgitt. De aktuelle fristene:

| Versjon | Slutt på støtte |
| --- | --- |
| 2.5.3.0 | **31. juli 2026** |
| 2.5.76.0 | 1. september 2026 |
| 2.5.79.0 | 23. oktober 2026 |
| 2.5.190.0 | 2. februar 2027 |
| 2.6.1.0 | 10. mars 2027 |
| 2.6.3.0 | 7. juli 2027 |

Den som fortsatt kjører 2.5.3.0, har altså bare to uker igjen av støtteperioden. Spørsmålet er da ikke om det skal oppdateres, men bare til hvilken versjon. Microsoft understreker dessuten at versjoner som er utenfor støtte, kan slutte å fungere «unexpectedly»; for de avviklede 1.x-versjonene er synkroniseringen nå faktisk slått av på serversiden. Minimumskravene er fortsatt .NET Framework 4.7.2 og TLS 1.2; installasjonsprogrammet finnes kun i Entra Admin Center (Entra ID → Entra Connect → Get started), ikke lenger i Download Center.

## Anbefaling etter utgangsversjon

Microsoft anbefaler å oppdatere «så snart som mulig». Denne anbefalingen sto imidlertid ordrett også over versjon 2.6.79.0, versjonen som senere ble trukket tilbake. Den nyere utgivelseshistorikken (tilbaketrukket installasjonsprogram, hurtigreparasjon på grunn av stoppede servere, UI-advarsler over flere versjoner) rettferdiggjør en nøktern vurdering fremfor en refleks.

Min vurdering for typiske miljøer:

**Det er forsvarlig å vente noen uker** dersom du kjører en fortsatt støttet versjon (2.5.190.0 eller nyere), ingen av de rettede problemene rammer deg akutt og ingen av de nye funksjonene trengs. Ifølge utgivelsesnotatene ligger de rettede sikkerhetssårbarhetene i medfølgende tredjepartskomponenter; en Entra Connect-server bør uansett være så godt isolert (ingen Internett-tilgang unntatt til Microsoft-endepunktene, ingen interaktive pålogginger, Tier 0-behandling) at tidsvinduet kan forsvares. Hvis versjonen forblir uten tilbakekalling i noen uker og Microsoft starter Auto-Upgrade-utrullingen, er det et langt bedre kvalitetssignal enn enhver kunngjøring.

**Du bør handle raskt** dersom ett av disse punktene gjelder:

- **Du har 2.6.79.0 installert.** Da er instruksjonen tydelig: Avinstaller og installer 2.6.84.0, ikke vent.
- **Du kjører 2.5.3.0** (slutt på støtte 31. juli 2026) eller en enda eldre versjon der støtten allerede er utløpt.
- **Ett av de rettede problemene rammer deg konkret**, for eksempel ABA-oppsett på TPM-servere, GSQL-connectoren eller revisjonskravet om at regelendringer må kunne tilordnes riktig administrator.

For selve oppgraderingen gjelder den vanlige fremgangsmåten, som er særlig anbefalt med denne utgivelseshistorikken: Eksporter konfigurasjonen på forhånd (veiviseren tilbyr **View or export current configuration**), installer først oppdateringen på en server i Staging Mode og kontroller synkroniseringssykluser, veiviser og sertifikatrotasjon der, før den aktive serveren oppdateres. Den som har en tilpasset `miiserver.exe.config`, sikkerhetskopierer den før oppdateringen og kontrollerer etterpå om den nye sammenslåingsmekanismen har overtatt tilpasningene riktig. Og den som kjører skript med `Set-ADSyncAADCompanyFeature` eller `Set-ADSyncAADPasswordSyncState`, tester disse før produksjonsutrullingen; ellers stopper de på den nye obligatoriske parameteren.

## Kilder

1. [Microsoft Entra Connect: Version release history – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-version-history): Offisielle utgivelsesnotater for 2.6.84.0, inkludert merknaden om tilbakekallingen av 2.6.79.0, retirement-tabellen og det kjente problemet med modifisert miiserver.exe.config.
1. [Microsoft Entra Connect: Upgrade from a previous version to the latest – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-upgrade-previous-version): Oppgraderingsprosedyre inkludert swing-migrering via en server i Staging Mode.
1. [Authenticate to Microsoft Entra ID by using application identity – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/authenticate-application-id): Hvordan Application-Based Authentication fungerer og erstatter den eldre tjenestekontoen.
1. [Microsoft Entra Connect: Phishing-resistant authentication – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-passwordless-authentication): Den nye passkey-/FIDO2-påloggingen i oppsettsveiviseren via Windows Web Account Manager.
1. [Microsoft Entra Connect: Automatic upgrade – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-install-automatic-upgrade): Mekanismen og forutsetningene for Auto-Upgrade, der utrullingen for 2.6.84.0 fortsatt gjenstår.
1. [Auditing administrator events in Microsoft Entra Connect Sync – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/admin-audit-logging): Administratorrevisjonsloggingen, der identitetstilordningen for synkroniseringsregler ble rettet i denne utgivelsen.
1. [SQL Server 2019 – Microsoft Lifecycle](https://learn.microsoft.com/en-us/lifecycle/products/sql-server-2019): Støttedatoer for den tidligere medfølgende LocalDB-basen, der ordinær støtte opphørte i februar 2025.
