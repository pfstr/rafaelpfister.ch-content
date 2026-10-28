---
title: "I vilket datacenter beräknar Claude? Spåra anslutningen och begränsa behandlingen till Schweiz eller EU"
navTitle: "Claude-datacenter"
description: "Vilka stationer en begäran till Claude passerar, varför spåret slutar vid Cloudflare-kanten och vad Anthropic offentliggör om sina datacenter. Dessutom en jämförelse av vägar med fast behandlingsplats (Anthropic API, AWS Bedrock, Google Vertex AI, Microsoft Foundry), begränsningarna för drift enbart i Schweiz och kostnaden för regional bindning."
date: "2026-10-02"
kategorie: "Claude"
timeToRead: "12 min lästid"
themen:
  - claude
produkte:
  - "claude"
protokolle:
  - "apis"
  - "tcp"
slug: "which-data-centre-processes-claude-trace-the-connection-and-restrict-processing-to-switzerland"
translationId: "article-d8b7299589ea96f4"
translationOf: claude-rechenzentrum-schweiz-eu
translationSourceHash: 2d2a3c4b5820e6ec99b710813e209ed3b2e6f6fa750028d120a7d5e4bf4e0c17
translationModel: gpt-5.6-terra
translatedAt: 2026-10-27T09:09:03.515Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/which-data-centre-processes-claude-trace-the-connection-and-restrict-processing-to-switzerland
---

När du använder Claude via claude.ai, Claude Code eller API:et får du inte veta i vilket datacenter modellen behandlar begäran. Det enda som syns är den första delen av anslutningen fram till närmaste Edge-nod. För företag som överlämnar personuppgifter eller konfidentiellt innehåll till en språkmodell är det för lite: de måste kunna styrka för kunder, dataskyddsrådgivare eller revisionen var behandlingen sker.

Den här artikeln visar hur långt anslutningen kan spåras, vad som är offentligt känt om Anthropics datacenter och via vilka vägar behandlingen kan begränsas bindande till EU eller området EU plus Schweiz. Priserna och regionerna gäller per den 25 oktober 2026.

**Schweiz-fokus:** Den reviderade dataskyddslagen (revDSG) är avgörande. Överlämnande av personuppgifter till utlandet är tillåtet enligt art. 16 revDSG om förbundsrådet intygar att destinationslandet har ett tillräckligt skydd (bilaga 1 till dataskyddsförordningen, DSV) eller om lämpliga garantier, såsom standardavtalsklausuler, finns. EU- och EES-staterna finns med på denna lista, USA endast för företag som är certifierade enligt Swiss-U.S. Data Privacy Framework. En behandlingsplats i EU förenklar därför motiveringen avsevärt, men ersätter inte övriga skyldigheter (personuppgiftsbiträdesavtal, information till de registrerade, datasäkerhet).

**EU-anmärkning:** För företag med säte i EU gäller i stället art. 44 ff. GDPR. EU-kommissionen har intygat att Schweiz har en adekvat dataskyddsnivå (adekvansbeslut 2000/518/EG, bekräftat i januari 2024); behandling i Zürich är därför inte en tredjelandsöverföring i problematisk mening för EU-företag.

<aside class="pilot-card">
  <span class="pilot-card__tag">Nytt från adeptio · Pilotföretag sökes</span>
  <p class="pilot-card__titel">Använd Claude och OpenAI utan att personuppgifter lämnar Schweiz</p>
  <p class="pilot-card__lead">Vårt mellanlager står mellan era applikationer i Schweiz och modellerna från Anthropic och OpenAI och anonymiserar varje begäran innan den lämnar landet. För drift i produktion söker vi företag som använder produkten i vardagen.</p>
  <div class="pilot-card__spalten">
    <div class="pilot-card__spalte">
      <p class="pilot-card__spalte-titel">Ert företag</p>
      <ul>
        <li>Personuppgifter lämnar Schweiz endast i anonymiserad form</li>
        <li>Enklare motivering inför dataskydd och revision</li>
        <li>Tillgång före allmän tillgänglighet och inflytande över funktionerna</li>
        <li>Direktkontakt med utvecklarna</li>
      </ul>
    </div>
    <div class="pilot-card__spalte">
      <p class="pilot-card__spalte-titel">adeptio</p>
      <ul>
        <li>Insikter från verkliga prompter i stället för testdata</li>
        <li>Mätvärden för latens och svarskvalitet under verklig belastning</li>
        <li>Referenser från den schweiziska marknaden</li>
      </ul>
    </div>
  </div>
  <div class="pilot-card__aktionen">
    <a class="pilot-card__cta" href="/kontakt">Anmäl er som pilotföretag</a>
    <a class="pilot-card__mehr" href="#anonymisierung-vor-dem-modell-pilotunternehmen-gesucht">Detaljer och rättslig bedömning</a>
  </div>
