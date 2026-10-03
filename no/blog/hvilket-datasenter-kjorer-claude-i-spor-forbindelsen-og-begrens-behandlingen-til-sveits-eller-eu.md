---
title: "Hvilket datasenter kjører Claude i? Spor forbindelsen og begrens behandlingen til Sveits eller EU"
navTitle: "Claude-datasenter"
description: "Hvilke stasjoner en forespørsel til Claude passerer, hvorfor sporet ender ved Cloudflare-edge, og hva Anthropic opplyser om sine datasentre. I tillegg sammenlignes veier med fast behandlingssted (Anthropic API, AWS Bedrock, Google Vertex AI, Microsoft Foundry), begrensningene for drift kun i Sveits og hva regional binding koster."
date: "2026-10-02"
kategorie: "Claude"
timeToRead: "12 min lesetid"
themen:
  - claude
produkte:
  - "claude"
protokolle:
  - "apis"
  - "tcp"
slug: "hvilket-datasenter-kjorer-claude-i-spor-forbindelsen-og-begrens-behandlingen-til-sveits-eller-eu"
translationId: "article-d8b7299589ea96f4"
aiPrompt: |
  Du bist mein Berater für Datenstandorte bei KI-Diensten. Hilf mir Schritt für Schritt zu entscheiden, über welchen Weg (Anthropic API, AWS Bedrock, Google Vertex AI oder Microsoft Foundry) wir Claude nutzen sollen, wenn die Verarbeitung in der Schweiz oder in der EU bleiben muss. Frage mich zuerst nach Anwendungsfall, Datenklassifizierung, benötigtem Modell, erwartetem Token-Volumen pro Monat und bestehendem Cloud-Anbieter. Berechne danach die Monatskosten für den globalen und den regional gebundenen Endpunkt und nenne die konkreten Konfigurationsschritte (Region, Inference Profile, Endpoint, Kontrollmöglichkeiten im Log).
translationOf: claude-rechenzentrum-schweiz-eu
url: https://rafaelpfister.ch/no/blog/hvilket-datasenter-kjorer-claude-i-spor-forbindelsen-og-begrens-behandlingen-til-sveits-eller-eu
translationSourceHash: dd2606a3781871ddf851af8ceaa22336d00316661e82ca7089325e078f709dc4
translationModel: gpt-5.6-terra
translatedAt: 2026-10-03T09:20:56.208Z
translationReview: automatic
---

Når du bruker Claude via claude.ai, Claude Code eller API-et, får du ikke vite hvilket datasenter modellen behandler forespørselen i. Bare den første delen av forbindelsen frem til nærmeste edge-node er synlig. For virksomheter som overfører personopplysninger eller konfidensielt innhold til en språkmodell, er dette ikke nok: De må kunne dokumentere overfor kunder, personvernrådgivere eller revisjonen hvor behandlingen finner sted.

Denne artikkelen viser hvor langt forbindelsen kan spores på egen hånd, hva som er offentlig kjent om Anthropics datasentre, og hvilke veier som gjør det mulig å begrense behandlingen bindende til EU eller området EU pluss Sveits. Prisene og regionene er per 2. oktober 2026.

**Sveits-fokus:** Den reviderte personvernloven (revDSG) er avgjørende. Utlevering av personopplysninger til utlandet er tillatt etter art. 16 revDSG dersom Forbundsrådet har attestert tilstrekkelig vern i mottakerstaten (vedlegg 1 til personvernforordningen, DSV), eller det finnes egnede garantier som standard kontraktsklausuler. EU- og EØS-statene står på denne listen, mens USA kun gjør det for selskaper som er sertifisert under Swiss-U.S. Data Privacy Framework. Et behandlingssted i EU gjør derfor begrunnelsen betydelig enklere, men erstatter ikke de øvrige pliktene (databehandleravtale, informasjon til de registrerte, datasikkerhet).

**EU-merknad:** For virksomheter med sete i EU gjelder i stedet art. 44 flg. i personvernforordningen (GDPR). EU-kommisjonen har attestert at Sveits har et tilstrekkelig personvernnivå (tilstrekkelighetsbeslutning 2000/518/EF, bekreftet i januar 2024); behandling i Zürich er dermed ikke en problematisk overføring til tredjeland for EU-virksomheter.

