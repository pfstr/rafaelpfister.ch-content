---
title: "Opptil 10 millioner gratis tokens per dag: Slik bruker du OpenAIs datadelingsprogram med kostnadsvern"
navTitle: "OpenAI-gratis-tokens"
description: "OpenAI gir organisasjoner som deler API-trafikken sin til trening, en daglig gratiskvote: opptil 10 millioner tokens, avhengig av tier. Med forhåndsbetalt saldo, prosjektgrenser og et tokenbudsjett i koden forblir bruken gratis over tid."
date: "2026-08-27"
kategorie: "OpenAI API"
timeToRead: "9 min lesetid"
themen:
  - openai-api
produkte:
  - "openai"
protokolle:
  - "apis"
  - "lizenzierung"
slug: "opptil-10-millioner-gratis-tokens-per-dag-slik-bruker-du-openais-datadelingsprogram-med"
translationId: "article-dde41cbe2dd858e6"
aiPrompt: |
  Du bist mein Assistent für die OpenAI-Plattform. Prüfe mit mir Schritt für Schritt, ob mein OpenAI-Konto für das Data-Sharing-Programm mit Gratis-Tokens sauber abgesichert ist: 1) Billing: Prepaid-Guthaben statt Rechnung, Auto-Reload aus. 2) Data controls → Sharing: "Share inputs and outputs" nur für ein dediziertes Projekt aktiviert, Enrollment-Hinweis sichtbar. 3) Projekt: eigenes Spend-Limit gesetzt, nur ein restricted API-Key. 4) Limits: Spend-Alerts konfiguriert. 5) Code: tägliches Token-Budget deutlich unter Gratis-Kontingent und Tages-Rate-Limit. Frage mich nach meinem Usage-Tier und Modell und rechne mir mein Gratis-Kontingent aus.
translationOf: openai-gratis-tokens-data-sharing
url: https://rafaelpfister.ch/no/blog/opptil-10-millioner-gratis-tokens-per-dag-slik-bruker-du-openais-datadelingsprogram-med
translationSourceHash: 0f0fef78a8ab264b755061045a34cc765916b1f1b433473f99a5eb6e0538a6b0
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:44:57.787Z
translationReview: automatic
---

# Opptil 10 millioner gratis tokens per dag: Slik bruker du OpenAIs datadelingsprogram med kostnadsvern

OpenAI betaler for treningsdata med datakraft i stedet for penger: Organisasjoner som deler API-inndata og -utdata til trening, har siden desember 2024 fått en daglig kvote med gratis tokens. Avhengig av usage tier og modellgruppe er dette mellom 250'000 og 10 millioner tokens per dag. For mange små automatiseringer er dette tilstrekkelig: En nattlig batch-oversettelse, en klassifiserings-Cronjob eller automatisk tagging av et offentlig arkiv forblir dermed gratis over tid.

For at det skal forbli gratis, trengs det grenser – på riktig sted. En token-teller i egen kode er en bekvemmelighetsfunksjon; det er bare grensene som OpenAI selv håndhever som er bindende.

## Programmet: Tokens mot treningsdata

Deltakelse skjer via innstillingen **Share inputs and outputs with OpenAI** under *Settings → Data controls → Sharing*. Den kan bare endres av Organization Owner, enten for hele organisasjonen eller for enkelte prosjekter. De som kvalifiserer for programmet, ser meldingen "You're eligible for free daily usage on traffic shared with OpenAI" på denne siden; etter aktivering endres den til "You're enrolled for complimentary daily tokens". Hvis meldingen mangler, er organisasjonen for øyeblikket ikke kvalifisert til å delta. Kontoer med Zero Data Retention og Enterprise-avtaler er utelukket fra deling av inndata og utdata.

Kvoten avhenger av organisasjonens usage tier og beregnes per modellgruppe:

| Modellgruppe | Tier 1–2 | Tier 3–5 |
|---|---|---|
| Store modeller (bl.a. gpt-5.6-sol, gpt-5.x, o-serien, gpt-4.1, gpt-4o) | 250'000 tokens/dag | 1 mill. tokens/dag |
| Små modeller (bl.a. gpt-5.6-terra, gpt-5.6-luna, mini- og nano-varianter) | 2,5 mill. tokens/dag | 10 mill. tokens/dag |

