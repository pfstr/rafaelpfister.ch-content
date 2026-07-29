---
title: "Drift din egen nyhetsbrevstjeneste med Cloudflare Workers og D1"
navTitle: "Nyhetsbrev på Workers"
description: "Den åpne malen gir påmelding, avmelding, kø og database i din egen Cloudflare-konto. En deploy-knapp setter opp Worker, D1 og CI uten lokal server."
date: "2026-07-22"
kategorie: "Cloudflare Workers"
timeToRead: "8 min lesetid"
themen:
  - cloudflare-workers
slug: "drift-din-egen-nyhetsbrevstjeneste-med-cloudflare-workers-og-d1"
translationOf: "serverloser-newsletter-cloudflare-workers-d1"
url: "https://rafaelpfister.ch/no/blog/drift-din-egen-nyhetsbrevstjeneste-med-cloudflare-workers-og-d1"
translationId: article-4e7139acdb90923b
translationReview: automatic
translationSourceHash: 90c100386e148f80be4d4be81dc928f373431ce83b5f6e2336cfb0daafd3945e
translatedAt: 2026-07-29T12:29:38.965Z
---

# Drift din egen nyhetsbrevstjeneste med Cloudflare Workers og D1

Med en hostet nyhetsbrevtjeneste ligger mottakerlisten hos leverandøren, og kostnadene øker ofte med antall abonnenter. En egen server gir mer kontroll, men medfører løpende arbeid: oppdateringer, overvåking, sikkerhetskopier og drift av et system som kanskje bare sender én gang i uken.

