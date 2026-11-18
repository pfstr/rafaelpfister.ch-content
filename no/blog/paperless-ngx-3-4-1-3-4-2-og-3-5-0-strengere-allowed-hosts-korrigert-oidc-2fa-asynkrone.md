---
title: "Paperless-ngx 3.4.1, 3.4.2 og 3.5.0: strengere ALLOWED_HOSTS, korrigert OIDC-2FA, asynkrone bulkoperasjoner"
navTitle: "Paperless-ngx 3.5.0"
description: "Paperless-ngx 3.5.0 tillater, når PAPERLESS_URL er angitt, bare vertsnavnet derfra og localhost, retter den virkningsløse AMR-standardverdien fra 3.4.x og kjører bulkoperasjoner asynkront. Kontroller før oppdateringen hvilke vertsnavn Paperless-ngx nås via, og legg dem til i PAPERLESS_ALLOWED_HOSTS."
date: "2026-11-17"
kategorie: "Paperless-ngx"
timeToRead: "7 min lesetid"
themen:
  - paperless-ngx
produkte:
  - "paperless-ngx"
protokolle:
  - "releases"
slug: "paperless-ngx-3-4-1-3-4-2-og-3-5-0-strengere-allowed-hosts-korrigert-oidc-2fa-asynkrone"
translationId: article-4c816dd3cd08ff6e
translationOf: paperless-ngx-3-5-0
url: https://rafaelpfister.ch/no/blog/paperless-ngx-3-4-1-3-4-2-og-3-5-0-strengere-allowed-hosts-korrigert-oidc-2fa-asynkrone
translationSourceHash: 2c1c3db7502c455d8090a922dd17654a159d2d094be4f29d0da51f29eababfb4
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T03:23:54.228Z
translationReview: required
---

# Paperless-ngx 3.4.1, 3.4.2 og 3.5.0: strengere ALLOWED_HOSTS, korrigert OIDC-2FA, asynkrone bulkoperasjoner

Siden 3.4.0 har Paperless-ngx-prosjektet utgitt tre versjoner: feilrettelsesutgivelsene 3.4.1 31. oktober 2026 og 3.4.2 3. november 2026, samt funksjonsutgivelsen 3.5.0 17. november 2026. De to patchversjonene retter hovedsakelig visningsfeil i det reviderte webgrensesnittet og oppdaterer Django og andre avhengigheter. 3.5.0 endrer standardverdien for tillatte vertsnavn, retter en feil som gjorde at innstillingen `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR` som ble introdusert i 3.4.0, ikke hadde noen virkning, kjører alle bulkoperasjoner asynkront, rydder automatisk opp i utløpte økter og legger til arbeidsflyter som kan utløses manuelt. Ingen av de tre utgivelsene nevner en sikkerhetsmelding med CVE eller GHSA. Den forrige statusen er beskrevet i artikkelen [Paperless-ngx 3.4.0: OIDC-innlogging uten dobbel 2FA, miniatyrbilder uten ImageMagick](/blog/paperless-ngx-3-4-0). Oppdateringene gjelder alle operatører av Paperless-ngx 3.x; særlig berørt er installasjoner som nås via interne vertsnavn, og installasjoner med OIDC-innlogging.

## Versjonene i oversikt

| Versjon | Dato | Type | Hovedpunkter |
|---|---|---|---|
| 3.4.1 | 31.10.2026 | Feilrettelsesutgivelse | Rettelser i API-skjemaet og webgrensesnittet, lavere minnebruk i hvilemodus |
| 3.4.2 | 03.11.2026 | Feilrettelsesutgivelse | Bulkredigeringsoppgaver i Celery-arbeidere, installasjon av Tesseract-språk før indeksbygging, filelock 4.2.6, oppdateringer av Django og frontend-avhengigheter |
| 3.5.0 | 17.11.2026 | Funksjonsutgivelse | ALLOWED_HOSTS uten wildcard når PAPERLESS_URL er angitt, korrigert AMR-standardverdi, asynkrone bulkoperasjoner, automatisk opprydding av økter, manuell arbeidsflytutløser, tagghierarki uten django-treenode |

## 3.4.1 og 3.4.2: Rettelser etter funksjonsutgivelsen