## Hvilke stasjoner en forespørsel til Claude passerer

En forespørsel til `claude.ai` eller `api.anthropic.com` går gjennom tre stasjoner:

| Stasjon | Hvem driver den | Synlig for deg? |
|---|---|---|
| Edge-node (TLS-terminering, vern mot misbruk) | Cloudflare, på vegne av Anthropic | Ja, plasseringen kan leses av via header |
| Internt nett til backend | Cloudflare og Anthropic | Nei |
| Inferens (modellen beregner) | Anthropic, på datakraft fra AWS, Google Cloud og egne datasentre | Nei, kun grov angivelse `us` eller `global` |

Vertsnavnene `claude.ai` og `api.anthropic.com` peker begge til adressen `160.79.104.10`. Den tilhører prefikset `160.79.104.0/23`, som er registrert på Anthropic (AS399358) og ifølge RPKI er gyldig fra Anthropic. I global ruting annonseres prefikset imidlertid utelukkende via Cloudflare (AS13335): Anthropic bringer sine egne IP-adresser inn i Cloudflares Anycast-nettverk. Dermed besvares samme adresse over hele verden fra hvert Cloudflare-sted, og forbindelsen havner ved nærmeste edge-node.

## Fastslå edge-noden selv

Cloudflare oppgir betjenende sted på en diagnostikkside og i headeren `cf-ray`. Forkortelsen bak Ray-ID-en er en IATA-flyplasskode: `ZRH` står for Zürich, `GVA` for Genève, `FRA` for Frankfurt, `MRS` for Marseille.

```bash
curl -s https://api.anthropic.com/cdn-cgi/trace
```

De relevante linjene i utdataene er `colo=` (edge-plassering) og `loc=` (landet Cloudflare tilordner avsenderadressen din). I Windows gir PowerShell det samme:

```powershell
$trace = Invoke-WebRequest -Uri "https://api.anthropic.com/cdn-cgi/trace" -UseBasicParsing
$trace.Content
```

Du leser headeren med en HEAD-forespørsel:

```bash
curl -sI https://api.anthropic.com/ \
  | grep -i -E '^(server|cf-ray):'
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `-s` | Undertrykker fremdriftsvisning og feilmeldinger fra curl |
| `-I` | Sender en HEAD-forespørsel og skriver kun ut svar-headerne |
| `grep -i` | Søker uten hensyn til store og små bokstaver |
| `-E '^(server\|cf-ray):'` | Utvidet regulært uttrykk: kun linjer som begynner med `server:` eller `cf-ray:` |

</details>

Fra et sveitsisk nettverk er det vanligvis forventet `ZRH` eller `GVA`. Dersom internettutgangen er et annet sted, for eksempel i et bedriftsnett med sentral utgang i utlandet, svarer edge-noden der, for eksempel med `colo=FRA` eller `colo=MRS`. Edge-plasseringen følger altså nettutgangen din, ikke behandlingsstedet.

## Hvorfor traceroute ender ved edge

En traceroute viser veien frem til edge-noden og ikke videre:

```powershell
Test-NetConnection -ComputerName api.anthropic.com -TraceRoute -Hops 20
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `-ComputerName api.anthropic.com` | Målhvert for målingen |
| `-TraceRoute` | Finner rutinghoppene på veien til målet |
| `-Hops 20` | Maksimalt antall hopp som kontrolleres |

</details>

I Linux og macOS tilsvarer dette `traceroute -n api.anthropic.com` (`-n` undertrykker DNS-oppløsning av hoppene). I målingen lå de siste hoppene før `160.79.104.10` i adresseområdet `162.158.0.0/15`, som tilhører Cloudflare. Deretter ender sporet: Forbindelsen termineres ved edge-noden, og videresendingen til backend skjer i en ny, intern forbindelse som ikke er synlig utenfra.

RIPEstat viser hvem som eier prefikset og hvilket nettverk det annonseres gjennom:

