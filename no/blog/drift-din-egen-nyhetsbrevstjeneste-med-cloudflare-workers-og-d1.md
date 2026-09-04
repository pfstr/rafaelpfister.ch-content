---
title: "Drifte en egen newsletter med Cloudflare Workers og D1"
navTitle: "Newsletter på Workers"
description: "Den åpne malen tilbyr påmelding, avmelding, kø og database i din egen Cloudflare-konto. En deploy-knapp setter opp Worker, D1 og CI uten lokal server."
date: "2026-07-22"
kategorie: "Cloudflare Workers"
timeToRead: "8 min lesetid"
themen:
  - cloudflare-workers
slug: "drift-din-egen-nyhetsbrevstjeneste-med-cloudflare-workers-og-d1"
translationOf: "serverloser-newsletter-cloudflare-workers-d1"
translationId: article-4e7139acdb90923b
translationReview: automatic
translationSourceHash: ad5b78d6330d06a17259e464c0fb8bb9713b3fdf5cd6c77ac1d300d9fea2a48e
translatedAt: 2026-09-04T08:41:16.569Z
url: https://rafaelpfister.ch/no/blog/drift-din-egen-nyhetsbrevstjeneste-med-cloudflare-workers-og-d1
translationModel: gpt-5.6-terra
---

# Drifte en egen newsletter med Cloudflare Workers og D1

Med en hostet newsletter-tjeneste ligger mottakerlisten hos leverandøren, og kostnadene øker ofte med antallet abonnenter. En egen server gir mer kontroll, men medfører løpende arbeid: oppdateringer, overvåking, sikkerhetskopiering og drift av et system som kanskje bare sender én gang i uken.

