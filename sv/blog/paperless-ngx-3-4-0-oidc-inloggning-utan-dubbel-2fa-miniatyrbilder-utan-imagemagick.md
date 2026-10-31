---
title: "Paperless-ngx 3.4.0: OIDC-inloggning utan dubbel 2FA, miniatyrbilder utan ImageMagick"
navTitle: "Paperless-ngx 3.4.0"
description: "Paperless-ngx 3.4.0 hoppar över lokal 2FA när OIDC-providern rapporterar en betrodd andra faktor, skapar PDF-miniatyrbilder med Poppler i stället för ImageMagick och uppdaterar flera beroenden av säkerhetsskäl. Kontrollera den nya AMR-inställningen före uppdateringen och, på Bare Metal-system, att pdftoppm är installerat."
date: "2026-10-30"
kategorie: "Paperless-ngx"
timeToRead: "5 min lästid"
themen:
  - paperless-ngx
produkte:
  - "paperless-ngx"
protokolle:
  - "releases"
slug: "paperless-ngx-3-4-0-oidc-inloggning-utan-dubbel-2fa-miniatyrbilder-utan-imagemagick"
translationId: article-57582656ad9c441b
translationOf: paperless-ngx-3-4-0
url: https://rafaelpfister.ch/sv/blog/paperless-ngx-3-4-0-oidc-inloggning-utan-dubbel-2fa-miniatyrbilder-utan-imagemagick
translationSourceHash: fe1a5dd3d088e7e544b9448ecc7099233f8287856a5eded7af7488e06c8f290a
translationModel: gpt-5.6-terra
translatedAt: 2026-10-31T03:20:04.494Z
translationReview: automatic
---

# Paperless-ngx 3.4.0: OIDC-inloggning utan dubbel 2FA, miniatyrbilder utan ImageMagick

Paperless-ngx-projektet släppte version 3.4.0 den 29 oktober 2026, en vecka efter 3.3.4. Funktionsversionen ger en inställning som gör att den lokala tvåfaktorsfrågan utelämnas vid OIDC-inloggningar om Identity Providern redan har bekräftat en andra faktor. Paperless-ngx skapar nu miniatyrbilder för PDF-dokument med Poppler-verktygen, och ImageMagick är inte längre ett beroende. Dessutom tillkommer utökningar för arbetsflöden och Custom Fields, ändringar av uppladdning och aviseringar i webbgränssnittet, åtta felrättningar samt uppdateringar av frontend- och Python-beroenden som projektet klassar som säkerhetsuppdateringar. Versionen nämner ingen egen säkerhetsrapport med CVE eller GHSA. Den riktar sig till alla som driver Paperless-ngx 3.x, särskilt installationer med OIDC-inloggning och Bare Metal-installationer vars systempaket du själv hanterar.

## Versionerna i översikt

| Version | Datum | Typ | Huvudpunkter |
|---|---|---|---|
| 3.4.0 | 29.10.2026 | Funktionsversion | hoppa över lokal 2FA vid betrodd OIDC-AMR, PDF-miniatyrbilder via Poppler, ImageMagick borttaget, Custom Fields i arbetsflödestitlar, säkerhetsuppdateringar av beroenden |

## OIDC-inloggning: lokal 2FA vid betrodd AMR

