---
slug: "entra-connect-sync-2-6-84-0-hva-som-endres-og-hvem-som-bor-oppdatere-na"
title: "Entra Connect Sync 2.6.84.0: Hva som endres, og hvem som bør oppdatere nå"
navTitle: "Entra Connect 2.6.84"
description: "Sikkerhetsutgivelsen gir støtte for passkeys og endringer i appautentisering, PowerShell og Password Hash Sync. Forgjengerversjonen ble trukket tilbake; derfor krever oppdateringen en gradvis vurdering."
date: "2026-07-17"
kategorie: "Microsoft Entra"
timeToRead: "11 min lesetid"
themen:
  - microsoft-entra
  - active-directory-entra
draft: false
translationOf: "entra-connect-2-6-84-0"
url: "https://rafaelpfister.ch/no/blog/entra-connect-sync-2-6-84-0-hva-som-endres-og-hvem-som-bor-oppdatere-na"
translationId: article-85bd27acb917e406
translationReview: automatic
translationSourceHash: e4dc8f6498301c03d85afdba4b310d0af7ba497f7ee781448e2d02e5c62d26d9
translatedAt: 2026-07-29T12:29:38.966Z
---

# Entra Connect Sync 2.6.84.0: Hva som endres, og hvem som bør oppdatere nå

Microsoft lanserte Entra Connect Sync 2.6.84.0 som en sikkerhetsutgivelse 7. juli 2026 og anbefaler en rask oppgradering. Samtidig ble den direkte forgjengeren 2.6.79.0 trukket tilbake på grunn av et installasjonsproblem som ble oppdaget i ettertid. Konsekvensen er verken «installer umiddelbart overalt» eller «vent og ignorer»: Berørte systemer og systemer som snart faller ut av støtte, bør bytte raskt, mens alle andre først kan teste oppdateringen kontrollert.

## Hvorfor denne utgivelsen fortjener særlig forsiktighet

2.6-linjen av Entra Connect Sync har hatt en humpete start. En kort oppsummering, fordi den er relevant for oppdateringsbeslutningen:

- **2.6.1.0** (februar 2026) rettet blant annet en feil der redigering av Entra ID-connector-konfigurasjonen i Synchronization Service Manager slettet parameterne for Application-Based Authentication, slik at wizard og sertifikatrotasjon mislyktes. For alle 2.5-versjoner gjaldt derfor den bemerkelsesverdige anbefalingen om rett og slett ikke å bruke produktets administrasjonsgrensesnitt.
- **2.6.3.0** (mars 2026) var en hurtigreparasjon for et problem der Auto-Upgrade kunne stoppe Entra Connect-serveren uventet. Den daværende nødløsningen: Auto-Upgrade oppdager manuelt endrede konfigurasjonsfiler og hopper ganske enkelt over slike servere.
- **2.6.79.0** (juni 2026) ble trukket helt tilbake etter publisering. Installasjonsprogrammet er ikke lenger tilgjengelig; ifølge Microsoft skal de som har installert versjonen, avinstallere den og installere 2.6.84.0. Microsoft dokumenterer ikke nøyaktig hva problemet var.

Per i dag er versjon 2.6.84.0 bare tilgjengelig som nedlasting via Microsoft Entra Admin Center («Released for download»). En Auto-Upgrade-utrulling er ennå ikke kunngjort. Også dette er et signal: Microsoft distribuerer ennå ikke versjonen bredt til eksisterende installasjoner.

## Nye funksjoner

### Phishing-resistent pålogging i setup-wizard (forhåndsversjon)

Setup-wizard støtter nå pålogging med passkeys og FIDO2-sikkerhetsnøkler via Windows Web Account Manager (WAM). Bakgrunnen er at Microsoft siden 2024/2025 gradvis har håndhevet MFA for pålogginger til Azure- og Entra-administrasjonsgrensesnitt, og mange organisasjoner har begrenset administratorkontoene sine til phishing-resistente metoder (FIDO2, passkeys, sertifikatbasert autentisering) med Conditional Access. Nettopp disse forsvarlig sikrede kontoene kunne hittil ikke logge på i Entra Connect-wizard, fordi den innebygde påloggingsdialogen ikke støttet metodene. I praksis førte dette til uheldige omgåelser: for eksempel egne «setup-kontoer» med svakere autentiseringskrav, bare for at wizard skulle fullføres. Dette gapet lukkes nå, om enn foreløpig som forhåndsversjon.