```bash
curl -s "https://stat.ripe.net/data/prefix-overview/data.json?resource=160.79.104.10"
curl -s "https://stat.ripe.net/data/bgp-state/data.json?resource=160.79.104.0/23"
```

I det andre resultatet ender alle AS-stier på `13335 399358`: Cloudflare er den eneste upstreamen for Anthropic-prefikset.

Etter min undersøkelse finnes det ingen offentlig måling som sporer veien helt til et inferensdatasenter, og på grunn av termineringen ved edge er det heller ikke mulig med nettverksverktøy. Måletjenester som llmlatency.dev måler kun svartiden til første byte ved edge (per september 2026: median 96 ms fra US-Central, 199 ms fra Tyskland). Tiden til første token (Time to First Token) kunne riktignok sammenlignes fra flere steder, men den avhenger mer av modell, promptlengde og belastning enn av avstand. Det er derfor ikke grunnlag for å trekke en pålitelig slutning om plasseringen.

## Hva som er kjent om Anthropics datasentre

Anthropic driver foreløpig ikke et eget, offentlig tilordnet inferensdatasenter med adresse. Datakraften kommer fra partnerskap som Tim Cadenbach har samlet i TCDEV-bloggen:

| Sted / partner | Nøkkeldata | Kjent rolle |
|---|---|---|
| Project Rainier, New Carlisle (Indiana, USA), Amazon Web Services | Åpnet oktober 2025, rundt 11 mrd. USD, omtrent 500 000 Trainium2-brikker, over 2,2 GW planlagt | Anthropic er hovedleietaker; hovedsakelig trening |
| Google Cloud | Avtale i oktober 2025 om opptil 1 mill. TPU-er, over 1 GW fra 2026 | Regioner ikke offentliggjort |
| Egne datasentre med Fluidstack | Annonsert november 2025, 50 mrd. USD, steder i Texas og New York, idriftsettelse fra 2026 | Fortsatt under oppbygging |

Trening og inferens skjer ikke nødvendigvis på samme sted. Inferens kan kjøre i vilkårlige AWS- eller Google Cloud-regioner, og Anthropic opplyser ikke hvilken lokasjon som beregner svaret for en bruker i Sveits. Det eneste sikre er det personvernerklæringen fastslår: Avtalepart og behandlingsansvarlig for kunder fra EØS, Storbritannia og Sveits er Anthropic Ireland, Limited i Dublin; dataene overføres til servere i USA eller andre land utenfor EØS, basert på standard kontraktsklausuler.

## Styringen direkte hos Anthropic: kun USA eller globalt

Claude API kjenner parameteren `inference_geo` siden modellene i generasjon 4.6. Den har nøyaktig to verdier:

| Verdi | Virkning | Pris |
|---|---|---|
| `global` (standard) | Inferens i en hvilken som helst tilgjengelig region | Listepris |
| `us` | Inferens utelukkende i USA | Listepris × 1,1 |

Det finnes ingen verdi for EU eller Sveits. Lagringsstedet for arbeidsområdet (`workspace geo`) kan for tiden også bare settes til `us`. Svaret oppgir i feltet `usage.inference_geo` hvilken innstilling som ble brukt, men ingen konkret region. For Haiku 4.5 og eldre modeller svarer API-et med feil 400 på parameteren.

For claude.ai og Claude Code med abonnement (Pro, Max, Team, Enterprise) finnes det ingen innstilling for behandlingssted. Gjennom Anthropics direkte tilbud kan behandlingen dermed verken begrenses til EU eller Sveits. Dette er kun mulig via skyleverandørene som driver Claude i sine egne regioner.

## AWS Bedrock: området EU pluss Sveits fra Zürich

AWS Bedrock driver Claude på AWS-infrastruktur. Ifølge AWS har Anthropic ikke tilgang til kundenes prompt, svar eller logger. Regionen velges via kilderegionen for API-kallet og via Inference Profile:

| Endepunkttype | Prefiks for modell-ID | Behandlingssted |
|---|---|---|
| Global Cross-Region Inference | `global.` | Valgfri kommersiell AWS-region over hele verden |
| Geographic Cross-Region Inference | `eu.` | Kun regioner innenfor geografien |
| In-Region | uten prefiks | Kun regionen som kalles opp |

