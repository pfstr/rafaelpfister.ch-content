---
title: "SEPPmail 15.0.6 och 15.0.6.1: Säkerhetskorrigeringar och nya adminfunktioner"
navTitle: "SEPPmail 15.0.6"
description: "SEPPmail släppte patchversionen 15.0.6 och hotfixen 15.0.6.1 i juli 2026. Utöver åtgärdade sårbarheter i PDF-generering och PGP-bearbetning innehåller versionerna ett separat MFA-fält, LDAP-autentisering för admin-GUI:t samt korrigeringar i RuleEngine, webbmail och REST-API."
date: "2026-07-29"
kategorie: "SEPPmail"
timeToRead: "5 min. lästid"
themen:
  - seppmail
slug: "seppmail-15-0-6-och-15-0-6-1-sakerhetskorrigeringar-och-nya-adminfunktioner"
translationId: "article-3046fc35b259929b"
draft: false
translationOf: seppmail-releases-15-0-6-und-15-0-6-1
url: https://rafaelpfister.ch/sv/blog/seppmail-15-0-6-och-15-0-6-1-sakerhetskorrigeringar-och-nya-adminfunktioner
translationSourceHash: 5cf19b84bb90403b0a7e2795222b8f853c29c3fe562429df8538e703e565217a
translationModel: gpt-5.6-terra
translatedAt: 2026-07-31T06:26:53.960Z
translationReview: automatic
---

# SEPPmail 15.0.6 och 15.0.6.1: Säkerhetskorrigeringar och nya adminfunktioner

SEPPmail släppte patchversionen 15.0.6 den 21 juli 2026 och hotfixen 15.0.6.1 en dag senare. Patchversionen åtgärdar flera sårbarheter, uppdaterar OpenSSH och OpenSSL och medför märkbara förbättringar för administrationen. Hotfixen korrigerar två fel i RuleEngine som infördes eller blev synliga med 15.0.6. Ändringarna berör även appliances som används som HIN Mailgateway, eftersom de bygger på samma SEPPmail-firmware.

## Hotfix 15.0.6.1 från den 22 juli 2026

Hotfixen åtgärdar två punkter i RuleEngine. För det första förhindrade ett odefinierat värde i Message-objektet att loggposter skrevs till e-postloggen. Berörda meddelanden passerade därmed systemet utan loggning. För det andra identifierar RuleEngine nu riktningen för arkiverade e-postmeddelanden, så att deras leverans hanteras korrekt.

Den som redan har installerat 15.0.6 eller planerar uppdateringen bör gå direkt till 15.0.6.1.

HIN-appliances verkar också ha fått hotfixen: En HIN Mailgateway med installerad version 15.0.6-RC-42-g278c81f84 visar nu 15.0.6-RC-88-g916e513cc som nästa version i 15.0-grenen. RC-beteckningarna för HIN-firmware kan inte direkt kopplas till en SEPPmail-release, men tidpunkten för erbjudandet talar för hotfixen.

## Säkerhetskorrigeringar i 15.0.6

Den viktigaste delen av patchversionen är tre korrigeringar av säkerhetsarkitekturen:

- En möjlig Path-Traversal-sårbarhet i PDF-genereringen har stängts. Den upptäcktes av InfoGuard.
- Allt innehåll som dekrypteras via PGP Base64-kodas nu för att förhindra MIME-Structure-Injection.
- Funktionen hashencrypt har lagts om till AES-256-CBC med PBKDF2.

Därtill kommer uppdaterade bibliotek: OpenSSH 10.4 och OpenSSL 3.0.21 åtgärdar tillsammans över tjugo CVE:er. Bara av dessa skäl rekommenderas uppdateringen för produktiva system.

## Nya funktioner för administrationen

Tre ändringar i admin-GUI:t märks i vardagen:

- **Separat MFA-inmatningsfält:** Den andra faktorn behöver inte längre läggas till lösenordet utan har ett eget fält. Det eliminerar en mångårig fallgrop vid inloggning.
- **LDAP-autentisering för admin-GUI:t:** Administratörer kan nu autentisera sig mot en extern LDAP-server i stället för att underhålla lokala konton på appliancen. Konfigurationen beskrivs i artikeln om [anslutning av admin-GUI:t till Active Directory](/blog/seppmail-admin-gui-ldap-authentifizierung). Om HIN Mailgateway också har fått denna funktion testar jag fortfarande och kompletterar artikeln därefter; eftersom HIN använder samma firmwarebas utgår jag från det.
- **AutoRenew-knapp för MPKI:** I inställningarna för MPKI-connectorn kan den automatiska certifikatförnyelsen startas manuellt via «Trigger AutoRenew...» .

