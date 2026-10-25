---
title: "Which data centre processes Claude? Tracing the connection and restricting processing to Switzerland or the EU"
navTitle: "Claude data centre"
description: "Which stops a request to Claude passes through, why the trace ends at the Cloudflare edge and what Anthropic discloses about its data centres. It also compares routes with a fixed processing location (Anthropic API, AWS Bedrock, Google Vertex AI, Microsoft Foundry), the limits of Switzerland-only operations and the cost of regional binding."
date: "2026-10-02"
kategorie: "Claude"
timeToRead: "12 min read"
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
translationSourceHash: 82442938b810b052119b19044fb328ab5edcdebb9350352ff47207901949421b
translationModel: gpt-5.6-terra
translatedAt: 2026-10-20T08:48:45.851Z
translationReview: automatic
url: https://rafaelpfister.ch/sv/blog/which-data-centre-processes-claude-trace-the-connection-and-restrict-processing-to-switzerland
aktualisiert: 2026-10-25
---

When you use Claude through claude.ai, Claude Code or the API, you cannot tell which data centre processes the request. Only the first section of the connection as far as the nearest edge node is visible. For companies that send personal data or confidential content to a language model, that is insufficient: they need to be able to demonstrate to customers, data protection advisers or auditors where processing takes place.

This article shows how far the connection itself can be traced, what is publicly known about Anthropic’s data centres and which routes reliably restrict processing to the EU or the EU plus Switzerland area. Prices and regions are current as of 2 October 2026.

**Switzerland focus:** The revised Federal Act on Data Protection (revFADP) is decisive. Under Art. 16 revFADP, disclosure of personal data abroad is permitted if the Federal Council attests that the destination country provides adequate protection (Annex 1 of the Data Protection Ordinance, DPO) or if appropriate safeguards such as standard contractual clauses are in place. EU and EEA countries are on this list; the US is included only for companies certified under the Swiss-U.S. Data Privacy Framework. A processing location in the EU therefore makes justification considerably easier, but does not replace the other obligations (data processing agreement, informing data subjects, data security).

**EU note:** Companies based in the EU are instead subject to Art. 44 et seq. GDPR. The European Commission has recognised Switzerland as providing an adequate level of data protection (adequacy decision 2000/518/EC, confirmed in January 2024); processing in Zurich is therefore not a problematic third-country transfer for EU companies.

<aside class="pilot-card">
  <span class="pilot-card__tag">New from adeptio · Pilot companies wanted</span>
  <p class="pilot-card__titel">Use Claude and OpenAI without personal data leaving Switzerland</p>
  <p class="pilot-card__lead">Our intermediary layer sits between your applications in Switzerland and the models from Anthropic and OpenAI, anonymising every request before it leaves the country. For live operation, we are looking for companies that use the product in their day-to-day work.</p>
  <div class="pilot-card__spalten">
    <div class="pilot-card__spalte">
      <p class="pilot-card__spalte-titel">Your company</p>
      <ul>
        <li>Personal data leaves Switzerland only in anonymised form</li>
        <li>Simpler justification to data protection and audit teams</li>
        <li>Access before general availability and a say in the features</li>
        <li>Direct contact with the developers</li>
      </ul>
    </div>
    <div class="pilot-card__spalte">
      <p class="pilot-card__spalte-titel">adeptio</p>
      <ul>
        <li>Insights from real prompts rather than test data</li>
        <li>Metrics on latency and response quality under real load</li>
        <li>References from the Swiss market</li>
      </ul>
    </div>
  </div>
  <div class="pilot-card__aktionen">
    <a class="pilot-card__cta" href="/kontakt">Apply as a pilot company</a>
    <a class="pilot-card__mehr" href="#anonymisierung-vor-dem-modell-pilotunternehmen-gesucht">Details and legal assessment</a>
  </div>
</aside>

## Vilka stationer en förfrågan till Claude passerar

En förfrågan till `claude.ai` eller `api.anthropic.com` passerar tre stationer:

| Station | Vem driver den | Synlig för dig? |
|---|---|---|
| Edge-nod (TLS-terminering, skydd mot missbruk) | Cloudflare, på uppdrag av Anthropic | Ja, platsen kan läsas av via header |
| Internt nät till backend | Cloudflare och Anthropic | Nej |
| Inferens (modellen beräknar) | Anthropic, med beräkningskapacitet från AWS, Google Cloud och egna datacenter | Nej, endast grov uppgift `us` eller `global` |