For Sveits er regionen `eu-central-2` (Zürich) avgjørende. Opus 5.5, Sonnet 5.5 og Haiku 4.5 er tilgjengelige der, men bare som Global- eller EU-profil og ikke In-Region. En profil som utelukkende beregner i Zürich, finnes ikke. Hvis EU-profilen kalles opp fra Zürich, fordeler AWS ifølge modellkortet (dokumentert for Haiku 4.5 og Sonnet 4.6) forespørslene på disse regionene:

| Region | Sted |
|---|---|
| `eu-central-2` | Zürich |
| `eu-central-1` | Frankfurt |
| `eu-north-1` | Stockholm |
| `eu-south-1` | Milano |
| `eu-south-2` | Spania |
| `eu-west-1` | Irland |
| `eu-west-3` | Paris |

Zürich er bare et mulig mål dersom forespørselen kommer fra Zürich. Behandlingen forblir dermed i området EU pluss Sveits. For Opus 5.5 og Sonnet 5.5 oppgir ikke modellkortet målregionene; API-et returnerer den faktiske listen:

```bash
aws bedrock get-inference-profile \
  --region eu-central-2 \
  --inference-profile-identifier eu.anthropic.claude-sonnet-5-5 \
  --query "models[].modelArn"
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `bedrock get-inference-profile` | Leser definisjonen av en Inference Profile |
| `--region eu-central-2` | Kilderegion Zürich; mållisten avhenger av kilderegionen |
| `--inference-profile-identifier` | ID for EU-profilen for Sonnet 5.5 |
| `--query "models[].modelArn"` | Skriver kun ut modell-ARN-ene; regionen står i hvert ARN |

</details>

Data lagres som standard kun i kilderegionen. Et unntak er innhold som holdes tilbake for misbruksdeteksjon: Det lagres i målregionen. Transporten mellom regionene skjer kryptert over AWS-nettverket.

### Dokumenter behandlingsregionen per forespørsel

Bedrock er den eneste av de beskrevne veiene som logger den faktiske behandlingsregionen per forespørsel. CloudTrail skriver den i kilderegionen i feltet `additionalEventData.inferenceRegion`:

```bash
aws cloudtrail lookup-events \
  --region eu-central-2 \
  --lookup-attributes AttributeKey=EventSource,AttributeValue=bedrock.amazonaws.com \
  --max-results 20 \
  --query "Events[].CloudTrailEvent" \
  --output text \
  | jq -r '[.eventTime, .eventName, .additionalEventData.inferenceRegion] | @tsv'
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `cloudtrail lookup-events` | Gjennomsøker CloudTrail-hendelsene de siste 90 dagene |
| `--region eu-central-2` | Kilderegionen der kallene logges |
| `--lookup-attributes AttributeKey=EventSource,AttributeValue=bedrock.amazonaws.com` | Filtrerer etter hendelser fra Bedrock |
| `--max-results 20` | Begrenser utdataene til 20 hendelser |
| `--query "Events[].CloudTrailEvent"` | Skriver kun ut hele hendelsen som JSON-tekst |
| `--output text` | Utdata uten JSON-innpakning, én hendelse per linje |
| `jq -r '… \| @tsv'` | Henter ut tidspunkt, handling og behandlingsregion som en tabulatorseparert linje |

</details>

I tillegg kan en Service Control Policy (SCP) i AWS Organizations sperre alle regioner utenfor EU og Sveits. Dersom en målregion i en profil er sperret, mislykkes forespørselen i stedet for å falle tilbake til en annen region. Med denne kombinasjonen er behandlingsstedet teknisk håndhevet og dokumenterbart per forespørsel.

### Claude Code via Bedrock i EU

Claude Code kan også drives via Bedrock. I en `eu-*`-region velger Claude Code prefikset `eu.` automatisk:

```bash
export CLAUDE_CODE_USE_BEDROCK=1
export AWS_REGION=eu-central-2
claude
```