Dessutom använder appliancen nu konsekvent giltiga tidszoner (standard: Europe/Zurich), och System Object ID under System >> Advanced View valideras som en giltig OID.

## E-postbearbetning och webbmail

Fyra punkter har korrigerats i RuleEngine. Ämneshanteringen fungerar nu även vid okänd kodning. Meddelanden studsar om en signatur uttryckligen begärs men inte kan skapas; tidigare kunde sådana meddelanden fortsätta osignerade. Arkivkopior går nu via leveransfunktionen och får därmed ARC-rubriker. Och för PGP-meddelanden utan MDC-data ignoreras MDC-fel i stället för att störa bearbetningen.

I webbmailen (GINA) har fyra fel åtgärdats: Den automatiska raderingen av oregistrerade konton efter att respitperioden löpt ut fungerar igen, funktionen hashdecrypt gav i vissa fall ett falskt positivt dekrypteringsresultat, tillägg av en bilaga tömde fälten Till och CC, och tidsangivelsen i SMS-loggarna var felaktig.

## REST-API, kluster och säkerhetskopiering

REST-API:t får korrigeringar på flera slutpunkter: /system/ifaliasconfig (hantering av null-värden), /system/applySysconfig (åtkomstkonfiguration), /crypto/domain/{domainName} (uppladdning av domäncertifikat) samt GET och POST /ssl/csr. Timeouten för REST-anrop har höjts från 300 till 900 sekunder, vilket gör långvariga förfrågningar, såsom större konfigurationsändringar, mer tillförlitliga.

Vid klusterdrift blockerade en befintlig CARP-IP tidigare IP-inställningarna för en ny medlem; detta har åtgärdats. Före den dagliga skapandet av snapshots kontrollerar säkerhetskopieringen nu dessutom om databasen är skadad innan snapshoten skrivs.

## Koppling till inloggningsfelet i 15.0.5

Vid uppdatering av ett kluster till 15.0.5 kunde inloggningen sluta fungera på båda noderna. Felbilden och återställningen beskrivs i artikeln om [inloggningsfel efter uppdateringen till 15.0.5](/blog/hin-update-issue-version-15.0.5). Tillverkaren kände redan till problemet vid den tidpunkten och utlovade en korrigering i en kommande version.

I release notes för 15.0.6 finns nu exakt en post som matchar denna felbild: «prevent password rehashing when cluster members use different firmware versions». Under en klusteruppdatering kör noderna oundvikligen tillfälligt med olika firmwareversioner. Om en nod under denna fas beräknar lösenordshashar på nytt och replikerar dem till klustret passar hasharna inte längre på den andra versionen, och inloggningen misslyckas på båda noderna, precis som vid det då observerade felet. Release notes nämner inte uttryckligen inloggningsfelet, men posten täcker exakt den konstellation som utlöste det. Orsaken har därmed hanterats i 15.0.6; den nödprocedur med upplösning av klustret som krävdes i 15.0.5 bör inte längre behövas vid framtida uppdateringar.

## Mindre korrigeringar

I e-postloggen har datumssorteringen korrigerats, som tidigare sorterade alfabetiskt i stället för kronologiskt, och den visade storleken på LFT-meddelanden stämmer åter. Åtkomst till obefintliga X-rubriker loggas inte längre. CertCentral-connectorn för MPKI hanterar inmatnings- och REST-fel mer robust.

## Bedömning

De två RuleEngine-felen från hotfixen talar för att hoppa över 15.0.6 och använda 15.0.6.1 direkt. Skapa snapshots av båda noderna i kluster före uppdateringen och följ uppdateringsordningen i tillverkarens dokumentation. Inloggningsfelet i 15.0.5 visade varför denna förberedelse inte bara är en formalitet.

## Källor

1.  [SEPPmail-dokumentation – «Revision History»](https://docs.seppmail.com/ch/20_revision-history.html): Officiella release notes för 15.0.6 och 15.0.6.1 med alla enskilda punkter.

2.  [HIN Mailgateway 15.0.5: Åtgärda inloggningsfel efter klusteruppdateringen](/blog/hin-update-issue-version-15.0.5): Varför snapshots och korrekt uppdateringsordning i klustret är avgörande.
