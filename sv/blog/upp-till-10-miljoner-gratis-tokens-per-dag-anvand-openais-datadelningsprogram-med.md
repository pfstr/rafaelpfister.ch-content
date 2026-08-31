---
title: "Upp till 10 miljoner gratis tokens per dag: använd OpenAIs datadelningsprogram med kostnadsspärrar"
navTitle: "OpenAI-gratis-tokens"
description: "OpenAI ger organisationer som delar sin API-trafik för träning en daglig gratiskvot: beroende på nivå upp till 10 miljoner tokens. Med förbetalt saldo, projektgränser och en tokenbudget i koden förblir användningen gratis på lång sikt."
date: "2026-08-27"
kategorie: "OpenAI API"
timeToRead: "9 min läsning"
themen:
  - openai-api
produkte:
  - "openai"
protokolle:
  - "apis"
  - "lizenzierung"
slug: "upp-till-10-miljoner-gratis-tokens-per-dag-anvand-openais-datadelningsprogram-med"
translationId: "article-dde41cbe2dd858e6"
aiPrompt: |
  Du bist mein Assistent für die OpenAI-Plattform. Prüfe mit mir Schritt für Schritt, ob mein OpenAI-Konto für das Data-Sharing-Programm mit Gratis-Tokens sauber abgesichert ist: 1) Billing: Prepaid-Guthaben statt Rechnung, Auto-Reload aus. 2) Data controls → Sharing: "Share inputs and outputs" nur für ein dediziertes Projekt aktiviert, Enrollment-Hinweis sichtbar. 3) Projekt: eigenes Spend-Limit gesetzt, nur ein restricted API-Key. 4) Limits: Spend-Alerts konfiguriert. 5) Code: tägliches Token-Budget deutlich unter Gratis-Kontingent und Tages-Rate-Limit. Frage mich nach meinem Usage-Tier und Modell und rechne mir mein Gratis-Kontingent aus.
translationOf: openai-gratis-tokens-data-sharing
url: https://rafaelpfister.ch/sv/blog/upp-till-10-miljoner-gratis-tokens-per-dag-anvand-openais-datadelningsprogram-med
translationSourceHash: 0f0fef78a8ab264b755061045a34cc765916b1f1b433473f99a5eb6e0538a6b0
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:44:30.439Z
translationReview: automatic
---

# Upp till 10 miljoner gratis tokens per dag: använd OpenAIs datadelningsprogram med kostnadsspärrar

OpenAI betalar för träningsdata med beräkningskapacitet i stället för pengar: Organisationer som delar sina API-in- och utdata för träning får sedan december 2024 en daglig kvot med gratis tokens. Beroende på Usage Tier och modellgrupp handlar det om mellan 250'000 och 10 miljoner tokens per dag. För många små automatiseringar räcker detta helt och hållet: en nattlig batchöversättning, ett klassificerings-Cron-jobb eller automatisk taggning av ett offentligt arkiv förblir därmed gratis på lång sikt.

För att det ska förbli gratis behövs gränser, och de måste finnas på rätt ställe. En tokenräknare i den egna koden är en bekvämlighetsfunktion; endast de gränser som OpenAI själv upprätthåller är bindande.

## Programmet: tokens mot träningsdata

Deltagandet hanteras via inställningen **Share inputs and outputs with OpenAI** under *Settings → Data controls → Sharing*. Endast Organization Owner kan ändra den, antingen för hela organisationen eller för enskilda projekt. Den som kvalificerar sig för programmet ser meddelandet "You're eligible for free daily usage on traffic shared with OpenAI" på denna sida; efter aktivering ändras det till "You're enrolled for complimentary daily tokens". Om meddelandet saknas är organisationen för närvarande inte behörig att delta. Konton med Zero Data Retention och Enterprise-avtal är undantagna från input-output-delning.

Kvoten beror på organisationens Usage Tier och beräknas per modellgrupp:

| Modellgrupp | Tier 1–2 | Tier 3–5 |
|---|---|---|
| Stora modeller (bl.a. gpt-5.6-sol, gpt-5.x, o-serien, gpt-4.1, gpt-4o) | 250'000 tokens/dag | 1 miljon tokens/dag |
| Små modeller (bl.a. gpt-5.6-terra, gpt-5.6-luna, mini- och nano-varianter) | 2,5 miljoner tokens/dag | 10 miljoner tokens/dag |