### Støtte for den franske Sovereign Cloud

2.6.84.0 gir støtte for det franske Sovereign Cloud-miljøet, inkludert Pass-through Authentication, Seamless Single Sign-On, Password Writeback og Health Agent-overvåking. I samme forbindelse ble en feil rettet der Application Proxy-skynavnet i France Cloud ikke ble løst korrekt, og PTA-registreringen mislyktes med «EnvironmentName attribute is invalid».

## Atferdsendringer i detalj

Den mest interessante delen av utgivelsen er ikke de nye funksjonene, men den endrede atferden. Flere av dem korrigerer designbeslutninger som i praksis har ført til overraskelser.

### Auto-Upgrade ødelegger ikke lenger tilpassede konfigurasjonsfiler

Dette er endringen med lengst forhistorie. Tidligere overskrev Auto-Upgrade filen `miiserver.exe.config` fullstendig ved oppdatering. Manuelle tilpasninger gikk tapt. Det høres ut som et randtilfelle, men var det ikke: Microsoft hadde selv instruert administratorer i FIPS-miljøer om å redigere nettopp denne filen, slik at Password Hash Synchronization fungerer med aktivert FIPS-modus. De som fulgte den offisielle veiledningen, hadde altså en «modifisert» konfigurasjonsfil.

Konsekvensene viste seg ved oppgradering til 2.5.190.0 og 2.6.1.0 som et kjent problem: Hvis installasjonsprogrammet oppdager en endret `miiserver.exe.config`, lar det filen være urørt; men da mangler den nye assembly-bindingen, og synkroniseringstjenesten dør etter oppgraderingen med `System.IO.FileLoadException: Could not load file or assembly 'System.Diagnostics.DiagnosticSource, Version=6.0.0.1'`. Den dokumenterte løsningen: Legg manuelt til en bindingRedirect i `assemblyBinding`-seksjonen i `miiserver.exe.config` (under `%programfiles%\Microsoft Azure AD Sync\Bin`):

```xml
<dependentAssembly>
  <assemblyIdentity name="System.Diagnostics.DiagnosticSource" publicKeyToken="cc7b13ffcd2ddd51" culture="neutral" />
  <bindingRedirect oldVersion="0.0.0.0-8.0.0.0" newVersion="8.0.0.0" />
</dependentAssembly>
```

Start deretter ADSync-tjenesten på nytt. Hurtigreparasjonen 2.6.3.0 reduserte problemet bare for Auto-Upgrade: Berørte servere ble ganske enkelt hoppet over og forble på den gamle versjonen. Med 2.6.84.0 kommer den egentlige løsningen: Oppgraderingsprosessen slår sammen kundetilpasninger med den nye konfigurasjonen og validerer resultatet før det tas i bruk. De som oppgraderer manuelt fra en berørt versjon, bør likevel kontrollere tilstanden til `miiserver.exe.config` på forhånd og sikkerhetskopiere filen: Sammenslåingsmekanismen er ny og dermed heller ikke utprøvd i praksis ennå.

### Application-Based Authentication: Slutt på stille fallback og stille overgang

Til påminnelse: Siden 2.5.76.0 har Application-Based Authentication (ABA) vært generelt tilgjengelig og standard. I stedet for den gamle Directory Synchronization Accounts (en skykonto med lagret passord), autentiserer synkroniseringsserveren seg som en Entra ID-applikasjon med et sertifikat, helst TPM-beskyttet. Dette er en betydelig mer robust arkitektur: Ingen passord som kan lekke, og en legitimasjon som er bundet til maskinen.

2.6.84.0 rydder opp i to atferdsmønstre som har undergravd denne sikkerhetsgevinsten:

