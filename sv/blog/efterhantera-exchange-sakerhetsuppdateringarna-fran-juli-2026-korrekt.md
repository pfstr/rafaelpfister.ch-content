---
title: "Efterbearbeta Exchange-säkerhetsuppdateringarna från juli 2026 korrekt"
navTitle: "Exchange SU 07/2026"
description: "Efter installationen krävs två upprensningsåtgärder: ta bort den gamla CVE-2026-42897-mitigeringen på ett kontrollerat sätt och granska överprivilegierade äldre grupper i Active Directory."
date: "2026-07-14"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "6 min. lästid"
themen:
  - exchange-updates
  - active-directory-entra
slug: "efterhantera-exchange-sakerhetsuppdateringarna-fran-juli-2026-korrekt"
translationOf: "exchange-security-updates-juli-2026"
translationId: article-731b5b840aee096c
translationReview: automatic
translationSourceHash: e5d9295515965d3e7801752cd605f6d2a78cacfc9fb965e0f63d645658b39e9b
translatedAt: 2026-09-05T07:53:35.066Z
url: https://rafaelpfister.ch/sv/blog/efterhantera-exchange-sakerhetsuppdateringarna-fran-juli-2026-korrekt
translationModel: gpt-5.6-terra
---

# Efterbearbeta Exchange-säkerhetsuppdateringarna från juli 2026 korrekt

Med installationen av Exchange-säkerhetsuppdateringarna den 14 juli 2026 är arbetet ännu inte klart. Därefter bör administratörer åtgärda två kvarvarande saker: mitigeringen för **CVE-2026-42897** som aktiverades i maj och två historiska Exchange-säkerhetsgrupper med omfattande behörigheter i Active Directory.

Båda uppgifterna är lätta att förbise. Mitigeringen finns avsiktligt kvar tills den tas bort på ett kontrollerat sätt. Grupperna kan i sin tur ha överlevt varje migrering obemärkt i många år.

## För vilka Exchange-versioner uppdateringen är tillgänglig

SU:erna finns tillgängliga för följande versioner:

- **Exchange Server Subscription Edition (SE) RTM**: som en regelbundet tillgänglig offentlig uppdatering.
- **Exchange Server 2019 CU14 och CU15**: endast för organisationer som är registrerade i **Period-2-ESU-programmet**.
- **Exchange Server 2016 CU23**: även detta endast via Period 2 ESU.

Exchange 2016 och 2019 är out of support. Den som inte är med i Period-2-ESU-programmet (gäller från maj till oktober 2026) får inte längre dessa uppdateringar och bör inte skjuta upp övergången till Exchange SE längre. Exchange Online-miljöer är redan skyddade; i hybridkonfigurationer måste SU:t ändå installeras på alla Exchange-servrar, även på rena hanteringsservrar. Vilka specifika CVE:er som åtgärdas framgår som vanligt av Security Update Guide (filtret «Server Software» för Exchange SE respektive «ESU» för 2016/2019).

Det finns ett känt problem i den aktuella versionen: I hybridmiljöer kan så kallade *wrapper-meddelanden* visas i inkorgen för delade postlådor. Detaljer finns i motsvarande Microsoft-supportartikel.

## Ta bort CVE-2026-42897-mitigeringen efter installationen

### Kort återblick

CVE-2026-42897 offentliggjordes den 14 maj 2026: en cross-site scripting-sårbarhet (spoofing) i Outlook Web Access. En angripare skickar ett särskilt preparerat e-postmeddelande; om offret öppnar det i OWA och vissa interaktionsvillkor är uppfyllda kan godtycklig JavaScript köras i webbläsarkontexten. Exchange 2016, 2019 och SE på *alla* patchnivåer berördes. Microsoft publicerade samma dag en akutmitigering (ID **M2.1.x**, den specifika IIS-regeln heter **M2.1.0**) och levererade den egentliga korrigeringen med SU:t från juni 2026.

### Varför juliuppdateringen *inte* tar bort mitigeringen automatiskt

Det är punkten som överraskar de flesta: Även efter installation av juli-SU:t förblir en redan tillämpad mitigering aktiv. Orsaken ligger i mekanismen. Mitigeringen är en **Content-Security-Policy-baserad IIS-URL-Rewrite-regel** som har lagts in *utanför* MSI-installationsprogrammet, antingen via Emergency-Mitigation-Service (EM Service) eller via EOMT-skriptet. MSI-patchen byter ut binärfiler, men hanterar inte dessa out-of-band-satta IIS-regler. Därför är borttagningen ett separat manuellt steg.