| Variabel | Virkning |
|---|---|
| `CLAUDE_CODE_USE_BEDROCK=1` | Bytter Claude Code fra Anthropic API til Bedrock |
| `AWS_REGION=eu-central-2` | Kilderegion Zürich; dette medfører EU-profilen |

Med `ANTHROPIC_BEDROCK_REGION_PREFIX` kan prefikset overstyres. AWS-autentisering skjer gjennom vanlige mekanismer (profil, SSO, miljøvariabler). Fakturering skjer via AWS-kontoen, ikke via et Claude-abonnement.

## Google Vertex AI: kun EU, uttrykkelig uten Sveits

På Google Cloud (Vertex AI, nå under navnet Gemini Enterprise Agent Platform) er situasjonen mindre gunstig for Sveits:

| Endepunkt | Modeller (utvalg) | Behandlingssted |
|---|---|---|
| `global` | alle | Valgfri Google Cloud-region, uten garanti |
| Multiregion `eu` (`aiplatform.eu.rep.googleapis.com`) | Opus 5.5, Sonnet 5.5, Opus 5, Sonnet 5, Opus 4.8, Opus 4.7 | Kun EU-medlemsstater |
| Region `europe-west1` (Belgia) | Haiku 4.5, Sonnet 4.6, Opus 4.6 og eldre | Ifølge modellsiden multiregion Europa |
| Region `europe-west6` (Zürich) | ingen | Claude ikke tilgjengelig |

Google utelukker uttrykkelig Sveits fra EU-multiregion-endepunktet: Det dekker bare EU-medlemsstater; Storbritannia og Sveits er ikke omfattet. Flere sveitsiske veiledningssider nevner `europe-west6` som en vei til Claude i Zürich; ifølge Googles offisielle stedstabell er ingen Claude-modeller tilgjengelige der. Google dokumenterer ikke et loggfelt med den faktiske behandlingsregionen per forespørsel. For det globale endepunktet fastslår Google at behandlingsregionen verken kan styres eller fastslås.

Vertex AI egner seg derfor for behandling utelukkende i EU, men ikke for behandling i Sveits.

## Microsoft Foundry: foreløpig ingen EU-løsning

Microsoft tilbyr Claude i Foundry i to varianter. «Hosted on Azure» beregner på Azure-infrastruktur, som Global Standard eller Data Zone Standard; Data Zone finnes bare for Claude i USA. «Hosted on Anthropic» beregner på Anthropic-infrastruktur, og Microsoft opplyser at data kan behandles utenfor Azure og utenfor den valgte regionen. Anthropic oppfører Foundry for Europa som «Coming soon». I Microsoft 365 Copilot og Copilot Studio er Anthropic-modellene unntatt fra EU Data Boundary og er som standard deaktivert i EU, EFTA (også i Sveits) og Storbritannia.

## Oversikt: hvilken vei garanterer hvilket behandlingssted

| Vei | Kun Sveits | Kun EU | Området EU pluss Sveits | Region per forespørsel dokumenterbar |
|---|---|---|---|---|
| claude.ai, Claude Code (abonnement) | nei | nei | nei | nei |
| Claude API med `inference_geo` | nei | nei | nei (kun `us`) | kun `us`/`global` |
| AWS Bedrock, kilde `eu-central-2`, profil `eu.` | nei | nei | ja | ja (CloudTrail) |
| AWS Bedrock, kilde i EU, profil `eu.` | nei | ja | ja | ja (CloudTrail) |
| Google Vertex AI, endepunkt `eu` | nei | ja | (kun EU-delen) | nei |
| Microsoft Foundry | nei | nei | nei | nei |

Ingen leverandør tilbyr for tiden en vei som garanterer at Claudes behandling begrenses til Sveits. AWS Bedrock med kilderegion Zürich og EU-profil kommer nærmest. Dersom det i tillegg må utelukkes at data forlater EU, for eksempel på grunn av kontraktsmessige løfter til EU-kunder, kan EU-profilen kalles opp fra en EU-region som Frankfurt; Zürich er da ikke lenger et mål.

## Hva regional binding koster

