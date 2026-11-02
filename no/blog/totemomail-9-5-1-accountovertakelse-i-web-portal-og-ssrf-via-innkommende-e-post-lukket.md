---
title: "totemomail 9.5.1: Accountovertakelse i Web Portal og SSRF via innkommende e-post lukket"
navTitle: "totemomail 9.5.1"
description: "totemomail 9.5.1 lukker åtte sikkerhetssårbarheter og bringer totemomail på nivå med Kiteworks-advisoryene for Email Protection Gateway som ble publisert i slutten av september. To sårbarheter kan utnyttes uten innlogging: en feil i tilbakestilling av passord i Web Portal som tillater overtakelse av administratorkontoer, og serversidige oppslag av URL-er fra innkommende meldinger. I tillegg kommer Path Traversal-rettinger i Admin API og Mail API samt oppdaterte Tomcat- og Batik-biblioteker."
date: "2026-10-31"
kategorie: "Totemomail"
timeToRead: "5 min. lesetid"
themen:
  - totemomail
  - sicherheitsluecken
produkte:
  - "totemomail"
  - "sicherheitsluecken"
protokolle:
  - "releases"
  - "haertung"
slug: "totemomail-9-5-1-accountovertakelse-i-web-portal-og-ssrf-via-innkommende-e-post-lukket"
translationId: "article-2bd15f63632ab728"
draft: false
translationOf: totemomail-9-5-1-sicherheitsupdate
url: https://rafaelpfister.ch/no/blog/totemomail-9-5-1-accountovertakelse-i-web-portal-og-ssrf-via-innkommende-e-post-lukket
translationSourceHash: 5d441bdc7ec39d7fcef423e826bf14772f11bca2aa3dbdf4e1e3c83297131635
translationModel: gpt-5.6-terra
translatedAt: 2026-10-31T19:47:40.074Z
translationReview: automatic
---

# totemomail 9.5.1: Accountovertakelse i Web Portal og SSRF via innkommende e-post lukket

totemo har publisert totemomail 9.5.1 (Build 101). Oppdateringen inneholder åtte sikkerhetsrettinger, én forbedring og tre feilrettinger. To av sårbarhetene kan utnyttes uten innlogging: en feil i tilbakestilling av passord i Web Portal, som også kan brukes til å overta administratorkontoer, og serversidige oppslag av URL-er som finnes i innkommende meldinger. Begge berører komponenter som i de fleste installasjoner er tilgjengelige fra Internett. Planlegg derfor oppdateringen snarlig.

## Direkte fra 9.2.1 til 9.5.1

Release Notes oppfører 9.5.1 rett etter 9.2.1; versjonene 9.3 og 9.4 forekommer ikke i totemomail-historikken. En oppføring i 9.5.1 viser likevel til en «security hardening introduced in 9.3.1». Mellomversjonene stammer fra Kiteworks Email Protection Gateway (EPG), som bygger på samme kodebase: Kiteworks' sikkerhetsadvarsler angir EPG 9.3.1, 9.4.0, 9.4.1, 9.5.0 og 9.5.1 som rettede versjoner. For totemomail-installasjoner med 9.2.1 er 9.5.1 neste oppdatering; Release Notes oppfører ingen ytterligere oppdatering imellom.

## Sårbarheter uten innlogging

**Server-Side Request Forgery via meldingsinnholdet.** totemomail hentet URL-er fra behandlede meldinger på serversiden uten å begrense målet tilstrekkelig. En avsender uten innlogging kunne utforme en melding slik at gatewayen sendte forespørsler til interne tjenester eller metadataendepunkter for skyinstanser og returnerte svarene. Det er tilstrekkelig med en e-post som gatewayen behandler; ved en gateway i innkommende e-postflyt kan enhver ekstern avsender gjøre dette. Dette er særlig kritisk ved installasjoner i Azure, AWS eller andre skyer, der metadataendepunktet `169.254.169.254` leverer konfigurasjonsdata og, avhengig av plattformen, tilgangstokener for instansen. Release Notes oppgir ikke noe CVE-nummer for denne sårbarheten. Kiteworks fører fem advisories for EPG om tilgang til interne nettverksressurser (CVE-2026-102095, CVE-2026-102102 til CVE-2026-102105, alle med CVSS 9.1, rettet i EPG 9.5.0); det fremgår ikke av de korte advisorytekstene hvilket av disse som tilsvarer totemomail-oppføringen.