För övrigt skyddade mitigeringen aldrig IE-klienter och Edge i IE-läge, eftersom Internet Explorer inte stöder CSP. Den som använde sådana klienter var aldrig skyddad enbart av mitigeringen. Det är ytterligare ett argument för att patcha skyndsamt i stället för att förlita sig på mitigeringen.

### Den känsliga punkten: EM Service lägger tillbaka mitigeringen

En regel som tas bort förhastat förblir inte borttagen permanent. EM Service körs varje timme och jämför aktuellt tillstånd med kraven som levereras av Office Config Service (flighting). Mappningen «vilken build behöver vilken mitigering» finns på serversidan. Först en ändring på serversidan markerar juli 2026-bygget som «mitigering behövs inte längre». Enligt Microsoft hade denna ändring inte distribuerats helt förrän omkring den 16 juli 2026. Fram till dess lägger EM Service helt enkelt tillbaka en borttagen M2.1.0-regel vid nästa körning varje timme.

I praktiken innebär det: Antingen väntar man med manuell borttagning till efter den 16 juli, eller så blockerar man uttryckligen mitigeringen så att den inte återaktiveras.

### Så tar du bort mitigeringen korrekt (EM Service-sökvägen)

Börja med att kontrollera vad som faktiskt har tillämpats:

```powershell
Get-ExchangeServer -Identity <Servername> | Format-List Name,MitigationsApplied,MitigationsBlocked
```

För att förhindra återaktivering läggs mitigerings-ID:t till i blockeringslistan: poster där ignoreras av EM Service vid den timvisa körningen.

```powershell
Set-ExchangeServer -Identity <Servername> -MitigationsBlocked @("M2.1.0")
```

Ta sedan bort själva IIS-regeln. Bra att veta och sällan dokumenterat: EM Service skapar sina URL-Rewrite-regler med **prefixet «EEMS `<Mitigation-ID>` `<Beschreibung>`»**. Därmed går de att hitta entydigt i IIS Manager under URL Rewrite (eller via `appcmd`/PowerShell i `applicationHost.config`) utan att behöva gissa vilken regel som hör till mitigeringen. Efter att ändringen på serversidan har distribuerats kan du häva blockeringen igen (`-MitigationsBlocked @()`), om den endast sattes som en tillfällig lösning.

### EOMT-sökvägen (separerade eller air-gapped-miljöer)