De viktigste reglene i detalj:

- Inndata- og utdata-tokens telles samlet, fordelt på alle modeller i en gruppe. Telleren nullstilles daglig kl. 00:00 UTC.
- Finjusterte modeller, finjusteringstrening, evals og verktøybruk er unntatt.
- Kontoen trenger en positiv saldo, ellers fungerer heller ikke gratis-tokenene.
- OpenAI forbeholder seg retten til å avslutte programmet med 30 dagers varsel.

Den viktigste faktureringsregelen: Requesten som overskrider dagskvoten, faktureres **i sin helhet** til ordinær pris, ikke bare den overskytende delen. Den som står på 975'000 av 1 million tokens og sender en request på 30'000 tokens, betaler for alle 30'000. For egen budsjettplanlegging betyr det: Legg inn en sikkerhetsmargin, ikke optimaliser helt opp mot kvoten.

## Hva du gir fra deg

Motytelsen er utvetydig: Alle inndata og utdata fra de delte prosjektene går til OpenAI og kan brukes til trening av fremtidige modeller. Dermed faller hele kategorier av bruksområder bort. Kundedata, supporthenvendelser, interne dokumenter, kode med konfigurasjonsdetaljer og alt som kan knyttes til personer, må ikke havne i et delt prosjekt; for sveitsiske selskaper setter allerede revDSG grensen her, før konfidensialiteten overfor kunder i det hele tatt kommer på tale.

Velegnet er arbeidslaster over data som uansett er offentlige. Et eksempel er nattlig oversettelse av en offentlig blogg til flere språk: Artiklene ligger på nettet, enhver crawler leser dem allerede i dag, og oversettelsene publiseres også. I et slikt tilfelle avslører delingen ingenting som ikke allerede er offentliggjort. Andre kandidater er alttekster for et offentlig bildearkiv, tagging av Open Source-dokumentasjon eller sammendrag av offentlige Release Notes til en changelog.

## Sett opp kostnadsvern i OpenAI-kontoen

Rekkefølgen er bevisst valgt: Først kommer grensene som OpenAI håndhever på serversiden. De gjelder også dersom egen kode har en feil, en Cronjob kjører dobbelt eller en key havner i feil hender.

**Forhåndsbetalt saldo, Auto-Reload av.** Sett billing til "Pay as you go" med forhåndsbetalt saldo, og deaktiver automatisk påfylling. Dermed er maksimal skade begrenset til gjenstående saldo: Når den er brukt opp, avviser API-et flere requests. Siden programmet forutsetter en positiv saldo, trengs en liten grunnsum; 5 til 10 dollar er nok og forblir urørt ved korrekt drift. Dette er det eneste tiltaket som faktisk stopper alt i verste fall, og derfor kommer det først.

**Et dedikert prosjekt for den delte trafikken.** Sett delingen til "Enabled for selected projects" og del bare et prosjekt opprettet spesifikt for dette formålet. Alle andre prosjekter i organisasjonen forblir unntatt fra trening, og utilsiktet trafikk fra andre applikasjoner havner verken i treningsdatasettet eller i feil budsjett.

**Sett prosjektets spend limit lavt.** Prosjekter har sin egen månedlige spend limit, og den er hard: Requests feiler så snart den er nådd. For et prosjekt som etter planen koster 0 dollar, kan den være svært lav; 5 dollar er nok som reserve dersom én enkelt kjøring overskrider gratiskvoten. Grensen på organisasjonsnivå er derimot ment som en øvre grense med varsler; varseltersklene (for eksempel ved 90 og 100 prosent) utløser e-poster.

**Én restricted key per prosjekt, kun som CI-secret.** API-keyen opprettes i prosjektet, ikke på organisasjonsnivå, og får bare tillatelsene arbeidslasten trenger. For en CI-workflow betyr det: nøyaktig én key med begrensede rettigheter, lagret som secret i CI-miljøet. Den forekommer ikke i noe repository, ingen lokal shell og ingen annen tjeneste.