**Tilbakestilling av passord i Web Portal (CVE-2026-85066).** En feil i prosessen for å tilbakestille passord gjorde det mulig å hoppe over kontrolltrinnene og sette et nytt passord for en eksisterende konto. Release Notes inkluderer uttrykkelig administratorkontoer. Kiteworks vurderer sårbarheten i det tilhørende advisoryet GHSA-rwpq-5xfv-54pv til CVSS 9.8 (kritisk) og oppgir EPG 9.5.0 som rettet versjon. Web Portal er beregnet på eksterne mottakere av krypterte meldinger og er derfor som regel offentlig tilgjengelig.

## Sårbarheter med administratorrettigheter

Tre ytterligere rettinger forutsetter innlogging med administratorrettigheter. I kombinasjon med accountovertakelsen i Web Portal får disse forutsetningene imidlertid mindre betydning.

- **Admin API, køadministrasjon (CVE-2026-34843):** Utilstrekkelig beskyttede filopplastinger gjorde det mulig å skrive filer til vilkårlige steder i filsystemet. Ifølge Release Notes var Remote Code Execution mulig på denne måten. Advisoryet GHSA-mhc9-4pph-2r9x vurderer sårbarheten til CVSS 7.2; i EPG har den vært rettet siden 9.4.0.
- **Rule Designer:** Ved import av en regelkonfigurasjon kunne behandlingspipelinen laste og kjøre kode utenfor de godkjente Mail Processing-komponentene. Dette forutsetter rettigheter til å konfigurere e-postregler.
- **Mail API:** Ved utpakking av opplastede ZIP-filer forelå det en Path Traversal-sårbarhet. Filer pakkes nå kun ut i den midlertidige katalogen.

Den fjerde rettingen gjelder egenskapen `security.login.requestingKeysAvailable`. Tidligere skjulte den bare lenken for å be om offentlige nøkler i Web Portal; selve funksjonen forble tilgjengelig gjennom en direkte forespørsel. Fra og med 9.5.1 deaktiverer egenskapen funksjonen. Hvis du har angitt egenskapen for å forhindre nøkkelforespørsler, virker den først som forventet med denne oppdateringen.

## Hva Kiteworks-registeret i tillegg lister opp

Kiteworks publiserte samlet 125 sikkerhetsadvarsler i sitt advisory-register på GitHub den 30. september 2026, hvorav 28 for Email Protection Gateway. totemomail Release Notes for 9.5.1 nevner langt færre punkter. Registeret skiller ikke mellom EPG og totemomail; det kan ikke leses ut derfra om hver EPG-sårbarhet også berører totemomail. De kritiske EPG-oppføringene er:

| CVE | CVSS | Konsekvens ifølge advisory | Rettet fra EPG |
|---|---|---|---|
| CVE-2026-54154 | 10.0 | Kjøring av kode med root-rettigheter uten innlogging | 9.4.1 |
| CVE-2026-85066 | 9.8 | Overtakelse av brukerkontoer (tilbakestilling av passord) | 9.5.0 |
| CVE-2026-85065 | 9.8 | Overtakelse av brukerkontoer | 9.5.0 |
| CVE-2026-102149 | 9.4 | Overtakelse av brukerkontoer | 9.5.1 |
| CVE-2026-102106 | 9.1 | Endring av administrasjonskonfigurasjonen | 9.5.0 |
| CVE-2026-102095, CVE-2026-102102 til 102105 | 9.1 | Tilgang til interne nettverksressurser | 9.5.0 |

I tillegg kommer 13 oppføringer klassifisert som høye, de fleste av dem kjøring av kode med administratorrettigheter (CVSS 7.2). Den høyeste oppgitte rettede versjonen er 9.5.1; Kiteworks beskrev 9.5.1 som den gjeldende versjonen i slutten av september, der alle kjente sårbarheter skulle være hensyntatt. Den som oppdaterer totemomail til 9.5.1, er dermed på nivået som alle EPG-advisoryene i registeret refererer til.

## Oppdaterte biblioteker

