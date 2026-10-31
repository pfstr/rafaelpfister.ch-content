---
title: "Paperless-ngx 3.4.0: OIDC-innlogging uten dobbel 2FA, miniatyrbilder uten ImageMagick"
navTitle: "Paperless-ngx 3.4.0"
description: "Paperless-ngx 3.4.0 hopper over lokal 2FA når OIDC-leverandøren rapporterer en pålitelig andre faktor, genererer PDF-miniatyrbilder med Poppler i stedet for ImageMagick og oppdaterer flere avhengigheter av sikkerhetshensyn. Kontroller den nye AMR-innstillingen før oppdateringen, og på Bare Metal-systemer at pdftoppm er installert."
date: "2026-10-30"
kategorie: "Paperless-ngx"
timeToRead: "5 min lesetid"
themen:
  - paperless-ngx
produkte:
  - "paperless-ngx"
protokolle:
  - "releases"
slug: "paperless-ngx-3-4-0-oidc-innlogging-uten-dobbel-2fa-miniatyrbilder-uten-imagemagick"
translationId: article-57582656ad9c441b
translationOf: paperless-ngx-3-4-0
url: https://rafaelpfister.ch/no/blog/paperless-ngx-3-4-0-oidc-innlogging-uten-dobbel-2fa-miniatyrbilder-uten-imagemagick
translationSourceHash: fe1a5dd3d088e7e544b9448ecc7099233f8287856a5eded7af7488e06c8f290a
translationModel: gpt-5.6-terra
translatedAt: 2026-10-31T03:20:24.900Z
translationReview: automatic
---

# Paperless-ngx 3.4.0: OIDC-innlogging uten dobbel 2FA, miniatyrbilder uten ImageMagick

Paperless-ngx-prosjektet lanserte versjon 3.4.0 29. oktober 2026, én uke etter 3.3.4. Funksjonsutgivelsen introduserer en innstilling som fjerner den lokale tofaktorforespørselen ved OIDC-innlogginger dersom identitetsleverandøren allerede har bekreftet en andre faktor. Paperless-ngx genererer nå miniatyrbilder for PDF-dokumenter med Poppler-verktøyene, og ImageMagick er ikke lenger en avhengighet. I tillegg kommer utvidelser for arbeidsflyter og Custom Fields, endringer i opplasting og varsler i webgrensesnittet, åtte feilrettinger samt oppdateringer av frontend- og Python-avhengigheter som prosjektet fører som sikkerhetsoppdateringer. Utgivelsen nevner ingen egen sikkerhetsmelding med CVE eller GHSA. Den er rettet mot alle som drifter Paperless-ngx 3.x, særlig installasjoner med OIDC-innlogging og Bare Metal-installasjoner der du selv administrerer systempakkene.

## Versjonene i oversikt

| Versjon | Dato | Type | Hovedpunkter |
|---|---|---|---|
| 3.4.0 | 29.10.2026 | Funksjonsutgivelse | hopp over lokal 2FA ved pålitelig OIDC-AMR, PDF-miniatyrbilder via Poppler, ImageMagick fjernet, Custom Fields i arbeidsflyttitler, sikkerhetsoppdateringer av avhengigheter |

## OIDC-innlogging: lokal 2FA ved pålitelig AMR

