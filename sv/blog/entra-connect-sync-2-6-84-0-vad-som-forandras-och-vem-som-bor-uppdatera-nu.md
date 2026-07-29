---
slug: "entra-connect-sync-2-6-84-0-vad-som-forandras-och-vem-som-bor-uppdatera-nu"
title: "Entra Connect Sync 2.6.84.0: Vad som förändras och vem som bör uppdatera nu"
navTitle: "Entra Connect 2.6.84"
description: "Säkerhetsversionen ger stöd för passkeys och ändringar i appautentisering, PowerShell och Password Hash Sync. Föregående version har dragits tillbaka, så uppdateringen kräver ett stegvis beslut."
date: "2026-07-17"
kategorie: "Microsoft Entra"
timeToRead: "11 min lästid"
themen:
  - microsoft-entra
  - active-directory-entra
draft: false
translationOf: "entra-connect-2-6-84-0"
url: "https://rafaelpfister.ch/sv/blog/entra-connect-sync-2-6-84-0-vad-som-forandras-och-vem-som-bor-uppdatera-nu"
---

# Entra Connect Sync 2.6.84.0: Vad som förändras och vem som bör uppdatera nu

Microsoft släppte Entra Connect Sync 2.6.84.0 den 7 juli 2026 som en säkerhetsversion och rekommenderar en snabb uppgradering. Samtidigt har den direkta föregångaren 2.6.79.0 dragits tillbaka på grund av ett installationsproblem som upptäcktes i efterhand. Konsekvensen är varken «installera omedelbart överallt» eller «vänta och ignorera»: berörda system och system som snart faller ur support bör byta skyndsamt, medan alla andra först kan prova uppdateringen under kontrollerade former.

## Varför den här versionen kräver särskild försiktighet

2.6-serien av Entra Connect Sync har haft en skakig start. En kort tillbakablick, eftersom den är relevant för uppdateringsbeslutet:

- **2.6.1.0** (februari 2026) åtgärdade bland annat ett fel där redigering av Entra ID-anslutningskonfigurationen i Synchronization Service Manager raderade parametrarna för Application-Based Authentication, vilket gjorde att guiden och certifikatrotation misslyckades. För alla 2.5-versioner gällde därför den anmärkningsvärda rekommendationen att helt enkelt inte använda produktens administrationsgränssnitt.
- **2.6.3.0** (mars 2026) var en snabbfix för ett problem där Auto-Upgrade oväntat kunde stoppa Entra Connect-servern. Den dåvarande nödlösningen: Auto-Upgrade upptäcker manuellt ändrade konfigurationsfiler och hoppar helt enkelt över sådana servrar.
- **2.6.79.0** (juni 2026) drogs tillbaka helt efter publiceringen. Installationsprogrammet är inte längre tillgängligt; den som har installerat versionen ska enligt Microsoft avinstallera den och installera 2.6.84.0. Microsoft dokumenterar inte vad problemet exakt var.

Version 2.6.84.0 finns i dag endast tillgänglig för nedladdning via Microsoft Entra Admin Center («Released for download»). Någon utrullning via Auto-Upgrade har ännu inte annonserats. Även det är en signal: Microsoft distribuerar ännu inte själv versionen brett till befintliga installationer.

## Nya funktioner

### Phishingresistent inloggning i installationsguiden (förhandsversion)

Installationsguiden har nu stöd för inloggning med passkeys och FIDO2-säkerhetsnycklar via Windows Web Account Manager (WAM). Bakgrunden är att Microsoft sedan 2024/2025 stegvis kräver MFA för inloggningar till administrationsgränssnitt för Azure och Entra, och många organisationer har begränsat sina administratörskonton med Conditional Access till phishingresistenta metoder (FIDO2, passkeys, certifikatbaserad autentisering). Just dessa väl skyddade konton kunde tidigare inte logga in i Entra Connect-guiden, eftersom den inbäddade inloggningsdialogen inte stödde metoderna. I praktiken ledde detta till oattraktiva lösningar: exempelvis särskilda «installationskonton» med svagare autentiseringskrav, bara för att guiden skulle kunna slutföras. Den luckan stängs nu, om än inledningsvis som förhandsversion.

