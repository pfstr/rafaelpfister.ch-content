---
title: "SEPPmail 15.0.6 och 15.0.6.1: säkerhetskorrigeringar och nya adminfunktioner"
navTitle: "SEPPmail 15.0.6"
description: "SEPPmail släppte patchversionen 15.0.6 och snabbkorrigeringen 15.0.6.1 i juli 2026. Utöver åtgärdade sårbarheter i PDF-generering och PGP-hantering innehåller versionerna ett separat MFA-fält, LDAP-autentisering för admin-GUI:n samt korrigeringar i RuleEngine, Webmail och REST-API."
date: "2026-07-29"
kategorie: "SEPPmail"
timeToRead: "5 min lästid"
themen:
  - seppmail
slug: "seppmail-15-0-6-och-15-0-6-1-sakerhetskorrigeringar-och-nya-adminfunktioner"
translationId: "article-3046fc35b259929b"
draft: false
translationOf: seppmail-releases-15-0-6-und-15-0-6-1
translationSourceHash: 636a7246234584a2b5797f53239fe65129de0f4463b8f773d0a7d9ed06d61f91
translationModel: gpt-5.6-terra
translatedAt: 2026-09-03T08:15:57.947Z
translationReview: automatic
url: https://rafaelpfister.ch/sv/blog/seppmail-15-0-6-och-15-0-6-1-sakerhetskorrigeringar-och-nya-adminfunktioner
---

# SEPPmail 15.0.6 och 15.0.6.1: säkerhetskorrigeringar och nya adminfunktioner

SEPPmail släppte patchversionen 15.0.6 den 21 juli 2026 och snabbkorrigeringen 15.0.6.1 en dag senare. Patchversionen åtgärdar flera sårbarheter, uppdaterar OpenSSH och OpenSSL och innebär märkbara förbättringar för administrationen. Snabbkorrigeringen rättar två fel i RuleEngine som infördes eller blev synliga med 15.0.6. Ändringarna berör även appliances som drivs som HIN Mailgateway, eftersom de bygger på samma SEPPmail-firmware.

## Snabbkorrigering 15.0.6.1 från 22 juli 2026

Snabbkorrigeringen åtgärdar två punkter i RuleEngine. För det första förhindrade ett odefinierat värde i Message-objektet att loggposter skrevs till e-postloggen. Berörda meddelanden passerade därmed systemet utan loggning. För det andra identifierar RuleEngine nu riktningen för arkiverade e-postmeddelanden så att leveransen av dem hanteras korrekt.

Den som redan har installerat 15.0.6 eller planerar uppdateringen bör gå direkt till 15.0.6.1.

HIN-appliances har uppenbarligen också fått snabbkorrigeringen: En HIN Mailgateway med installerad version 15.0.6-RC-42-g278c81f84 visar nu 15.0.6-RC-88-g916e513cc som nästa version i 15.0-grenen. RC-beteckningarna i HIN-firmwaren kan inte direkt kopplas till en SEPPmail-version, men tidpunkten för erbjudandet talar för snabbkorrigeringen.

## Säkerhetskorrigeringar i 15.0.6

Den viktigaste delen av patchversionen är tre korrigeringar i säkerhetsarkitekturen:

- En möjlig Path-Traversal-sårbarhet i PDF-genereringen har stängts. Den upptäcktes av InfoGuard.
- Allt innehåll som dekrypteras via PGP Base64-kodas nu för att förhindra MIME-Structure-Injection.
- Funktionen hashencrypt har lagts om till AES-256-CBC med PBKDF2.

Dessutom har bibliotek uppdaterats: OpenSSH 10.4 och OpenSSL 3.0.21 åtgärdar tillsammans över tjugo CVE:er. Bara av dessa skäl rekommenderas uppdateringen för produktionssystem.

## Nya funktioner för administrationen

Tre ändringar i admin-GUI:n märks i vardagen:

- **Separat MFA-inmatningsfält:** Den andra faktorn behöver inte längre läggas till lösenordet utan har ett eget fält. Det eliminerar en långvarig felkälla vid inloggning.
- **LDAP-autentisering för admin-GUI:n:** Administratörer kan nu autentisera sig mot en extern LDAP-server i stället för att underhålla lokala konton på appliance-enheten. Konfigurationen beskrivs i artikeln om [anslutning av admin-GUI:n till Active Directory](/blog/seppmail-admin-gui-ldap-authentifizierung). Om HIN Mailgateway också har fått denna funktion testar jag fortfarande och kompletterar sedan artikeln; eftersom HIN använder samma firmware-bas utgår jag från det.
- **AutoRenew-knapp för MPKI:** I inställningarna för MPKI-Connector kan automatisk certifikatförnyelse startas manuellt via «Trigger AutoRenew...» .

