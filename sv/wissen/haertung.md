---
title: "Härdning: baslinjer, förtroendegränser och administratörskontroller"
blatt: "haertung"
description: "Teknisk härdning för administratörer av meddelandesystem och infrastruktur: säkerhetsbaslinjer, Least Functionality och Least Privilege, identitets- och hanteringslager, gränser för nätverk och e-postprotokoll, nycklar, patch- och leveranskedjerisker, loggning, avvikelsedetektering och verifiering."
fakten:
  - label: Mål
    wert: Kontrollerat minska attackytan, implicit förtroende och skadeomfång
    href: https://csrc.nist.gov/pubs/sp/800/123/final
  - label: Designprinciper
    wert: Fail-safe Defaults · Complete Mediation · Least Privilege
    href: https://web.mit.edu/Saltzer/www/publications/protection/Basic.html
  - label: Baslinje
    wert: dokumenterat, godkänt och granskningsbart önskat tillstånd
    href: https://www.cisecurity.org/controls/secure-configuration-of-enterprise-assets-and-software
  - label: Least Functionality
    wert: endast verksamhetsnödvändiga funktioner, portar, protokoll, programvara och tjänster
    href: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-53r5.pdf
  - label: Åtkomstmodell
    wert: ingen implicit tilldelning av förtroende enbart på grund av nätverksplats eller ägande
    href: https://csrc.nist.gov/pubs/sp/800/207/final
  - label: Kontrollytor
    wert: Värd · Identitet · Hantering · Nätverk/protokoll · Applikation/data · Leveranskedja/telemetri
    href: https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final
  - label: E-postgränser
    wert: Internet-relä · Submission/Access · intern leverans · administration
    href: https://csrc.nist.gov/pubs/sp/800/177/r1/final
  - label: Administratörsåtkomst
    wert: personlig · starkt autentiserad · minimalt behörig · spårbar
    href: https://www.cisecurity.org/controls/access-control-management
  - label: Hanteringsnätverk
    wert: restriktiv administrationszon separerad från produktiva dataflöden
    href: https://www.cisa.gov/resources-tools/resources/enhanced-visibility-and-hardening-guidance-communications-infrastructure
  - label: Patchning
    wert: identifiera · prioritera · hämta · installera · verifiera installationen
    href: https://csrc.nist.gov/pubs/sp/800/40/r4/final
  - label: Avvikelsebevis
    wert: jämförelse mellan önskat och faktiskt tillstånd för konfiguration, tjänster, konton, regler och loggar
    href: https://www.cisecurity.org/controls/audit-log-management
  - label: Referenser
    wert: tillverkarbaslinje · CIS Benchmark · BSI IT-Grundschutz · egen riskacceptans
    href: https://www.cisecurity.org/cis-benchmarks
werbung:
  - tools
  - newsletter
ctaThemen:
  - haertung
  - messaging
  - security
translationSourceHash: f97c0de00da2f8c1a5d89407341fb9a46b9fc213b01559d4e297feb29c0a3c83
translationModel: gpt-5.6-terra
translatedAt: 2026-10-06T11:14:22.156Z
translationReview: automatic
---

# Härdning: baslinjer, förtroendegränser och administratörskontroller

