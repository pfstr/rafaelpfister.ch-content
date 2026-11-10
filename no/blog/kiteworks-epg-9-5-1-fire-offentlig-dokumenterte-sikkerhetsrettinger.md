---
title: "Kiteworks EPG 9.5.1: Fire offentlig dokumenterte sikkerhetsrettinger"
navTitle: "Kiteworks EPG 9.5.1"
description: "Kiteworks Email Protection Gateway 9.5.1 retter fire offentlig dokumenterte sårbarheter, inkludert en konto-overtakelse med CVSS 9.4. Installasjoner før 9.5.1 bør oppdateres til denne eller en senere versjon."
date: "2026-11-06"
kategorie: "Totemomail"
timeToRead: "3 min lesetid"
themen:
  - totemomail
produkte:
  - "totemomail"
protokolle:
  - "releases"
slug: "kiteworks-epg-9-5-1-fire-offentlig-dokumenterte-sikkerhetsrettinger"
translationId: article-d4b9d984c72a54ea
translationOf: kiteworks-epg-releases-9-5-1
url: https://rafaelpfister.ch/no/blog/kiteworks-epg-9-5-1-fire-offentlig-dokumenterte-sikkerhetsrettinger
translationSourceHash: 7d961f479006e9eb0da1a74925d77e25e7205a64a1ed783dccb55b6eed8c2c5c
translationModel: gpt-5.6-terra
translatedAt: 2026-11-07T03:20:05.800Z
translationReview: automatic
---

# Kiteworks EPG 9.5.1: Fire offentlig dokumenterte sikkerhetsrettinger

Kiteworks viderefører den tidligere totemo-teknologien som Email Protection Gateway (EPG). Versjon 9.5.1 var senest 25. september 2026 den gjeldende Kiteworks-versjonen; produsenten oppgir ingen offentlig publiseringsdato. Produsentens advisories, publisert 30. september, dokumenterer fire rettinger i EPG 9.5.1: en konto-overtakelse, to uautentiserte feil for henholdsvis skriving av filer og utløsing av tjenestenekt, samt en feil som gjorde det mulig for administratorer å skrive vilkårlige filer. Vurderingen for totemomail-versjonen med samme navn finnes i artikkelen om [totemomail 9.5.1](/blog/totemomail-9-5-1-sicherheitsupdate).

## Versjonene i oversikt

| Versjon | Dato | Type | Hovedpunkter |
|---|---|---|---|
| 9.5.1 | offentlig bekreftet senest 25.09.2026 | sikkerhetsoppdatering | Fire offentlig dokumenterte EPG-rettinger, inkludert CVE-2026-102149 med CVSS 9.4 |

## Sikkerhetsrettinger i 9.5.1

Alle de fire advisoryene gjelder ifølge Kiteworks alle EPG-versjoner før 9.5.1 og oppgir 9.5.1 som rettet versjon. Advisoryene ble publisert 30. september 2026. De beskriver sikkerhetsstatusen for utgivelsen i ettertid, ikke den nøyaktige publiseringsdatoen.

- **CVE-2026-102149, CVSS 9.4 (kritisk):** En svakhet i tilgangskontrollen kunne gi en angriper tilgang til brukerkontoer via nettverket. CVSS-vurderingen forutsetter verken rettigheter eller brukerinteraksjon. Kiteworks klassifiserer årsaken som CWE-306 (manglende autentisering for kritiske funksjoner) og CWE-639 (omgåelse av autorisering via en brukerkontrollert nøkkel).
- **CVE-2026-102143, CVSS 7.5 (høy):** En uautentisert angriper kunne skrive filer til apparatet. Kiteworks oppgir et autentiseringsproblem samt CWE-306 og CWE-434 (ubegrenset opplasting av farlige filtyper).
- **CVE-2026-102146, CVSS 6.5 (middels):** En administrator kunne skrive vilkårlige filer. Kiteworks beskriver en rettighetsfeil; CVSS-metrikken krever høye rettigheter. Produsenten oppgir CWE-73 (ekstern kontroll av filnavn eller sti) og CWE-1336 (utilstrekkelig nøytralisering i en malmotor) som svakheter.
- **CVE-2026-102144, CVSS 5.3 (middels):** En angriper i nettverket kunne utløse tjenestenekt. Advisoryet oppgir manglende autentisering for kritiske funksjoner og ukontrollert ressursforbruk (CWE-306, CWE-400). Denne vurderingen forutsetter heller ikke rettigheter eller brukerinteraksjon.

For driftsansvarlige er konto-overtakelsen den viktigste rettingen, fordi den ifølge advisoryet er mulig uten en eksisterende konto. De to øvrige feilene uten innlogging gjelder også alle versjoner før 9.5.1. Produsentens advisories publiserer ingen tekniske detaljer om endepunkter, konfigurasjoner eller midlertidige løsninger.

## Anbefalt fremgangsmåte

Oppgrader EPG til minst 9.5.1 dersom en eldre versjon brukes. Kiteworks oppgir uttrykkelig oppgradering til 9.5.1 eller nyere som tiltak i alle fire advisoryene. Dersom en nyere produsentversjon allerede er frigitt, oppfyller den også denne minimumsversjonen. Kiteworks har ikke publisert en offentlig oppgraderingssti, kjente problemer eller støttefrister for EPG 9.5.1 i de åpne kildene.

## Kilder

1.  [Kiteworks: Precautionary Shutdown Advisory for Customers](https://www.kiteworks.com/company/press-releases/kiteworks-precautionary-shutdown-advisory/): Produsentmelding fra 25. september 2026 som oppgir 9.5.1 som gjeldende versjon.

2.  [Kiteworks Security Advisories](https://github.com/kiteworks/security-advisories/security/advisories): Offisiell oversikt over advisoryene publisert 30. september 2026, inkludert EPG-oppføringene for 9.5.1.

3.  [GHSA-c9w5-4frw-7wqq: Account takeover (CVE-2026-102149)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-c9w5-4frw-7wqq): Kritisk svakhet i tilgangskontroll, CVSS 9.4, rettet i EPG 9.5.1.

4.  [GHSA-3p9g-jh62-8f89: Unauthorized file modification (CVE-2026-102143)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-3p9g-jh62-8f89): Uautentisert skriving av filer, CVSS 7.5, rettet i EPG 9.5.1.

5.  [GHSA-5pgq-v8g2-rg2f: Unauthorized file modification (CVE-2026-102146)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-5pgq-v8g2-rg2f): Skriving av vilkårlige filer med administratorrettigheter, CVSS 6.5, rettet i EPG 9.5.1.

6.  [GHSA-wwhf-5862-rjxq: Denial of service (CVE-2026-102144)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-wwhf-5862-rjxq): Uautentisert tjenestenekt, CVSS 5.3, rettet i EPG 9.5.1.

7.  [Kiteworks: EPG with totemo Integration](https://www.kiteworks.com/video-why-kiteworks-and-totemo-are-better-together/): Produsentopplysning om videreføringen og omdøpingen av totemo-teknologien til Email Protection Gateway.