Ved innlogging via en OIDC-leverandør evaluerer Paperless-ngx nå `amr`-claimen i ID-tokenet (PR #14403). Dersom den inneholder én av verdiene fra `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`, hopper Paperless-ngx over sin egen tofaktorforespørsel for denne innloggingen. I henhold til konfigurasjonsdokumentasjonen er standardverdien for variabelen `mfa`. Dokumentasjonen nevner Authelia, som rapporterer `mfa`, og Pocket ID, som rapporterer `phr` ved passnøkler, som eksempler. Pocket ID sender verdien `otp` for engangskoder via e-post; dokumentasjonen vurderer denne som upålitelig.

```shell
PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR=mfa
```

Siden standardverdien allerede er satt, endres oppførselen etter oppdateringen for brukere med lokalt konfigurert 2FA så snart identitetsleverandøren deres leverer `mfa` i `amr`-claimen. Kontroller derfor hvilke AMR-verdier leverandøren faktisk sender, og om disse verdiene bare angis ved en reell andre faktor. Dokumentasjonen krever uttrykkelig at du kun angir verdier som betyr at en andre faktor faktisk ble brukt ved innloggingen. Samtidig oppdaterer 3.4.0 django-allauth til 65.20 og oauthlib til 4.0 (PR #14570).

## PDF-miniatyrbilder med Poppler, ImageMagick fjernet

Tidligere kalte Paperless-ngx ImageMagick (`convert -density 300`) for miniatyrbildet av en PDF, som rendret første side via Ghostscript med 300 DPI. Fra og med 3.4.0 renderer `pdftoppm` fra Poppler-verktøyene første side, Pillow skalerer bildet og lagrer det som WebP (PR #14420). Paperless-ngx beregner oppløsningen fra sidestørrelsen, som det henter med `pdfinfo`, med maksimalt 300 DPI. Ghostscript-reserven for miniatyrbilder faller bort, mens reparasjon med qpdf beholdes. Forfatteren av PR-en har målt betydelig kortere kjøretider og lavere minnebruk, for et tomt A0-dokument omtrent 0,21 i stedet for 12,9 sekunder; verdiene er ikke uavhengig kontrollert.

Med PR #14440 faller ImageMagick helt bort. De gjenværende oppgavene, i hovedsak fjerning av en alfakanal fra bilder, håndteres av en hjelper basert på Pillow; WebP-filer lagres tapsfritt. ImageMagick-policyen som tidligere måtte tilpasses for PDF-behandling, er dermed ikke lenger nødvendig. Ghostscript forblir installert fordi OCRmyPDF fortsatt trenger det.

For administratorer har dette to følger:

- **Bare Metal:** `pdftoppm` må være tilgjengelig. Hvis det mangler, viser Paperless-ngx en advarsel ved oppstart. Sammenlign de installerte systempakkene med den oppdaterte avhengighetslisten i installasjonsdokumentasjonen.
- **Konfigurasjon:** `PAPERLESS_CONVERT_MEMORY_LIMIT` og `PAPERLESS_CONVERT_TMPDIR` er ifølge PR #14420 foreldet og uten virkning, og `PAPERLESS_GS_BINARY` er også merket som foreldet. Du kan fjerne disse oppføringene fra `paperless.conf` eller `.env`-filen.

## Arbeidsflyter, Custom Fields og webgrensesnitt

- **Arbeidsflyter:** Tittelmaler i arbeidsflyter kan bruke Custom Fields (PR #14564). I taggfeltene for arbeidsflyter kan over- og underordnede tagger velges uavhengig av hverandre (PR #14400).
- **Custom Fields:** Spørringer mot datofelt kan gjøres relativt (PR #14549).
- **Opplasting og varsler:** Navigasjonslinjen har en opplastingsknapp (PR #14442). Knappen viser antallet pågående opplastinger og peker til oversikten, mens antallet varsler reduseres (PR #14501, #14500).
- **Grensesnitt:** Dokumentdetaljene er revidert (PR #14485), oppgavemerket lastes raskere (PR #14552), og KI-chatten gjengir Markdown (PR #14409, #14415).

## Feilrettinger

- `document_exporter` avbrøt ved eksport til en mappe med `--delete` dersom en foreldet `manifest.json.tmp` fantes (PR #14526). De som bruker eksportøren med `--delete` til sikkerhetskopier, bør etter oppdateringen kontrollere at eksportene igjen fullføres.
- Ved redigering av et dokument oppdateres søkeindeksen først etter arbeidsflytkjøringene (PR #14558).
- Dersom en strekkode ikke gir dokumentseparasjon, blir tagger ikke lenger tildelt forsinket (PR #14497).
- Avbrutte eller utløpte oppgaver vises mer forståelig (PR #14566).
- Skjemavisningen for API-et utløser ikke lenger tillatelsesspørringer mot dokumenter (PR #14503), og listetypen for filtrene for dokumentobjekter i API-skjemaet er korrigert (PR #14524).
- Lange versjonsbetegnelser forkortes i nedtrekksmenyen, og størrelsen på nedtrekksmenyene er standardisert (PR #14578, #14510).

## Avhengigheter og sikkerhet

Release Notes fører oppdateringer av frontend-avhengigheter (PR #14581) samt av brotli, djangorestframework, filelock, msgpack og setuptools (PR #14575) som sikkerhetsoppdateringer. Release Notes oppgir ikke hvilke sårbarheter i disse pakkene som lukkes. Angular-komponentene er på 22.2.2. Prosjektet har ingen egen Security Advisory for 3.4.0 (per 30. oktober 2026).

Den nyeste adviseringen GHSA-c8f3-pqr2-rr24 fra 16. oktober 2026 gjelder versjonene 3.0 til før 3.3.2 og er rettet i 3.3.2. Med aktiverte KI-funksjoner kunne KI-klassifiseringsarbeidsflyter for dokumenter uten eier overføre utdrag og metadata fra andre brukeres private dokumenter til den konfigurerte KI-leverandøren (CVSS 3.1: 4.3, ingen CVE). Systemer som fortsatt kjører en eldre 3.x-versjon, lukker også dette sikkerhetshullet med oppdateringen til 3.4.0.

## Anbefalt fremgangsmåte

Målversjonen er 3.4.0. Stopp pågående behandlinger som dokumentinntak før oppdateringen, og opprett en sikkerhetskopi slik administrasjonsdokumentasjonen krever. For Docker-installasjoner med image fra Docker Hub stopper du stakken med `docker compose down`, laster ned det nye imaget med `docker compose pull` og starter med `docker compose up`, der databasemigreringer kjøres automatisk; når alt fungerer, starter du stakken med `-d` i bakgrunnen. For Bare Metal-installasjoner installerer du først de nye systemavhengighetene, spesielt Poppler med `pdftoppm`, oppdaterer Python-pakkene med `pip install -r requirements.txt` og kjører migreringene i katalogen `src` med `python3 manage.py migrate`. Før første OIDC-innlogging etter oppdateringen må du bestemme hvilke verdier som skal stå i `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`. En Docker-installasjon med eksternt dokumentlager beskrives i artikkelen [Drift av Paperless-ngx med lite lagringsplass](/blog/paperless-dokumente-clouddienst-auslagern). Release Notes for 3.4.0 nevner ingen kjente problemer eller brytende endringer (per 30. oktober 2026).

## Kilder

1.  [GitHub: Paperless-ngx Release v3.4.0](https://github.com/paperless-ngx/paperless-ngx/releases/tag/v3.4.0): Release Notes med dato, nye funksjoner, feilrettinger og avhengigheter.

2.  [Paperless-ngx: Endringslogg](https://github.com/paperless-ngx/paperless-ngx/blob/main/docs/changelog.md): Endringslogg i prosjektdokumentasjonen med oppføringene for 3.3.x og 3.4.0.

3.  [GitHub: PR #14403, hopp over lokal 2FA for pålitelig OIDC AMR](https://github.com/paperless-ngx/paperless-ngx/pull/14403): Ny evaluering av AMR-claimen ved OIDC-innlogging.

4.  [Paperless-ngx: Konfigurasjonsdokumentasjon](https://github.com/paperless-ngx/paperless-ngx/blob/main/docs/configuration.md): Beskrivelse og standardverdi for `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`.

5.  [GitHub: PR #14420, PDF-miniatyrbilder via Poppler](https://github.com/paperless-ngx/paperless-ngx/pull/14420): Overgang til `pdftoppm`, foreldede innstillinger og forfatterens måleverdier.

6.  [GitHub: PR #14440, ImageMagick fjernet som avhengighet](https://github.com/paperless-ngx/paperless-ngx/pull/14440): Erstatning av de gjenværende ImageMagick-kallene med Pillow.

7.  [GitHub: Security Advisory GHSA-c8f3-pqr2-rr24](https://github.com/paperless-ngx/paperless-ngx/security/advisories/GHSA-c8f3-pqr2-rr24): Informasjonslekkasje i KI-klassifiseringsarbeidsflyter, berørt 3.0 til før 3.3.2, CVSS 4.3.

8.  [Paperless-ngx: Administrasjonsdokumentasjon](https://github.com/paperless-ngx/paperless-ngx/blob/main/docs/administration.md): Oppdateringsprosedyre for Docker og Bare Metal, eksportørvalg `--delete`.

9.  [GitHub API: Paperless-ngx Releases](https://api.github.com/repos/paperless-ngx/paperless-ngx/releases): Publiseringsdatoer for 3.3.4 og 3.4.0.