</aside>

## Vilka stationer en begäran till Claude passerar

En begäran till `claude.ai` eller `api.anthropic.com` passerar tre stationer:

| Station | Vem driver den | Synlig för dig? |
|---|---|---|
| Edge-nod (TLS-terminering, skydd mot missbruk) | Cloudflare, på uppdrag av Anthropic | Ja, platsen kan avläsas via header |
| Internt nät till backend | Cloudflare och Anthropic | Nej |
| Inferens (modellen beräknar) | Anthropic, på beräkningskapacitet från AWS, Google Cloud och egna datacenter | Nej, endast den grova uppgiften `us` eller `global` |

Värdnamnen `claude.ai` och `api.anthropic.com` pekar båda på adressen `160.79.104.10`. Den tillhör prefixet `160.79.104.0/23`, som är registrerat på Anthropic (AS399358) och enligt RPKI är giltigt utfärdat av Anthropic. I den globala routningen annonseras prefixet dock uteslutande via Cloudflare (AS13335): Anthropic för in sina egna IP-adresser i Cloudflares Anycast-nät. Samma adress besvaras därmed världen över från varje Cloudflare-plats, och anslutningen landar vid närmaste Edge-nod.

## Bestäm Edge-noden själv

Cloudflare avslöjar platsen som betjänar begäran på en diagnossida och i headern `cf-ray`. Förkortningen efter Ray-ID:t är en IATA-flygplatskod: `ZRH` står för Zürich, `GVA` för Genève, `FRA` för Frankfurt, `MRS` för Marseille.

```bash
curl -s https://api.anthropic.com/cdn-cgi/trace
```

I utdata är raderna `colo=` (Edge-plats) och `loc=` (landet som Cloudflare tilldelar din källadress) relevanta. I Windows ger PowerShell samma resultat:

```powershell
$trace = Invoke-WebRequest -Uri "https://api.anthropic.com/cdn-cgi/trace" -UseBasicParsing
$trace.Content
```

Du läser headern med en HEAD-begäran:

```bash
curl -sI https://api.anthropic.com/ \
  | grep -i -E '^(server|cf-ray):'
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-s` | Undertrycker curl:s förloppsvisning och felmeddelanden |
| `-I` | Skickar en HEAD-begäran och skriver endast ut svarsheaders |
| `grep -i` | Söker utan hänsyn till versaler och gemener |
| `-E '^(server\|cf-ray):'` | Utökat reguljärt uttryck: endast rader som börjar med `server:` eller `cf-ray:` |

</details>

Från ett schweiziskt nät kan man normalt förvänta sig `ZRH` eller `GVA`. Om internetutgången ligger på en annan plats, exempelvis i ett företagsnät med central utgång utomlands, svarar Edge-noden där, till exempel med `colo=FRA` eller `colo=MRS`. Edge-platsen följer alltså din nätutgång, inte behandlingsplatsen.

## Varför traceroute slutar vid Edge-noden

En traceroute visar vägen till Edge-noden och inte längre:

```powershell
Test-NetConnection -ComputerName api.anthropic.com -TraceRoute -Hops 20
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-ComputerName api.anthropic.com` | Mätningens målhost |
| `-TraceRoute` | Fastställer routerhoppen på vägen till målet |
| `-Hops 20` | Maximalt antal hopp som kontrolleras |