**Ingen stille fallback lenger.** Hvis ABA-oppsettet i wizard mislyktes, falt oppsettet tidligere lydløst tilbake til den eldre kontoen. Resultatet: Administratoren trodde at de hadde sertifikatbasert pålogging, mens serveren i virkeligheten kjørte med den gamle passordkontoen. Et klassisk fail-open-mønster. Nå avbryter wizard med en tydelig feilmelding («Microsoft Entra Connect could not configure application-based authentication for this server. Setup cannot continue.»), slik at den faktiske årsaken blir rettet i stedet for skjult.

**Ingen automatisk overgang i bakgrunnen lenger.** Tidligere byttet Entra Connect eksisterende servere automatisk fra den eldre kontoen til ABA mens synkroniseringen kjørte. Godt ment fra et sikkerhetsperspektiv, men et mareritt driftsmessig: En autentiseringsmetode endres uten forvarsel, uten endringsvindu og uten at noen vet om det. Og hvis noe går galt (TPM-problemer, Conditional Access-konflikter, brannmur), stopper synkroniseringen. Nå gjelder: Bare nye installasjoner konfigurerer ABA automatisk; eksisterende servere bytter først når en administrator starter wizard og eksplisitt velger **Configure application-based authentication to Microsoft Entra ID**. Overgangen hører dermed igjen hjemme der den hører til: i en planlagt endring.

I tillegg er TPM-håndteringen forbedret: Oppsettet tester nå et sertifikats signeringsevne på forhånd og håndterer TPM-signaturkontrollen korrekt. På servere med feilaktig TPM-fastvare som ikke kan generere en gyldig signatur, faller oppsettet kontrollert tilbake til et programvarebasert sertifikat. Også dette har en forhistorie: TPM-relaterte ABA-feil gikk igjen gjennom flere tidligere utgivelser (2.5.79.0, 2.5.190.0), blant annet på grunn av inkompatibiliteter mellom TPM-implementasjoner og standard signeringsmetode i MSAL-biblioteket.

### PowerShell-cmdlets krever nå eksplisitt administratorpålogging

En endring skriptoperatører må ha på radaren: Cmdletene `Set-ADSyncAADCompanyFeature` og `Set-ADSyncAADPasswordSyncState`, som endrer skykonfigurasjon, krever nå parameteren `-AADUsername` for interaktiv administratorautentisering. Wizard selv skriver heller ikke lenger skyendringer med lagrede tjenestelegitimasjoner, men via en interaktiv MSAL-pålogging. Avinstallasjons-wizard ber dessuten om administratorlegitimasjon for å rydde opp i skykonfigurasjonen; hvis dette hoppes over, ryddes det bare opp lokalt.

Bakgrunnen er den samme røde tråden som ved ABA: Handlinger mot tenant skal kunne tilordnes en reell, sporbar administratoridentitet fremfor en anonym tjenestekonto. Dette passer med en feilretting i samme utgivelse: Tidligere logget administratorrevisjonen ved endringer i synkroniseringsregler identiteten til tjenestekontoen i stedet for administratoren som faktisk handlet: et revisjonsspor som ikke oppfyller formålet sitt. Først sammen gir de brukbar revisjon. Den praktiske konsekvensen: De som hittil har kalt disse cmdletene uovervåket i skript, må bygge om disse prosessene: Interaktiv autentisering og automatisering passer ikke sammen.

### PHS-self-healing fjernet

Den mest diskrete, men konseptuelt interessante endringen: Password Hash Synchronization reaktiverer ikke lenger sitt skyfunksjonsflagg automatisk i bakgrunnen. Hvis flagget er deaktivert, må en administrator aktivere det igjen eksplisitt.

Tidligere gjaldt: Hvis PHS ble deaktivert på tenantnivå (bevisst eller ved et uhell), «helbredet» funksjonen seg selv og slo seg på igjen. For miljøer som med hensikt hadde deaktivert PHS (for eksempel av compliance-grunner, fordi ingen passordhasher skal flyte til skyen, eller under en migreringsfase), var dette en funksjon som overstyrte en dokumentert administratorbeslutning. At nettopp en mekanisme som synkroniserer passordhasher, reaktiverte seg selv på eget initiativ, var vanskelig å forklare.