Alle tre skyleverandørene og Anthropic selv krever et tillegg på 10 % sammenlignet med det globale endepunktet for binding til en region eller geografi. Prisene i Bedrock følger kilderegionen; Zürich, Frankfurt og North Virginia koster det samme. Ruting mellom regioner koster ikke ekstra.

Listepriser i USD per 1 mill. token (input / output):

| Modell | Anthropic API, `global` | Anthropic API, `us` | Bedrock eller Vertex, global | Bedrock `eu.` eller Vertex `eu` |
|---|---|---|---|---|
| Opus 5.5 | 4.00 / 20.00 | 4.40 / 22.00 | 4.00 / 20.00 | 4.40 / 22.00 |
| Sonnet 5.5 | 2.00 / 10.00 | 2.20 / 11.00 | 2.00 / 10.00 | 2.20 / 11.00 |
| Haiku 4.5 | 1.00 / 5.00 | ikke tilgjengelig | 1.00 / 5.00 | 1.10 / 5.50 |
| Sonnet 4.6 | 3.00 / 15.00 | 3.30 / 16.50 | 3.00 / 15.00 | 3.30 / 16.50 |

For Vertex AI gjelder prisen i EU-kolonnen for Haiku 4.5 og Sonnet 4.6 i regionen `europe-west1`. Et regneeksempel for en intern applikasjon med 50 mill. input- og 10 mill. output-token per måned:

| Modell | Globalt | EU-bundet | Ekstrakostnad per måned |
|---|---|---|---|
| Sonnet 5.5 | 50 × 2 + 10 × 10 = 200 USD | 220 USD | 20 USD |
| Opus 5.5 | 50 × 4 + 10 × 20 = 400 USD | 440 USD | 40 USD |

Regional binding koster altså 10 % mer enn det globale endepunktet. Sammenlignet med listeprisen direkte hos Anthropic med `inference_geo: "us"` er en EU-profil på Bedrock like dyr. Andre kostnader har større betydning ved valget: En AWS- eller Google Cloud-konto med organisasjonsregler, logging og budsjettvarsler må bygges opp og driftes, og de faste månedsprisene for Claude-abonnementene faller bort til fordel for ren forbruksavregning. Om byttet lønner seg, avhenger av volumet; et abonnement har fast pris per person, men gir ingen kontroll over behandlingsstedet.

## Oppbevaring og trening

I tillegg til stedet teller det hvor lenge data lagres:

- **Anthropic API:** Prompt og svar brukes ikke til trening så lenge det ikke foreligger uttrykkelig samtykke. Zero Data Retention (ZDR) er tilgjengelig på forespørsel per organisasjon og kan kombineres med `inference_geo`. Innhold som misbruksdeteksjonen markerer, lagres også med ZDR i opptil to år. For modellene Fable og Mythos er oppbevaring i 30 dager obligatorisk.
- **AWS Bedrock:** Innhold sendes ikke til Anthropic. Oppbevaring kan velges per modell mellom `none`, `default` og `aws_review`; Fable krever `aws_review` med opptil 30 dagers lagring for gjennomgang hos AWS.
- **Google Vertex AI:** Google bruker kun dataene til trening eller finjustering med forhåndssamtykke.

## Kilder