### Stöd för den franska Sovereign Cloud

2.6.84.0 ger stöd för den franska Sovereign Cloud-miljön, inklusive Pass-through Authentication, Seamless Single Sign-On, Password Writeback och övervakning med Health Agent. I samband med detta har ett fel åtgärdats där Application Proxy-molnnamnet i France Cloud inte löstes korrekt och PTA-registreringen misslyckades med «EnvironmentName-attributet är ogiltigt».

## Beteendeförändringar i detalj

Den mest intressanta delen av versionen är inte de nya funktionerna, utan de förändrade beteendena. Flera av dem korrigerar designbeslut som i praktiken har orsakat överraskningar.

### Auto-Upgrade förstör inte längre anpassade konfigurationsfiler

Detta är ändringen med längst historik. Tidigare skrev Auto-Upgrade över filen `miiserver.exe.config` helt vid uppdatering. Manuella anpassningar gick förlorade. Det låter som ett specialfall, men var det inte: Microsoft hade själv instruerat administratörer i FIPS-miljöer att redigera just den här filen för att Password Hash Synchronization skulle fungera med aktiverat FIPS-läge. Den som följde den officiella instruktionen hade alltså en «modifierad» konfigurationsfil.

Konsekvenserna visade sig som ett känt problem vid uppgradering till 2.5.190.0 och 2.6.1.0: om installationsprogrammet upptäcker en ändrad `miiserver.exe.config` lämnar det filen orörd; då saknas dock den nya assembly-bindningen och synkroniseringstjänsten slutar fungera efter uppgraderingen med `System.IO.FileLoadException: Could not load file or assembly 'System.Diagnostics.DiagnosticSource, Version=6.0.0.1'`. Den dokumenterade lösningen: lägg manuellt till en bindingRedirect i avsnittet `assemblyBinding` i `miiserver.exe.config` (under `%programfiles%\Microsoft Azure AD Sync\Bin`):

```xml
<dependentAssembly>
  <assemblyIdentity name="System.Diagnostics.DiagnosticSource" publicKeyToken="cc7b13ffcd2ddd51" culture="neutral" />
  <bindingRedirect oldVersion="0.0.0.0-8.0.0.0" newVersion="8.0.0.0" />
</dependentAssembly>
```

Starta sedan om ADSync-tjänsten. Snabbfixen 2.6.3.0 mildrade problemet endast för Auto-Upgrade: berörda servrar hoppades helt enkelt över och blev kvar på den gamla versionen. Med 2.6.84.0 kommer den faktiska lösningen: uppgraderingsprocessen sammanför kundanpassningar med den nya konfigurationen och validerar resultatet innan det tillämpas. Den som uppgraderar manuellt från en berörd version bör ändå kontrollera tillståndet för sin `miiserver.exe.config` i förväg och säkerhetskopiera filen: sammanslagningsmekanismen är ny och därmed ännu inte beprövad i praktiken.

### Application-Based Authentication: slut på tyst återgång och tyst omställning

Som påminnelse: sedan 2.5.76.0 är Application-Based Authentication (ABA) allmänt tillgänglig och standard. I stället för det gamla Directory Synchronization Account (ett molnkonto med sparat lösenord) autentiserar synkroniseringsservern sig som en Entra ID-applikation med ett certifikat, helst TPM-skyddat. Det är en betydligt robustare arkitektur: inget lösenord som kan läcka och en autentiseringsuppgift som är knuten till maskinen.

2.6.84.0 rensar upp i två beteenden som har undergrävt denna säkerhetsvinst:

**Ingen tyst återgång längre.** Om ABA-konfigurationen misslyckades i guiden föll installationen tidigare utan kommentar tillbaka till det äldre kontot. Resultatet: administratören trodde att certifikatbaserad inloggning användes, men i verkligheten körde servern med det gamla lösenordskontot. Ett klassiskt fail-open-mönster. Nu avbryts guiden med ett tydligt felmeddelande («Microsoft Entra Connect kunde inte konfigurera applikationsbaserad autentisering för den här servern. Installationen kan inte fortsätta.»), så att den egentliga orsaken åtgärdas i stället för att döljas.

**Ingen automatisk omställning i bakgrunden längre.** Tidigare växlade Entra Connect självständigt befintliga servrar från det äldre kontot till ABA under pågående synkroniseringsdrift. Välmenat ur säkerhetssynpunkt, men en mardröm ur driftsynpunkt: en autentiseringsmetod ändras utan förvarning, utan ändringsfönster och utan att någon vet om det. Och om något går fel (TPM-problem, Conditional Access-konflikter, brandvägg) stannar synkroniseringen. Nu gäller: endast nya installationer konfigurerar ABA automatiskt; befintliga servrar byter först när en administratör startar guiden och uttryckligen väljer **Configure application-based authentication to Microsoft Entra ID**. Bytet hamnar därmed åter där det hör hemma: i en planerad ändring.

Dessutom har TPM-hanteringen förbättrats: installationen testar nu i förväg om ett certifikat kan signera och hanterar TPM-signaturkontrollen korrekt. På servrar med felaktig TPM-firmware, som inte kan skapa en giltig signatur, återgår installationen kontrollerat till ett programvarubaserat certifikat. Även detta har en bakgrund: TPM-relaterade ABA-fel återkom i flera tidigare versioner (2.5.79.0, 2.5.190.0), bland annat på grund av inkompatibiliteter mellan TPM-implementeringar och standardmetoden för signering i MSAL-biblioteket.

### PowerShell-cmdlets kräver nu en uttrycklig administratörsinloggning

En förändring som skriptansvariga måste ha koll på: cmdletarna `Set-ADSyncAADCompanyFeature` och `Set-ADSyncAADPasswordSyncState`, som ändrar molnkonfiguration, kräver nu parametern `-AADUsername` för interaktiv administratörsautentisering. Även guiden själv skriver inte längre molnändringar med sparade tjänsteuppgifter, utan via en interaktiv MSAL-inloggning. Avinstallationsguiden begär också administratörsuppgifter för att rensa molnkonfigurationen; om detta hoppas över rensas endast den lokala miljön.

Bakgrunden är samma röda tråd som för ABA: åtgärder mot klientorganisationen ska kunna kopplas till en verklig, spårbar administratörsidentitet i stället för ett anonymt tjänstkonto. Det passar ihop med en buggfix i samma version: tidigare loggade administratörsgranskningen identiteten för tjänstkontot i stället för den administratör som faktiskt gjorde ändringen vid ändringar av synkroniseringsregler – ett granskningsspår som missar sitt syfte. Först tillsammans ger båda delarna användbar granskning. Den praktiska konsekvensen: den som tidigare har anropat dessa cmdlets oövervakat i skript måste bygga om sina flöden; interaktiv autentisering och automatisering går inte ihop.

### PHS-självläkning borttagen

Den mest diskreta men konceptuellt intressanta ändringen: Password Hash Synchronization återaktiverar inte längre självständigt sin molnfunktionsflagga i bakgrunden. Om flaggan är inaktiverad måste en administratör uttryckligen aktivera den igen.

Tidigare gällde följande: om PHS inaktiverades på klientorganisationsnivå (medvetet eller av misstag), «läkte» funktionen sig själv och aktiverades igen. För miljöer som medvetet hade inaktiverat PHS – exempelvis av efterlevnadsskäl, eftersom inga lösenordshashar får skickas till molnet, eller under en migreringsfas – var detta en funktion som körde över ett dokumenterat administratörsbeslut. Att just en mekanism som synkroniserar lösenordshashar självmant återaktiveras var svårt att motivera.