Värdnamnen `claude.ai` och `api.anthropic.com` pekar båda på adressen `160.79.104.10`. Den tillhör prefixet `160.79.104.0/23`, som är registrerat på Anthropic (AS399358) och enligt RPKI har ett giltigt ursprung hos Anthropic. I den globala routningen annonseras prefixet dock uteslutande via Cloudflare (AS13335): Anthropic för in sina egna IP-adresser i Cloudflares Anycast-nätverk. Därmed besvaras samma adress världen över från varje Cloudflare-plats, och anslutningen hamnar vid närmaste edge-nod.

## Fastställ edge-noden själv

Cloudflare avslöjar den betjänande platsen på en diagnossida och i headern `cf-ray`. Förkortningen efter Ray-ID:t är en IATA-flygplatskod: `ZRH` står för Zürich, `GVA` för Genève, `FRA` för Frankfurt och `MRS` för Marseille.

```bash
curl -s https://api.anthropic.com/cdn-cgi/trace
```

I utdata är raderna `colo=` (edge-plats) och `loc=` (det land som Cloudflare kopplar din avsändaradress till) relevanta. I Windows ger PowerShell samma resultat:

```powershell
$trace = Invoke-WebRequest -Uri "https://api.anthropic.com/cdn-cgi/trace" -UseBasicParsing
$trace.Content
```

Du läser headern med en HEAD-förfrågan:

```bash
curl -sI https://api.anthropic.com/ \
  | grep -i -E '^(server|cf-ray):'
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-s` | Undertrycker curl:s förloppsindikator och felmeddelanden |
| `-I` | Skickar en HEAD-förfrågan och visar endast svarsheadern |
| `grep -i` | Söker utan hänsyn till versaler/gemener |
| `-E '^(server\|cf-ray):'` | Utökat reguljärt uttryck: endast rader som börjar med `server:` eller `cf-ray:` |

</details>

Från ett schweiziskt nätverk kan man normalt förvänta sig `ZRH` eller `GVA`. Om internetutgången ligger på en annan plats, exempelvis i ett företagsnätverk med central utgång utomlands, svarar edge-noden där, till exempel med `colo=FRA` eller `colo=MRS`. Edge-platsen följer alltså din nätverksutgång, inte platsen för behandlingen.

## Varför traceroute slutar vid edge-noden

En traceroute visar vägen till edge-noden och inte längre:

```powershell
Test-NetConnection -ComputerName api.anthropic.com -TraceRoute -Hops 20
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-ComputerName api.anthropic.com` | Mätningens målvärd |
| `-TraceRoute` | Fastställer routerhoppen på vägen till målet |
| `-Hops 20` | Maximalt antal hopp som kontrolleras |

</details>

I Linux och macOS motsvaras detta av `traceroute -n api.anthropic.com` (`-n` undertrycker DNS-upplösning av hoppen). I mätningen låg de sista hoppen före `160.79.104.10` inom adressintervallet `162.158.0.0/15`, som tillhör Cloudflare. Därefter slutar spåret: anslutningen termineras vid edge-noden, och vidarebefordringen till backend sker i en ny intern anslutning som inte är synlig utifrån.

RIPEstat visar vem prefixet tillhör och via vilket nätverk det annonseras:

```bash
curl -s "https://stat.ripe.net/data/prefix-overview/data.json?resource=160.79.104.10"
curl -s "https://stat.ripe.net/data/bgp-state/data.json?resource=160.79.104.0/23"
```

I det andra resultatet slutar alla AS-vägar på `13335 399358`: Cloudflare är den enda upstream-leverantören för Anthropics prefix.

En offentlig mätning som spårar vägen hela vägen till ett inferensdatacenter finns enligt min efterforskning inte, och på grund av termineringen vid edge-noden är den inte heller möjlig med nätverksverktyg. Mättjänster som llmlatency.dev registrerar endast svarstiden till den första byten vid edge-noden (september 2026: median 96 ms från US-Central, 199 ms från Tyskland). Tiden till den första tokenen (Time to First Token) skulle visserligen kunna jämföras från flera platser, men den beror mer på modell, promptlängd och belastning än på avstånd. Den ger därför ingen tillförlitlig slutsats om en plats.