For dette slanke bruksområdet holder det med HTTP-endepunkter, en liten database og en tidsstyrt sendejobb. Cloudflare Workers og D1 tilbyr nettopp disse byggesteinene. Den åpne malen min setter dem opp i din egen konto via en **Deploy-to-Cloudflare-knapp**. Du trenger verken en lokal kommandolinje eller en server som må vedlikeholdes kontinuerlig. Kildekoden, lisensiert under MIT, ligger på [GitHub](https://github.com/pfstr/newsletter-template).

[![Deploy to Cloudflare](../images/serverloser-newsletter-cloudflare-workers-d1/deploy-to-cloudflare.svg)](https://deploy.workers.cloudflare.com/?url=https://github.com/pfstr/newsletter-template)

![Malens hostede påmeldingsskjema](../images/serverloser-newsletter-cloudflare-workers-d1/newsletter-template-signup.png)

## Hva malen kan gjøre

- **Påmelding**: en hostet påmeldingsside, et innebyggbart skjema for ditt eget nettsted og et JSON-endepunkt
- **Avmelding med ett klikk**: i samsvar med RFC 8058, med individuelt token per abonnent
- **Påkrevde opplysninger innebygd**: Hver e-post får automatisk en bunntekst med avmeldingslenke og postadresse; tidspunkter for samtykke og avmelding lagres
- **Utsending**: På en beskyttet side kan du angi emne og HTML, sende en test-e-post og sette kampanjen i kø; en bakgrunnsjobb sender i batcher og gjentar mislykkede forsøk
- **Egne data**: Abonnentene ligger i en D1-database på din konto og kan eksporteres når som helst
- **Valgfritt, deaktivert som standard**: Double opt-in, botbeskyttelse med Turnstile og automatisk utsending av nye blogginnlegg fra RSS-feeden

## Arkitektur: én Worker, én database

Hele systemet er én enkelt Cloudflare Worker med to handlere: `fetch` for HTTP (rutet med Hono) og `scheduled` for cron-utløseren, i tillegg til en D1-database. Det finnes ingen annen tjeneste, ingen separat kømegler og ingen egen admin-backend; selv utsendingskøen er bare en D1-tabell.

| Rute | Funksjon |
| --- | --- |
| `GET /` | Hostet påmeldingsside |
| `GET /embed` | Gjennomsiktig skjema som kan bygges inn via iframe |
| `POST /api/subscribe` | Påmelding (CORS-åpen for eget nettsted) |
| `GET /confirm` | Bekreftelseslenke ved double opt-in |
| `GET/POST /unsubscribe` | Avmelding: bekreftelsesside via GET, utførelse via POST (ett klikk etter RFC 8058) |
| `GET /admin` | Utsendingsside (skjema) |
| `POST /api/send` | Sett kampanje i kø, beskyttet med admin-token |

Datamodellen omfatter fire tabeller: `subscribers` (e-post som primærnøkkel, navn, status, avmeldings- og bekreftelsestoken, en JSON-kolonne for selvdefinerte tilleggsfelt samt tidsstempler for bekreftelse og avmelding), `campaigns` med emne, innhold og tellere per utsending, `outbox` som utsendingskø (én rad per mottaker) og `sent_posts` for deduplisering av RSS-utsendinger.

## Distribusjon uten kommandolinje

Den mest interessante delen er ikke koden, men veien til et kjørende system. Deploy-to-Cloudflare-knappen leser Wrangler-konfigurasjonen i repositoriet og utfører hele oppsettet: Den kloner repositoriet til din egen GitHub-konto, klargjør D1-databasen, kjører skjemamigreringene og setter opp CI, slik at hver push distribueres automatisk. Siden juli 2025 spør deploy-flyten også etter miljøvariabler og hemmeligheter direkte i skjemaet: i dette malens tilfelle administratorpassordet (`ADMIN_TOKEN`), avsendernavn og -adresse, double-opt-in-bryteren og størrelsen på utsendingsbatchen (`SEND_BATCH`).

Resultatet etter ett klikk og ett skjema: Påmeldingssiden er live på `https://<worker-name>.workers.dev` og samler abonnenter. Du åpner aldri en terminal.

## Samle abonnenter

For integrering på ditt eget nettsted finnes det tre måter, i stigende grad av integrasjon. Den enkleste er å dele lenken til den hostede påmeldingssiden. Den mest praktiske for nettstedbyggere (WordPress, Webflow, Squarespace, Framer) er en iframe på én linje i en valgfri HTML-embed-blokk.

```html
<iframe
  src="https://<worker-name>.workers.dev/embed"
  style="width:100%;max-width:420px;height:90px;border:0"
></iframe>
```

Hvis du vil ha skjemaet i ditt eget design, poster du direkte til endepunktet:

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

Skjemaet samler som standard inn e-post og valgfritt navn. Du definerer flere felt (firma, land, …) i én enkelt fil (`src/fields.ts`); de vises automatisk i begge skjemaene og lagres som JSON i databasen.

## Utsending: egen leverandør i stedet for innebygd tjeneste

For e-postutsending tar malen et bevisst valg: Den er **leverandøruavhengig**. Filen `src/email.ts` inneholder én enkelt `sendEmail()`-adapter med et kommentert eksempel for et generisk HTTP-API. Hvilken utsendingstjeneste du kobler til der, er ditt valg. Ingen leverandør er hardkodet, og ingen registrering hos en bestemt tjeneste forutsettes. Innsamling av abonnenter fungerer allerede fullt ut uten utsendingskonfigurasjon; utsending aktiveres så snart adapteren er implementert og leverandørhemmeligheten er satt. Hvis leverandøren også tilbyr et batch-endepunkt (ett API-kall, mange e-poster), kan en valgfri `sendEmailBatch()`-adapter legges til i samme fil; også for dette finnes et kommentert eksempel.

Utsendingen betjenes via siden `/admin`: Lim inn emne og e-post-HTML, send en test til din egen adresse, og sett deretter kampanjen i kø for alle abonnenter. I e-postene er merge-taggene `{{unsubscribe_url}}`, `{{email}}` og `{{name}}` tilgjengelige.

Selve utsendingen skjer i bakgrunnen, etter Transactional-Outbox-mønsteret: `POST /api/send` skriver kampanjen og én rad per mottaker til databasen og svarer umiddelbart. En cron-jobb hvert minutt leverer deretter `SEND_BATCH` e-poster per kjøring, som standard 40: valgt slik at hver kjøring holder seg innenfor subrequest-grensene i Workers Free-planen. Radene reserveres atomisk, så overlappende kjøringer aldri kan sende dobbelt; mislykkede leveringer gjentas opptil tre ganger, og avbrutte kjøringer tas opp igjen etter ti minutter. Og den som melder seg av mens e-posten fortsatt står i kø, mottar den ikke: Opt-out avbryter også meldinger som allerede er satt i kø.

## Avmelding og dokumentasjon er en del av kjernen

Den som sender et nyhetsbrev, er underlagt lover om anti-spam og personvern: den amerikanske CAN-SPAM Act, GDPR og ePrivacy i EU, og UWG i Sveits. En vesentlig del av det du betaler nyhetsbrevtjenester for, er nettopp oppfyllelse av disse kravene. Malen håndterer den mekaniske delen:

- **Påkrevd bunntekst**: Hver kampanje-e-post får automatisk en bunntekst med fungerende avmeldingslenke og avsenderens postadresse (`SENDER_ADDRESS`); CAN-SPAM krever en fysisk adresse i kommersielle e-poster. Utsendingssiden advarer så lenge adressen mangler.
- **List-Unsubscribe-header etter RFC 8058** ved hver utsending: den innebygde avmeldingsknappen i Gmail og Outlook, som Gmail og Yahoo har krevd av masseutsendere siden 2024. Appen setter sammen headerne ferdig; din egen leverandøradapter sender dem bare videre.
- **Skannersikker avmelding**: Avmeldingslenken fører til en bekreftelsesside med én enkelt knapp. Bedrifts-e-postskannere som henter hver lenke i en e-post på forhånd, kan dermed ikke melde noen av ved en feil; e-postklienter bruker One-Click-POST direkte.
- **Dataminimering og dokumentasjon**: En opt-out trer i kraft umiddelbart, sletter navn og tilleggsfelt og lagres med tidsstempel, i likhet med påmelding og double-opt-in-bekreftelse. Samtykket kan dermed dokumenteres i ettertid (GDPRs ansvarlighetsprinsipp).
- **Personvernlenke**: Med satt `PRIVACY_URL` vises en lenke til din egen personvernerklæring under påmeldingsskjemaet.

Operatøren må fortsatt bruke sannferdige avsender- og emnelinjer, bare sende til adresser som faktisk er påmeldt, og håndtere domeneautentisering (SPF/DKIM/DMARC) hos utsendingstjenesten. Dette er ikke juridisk rådgivning.

## Valg: Double opt-in, Turnstile, RSS-automatisering

Tre funksjoner er innebygd, men deaktivert som standard slik at systemet forblir kjørbart uten konfigurasjon:

- **Double opt-in** (`DOUBLE_OPT_IN = "true"`): Nye abonnenter lagres som `pending` og blir først aktive etter et klikk på en bekreftelseslenke. For Sveits (DSG) og EU er denne fremgangsmåten det ryddigere valget.
- **Botbeskyttelse** med Cloudflare Turnstile: Sett site- og secret-key som variabler, ikke mer; widgeten vises automatisk på begge skjemaene, og Workeren verifiserer hver påmelding på serversiden. Uten gyldig token avvises påmeldingen.
- **Automatisk RSS-utsending**: En cron-jobb sjekker din egen bloggfeed (RSS 2.0 eller Atom) hvert 15. minutt og setter nye artikler automatisk i utsendingskøen. To sikringer er innebygd: Ved aller første kjøring markeres den eksisterende feeden bare som baseline (arkivet sendes altså ikke ut som en e-postflom), og hver artikkel-ID lagres i `sent_posts`, slik at ingen innlegg sendes ut to ganger.

## Begrensninger

Malen er bevisst holdt minimal. Køutsendingen leverer som standard rundt 40 e-poster per minutt i Free-planen; en kampanje til 1 000 mottakere tar dermed omtrent 25 minutter, noe som ikke spiller noen rolle for et nyhetsbrev. I den betalte Workers-planen (10 000 subrequests per kall i stedet for 50) kan `SEND_BATCH` økes til flere hundre; med en batch-adapter (ett API-kall, opptil rundt 1 000 e-poster) sender også Free-planen store lister på få minutter. Leveringsevnen avhenger, som i alle systemer, av ditt eget avsenderdomene: SPF, DKIM og DMARC må være verifisert hos den valgte utsendingstjenesten, ellers havner nyhetsbrevet i søppelposten. Og standardvalget med single opt-in er den enkleste starten, men ikke den mest konservative compliance-varianten; til det finnes bryteren.

Når det gjelder kostnader: Workers og D1 har romslige kvoter i gratisnivået (blant annet 100 000 forespørsler per dag), som et påmeldingsskjema og ukentlige utsendinger til en liten til mellomstor liste ikke bruker opp. Hvis en grense nås, begrenser Cloudflare i Free-planen i stedet for å sende en regning.

## Prøv det

Kildekoden, inkludert deploy-knappen, ligger på [GitHub](https://github.com/pfstr/newsletter-template); der finner du også den komplette dokumentasjonen av konfigurasjonsvariablene.

[![GitHub: pfstr/newsletter-template](../images/serverloser-newsletter-cloudflare-workers-d1/github-newsletter-template.svg)](https://github.com/pfstr/newsletter-template)

## Kilder

1.  [pfstr/newsletter-template](https://github.com/pfstr/newsletter-template): Malens kildekode (MIT) med deploy-knapp og dokumentasjon.

2.  [Deploy to Cloudflare buttons](https://developers.cloudflare.com/workers/platform/deploy-buttons/): automatisk klargjøring av ressurser, repo-kloning og CI ved deploy.

3.  [Deploy buttons: environment variables and secrets](https://developers.cloudflare.com/changelog/post/2025-07-01-workers-deploy-button-supports-environment-variables-and-secrets/): Hemmeligheter og variabler hentes inn i deploy-skjemaet siden juli 2025.

4.  [Cloudflare D1](https://developers.cloudflare.com/d1/): serverløs SQLite, her for abonnenter, sendeloggen og RSS-deduplisering.

5.  [Cloudflare Turnstile](https://developers.cloudflare.com/turnstile/): Botbeskyttelse uten CAPTCHA-gåter, kan valgfritt aktiveres i malen.

6.  [RFC 8058](https://datatracker.ietf.org/doc/html/rfc8058): Signaling One-Click Functionality for List Email Headers; grunnlaget for den innebygde avmeldingsknappen i Gmail og Outlook.

7.  [Workers limits](https://developers.cloudflare.com/workers/platform/limits/): Subrequest-grenser per kall (50 i Free-planen, 10 000 i den betalte planen); dette danner grunnlaget for batch-størrelsen ved køutsending.

8.  [FTC: CAN-SPAM Act Compliance Guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business): Krav for kommersielle e-poster, blant annet postadresse og fungerende opt-out.