De medfølgende bibliotekene fra Apache Tomcat og Apache XML Graphics Batik er oppdatert. Release Notes oppgir CVE-2026-50229, CVE-2026-53404, CVE-2026-53434, CVE-2026-55276, CVE-2026-55955, CVE-2026-55956 og CVE-2026-59083, uten å tilordne dem individuelt til de to bibliotekene. I tillegg ble woodstox-core-asl oppdatert på grunn av en sårbarhet; det er ikke oppgitt noe CVE-nummer for dette.

## Forbedring og feilrettinger

- **EPG-migrering:** Ved migrering fra en Oracle-database støtter totemomail nå Oracle Wallet. Hvis det ikke er konfigurert noe databasepassord, kobler migreringen seg til via Wallet.
- **RMS:** Meldinger kunne bli hengende i behandlingen fordi RMS-handlinger ventet ubegrenset på autentisering. Ventetiden er nå begrenset til ett minutt. Dette berører installasjoner med Microsoft Purview Information Protection (RMS).
- **TRE:** Funksjonen «Open Online» for Registered Envelopes fungerer igjen. Den hadde sluttet å fungere siden den nevnte hardeningen i 9.3.1.
- **Lisensrapport:** For skylisenser teller rapporten igjen interne brukere med forhåndsdefinerte Feature-typer korrekt. Artikkelen om [lisensgrensen og opprydding via LDAP](/blog/totemomail-licensed-user-limit-ldap-cleanup) beskriver hvordan foreldreløse kontoer påvirker lisenstellingen.

## Installasjon

Oppdateringen kan installeres på systemer fra totemomail 7 b985 under Windows, Linux, Solaris og FreeBSD. Logg inn på administrasjonskonsollen, last opp den medfølgende TPF-filen under `[Instanzname]` > «Update», og start oppdateringen. Ved flere instanser må dette gjøres separat for hver instans. totemomail starter deretter på nytt automatisk der det er mulig; planlegg et vedlikeholdsvindu for dette og opprett en sikkerhetskopi eller et øyeblikksbilde på forhånd.

Inntil oppdateringen er installert, reduserer to tiltak risikoen: Gjør admin-konsollen og Admin API kun tilgjengelig fra administrasjonsnettet, og begrens gatewayens utgående forbindelser til de målene som faktisk er nødvendige. Ved skyinstallasjoner er metadataendepunktet `169.254.169.254` blant målene gatewayen ikke trenger å nå. Begge tiltakene erstatter ikke oppdateringen; sårbarheten i tilbakestillingen av passord i Web Portal lukkes bare av oppdateringen.

## Kilder

1.  totemomail Release Notes 9.5.1 (Build 101), totemo ag: Installasjonsveiledning og Release Notes som PDF, tilgjengelig for kunder via produsentens support; grunnlag for alle opplysninger i denne artikkelen.

2.  [Kiteworks Security Advisories på GitHub](https://github.com/kiteworks/security-advisories/security/advisories): Offisielt register med CVSS-verdier og rettede versjoner for Email Protection Gateway, Core, MFT og Advanced Forms.

3.  [GHSA-rwpq-5xfv-54pv: Account takeover (CVE-2026-85066)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-rwpq-5xfv-54pv): Advisory om feilen ved tilbakestilling av passord, CVSS 9.8, rettet i EPG 9.5.0.

4.  [GHSA-5xhq-9wq3-rvj6: Arbitrary code execution (CVE-2026-54154)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-5xhq-9wq3-rvj6): Den mest kritiske EPG-sårbarheten med CVSS 10.0, rettet i EPG 9.4.1.

5.  [P.K. Sharma: Kiteworks published 125 advisories](https://www.pk-sharma.com/briefing/kiteworks-125-advisories-which-is-the-shutdown-flaw): Evaluering av publiseringen fra 30. september 2026 med opptelling etter produkt og alvorlighetsgrad samt Kiteworks' uttalelser om 9.5.1.

6.  [Forstå e-postruting mellom totemomail og Exchange Online](/blog/totemomail-m365): Hvordan totemomail behandler innkommende meldinger; relevant for spørsmålet om hvilke e-poster som kan utløse SSRF-sårbarheten.

7.  [Totemomail-lisensgrense nådd: rydd opp i foreldreløse brukere via LDAP](/blog/totemomail-licensed-user-limit-ldap-cleanup): Bakgrunn for lisenstellingen som rettingen av lisensrapporten berører.