Men baksiden må ikke forties: Self-healing har også reddet miljøer der flagget ble deaktivert av en feil eller et mislykket skript, uten at noen merket det. Denne sikringen faller nå bort. De som bruker PHS i produksjon (selv om det bare er som fallback for nødinnlogging), bør fremover overvåke PHS-status aktivt, for eksempel via Entra Connect Health eller ved å se på synkroniseringens heartbeat-verdier.

### Oppdaterte komponenter: SQL LocalDB 2022, MSAL, VC++-runtime

Mindre spektakulær, men på overtid, er moderniseringen av de medfølgende komponentene:

- **SQL Server LocalDB 2019 → 2022.** Den interne databasen i Entra Connect var tidligere basert på SQL Server 2019 Express LocalDB (en versjon der mainstream-støtten opphørte i februar 2025). Med SQL Server 2022 er installasjonen igjen på en versjon med løpende støtte.
- **MSAL 4.64.1 → 4.83.3.** Microsoft Authentication Library er den sentrale komponenten for all tokeninnhenting (ABA, wizard-pålogging, PowerShell). Hoppet på rundt tjue mindre versjoner inkluderer de akkumulerte feilrettingene og forbedringene i biblioteket.
- **Visual C++ Redistributable 2013 → 2015–2022 (14.42).** Det bemerkelsesverdige her er mindre oppdateringen enn den gamle avhengigheten: Frem til denne utgivelsen krevde Entra Connect et kjøretidsmiljø med støtte som utløp i april 2024. VC++ 2013-avhengigheten er nå fjernet helt.

Dette passer med den generelle merknaden i release notes om at «multiple security vulnerabilities in bundled third-party dependencies» er rettet. Dette er trolig hovedårsaken til klassifiseringen som sikkerhetsutgivelse: Utdaterte medfølgende komponenter er ikke et kosmetisk problem i et produkt som kjører med rettigheter nær Domain Admin i sentrum av identitetsinfrastrukturen.

## De øvrige feilrettingene

For fullstendighetens skyld, de resterende korrigeringene:

- **Metaverse-søk i Synchronization Service Manager** er reparert. Etter advarselen om ikke å bruke grensesnittet i eldre versjoner i det hele tatt, blir det nå tilsynelatende vedlikeholdt igjen.
- **PowerShell-diagnoserapport (HTML)** gjengis igjen korrekt; relevant for alle som bruker `Invoke-ADSyncDiagnostics` i supportsaker.
- **Generic SQL Connector:** Profileropprettelse mislyktes fordi obligatoriske parametere ikke ble fylt ut ved konfigurasjon. Gjelder miljøer som kobler til flere kataloger via GSQL-connectoren.
- **China Cloud:** Instansnavnet ble ikke løst korrekt av Discovery Endpoint API, noe som kunne føre til at gjenkjenningen av skyinstansen mislyktes.
- **Administratorrevisjon** logger nå den faktiske administratoren i stedet for tjenestekontoen ved endringer i synkroniseringsregler (se ovenfor).

## Støttefrister: Hvem må likevel handle nå

Siden mars 2023 gjelder en streng retirement-policy for Entra Connect Sync 2.x: Hver versjon faller ut av støtte tolv måneder etter at etterfølgerversjonen kom. De aktuelle fristene:

| Versjon | Slutt på støtte |
| --- | --- |
| 2.5.3.0 | **31. juli 2026** |
| 2.5.76.0 | 1. september 2026 |
| 2.5.79.0 | 23. oktober 2026 |
| 2.5.190.0 | 2. februar 2027 |
| 2.6.1.0 | 10. mars 2027 |
| 2.6.3.0 | 7. juli 2027 |

De som fortsatt kjører 2.5.3.0, har dermed bare to uker igjen av støtteperioden. Her er spørsmålet ikke om det skal oppdateres, men bare til hvilken versjon. Microsoft understreker dessuten at versjoner som er ute av støtte, kan slutte å fungere «unexpectedly»; for de utgåtte 1.x-versjonene er synkroniseringen nå faktisk slått av på serversiden. Minimumskravene forblir .NET Framework 4.7.2 og TLS 1.2; installasjonsprogrammet finnes utelukkende i Entra Admin Center (Entra ID → Entra Connect → Get started), ikke lenger i Download Center.