Nytt är att Paperless-ngx vid inloggning via en OIDC-provider utvärderar `amr`-anspråket i ID-tokenen (PR #14403). Om det innehåller något av värdena från `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR` hoppar Paperless-ngx över sin egen tvåfaktorsfråga för denna inloggning. Enligt konfigurationsdokumentationen är variabelns standardvärde `mfa`. Dokumentationen nämner som exempel Authelia, som rapporterar `mfa`, och Pocket ID, som rapporterar `phr` vid passkeys. För engångskoder via e-post skickar Pocket ID värdet `otp`; dokumentationen bedömer detta som ej betrott.

```shell
PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR=mfa
```

Eftersom standardvärdet redan är angivet förändras beteendet efter uppdateringen för användare med lokalt konfigurerad 2FA så snart deras Identity Provider levererar `mfa` i `amr`-anspråket. Kontrollera därför vilka AMR-värden din provider faktiskt skickar och om dessa värden bara sätts vid en verklig andra faktor. Dokumentationen kräver uttryckligen att endast ange värden som innebär att en andra faktor faktiskt användes vid inloggningen. Parallellt uppdaterar 3.4.0 django-allauth till 65.20 och oauthlib till 4.0 (PR #14570).

## PDF-miniatyrbilder med Poppler, ImageMagick borttaget

Tidigare anropade Paperless-ngx ImageMagick (`convert -density 300`) för miniatyrbilden av en PDF, som renderade första sidan via Ghostscript med 300 DPI. Från och med 3.4.0 renderar `pdftoppm` från Poppler-verktygen första sidan, medan Pillow skalar bilden och sparar den som WebP (PR #14420). Paperless-ngx beräknar upplösningen utifrån sidstorleken, som hämtas via `pdfinfo`, med högst 300 DPI. Ghostscript-reservlösningen för miniatyrbilder försvinner, medan reparation via qpdf kvarstår. PR:ens författare har mätt betydligt kortare körtider och lägre minnesbehov, för ett tomt A0-dokument cirka 0,21 i stället för 12,9 sekunder; värdena har inte verifierats oberoende.

Med PR #14440 tas ImageMagick bort helt. De återstående uppgifterna, främst att ta bort en alfakanal från bilder, hanteras av en hjälpfunktion baserad på Pillow; WebP-filer sparas då förlustfritt. ImageMagick-policyn som tidigare behövde anpassas för PDF-bearbetning behövs därmed inte längre. Ghostscript förblir installerat eftersom OCRmyPDF fortfarande behöver det.

För administratörer får detta två följder:

- **Bare Metal:** `pdftoppm` måste finnas. Om det saknas visar Paperless-ngx en varning vid start. Jämför de installerade systempaketen med den uppdaterade beroendelistan i installationsdokumentationen.
- **Konfiguration:** `PAPERLESS_CONVERT_MEMORY_LIMIT` och `PAPERLESS_CONVERT_TMPDIR` är enligt PR #14420 föråldrade och saknar verkan, `PAPERLESS_GS_BINARY` har också markerats som föråldrad. Du kan ta bort dessa poster från `paperless.conf` respektive `.env`-filen.

## Arbetsflöden, Custom Fields och webbgränssnitt

- **Arbetsflöden:** Titelmallar i arbetsflöden kan använda Custom Fields (PR #14564). I taggfält i arbetsflöden kan över- och underordnade taggar väljas oberoende av varandra (PR #14400).
- **Custom Fields:** Frågor mot datumfält kan göras relativt (PR #14549).
- **Uppladdning och aviseringar:** Navigeringsfältet har en uppladdningsknapp (PR #14442). Knappen visar antalet pågående uppladdningar och länkar till översikten, antalet aviseringar minskar (PR #14501, #14500).
- **Gränssnitt:** Dokumentdetaljerna har omarbetats (PR #14485), uppgiftsbadgen laddar snabbare (PR #14552) och AI-chatten visar Markdown (PR #14409, #14415).

## Felrättningar

- `document_exporter` avbröts vid export till en mapp med `--delete` om en föråldrad `manifest.json.tmp` fanns (PR #14526). Den som använder exportverktyget med `--delete` för säkerhetskopior bör efter uppdateringen kontrollera att exporterna åter körs helt färdigt.
- Vid redigering av ett dokument uppdateras sökindexet först efter arbetsflödeskörningarna (PR #14558).
- Om en streckkod inte ger någon dokumentseparering tilldelas taggar inte längre fördröjt (PR #14497).
- Avbrutna eller utgångna uppgifter visas tydligare (PR #14566).
- API:ets schemavy utlöser inte längre behörighetsförfrågningar för dokument (PR #14503), och listtypen för filtren för dokumentobjekt i API-schemat har korrigerats (PR #14524).
- Långa versionsbeteckningar kortas i rullgardinsmenyn och rullgardinsmenyernas storlek har standardiserats (PR #14578, #14510).

## Beroenden och säkerhet

Release Notes listar uppdateringar av frontend-beroenden (PR #14581) samt av brotli, djangorestframework, filelock, msgpack och setuptools (PR #14575) som säkerhetsuppdateringar. Release Notes anger inte vilka sårbarheter som åtgärdas i dessa paket. Angular-komponenterna är på 22.2.2. Det finns ingen egen säkerhetsrapport från projektet för 3.4.0 (per den 30 oktober 2026).

Den senaste rapporten GHSA-c8f3-pqr2-rr24 från den 16 oktober 2026 gäller versionerna 3.0 till före 3.3.2 och är åtgärdad i 3.3.2. När AI-funktioner var aktiverade kunde AI-klassificeringsarbetsflöden för dokument utan ägare överföra utdrag och metadata från andra användares privata dokument till den konfigurerade AI-providern (CVSS 3.1: 4.3, ingen CVE). System som fortfarande kör en äldre 3.x-version täpper också till denna lucka genom uppdateringen till 3.4.0.

## Rekommenderat tillvägagångssätt

Målversionen är 3.4.0. Stoppa pågående bearbetningar såsom dokumentimport före uppdateringen och skapa en säkerhetskopia, enligt administrationsdokumentationen. För Docker-installationer med en image från Docker Hub stoppar du stacken med `docker compose down`, hämtar den nya imagen med `docker compose pull` och startar med `docker compose up`, varvid databas-migreringar körs automatiskt; när allt fungerar startar du stacken i bakgrunden med `-d`. För Bare Metal-installationer installerar du först de nya systemberoendena (särskilt Poppler med `pdftoppm`), uppdaterar Python-paketen med `pip install -r requirements.txt` och kör migreringarna i katalogen `src` med `python3 manage.py migrate`. Bestäm före den första OIDC-inloggningen efter uppdateringen vilka värden som ska finnas i `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`. En Docker-installation med extern dokumentlagring beskrivs i artikeln [Köra Paperless-ngx med lite lagringsutrymme](/blog/paperless-dokumente-clouddienst-auslagern). Release Notes för 3.4.0 nämner inga Known Issues eller Breaking Changes (per den 30 oktober 2026).

## Källor

1.  [GitHub: Paperless-ngx Release v3.4.0](https://github.com/paperless-ngx/paperless-ngx/releases/tag/v3.4.0): Release Notes med datum, nya funktioner, felrättningar och beroenden.

2.  [Paperless-ngx: Changelog](https://github.com/paperless-ngx/paperless-ngx/blob/main/docs/changelog.md): Changelog i projektdokumentationen med posterna för 3.3.x och 3.4.0.

3.  [GitHub: PR #14403, skip local 2FA for trusted OIDC AMR](https://github.com/paperless-ngx/paperless-ngx/pull/14403): Ny utvärdering av AMR-anspråket vid OIDC-inloggning.

4.  [Paperless-ngx: Konfigurationsdokumentation](https://github.com/paperless-ngx/paperless-ngx/blob/main/docs/configuration.md): Beskrivning och standardvärde för `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`.

5.  [GitHub: PR #14420, PDF-miniatyrbilder via Poppler](https://github.com/paperless-ngx/paperless-ngx/pull/14420): Övergång till `pdftoppm`, föråldrade inställningar och författarens mätvärden.

6.  [GitHub: PR #14440, ImageMagick borttaget som beroende](https://github.com/paperless-ngx/paperless-ngx/pull/14440): Ersättning av de återstående ImageMagick-anropen med Pillow.

7.  [GitHub: Security Advisory GHSA-c8f3-pqr2-rr24](https://github.com/paperless-ngx/paperless-ngx/security/advisories/GHSA-c8f3-pqr2-rr24): Informationsläcka i AI-klassificeringsarbetsflöden, berör 3.0 till före 3.3.2, CVSS 4.3.

8.  [Paperless-ngx: Administrationsdokumentation](https://github.com/paperless-ngx/paperless-ngx/blob/main/docs/administration.md): Uppdateringsförfarande för Docker och Bare Metal, exportverktygets alternativ `--delete`.

9.  [GitHub API: Paperless-ngx Releases](https://api.github.com/repos/paperless-ngx/paperless-ngx/releases): Publiceringsdatum för 3.3.4 och 3.4.0.