## Vad som är känt om Anthropics datacenter

Anthropic driver hittills inget eget, offentligt identifierat inferensdatacenter med adress. Beräkningskapaciteten kommer från partnerskap som Tim Cadenbach har sammanställt på TCDEV-bloggen:

| Plats / partner | Nyckeldata | Känd roll |
|---|---|---|
| Project Rainier, New Carlisle (Indiana, USA), Amazon Web Services | Öppnat i oktober 2025, omkring 11 miljarder USD, cirka 500 000 Trainium2-chip, över 2,2 GW planerat | Anthropic är huvudhyresgäst; främst träning |
| Google Cloud | Avtal i oktober 2025 om upp till 1 miljon TPU:er, över 1 GW från 2026 | Regioner ej publicerade |
| Egna datacenter med Fluidstack | Tillkännagivet i november 2025, 50 miljarder USD, platser i Texas och New York, driftstart från 2026 | Fortfarande under uppbyggnad |

Träning och inferens sker inte nödvändigtvis på samma plats. Inferens kan köras i valfria AWS- eller Google Cloud-regioner, och Anthropic uppger inte vilken plats som beräknar svaret för en användare i Schweiz. Det enda som är säkert är vad integritetspolicyn anger: avtalspart och personuppgiftsansvarig för kunder i EES, Storbritannien och Schweiz är Anthropic Ireland, Limited i Dublin; uppgifterna överförs till servrar i USA eller andra länder utanför EES med stöd av standardavtalsklausuler.

## Styrningen direkt hos Anthropic: endast USA eller globalt

Claude API har sedan modellerna i generation 4.6 parametern `inference_geo`. Den har exakt två värden:

| Värde | Effekt | Pris |
|---|---|---|
| `global` (standard) | Inferens i valfri tillgänglig region | Listpris |
| `us` | Inferens uteslutande i USA | Listpris × 1,1 |

Det finns inget värde för EU eller Schweiz. Workspace-lagringsplatsen (`workspace geo`) kan för närvarande också endast sättas till `us`. Svaret anger i fältet `usage.inference_geo` vilken inställning som tillämpades, men ingen konkret region. För Haiku 4.5 och äldre modeller besvarar API:t parametern med fel 400.

För claude.ai och Claude Code med abonnemang (Pro, Max, Team, Enterprise) finns ingen inställning för behandlingsplatsen. Via Anthropics direkta erbjudanden går det alltså inte att begränsa behandlingen till vare sig EU eller Schweiz. Det går endast via molnleverantörerna, som kör Claude i sina egna regioner.

## AWS Bedrock: EU plus Schweiz från Zürich

AWS Bedrock kör Claude på AWS-infrastruktur. Enligt AWS har Anthropic ingen åtkomst till kundernas prompter, svar eller loggar. Du väljer region via källregionen för ditt API-anrop och via Inference Profile:

| Slutpunktstyp | Prefix för modell-ID | Behandlingsplats |
|---|---|---|
| Global Cross-Region Inference | `global.` | Valfri kommersiell AWS-region världen över |
| Geographic Cross-Region Inference | `eu.` | Endast regioner inom geografin |
| In-Region | utan prefix | Endast den anropade regionen |

För Schweiz är regionen `eu-central-2` (Zürich) avgörande. Där finns Opus 5.5, Sonnet 5.5 och Haiku 4.5 tillgängliga, men endast som Global- eller EU-profil och inte In-Region. Det finns ingen profil som beräknar uteslutande i Zürich. Om EU-profilen anropas från Zürich fördelar AWS enligt modellkortet (belagt för Haiku 4.5 och Sonnet 4.6) förfrågningarna till följande regioner:

| Region | Plats |
|---|---|
| `eu-central-2` | Zürich |
| `eu-central-1` | Frankfurt |
| `eu-north-1` | Stockholm |
| `eu-south-1` | Milano |
| `eu-south-2` | Spanien |
| `eu-west-1` | Irland |
| `eu-west-3` | Paris |

Zürich är endast ett möjligt mål om förfrågan kommer från Zürich. Behandlingen stannar därmed inom EU plus Schweiz. För Opus 5.5 och Sonnet 5.5 anger modellkortet inte målregionerna; API:t returnerar den faktiska listan:

```bash
aws bedrock get-inference-profile \
  --region eu-central-2 \
  --inference-profile-identifier eu.anthropic.claude-sonnet-5-5 \
  --query "models[].modelArn"
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `bedrock get-inference-profile` | Läser definitionen av en Inference Profile |
| `--region eu-central-2` | Källregion Zürich; mållistan beror på källregionen |
| `--inference-profile-identifier` | ID för EU-profilen för Sonnet 5.5 |
| `--query "models[].modelArn"` | Visar endast modell-ARN:er; regionen anges i respektive ARN |

</details>

Data lagras som standard endast i källregionen. Ett undantag är innehåll som sparas för missbruksdetektering: det ligger i målregionen. Transporten mellan regionerna sker krypterat över AWS-nätverket.

### Bevisa behandlingsregionen per förfrågan

Bedrock är den enda av de beskrivna vägarna som loggar den faktiska behandlingsregionen för varje förfrågan. CloudTrail skriver den i källregionen i fältet `additionalEventData.inferenceRegion`:

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
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `cloudtrail lookup-events` | Söker igenom CloudTrail-händelser från de senaste 90 dagarna |
| `--region eu-central-2` | Källregionen där anropen loggas |
| `--lookup-attributes AttributeKey=EventSource,AttributeValue=bedrock.amazonaws.com` | Filtrerar på händelser från Bedrock |
| `--max-results 20` | Begränsar utdata till 20 händelser |
| `--query "Events[].CloudTrailEvent"` | Visar endast hela händelsen som JSON-text |
| `--output text` | Utdata utan JSON-omslag, en händelse per rad |
| `jq -r '… \| @tsv'` | Extraherar tidpunkt, åtgärd och behandlingsregion som en tabbavgränsad rad |

</details>

Dessutom kan en Service Control Policy (SCP) i AWS Organizations spärra alla regioner utanför EU och Schweiz. Om en målregion i en profil är spärrad misslyckas förfrågan i stället för att växla till en annan region. Med denna kombination är behandlingsplatsen tekniskt framtvingad och kan bevisas för varje förfrågan.

### Claude Code via Bedrock i EU

Claude Code kan också köras via Bedrock. I en `eu-*`-region väljer Claude Code automatiskt prefixet `eu.`:

```bash
export CLAUDE_CODE_USE_BEDROCK=1
export AWS_REGION=eu-central-2
claude
```

| Variabel | Effekt |
|---|---|
| `CLAUDE_CODE_USE_BEDROCK=1` | Växlar Claude Code från Anthropic API till Bedrock |
| `AWS_REGION=eu-central-2` | Källregion Zürich; detta leder till EU-profilen |

Prefixet kan åsidosättas med `ANTHROPIC_BEDROCK_REGION_PREFIX`. AWS-autentisering sker via de vanliga mekanismerna (profil, SSO, miljövariabler). Debiteringen sker via AWS-kontot, inte via ett Claude-abonnemang.

## Google Vertex AI: endast EU, uttryckligen utan Schweiz

På Google Cloud (Vertex AI, numera under namnet Gemini Enterprise Agent Platform) är läget mindre gynnsamt för Schweiz:

| Slutpunkt | Modeller (urval) | Behandlingsplats |
|---|---|---|
| `global` | alla | Valfri Google Cloud-region, utan garanti |
| Multiregion `eu` (`aiplatform.eu.rep.googleapis.com`) | Opus 5.5, Sonnet 5.5, Opus 5, Sonnet 5, Opus 4.8, Opus 4.7 | Endast EU-medlemsstater |
| Region `europe-west1` (Belgien) | Haiku 4.5, Sonnet 4.6, Opus 4.6 och äldre | Enligt modellsidan multiregion Europa |
| Region `europe-west6` (Zürich) | inga | Claude är inte tillgängligt |

Google utesluter uttryckligen Schweiz från EU-multiregionens slutpunkt: den omfattar endast EU-medlemsstater, och Storbritannien och Schweiz ingår inte. Flera schweiziska rådgivningssidor nämner `europe-west6` som en väg till Claude i Zürich; enligt Googles officiella platstabell finns ingen Claude-modell tillgänglig där. Google dokumenterar inte något loggfält med den faktiska behandlingsregionen per förfrågan. För den globala slutpunkten uppger Google att behandlingsregionen varken kan styras eller fastställas.

Vertex AI lämpar sig därmed för behandling uteslutande inom EU, men inte för behandling i Schweiz.

## Microsoft Foundry: för närvarande inget EU-alternativ

Microsoft erbjuder Claude i Foundry i två varianter. ”Hosted on Azure” kör på Azure-infrastruktur, som Global Standard eller Data Zone Standard; Data Zone finns för Claude endast för USA. ”Hosted on Anthropic” kör på Anthropics infrastruktur, och Microsoft påpekar att data kan behandlas utanför Azure och utanför den valda regionen. Anthropic listar Foundry för Europa som ”Coming soon”. I Microsoft 365 Copilot och Copilot Studio är Anthropic-modellerna undantagna från EU Data Boundary och är som standard inaktiverade i EU, EFTA (alltså även i Schweiz) och Storbritannien.

## Översikt: vilken väg garanterar vilken behandlingsplats

| Väg | Endast Schweiz | Endast EU | EU plus Schweiz | Region per förfrågan kan bevisas |
|---|---|---|---|---|
| claude.ai, Claude Code (abonnemang) | nej | nej | nej | nej |
| Claude API med `inference_geo` | nej | nej | nej (endast `us`) | endast `us`/`global` |
| AWS Bedrock, källa `eu-central-2`, profil `eu.` | nej | nej | ja | ja (CloudTrail) |
| AWS Bedrock, källa i EU, profil `eu.` | nej | ja | ja | ja (CloudTrail) |
| Google Vertex AI, slutpunkt `eu` | nej | ja | (endast EU-delen) | nej |
| Microsoft Foundry | nej | nej | nej | nej |

Det finns för närvarande ingen väg hos någon leverantör som garanterar att Claudes behandling begränsas till Schweiz. AWS Bedrock med källregionen Zürich och EU-profil kommer närmast. Om det dessutom måste uteslutas att data lämnar EU, exempelvis på grund av avtalsåtaganden gentemot EU-kunder, kan EU-profilen anropas från en EU-region som Frankfurt; Zürich är då inte längre ett mål.

## Anonymisering före modellen: pilotföretag sökes

I de flesta fall är behandlingsplatsen endast relevant eftersom prompten innehåller personuppgifter eller konfidentiell information. Där kommer en ny produkt från mitt företag, adeptio gmbh, in: den fungerar som ett mellanlager mellan applikationerna i Schweiz och modellerna från Anthropic och OpenAI och anonymiserar förfrågningarna innan de lämnar Schweiz. Modellen får då endast innehåll utan personkoppling, oavsett vilket datacenter den beräknar i.

**Switzerland focus:** Information som inte längre kan kopplas till en viss eller identifierbar person är inte personuppgifter i den mening som avses i Art. 5 lit. a revFADP. Reglerna om utlämnande utomlands (Art. 16 et seq. revFADP) gäller inte för sådant innehåll. Om en anonymisering når denna tröskel beror på genomförandet och de konkreta uppgifterna; det är just detta vi vill visa i live-drift.

För införandet söker vi företag som använder Claude eller modeller från OpenAI i produktion, eller planerar införandet, och som använder produkten som pilotföretag i sin vardag. Båda sidor gynnas:

| Ditt företag | adeptio |
|---|---|
| Personuppgifter och konfidentiell information lämnar Schweiz endast i anonymiserad form. Det blir enklare att motivera detta för dataskyddsrådgivare, kunder och revision. | Insikter från verkliga prompter och arbetsflöden i stället för testdata: vilka uppgifter som måste identifieras och var anonymiseringen tar bort för mycket eller för lite. |
| Valet mellan Anthropic, OpenAI, Bedrock eller Vertex AI styrs åter av modellutbud och pris i stället för enbart behandlingsplatsen. | Mätvärden för latens och svarskvalitet under verklig belastning. |
| Tillgång till produkten före allmän tillgänglighet och inflytande över vilka datatyper och användningsfall den först omfattar. | Återkoppling från dataskydd och revision som produktens underlag ska utgå från. |
| Direkt kontakt med utvecklarna i stället för ärendesupport. | Referenser från den schweiziska marknaden inför allmän tillgänglighet. |

<aside class="offer-box">
  <span class="offer-box__tag">Pilotföretag sökes</span>
  <p><strong>Använder du Claude eller OpenAI med personuppgifter?</strong> Använd adeptios anonymiseringslager i live-drift och bidra till att forma den fortsatta utvecklingen.</p>
  <a class="offer-box__cta" href="/kontakt">Kontakta oss</a>
</aside>

## Vad den regionala bindningen kostar

Alla tre molnleverantörer och Anthropic själva tar ut ett påslag på 10 % jämfört med den globala slutpunkten för bindning till en region eller geografi. Bedrock-priserna baseras på källregionen; Zürich, Frankfurt och North Virginia kostar lika mycket. Routning mellan regioner medför ingen extra kostnad.

Listpriser i USD per 1 miljon token (inmatning / utmatning):

| Modell | Anthropic API, `global` | Anthropic API, `us` | Bedrock eller Vertex, global | Bedrock `eu.` eller Vertex `eu` |
|---|---|---|---|---|
| Opus 5.5 | 4.00 / 20.00 | 4.40 / 22.00 | 4.00 / 20.00 | 4.40 / 22.00 |
| Sonnet 5.5 | 2.00 / 10.00 | 2.20 / 11.00 | 2.00 / 10.00 | 2.20 / 11.00 |
| Haiku 4.5 | 1.00 / 5.00 | inte tillgängligt | 1.00 / 5.00 | 1.10 / 5.50 |
| Sonnet 4.6 | 3.00 / 15.00 | 3.30 / 16.50 | 3.00 / 15.00 | 3.30 / 16.50 |

För Vertex AI gäller priset i EU-kolumnen för Haiku 4.5 och Sonnet 4.6 i regionen `europe-west1`. Ett räkneexempel för en intern applikation med 50 miljoner input- och 10 miljoner output-token per månad:

| Modell | Globalt | EU-bundet | Extrakostnad per månad |
|---|---|---|---|
| Sonnet 5.5 | 50 × 2 + 10 × 10 = 200 USD | 220 USD | 20 USD |
| Opus 5.5 | 50 × 4 + 10 × 20 = 400 USD | 440 USD | 40 USD |

Den regionala bindningen kostar alltså 10 % mer än den globala slutpunkten. Jämfört med listpriset direkt hos Anthropic med `inference_geo: "us"` kostar en EU-profil på Bedrock lika mycket. Vid valet väger snarare kringkostnaderna tungt: ett AWS- eller Google Cloud-konto med organisationspolicyer, loggning och budgetvarningar måste byggas upp och drivas, och Claude-abonnemangens fasta månadspriser försvinner till förmån för ren förbrukningsbaserad debitering. Om bytet lönar sig beror på volymen; ett abonnemang har ett fast pris per person men ger ingen kontroll över behandlingsplatsen.

## Lagring och träning

Utöver platsen räknas hur länge data lagras:

- **Anthropic API:** Prompter och svar används inte för träning så länge det inte finns ett uttryckligt samtycke. Zero Data Retention (ZDR) kan fås på begäran per organisation och kan kombineras med `inference_geo`. Innehåll som flaggas av missbruksdetekteringen lagras även med ZDR i upp till två år. För modellerna Fable och Mythos krävs 30 dagars lagring.
- **AWS Bedrock:** Innehåll skickas inte till Anthropic. Lagring kan väljas per modell mellan `none`, `default` och `aws_review`; Fable kräver `aws_review` med upp till 30 dagars lagring för granskning av AWS.
- **Google Vertex AI:** Google använder endast data för träning eller finjustering efter föregående samtycke.

## Källor

1.  [Claude Platform Docs: Data residency](https://platform.claude.com/docs/en/manage-claude/data-residency): Parameter `inference_geo`, värden `us` och `global`, Workspace geo, stödda modeller, fält `usage.inference_geo`.

2.  [Claude Platform Docs: Pricing](https://platform.claude.com/docs/en/about-claude/pricing): Listpriser för Anthropic API och faktor 1,1 för `inference_geo: "us"`.

3.  [Claude Platform Docs: API and data retention](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention): Zero Data Retention, undantag för flaggat innehåll och för Fable/Mythos.

4.  [Anthropic: Privacy Policy](https://www.anthropic.com/legal/privacy): Anthropic Ireland som personuppgiftsansvarig för EES, Storbritannien och Schweiz, överföring till USA på basis av standardavtalsklausuler.

5.  [Anthropic: Regional compliance](https://claude.com/regional-compliance): Översikt över vilken plattform som erbjuder dataresidens i vilken region; Foundry Europa som ”Coming soon”.

6.  [Claude Platform Docs: Claude in Amazon Bedrock](https://platform.claude.com/docs/en/build-with-claude/claude-in-amazon-bedrock): Slutpunktstyper per region, 10 % påslag för regionala slutpunkter.

7.  [AWS: Modellkort Claude Sonnet 4.6](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-4-6.html): EU-profilens målregioner per källregion, inklusive Zürich.

8.  [AWS: Modellkort Claude Sonnet 5.5](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-5-5.html): Tillgänglighet i `eu-central-2` endast som Geo- och Global-profil.

9.  [AWS: Geographic cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/geographic-cross-region-inference.html): Datalagring inom geografin, lagringsplats, spärrning av målregioner via SCP.

10.  [AWS: Cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html): Fält `additionalEventData.inferenceRegion` i CloudTrail, pris enligt källregion.

11.  [AWS: Data protection in Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/data-protection.html): Modellleverantörer har ingen åtkomst till prompter, svar och loggar.

12.  [AWS: Amazon Bedrock Pricing](https://aws.amazon.com/bedrock/pricing/): Listpriser för Global- och Geo-profiler, identiska för Zürich, Frankfurt och North Virginia.

13.  [AWS Alps Blog: Cross-region inference for EU data processing in Switzerland](https://aws.amazon.com/blogs/alps/unlocking-ai-flexibility-in-switzerland-a-guide-to-cross-region-inference-for-eu-data-processing-and-model-access/): Instruktioner från AWS Schweiz för EU-profilen från Zürich.

14.  [Claude Code Docs: Amazon Bedrock](https://code.claude.com/docs/en/amazon-bedrock): Miljövariabler och automatiskt prefix `eu.` i EU-regioner.

15.  [Google Cloud: Generative AI locations](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/locations): Platstabell för Claude-modeller, ingen tillgänglighet i `europe-west6`.

16.  [Google Cloud: Data residency](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/data-residency): EU-multiregion endast för EU-medlemsstater, Schweiz och Storbritannien undantagna.

17.  [Google Cloud: Vertex AI Pricing](https://cloud.google.com/vertex-ai/generative-ai/pricing): Listpriser globalt, EU-multiregion och `europe-west1`.

18.  [Microsoft Learn: Claude models hosting comparison](https://learn.microsoft.com/en-us/azure/foundry/foundry-models/concepts/claude-models-hosting-comparison): Hosted on Azure och Hosted on Anthropic, Data Zone endast USA.

19.  [Microsoft Learn: Anthropic as AI subprocessor in Microsoft 365 Copilot](https://learn.microsoft.com/en-us/microsoft-365/copilot/connect-to-ai-subprocessor): Undantag från EU Data Boundary, standardinställning i EU/EFTA.

20.  [TCDEV Blog: Where Are Claude's Data Centers?](https://www.tcdev.de/blog/where-are-claudes-data-centers/): Tim Cadenbachs översikt över Project Rainier, Google TPU-avtalet och egna datacenter med Fluidstack.

21.  [Anthropic: Investering på 50 miljarder USD i amerikansk infrastruktur](https://www.anthropic.com/news/anthropic-invests-50-billion-in-american-ai-infrastructure): Tillkännagivande av egna datacenter i Texas och New York med Fluidstack.

22.  [Claude Platform Docs: IP addresses](https://platform.claude.com/docs/en/api/ip-addresses): Inkommande adressintervall `160.79.104.0/23`.

23.  [RIPEstat](https://stat.ripe.net/): Prefixöversikt och BGP-status för `160.79.104.0/23` (AS399358, upstream AS13335).

24.  [llmlatency.dev: Anthropic](https://llmlatency.dev/provider/anthropic): Svarstider vid edge-noden efter mätplats.

25.  [Fedlex: Data Protection Ordinance (DPO), Annex 1](https://www.fedlex.admin.ch/eli/cc/2022/568/de): Lista över stater med adekvat dataskydd, inklusive EU/EES och USA för certifierade företag.