## Anbefaling etter utgangsversjon

Microsoft anbefaler å oppdatere «så snart som mulig». Denne anbefalingen sto imidlertid ordrett også over versjon 2.6.79.0, versjonen som senere ble trukket tilbake. Den nyere utgivelseshistorikken (tilbaketrukket installasjonsprogram, hurtigreparasjon på grunn av stoppede servere, UI-advarsler over flere versjoner) begrunner en nøktern avveiing fremfor en refleks.

Min vurdering for typiske miljøer:

**Det er forsvarlig å vente noen uker** hvis du kjører en fortsatt støttet versjon (2.5.190.0 eller nyere), ingen av de rettede problemene rammer deg akutt og ingen av de nye funksjonene trengs. Ifølge release notes ligger de rettede sikkerhetssårbarhetene i medfølgende tredjepartskomponenter; en Entra Connect-server bør uansett være så godt isolert (ingen internettilgang unntatt til Microsoft-endepunktene, ingen interaktive pålogginger, Tier 0-behandling) at tidsvinduet kan forsvares. Hvis versjonen forblir uten tilbakekalling i noen uker og Microsoft starter Auto-Upgrade-utrullingen, er det et betydelig bedre kvalitetssignal enn enhver kunngjøring.

**Du bør handle raskt** hvis ett av disse punktene gjelder:

- **Du har installert 2.6.79.0.** Da er instruksen tydelig: Avinstaller og installer 2.6.84.0, ikke vent.
- **Du kjører 2.5.3.0** (støtten utløper 31. juli 2026) eller en enda eldre versjon som allerede har utløpt.
- **Et av de rettede problemene rammer deg konkret**, for eksempel ABA-oppsett på TPM-servere, GSQL-connectoren eller revisjonskravet om at regelendringer skal tilordnes riktig administrator.

For selve oppgraderingen gjelder den vanlige fremgangsmåten, som er spesielt anbefalt med denne utgivelseshistorikken: Eksporter konfigurasjonen på forhånd (wizard tilbyr **View or export current configuration**), installer først oppdateringen på en server i staging mode og test der synkroniseringssykluser, wizard og sertifikatrotasjon, deretter den aktive serveren. De som har en tilpasset `miiserver.exe.config`, sikkerhetskopierer den før oppdateringen og kontrollerer etterpå om den nye sammenslåingsmekanismen har overtatt tilpasningene korrekt. De som kjører skript med `Set-ADSyncAADCompanyFeature` eller `Set-ADSyncAADPasswordSyncState`, tester disse før produksjonsutrullingen; ellers stopper de på den nye obligatoriske parameteren.

## Kilder

1. [Microsoft Entra Connect: Version release history – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-version-history): Offisielle release notes for 2.6.84.0, inkludert merknad om tilbakekalling av 2.6.79.0, retirement-tabell og det kjente problemet med modifisert miiserver.exe.config.
1. [Microsoft Entra Connect: Upgrade from a previous version to the latest – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-upgrade-previous-version): Oppgraderingsprosedyre, inkludert swing-migrering via en server i staging mode.
1. [Authenticate to Microsoft Entra ID by using application identity – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/authenticate-application-id): Hvordan Application-Based Authentication fungerer, og hvordan den erstatter den eldre tjenestekontoen.
1. [Microsoft Entra Connect: Phishing-resistant authentication – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-passwordless-authentication): Den nye passkey-/FIDO2-påloggingen i setup-wizard via Windows Web Account Manager.
1. [Microsoft Entra Connect: Automatic upgrade – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-install-automatic-upgrade): Mekanismen og forutsetningene for Auto-Upgrade, der utrullingen for 2.6.84.0 fortsatt gjenstår.
1. [Auditing administrator events in Microsoft Entra Connect Sync – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/admin-audit-logging): Administratorrevisjonen, der identitetstilordningen ved synkroniseringsregler ble korrigert i denne utgivelsen.
1. [SQL Server 2019 – Microsoft Lifecycle](https://learn.microsoft.com/en-us/lifecycle/products/sql-server-2019): Støttedatoer for den tidligere medfølgende LocalDB-basen, der mainstream-støtten opphørte i februar 2025.