1.  [Claude Platform Docs: Data residency](https://platform.claude.com/docs/en/manage-claude/data-residency): Parameter `inference_geo`, verdiene `us` og `global`, arbeidsområdets geo, støttede modeller, feltet `usage.inference_geo`.

2.  [Claude Platform Docs: Pricing](https://platform.claude.com/docs/en/about-claude/pricing): Listepriser for Anthropic API og faktor 1,1 for `inference_geo: "us"`.

3.  [Claude Platform Docs: API and data retention](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention): Zero Data Retention, unntak for markert innhold og for Fable/Mythos.

4.  [Anthropic: Privacy Policy](https://www.anthropic.com/legal/privacy): Anthropic Ireland som behandlingsansvarlig for EØS, Storbritannia og Sveits, overføring til USA basert på standard kontraktsklausuler.

5.  [Anthropic: Regional compliance](https://claude.com/regional-compliance): Oversikt over hvilken plattform som tilbyr dataresidens i hvilken region; Foundry Europa som «Coming soon».

6.  [Claude Platform Docs: Claude in Amazon Bedrock](https://platform.claude.com/docs/en/build-with-claude/claude-in-amazon-bedrock): Endepunkttyper per region, 10 % tillegg for regionale endepunkter.

7.  [AWS: Modellkort Claude Sonnet 4.6](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-4-6.html): Målregioner for EU-profilen per kilderegion, inkludert Zürich.

8.  [AWS: Modellkort Claude Sonnet 5.5](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-5-5.html): Tilgjengelighet i `eu-central-2` kun som Geo- og Global-profil.

9.  [AWS: Geographic cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/geographic-cross-region-inference.html): Data forblir innenfor geografien, lagringssted, sperring av målregioner via SCP.

10.  [AWS: Cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html): Feltet `additionalEventData.inferenceRegion` i CloudTrail, pris etter kilderegion.

11.  [AWS: Data protection in Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/data-protection.html): Modellleverandørene har ikke tilgang til prompt, svar og logger.

12.  [AWS: Amazon Bedrock Pricing](https://aws.amazon.com/bedrock/pricing/): Listepriser for Global- og Geo-profiler, identiske for Zürich, Frankfurt og North Virginia.

13.  [AWS Alps Blog: Cross-region inference for EU data processing in Switzerland](https://aws.amazon.com/blogs/alps/unlocking-ai-flexibility-in-switzerland-a-guide-to-cross-region-inference-for-eu-data-processing-and-model-access/): AWS Sveits' veiledning om EU-profilen fra Zürich.

14.  [Claude Code Docs: Amazon Bedrock](https://code.claude.com/docs/en/amazon-bedrock): Miljøvariabler og automatisk prefiks `eu.` i EU-regioner.

15.  [Google Cloud: Generative AI locations](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/locations): Stedstabell for Claude-modellene, ingen tilgjengelighet i `europe-west6`.

16.  [Google Cloud: Data residency](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/data-residency): EU-multiregion kun for EU-medlemsstater, Sveits og Storbritannia er utelukket.

17.  [Google Cloud: Vertex AI Pricing](https://cloud.google.com/vertex-ai/generative-ai/pricing): Listepriser globalt, EU-multiregion og `europe-west1`.

18.  [Microsoft Learn: Claude models hosting comparison](https://learn.microsoft.com/en-us/azure/foundry/foundry-models/concepts/claude-models-hosting-comparison): Hosted on Azure og Hosted on Anthropic, Data Zone kun USA.

19.  [Microsoft Learn: Anthropic as AI subprocessor in Microsoft 365 Copilot](https://learn.microsoft.com/en-us/microsoft-365/copilot/connect-to-ai-subprocessor): Unntak fra EU Data Boundary, standardinnstilling i EU/EFTA.

20.  [TCDEV Blog: Where Are Claude's Data Centers?](https://www.tcdev.de/blog/where-are-claudes-data-centers/): Tim Cadenbachs oversikt over Project Rainier, Google-TPU-avtalen og egne datasentre med Fluidstack.

21.  [Anthropic: Investering på 50 mrd. USD i amerikansk infrastruktur](https://www.anthropic.com/news/anthropic-invests-50-billion-in-american-ai-infrastructure): Annonsering av egne datasentre i Texas og New York med Fluidstack.

22.  [Claude Platform Docs: IP addresses](https://platform.claude.com/docs/en/api/ip-addresses): Innkommende adresseområde `160.79.104.0/23`.

23.  [RIPEstat](https://stat.ripe.net/): Prefiksoversikt og BGP-status for `160.79.104.0/23` (AS399358, upstream AS13335).

24.  [llmlatency.dev: Anthropic](https://llmlatency.dev/provider/anthropic): Svartider ved edge etter måleplassering.

25.  [Fedlex: Personvernforordningen (DSV), vedlegg 1](https://www.fedlex.admin.ch/eli/cc/2022/568/de): Liste over stater med tilstrekkelig personvern, inkludert EU/EØS og USA for sertifiserte selskaper.