**Velg en modell fra den rimelige gruppen.** Forskjellen mellom gruppene er en faktor på 10. Den som arbeider i Tier 1, har 2,5 millioner tokens per dag med en modell fra den lille gruppen, i stedet for 250'000. For strukturerte oppgaver som oversettelse, klassifisering eller ekstraksjon er den lille gruppen som regel tilstrekkelig.

## Den andre forsvarslinjen i koden

Kontogrensene forhindrer økonomisk skade, men de fører til harde feil: En nådd spend limit avbryter kjøringen midt i batchen. Den som vil holde seg ryddig innenfor gratiskvoten, kan derfor i tillegg telle selv. En enkel dagteller har vist seg å fungere godt, konfigurert for eksempel slik:

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

Mekanismen bak består av fire regler:

- Etter hvert svar legger jobben sammen `input_tokens` og `output_tokens` rapportert av API-et i en teller i en state-fil. Det finnes ingen estimering og ingen andre forespørsler, bare usage-opplysningene fra selve svaret.
- Før hver request kontrollerer den restbudsjettet. Hvis det ikke lenger sikkert er nok til et fullstendig svar, avsluttes kjøringen normalt med stoppgrunnen `token-budget` i stedet for med en feil.
- Telleren arbeider med UTC-kalenderdager og er dermed synkronisert med nullstillingen av gratiskvoten kl. 00:00 UTC.
- Uavhengig av budsjettet er antall API-kall per kjøring begrenset, slik at heller ikke en serie mislykkede forsøk kan bruke opp kvoten. Transport- og kvotefeil avbryter kjøringen uten automatisk gjentakelse.

Budsjettet i dette eksempelet ligger med 1 million bevisst klart under kvoten på 2,5 millioner. Avstanden følger av to særegenheter ved faktureringen. For det første kjenner ikke telleren størrelsen på neste request på forhånd; et knapt beregnet budsjett kan derfor overskrides med størrelsen på én request, og nettopp denne requesten ville da bli fakturert i sin helhet etter regelen beskrevet ovenfor. For det andre ligger rate limits per dag (TPD) avhengig av tier og modell under gratiskvoten; et budsjett over TPD-grensen ville aldri blitt nådd normalt, fordi API-et først avviser med HTTP 429.

## Kontroll: Dashboardet må vise 0.00

Om regnestykket går opp, vises i plattformens Usage-dashboard. To visninger er nok:

- **Usage**-visningen teller alle tokens, også de som faktureres gratis. Her står arbeidslastens samlede forbruk.
- **Costs**-visningen (og feltet "Monthly spend" i prosjektlisten) viser bare betalte tokens. Her må det permanent stå 0.00.

De som vil vite mer nøyaktig, kan gruppere Usage-visningen etter *Service tier*: Gratis fakturerte tokens vises der som en egen post, "data sharing incentive tier". En kalenderoppføring én gang i måneden for å se på dashboardet fullfører kjeden av guardrails, for OpenAI kan avslutte programmet med 30 dagers varsel, og fra den dagen ville samme arbeidslast fortsette til ordinær pris.

## Kilder

1.  [OpenAI Help Center: Sharing feedback, evaluation and fine-tuning data, and API inputs and outputs](https://help.openai.com/en/articles/10306912-sharing-feedback-evaluation-and-fine-tuning-data-and-api-inputs-and-outputs-with-openai): autoritativ programbeskrivelse med modellgrupper, tier-kvoter, UTC-nullstilling og faktureringsregelen for requests som overskrider kvoten.

2.  [OpenAI Developer Community: Extended: Free tokens on traffic shared with OpenAI](https://community.openai.com/t/good-news-extended-free-tokens-on-traffic-shared-with-openai/1241322): kunngjøring av programforlengelsen i april 2025 med løftet om 30 dagers varsel.

3.  [OpenAI Platform: Data sharing settings](https://platform.openai.com/settings/organization/data-controls/sharing): opt-in-bryter og enrollment-status for egen organisasjon (innlogging kreves).

4.  [OpenAI Platform: Rate limits guide](https://platform.openai.com/docs/guides/rate-limits): forklaring av TPM-, RPM- og TPD-grensene som gjelder i tillegg til gratiskvoten.

5.  [OpenAI Platform: Pricing](https://platform.openai.com/docs/pricing): ordinære priser som overskridelser av kvoten faktureres til.
