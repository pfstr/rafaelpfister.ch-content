---
slug: "entra-connect-sync-2-6-84-0-vad-som-forandras-och-vem-som-bor-uppdatera-nu"
title: "Entra Connect Sync 2.6.84.0: Vad som förändras och vem som bör uppdatera nu"
navTitle: "Entra Connect 2.6.84"
description: "Säkerhetsreleasen ger stöd för passkeys och förändringar i appautentisering, PowerShell och Password Hash Sync. Föregående version har dragits tillbaka; därför kräver uppdateringen ett stegvis beslut."
date: "2026-07-17"
kategorie: "Microsoft Entra"
timeToRead: "11 min lästid"
themen:
  - microsoft-entra
  - active-directory-entra
draft: false
translationOf: "entra-connect-2-6-84-0"
translationId: article-85bd27acb917e406
translationReview: required
translationSourceHash: da16eeec10c227af5ba6f33ae138e0148db5b34736874eeed6b2b60c0b469a81
translatedAt: 2026-09-05T07:46:24.977Z
url: https://rafaelpfister.ch/sv/blog/entra-connect-sync-2-6-84-0-vad-som-forandras-och-vem-som-bor-uppdatera-nu
translationModel: gpt-5.6-terra
---

# Entra Connect Sync 2.6.84.0: Vad som förändras och vem som bör uppdatera nu

Microsoft släppte Entra Connect Sync 2.6.84.0 den 7 juli 2026 som en säkerhetsrelease och rekommenderar en snabb uppgradering. Samtidigt drogs den direkta föregångaren 2.6.79.0 tillbaka på grund av ett installationsproblem som upptäcktes i efterhand. Konsekvensen är varken ”installera omedelbart överallt” eller ”vänta och ignorera”: berörda system och system som snart lämnar supporten bör byta skyndsamt, medan övriga först kan testa uppdateringen under kontrollerade former.

## Varför denna release förtjänar särskild försiktighet

2.6-serien av Entra Connect Sync har haft en skakig start. En kort tillbakablick, eftersom den är relevant för beslutet om uppdatering:

- **2.6.1.0** (februari 2026) åtgärdade bland annat ett fel där redigering av Entra ID-anslutningskonfigurationen i Synchronization Service Manager raderade parametrarna för Application-Based Authentication, vilket gjorde att guiden och certifikatrotationen misslyckades. För alla 2.5-versioner gällde därför den anmärkningsvärda rekommendationen att helt enkelt inte använda produktens administrationsgränssnitt.
- **2.6.3.0** (mars 2026) var en snabbkorrigering för ett problem där Auto-Upgrade oväntat kunde stoppa Entra Connect-servern. Den tillfälliga lösningen då: Auto-Upgrade identifierar manuellt ändrade konfigurationsfiler och hoppar helt enkelt över sådana servrar.
- **2.6.79.0** (juni 2026) drogs tillbaka helt efter publiceringen. Installationsprogrammet är inte längre tillgängligt; enligt Microsoft ska den som har versionen installerad avinstallera den och installera 2.6.84.0. Microsoft dokumenterar inte exakt vad problemet var.

Version 2.6.84.0 finns i dagsläget endast tillgänglig för nedladdning via Microsoft Entra Admin Center (”Released for download”). Någon utrullning via Auto-Upgrade har ännu inte annonserats. Även detta är en signal: Microsoft distribuerar ännu inte själv versionen brett till befintliga installationer.

## Nya funktioner

### Phishingresistent inloggning i installationsguiden (förhandsversion)

Installationsguiden har nu stöd för inloggning med passkeys och FIDO2-säkerhetsnycklar via Windows Web Account Manager (WAM). Bakgrunden är att Microsoft sedan 2024/2025 stegvis har krävt MFA för inloggningar till administrationsgränssnitt för Azure och Entra, och många organisationer har begränsat sina administratörskonton med Conditional Access till phishingresistenta metoder (FIDO2, passkeys, certifikatbaserad autentisering). Just dessa väl skyddade konton kunde tidigare inte logga in i Entra Connect-guiden eftersom den inbäddade inloggningsdialogen inte stödde metoderna. I praktiken ledde det till mindre eleganta lösningar: till exempel separata ”installationskonton” med svagare autentiseringskrav, enbart för att guiden skulle kunna köras. Denna lucka stängs nu, om än tills vidare som förhandsversion.

