---
title: "Efterhantera Exchange-säkerhetsuppdateringarna från juli 2026 korrekt"
navTitle: "Exchange SU 07/2026"
description: "Efter installationen krävs två städåtgärder: ta kontrollerat bort den gamla mitigationen för CVE-2026-42897 och kontrollera överprivilegierade äldre grupper i Active Directory."
date: "2026-07-14"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "6 min lästid"
themen:
  - "exchange-onprem-hybrid"
  - "active-directory-entra"
slug: "efterhantera-exchange-sakerhetsuppdateringarna-fran-juli-2026-korrekt"
translationOf: "exchange-security-updates-juli-2026"
url: "https://rafaelpfister.ch/sv/blog/efterhantera-exchange-sakerhetsuppdateringarna-fran-juli-2026-korrekt"
---

# Efterhantera Exchange-säkerhetsuppdateringarna från juli 2026 korrekt

Arbetet är inte avslutat när Exchange-säkerhetsuppdateringarna från den 14 juli 2026 har installerats. Därefter bör administratörer åtgärda två kvarvarande delar: mitigationen för **CVE-2026-42897** som aktiverades i maj och två historiska Exchange-säkerhetsgrupper med omfattande behörigheter i Active Directory.

Båda uppgifterna är lätta att förbise. Mitigationen ligger avsiktligt kvar tills den tas bort kontrollerat. Grupperna kan i sin tur ha överlevt varje migrering obemärkt i många år.

## Vilka Exchange-versioner uppdateringen är tillgänglig för

SU:erna finns tillgängliga för följande versioner:

- **Exchange Server Subscription Edition (SE) RTM**: som en vanligt tillgänglig, offentlig uppdatering.
- **Exchange Server 2019 CU14 och CU15**: endast för organisationer som är registrerade i **Period-2-ESU-programmet**.
- **Exchange Server 2016 CU23**: även detta endast via Period 2 ESU.

Exchange 2016 och 2019 har nått end of support. Den som inte ingår i Period-2-ESU-programmet (giltigt från maj till oktober 2026) får inte längre dessa uppdateringar och bör inte längre skjuta upp övergången till Exchange SE. Exchange Online-miljöer är redan skyddade; i hybridinstallationer måste SU:t ändå installeras på alla Exchange-servrar, även på rena hanteringsservrar. Vilka specifika CVE:er som åtgärdas anges som vanligt i Security Update Guide (filtret «Server Software» för Exchange SE respektive «ESU» för 2016/2019).

Det finns ett känt problem i den aktuella versionen: I hybridmiljöer kan så kallade *wrapper-meddelanden* visas i inkorgen för delade postlådor. Mer information finns i den relevanta Microsoft-supportartikeln.

## Ta bort CVE-2026-42897-mitigationen efter installationen

### Kort tillbakablick

CVE-2026-42897 offentliggjordes den 14 maj 2026: en cross-site scripting-sårbarhet (spoofing) i Outlook Web Access. En angripare skickar ett särskilt preparerat e-postmeddelande; om offret öppnar det i OWA och vissa interaktionsvillkor är uppfyllda kan godtycklig JavaScript köras i webbläsarkontexten. Exchange 2016, 2019 och SE på *alla* patchnivåer berördes. Microsoft publicerade redan samma dag en akut mitigation (ID **M2.1.x**, den konkreta IIS-regeln heter **M2.1.0**) och levererade den egentliga korrigeringen med SU:t från juni 2026.

### Varför juliuppdateringen *inte* automatiskt tar bort mitigationen

Detta är punkten som överraskar de flesta: Även efter installation av juli-SU:t förblir en redan tillämpad mitigation aktiv. Orsaken ligger i mekanismen. Mitigationen är en **Content-Security-Policy-baserad IIS URL Rewrite-regel** som har lagts till *utanför* MSI-installationsprogrammet, antingen via Emergency Mitigation Service (EM Service) eller via EOMT-skriptet. MSI-patchen byter ut binärfiler men hanterar inte dessa IIS-regler som har satts out-of-band. Därför är borttagningen ett separat manuellt steg.

För övrigt: Mitigationen skyddade ändå aldrig IE-klienter och Edge i IE-läge, eftersom Internet Explorer inte har stöd för CSP. Den som hade sådana klienter i drift var aldrig skyddad enbart genom mitigationen. Det är ytterligare ett argument för att patcha skyndsamt i stället för att förlita sig på mitigationen.

### Fallgropen: EM Service lägger till mitigationen igen

