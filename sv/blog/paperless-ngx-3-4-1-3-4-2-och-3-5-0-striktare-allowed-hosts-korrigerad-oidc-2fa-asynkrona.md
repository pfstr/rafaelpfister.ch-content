---
title: "Paperless-ngx 3.4.1, 3.4.2 och 3.5.0: striktare ALLOWED_HOSTS, korrigerad OIDC-2FA, asynkrona bulkåtgärder"
navTitle: "Paperless-ngx 3.5.0"
description: "Paperless-ngx 3.5.0 tillåter, när PAPERLESS_URL är angiven, endast dess värdnamn och localhost, korrigerar det verkningslösa AMR-standardvärdet från 3.4.x och utför bulkåtgärder asynkront. Kontrollera före uppdateringen vilka värdnamn som används för att nå Paperless-ngx och lägg till dem i PAPERLESS_ALLOWED_HOSTS."
date: "2026-11-17"
kategorie: "Paperless-ngx"
timeToRead: "7 min. läsning"
themen:
  - paperless-ngx
produkte:
  - "paperless-ngx"
protokolle:
  - "releases"
slug: "paperless-ngx-3-4-1-3-4-2-och-3-5-0-striktare-allowed-hosts-korrigerad-oidc-2fa-asynkrona"
translationId: article-4c816dd3cd08ff6e
translationOf: paperless-ngx-3-5-0
url: https://rafaelpfister.ch/sv/blog/paperless-ngx-3-4-1-3-4-2-och-3-5-0-striktare-allowed-hosts-korrigerad-oidc-2fa-asynkrona
translationSourceHash: 2c1c3db7502c455d8090a922dd17654a159d2d094be4f29d0da51f29eababfb4
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T03:23:17.867Z
translationReview: required
---

# Paperless-ngx 3.4.1, 3.4.2 och 3.5.0: striktare ALLOWED_HOSTS, korrigerad OIDC-2FA, asynkrona bulkåtgärder

Sedan 3.4.0 har Paperless-ngx-projektet släppt tre versioner: bugfixreleaserna 3.4.1 den 31 oktober 2026 och 3.4.2 den 3 november 2026 samt funktionsreleasen 3.5.0 den 17 november 2026. De två patchversionerna korrigerar främst visningsfel i det omarbetade webbgränssnittet och uppdaterar Django och andra beroenden. 3.5.0 ändrar standardvärdet för tillåtna värdnamn, åtgärdar ett fel som gjorde att inställningen `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR` som infördes i 3.4.0 inte hade någon effekt, utför alla bulkåtgärder asynkront, rensar automatiskt upp utgångna sessioner och introducerar manuellt utlösbara arbetsflöden. Ingen av de tre releaserna nämner någon säkerhetsnotis med CVE eller GHSA. Föregående status beskrivs i artikeln [Paperless-ngx 3.4.0: OIDC-inloggning utan dubbel 2FA, miniatyrbilder utan ImageMagick](/blog/paperless-ngx-3-4-0). Uppdateringarna riktar sig till alla operatörer av Paperless-ngx 3.x; särskilt berörda är installationer som nås via interna värdnamn och installationer med OIDC-inloggning.

## Versionerna i korthet

| Version | Datum | Typ | Huvudpunkter |
|---|---|---|---|
| 3.4.1 | 31.10.2026 | Bugfixrelease | Korrigeringar av API-schemat och webbgränssnittet, lägre minnesanvändning vid inaktivitet |
| 3.4.2 | 03.11.2026 | Bugfixrelease | Bulkredigeringsuppgifter i Celery-arbetare, installation av Tesseract-språk före indexbygge, filelock 4.2.6, uppdateringar av Django och frontendberoenden |
| 3.5.0 | 17.11.2026 | Funktionsrelease | ALLOWED_HOSTS utan jokertecken när PAPERLESS_URL är angiven, AMR-standardvärde korrigerat, asynkrona bulkåtgärder, automatisk rensning av sessioner, manuell arbetsflödesutlösare, tagghierarki utan django-treenode |

## 3.4.1 och 3.4.2: korrigeringar efter funktionsreleasen