### Stöd för den franska Sovereign Cloud

2.6.84.0 ger stöd för den franska Sovereign Cloud-miljön, inklusive Pass-through Authentication, Seamless Single Sign-On, Password Writeback och övervakning med Health-agenten. I samband med detta har ett fel åtgärdats där Application Proxy-molnnamnet i France Cloud inte löstes korrekt och PTA-registreringen misslyckades med ”EnvironmentName attribute is invalid”.

## Beteendeförändringar i detalj

Den mest intressanta delen av releasen är inte de nya funktionerna, utan de ändrade beteendena. Flera av dem korrigerar designbeslut som har orsakat överraskningar i praktiken.

### Auto-Upgrade förstör inte längre anpassade konfigurationsfiler

Detta är förändringen med längst förhistoria. Tidigare skrev Auto-Upgrade över filen `miiserver.exe.config` helt vid uppdateringen. Manuella anpassningar gick förlorade. Det låter som ett specialfall, men det var det inte: Microsoft hade självt instruerat administratörer i FIPS-miljöer att redigera just denna fil, så att Password Hash Synchronization fungerar med aktiverat FIPS-läge. Den som följde den officiella instruktionen hade alltså en ”modifierad” konfigurationsfil.

Konsekvenserna blev synliga vid uppgraderingen till 2.5.190.0 och 2.6.1.0 som ett känt problem: om installationsprogrammet upptäcker en ändrad `miiserver.exe.config`, lämnar det filen orörd; då saknas dock den nya assembly-bindningen och synkroniseringstjänsten misslyckas efter uppgraderingen med `System.IO.FileLoadException: Could not load file or assembly 'System.Diagnostics.DiagnosticSource, Version=6.0.0.1'`. Den dokumenterade lösningen: lägg manuellt till en bindingRedirect i avsnittet `assemblyBinding` i `miiserver.exe.config` (under `%programfiles%\Microsoft Azure AD Sync\Bin`):

```xml
<dependentAssembly>
  <assemblyIdentity name="System.Diagnostics.DiagnosticSource" publicKeyToken="cc7b13ffcd2ddd51" culture="neutral" />
  <bindingRedirect oldVersion="0.0.0.0-8.0.0.0" newVersion="8.0.0.0" />
</dependentAssembly>
```

Starta sedan om ADSync-tjänsten. Snabbkorrigeringen 2.6.3.0 mildrade problemet endast för Auto-Upgrade: berörda servrar hoppades helt enkelt över och stannade kvar på den gamla versionen. Med 2.6.84.0 kommer den egentliga lösningen: uppgraderingsprocessen sammanför kundanpassningar med den nya konfigurationen och validerar resultatet innan det tillämpas. Den som uppgraderar manuellt från en berörd version bör ändå kontrollera tillståndet för sin `miiserver.exe.config` i förväg och säkerhetskopiera filen: sammanslagningsmekanismen är ny och därmed ännu inte beprövad i praktiken.

### Application-Based Authentication: slut på tyst fallback och tyst omställning

Som påminnelse: sedan 2.5.76.0 är Application-Based Authentication (ABA) allmänt tillgänglig och standard. I stället för det gamla Directory Synchronization Account (ett molnkonto med sparat lösenord) autentiserar synkroniseringsservern sig som en Entra ID-applikation med ett certifikat, helst TPM-skyddat. Det är en betydligt robustare arkitektur: inget lösenord som kan läcka och en autentiseringsuppgift som är bunden till maskinen.

2.6.84.0 rättar till två beteenden som har undergrävt denna säkerhetsvinst:

**Ingen tyst fallback längre.** Om ABA-konfigurationen misslyckades i guiden föll installationen tidigare tillbaka till det äldre kontot utan kommentar. Resultatet: administratören trodde att certifikatbaserad inloggning användes, medan servern i själva verket körde med det gamla lösenordskontot. Ett klassiskt fail-open-mönster. Nu avbryts guiden med ett tydligt felmeddelande (”Microsoft Entra Connect could not configure application-based authentication for this server. Setup cannot continue.”), så att den verkliga orsaken åtgärdas i stället för att döljas.

**Ingen automatisk omställning i bakgrunden längre.** Tidigare ställde Entra Connect automatiskt om befintliga servrar från det äldre kontot till ABA under pågående synkronisering. Välmenat ur säkerhetssynpunkt, men en betydande risk ur driftsperspektiv: en autentiseringsmetod ändras utan förvarning, utan ändringsfönster och utan att någon känner till det. Om något går fel (TPM-problem, Conditional Access-konflikter, brandvägg) stannar synkroniseringen. Nu gäller: endast nya installationer konfigurerar ABA automatiskt; befintliga servrar byter först när en administratör startar guiden och uttryckligen väljer **Configure application-based authentication to Microsoft Entra ID**. Bytet hör därmed åter hemma där det ska vara: i en planerad ändring.

Dessutom har hanteringen av TPM förbättrats: installationen testar nu i förväg om ett certifikat kan signera och hanterar TPM-signaturkontrollen korrekt. På servrar med felaktig TPM-firmware som inte kan skapa en giltig signatur faller installationen kontrollerat tillbaka till ett programvarubaserat certifikat. Även detta har en förhistoria: TPM-relaterade ABA-fel förekom i flera tidigare releaser (2.5.79.0, 2.5.190.0), bland annat på grund av inkompatibiliteter mellan TPM-implementationer och MSAL-bibliotekets standardsignaturmetod.

### PowerShell-cmdlets kräver nu en uttrycklig administratörsinloggning

En förändring som skriptoperatörer måste känna till: cmdletarna `Set-ADSyncAADCompanyFeature` och `Set-ADSyncAADPasswordSyncState`, som ändrar molnkonfigurationen, kräver nu parametern `-AADUsername` för interaktiv administratörsautentisering. Även guiden själv skriver inte längre molnändringar med sparade tjänsteautentiseringsuppgifter, utan via en interaktiv MSAL-inloggning. Avinstallationsguiden frågar också efter administratörsautentiseringsuppgifter för att rensa upp molnkonfigurationen; om man hoppar över detta rensas endast lokalt.

Bakgrunden är samma röda tråd som för ABA: åtgärder mot klientorganisationen ska kopplas till en verklig, spårbar administratörsidentitet i stället för ett anonymt tjänstkonto. Det passar ihop med en buggfix i samma release: tidigare loggade administratörsgranskningen vid ändringar i synkroniseringsregler tjänstkontots identitet i stället för den administratör som faktiskt utförde åtgärden – ett granskningsspår som missar sitt syfte. Först tillsammans ger båda delarna användbar granskning. Den praktiska konsekvensen: den som tidigare anropade dessa cmdlets obevakat i skript måste bygga om dessa flöden; interaktiv autentisering och automatisering fungerar inte tillsammans.

### PHS-självläkning borttagen

Den mest diskreta men konceptuellt intressanta förändringen: Password Hash Synchronization återaktiverar inte längre självständigt sitt molnfunktionsflagga i bakgrunden. Om flaggan är inaktiverad måste en administratör uttryckligen slå på den igen.

Tidigare gällde följande: om PHS inaktiverades på klientorganisationsnivå (medvetet eller av misstag) ”läkte” funktionen sig själv och aktiverades igen. För miljöer som avsiktligt hade inaktiverat PHS (exempelvis av efterlevnadsskäl, eftersom inga lösenordshashar får skickas till molnet, eller under en migreringsfas) var detta en funktion som åsidosatte ett dokumenterat administratörsbeslut. Att just en mekanism som synkroniserar lösenordshashar återaktiverar sig på eget initiativ var svårt att motivera.