Baksidan får dock inte utelämnas: självläkningen räddade även miljöer där flaggan inaktiverades av ett fel eller ett misslyckat skript utan att någon märkte det. Detta skydd försvinner nu. Den som använder PHS i produktion – även om det bara är som reserv för nöd-inloggning – bör framöver aktivt övervaka PHS-statusen, exempelvis via Entra Connect Health eller genom att kontrollera synkroniseringens heartbeat-värden.

### Uppdaterade komponenter: SQL LocalDB 2022, MSAL, VC++-runtime

Mindre spektakulär men försenad är moderniseringen av de medföljande komponenterna:

- **SQL Server LocalDB 2019 → 2022.** Entra Connects interna databas baserades tidigare på SQL Server 2019 Express LocalDB (en version vars mainstream-support upphörde i februari 2025). Med SQL Server 2022 körs installationen åter på en version med aktiv support.
- **MSAL 4.64.1 → 4.83.3.** Microsoft Authentication Library är den centrala komponenten för all tokeninhämtning (ABA, guideinloggning, PowerShell). Hoppet över omkring tjugo mindre versioner tar med de ackumulerade korrigeringarna och förbättringarna i biblioteket.
- **Visual C++ Redistributable 2013 → 2015–2022 (14.42).** Det anmärkningsvärda här är mindre uppdateringen än det tekniska arvet: fram till denna version förutsatte Entra Connect en runtime-miljö vars support löpte ut i april 2024. VC++ 2013-beroendet har nu tagits bort helt.

Därtill kommer den generella noteringen i versionsinformationen att «flera säkerhetsbrister i medföljande tredjepartsberoenden» har åtgärdats. Det torde vara huvudskälet till klassificeringen som säkerhetsversion: föråldrade medföljande komponenter är inget kosmetiskt problem i en produkt som körs med rättigheter nära Domain Admin i centrum av identitetsinfrastrukturen.

## Övriga buggfixar

För fullständighetens skull återstående korrigeringar:

- **Metaverse-sökning i Synchronization Service Manager** har reparerats. Efter varningen om att inte använda gränssnittet alls i äldre versioner verkar det nu åter underhållas.
- **PowerShell-diagnosrapport (HTML)** renderas korrekt igen; relevant för alla som använder `Invoke-ADSyncDiagnostics` för supportärenden.
- **Generic SQL Connector:** profilskapandet misslyckades eftersom obligatoriska parametrar inte fylldes i under konfigurationen. Påverkar miljöer som ansluter ytterligare kataloger via GSQL-anslutningen.
- **China Cloud:** instansnamnet löstes inte korrekt av Discovery Endpoint-API:t, vilket kunde göra att identifieringen av molninstansen misslyckades.
- **Administratörsgranskning** loggar nu den faktiska administratören i stället för tjänstkontot vid ändringar av synkroniseringsregler (se ovan).

## Supporttider: vem måste ändå agera nu

Sedan mars 2023 gäller en strikt utfasningspolicy för Entra Connect Sync 2.x: varje version faller ur support tolv månader efter att efterföljande version har släppts. Aktuella datum:

| Version | Slut på support |
| --- | --- |
| 2.5.3.0 | **31 juli 2026** |
| 2.5.76.0 | 1 september 2026 |
| 2.5.79.0 | 23 oktober 2026 |
| 2.5.190.0 | 2 februari 2027 |
| 2.6.1.0 | 10 mars 2027 |
| 2.6.3.0 | 7 juli 2027 |

Den som fortfarande kör 2.5.3.0 har alltså bara två veckors support kvar. Frågan är inte om, utan endast till vilken version uppdatering ska ske. Microsoft betonar dessutom att versioner som har fallit ur support kan sluta fungera «oväntat»; för de utfasade 1.x-versionerna har synkroniseringen numera faktiskt stängts av på serversidan. Minimikraven är fortfarande .NET Framework 4.7.2 och TLS 1.2; installationsprogrammet finns endast i Entra Admin Center (Entra ID → Entra Connect → Get started), inte längre i Download Center.

## Rekommendation efter utgångsversion