3.4.1 korrigerar placeringen av en schemakorrigering för `id`-fält (PR #14604), tar bort dubbla behörighetskontroller vid öppningsknappen på dokumentkort (PR #14601), döljer de nya PDF-kontrollerna i stort sett (PR #14595), flyttar förhandsgranskningsknappen i dokumentlistan (PR #14594) och åtgärdar visningen av kontrollsumman i ljust läge samt positionen för rullgardinsmenyn för anpassade fält på mobila enheter (PR #14590). Dessutom laddar Paperless-ngx fler moduler först vid behov, vilket minskar minnesanvändningen vid inaktivitet (PR #14599).

3.4.2 innehåller elva felrättningar. Följande är relevanta för drift:

- Importuppgifter från `bulk_edit` körs säkert i Celery-arbetarna (PR #14653).
- Uppbyggnaden av sökindexet och LLM-indexet väntar tills Tesseract-språkpaketen har installerats (PR #14670).
- filelock är uppdaterat till 4.2.6; release notes motiverar uppdateringen med korrigeringar av ACL:er (PR #14656).
- Dokumenthistoriken visar taggnamn i stället för ID:n (PR #14625), zoomknappar skickar inte längre formulär (PR #14624), och namnet på en sparad vy behålls när den sparas och stängs (PR #14667).

Bland beroendena uppdaterar 3.4.2 Django, django-guardian, django-filter och drf-spectacular-sidecar (PR #14661), turbohtml till 1.15.3 (PR #14662) och pdfjs-dist till 6.4.299 (PR #14644). Projektet ersätter det inte längre underhållna biblioteket ngx-bootstrap-icons med en egen lösning (PR #14665). I exempel-Compose-filerna uppdateras Gotenberg till 8.38 (PR #14634) och nginx till 1.31.7 (PR #14633). Release notes för 3.4.1 och 3.4.2 markerar ingen av dessa poster som säkerhetsuppdateringar.

## 3.5.0: ALLOWED_HOSTS utan jokertecken

Release notes för 3.5.0 börjar med upplysningen att vissa användare måste anpassa sin inställning `PAPERLESS_ALLOWED_HOSTS` (PR #14735). Fram till 3.4.2 dokumenterade projektet standardvärdet `*` för `PAPERLESS_ALLOWED_HOSTS`, alltså alla värdar. Från och med 3.5.0 gäller enligt konfigurationsdokumentationen:

- Om varken `PAPERLESS_URL` eller `PAPERLESS_ALLOWED_HOSTS` är angiven accepterar Paperless-ngx fortfarande alla värdar.
- Om endast `PAPERLESS_URL` är angiven accepterar Paperless-ngx bara värdnamnet från denna URL och `localhost`.
- `PAPERLESS_ALLOWED_HOSTS` lägger till fler värdar till `localhost` och värdnamnet från `PAPERLESS_URL`. `localhost` är alltid tillåtet eftersom Docker-hälsokontrollen behöver det.
- `PAPERLESS_ALLOWED_HOSTS=*` tillåter alla värdar, även om `PAPERLESS_URL` är angiven.

Installationer med angiven `PAPERLESS_URL` som dessutom nås via andra namn berörs, exempelvis via ett internt värdnamn, en IP-adress eller tjänstnamnet i ett containernätverk som andra tjänster använder för att anropa API:t. Django avvisar sådana förfrågningar efter uppdateringen. Lägg före uppdateringen in alla ytterligare namn i `PAPERLESS_ALLOWED_HOSTS`, till exempel:

```shell
PAPERLESS_URL=https://paperless.example.com
PAPERLESS_ALLOWED_HOSTS=paperless.intern.example.com
```

Jokerteckenvärdet `*` bör du bara ange om Paperless-ngx enbart är tillgängligt i ett isolerat nätverk; dokumentationen rekommenderar uttryckligen att tillåtna värdar anges för installationer på det öppna internet.

## 3.5.0: OIDC-inloggning och sessioner

Med 3.4.0 kom inställningen `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`, med vilken Paperless-ngx hoppar över den lokala tvåfaktorsfrågan om OIDC-leverantören rapporterar en betrodd andra faktor i `amr`-claimet. Enligt Issue #14692 förblev listan över betrodda värden alltid tom i 3.4.x: koden skickade `mfa` som avgränsare i stället för som standardvärde. Utan angiven variabel och även med `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR=mfa` frågade Paperless-ngx därför fortfarande efter den lokala andra faktorn. PR #14693 åtgärdar detta i 3.5.0. Det dokumenterade standardvärdet är `mfa`; inloggningar med lösenord kräver fortfarande koden från Paperless-ngx.

Därmed träder beteendeförändringen som beskrivs i artikeln om 3.4.0 faktiskt i kraft först med 3.5.0: användare med lokalt konfigurerad 2FA, vars Identity Provider levererar `mfa` i `amr`-claimet, ser inte längre någon andra fråga efter uppdateringen. Kontrollera före uppdateringen vilka AMR-värden din leverantör skickar. Dokumentationen kräver att endast värden anges som innebär att en andra faktor faktiskt användes vid inloggningen. Samtidigt uppdateras django-allauth till 65.21.1 (PR #14672).

Paperless-ngx har hittills aldrig tagit bort utgångna sessioner från databasen. Ändringen utlöstes av Issue #14721: med `PAPERLESS_AUTO_LOGIN_USERNAME` skapades en ny session var 30:e sekund, som aldrig raderades. Från och med 3.5.0 tar en schemalagd uppgift bort utgångna sessioner (PR #14722). Tidpunkten styrs av `PAPERLESS_SESSION_CLEANUP_TASK_CRON`, standardvärdet är `0 3 * * *`, alltså varje dag klockan 03:00.

## 3.5.0: bulkåtgärder, arbetsflöden och taggar

- **Asynkrona bulkåtgärder:** Alla bulkåtgärder på dokument och bulkredigering av metadata körs via uppgiftskön (PR #14773, #14711) och visas i uppgiftslistan. Enligt API-dokumentationen svarar köade bulkåtgärder med `{"result":"OK","task_id":"…"}`; du hämtar resultatet via `/api/tasks/?task_id=<uuid>`. När bulkuppgiften är klar är följduppgifter, såsom ombearbetning av filer, bara schemalagda och inte slutförda. Anpassa därför skript som arbetar vidare med resultatet direkt efter anropet.
- **Manuell arbetsflödesutlösare:** Arbetsflöden kan med den nya utlösartypen tillämpas manuellt på utvalda dokument, vid bulkredigering och i dokumentdetaljerna (PR #14704).
- **Tagghierarki:** Paperless-ngx lagrar tagghierarkin på nytt som en adjacenslista och använder inte längre django-treenode (PR #14612). Uppdateringen innehåller en databas­migrering för detta. Författaren uppmätte i en prototyp med 6 000 taggar på SQLite betydligt kortare tider, exempelvis 1,3 i stället för 4 409 millisekunder för att skapa en tagg; värdena har inte verifierats oberoende. När en tagg raderas kontrollerar Paperless-ngx dessutom behörigheterna för underordnade taggar (PR #14687).
- **Övriga nyheter:** Anteckningar till dokument visas som Markdown (PR #14626). En lokalt cachad Hugging Face-embeddingmodell laddas utan anslutning till hubben (PR #14728), och AI-chatten filtrerar även bland fler än 32 700 dokument (PR #14754).

## 3.5.0: ytterligare felrättningar

- **OCR-argument:** Argumenten som Paperless-ngx själv skickar till OCRmyPDF (`input_file_or_options`, `output_file` och `sidecar`) avvisas av API:t; i `PAPERLESS_OCR_USER_ARGS` ignoreras de (PR #14734). Kontrollera om din konfiguration innehåller dessa argument.
- **Arbetsflöden:** Filnamnsfiltret för schemalagda arbetsflöden använder ett reguljärt uttryck som är kompatibelt med PostgreSQL (PR #14783).
- **E-post:** Sparade e-postmeddelanden skrivs med CRLF-radslut (PR #14745), och PDF/A-konverteringen av e-postarkiv körs endast en gång (PR #14698).
- **Miniatyrbilder:** För bilder utan arkivversion skapar Paperless-ngx åter miniatyrbilder (PR #14741).
- **Behörigheter och konton:** Behörigheterna för konfigurationsvyn har enhetliggjorts (PR #14709), vid registrering av den första användaren låser Paperless-ngx de berörda databasraderna (PR #14688), och lösenord visas med en konstant mask (PR #14784). Författaren till PR #14709 bedömer uttryckligen inte ändringen som en sårbarhet.

## Beroenden och säkerhet

Det finns ingen Security Advisory från projektet för 3.4.1, 3.4.2 eller 3.5.0 (status 17 november 2026). Den senaste advisorn GHSA-c8f3-pqr2-rr24 från den 16 oktober 2026 gäller AI-klassificeringsarbetsflöden för dokument utan ägare och har åtgärdats sedan 3.3.2; detaljer finns i artikeln om 3.4.0. System med en äldre 3.x-version stänger också denna lucka genom uppdateringen till 3.5.0.

## Kända problem

- **Lokal 2FA vid OIDC trots AMR `mfa` (3.4.0 till 3.4.2):** åtgärdat i 3.5.0 (Issue #14692, PR #14693).
- **Known Issues för 3.5.0:** Release notes nämner inga (status 17 november 2026). Upplysningen om `PAPERLESS_ALLOWED_HOSTS` är en avsiktlig beteendeförändring.

## Rekommenderat tillvägagångssätt

Målversionen är 3.5.0. Fastställ före uppdateringen vilka värdnamn som används för att nå Paperless-ngx och lägg till dem i `PAPERLESS_ALLOWED_HOSTS`; kontrollera även värdena i `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`. Stoppa pågående bearbetningar såsom dokumentimporten och skapa en säkerhetskopia, enligt administrationsdokumentationen. För Docker-installationer med image från Docker Hub stoppar du stacken med `docker compose down`, hämtar den nya imagen med `docker compose pull` och startar med `docker compose up`; databas­migreringarna, inklusive den nya migreringen av tagghierarkin, körs då automatiskt. För Bare Metal-installationer uppdaterar du Python-paketen med `pip install -r requirements.txt` och kör migreringarna i katalogen `src` med `python3 manage.py migrate`; därefter uppdaterar `python3 manage.py document_index reindex --if-needed` och `python3 manage.py document_llmindex migrate` vid behov sökindexet och LLM-indexet. Vid en uppdatering direkt från 3.3.x gäller dessutom informationen i artikeln om 3.4.0, särskilt om `pdftoppm`. En Docker-installation med externt dokumentlager beskrivs i artikeln [Köra Paperless-ngx med lite lagringsutrymme](/blog/paperless-dokumente-clouddienst-auslagern).

## Källor

1.  [GitHub: Paperless-ngx Release v3.5.0](https://github.com/paperless-ngx/paperless-ngx/releases/tag/v3.5.0): Release notes med information om `PAPERLESS_ALLOWED_HOSTS`, nya funktioner och felrättningar.

2.  [GitHub: Paperless-ngx Release v3.4.2](https://github.com/paperless-ngx/paperless-ngx/releases/tag/v3.4.2): Felrättningar och uppdaterade beroenden.

3.  [GitHub: Paperless-ngx Release v3.4.1](https://github.com/paperless-ngx/paperless-ngx/releases/tag/v3.4.1): Felrättningar och lägre minnesanvändning vid inaktivitet.

4.  [GitHub API: Paperless-ngx Releases](https://api.github.com/repos/paperless-ngx/paperless-ngx/releases): Publiceringsdatum för 3.4.1, 3.4.2 och 3.5.0.

5.  [Paperless-ngx: Konfigurationsdokumentation 3.5.0](https://github.com/paperless-ngx/paperless-ngx/blob/v3.5.0/docs/configuration.md): nytt beteende för `PAPERLESS_ALLOWED_HOSTS`, standardvärde för `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`, `PAPERLESS_SESSION_CLEANUP_TASK_CRON` och ignorerade OCR-argument.

6.  [Paperless-ngx: Konfigurationsdokumentation 3.4.2](https://github.com/paperless-ngx/paperless-ngx/blob/v3.4.2/docs/configuration.md): tidigare standardvärde `*` för `PAPERLESS_ALLOWED_HOSTS`.

7.  [GitHub: Issue #14692, MFA trusted AMR alltid tom](https://github.com/paperless-ngx/paperless-ngx/issues/14692): Orsak och symtom för det verkningslösa AMR-standardvärdet i 3.4.x.

8.  [GitHub: PR #14722, automatisk rensning av sessioner](https://github.com/paperless-ngx/paperless-ngx/pull/14722): Bakgrund till sessioner som aldrig raderades vid automatisk inloggning.

9.  [GitHub: PR #14612, tagghierarki som adjacenslista](https://github.com/paperless-ngx/paperless-ngx/pull/14612): Ersättning av django-treenode, migrering och författarens mätvärden.

10.  [Paperless-ngx: API-dokumentation 3.5.0](https://github.com/paperless-ngx/paperless-ngx/blob/v3.5.0/docs/api.md): asynkrona bulkåtgärder och hämtning via `task_id`.

11.  [Paperless-ngx: Administrationsdokumentation 3.5.0](https://github.com/paperless-ngx/paperless-ngx/blob/v3.5.0/docs/administration.md): Uppdateringsförfarande för Docker och Bare Metal.

12.  [GitHub: Paperless-ngx Security Advisories](https://github.com/paperless-ngx/paperless-ngx/security/advisories): Översikt över advisories, inga för 3.4.1 till 3.5.0.
