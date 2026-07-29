---
title: "Driva ett eget nyhetsbrev med Cloudflare Workers och D1"
navTitle: "Nyhetsbrev på Workers"
description: "Den öppna mallen tillhandahåller prenumeration, avregistrering, kö och databas i ditt eget Cloudflare-konto. En deploy-knapp konfigurerar Worker, D1 och CI utan lokal server."
date: "2026-07-22"
kategorie: "Cloudflare Workers"
timeToRead: "8 min läsning"
themen:
  - cloudflare-workers
slug: "driva-ett-eget-nyhetsbrev-med-cloudflare-workers-och-d1"
translationOf: "serverloser-newsletter-cloudflare-workers-d1"
url: "https://rafaelpfister.ch/sv/blog/driva-ett-eget-nyhetsbrev-med-cloudflare-workers-och-d1"
translationId: article-4e7139acdb90923b
translationReview: automatic
translationSourceHash: 90c100386e148f80be4d4be81dc928f373431ce83b5f6e2336cfb0daafd3945e
translatedAt: 2026-07-29T12:29:38.957Z
---

# Driva ett eget nyhetsbrev med Cloudflare Workers och D1

Med en hostad nyhetsbrevstjänst ligger mottagarlistan hos leverantören, och kostnaderna ökar ofta med antalet prenumeranter. En egen server ger mer kontroll, men medför löpande arbete: uppdateringar, övervakning, säkerhetskopior och drift av ett system som kanske bara skickar en gång i veckan.