Den som tar bort regeln för tidigt får en överraskning. EM Service kör varje timme och jämför aktuellt tillstånd med kraven som levereras av Office Config Service (Flighting). Mappningen «vilken build behöver vilken mitigation» finns på serversidan. Först en ändring på serversidan markerar juli 2026-builden som «mitigation behövs inte längre». Enligt Microsoft blev denna ändring helt utrullad först omkring den 16 juli 2026. Fram till dess lägger EM Service helt enkelt tillbaka en borttagen M2.1.0-regel vid nästa timkörning.

I praktiken innebär det: Antingen väntar man med manuell borttagning till efter den 16 juli, eller så blockerar man uttryckligen mitigationen så att den inte återaktiveras.

### Så tar du bort mitigationen korrekt (EM Service-sökvägen)

Kontrollera först vad som över huvud taget har tillämpats:

```powershell
Get-ExchangeServer -Identity <servernamn> | Format-List Name,MitigationsApplied,MitigationsBlocked
```

För att förhindra återaktivering läggs mitigation-ID:t till i blockeringslistan: poster där ignoreras av EM Service vid timkörningen.

```powershell
Set-ExchangeServer -Identity <servernamn> -MitigationsBlocked @("M2.1.0")
```

Ta därefter bort den faktiska IIS-regeln. Bra att veta och sällan dokumenterat: EM Service skapar sina URL Rewrite-regler med **prefixet «EEMS `<Mitigation-ID>` `<Beschreibung>`»**. Därmed kan de hittas entydigt i IIS Manager under URL Rewrite (eller via `appcmd`/PowerShell i `applicationHost.config`) utan att behöva gissa vilken regel som hör till mitigationen. När ändringen på serversidan har rullats ut kan blockeringen tas bort igen (`-MitigationsBlocked @()`) om den bara sattes som en tillfällig lösning.

### EOMT-sökvägen (isolerade eller air-gapped-miljöer)