3.4.1 retter plasseringen av en skjemakorrektur for `id`-felt (PR #14604), fjerner doble tilgangskontroller på åpningsknappen i dokumentkortene (PR #14601), skjuler i stor grad de nye PDF-kontrollene (PR #14595), flytter forhåndsvisningsknappen i dokumentlisten (PR #14594) og retter visningen av kontrollsummen i lys modus samt plasseringen av nedtrekksmenyen for egendefinerte felt på mobile enheter (PR #14590). I tillegg laster Paperless-ngx flere moduler først ved behov, noe som reduserer minnebruken i hvilemodus (PR #14599).

3.4.2 inneholder elleve feilrettinger. Følgende er relevante for drift:

- Importoppgaver fra `bulk_edit` kjører trygt i Celery-arbeiderne (PR #14653).
- Byggingen av søkeindeksen og LLM-indeksen venter til Tesseract-språkpakker er installert (PR #14670).
- filelock er på 4.2.6; utgivelsesnotatene begrunner oppdateringen med rettelser knyttet til ACL-er (PR #14656).
- Dokumenthistorikken viser taggnavn i stedet for ID-er (PR #14625), zoomknapper sender ikke lenger inn skjemaer (PR #14624), og navnet på en lagret visning beholdes ved lagring og lukking (PR #14667).

Blant avhengighetene oppdaterer 3.4.2 Django, django-guardian, django-filter og drf-spectacular-sidecar (PR #14661), turbohtml til 1.15.3 (PR #14662) og pdfjs-dist til 6.4.299 (PR #14644). Prosjektet erstatter det ikke lenger vedlikeholdte biblioteket ngx-bootstrap-icons med en egen løsning (PR #14665). I eksempel-Compose-filene oppgraderes Gotenberg til 8.38 (PR #14634) og nginx til 1.31.7 (PR #14633). Utgivelsesnotatene for 3.4.1 og 3.4.2 klassifiserer ingen av disse oppføringene som sikkerhetsoppdateringer.

## 3.5.0: ALLOWED_HOSTS uten wildcard

Utgivelsesnotatene for 3.5.0 starter med en merknad om at enkelte brukere må tilpasse innstillingen `PAPERLESS_ALLOWED_HOSTS` (PR #14735). Frem til 3.4.2 dokumenterte prosjektet standardverdien `*` for `PAPERLESS_ALLOWED_HOSTS`, altså alle verter. Fra 3.5.0 gjelder ifølge konfigurasjonsdokumentasjonen:

- Hvis verken `PAPERLESS_URL` eller `PAPERLESS_ALLOWED_HOSTS` er angitt, aksepterer Paperless-ngx fortsatt alle verter.
- Hvis bare `PAPERLESS_URL` er angitt, aksepterer Paperless-ngx bare vertsnavnet fra denne URL-en og `localhost`.
- `PAPERLESS_ALLOWED_HOSTS` legger til flere verter til `localhost` og vertsnavnet fra `PAPERLESS_URL`. `localhost` er alltid tillatt fordi Docker-helsekontrollen trenger det.
- `PAPERLESS_ALLOWED_HOSTS=*` tillater enhver vert, også når `PAPERLESS_URL` er angitt.

Berørt er installasjoner med angitt `PAPERLESS_URL` som i tillegg nås via andre navn, for eksempel via et internt vertsnavn, en IP-adresse eller tjenestenavnet i et containernettverk der andre tjenester kaller API-et. Django avviser slike forespørsler etter oppdateringen. Legg inn alle ekstra navn i `PAPERLESS_ALLOWED_HOSTS` før oppdateringen, for eksempel:

```shell
PAPERLESS_URL=https://paperless.example.com
PAPERLESS_ALLOWED_HOSTS=paperless.intern.example.com
```

Du bør bare angi wildcard-verdien `*` dersom Paperless-ngx utelukkende er tilgjengelig i et isolert nettverk; dokumentasjonen anbefaler uttrykkelig å angi tillatte verter for installasjoner på det åpne internettet.

## 3.5.0: OIDC-innlogging og økter

Med 3.4.0 kom innstillingen `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`, som gjør at Paperless-ngx hopper over den lokale tofaktorforespørselen når OIDC-leverandøren rapporterer en pålitelig andre faktor i `amr`-claimet. Ifølge Issue #14692 forble listen over pålitelige verdier alltid tom i 3.4.x: Koden sendte `mfa` som skilletegn i stedet for som standardverdi. Uten angitt variabel og også med `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR=mfa` spurte Paperless-ngx derfor fortsatt etter den lokale andre faktoren. PR #14693 retter dette i 3.5.0. Den dokumenterte standardverdien er `mfa`; innlogginger med passord krever fortsatt koden fra Paperless-ngx.

Dermed trer atferdsendringen som ble beskrevet i artikkelen om 3.4.0, først faktisk i kraft med 3.5.0: Brukere med lokalt konfigurert 2FA, der Identity Provider leverer `mfa` i `amr`-claimet, ser ikke lenger en ny forespørsel etter oppdateringen. Kontroller før oppdateringen hvilke AMR-verdier leverandøren din sender. Dokumentasjonen krever at du bare angir verdier som betyr at en andre faktor faktisk ble brukt ved innlogging. Samtidig oppgraderes django-allauth til 65.21.1 (PR #14672).

Paperless-ngx har tidligere aldri fjernet utløpte økter fra databasen. Bakgrunnen for endringen var Issue #14721: Med `PAPERLESS_AUTO_LOGIN_USERNAME` ble det opprettet en ny økt hvert 30. sekund, som aldri ble slettet. Fra 3.5.0 fjerner en planlagt oppgave utløpte økter (PR #14722). Tidspunktet styres av `PAPERLESS_SESSION_CLEANUP_TASK_CRON`, med standardverdien `0 3 * * *`, altså daglig klokken 03.00.

## 3.5.0: Bulkoperasjoner, arbeidsflyter og tagger

- **Asynkrone bulkoperasjoner:** Alle bulkoperasjoner på dokumenter og bulkredigering av metadata kjører via oppgavekøen (PR #14773, #14711) og vises i oppgavelisten. Ifølge API-dokumentasjonen svarer køsatte bulkoperasjoner med `{"result":"OK","task_id":"…"}`; du henter resultatet via `/api/tasks/?task_id=<uuid>`. Når bulkoppgaven er fullført, er etterfølgende oppgaver som ny behandling av filer bare planlagt, ikke ferdigstilt. Skript som arbeider videre med resultatet rett etter kallet, bør tilpasses tilsvarende.
- **Manuell arbeidsflytutløser:** Arbeidsflyter kan brukes på valgte dokumenter med den nye utløsertypen manuelt, i bulkredigering og i dokumentdetaljene (PR #14704).
- **Tagghierarki:** Paperless-ngx lagrer tagghierarkiet på nytt som en naboskapsliste og bruker ikke lenger django-treenode (PR #14612). Oppdateringen inneholder en databasemigrering for dette. Forfatteren målte i en prototype med 6000 tagger på SQLite betydelig kortere tider, rundt 1,3 i stedet for 4409 millisekunder for å opprette en tagg; verdiene er ikke uavhengig kontrollert. Ved sletting av en tagg kontrollerer Paperless-ngx dessuten tillatelsene på underordnede tagger (PR #14687).
- **Andre nyheter:** Notater til dokumenter vises som Markdown (PR #14626). En lokalt mellomlagret Hugging-Face-embeddingmodell lastes uten tilkobling til Hub-en (PR #14728), og KI-chatten filtrerer også over mer enn 32 700 dokumenter (PR #14754).

## 3.5.0: Flere feilrettinger

- **OCR-argumenter:** Argumentene som Paperless-ngx selv sender til OCRmyPDF (`input_file_or_options`, `output_file` og `sidecar`), avvises av API-et; i `PAPERLESS_OCR_USER_ARGS` ignoreres de (PR #14734). Kontroller om konfigurasjonen din inneholder disse argumentene.
- **Arbeidsflyter:** Filnavnfilteret for planlagte arbeidsflyter bruker et regulært uttrykk som er kompatibelt med PostgreSQL (PR #14783).
- **E-post:** Lagrede e-poster skrives med CRLF-linjeslutt (PR #14745), og PDF/A-konverteringen av e-postarkiver kjører bare én gang (PR #14698).
- **Forhåndsvisningsbilder:** For bilder uten arkivversjon oppretter Paperless-ngx igjen forhåndsvisningsbilder (PR #14741).
- **Tillatelser og kontoer:** Tillatelsene for konfigurasjonsvisningen er standardisert (PR #14709), ved registrering av den første brukeren låser Paperless-ngx de berørte databaselinjene (PR #14688), og passord vises med en konstant maskering (PR #14784). Forfatteren av PR #14709 vurderer uttrykkelig ikke endringen som en sårbarhet.

## Avhengigheter og sikkerhet

Det finnes ingen sikkerhetsadvarsel fra prosjektet for 3.4.1, 3.4.2 eller 3.5.0 (per 17. november 2026). Den nyeste advarselen, GHSA-c8f3-pqr2-rr24 fra 16. oktober 2026, gjelder KI-klassifiseringsarbeidsflyter på dokumenter uten eier og er rettet siden 3.3.2; detaljer finnes i artikkelen om 3.4.0. Systemer på en eldre 3.x-versjon lukker også dette hullet ved oppdatering til 3.5.0.

## Kjente problemer

- **Lokal 2FA ved OIDC til tross for AMR `mfa` (3.4.0 til 3.4.2):** rettet i 3.5.0 (Issue #14692, PR #14693).
- **Known Issues for 3.5.0:** Utgivelsesnotatene nevner ingen (per 17. november 2026). Merknaden om `PAPERLESS_ALLOWED_HOSTS` er en tilsiktet atferdsendring.

## Anbefalt fremgangsmåte

Målversjonen er 3.5.0. Før oppdateringen må du fastslå hvilke vertsnavn Paperless-ngx nås via, og legge dem til i `PAPERLESS_ALLOWED_HOSTS`; kontroller også verdiene i `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`. Stopp pågående behandlinger som dokumentimport og opprett en sikkerhetskopi, slik administrasjonsdokumentasjonen krever. For Docker-installasjoner med image fra Docker Hub stopper du stakken med `docker compose down`, laster ned det nye imaget med `docker compose pull` og starter med `docker compose up`; da kjører databasemigreringene, inkludert den nye migreringen av tagghierarkiet, automatisk. For Bare-Metal-installasjoner oppdaterer du Python-pakkene med `pip install -r requirements.txt` og kjører migreringene i katalogen `src` med `python3 manage.py migrate`; deretter oppdaterer `python3 manage.py document_index reindex --if-needed` og `python3 manage.py document_llmindex migrate` ved behov søkeindeksen og LLM-indeksen. Ved oppdatering direkte fra 3.3.x gjelder i tillegg merknadene i artikkelen om 3.4.0, særlig om `pdftoppm`. En Docker-installasjon med ekstern dokumentlagring beskrives i artikkelen [Drift av Paperless-ngx med lite lagringsplass](/blog/paperless-dokumente-clouddienst-auslagern).

## Kilder

1.  [GitHub: Paperless-ngx Release v3.5.0](https://github.com/paperless-ngx/paperless-ngx/releases/tag/v3.5.0): Utgivelsesnotater med merknad om `PAPERLESS_ALLOWED_HOSTS`, nye funksjoner og feilrettinger.

2.  [GitHub: Paperless-ngx Release v3.4.2](https://github.com/paperless-ngx/paperless-ngx/releases/tag/v3.4.2): Feilrettinger og oppdaterte avhengigheter.

3.  [GitHub: Paperless-ngx Release v3.4.1](https://github.com/paperless-ngx/paperless-ngx/releases/tag/v3.4.1): Feilrettinger og lavere minnebruk i hvilemodus.

4.  [GitHub API: Paperless-ngx Releases](https://api.github.com/repos/paperless-ngx/paperless-ngx/releases): Publiseringsdatoer for 3.4.1, 3.4.2 og 3.5.0.

5.  [Paperless-ngx: Konfigurasjonsdokumentasjon 3.5.0](https://github.com/paperless-ngx/paperless-ngx/blob/v3.5.0/docs/configuration.md): Ny atferd for `PAPERLESS_ALLOWED_HOSTS`, standardverdi for `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`, `PAPERLESS_SESSION_CLEANUP_TASK_CRON` og ignorerte OCR-argumenter.

6.  [Paperless-ngx: Konfigurasjonsdokumentasjon 3.4.2](https://github.com/paperless-ngx/paperless-ngx/blob/v3.4.2/docs/configuration.md): Tidligere standardverdi `*` for `PAPERLESS_ALLOWED_HOSTS`.

7.  [GitHub: Issue #14692, MFA trusted AMR alltid tom](https://github.com/paperless-ngx/paperless-ngx/issues/14692): Årsak og symptomer på den virkningsløse AMR-standardverdien i 3.4.x.

8.  [GitHub: PR #14722, automatisk opprydding av økter](https://github.com/paperless-ngx/paperless-ngx/pull/14722): Bakgrunn for aldri slettede økter ved automatisk innlogging.

9.  [GitHub: PR #14612, tagghierarki som naboskapsliste](https://github.com/paperless-ngx/paperless-ngx/pull/14612): Erstatning av django-treenode, migrering og forfatterens måleverdier.

10.  [Paperless-ngx: API-dokumentasjon 3.5.0](https://github.com/paperless-ngx/paperless-ngx/blob/v3.5.0/docs/api.md): Asynkrone bulkoperasjoner og oppslag via `task_id`.

11.  [Paperless-ngx: Administrasjonsdokumentasjon 3.5.0](https://github.com/paperless-ngx/paperless-ngx/blob/v3.5.0/docs/administration.md): Oppdateringsprosedyre for Docker og Bare Metal.

12.  [GitHub: Paperless-ngx Security Advisories](https://github.com/paperless-ngx/paperless-ngx/security/advisories): Oversikt over advarsler, ingen for 3.4.1 til 3.5.0.