Härdning är den kontrollerade övergången av ett system till ett **dokumenterat, motiverat och verifierbart önskat tillstånd**. Den begränsar funktioner, åtkomster och förtroenderelationer till det operativa minimumet utan att okontrollerat skada den avsedda tjänsten. Resultatet är inte en så lång lista som möjligt över aktiverade säkerhetsalternativ, utan en arkitektur där varje nåbart gränssnitt, varje privilegium och varje dataflöde har ett angivet syfte, en ägare och ett bevis. NIST beskriver serversäkerhet på motsvarande sätt som urval, implementering och löpande underhåll av lämpliga kontroller; CIS Control 4 kräver säkra konfigurationer för tillgångar och programvara ([NIST SP 800-123](https://csrc.nist.gov/pubs/sp/800/123/final), [CIS Control 4: Secure Configuration](https://www.cisecurity.org/controls/secure-configuration-of-enterprise-assets-and-software)).

En produktstandard är varken automatiskt osäker eller automatiskt rätt produktionsbaslinje. Tillverkare måste täcka breda funktions- och kompatibilitetsområden. Operatören känner däremot till exponering, skyddsbehov, beroenden, återställningsförmåga och accepterade kvarstående risker. En baslinje förenar därför tillverkarrekommendationer, en lämplig CIS- eller BSI-referens och den egna arkitekturbeslutet. Avvikelser behålls inte tyst, utan dokumenteras med orsak, risk, kompenserande kontroll, ägare och utgångsdatum. CIS Benchmarks är konsensusbaserade konfigurationsrekommendationer; BSI skiljer mellan allmänna serverkrav och e-postrelaterade krav ([CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks), [BSI SYS.1.1 Allgemeiner Server](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Grundschutz/IT-GS-Kompendium_Einzel_PDFs_2022/07_SYS_IT_Systeme/SYS_1_1_Allgemeiner_Server_Edition_2022.pdf?__blob=publicationFile&v=3), [BSI APP.5.3 Allgemeiner E-Mail-Client und -Server](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Grundschutz/IT-GS-Kompendium_Einzel_PDFs_2023/06_APP_Anwendungen/APP_5_3_Allgemeiner_E-Mail_Client_und_Server_Edition_2023.pdf?__blob=publicationFile&v=3)).

Härdning börjar med den faktiska tjänsten och dess administrationsvägar, inte med en godtycklig lista över registervärden. Först inventeras komponenter, identiteter, data och nätverkssökvägar; utifrån detta skapas baslinje, undantag och verifierbara kontroller.

## Från checklista till modell för kontrollytor

För administratörer är en granskning indelad efter kontrollytor mer robust än en enda checklista för värden. Följande modell sammanfattar kontroller från NIST SP 800-53: konfigurationshantering och Least Functionality, åtkomst och autentisering, kommunikationsskydd, systemintegritet, revision samt kontroller för leveranskedja och återställning. Det är en granskningsmodell, inte en ytterligare standard ([NIST SP 800-53 Rev. 5](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final), [NIST SP 800-53 Rev. 5, PDF](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-53r5.pdf)).

| Kontrollyta | Skyddsobjekt | Typiska målvärden | Driftsbevis |
|---|---|---|---|
| Värd och runtime | Operativsystem, containrar, tjänster, filrättigheter, kernel-/runtimefunktioner | minimala paket, minimala lyssnare, oprivilegierade processer, säkra filrättigheter | tjänste- och portinventering, baslinjeskanning, integritetskontroll |
| Identitet och behörighet | Människor, tjänstekonton, roller, token, certifikat | personlig administratörsidentitet, MFA, Least Privilege, separata tjänsteidentiteter | granskning av konton/roller, autentiserings- och privilegiehändelser |
| Hanteringslager | GUI, API, SSH, PowerShell, SNMP, backup- och uppdateringskanal | dedikerad administratörszon, krypterade protokoll, Default Deny, Break-Glass-process | nåbara hanteringsvägar, AAA-loggar, konfigurationsändringar |
| Nätverk och protokoll | Lyssnare, egress, TLS, DNS, relä-, submission- och åtkomstvägar | explicita flöden per roll, inga onödiga klartextprotokoll, verifierade certifikat | brandväggsregler, paket-/TLS-tester, DNS- och e-postflödesövervakning |
| Applikation och data | Kö, postlåda, policy, parser, temporära data, nycklar | separata roller, restriktiva filrättigheter, säkra standardvärden, begränsade parser- och egressbehörigheter | fackmässiga negativa tester, kö-/policyloggar, granskning av hemligheter och nycklar |
| Leveranskedja och telemetri | Images, paket, signaturer, beroenden, loggar, tid | artefakter med support, verifierat ursprung, patchprocess, centrala manipulationsskyddade loggar | inventering, hash/signatur, patch- och avvikelserapport, larmtest |

NIST Zero Trust tillför en viktig gräns: En användare, tjänst eller enhet får inte förtroende enbart för att den finns i det interna nätverket eller tillhör organisationen. Autentisering och auktorisering utvärderas före åtkomst till en resurs. Segmentering förblir användbar, men ersätter inte identitet, policy och löpande beslut ([NIST SP 800-207](https://csrc.nist.gov/pubs/sp/800/207/final)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-haertung.svg?v=20260813" title="Interaktive Infografik: Härtungsmodell für Messaging-Systeme mit externen Mailgrenzen, Managementebene, Identität, Host, Anwendung, Daten, Lieferkette, Telemetrie und Verifikationsschleife" loading="lazy">
  <a href="/images/kb-interaktiv-haertung.svg?v=20260813">Öppna interaktiv grafik direkt</a>.
</iframe>

## Teknikstack som härdningsinventering

Härdning har ingen egen programmeringsspråks- eller produktstack. Den tillämpas på den **teknikstack som faktiskt används**. Inventeringen omfattar därför minst firmware och hypervisor, operativsystem eller containerbas, runtime och programmeringsspråk, webb- och e-postserver, bibliotek och parser, databas, kö och objektlagring, identitets- och nyckelkomponenter, hanteringsprotokoll samt loggnings- och uppdateringsvägar. NIST CM-8 kräver en inventering av systemkomponenter; CM-7 kopplar denna inventering till begränsningen till nödvändiga funktioner ([NIST SP 800-53 Rev. 5](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final)).

För varje lager dokumenteras tillverkare, ursprung, supportstatus, aktiva moduler, privilegier, lyssnare, egressmål, konfigurationskälla, patchväg och återställningsobjekt. Endast så kan en rekommendation som ”inaktivera onödiga tjänster” tillämpas på en konkret process och dess beroenden, utan att skada e-postflödet eller återställbarheten.

## Förtroendegränser för en meddelandeplattform

En meddelandeplattform har flera tekniskt olika ingångsvägar. De får inte behandlas med en enda regel som ”endast autentiserade anslutningar”:

- **Internet-relä:** En offentligt nåbar MTA tar emot meddelanden på [SMTP](/kb/smtp) port 25 från MTAs som inte är kända i förväg. Här begränsar mottagarkontroll, reläpolicy, protokolltillstånd, resursgränser, rykte och innehållskontroller risken; användarinloggning är inte den allmänna förtroendemodellen.
- **Message Submission:** Användare och applikationer överlämnar nya meddelanden som identifierade avsändare. Submission skiljer denna roll från reläet; autentisering, auktorisering, hastighetsbegränsningar och [TLS](/kb/tls) ingår i policyn.
- **E-poståtkomst:** IMAP, POP eller HTTP får åtkomst till befintliga postlådedata. RFC 8314 betraktar klartext för submission och e-poståtkomst som föråldrad och föredrar implicit TLS.
- **Interna tjänstevägar:** Gateways, kataloger, databaser, objektlagringar, köer och skannrar kommunicerar sinsemellan som tjänster. Nätverksplatsen ensam är inget identitetsbevis; varje anslutning behöver en minimal, riktad data- och behörighetsväg.
- **Hantering och uppdateringar:** Administratörs-GUI, API, SSH, fjärrhantering, backup och programvaruanskaffning har större skadeomfång än en vanlig klientväg och hör hemma i en separat hanterings- och förtroendezon.

NIST SP 800-177 behandlar domänautentisering, TLS och innehållskryptografi som kompletterande säkerhetsmekanismer kring SMTP, som fortsatt används. RFC 8314 skiljer medvetet relä från submission och access. Slutsatsen är: Härdning måste per roll granska **vem som får initiera en anslutning, vilken identitet den bär, vilka data den behandlar och vart den får kommunicera vidare** ([NIST SP 800-177 Rev. 1](https://csrc.nist.gov/pubs/sp/800/177/r1/final), [RFC 8314](https://datatracker.ietf.org/doc/html/rfc8314), [RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321)).

Inventeringen visar vad som måste skyddas. En baslinje översätter den till konkreta inställningar som versionshanteras, testas och ändras spårbart vid motiverade undantag.

## Baslinjens livscykel och kontrollerad avvikelse

En effektiv baslinje genomgår en livscykel:

1. **Inventera:** Registrera produkt, roll, programvaruversion, moduler, lyssnare, konton, dataflöden, nycklar och beroenden.
2. **Välj referens:** Mappa tillverkarbaslinje, CIS Benchmark, BSI-modul och lagkrav till den konkreta rollen.
3. **Anpassa:** Ta bort regler som inte är tillämpliga, lägg till striktare regler och motivera avvikelser riskbaserat.
4. **Pilotera:** Kontrollera funktion, prestanda, e-postflöde, övervakning, backup och [Disaster Recovery](/kb/backup-dr) i en representativ miljö.
5. **Rulla ut deklarativt:** Använd GPO, Configuration Management, image, Policy-as-Code eller tillverkar-API i stället för manuella enskilda ändringar.
6. **Kontrollera kontinuerligt:** Upptäck avvikelser, nya konton, lyssnare, paket, certifikat, regler och baslinjeändringar.
7. **Ta ur drift:** Ta kontrollerat bort åtkomst, DNS, certifikat, nycklar, data, backuper och övervakning.

Microsofts Security Compliance Toolkit kan spara, analysera, jämföra, redigera och tillämpa rekommenderade Windows-baslinjer som GPO. Det ersätter inte anpassning: En baslinje kontrolleras först i en pilotgrupp med avseende på funktion och bieffekter. Detsamma gäller CIS- och BSI-rekommendationer ([Microsoft Security Compliance Toolkit](https://learn.microsoft.com/en-us/windows/security/operating-system-security/device-management/windows-security-configuration-framework/security-compliance-toolkit-10), [CIS Benchmarks FAQ](https://www.cisecurity.org/cis-benchmarks/cis-benchmarks-faq)).

## Least Functionality: tjänster, portar och programvara

NIST-kontroll CM-7 kräver att ett system konfigureras för verksamhetsnödvändiga funktioner och att funktioner, portar, protokoll, programvara eller tjänster förbjuds eller begränsas. Den tekniska frågan är inte ”Är port 443 säker?”, utan: **Vilken process lyssnar på vilken adress, för vilken roll, från vilken zon och med vilken patch- och identitetsmodell?** ([NIST SP 800-53 Rev. 5, CM-7](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-53r5.pdf)).

Onödiga webbgränssnitt, felsökningsändpunkter, upptäcktsprotokoll, lokala databaslyssnare och äldre hanteringstjänster inaktiveras. Nödvändiga tjänster binds i möjligaste mån endast till avsedda gränssnitt. En MTA får lyssna publikt på SMTP, men inte dess databas. En API-port för administration kan vara nödvändig, men hör inte automatiskt hemma på internet. CISA rekommenderar att onödiga eller okrypterade tjänster som Telnet, FTP, TFTP, HTTP och äldre SNMP-varianter stängs av i kommunikationsinfrastruktur samt att offentligt nåbara tjänster inventeras löpande ([CISA: Enhanced Visibility and Hardening Guidance](https://www.cisa.gov/resources-tools/resources/enhanced-visibility-and-hardening-guidance-communications-infrastructure)).

### Inventera lyssnare och aktiva tjänster

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Listener- und Dienstinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-NetTCPConnection -State Listen |
  Sort-Object LocalPort |
  Select-Object LocalAddress, LocalPort, OwningProcess
Get-Service | Where-Object Status -eq Running |
  Sort-Object Name
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
ss -lntup
systemctl list-units --type=service --state=running
```

  </div>
</div>

[`Get-NetTCPConnection`](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection) och [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) visar lokala lyssnare och processer. [`Get-Service`](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-service) och [`systemctl`](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html) visar aktiva tjänster. Jämförelsen mellan önskat och faktiskt tillstånd behöver därefter en godkänd port- och tjänstmatris; en okänd lyssnare är ett fynd, men ännu ingen orsaksanalys.

Efter att onödiga funktioner har tagits bort återstår de konton och tjänster som faktiskt får agera. Deras rättigheter, inloggningsvägar och hemligheter avgör den största delen av den administrativa attackytan.

## Identiteter, konton och Least Privilege

Konton separeras efter roll: normal användaridentitet, personlig administratörsidentitet, icke-interaktivt tjänstekonto och strikt kontrollerat nödkonto. Delade administratörsinloggningar förhindrar robust attribuering. Permanenta högprivilegierade vardagskonton ökar skadeomfånget vid nätfiske samt kompromettering av webbläsare och klienter. CIS Control 5 omfattar användar-, administratörs- och tjänstekonton; CIS Control 6 omfattar tilldelning, underhåll och återkallelse av deras autentiseringsuppgifter och privilegier ([CIS Control 5: Account Management](https://www.cisecurity.org/controls/account-management), [CIS Control 6: Access Control Management](https://www.cisecurity.org/controls/access-control-management)).

Central identitet förbättrar Joiner/Mover/Leaver-processer, men ersätter inte en lokal nödväg. Ett LDAP-, Kerberos- eller SSO-avbrott får inte omöjliggöra auktoriserad återställningsåtkomst. Break-Glass-konton är därför ett medvetet litet undantag: offline-dokumenterade, starkt skyddade, inte använda i vardagen, med omedelbart larm vid användning och regelbundet testade. Tjänsteidentiteter får ingen interaktiv inloggning och endast de rättigheter, nätverksmål och hemligheter som deras uppgift kräver. Där det är möjligt föredras kortlivade token, Managed Identities eller certifikat framför statiska lösenord; deras livscykel och återställning förblir en del av driften.

MFA minskar risken för stulna lösenord, men ersätter inte minimala rättigheter och säker återställning. NIST Zero Trust kräver ett åtkomstbeslut för subjektet och, vid behov, enheten före sessionen; nätverksplats eller ägande i sig räcker inte ([NIST SP 800-207](https://csrc.nist.gov/pubs/sp/800/207/final)).

### Granska lokala och privilegierade konton

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für lokales Konteninventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-LocalUser | Select-Object Name, Enabled, LastLogon, PasswordExpires
Get-LocalGroupMember -Group Administrators
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
getent passwd
getent group sudo wheel
```

  </div>
</div>

[`Get-LocalUser`](https://learn.microsoft.com/powershell/module/microsoft.powershell.localaccounts/get-localuser) och [`Get-LocalGroupMember`](https://learn.microsoft.com/powershell/module/microsoft.powershell.localaccounts/get-localgroupmember) läser lokala Windows-konton och gruppmedlemskap. [`getent`](https://man7.org/linux/man-pages/man1/getent.1.html) frågar de konfigurerade namntjänstdatabaserna och kan därför visa både lokala och centralt upplösta konton. En granskning måste dessutom omfatta faktiska roller i produkten, API-token, SSH-nycklar, certifikat och Cloud-IAM.

## Hanteringslager och administrationsvägar

Hanteringslagret kan ändra konfiguration, nycklar, routning, uppdateringar och loggar och förtjänar en striktare gräns än nyttodatavägen. CISA rekommenderar ett Out-of-Band-hanteringsnät som är fysiskt eller logiskt separerat från det operativa dataflödet, Default-Deny-regler, dedikerade administratörsarbetsstationer och central AAA-loggning. Laterala hanteringsanslutningar mellan enheter bör också begränsas ([CISA: Enhanced Visibility and Hardening Guidance](https://www.cisa.gov/resources-tools/resources/enhanced-visibility-and-hardening-guidance-communications-infrastructure)).

För meddelandesystem innebär det:

- Administratörs-GUI, API, SSH och fjärrhantering är endast nåbara från definierade administratörszoner eller via en kontrollerad bastionsväg.
- Certifikat, konton och brandväggsregler för hantering och e-postflöde hanteras separat.
- Utgående anslutningar från hanteringslagret begränsas till mål för uppdatering, identitet, tid, loggning och backup.
- Konfigurationsändringar kräver personlig identitet, om möjligt MFA, revision och vid hög risk godkännande enligt fyrögonprincipen.
- En nödväg fungerar utan den vanliga identitets- eller hanteringsplattformen, men drivs inte som en dold permanent åtkomst.

SSH är endast en transport för administration; dess säkerhet beror på autentisering, tillåtna användargrupper, nyckelalgoritmer, vidarebefordran, filrättigheter och målbehörigheter. Att granska den effektiva serverkonfigurationen i stället för endast textfilen identifierar inkluderade filer och standardvärden.

### Visa effektiv SSH-serverkonfiguration

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für effektive OpenSSH-Konfiguration">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH.Server*'
& "$env:WINDIR\System32\OpenSSH\sshd.exe" -T
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
sshd -T
```

  </div>
</div>

[`Get-WindowsCapability`](https://learn.microsoft.com/powershell/module/dism/get-windowscapability) visar den installerade OpenSSH-komponenten; Microsoft dokumenterar sökvägar och särdrag för [`sshd_config`](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-server-configuration). [`sshd -T`](https://man.openbsd.org/sshd) visar den effektiva konfigurationen. Alternativ får inte sättas blint utifrån checklistor på internet: tillgänglighet för nödåtkomst, använda nyckeltyper och automatisering ska ingå i testet.

## Nätverksvägar: Default Deny med explicit riktning

En brandväggsregel dokumenteras som ett riktat avtal: **källa, mål, protokoll/port, initiativtagare, identitet, användningssyfte, ägare och utgångsdatum**. ”E-postserver får åtkomst till internet” är ingen teknisk specifikation. En inkommande SMTP-lyssnare behöver andra egressmål än en sandbox-worker för skadlig programvara eller administratörs-API:t. Egressfiltrering begränsar Command-and-Control, exfiltration och okontrollerad nedladdning; den måste medvetet ta hänsyn till DNS, tid, certifikatverifiering, uppdateringar och leveransmål.

Segmentering minskar rörelseutrymmet efter en kompromettering. Den är särskilt viktig mellan internetedge, e-postbehandling, postlåde-/datalagring, katalog, hantering, backup och övervakning. NIST Zero Trust varnar samtidigt för att använda nätverksposition som enda förtroendegrund. CISA rekommenderar Default Deny för hanteringsvägar och en zon som är separerad från kunddatatrafiken ([NIST SP 800-207](https://csrc.nist.gov/pubs/sp/800/207/final), [CISA: Enhanced Visibility and Hardening Guidance](https://www.cisa.gov/resources-tools/resources/enhanced-visibility-and-hardening-guidance-communications-infrastructure)).

### Granska värdbrandvägg och regelriktning

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für lokale Firewall-Regeln">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-NetFirewallProfile |
  Select-Object Name, Enabled, DefaultInboundAction, DefaultOutboundAction
Get-NetFirewallRule -Enabled True |
  Select-Object DisplayName, Direction, Action, Profile
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nft list ruleset
```

  </div>
</div>

[`Get-NetFirewallProfile`](https://learn.microsoft.com/powershell/module/netsecurity/get-netfirewallprofile) och [`Get-NetFirewallRule`](https://learn.microsoft.com/powershell/module/netsecurity/get-netfirewallrule) visar Windows-profiler och aktiva regler. [`nft`](https://netfilter.org/projects/nftables/manpage.html) visar nftables-reglerna inklusive kedjor och riktning. Utdata jämförs med den godkända dataflödesmatrisen; en Default-Deny-policy utan nödvändiga mål för DNS, tid eller certifikat är inget framgångsrikt härdningstillstånd.

## E-postprotokoll och transportförtroende

Relä, submission och access kräver olika regler för TLS och autentisering. För submission och åtkomst rekommenderar RFC 8314 TLS 1.2 eller senare och föredrar implicit TLS; åtkomst i klartext ska inte längre erbjudas. För SMTP-relä beskriver STARTTLS däremot en förhandling per hopp. Utan ytterligare policy kan en sändande MTA fortsätta leverera i klartext om TLS saknas. [DANE](/kb/tls) och MTA-STS skapar olika, mer explicita transportpolicys ([RFC 8314](https://datatracker.ietf.org/doc/html/rfc8314), [RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207), [RFC 7672](https://datatracker.ietf.org/doc/html/rfc7672), [RFC 8461](https://datatracker.ietf.org/doc/html/rfc8461)).

[SPF, DKIM och DMARC](/kb/mail-auth) autentiserar domänrelationer och policy, inte användarkonton eller innehållet i sig. [S/MIME och OpenPGP](/kb/verschluesselung) skyddar delar av meddelandet, men förändrar inget med en osäker administratörsåtkomst eller komprometterad nyckellagring. En härdningsgranskning håller dessa säkerhetsfunktioner åtskilda och kontrollerar deras beroenden: DNS, certifikat, nycklar, tid, rapporter och undantagsregler. NIST SP 800-177 placerar just dessa kompletterande mekanismer kring SMTP och DNS ([NIST SP 800-177 Rev. 1](https://csrc.nist.gov/pubs/sp/800/177/r1/final)).

### Kontrollera nåbarhet och TLS-beteende

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Mail- und Management-TLS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Test-NetConnection mx.example.ch -Port 25 -InformationLevel Detailed
curl.exe --verbose --ssl-reqd smtp://mx.example.ch:25
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nc -vz mx.example.ch 25
openssl s_client -starttls smtp -connect mx.example.ch:25 \
  -servername mx.example.ch -verify_return_error
```

  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) och [`nc`](https://man.openbsd.org/nc) kontrollerar TCP-vägen. [`curl`](https://curl.se/docs/manpage.html) och [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) visar STARTTLS, certifikatkedja och fel. Först MTA-policy och loggar svarar på om systemet vid fel har fördröjt, avvisat eller återfallit till klartext.

## Applikation, data, parser och nycklar

E-postservrar behandlar avsiktligt komplexa format som inte är betrodda. MIME-meddelanden, arkiv, dokument, bilder och HTML når parser, skannrar, konverterare, förhandsvisningar och sandlådor. Härdning begränsar därför inte bara nätverksportar, utan även processrättigheter, filsystemåtkomst, temporär lagring, CPU/RAM/filstorlek, rekursion, körbarhet och egress för analyskomponenterna. En skanner behöver åtkomst till ett granskningsobjekt, men inte automatiskt till alla postlådor, administratörshemligheter eller hanterings-API:t.

Köer och temporära kataloger innehåller konfidentiellt innehåll. Filrättigheter, kryptering, regler för radering och bevarande samt felsökningsdumpar kontrolleras uttryckligen. Loggar ska göra tillstånd och beslut spårbara, men får inte registrera lösenord, token, privata nycklar eller onödigt meddelandeinnehåll. Nycklar separeras efter syfte: TLS, DKIM, S/MIME/OpenPGP, JWT/API och backupkryptering har olika livscykler, behörigheter och återställningsregler. NIST SP 800-53 kopplar samman Least Privilege, systemintegritet, kommunikationsskydd och revision; BSI APP.5.3 konkretiserar skyddsbehovet för e-postklient och e-postserver ([NIST SP 800-53 Rev. 5](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final), [BSI APP.5.3 Allgemeiner E-Mail-Client und -Server](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Grundschutz/IT-GS-Kompendium_Einzel_PDFs_2023/06_APP_Anwendungen/APP_5_3_Allgemeiner_E-Mail_Client_und_Server_Edition_2023.pdf?__blob=publicationFile&v=3)).

## Patchar, images och leveranskedja

Patchhantering är förebyggande underhåll, inte en sporadisk nödsituation. NIST definierar processen som identifiering, prioritering, anskaffning, installation och verifiering av patchar, uppdateringar och uppgraderingar. För exponerade e-post- och hanteringskomponenter måste sårbarhetsinformation, nåbar attackyta, aktiv exploatering, datakritikalitet och tillgänglig kompensation påverka prioriteten ([NIST SP 800-40 Rev. 4](https://csrc.nist.gov/pubs/sp/800/40/r4/final)).

Uppdateringsvägen är i sig en förtroendegräns. Paket, images, containrar, plugin-program, virussignaturer och appliance-firmware hämtas från autentiserade källor och kontrolleras med tillverkarens signatur eller publicerade hash. Beroenden och byten av repository ingår i inventeringen. NIST SP 800-161 behandlar risker med produkter och tjänster vars utveckling, integration och tillhandahållande operatören endast i begränsad utsträckning kan se eller kontrollera ([NIST SP 800-161 Rev. 1](https://csrc.nist.gov/pubs/sp/800/161/r1/upd1/final)).

En härdningsuppdatering testas i ett representativt steg: start, e-postflöde, kö, TLS, katalog, policy, övervakning, backup och återställning. ”Inte patcha eftersom e-post är kritisk” byter en känd driftsrisk mot en växande säkerhetsrisk. Den bättre utformningen skapar redundans, underhållsfönster, reproducerbara byggen och testade återfallsvägar.

### Artefaktintegritet och säkerhetsrelevanta händelser

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Artefakt- und Ereignisprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-FileHash .\mail-gateway-update.bin -Algorithm SHA256
Get-WinEvent -FilterHashtable @{LogName='Security'; StartTime=(Get-Date).AddHours(-4)} |
  Select-Object TimeCreated, Id, ProviderName, Message
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
sha256sum mail-gateway-update.bin
journalctl --since '-4 hours' --priority=notice..alert
```

  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash) och [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) jämför en artefakt med en förväntad hash från en autentiserad tillverkarkälla. [`Get-WinEvent`](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent) och [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) läser händelser; produktiv detektering kräver dessutom korrekt revisionspolicy, central insamling, tidssynkronisering och definierade larm.

En härdad konfiguration förblir endast effektiv om ändringar, misslyckade kontroller och avvikelser blir synliga. Därför hör loggning och avvikelsedetektering till driften och inte först till efterkontrollen.

## Loggning, telemetri och avvikelser

Ett härdat tillstånd är inte varaktigt utan observation. Relevanta signaler är bland annat:

- lyckade och misslyckade inloggningar, MFA- och Break-Glass-användning;
- ändringar av konton, roller, token, certifikat och nycklar;
- konfigurationsändringar och avvikelser från baslinjen;
- nya lyssnare, tjänster, paket, uppgifter, containrar eller utgående mål;
- brandväggsdroppar, oväntade egressanslutningar och hanteringsåtkomst;
- fel i TLS-, DNS-, SMTP-policy-, kö- och e-postautentisering;
- inaktiverade sensorer, loggluckor, lagringsbrist och tidsavvikelse.

Loggar samlas centralt och åtkomstskyddat så att en komprometterad värd inte enkelt kan ta bort sina spår tillsammans med systemtillståndet. CIS Control 8 kräver en logghanteringsprocess, tillräckligt lagringsutrymme, standardiserad tid, detaljerade och centraliserade revisionsloggar samt granskningar. Ett larm anses först implementerat när en kontrollerad händelse utlöser det, den ansvariga funktionen ser det och en runbook leder till åtgärd ([CIS Control 8: Audit Log Management](https://www.cisecurity.org/controls/audit-log-management)).

Avvikelsedetektering jämför det faktiska tillståndet med den versionshanterade baslinjen. Jämförelsen omfattar mer än filhashar: effektiv konfiguration, konton, grupper, IAM-roller, certifikat, brandväggsregler, lyssnare, tjänster, installerade paket, images, schemalagda jobb och leverantörspolicy. Nödändringar dokumenteras i efterhand eller återställs automatiskt; annars blir det ”tillfälliga” undantagstillståndet det nya, odokumenterade standardvärdet.

## Teknisk utveckling

Saltzer och Schroeder formulerade 1975 grundläggande skyddsprinciper som små och enkla mekanismer, säkra standardvärden, fullständig auktoriseringskontroll, separation av privilegier och Least Privilege. Deras utgångspunkt var inte ett specifikt operativsystem, utan arkitekturen för kontrollerad informationsdelning i fleranvändarsystem ([Saltzer/Schroeder: Basic Principles of Information Protection](https://web.mit.edu/Saltzer/www/publications/protection/Basic.html)).

Med spridda nätverksservrar flyttades härdning ytterligare till fjärrtjänster, protokoll, patchning, revision och säker konfigurationshantering. NIST SP 800-123 sammanfattade denna serverpraxis systematiskt 2008. Konsensusbaserade CIS Benchmarks, BSI-Grundschutz-moduler och tillverkarbaslinjer gjorde säkra målkonfigurationer mer reproducerbara och jämförbara ([NIST SP 800-123](https://csrc.nist.gov/pubs/sp/800/123/final), [CIS Benchmarks FAQ](https://www.cisecurity.org/cis-benchmarks/cis-benchmarks-faq)).

Cloud-, SaaS-, API- och hybridarkitekturer försvagade senare antagandet om en tydlig intern perimeter. NIST SP 800-207 beskrev 2020 Zero Trust som en resursorienterad arkitektur utan implicit förtroende baserat på nätverksplats eller ägande. Parallellt blev programvaruleveranskedja, imageursprung och automatiserad baslinjeavvikelse egna kontrollytor. Modern härdning förenar därför klassisk minimering av värden med identitet, policy mellan tjänster, deklarativ konfiguration, bevis för leveranskedjan, telemetri och testad återställning ([NIST SP 800-207](https://csrc.nist.gov/pubs/sp/800/207/final), [NIST SP 800-161 Rev. 1](https://csrc.nist.gov/pubs/sp/800/161/r1/upd1/final)).

## Checklista för administratörer

Härdning är först avslutad när de valda åtgärderna kan verifieras i normal drift och vid återställning. Checklistan kopplar därför samman konfiguration, ansvar och bevis.

- [ ] Systemets roll, skyddsbehov, dataflöden och förtroendegränser är dokumenterade.
- [ ] Rekommendationer från tillverkare, CIS och BSI har mappats till en versionshanterad baslinje.
- [ ] Varje avvikelse har en motivering, kompenserande kontroll, ägare och utgångsdatum.
- [ ] Lyssnare, tjänster, paket, moduler och utgående mål har reducerats till det nödvändiga minimumet.
- [ ] Användar-, administratörs-, tjänste- och Break-Glass-identiteter är separerade och granskas regelbundet.
- [ ] Administratörsåtkomst använder personlig identitet, MFA, minimala rättigheter och central revision.
- [ ] Hanteringslagret och produktivt e-postflöde finns i separata, restriktiva zoner.
- [ ] Relä, submission, access och interna tjänstevägar har egna policyer för TLS, autentisering och hastighetsbegränsning.
- [ ] Parser, skannrar, temporära data, köer, nycklar och hemligheter har minimala process- och filrättigheter.
- [ ] Patchar och images kommer från autentiserade källor; ursprung och integritet verifieras.
- [ ] Baslinje-, e-postflödes-, backup- och återställningstester körs före bred utrullning.
- [ ] Loggar är centrala, tidsmässigt konsekventa, skyddade mot ändring och kopplade till testade larm.
- [ ] Avvikelser i konton, konfiguration, regler, tjänster, certifikat och programvara identifieras automatiskt.
- [ ] Återställning och nödåtkomst har testats praktiskt under de härdade förhållandena.

## Källor

- [NIST – SP 800-123, Guide to General Server Security](https://csrc.nist.gov/pubs/sp/800/123/final)
- [CIS – Control 4: Secure Configuration](https://www.cisecurity.org/controls/secure-configuration-of-enterprise-assets-and-software)
- [CIS – Benchmarks](https://www.cisecurity.org/cis-benchmarks)
- [BSI – SYS.1.1 Allgemeiner Server](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Grundschutz/IT-GS-Kompendium_Einzel_PDFs_2022/07_SYS_IT_Systeme/SYS_1_1_Allgemeiner_Server_Edition_2022.pdf?__blob=publicationFile&v=3)
- [BSI – APP.5.3 Allgemeiner E-Mail-Client und -Server](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Grundschutz/IT-GS-Kompendium_Einzel_PDFs_2023/06_APP_Anwendungen/APP_5_3_Allgemeiner_E-Mail_Client_und_Server_Edition_2023.pdf?__blob=publicationFile&v=3)
- [NIST – SP 800-53 Rev. 5](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final)
- [NIST – SP 800-53 Rev. 5, PDF](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-53r5.pdf)
- [NIST – SP 800-207, Zero Trust Architecture](https://csrc.nist.gov/pubs/sp/800/207/final)
- [NIST – SP 800-177 Rev. 1, Trustworthy Email](https://csrc.nist.gov/pubs/sp/800/177/r1/final)
- [IETF RFC 8314 – TLS for Email Submission and Access](https://datatracker.ietf.org/doc/html/rfc8314)
- [IETF RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321)
- [Microsoft Learn – Security Compliance Toolkit](https://learn.microsoft.com/en-us/windows/security/operating-system-security/device-management/windows-security-configuration-framework/security-compliance-toolkit-10)
- [CIS – Benchmarks FAQ](https://www.cisecurity.org/cis-benchmarks/cis-benchmarks-faq)
- [CISA – Enhanced Visibility and Hardening Guidance](https://www.cisa.gov/resources-tools/resources/enhanced-visibility-and-hardening-guidance-communications-infrastructure)
- [Microsoft Learn – Get-NetTCPConnection](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection)
- [Linux man-pages – ss](https://man7.org/linux/man-pages/man8/ss.8.html)
- [Microsoft Learn – Get-Service](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-service)
- [systemd – systemctl](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html)
- [CIS – Control 5: Account Management](https://www.cisecurity.org/controls/account-management)
- [CIS – Control 6: Access Control Management](https://www.cisecurity.org/controls/access-control-management)
- [Microsoft Learn – Get-LocalUser](https://learn.microsoft.com/powershell/module/microsoft.powershell.localaccounts/get-localuser)
- [Microsoft Learn – Get-LocalGroupMember](https://learn.microsoft.com/powershell/module/microsoft.powershell.localaccounts/get-localgroupmember)
- [Linux man-pages – getent](https://man7.org/linux/man-pages/man1/getent.1.html)
- [Microsoft Learn – Get-WindowsCapability](https://learn.microsoft.com/powershell/module/dism/get-windowscapability)
- [Microsoft Learn – OpenSSH Server Configuration](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-server-configuration)
- [OpenBSD – sshd manpage](https://man.openbsd.org/sshd)
- [Microsoft Learn – Get-NetFirewallProfile](https://learn.microsoft.com/powershell/module/netsecurity/get-netfirewallprofile)
- [Microsoft Learn – Get-NetFirewallRule](https://learn.microsoft.com/powershell/module/netsecurity/get-netfirewallrule)
- [Netfilter – nft manpage](https://netfilter.org/projects/nftables/manpage.html)
- [IETF RFC 3207 – SMTP STARTTLS](https://datatracker.ietf.org/doc/html/rfc3207)
- [IETF RFC 7672 – SMTP Security via DANE](https://datatracker.ietf.org/doc/html/rfc7672)
- [IETF RFC 8461 – SMTP MTA Strict Transport Security](https://datatracker.ietf.org/doc/html/rfc8461)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc manpage](https://man.openbsd.org/nc)
- [curl – command line manpage](https://curl.se/docs/manpage.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [NIST – SP 800-40 Rev. 4, Enterprise Patch Management](https://csrc.nist.gov/pubs/sp/800/40/r4/final)
- [NIST – SP 800-161 Rev. 1, Cybersecurity Supply Chain Risk Management](https://csrc.nist.gov/pubs/sp/800/161/r1/upd1/final)
- [Microsoft Learn – Get-FileHash](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash)
- [GNU Coreutils – sha256sum](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Microsoft Learn – Get-WinEvent](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent)
- [systemd – journalctl](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
- [CIS – Control 8: Audit Log Management](https://www.cisecurity.org/controls/audit-log-management)
- [Saltzer/Schroeder – Basic Principles of Information Protection](https://web.mit.edu/Saltzer/www/publications/protection/Basic.html)