Nackdelen får dock inte utelämnas: självläkningen räddade även miljöer där flaggan inaktiverades av ett fel eller ett misslyckat skript utan att någon märkte det. Detta skydd försvinner nu. Den som använder PHS i produktion (även om det bara är som reserv för nödinloggning) bör framöver aktivt övervaka PHS-statusen, exempelvis via Entra Connect Health eller genom att kontrollera synkroniseringens heartbeat-värden.

### Uppdaterade komponenter: SQL LocalDB 2022, MSAL, VC++-runtime

Mindre spektakulär, men välbehövlig, är moderniseringen av de medföljande komponenterna:

- **SQL Server LocalDB 2019 → 2022.** Entra Connects interna databas baserades tidigare på SQL Server 2019 Express LocalDB (en version vars mainstream-support upphörde i februari 2025). Med SQL Server 2022 använder installationen åter en version med aktiv support.
- **MSAL 4.64.1 → 4.83.3.** Microsoft Authentication Library är den centrala komponenten för all tokenhämtning (ABA, guideinloggning, PowerShell). Hoppet över omkring tjugo minor-versioner innehåller de ackumulerade korrigeringarna och förbättringarna i biblioteket.
- **Visual C++ Redistributable 2013 → 2015–2022 (14.42).** Det anmärkningsvärda här är mindre uppdateringen än det gamla arvet: fram till denna release förutsatte Entra Connect en runtime-miljö vars support upphörde i april 2024. Beroendet av VC++ 2013 är nu helt borttaget.

Detta passar ihop med den generella noteringen i release notes om att ”multiple security vulnerabilities in bundled third-party dependencies” har åtgärdats. Det är sannolikt huvudorsaken till klassificeringen som säkerhetsrelease: föråldrade medföljande komponenter är inte ett kosmetiskt problem i en produkt som körs med rättigheter nära Domain Admin i centrum av identitetsinfrastrukturen.

## Övriga buggfixar

För fullständighetens skull, de återstående korrigeringarna:

- **Metaverse-sökning i Synchronization Service Manager** har reparerats. Efter varningen att inte använda gränssnittet alls i äldre versioner verkar det nu åter underhållas.
- **PowerShell-diagnosrapporten (HTML)** renderas åter korrekt; relevant för alla som använder `Invoke-ADSyncDiagnostics` för supportärenden.
- **Generic SQL Connector:** profilskapandet misslyckades eftersom obligatoriska parametrar inte fylldes i vid konfigurationen. Berör miljöer som ansluter ytterligare kataloger via GSQL Connector.
- **China Cloud:** instansnamnet löstes inte korrekt av Discovery Endpoint API, vilket kunde få identifieringen av molninstansen att misslyckas.
- **Administratörsgranskningsloggning** loggar nu den faktiska administratören i stället för tjänstkontot vid ändringar i synkroniseringsregler (se ovan).

## Supportfrister: vem som ändå måste agera nu

Sedan mars 2023 gäller en strikt pensioneringspolicy för Entra Connect Sync 2.x: varje version lämnar supporten tolv månader efter att efterföljande version har släppts. Aktuella tidsfrister:

| Version | Supportens slut |
| --- | --- |
| 2.5.3.0 | **31 juli 2026** |
| 2.5.76.0 | 1 september 2026 |
| 2.5.79.0 | 23 oktober 2026 |
| 2.5.190.0 | 2 februari 2027 |
| 2.6.1.0 | 10 mars 2027 |
| 2.6.3.0 | 7 juli 2027 |

Den som fortfarande kör 2.5.3.0 har alltså endast två veckors support kvar. Här är frågan inte om, utan endast till vilken version uppdateringen ska ske. Microsoft betonar dessutom att versioner som har lämnat supporten kan sluta fungera ”unexpectedly”; för de utfasade 1.x-versionerna har synkroniseringen numera faktiskt stängts av på serversidan. Minimikraven förblir .NET Framework 4.7.2 och TLS 1.2; installationsprogrammet finns endast i Entra Admin Center (Entra ID → Entra Connect → Get started), inte längre i Download Center.