För detta slimmade användningsfall räcker HTTP-ändpunkter, en liten databas och ett tidsstyrt utskicksjobb. Cloudflare Workers och D1 tillhandahåller just dessa byggstenar. Min öppna mall konfigurerar dem i ditt eget konto via en **Deploy-to-Cloudflare-knapp**. Ingen lokal kommandorad eller server som kräver löpande underhåll behövs. Källkoden med MIT-licens finns på [GitHub](https://github.com/pfstr/newsletter-template).

[![Deploy to Cloudflare](../images/serverloser-newsletter-cloudflare-workers-d1/deploy-to-cloudflare.svg)](https://deploy.workers.cloudflare.com/?url=https://github.com/pfstr/newsletter-template)

![Mallens hostade prenumerationsformulär](../images/serverloser-newsletter-cloudflare-workers-d1/newsletter-template-signup.png)

## Vad mallen kan göra

- **Prenumeration**: en hostad prenumerationssida, ett inbäddningsbart formulär för den egna webbplatsen och en JSON-ändpunkt
- **Avregistrering med ett klick**: kompatibel med RFC 8058, med individuell token per prenumerant
- **Obligatorisk information inbyggd**: Varje e-postmeddelande får automatiskt en sidfot med avregistreringslänk och postadress; tidpunkter för samtycke och avregistrering sparas
- **Utskick**: På en skyddad sida kan ämne och HTML anges, ett testmejl skickas och kampanjen köas; ett bakgrundsjobb skickar i batchar och försöker igen vid misslyckade försök
- **Egna data**: Prenumeranter lagras i en D1-databas på ditt konto och kan när som helst exporteras
- **Valfritt, avstängt som standard**: Double opt-in, botskydd med Turnstile och automatiskt utskick av nya bloggartiklar från RSS-flödet

## Arkitektur: en Worker, en databas

Hela systemet är en enda Cloudflare Worker med två hanterare: `fetch` för HTTP (routad med Hono) och `scheduled` för Cron-utlösaren, plus en D1-databas. Det finns ingen andra tjänst, ingen separat kömäklare, ingen egen admin-backend; till och med utskickskön är bara en D1-tabell.

| Rutt | Funktion |
| --- | --- |
| `GET /` | Hostad prenumerationssida |
| `GET /embed` | Transparent formulär för inbäddning via iframe |
| `POST /api/subscribe` | Prenumeration (CORS-öppen för den egna webbplatsen) |
| `GET /confirm` | Bekräftelselänk vid double opt-in |
| `GET/POST /unsubscribe` | Avregistrering: bekräftelsesida via GET, utförande via POST (ett klick enligt RFC 8058) |
| `GET /admin` | Utskicksida (formulär) |
| `POST /api/send` | Lägg kampanj i kön, skyddad med admin-token |

Datamodellen omfattar fyra tabeller: `subscribers` (e-post som primärnyckel, namn, status, avregistrerings- och bekräftelsetoken, en JSON-kolumn för självdefinierade extrafält samt tidsstämplar för bekräftelse och avregistrering), `campaigns` med ämne, innehåll och räknare per utskick, `outbox` som utskickskö (en rad per mottagare) och `sent_posts` för deduplicering av RSS-utskick.

## Deployment utan kommandorad

Den mest intressanta delen är inte koden, utan vägen till ett fungerande system. Deploy-to-Cloudflare-knappen läser Wrangler-konfigurationen i repot och utför hela installationen: Den klonar repot till ditt eget GitHub-konto, provisionerar D1-databasen, kör schemamigreringarna och konfigurerar CI så att varje push automatiskt deployas. Sedan juli 2025 frågar deploy-flödet dessutom efter miljövariabler och hemligheter direkt i formuläret: i detta malls fall admin-lösenordet (`ADMIN_TOKEN`), avsändarnamn och -adress, double-opt-in-reglaget och storleken på utskicksbatchen (`SEND_BATCH`).

Resultatet efter ett klick och ett formulär: Prenumerationssidan är live på `https://<worker-name>.workers.dev` och samlar prenumeranter. Ingen terminal öppnas någonstans.

## Samla prenumeranter

För integration på den egna webbplatsen finns tre sätt, i stigande grad av integration. Det enklaste: dela länken till den hostade prenumerationssidan. Det mest praktiska för webbplatsbyggare (WordPress, Webflow, Squarespace, Framer): en iframe-rad i valfritt HTML-inbäddningsblock.

```html
<iframe
  src="https://<worker-name>.workers.dev/embed"
  style="width:100%;max-width:420px;height:90px;border:0"
></iframe>
```

Den som vill ha formuläret i sin egen design postar direkt till ändpunkten:

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

Formuläret samlar som standard in e-post och valfritt namn. Fler fält (företag, land, …) definierar du i en enda fil (`src/fields.ts`); de visas automatiskt i båda formulären och lagras som JSON i databasen.

## Utskick: egen leverantör i stället för inbyggd vendor

För e-postutskick gör mallen ett medvetet val: Den är **leverantörsagnostisk**. Filen `src/email.ts` innehåller en enda `sendEmail()`-adapter med ett kommenterat exempel för ett generiskt HTTP-API. Vilken utskickstjänst du ansluter där är ditt val. Ingen leverantör är hårdkodad, ingen registrering hos en viss tjänst förutsätts. Insamling av prenumeranter fungerar redan helt utan utskickskonfiguration; utskick aktiveras så snart adaptern är implementerad och leverantörshemligheten är inställd. Om leverantören dessutom erbjuder en batch-ändpunkt (ett API-anrop, många e-postmeddelanden) kan en valfri `sendEmailBatch()`-adapter läggas till i samma fil; även för detta finns ett kommenterat exempel.

Utskicken hanteras via sidan `/admin`: klistra in ämne och e-postens HTML, skicka ett test till din egen adress och köa sedan kampanjen för alla prenumeranter. I e-postmeddelandena finns merge-taggarna `{{unsubscribe_url}}`, `{{email}}` och `{{name}}` tillgängliga.

Själva utskicket sker i bakgrunden enligt Transactional-Outbox-mönstret: `POST /api/send` skriver kampanjen och en rad per mottagare till databasen och svarar omedelbart. Ett Cron-jobb varje minut levererar sedan `SEND_BATCH` e-postmeddelanden per körning, som standard 40: valt så att varje körning håller sig inom gränserna för subrequests i Workers Free-planen. Raderna tas i anspråk atomärt, så överlappande körningar kan aldrig skicka dubbelt; misslyckade leveranser försöks igen upp till tre gånger, och kraschade körningar återupptas efter tio minuter. Den som avregistrerar sig medan det egna e-postmeddelandet fortfarande ligger i kön får det inte längre: opt-out avbryter även redan köade meddelanden.

## Avregistrering och dokumentation är kärnfunktioner

Den som skickar ett nyhetsbrev omfattas av lagstiftning mot spam och för dataskydd: den amerikanska CAN-SPAM Act, GDPR och ePrivacy i EU, samt UWG i Schweiz. En väsentlig del av det man betalar nyhetsbrevstjänster för är just att uppfylla dessa krav. Mallen tar hand om den mekaniska delen:

- **Obligatorisk sidfot**: Varje kampanjmejl får automatiskt en sidfot med fungerande avregistreringslänk och avsändarens postadress (`SENDER_ADDRESS`); CAN-SPAM kräver en fysisk adress i kommersiella e-postmeddelanden. Utskicksidan varnar så länge adressen saknas.
- **List-Unsubscribe-header enligt RFC 8058** vid varje utskick: den inbyggda avregistreringsknappen i Gmail och Outlook, som Gmail och Yahoo kräver av massutskickare sedan 2024. Appen bygger ihop headern färdigt; din egen leverantörsadapter skickar bara vidare den.
- **Avregistrering säker mot skannrar**: Avregistreringslänken leder till en bekräftelsesida med en enda knapp. Företagsmejlskannrar som i förväg hämtar varje länk i ett e-postmeddelande kan därmed inte avregistrera någon av misstag; e-postklienter använder One-Click-POST direkt.
- **Dataminimering och bevis**: Ett opt-out får omedelbar verkan, raderar namn och extrafält och registreras med tidsstämpel, liksom prenumeration och double-opt-in-bekräftelse. Samtycket kan därmed styrkas i efterhand (GDPR:s ansvarsskyldighet).
- **Länk till integritetspolicy**: När `PRIVACY_URL` är inställd visas en länk till din egen integritetspolicy under prenumerationsformuläret.

Operatören ansvarar fortfarande för sanningsenliga avsändar- och ämnesrader, utskick endast till faktiskt anmälda adresser och domänautentisering (SPF/DKIM/DMARC) hos utskickstjänsten. Detta är inte juridisk rådgivning.

## Alternativ: double opt-in, Turnstile, RSS-automatik

Tre funktioner är inbyggda men avstängda som standard, så att systemet fungerar utan konfiguration:

- **Double opt-in** (`DOUBLE_OPT_IN = "true"`): Nya prenumeranter sparas som `pending` och blir aktiva först efter ett klick på en bekräftelselänk. För Schweiz (DSG) och EU är detta förfarande det mer korrekta valet.
- **Botskydd** med Cloudflare Turnstile: Ange bara webbplats- och hemlig nyckel som variabler; widgeten visas automatiskt i båda formulären och Workern verifierar varje prenumeration på serversidan. Utan giltig token nekas prenumerationen.
- **Automatiskt RSS-utskick**: Ett Cron-jobb kontrollerar den egna bloggens flöde (RSS 2.0 eller Atom) var 15:e minut och köar automatiskt nya artiklar för utskick. Två skydd är inbyggda: Vid den allra första körningen markeras det befintliga flödet bara som baslinje (arkivet skickas alltså inte som en e-postflod), och varje artikel-ID registreras i `sent_posts`, så att inget inlägg skickas två gånger.

## Begränsningar

Mallen är medvetet minimalistisk. Köutskicket levererar i Free-planen som standard omkring 40 e-postmeddelanden per minut; en kampanj till 1'000 mottagare tar därför omkring 25 minuter, vilket inte spelar någon roll för ett nyhetsbrev. I den betalda Workers-planen (10'000 subrequests per anrop i stället för 50) kan `SEND_BATCH` höjas till hundratals; med en batch-adapter (ett API-anrop, upp till omkring 1'000 e-postmeddelanden) skickar även Free-planen stora listor på några minuter. Leveransbarheten beror, som i alla system, på den egna avsändardomänen: SPF, DKIM och DMARC måste vara verifierade hos den valda utskickstjänsten, annars hamnar nyhetsbrevet i spam. Och single opt-in som standard är den enklaste starten, men inte den mest konservativa compliance-varianten; för det finns reglaget.

När det gäller kostnader har Workers och D1 generösa Free Tier-kvoter (bland annat 100'000 requests per dag), som ett prenumerationsformulär och veckovisa utskick till en liten eller medelstor lista inte förbrukar. Om en gräns nås stryper Cloudflare i Free-planen i stället för att skicka en faktura.

## Prova

Källkoden inklusive deploy-knapp finns på [GitHub](https://github.com/pfstr/newsletter-template); där finns även den fullständiga dokumentationen av konfigurationsvariablerna.

[![GitHub: pfstr/newsletter-template](../images/serverloser-newsletter-cloudflare-workers-d1/github-newsletter-template.svg)](https://github.com/pfstr/newsletter-template)

## Källor

1.  [pfstr/newsletter-template](https://github.com/pfstr/newsletter-template): Mallens källkod (MIT) med deploy-knapp och dokumentation.

2.  [Deploy to Cloudflare buttons](https://developers.cloudflare.com/workers/platform/deploy-buttons/): automatisk provisionering av resurser, repokloning och CI vid deploy.

3.  [Deploy buttons: environment variables and secrets](https://developers.cloudflare.com/changelog/post/2025-07-01-workers-deploy-button-supports-environment-variables-and-secrets/): Hemligheter och variabler efterfrågas i deploy-formuläret sedan juli 2025.

4.  [Cloudflare D1](https://developers.cloudflare.com/d1/): serverlös SQLite, här för prenumeranter, sändningslogg och RSS-deduplicering.

5.  [Cloudflare Turnstile](https://developers.cloudflare.com/turnstile/): botskydd utan CAPTCHA-gåtor, kan aktiveras valfritt i mallen.

6.  [RFC 8058](https://datatracker.ietf.org/doc/html/rfc8058): Signaling One-Click Functionality for List Email Headers; grund för den inbyggda avregistreringsknappen i Gmail och Outlook.

7.  [Workers limits](https://developers.cloudflare.com/workers/platform/limits/): gränser för subrequests per anrop (50 i Free-planen, 10'000 i den betalda planen); därifrån härleds batchstorleken för köutskicket.

8.  [FTC: CAN-SPAM Act Compliance Guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business): krav för kommersiella e-postmeddelanden, bland annat postadress och fungerande opt-out.