De viktigaste reglerna i detalj:

- Input- och output-tokens räknas tillsammans och delas mellan alla modeller i en grupp. Räknaren återställs dagligen klockan 00:00 UTC.
- Finjusterade modeller, finjusteringsträning, Evals och verktygsanvändning är undantagna.
- Kontot behöver ha ett positivt saldo, annars fungerar inte heller gratistokensen.
- OpenAI förbehåller sig rätten att avsluta programmet med 30 dagars varsel.

Den viktigaste faktureringsregeln: Requesten som överskrider dagskvoten debiteras **helt och hållet** till ordinarie pris, inte bara den överskridande delen. Den som ligger på 975'000 av 1 miljon tokens och skickar en request med 30'000 tokens betalar för alla 30'000. För den egna budgetplaneringen innebär det: planera med en säkerhetsmarginal, inte för att maximera kvoten.

## Vad man lämnar ifrån sig

Motprestationen är otvetydig: Alla in- och utdata i de delade projekten skickas till OpenAI och kan användas för träning av framtida modeller. Därmed faller hela kategorier av användningsfall bort. Kunddata, supportärenden, interna dokument, kod med konfigurationsdetaljer och allt som innehåller personuppgifter får inte hamna i ett delat projekt; för schweiziska företag sätter redan revDSG gränsen här, innan konfidentialiteten gentemot kunder ens kommer på tal.

Lämpliga är arbetslaster med data som ändå är offentliga. Ett exempel är nattlig översättning av en offentlig blogg till flera språk: Artiklarna finns på nätet, varje crawler kan redan läsa dem i dag, och översättningarna publiceras också. Delningen avslöjar i ett sådant fall inget som inte redan är offentligt. Andra kandidater är alttexter för ett offentligt bildarkiv, taggning av open source-dokumentation eller sammanfattningar av offentliga release notes för en changelog.

## Konfigurera kostnadsspärrar i OpenAI-kontot

Ordningen är medvetet vald: Först kommer gränserna som OpenAI upprätthåller på serversidan. De fungerar även om den egna koden innehåller ett fel, ett Cron-jobb körs dubbelt eller en nyckel hamnar i fel händer.

**Förbetalt saldo, Auto-Reload av.** Ställ in faktureringen på "Pay as you go" med förbetalt saldo och inaktivera automatisk påfyllning. Därmed begränsas den maximala skadan till återstående saldo: När det är förbrukat avvisar API:t ytterligare requests. Eftersom programmet förutsätter ett positivt saldo behövs en liten grundsumma; 5 till 10 dollar räcker och förblir orörda vid korrekt drift. Detta steg är det enda som verkligen stoppar allt i värsta fall, därför kommer det först.

**Ett dedikerat projekt för den delade trafiken.** Ställ delningen på "Enabled for selected projects" och dela endast ett projekt som skapats specifikt för detta ändamål. Alla andra projekt i organisationen förblir undantagna från träning, och oavsiktlig trafik från andra applikationer hamnar varken i träningsdatasetet eller i fel budget.

**Sätt projektets utgiftsgräns lågt.** Projekt har en egen månatlig Spend Limit, och den är hård: requests misslyckas så snart den har nåtts. För ett projekt som enligt plan kostar 0 dollar kan den vara mycket låg; 5 dollar räcker som reserv om en enskild körning överskrider gratiskvoten. Gränsen på organisationsnivå är däremot tänkt som en övre gräns med alerts; varningströsklarna, exempelvis vid 90 och 100 procent, utlöser e-postmeddelanden.

**En restricted key per projekt, endast som CI-secret.** API-nyckeln skapas i projektet, inte på organisationsnivå, och får endast de behörigheter som arbetslasten behöver. För ett CI-workflow innebär det: exakt en nyckel med begränsade rättigheter, lagrad som en secret i CI-miljön. Den förekommer inte i något repository, något lokalt shell eller någon annan tjänst.

**Välj en modell ur den billiga gruppen.** Skillnaden mellan grupperna är en faktor 10. Den som arbetar i Tier 1 har 2,5 miljoner tokens per dag med en modell från den lilla gruppen i stället för 250'000. För strukturerade uppgifter som översättning, klassificering eller extraktion räcker den lilla gruppen i regel.