Om mitigationen sattes via det nedladdningsbara **EOMT-skriptet** (https://aka.ms/UnifiedEOMT) görs återställningen med rollback-växeln:

```powershell
.\EOMT.ps1 -RollbackMitigation -CVE "CVE-2026-42897"
```

Även här finns en mindre känd detalj: EOMT sparar IIS-ursprungstillståndet i en **CVE-specifik JSON-backupfil** under `%WINDIR%\System32\inetsrv\config\` före varje ändring. Rollbacken läser exakt den filen och återställer de ursprungliga inställningarna. Viktigt: En mitigation som satts med ett äldre skript (EOMTv2 osv.) måste även tas bort med dess egen rollback-mekanism: backupformaten är inte kompatibla.

### Varför det är värt att ta bort den

Mitigationen är inte «gratis». Så länge den är aktiv följer dess kända bieffekter med: OWA-funktionen «Skriv ut kalender» fungerar inte, infogade bilder kan under vissa omständigheter inte visas korrekt i OWAs läsfönster, OWA Light (`/?layout=light`) är trasigt (och kommer ändå snart att avvecklas), och publicerade kalendrar returnerar ibland fel 500. Särskilt lömskt för övervakning: Healthset **OWACalendar.Proxy** kan växla till *unhealthy* och därmed utlösa falsklarm i övervakningen. Den som har installerat SU:t men låter mitigationen stå kvar jagar till slut spöken. Så snart uppdateringen är installerad *och* mitigationen är borttagen försvinner även dessa Known Issues.

Ett specialfall: I blandade miljöer kan ännu ej uppdaterade servrar behålla mitigationen. Man bör dock veta att Office Online Server-integrationen (OOS) under vissa omständigheter inte fungerar korrekt igen förrän *alla* Exchange-servrar i organisationen är på julinivån.

## Health Checker: hitta uråldriga säkerhetsgrupper

Den andra punkten, som är oberoende av SU-versionen: **Exchange Health Checker** (https://aka.ms/ExchangeHealthChecker) kontrollerar nu om två sedan länge föråldrade säkerhetsgrupper finns: **«Exchange Domain Servers»** och **«Exchange Enterprise Servers»**.

### Varifrån grupperna kommer och varför de är en risk

Dessa båda grupper kommer från behörighetsmodellen i Exchange 2000/2003 och har varit föråldrade sedan Exchange 2007. Med Exchange 2007/2010 kom modellen Split Permissions respektive RBAC, och sedan dess används de helt enkelt inte längre. Problemet är att de inte försvann för den skull. I många kataloger ligger de obemärkta sedan omkring två årtionden och har delvis fortfarande omfattande ACL:er från den gamla modellen, alltså fler behörigheter än en modern Exchange-säkerhetsgrupp någonsin skulle ha.

Det är just detta som gör dem till en attackvektor. En vilande grupp med kvarstående, breda behörigheter är en klassisk eskaleringskedja: Den som lyckas lägga till sig själv (eller ett kontrollerat konto) i en sådan grupp ärver dess behörigheter i katalogen. Eftersom ingen aktivt övervakar gruppen märks sådan manipulation knappt.

### Varför de flesta administratörer inte har dem på radarn

Dessa grupper är en blind fläck av flera skäl: De har varit inaktiva i cirka 20 år, existerade oftast redan före nuvarande teams tid, överlever utan problem varje migrering och har hittills aldrig visats av Health Checker. Särskilt känsligt: De överlever till och med en *fullständig* avveckling av lokal Exchange. Den som har tagit bort den sista Exchange-servern rensar normalt bort serverobjekten, men förbiser helt dessa äldre grupper.

### Rensa upp

Health Checker kommer framöver att automatiskt rapportera grupperna. Manuellt hittar du dem i Active Directory (vanligtvis i `Users`-containern) eller via PowerShell:

```powershell
Get-ADGroup -Filter "Name -eq 'Exchange Domain Servers' -or Name -eq 'Exchange Enterprise Servers'"
```

Tillvägagångssätt: Kontrollera medlemskap och eventuella anpassade ACL-referenser, säkerställ att inget produktivt är beroende av dem och ta sedan bort grupperna. Eftersom de har varit föråldrade sedan 2007 kan de i den överväldigande majoriteten av miljöer tas bort utan risk. Den som inte längre använder lokal Exchange bör samtidigt planera en mer omfattande AD-rensning enligt Microsofts officiella anvisningar.

Hayes Jupe har skrivit en detaljerad vägledning för att ta bort grupperna i sitt blogginlägg [Latest Exchange health check script and deprecated groups](https://www.hayesjupe.com/latest-exchange-health-check-script-and-deprecated-groups/).

## Rekommenderat tillvägagångssätt

Kort sammanfattat är den praktiska ordningen följande: Inventera först miljön med Health Checker (den visar saknade CU:er/SU:er, öppna manuella steg *och* nu även äldre grupper). Installera sedan aktuellt CU och juli-SU:t, starta om servern och kontrollera att alla Exchange-tjänster har startat korrekt. Kör därefter Health Checker igen, ta bort CVE-2026-42897-mitigationen (efter den 16 juli eller genom att först blockera ID:t M2.1.0) och rensa slutligen bort de föråldrade säkerhetsgrupperna. SU:er är kumulativa: Den som kör en CU-version som stöds behöver inte installera varje mellanliggande SU, utan installerar direkt den senaste.

## Källor

1.  [Released: July 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-july-2026-exchange-server-security-updates/4534146): Officiellt tillkännagivande av juliutgåvan med versioner som stöds och det kända problemet med wrapper-meddelanden.

2.  [Addressing Exchange Server May 2026 vulnerability CVE-2026-42897 – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/addressing-exchange-server-may-2026-vulnerability-cve-2026-42897/4518498): Ursprungligt säkerhetsmeddelande med akut mitigation och kända bieffekter i OWA.

3.  [Released: June 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-june-2026-exchange-server-security-updates/4524491): Juniutgåvan som levererade den egentliga korrigeringen för CVE-2026-42897.

4.  [Exchange Emergency Mitigation Service (Exchange EM Service) – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/security-best-practices/exchange-emergency-mitigation-service): Hur EM Service fungerar, som jämför mitigationer varje timme och lägger tillbaka en regel som tagits bort för tidigt.

5.  [Set-ExchangeServer (ExchangePowerShell) – Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-exchangeserver): Parametrarna `MitigationsApplied` och `MitigationsBlocked` för att kontrollera mitigationer och förhindra återaktivering.

6.  [Exchange On-premises Mitigation Tool (EOMT) – Microsoft CSS-Exchange](https://microsoft.github.io/CSS-Exchange/Security/EOMT/): EOMT-skriptet inklusive rollback-växel och CVE-specifik JSON-säkring av IIS-ursprungstillståndet.

7.  [CVE-2026-42897 Detail – NVD](https://nvd.nist.gov/vuln/detail/CVE-2026-42897): Teknisk beskrivning och bedömning av sårbarheten i National Vulnerability Database.