</details>

I Linux och macOS motsvaras detta av `traceroute -n api.anthropic.com` (`-n` undertrycker DNS-upplösning av hoppen). I mätningen låg de sista hoppen före `160.79.104.10` i adressområdet `162.158.0.0/15`, som tillhör Cloudflare. Därefter slutar spåret: anslutningen termineras vid Edge-noden, medan vidarebefordran till backend sker i en ny intern anslutning som inte är synlig utifrån.

RIPEstat visar vem prefixet tillhör och via vilket nät det annonseras:

```bash
curl -s "https://stat.ripe.net/data/prefix-overview/data.json?resource=160.79.104.10"
curl -s "https://stat.ripe.net/data/bgp-state/data.json?resource=160.79.104.0/23"
```

I det andra resultatet slutar alla AS-vägar på `13335 399358`: Cloudflare är den enda upstream-leverantören för Anthropics prefix.

En offentlig mätning som spårar vägen ända till ett inferensdatacenter finns enligt min research inte, och på grund av termineringen vid Edge-noden är den inte heller möjlig med nätverksverktyg. Mättjänster som llmlatency.dev registrerar endast svarstiden till första byte vid Edge-noden (september 2026: median 96 ms från US-Central, 199 ms från Tyskland). Tiden till första token (Time to First Token) skulle visserligen kunna jämföras från flera platser, men den beror mer på modell, promptlängd och belastning än på avståndet. En slutsats om en plats är därför inte tillförlitlig.

## Vad som är känt om Anthropics datacenter

Anthropic driver hittills inget eget, offentligt tillordnat inferensdatacenter med adress. Beräkningskapaciteten kommer från partnerskap som Tim Cadenbach har sammanställt i TCDEV-bloggen:

| Plats / partner | Nyckeldata | Känd roll |
|---|---|---|
| Project Rainier, New Carlisle (Indiana, USA), Amazon Web Services | Öppnat i oktober 2025, cirka 11 miljarder USD, omkring 500 000 Trainium2-chip, över 2,2 GW planerat | Anthropic är huvudhyresgäst; främst träning |
| Google Cloud | Avtal i oktober 2025 om upp till 1 miljon TPU:er, över 1 GW från 2026 | Regioner inte publicerade |
| Egna datacenter med Fluidstack | Tillkännagivna i november 2025, 50 miljarder USD, platser i Texas och New York, driftsättning från 2026 | Fortfarande under uppbyggnad |

Träning och inferens sker inte nödvändigtvis på samma plats. Inferensen kan köras i valfria AWS- eller Google Cloud-regioner, och Anthropic offentliggör inte vilken plats som beräknar svaret för en användare i Schweiz. Det enda som är säkerställt är vad integritetspolicyn anger: avtalspart och personuppgiftsansvarig för kunder från EES, Storbritannien och Schweiz är Anthropic Ireland, Limited i Dublin; uppgifterna överförs till servrar i USA eller andra länder utanför EES med stöd av standardavtalsklausuler.

## Styrning direkt hos Anthropic: endast USA eller globalt

Claude API har sedan modellerna i generation 4.6 parametern `inference_geo`. Den har exakt två värden:

| Värde | Effekt | Pris |
|---|---|---|
| `global` (standard) | Inferens i valfri tillgänglig region | Listpris |
| `us` | Inferens uteslutande i USA | Listpris × 1,1 |

Det finns inget värde för EU eller Schweiz. Inte heller Workspace-lagringsplatsen (`workspace geo`) kan för närvarande bara ställas in på `us`. Svaret anger i fältet `usage.inference_geo` vilken inställning som tillämpades, men ingen konkret region. För Haiku 4.5 och äldre modeller besvarar API:et parametern med fel 400.

För claude.ai och Claude Code med abonnemang (Pro, Max, Team, Enterprise) finns ingen inställning för behandlingsplats. Via Anthropics direkta erbjudanden kan behandlingen alltså varken begränsas till EU eller Schweiz. Det går endast via molnleverantörerna som driver Claude i sina egna regioner.