## Rekommendation efter utgångsversion

Microsoft rekommenderar att uppdatera ”så snart som möjligt”. Den rekommendationen stod dock ordagrant även för version 2.6.79.0, versionen som sedan drogs tillbaka. Den senaste releasehistoriken (tillbakadraget installationsprogram, snabbkorrigering på grund av stoppade servrar, varningar om gränssnittet över flera versioner) motiverar en saklig avvägning snarare än en reflex.

Min bedömning för typiska miljöer:

**Det är försvarbart att vänta några veckor** om ni kör en fortfarande stödd version (2.5.190.0 eller senare), inget av de åtgärdade problemen påverkar er akut och ingen av de nya funktionerna behövs. Enligt release notes finns de åtgärdade säkerhetsbristerna i medföljande tredjepartskomponenter; en Entra Connect-server bör ändå vara så väl isolerad (ingen internetåtkomst utöver Microsoft-slutpunkter, inga interaktiva inloggningar, Tier 0-hantering) att tidsfönstret kan försvaras. Om versionen förblir utan återkallelse i några veckor och Microsoft inleder utrullningen av Auto-Upgrade är det en betydligt bättre kvalitetssignal än något tillkännagivande.

**Ni bör agera skyndsamt** om någon av dessa punkter gäller:

- **Ni har 2.6.79.0 installerad.** Då är instruktionen tydlig: avinstallera och installera 2.6.84.0, vänta inte.
- **Ni kör 2.5.3.0** (supporten upphör 31 juli 2026) eller en ännu äldre version som redan har löpt ut.
- **Ett av de åtgärdade problemen berör er konkret**, exempelvis ABA-konfiguration på TPM-servrar, GSQL Connector eller granskningskravet att regeländringar kopplas till rätt administratör.

För själva uppgraderingen gäller det vanliga förfarandet, vilket särskilt rekommenderas med denna releasehistorik: exportera konfigurationen i förväg (guiden erbjuder **View or export current configuration**), installera först uppdateringen på en server i staging mode och kontrollera där synkroniseringscykler, guide och certifikatrotation, därefter den aktiva servern. Den som har en anpassad `miiserver.exe.config` säkerhetskopierar den före uppdateringen och kontrollerar efteråt om den nya sammanslagningsmekanismen har tagit över anpassningarna korrekt. Den som kör skript med `Set-ADSyncAADCompanyFeature` eller `Set-ADSyncAADPasswordSyncState` testar dem före produktionsutrullningen; annars avbryts de av den nya obligatoriska parametern.

## Källor

1. [Microsoft Entra Connect: Version release history – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-version-history): Officiella release notes för 2.6.84.0, inklusive information om återkallelsen av 2.6.79.0, pensioneringstabellen och det kända problemet med modifierad miiserver.exe.config.
1. [Microsoft Entra Connect: Upgrade from a previous version to the latest – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-upgrade-previous-version): Uppgraderingsförfarande inklusive swing-migrering via en server i staging mode.
1. [Authenticate to Microsoft Entra ID by using application identity – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/authenticate-application-id): Hur Application-Based Authentication fungerar, som ersätter det äldre tjänstkontot.
1. [Microsoft Entra Connect: Phishing-resistant authentication – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-passwordless-authentication): Den nya inloggningen med passkeys/FIDO2 i installationsguiden via Windows Web Account Manager.
1. [Microsoft Entra Connect: Automatic upgrade – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-install-automatic-upgrade): Mekaniken och förutsättningarna för Auto-Upgrade, vars utrullning för 2.6.84.0 fortfarande väntar.
1. [Auditing administrator events in Microsoft Entra Connect Sync – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/admin-audit-logging): Administratörsgranskningsloggningen, vars identitetskoppling för synkroniseringsregler korrigerades i denna release.
1. [SQL Server 2019 – Microsoft Lifecycle](https://learn.microsoft.com/en-us/lifecycle/products/sql-server-2019): Supportdatum för den tidigare medföljande LocalDB-basen, vars mainstream-support upphörde i februari 2025.