Dessutom använder appliance-enheten nu konsekvent giltiga tidszoner (standard: Europe/Zurich), och System Object ID under System >> Advanced View valideras som en giltig OID.

## E-posthantering och Webmail

Fyra punkter har korrigerats i RuleEngine. Ämneshanteringen fungerar nu även vid okänd kodning. Meddelanden studsar om en signatur uttryckligen begärs men inte kan skapas; tidigare kunde sådana meddelanden fortsätta osignerade. Arkivkopior passerar nu leveransfunktionen och får därmed ARC-rubriker. Och för PGP-meddelanden utan MDC-data ignoreras MDC-fel i stället för att störa hanteringen.

I Webmail (GINA) har fyra fel åtgärdats: Automatisk radering av oregistrerade konton efter att respitperioden löpt ut fungerar igen, funktionen hashdecrypt gav i vissa fall ett falskt positivt dekrypteringsresultat, tillägg av en bilaga tömde fälten Till och CC, och tidsvisningen i SMS-loggarna var felaktig.

## REST-API, kluster och backup

REST-API:t får korrigeringar i flera ändpunkter: /system/ifaliasconfig (hantering av null-värden), /system/applySysconfig (åtkomstkonfiguration), /crypto/domain/{domainName} (uppladdning av domäncertifikat) samt GET och POST /ssl/csr. Timeouten för REST-anrop har höjts från 300 till 900 sekunder, vilket gör långvariga förfrågningar som större konfigurationsändringar mer tillförlitliga.

I klusterdrift blockerade en befintlig CARP-IP tidigare IP-inställningarna för en ny medlem; detta har åtgärdats. Före den dagliga snapshot-skapandet kontrollerar backupen nu dessutom om databasen är korrupt innan snapshoten skrivs.

## Koppling till inloggningsfelet under 15.0.5

Vid uppdatering av ett kluster till 15.0.5 kunde inloggningen sluta fungera på båda noderna. Felbilden och återställningen beskrivs i artikeln om [att åtgärda inloggningsfelet efter uppdateringen till 15.0.5](/blog/hin-update-issue-version-15.0.5). Tillverkaren kände redan då till problemet och utlovade en korrigering i en kommande version.

I Release Notes för 15.0.6 finns nu exakt en post som passar denna felbild: «prevent password rehashing when cluster members use different firmware versions». Under en klusteruppdatering kör noderna oundvikligen tillfälligt med olika firmware-versioner. Om en nod i denna fas beräknar om lösenordshashar och replikerar dem till klustret passar hasharna inte längre med den andra versionen, och inloggningen misslyckas på båda noderna, precis som vid det då observerade felet. Release Notes nämner inte uttryckligen inloggningsfelet, men posten täcker exakt den konstellation som utlöste det. Orsaken hanteras därmed i 15.0.6; den nödprocedur med upplösning av klustret som krävdes i 15.0.5 bör inte längre behövas vid framtida uppdateringar.

## Mindre korrigeringar

I e-postloggen har datumsorteringen korrigerats, som tidigare sorterade alfabetiskt i stället för kronologiskt, och den visade storleken på LFT-meddelanden stämmer åter. Åtkomst till obefintliga X-rubriker loggas inte längre. CertCentral-Connector för MPKI hanterar inmatnings- och REST-fel mer robust.

## Bedömning

De två RuleEngine-felen från snabbkorrigeringen talar för att hoppa över 15.0.6 och använda 15.0.6.1 direkt. Skapa snapshots av båda noderna i kluster före uppdateringen och följ uppdateringsordningen i tillverkarens dokumentation. Inloggningsfelet under 15.0.5 visade varför denna förberedelse inte bara är en formalitet.

## Källor

1.  [SEPPmail-dokumentation – «Revision History»](https://docs.seppmail.com/ch/20_revision-history.html): Officiella Release Notes för 15.0.6 och 15.0.6.1 med alla enskilda punkter.

2.  [HIN Mailgateway 15.0.5: Åtgärda inloggningsfel efter klusteruppdateringen](/blog/hin-update-issue-version-15.0.5): Varför snapshots och rätt uppdateringsordning i klustret är avgörande.
