---
title: "Kiteworks EPG 9.5.1: Fyra offentligt dokumenterade säkerhetskorrigeringar"
navTitle: "Kiteworks EPG 9.5.1"
description: "Kiteworks Email Protection Gateway 9.5.1 åtgärdar fyra offentligt dokumenterade sårbarheter, däribland ett kontoövertagande med CVSS 9.4. Installationer före 9.5.1 bör uppdateras till denna eller en senare version."
date: "2026-11-06"
kategorie: "Totemomail"
timeToRead: "3 min. lästid"
themen:
  - totemomail
produkte:
  - "totemomail"
protokolle:
  - "releases"
slug: "kiteworks-epg-9-5-1-fyra-offentligt-dokumenterade-sakerhetskorrigeringar"
translationId: article-d4b9d984c72a54ea
translationOf: kiteworks-epg-releases-9-5-1
url: https://rafaelpfister.ch/sv/blog/kiteworks-epg-9-5-1-fyra-offentligt-dokumenterade-sakerhetskorrigeringar
translationSourceHash: 7d961f479006e9eb0da1a74925d77e25e7205a64a1ed783dccb55b6eed8c2c5c
translationModel: gpt-5.6-terra
translatedAt: 2026-11-07T03:19:55.047Z
translationReview: automatic
---

# Kiteworks EPG 9.5.1: Fyra offentligt dokumenterade säkerhetskorrigeringar

Kiteworks fortsätter den tidigare totemo-tekniken som Email Protection Gateway (EPG). Version 9.5.1 var senast den 25 september 2026 den aktuella Kiteworks-versionen; tillverkaren anger inget offentligt publiceringsdatum. Tillverkarens advisories, som publicerades den 30 september, dokumenterar fyra korrigeringar för EPG 9.5.1: ett kontoövertagande, två oautentiserade fel för att skriva filer respektive utlösa en Denial of Service samt ett fel som tillät administratörer att skriva godtyckliga filer. Klassificeringen för totemomail-versionen med samma namn finns i artikeln om [totemomail 9.5.1](/blog/totemomail-9-5-1-sicherheitsupdate).

## Versionerna i översikt

| Version | Datum | Typ | Huvudpunkter |
|---|---|---|---|
| 9.5.1 | offentligt bekräftad senast 25.09.2026 | säkerhetsuppdatering | Fyra offentligt dokumenterade EPG-korrigeringar, däribland CVE-2026-102149 med CVSS 9.4 |

## Säkerhetskorrigeringar i 9.5.1

Enligt Kiteworks berör alla fyra advisories samtliga EPG-versioner före 9.5.1 och anger 9.5.1 som korrigerad version. Advisories publicerades den 30 september 2026. De beskriver i efterhand säkerhetsstatusen för releasen, inte dess exakta publiceringsdatum.

- **CVE-2026-102149, CVSS 9.4 (kritisk):** En svaghet i åtkomstkontrollen kunde ge en angripare åtkomst till användarkonton via nätverket. CVSS-bedömningen förutsätter varken behörigheter eller användarinteraktion. Kiteworks klassificerar orsaken som CWE-306 (saknad autentisering för kritiska funktioner) och CWE-639 (kringgående av auktorisering via en användarkontrollerad nyckel).
- **CVE-2026-102143, CVSS 7.5 (hög):** En oautentiserad angripare kunde skriva filer till appliance-enheten. Kiteworks nämner ett autentiseringsproblem samt CWE-306 och CWE-434 (obegränsad uppladdning av farliga filtyper).
- **CVE-2026-102146, CVSS 6.5 (medel):** En administratör kunde skriva godtyckliga filer. Kiteworks beskriver ett behörighetsfel; CVSS-måttet kräver höga behörigheter. Tillverkaren anger CWE-73 (extern kontroll av filnamn eller sökväg) och CWE-1336 (otillräcklig neutralisering i en Template Engine) som svagheter.
- **CVE-2026-102144, CVSS 5.3 (medel):** En angripare i nätverket kunde utlösa en Denial of Service. Advisoret nämner saknad autentisering för kritiska funktioner och okontrollerad resursförbrukning (CWE-306, CWE-400). Inte heller denna bedömning förutsätter behörigheter eller användarinteraktion.

För operatörer är kontoövertagandet den prioriterade korrigeringen, eftersom det enligt advisoret är möjligt utan ett befintligt konto. De två andra felen utan inloggning berör också alla versioner före 9.5.1. Tillverkarens advisories publicerar inga tekniska detaljer om slutpunkter, konfigurationer eller tillfälliga lösningar.

## Rekommenderat tillvägagångssätt

Uppdatera EPG till minst 9.5.1 om en äldre version används. Kiteworks anger uttryckligen uppgradering till 9.5.1 eller senare som åtgärd för alla fyra advisories. Om en senare tillverkarversion redan har godkänts uppfyller den också denna minimiversion. Kiteworks har inte publicerat någon offentlig uppdateringsväg, kända problem eller supportfrister för EPG 9.5.1 i de öppnade källorna.

## Källor

1.  [Kiteworks: Precautionary Shutdown Advisory for Customers](https://www.kiteworks.com/company/press-releases/kiteworks-precautionary-shutdown-advisory/): Tillverkarmeddelande från den 25 september 2026 som anger 9.5.1 som aktuell version.

2.  [Kiteworks Security Advisories](https://github.com/kiteworks/security-advisories/security/advisories): Officiell översikt över advisories som publicerades den 30 september 2026, inklusive EPG-posterna för 9.5.1.

3.  [GHSA-c9w5-4frw-7wqq: Account takeover (CVE-2026-102149)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-c9w5-4frw-7wqq): Kritisk svaghet i åtkomstkontrollen, CVSS 9.4, åtgärdad i EPG 9.5.1.

4.  [GHSA-3p9g-jh62-8f89: Unauthorized file modification (CVE-2026-102143)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-3p9g-jh62-8f89): Oautentiserad skrivning av filer, CVSS 7.5, åtgärdad i EPG 9.5.1.

5.  [GHSA-5pgq-v8g2-rg2f: Unauthorized file modification (CVE-2026-102146)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-5pgq-v8g2-rg2f): Skrivning av godtyckliga filer med administratörsbehörighet, CVSS 6.5, åtgärdad i EPG 9.5.1.

6.  [GHSA-wwhf-5862-rjxq: Denial of service (CVE-2026-102144)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-wwhf-5862-rjxq): Oautentiserad Denial of Service, CVSS 5.3, åtgärdad i EPG 9.5.1.

7.  [Kiteworks: EPG with totemo Integration](https://www.kiteworks.com/video-why-kiteworks-and-totemo-are-better-together/): Tillverkaruppgift om att totemo-tekniken fortsätter och byter namn till Email Protection Gateway.