Om mitigeringen sattes via det nedladdningsbara **EOMT-skriptet** (https://aka.ms/UnifiedEOMT) görs återställningen med rollback-växeln:

```powershell
.\EOMT.ps1 -RollbackMitigation -CVE "CVE-2026-42897"
```

Även här finns en föga känd detalj: EOMT sparar IIS-utgångsläget i en **CVE-specifik JSON-säkerhetskopieringsfil** under `%WINDIR%\System32\inetsrv\config\` före varje ändring. Rollback läser just den filen och återställer de ursprungliga inställningarna. Viktigt: En mitigering som har satts med ett legacy-skript (EOMTv2 etc.) måste också tas bort med dess egen rollback-mekanism: säkerhetskopieringsformaten är inte kompatibla.

### Varför det är värt att ta bort den

Mitigeringen är inte «gratis». Så länge den är aktiv får man dess kända bieffekter: OWA-funktionen «Skriv ut kalender» fungerar inte, infogade bilder kanske inte visas korrekt i OWA-läsfönstret, OWA Light (`/?layout=light`) är trasigt (och kommer ändå snart att stängas av), och publicerade kalendrar ger delvis fel 500. Särskilt förrädiskt för övervakning: Healthset **OWACalendar.Proxy** kan bli *unhealthy* och därmed orsaka falsklarm i övervakningen. Den som har installerat SU:t men lämnar kvar mitigeringen letar till slut efter fel som inte finns. När uppdateringen har installerats *och* mitigeringen har tagits bort försvinner även dessa Known Issues.

Ett specialfall: I blandade miljöer kan ännu inte uppdaterade servrar behålla mitigeringen. Man bör dock veta att Office Online Server-integrationen (OOS) eventuellt inte fungerar korrekt igen förrän *alla* Exchange-servrar i organisationen är på julinivån.

## Health Checker: hitta uråldriga säkerhetsgrupper

Den andra punkten, oberoende av SU-versionen: **Exchange Health Checker** (https://aka.ms/ExchangeHealthChecker) kontrollerar nu om två sedan länge deprecierade säkerhetsgrupper finns: **«Exchange Domain Servers»** och **«Exchange Enterprise Servers»**.

### Var grupperna kommer ifrån och varför de är en risk

Dessa båda grupper kommer från behörighetsmodellen i Exchange 2000/2003 och har varit deprecierade sedan Exchange 2007. Med Exchange 2007/2010 kom modellen Split Permissions respektive RBAC, och sedan dess används de helt enkelt inte längre. Problemet är att de inte har försvunnit för den skull. I många kataloger har de legat obemärkta i omkring två decennier och har delvis fortfarande omfattande ACL:er från den gamla modellen, alltså fler behörigheter än en modern Exchange-säkerhetsgrupp någonsin skulle ha.

Det är just detta som gör dem till en angreppsvektor. En inaktiv grupp med kvarstående, breda behörigheter är en klassisk eskaleringskedja: Den som lyckas lägga till sig själv (eller ett kontrollerat konto) i en sådan grupp ärver dess behörigheter i katalogen. Eftersom ingen aktivt övervakar gruppen märks en sådan manipulation knappast.

### Varför de flesta administratörer inte känner till dem

Dessa grupper är en blind fläck av flera skäl: De har varit inaktiva i omkring 20 år, fanns vanligen redan före det nuvarande teamets tid, överlever utan problem varje migrering och har hittills aldrig visats av Health Checker. Särskilt känsligt: De överlever till och med en *fullständig* avveckling av lokal Exchange. Den som har tagit bort den sista Exchange-servern rensar vanligen bort serverobjekten, men missar helt dessa äldre grupper.

### Rensa upp

Health Checker kommer framöver att automatiskt rapportera grupperna. Manuellt hittar du dem i Active Directory (vanligen i `Users`-containern) eller via PowerShell:

```powershell
Get-ADGroup -Filter "Name -eq 'Exchange Domain Servers' -or Name -eq 'Exchange Enterprise Servers'"
```

Tillvägagångssätt: Kontrollera medlemskap och eventuella anpassade ACL-referenser, säkerställ att inget produktivt hänvisar till dem och ta sedan bort grupperna. Eftersom de har varit deprecierade sedan 2007 kan de avlägsnas utan risk i den överväldigande majoriteten av miljöer. Den som inte längre använder någon lokal Exchange bör samtidigt planera en mer omfattande AD-rensning enligt Microsofts officiella instruktioner.

Hayes Jupe har skrivit en detaljerad guide för att ta bort grupperna i sitt blogginlägg [Latest Exchange health check script and deprecated groups](https://www.hayesjupe.com/latest-exchange-health-check-script-and-deprecated-groups/).

## Rekommenderat tillvägagångssätt

Kort sammanfattat är det praktiska förloppet: Inventera först miljön med Health Checker (den visar saknade CU:er/SU:er, öppna manuella steg *och* nu även äldre grupper). Installera sedan aktuellt CU och juli-SU:t, starta om servern och kontrollera att alla Exchange-tjänster har startat korrekt. Kör därefter Health Checker igen, ta bort CVE-2026-42897-mitigeringen (efter den 16 juli eller med föregående blockering av ID M2.1.0) och rensa slutligen upp de deprecierade säkerhetsgrupperna. SU:er är kumulativa: Den som använder ett CU som stöds behöver inte installera varje mellanliggande SU, utan installerar direkt den senaste.

## Källor

1.  [Released: July 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-july-2026-exchange-server-security-updates/4534146): Officiellt tillkännagivande av juliversionen med de versioner som stöds och det kända problemet med wrapper-meddelanden.

2.  [Addressing Exchange Server May 2026 vulnerability CVE-2026-42897 – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/addressing-exchange-server-may-2026-vulnerability-cve-2026-42897/4518498): Ursprungligt säkerhetsmeddelande inklusive akutmitigering och de kända bieffekterna i OWA.

3.  [Released: June 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-june-2026-exchange-server-security-updates/4524491): Juniversionen som levererade den egentliga korrigeringen för CVE-2026-42897.

4.  [Exchange Emergency Mitigation Service (Exchange EM Service) – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/security-best-practices/exchange-emergency-mitigation-service): Hur EM Service fungerar, vilken jämför mitigeringar varje timme och lägger tillbaka en regel som har tagits bort förhastat.

5.  [Set-ExchangeServer (ExchangePowerShell) – Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-exchangeserver): Parametrarna `MitigationsApplied` och `MitigationsBlocked` för att kontrollera mitigeringar och förhindra återaktivering.

6.  [Exchange On-premises Mitigation Tool (EOMT) – Microsoft CSS-Exchange](https://microsoft.github.io/CSS-Exchange/Security/EOMT/): EOMT-skriptet inklusive rollback-växel och CVE-specifik JSON-säkerhetskopia av IIS-utgångsläget.

7.  [CVE-2026-42897 Detail – NVD](https://nvd.nist.gov/vuln/detail/CVE-2026-42897): Teknisk beskrivning och bedömning av sårbarheten i National Vulnerability Database.