For dette slanke bruksområdet holder det med HTTP-endepunkter, en liten database og en tidsstyrt sendejobb. Cloudflare Workers og D1 tilbyr nettopp disse byggeklossene. Den åpne malen min setter dem opp i din egen konto via en **Deploy-to-Cloudflare-knapp**. Verken en lokal kommandolinje eller en server som må vedlikeholdes kontinuerlig, er nødvendig. Kildekoden under MIT-lisensen ligger på [GitHub](https://github.com/pfstr/newsletter-template).

[![Deploy to Cloudflare](../images/serverloser-newsletter-cloudflare-workers-d1/deploy-to-cloudflare.svg)](https://deploy.workers.cloudflare.com/?url=https://github.com/pfstr/newsletter-template)

![Det hostede påmeldingsskjemaet i malen](../images/serverloser-newsletter-cloudflare-workers-d1/newsletter-template-signup.png)

## Hva malen kan gjøre

- **Påmelding**: en hostet påmeldingsside, et innebyggbart skjema for ditt eget nettsted og et JSON-endepunkt
- **One-click-avmelding**: i samsvar med RFC 8058, med individuelt token per abonnent
- **Obligatorisk informasjon innebygd**: Hver e-post får automatisk en bunntekst med avmeldingslenke og postadresse; tidspunkter for samtykke og avmelding lagres
- **Utsending**: På en beskyttet side kan du angi emne og HTML, sende en test-e-post og legge kampanjen i kø; en bakgrunnsjobb sender i batcher og gjentar mislykkede forsøk
- **Egne data**: Abonnenter lagres i en D1-database i din konto og kan eksporteres når som helst
- **Valgfritt, deaktivert som standard**: Double opt-in, botbeskyttelse med Turnstile og automatisk utsending av nye blogginnlegg fra RSS-feeden

## Arkitektur: én Worker, én database

Hele systemet består av én enkelt Cloudflare Worker med to handlere: `fetch` for HTTP (routet med Hono) og `scheduled` for Cron-utløseren, samt en D1-database. Det finnes ingen annen tjeneste, ingen separat kømegler, ingen egen admin-backend; selv sendekøen er bare en D1-tabell.

| Rute | Funksjon |
| --- | --- |
| `GET /` | Hostet påmeldingsside |
| `GET /embed` | Transparent skjema for innbygging via iframe |
| `POST /api/subscribe` | Påmelding (CORS-åpen for eget nettsted) |
| `GET /confirm` | Bekreftelseslenke ved double opt-in |
| `GET/POST /unsubscribe` | Avmelding: bekreftelsesside via GET, utførelse via POST (one-click i henhold til RFC 8058) |
| `GET /admin` | Utsendingsside (skjema) |
| `POST /api/send` | Legg kampanje i kø, beskyttet med admin-token |

Datamodellen består av fire tabeller: `subscribers` (e-post som primærnøkkel, navn, status, avmeldings- og bekreftelsestoken, en JSON-kolonne for selvdefinerte tilleggsfelt samt tidsstempler for bekreftelse og avmelding), `campaigns` med emne, innhold og tellere per utsending, `outbox` som sendekø (én rad per mottaker) og `sent_posts` for deduplisering av RSS-utsendinger.

## Deployment uten kommandolinje

Mer interessant enn koden er veien til et system som kjører. Deploy-to-Cloudflare-knappen leser Wrangler-konfigurasjonen i repositoriet og utfører hele oppsettet: Den kloner repositoriet til din egen GitHub-konto, klargjør D1-databasen, kjører skjemamigreringene og setter opp CI, slik at hver push deployes automatisk. Siden juli 2025 ber deploy-flyten i tillegg om miljøvariabler og secrets direkte i skjemaet: for denne malen admin-passordet (`ADMIN_TOKEN`), avsendernavn og -adresse, double-opt-in-bryteren og størrelsen på utsendingsbatchen (`SEND_BATCH`).

Resultatet etter ett klikk og ett skjema: Påmeldingssiden er live på `https://<worker-name>.workers.dev` og samler abonnenter. Ingen terminal åpnes på noe tidspunkt.

## Samle abonnenter

For integrering på eget nettsted finnes det tre alternativer, i stigende grad av integrasjon. Det enkleste: del lenken til den hostede påmeldingssiden. Det mest praktiske for nettstedbyggere (WordPress, Webflow, Squarespace, Framer): en iframe på én linje i en valgfri HTML-embed-blokk.

```html
<iframe
  src="https://<worker-name>.workers.dev/embed"
  style="width:100%;max-width:420px;height:90px;border:0"
></iframe>
```

De som vil ha skjemaet i eget design, poster direkte til endepunktet:

```html
<form
  onsubmit="event.preventDefault();
  fetch('https://<worker-name>.workers.dev/api/subscribe', {
    method:'POST', headers:{'Content-Type':'application/json'},
    body: JSON.stringify({ email: this.email.value })
  }).then(()=>this.reset());"
>
  <input name="email" type="email" placeholder="you@example.com" required />
  <button>Abonnieren</button>
</form>
```

Skjemaet samler som standard inn e-post og valgfritt navn. Flere felt (firma, land, …) definerer du i én enkelt fil (`src/fields.ts`); de vises automatisk i begge skjemaene og lagres som JSON i databasen.

## Utsending: egen leverandør i stedet for innebygd vendor

For e-postutsendingen tar malen et bevisst valg: Den er **leverandøragnostisk**. Filen `src/email.ts` inneholder én enkelt `sendEmail()`-adapter med et kommentert eksempel for et generisk HTTP-API. Hvilken utsendingstjeneste du kobler til der, er ditt valg. Ingen leverandør er hardkodet, og registrering hos en bestemt tjeneste er ikke en forutsetning. Innsamling av abonnenter fungerer allerede fullt ut uten utsendingskonfigurasjon; utsending aktiveres så snart adapteren er implementert og leverandørens secret er satt. Dersom leverandøren også tilbyr et batch-endepunkt (ett API-kall, mange e-poster), kan en valgfri `sendEmailBatch()`-adapter legges til i samme fil; også for dette finnes et kommentert eksempel.

Utsendingen betjenes via `/admin`-siden: Legg inn emne og e-post-HTML, send en test til din egen adresse, og legg deretter kampanjen i kø for alle abonnenter. I e-postene er merge-tagene `{{unsubscribe_url}}`, `{{email}}` og `{{name}}` tilgjengelige.

Selve utsendingen skjer i bakgrunnen, etter Transactional Outbox-mønsteret: `POST /api/send` skriver kampanjen og én rad per mottaker til databasen og svarer umiddelbart. En Cron-jobb hvert minutt leverer deretter `SEND_BATCH` e-poster per kjøring, som standard 40: valgt slik at hver kjøring holder seg innenfor grensene for subrequests i Workers Free-planen. Radene reserveres atomisk, slik at overlappende kjøringer aldri kan sende dobbelt; mislykkede leveringer gjentas opptil tre ganger, og avbrutte kjøringer gjenopptas etter ti minutter. Og den som melder seg av mens egen e-post fortsatt står i køen, mottar den ikke: Opt-out kansellerer også meldinger som allerede er lagt i kø.

## Avmelding og dokumentasjon er en kjernefunksjon

De som sender newslettere, er underlagt anti-spam- og personvernregelverk: den amerikanske CAN-SPAM Act, GDPR og ePrivacy i EU, og UWG i Sveits. En vesentlig del av det man betaler newsletter-tjenester for, er nettopp å oppfylle disse kravene. Malen tar seg av den mekaniske delen:

- **Obligatorisk bunntekst**: Hver kampanje-e-post får automatisk en bunntekst med fungerende avmeldingslenke og avsenderens postadresse (`SENDER_ADDRESS`); CAN-SPAM krever en fysisk adresse i kommersielle e-poster. Utsendingssiden advarer så lenge adressen mangler.
- **List-Unsubscribe-header i henhold til RFC 8058** for hver utsending: den innebygde avmeldingsknappen i Gmail og Outlook, som Gmail og Yahoo har krevd av masseavsendere siden 2024. Appen setter sammen headerne ferdig; din egen leverandøradapter trenger bare å videresende dem.
- **Skannersikker avmelding**: Avmeldingslenken leder til en bekreftelsesside med én enkelt knapp. Bedrifts-e-postskannere som henter alle lenker i en e-post på forhånd, kan dermed ikke avmelde noen ved en feil; e-postklienter bruker One-Click-POST direkte.
- **Dataminimering og dokumentasjon**: En opt-out trer i kraft umiddelbart, sletter navn og tilleggsfelt og registreres med tidsstempel, det samme gjelder påmelding og double-opt-in-bekreftelse. Samtykket kan dermed dokumenteres i ettertid (GDPRs ansvarlighetsprinsipp).
- **Personvernlenke**: Når `PRIVACY_URL` er satt, vises en lenke til din egen personvernerklæring under påmeldingsskjemaet.

Operatøren er fortsatt ansvarlig for sannferdige avsender- og emnelinjer, utsending kun til adresser som faktisk er påmeldt, og domeneautentisering (SPF/DKIM/DMARC) hos utsendingsleverandøren. Dette er ikke juridisk rådgivning.

## Alternativer: double opt-in, Turnstile, RSS-automatisering

Tre funksjoner er innebygd, men deaktivert som standard, slik at systemet kan kjøre uten konfigurasjon:

- **Double opt-in** (`DOUBLE_OPT_IN = "true"`): Nye abonnenter lagres som `pending` og blir først aktive etter et klikk på en bekreftelseslenke. For Sveits (DSG) og EU er denne metoden det ryddigere valget.
- **Botbeskyttelse** med Cloudflare Turnstile: Sett site- og secret-key som variabler; widgeten vises automatisk i begge skjemaene, og Workeren verifiserer hver påmelding på serversiden. Uten gyldig token blir påmeldingen avvist.
- **RSS-autoutsending**: En Cron-jobb sjekker din egen bloggfeed (RSS 2.0 eller Atom) hvert 15. minutt og legger nye artikler automatisk i sendekøen. To sikkerhetsmekanismer er innebygd: Ved aller første kjøring markeres den eksisterende feeden kun som en baseline (arkivet sendes altså ikke som en e-postflom), og hver artikkel-ID registreres i `sent_posts`, slik at ingen innlegg sendes to ganger.

## Begrensninger

Malen er bevisst holdt minimal. Købasert utsending leverer i Free-planen som standard rundt 40 e-poster per minutt; en kampanje til 1'000 mottakere tar dermed omtrent 25 minutter, noe som ikke spiller noen rolle for en newsletter. I den betalte Workers-planen (10'000 subrequests per kall i stedet for 50) kan `SEND_BATCH` økes til flere hundre; med en batch-adapter (ett API-kall, opptil rundt 1'000 e-poster) sender også Free-planen store lister på få minutter. Leveringsdyktigheten avhenger, som i ethvert system, av din egen avsenderdomene: SPF, DKIM og DMARC må være verifisert hos den valgte utsendingsleverandøren, ellers havner newsletteren i spam. Og single-opt-in som standard er den enkleste starten, men ikke den mest konservative compliance-varianten; til det finnes bryteren.

Når det gjelder kostnader: Workers og D1 har romslige Free Tier-kvoter (blant annet 100'000 forespørsler per dag), som et påmeldingsskjema og ukentlige utsendinger til en liten til mellomstor liste ikke bruker opp. Når en grense nås, struper Cloudflare i Free-planen i stedet for å sende en faktura.

## Prøv det ut

Kildekoden inkludert deploy-knappen ligger på [GitHub](https://github.com/pfstr/newsletter-template); der finnes også den komplette dokumentasjonen av konfigurasjonsvariablene.

[![GitHub: pfstr/newsletter-template](../images/serverloser-newsletter-cloudflare-workers-d1/github-newsletter-template.svg)](https://github.com/pfstr/newsletter-template)

## Kilder

1.  [pfstr/newsletter-template](https://github.com/pfstr/newsletter-template): Kildekoden til malen (MIT) med deploy-knapp og dokumentasjon.

2.  [Deploy to Cloudflare buttons](https://developers.cloudflare.com/workers/platform/deploy-buttons/): automatisk klargjøring av ressurser, repo-kloning og CI ved deploy.

3.  [Deploy buttons: environment variables and secrets](https://developers.cloudflare.com/changelog/post/2025-07-01-workers-deploy-button-supports-environment-variables-and-secrets/): Secrets og variabler spørres om i deploy-skjemaet siden juli 2025.

4.  [Cloudflare D1](https://developers.cloudflare.com/d1/): serverløs SQLite, her for abonnenter, sendeprotokoll og RSS-deduplisering.

5.  [Cloudflare Turnstile](https://developers.cloudflare.com/turnstile/): botbeskyttelse uten CAPTCHA-gåter, kan valgfritt aktiveres i malen.

6.  [RFC 8058](https://datatracker.ietf.org/doc/html/rfc8058): Signaling One-Click Functionality for List Email Headers; grunnlaget for den innebygde avmeldingsknappen i Gmail og Outlook.

7.  [Workers limits](https://developers.cloudflare.com/workers/platform/limits/): grenser for subrequests per kall (50 i Free-planen, 10'000 i den betalte planen); dette bestemmer batchstørrelsen for købasert utsending.

8.  [FTC: CAN-SPAM Act Compliance Guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business): krav for kommersielle e-poster, inkludert postadresse og fungerende opt-out.