Microsoft rekommenderar uppdatering «så snart som möjligt». Samma rekommendation stod dock ordagrant även för version 2.6.79.0, versionen som därefter drogs tillbaka. Den senaste versionshistoriken (tillbakadraget installationsprogram, snabbfix på grund av stoppade servrar, UI-varningar över flera versioner) motiverar en saklig avvägning i stället för en reflex.

Min bedömning för typiska miljöer:

**Det är försvarbart att vänta några veckor** om ni kör en version som fortfarande stöds (2.5.190.0 eller senare), inget av de åtgärdade problemen påverkar er akut och ingen av de nya funktionerna behövs. Enligt versionsinformationen finns de åtgärdade säkerhetsbristerna i medföljande tredjepartskomponenter; en Entra Connect-server bör ändå vara så väl isolerad (ingen internetåtkomst utöver Microsoft-slutpunkter, inga interaktiva inloggningar, Tier 0-hantering) att tidsfönstret är försvarbart. Om versionen förblir utan återkallelse i några veckor och Microsoft startar Auto-Upgrade-utrullningen är det en betydligt bättre kvalitetssignal än något tillkännagivande.

**Ni bör agera skyndsamt** om någon av följande punkter gäller:

- **Ni har 2.6.79.0 installerat.** Då är instruktionen tydlig: avinstallera och installera 2.6.84.0, vänta inte.
- **Ni kör 2.5.3.0** (slut på support den 31 juli 2026) eller en ännu äldre version som redan har gått ur support.
- **Ett av de åtgärdade problemen påverkar er konkret**, exempelvis ABA-konfiguration på TPM-servrar, GSQL-anslutningen eller granskningskravet att regeländringar ska kopplas till rätt administratör.

För själva uppgraderingen gäller det vanliga förfarandet, vilket särskilt rekommenderas med denna versionshistorik: exportera konfigurationen i förväg (guiden erbjuder **View or export current configuration**), installera först uppdateringen på en server i staging-läge och kontrollera där synkroniseringscykler, guide och certifikatrotation, och uppdatera först därefter den aktiva servern. Den som har en anpassad `miiserver.exe.config` säkerhetskopierar den före uppdateringen och kontrollerar sedan om den nya sammanslagningsmekanismen har tagit över anpassningarna korrekt. Och den som kör skript med `Set-ADSyncAADCompanyFeature` eller `Set-ADSyncAADPasswordSyncState` testar dem före produktionsutrullningen; annars avbryts de på den nya obligatoriska parametern.

## Källor

1. [Microsoft Entra Connect: Version release history – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-version-history): Officiella versionsanteckningar för 2.6.84.0 inklusive återkallelseinformationen för 2.6.79.0, utfasningstabellen och det kända problemet med modifierad miiserver.exe.config.
1. [Microsoft Entra Connect: Upgrade from a previous version to the latest – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-upgrade-previous-version): Uppgraderingsförfarande inklusive swing-migrering via en server i staging-läge.
1. [Authenticate to Microsoft Entra ID by using application identity – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/authenticate-application-id): Så fungerar Application-Based Authentication, som ersätter det äldre tjänstkontot.
1. [Microsoft Entra Connect: Phishing-resistant authentication – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-passwordless-authentication): Den nya inloggningen med passkey/FIDO2 i installationsguiden via Windows Web Account Manager.
1. [Microsoft Entra Connect: Automatic upgrade – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-install-automatic-upgrade): Mekanik och förutsättningar för Auto-Upgrade, vars utrullning för 2.6.84.0 fortfarande väntar.
1. [Auditing administrator events in Microsoft Entra Connect Sync – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/admin-audit-logging): Administratörsgranskningen vars identitetskoppling för synkroniseringsregler korrigerades i denna version.
1. [SQL Server 2019 – Microsoft Lifecycle](https://learn.microsoft.com/en-us/lifecycle/products/sql-server-2019): Supportdatum för den tidigare medföljande LocalDB-basen, vars mainstream-support upphörde i februari 2025.