## AWS Bedrock: området EU plus Schweiz från Zürich

AWS Bedrock driver Claude på AWS-infrastruktur. Enligt AWS har Anthropic ingen åtkomst till kundernas prompter, svar eller loggar. Du väljer regionen via källregionen för ditt API-anrop och via Inference Profile:

| Ändpunktstyp | Prefix för modell-ID | Behandlingsplats |
|---|---|---|
| Global Cross-Region Inference | `global.` | Valfri kommersiell AWS-region i hela världen |
| Geographic Cross-Region Inference | `eu.` | Endast regioner inom geografin |
| In-Region | utan prefix | Endast den anropade regionen |

För Schweiz är regionen `eu-central-2` (Zürich) avgörande. Där finns Opus 5.5, Sonnet 5.5, Haiku 4.5 och sedan den 7 oktober 2026 Haiku 5.5 tillgängliga, dock endast som Global- eller EU-profil och inte In-Region. Det finns ingen profil som beräknar uteslutande i Zürich. Om EU-profilen anropas från Zürich fördelar AWS enligt modellkortet (belagt för Opus 5.5, Sonnet 5.5, Haiku 5.5, Haiku 4.5 och Sonnet 4.6) begärandena över dessa regioner:

| Region | Plats |
|---|---|
| `eu-central-2` | Zürich |
| `eu-central-1` | Frankfurt |
| `eu-north-1` | Stockholm |
| `eu-south-1` | Milano |
| `eu-south-2` | Spanien |
| `eu-west-1` | Irland |
| `eu-west-3` | Paris |

Zürich är endast ett möjligt mål när begäran kommer från Zürich. Behandlingen stannar därmed inom området EU plus Schweiz. Bindande är listan som API:et returnerar för den egna källregionen:

```bash
aws bedrock get-inference-profile \
  --region eu-central-2 \
  --inference-profile-identifier eu.anthropic.claude-sonnet-5-5 \
  --query "models[].modelArn"
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `bedrock get-inference-profile` | Läser definitionen av en Inference Profile |
| `--region eu-central-2` | Källregion Zürich; mållistan beror på källregionen |
| `--inference-profile-identifier` | ID för EU-profilen för Sonnet 5.5 |
| `--query "models[].modelArn"` | Skriver endast ut modell-ARN:er; regionen anges i respektive ARN |

</details>

Som standard lagras data endast i källregionen. Ett undantag är innehåll som hålls kvar för missbruksdetektion: det lagras i målregionen. Transporten mellan regionerna sker krypterat över AWS-nätet.

### Styrk behandlingsregionen per begäran

Bedrock är den enda av de beskrivna vägarna som loggar den faktiska behandlingsregionen per begäran. CloudTrail skriver den i källregionen i fältet `additionalEventData.inferenceRegion`:

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
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `cloudtrail lookup-events` | Söker igenom CloudTrail-händelserna från de senaste 90 dagarna |
| `--region eu-central-2` | Källregion där anropen loggas |
| `--lookup-attributes AttributeKey=EventSource,AttributeValue=bedrock.amazonaws.com` | Filtrerar på händelser från Bedrock |
| `--max-results 20` | Begränsar utdata till 20 händelser |
| `--query "Events[].CloudTrailEvent"` | Skriver endast ut hela händelsen som JSON-text |
| `--output text` | Utdata utan JSON-omslag, en händelse per rad |
| `jq -r '… \| @tsv'` | Extraherar tidpunkt, åtgärd och behandlingsregion som en tabbseparerad rad |

</details>

Dessutom kan alla regioner utanför EU och Schweiz spärras med en Service Control Policy (SCP) i AWS Organizations. Om en målregion i en profil är spärrad misslyckas begäran i stället för att växla till en annan region. Med denna kombination tvingas behandlingsplatsen tekniskt och kan styrkas per begäran.

### Claude Code via Bedrock i EU

Claude Code kan också köras via Bedrock. I en `eu-*`-region väljer Claude Code prefixet `eu.` automatiskt:

```bash
export CLAUDE_CODE_USE_BEDROCK=1
export AWS_REGION=eu-central-2
claude
```

| Variabel | Effekt |
|---|---|
| `CLAUDE_CODE_USE_BEDROCK=1` | Växlar Claude Code från Anthropic API till Bedrock |
| `AWS_REGION=eu-central-2` | Källregion Zürich; detta leder till EU-profilen |

Med `ANTHROPIC_BEDROCK_REGION_PREFIX` kan prefixet åsidosättas. AWS-inloggningen sker via de vanliga mekanismerna (profil, SSO, miljövariabler). Debiteringen sker via AWS-kontot, inte via ett Claude-abonnemang.

## Google Vertex AI: endast EU, uttryckligen utan Schweiz

På Google Cloud (Vertex AI, numera under namnet Gemini Enterprise Agent Platform) är läget mindre fördelaktigt för Schweiz:

| Ändpunkt | Modeller (urval) | Behandlingsplats |
|---|---|---|
| `global` | alla | Valfri Google Cloud-region, utan garanti |
| Multiregion `eu` (`aiplatform.eu.rep.googleapis.com`) | Opus 5.5, Sonnet 5.5, Haiku 5.5, Opus 5, Sonnet 5, Opus 4.8, Opus 4.7 | Endast EU-medlemsstater |
| Region `europe-west1` (Belgien) | Haiku 4.5, Sonnet 4.6, Opus 4.6 och äldre | Enligt modellsidan multiregion Europa |
| Region `europe-west6` (Zürich) | inga | Claude inte tillgängligt |

Google utesluter uttryckligen Schweiz från EU-multiregionsändpunkten: den omfattar endast EU-medlemsstater, Storbritannien och Schweiz ingår inte. Flera schweiziska rådgivningssidor nämner `europe-west6` som en väg till Claude i Zürich; enligt Googles officiella platstabell finns ingen Claude-modell tillgänglig där. Google dokumenterar inte något loggfält med den faktiska behandlingsregionen per begäran. För den globala ändpunkten anger Google att behandlingsregionen varken kan styras eller fastställas.

Vertex AI lämpar sig därmed för behandling uteslutande inom EU, men inte för behandling i Schweiz.

## Microsoft Foundry: för närvarande inget EU-alternativ

Microsoft erbjuder Claude i Foundry i två varianter. ”Hosted on Azure” beräknar på Azure-infrastruktur, som Global Standard eller Data Zone Standard; Data Zone finns för Claude endast för USA. ”Hosted on Anthropic” beräknar på Anthropics infrastruktur, och Microsoft påpekar att data kan behandlas utanför Azure och utanför den valda regionen. Anthropic listar Foundry för Europa som ”Coming soon”. I Microsoft 365 Copilot och Copilot Studio är Anthropic-modellerna undantagna från EU Data Boundary och är som standard inaktiverade i EU, EFTA (alltså även i Schweiz) och Storbritannien.

## Översikt: vilken väg garanterar vilken behandlingsplats

| Väg | Endast Schweiz | Endast EU | Området EU plus Schweiz | Region per begäran kan styrkas |
|---|---|---|---|---|
| claude.ai, Claude Code (abonnemang) | nej | nej | nej | nej |
| Claude API med `inference_geo` | nej | nej | nej (endast `us`) | endast `us`/`global` |
| AWS Bedrock, källa `eu-central-2`, profil `eu.` | nej | nej | ja | ja (CloudTrail) |
| AWS Bedrock, källa i EU, profil `eu.` | nej | ja | ja | ja (CloudTrail) |
| Google Vertex AI, ändpunkt `eu` | nej | ja | (endast EU-delen) | nej |
| Microsoft Foundry | nej | nej | nej | nej |

Det finns för närvarande ingen väg hos någon leverantör som garanterar att Claudes behandling begränsas till Schweiz. AWS Bedrock med källregion Zürich och EU-profil kommer närmast. Om det dessutom måste uteslutas att data lämnar EU (exempelvis på grund av avtalsmässiga åtaganden gentemot EU-kunder) kan EU-profilen anropas från en EU-region som Frankfurt; Zürich är då inte längre ett mål.

## Anonymisering före modellen: pilotföretag sökes

Behandlingsplatsen är i de flesta fall relevant endast eftersom personuppgifter eller konfidentiella uppgifter finns i prompten. Här kommer en ny produkt från mitt företag, adeptio gmbh, in: den fungerar som ett mellanlager mellan applikationerna i Schweiz och modellerna från Anthropic och OpenAI och anonymiserar begärandena innan de lämnar Schweiz. Modellen får då endast innehåll utan personkoppling, oavsett i vilket datacenter den beräknar.

**Schweiz-fokus:** Uppgifter som inte längre kan kopplas till en viss eller identifierbar person är inte personuppgifter i den mening som avses i art. 5 lit. a revDSG. Reglerna om överlämnande till utlandet (art. 16 f. revDSG) gäller inte för sådant innehåll. Om en anonymisering når denna tröskel beror på genomförandet och de konkreta uppgifterna; det är precis detta vi vill styrka i produktionsdrift.

För införandet söker vi företag som använder Claude eller modeller från OpenAI i produktion eller planerar införandet och som använder produkten som pilotföretag i vardagen. Båda sidor gynnas av detta:

| Ert företag | adeptio |
|---|---|
| Personuppgifter och konfidentiella uppgifter lämnar Schweiz endast i anonymiserad form. Motiveringen inför dataskyddsrådgivning, kunder och revision blir enklare. | Insikter från verkliga prompter och arbetsflöden i stället för testdata: vilka uppgifter som måste identifieras och var anonymiseringen tar bort för mycket eller för lite. |
| Valet mellan Anthropic, OpenAI, Bedrock eller Vertex AI styrs åter av modellutbud och pris i stället för enbart behandlingsplatsen. | Mätvärden för latens och svarskvalitet under verklig belastning. |
| Tillgång till produkten före allmän tillgänglighet och inflytande över vilka datatyper och användningsfall den först täcker. | Återkoppling från dataskydd och revision, som produktens bevisning kan utgå från. |
| Direktkontakt med utvecklarna i stället för support via ärenden. | Referenser från den schweiziska marknaden inför allmän tillgänglighet. |

<aside class="offer-box">
  <span class="offer-box__tag">Pilotföretag sökes</span>
  <p><strong>Använder ni Claude eller OpenAI med personuppgifter?</strong> Använd adeptios anonymiseringslager i produktionsdrift och var med och påverka den fortsatta utvecklingen.</p>
  <a class="offer-box__cta" href="/kontakt">Kontakta oss</a>
</aside>

## Vad den regionala bindningen kostar

Alla tre molnleverantörer och Anthropic självt tar ut ett påslag på 10 % jämfört med den globala ändpunkten för bindning till en region eller geografi. Priserna för Bedrock utgår från källregionen; Zürich, Frankfurt och North Virginia kostar lika mycket. Routning mellan regioner kostar inte extra.

Listpriser i USD per 1 miljon token (inmatning / utmatning):

| Modell | Anthropic API, `global` | Anthropic API, `us` | Bedrock eller Vertex, globalt | Bedrock `eu.` eller Vertex `eu` |
|---|---|---|---|---|
| Opus 5.5 | 4.00 / 20.00 | 4.40 / 22.00 | 4.00 / 20.00 | 4.40 / 22.00 |
| Sonnet 5.5 | 2.00 / 10.00 | 2.20 / 11.00 | 2.00 / 10.00 | 2.20 / 11.00 |
| Haiku 5.5 (prompter upp till 100 000 token) | 0.10 / 0.50 | 0.11 / 0.55 | 0.10 / 0.50 | 0.11 / 0.55 |
| Haiku 4.5 | 1.00 / 5.00 | inte tillgängligt | 1.00 / 5.00 | 1.10 / 5.50 |
| Sonnet 4.6 | 3.00 / 15.00 | 3.30 / 16.50 | 3.00 / 15.00 | 3.30 / 16.50 |

För Vertex AI gäller priset i EU-kolumnen för Haiku 4.5 och Sonnet 4.6 i regionen `europe-west1`. Ett räkneexempel för en intern applikation med 50 miljoner inmatnings- och 10 miljoner utmatningstoken per månad:

| Modell | Globalt | EU-bundet | Extrakostnad per månad |
|---|---|---|---|
| Sonnet 5.5 | 50 × 2 + 10 × 10 = 200 USD | 220 USD | 20 USD |
| Opus 5.5 | 50 × 4 + 10 × 20 = 400 USD | 440 USD | 40 USD |

Den regionala bindningen kostar alltså 10 % mer än den globala ändpunkten. Jämfört med listpriset hos Anthropic direkt med `inference_geo: "us"` kostar en EU-profil i Bedrock lika mycket. Vid valet väger snarare kringkostnaderna tyngre: ett AWS- eller Google Cloud-konto med organisationspolicyer, loggning och budgetvarningar måste byggas upp och drivas, och de fasta månadspriserna för Claude-abonnemangen ersätts av ren användningsbaserad debitering. Om bytet lönar sig beror på volymen; ett abonnemang har ett fast pris per person men erbjuder ingen kontroll över behandlingsplatsen.

## Lagring och träning

Utöver platsen spelar det roll hur länge data lagras:

- **Anthropic API:** Prompter och svar används inte för träning så länge inget uttryckligt samtycke finns. Zero Data Retention (ZDR) är tillgängligt på begäran per organisation och kan kombineras med `inference_geo`. Innehåll som missbruksdetekteringen markerar lagras upp till två år även med ZDR. För modellerna Fable och Mythos är en lagringstid på 30 dagar obligatorisk.
- **AWS Bedrock:** Innehåll går inte till Anthropic. Lagring kan väljas per modell mellan `none`, `default` och `aws_review`; Fable kräver `aws_review` med upp till 30 dagars lagring för granskning av AWS.
- **Google Vertex AI:** Google använder endast uppgifterna för träning eller finjustering med föregående samtycke.

## Ändringshistorik

- **25 oktober 2026:** Tillagt: Haiku 5.5 (sedan 7 oktober 2026) med tillgänglighet i Zürich och i Google Clouds EU-multiregion samt priser; målregionerna för EU-profilen är nu även belagda för Opus 5.5 och Sonnet 5.5, status för priser och regioner 25 oktober 2026. Källänkar uppdaterade till de nya adresserna i Google Cloud-dokumentationen.

## Källor

1.  [Claude Platform Docs: Data residency](https://platform.claude.com/docs/en/manage-claude/data-residency): Parameter `inference_geo`, värden `us` och `global`, Workspace geo, modeller som stöds, fält `usage.inference_geo`.

2.  [Claude Platform Docs: Pricing](https://platform.claude.com/docs/en/about-claude/pricing): Listpriser för Anthropic API och faktor 1,1 för `inference_geo: "us"`.

3.  [Claude Platform Docs: API and data retention](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention): Zero Data Retention, undantag för markerat innehåll och för Fable/Mythos.

4.  [Anthropic: Privacy Policy](https://www.anthropic.com/legal/privacy): Anthropic Ireland som personuppgiftsansvarig för EES, UK och Schweiz, överföring till USA på grundval av standardavtalsklausuler.

5.  [Anthropic: Regional compliance](https://claude.com/regional-compliance): Översikt över vilken plattform som erbjuder dataresidens i vilken region; Foundry Europa som ”Coming soon”.

6.  [Claude Platform Docs: Claude in Amazon Bedrock](https://platform.claude.com/docs/en/build-with-claude/claude-in-amazon-bedrock): Ändpunktstyper per region, 10 % påslag för regionala ändpunkter.

7.  [AWS: Modellkort Claude Sonnet 4.6](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-4-6.html): Målregioner för EU-profilen per källregion, inklusive Zürich.

8.  [AWS: Modellkort Claude Sonnet 5.5](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-5-5.html): Tillgänglighet i `eu-central-2` endast som Geo- och Global-profil, målregioner för EU-profilen från Zürich.

9.  [AWS: Geographic cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/geographic-cross-region-inference.html): Datakvarhållning inom geografin, lagringsplats, spärrning av målregioner via SCP.

10.  [AWS: Cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html): Fält `additionalEventData.inferenceRegion` i CloudTrail, pris enligt källregion.

11.  [AWS: Data protection in Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/data-protection.html): Modellsleverantörer har ingen åtkomst till prompter, svar och loggar.

12.  [AWS: Amazon Bedrock Pricing](https://aws.amazon.com/bedrock/pricing/): Listpriser för Global- och Geo-profiler, identiska för Zürich, Frankfurt och North Virginia.

13.  [AWS Alps Blog: Cross-region inference for EU data processing in Switzerland](https://aws.amazon.com/blogs/alps/unlocking-ai-flexibility-in-switzerland-a-guide-to-cross-region-inference-for-eu-data-processing-and-model-access/): Instruktioner från AWS Schweiz för EU-profilen från Zürich.

14.  [Claude Code Docs: Amazon Bedrock](https://code.claude.com/docs/en/amazon-bedrock): Miljövariabler och automatiskt prefix `eu.` i EU-regioner.

15.  [Google Cloud: Generative AI locations](https://docs.cloud.google.com/gemini-enterprise-agent-platform/resources/locations): Platstabell för Claude-modeller, ingen tillgänglighet i `europe-west6`.

16.  [Google Cloud: Data residency](https://docs.cloud.google.com/gemini-enterprise-agent-platform/resources/data-residency): EU-multiregion endast för EU-medlemsstater, Schweiz och UK undantagna.

17.  [Google Cloud: Vertex AI Pricing](https://cloud.google.com/gemini-enterprise-agent-platform/generative-ai/pricing): Listpriser globalt, EU-multiregion och `europe-west1`.

18.  [Microsoft Learn: Claude models hosting comparison](https://learn.microsoft.com/en-us/azure/foundry/foundry-models/concepts/claude-models-hosting-comparison): Hosted on Azure och Hosted on Anthropic, Data Zone endast USA.

19.  [Microsoft Learn: Anthropic as AI subprocessor in Microsoft 365 Copilot](https://learn.microsoft.com/en-us/microsoft-365/copilot/connect-to-ai-subprocessor): Undantag från EU Data Boundary, standardinställning i EU/EFTA.

20.  [TCDEV Blog: Where Are Claude's Data Centers?](https://www.tcdev.de/blog/where-are-claudes-data-centers/): Tim Cadenbachs översikt över Project Rainier, Google-TPU-avtalet och de egna datacentren med Fluidstack.

21.  [Anthropic: Investering på 50 miljarder USD i amerikansk infrastruktur](https://www.anthropic.com/news/anthropic-invests-50-billion-in-american-ai-infrastructure): Tillkännagivande av egna datacenter i Texas och New York med Fluidstack.

22.  [Claude Platform Docs: IP addresses](https://platform.claude.com/docs/en/api/ip-addresses): Inkommande adressområde `160.79.104.0/23`.

23.  [RIPEstat](https://stat.ripe.net/): Prefixöversikt och BGP-status för `160.79.104.0/23` (AS399358, upstream AS13335).

24.  [llmlatency.dev: Anthropic](https://llmlatency.dev/provider/anthropic): Svarstider vid Edge-noden per mätplats.

25.  [Fedlex: Dataskyddsförordning (DSV), bilaga 1](https://www.fedlex.admin.ch/eli/cc/2022/568/de): Lista över stater med adekvat dataskydd, inklusive EU/EES och USA för certifierade företag.

26.  [Anthropic: Claude Haiku 5.5](https://www.anthropic.com/news/claude-haiku-5-5): Publicering den 7 oktober 2026, tillgänglighet hos AWS, Google Cloud och Microsoft, priser med nivå från 100 000 token.

27.  [AWS: Modellkort Claude Haiku 5.5](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-haiku-5-5.html): Tillgänglighet i `eu-central-2` som Geo- och Global-profil samt målregioner för EU-profilen från Zürich.