## Den andra försvarslinjen i koden

Kontogränserna förhindrar ekonomisk skada, men de leder till hårda fel: En uppnådd Spend Limit avbryter körningen mitt i batchen. Den som vill hålla sig strikt inom gratiskvoten kan därför även räkna själv. En enkel daglig räknare har visat sig fungera väl, konfigurerad exempelvis så här:

```json
{
  "openai": {
    "model": "gpt-5.6-terra",
    "reasoningEffort": "none",
    "maxOutputTokens": 32000,
    "dailyTokenBudget": 1000000
  }
}
```

Mekanismen bakom består av fyra regler:

- Efter varje svar lägger jobbet till de `input_tokens` och `output_tokens` som API:t rapporterar till en räknare i en state-fil. Det sker ingen uppskattning och ingen andra förfrågan, endast användningsuppgifterna från svaret självt.
- Före varje request kontrollerar det återstående budget. Om den inte längre säkert räcker för ett fullständigt svar avslutas körningen normalt med stopporsaken `token-budget` i stället för med ett fel.
- Räknaren arbetar med UTC-kalenderdagar och är därmed synkroniserad med återställningen av gratiskvoten klockan 00:00 UTC.
- Oberoende av budgeten begränsas antalet API-anrop per körning, så att inte heller en serie misslyckade försök kan förbruka kvoten. Transport- och kvotfel avbryter körningen utan automatisk upprepning.

Budgeten i detta exempel ligger med 1 miljon medvetet tydligt under kvoten på 2,5 miljoner. Avståndet följer av två särdrag i faktureringen. För det första känner räknaren inte storleken på nästa request i förväg; en snävt beräknad budget kan därför överskridas med storleken på en request, och just den requesten skulle enligt regeln ovan debiteras helt och hållet. För det andra ligger rate limits per dag (TPD), beroende på Tier och modell, under gratiskvoten; en budget över TPD-gränsen skulle aldrig nås normalt eftersom API:t först avvisar med HTTP 429.

## Kontroll: Dashboarden måste visa 0.00

Om kalkylen går ihop visar plattformens Usage-dashboard. Två vyer räcker:

- Vyn **Usage** räknar alla tokens, även de som debiteras gratis. Här visas arbetslastens totala förbrukning.
- Vyn **Costs** (och fältet "Monthly spend" i projektlistan) visar endast betalda tokens. Här måste 0.00 visas permanent.

Den som vill veta mer i detalj kan gruppera Usage-vyn efter *Service tier*: Gratisdebiterade tokens visas där som en egen post, "data sharing incentive tier". En kalenderpåminnelse en gång i månaden för att kontrollera dashboarden sluter kedjan av skyddsräcken, eftersom OpenAI kan avsluta programmet med 30 dagars varsel och samma arbetslast från den dagen skulle fortsätta till ordinarie pris.

## Källor

1.  [OpenAI Help Center: Sharing feedback, evaluation and fine-tuning data, and API inputs and outputs](https://help.openai.com/en/articles/10306912-sharing-feedback-evaluation-and-fine-tuning-data-and-api-inputs-and-outputs-with-openai): den auktoritativa programbeskrivningen med modellgrupper, Tier-kvoter, UTC-återställning och faktureringsregeln för överskridande requests.

2.  [OpenAI Developer Community: Extended: Free tokens on traffic shared with OpenAI](https://community.openai.com/t/good-news-extended-free-tokens-on-traffic-shared-with-openai/1241322): tillkännagivandet av programförlängningen i april 2025 med löftet om 30 dagars varsel.

3.  [OpenAI Platform: Data sharing settings](https://platform.openai.com/settings/organization/data-controls/sharing): opt-in-reglage och enrollment-status för den egna organisationen (inloggning krävs).

4.  [OpenAI Platform: Rate limits guide](https://platform.openai.com/docs/guides/rate-limits): förklaring av TPM-, RPM- och TPD-gränserna som gäller utöver gratiskvoten.

5.  [OpenAI Platform: Pricing](https://platform.openai.com/docs/pricing): ordinarie priser som tillämpas vid överskridning av kvoten.
