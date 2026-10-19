---
title: "Hvilket datasenter behandler Claude i? Spor forbindelsen og begrens behandlingen til Sveits eller EU"
navTitle: "Claude-datasenter"
description: "Hvilke stasjoner en forespørsel til Claude går gjennom, hvorfor sporet stopper ved Cloudflare-kanten, og hva Anthropic opplyser om datasentrene sine. I tillegg sammenlignes veier med fast behandlingssted (Anthropic API, AWS Bedrock, Google Vertex AI, Microsoft Foundry), begrensningene ved drift kun i Sveits og hva regional binding koster."
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
translationOf: claude-rechenzentrum-schweiz-eu
translationSourceHash: 93ccc2c90b4f24a12ada6d9fd9417465b4e821e9d371fa5d045ed64cfd70e07c
translationModel: gpt-5.6-terra
translatedAt: 2026-10-19T14:55:41.901Z
translationReview: automatic
url: https://rafaelpfister.ch/no/blog/hvilket-datasenter-kjorer-claude-i-spor-forbindelsen-og-begrens-behandlingen-til-sveits-eller-eu
---

Når du bruker Claude via claude.ai, Claude Code eller API-et, får du ikke vite hvilket datasenter modellen behandler forespørselen i. Det er bare den første delen av forbindelsen frem til nærmeste Edge-node som er synlig. For virksomheter som overfører personopplysninger eller fortrolig innhold til en språkmodell, er dette ikke nok: De må kunne dokumentere overfor kunder, personvernrådgivere eller revisjonen hvor behandlingen finner sted.

Denne artikkelen viser hvor langt forbindelsen selv kan spores, hva som er offentlig kjent om Anthropics datasentre, og hvilke veier som på en bindende måte lar behandlingen begrenses til EU eller området EU pluss Sveits. Prisene og regionene er per 2. oktober 2026.

**Sveits-fokus:** Den reviderte personvernloven (revDSG) er avgjørende. Overføring av personopplysninger til utlandet er tillatt etter art. 16 revDSG dersom Forbundsrådet bekrefter at mottakerstaten har tilstrekkelig vern (vedlegg 1 til personvernforordningen, DSV), eller dersom egnede garantier som standard kontraktsklausuler foreligger. EU- og EØS-statene står på denne listen, mens USA kun gjør det for selskaper som er sertifisert etter Swiss-U.S. Data Privacy Framework. Et behandlingssted i EU gjør dermed begrunnelsen betydelig enklere, men erstatter ikke de øvrige pliktene (databehandleravtale, informasjon til de berørte personene, datasikkerhet).

**EU-merknad:** For virksomheter med sete i EU gjelder i stedet art. 44 flg. GDPR. EU-kommisjonen har bekreftet at Sveits har et tilstrekkelig personvernnivå (tilstrekkelighetsbeslutning 2000/518/EF, bekreftet i januar 2024); behandling i Zürich er derfor ikke en problematisk tredjelandsoverføring for EU-virksomheter.

## Hvilke stasjoner en forespørsel til Claude går gjennom

En forespørsel til `claude.ai` eller `api.anthropic.com` går gjennom tre stasjoner:

| Stasjon | Hvem driver den | Synlig for deg? |
|---|---|---|
| Edge-node (TLS-terminering, vern mot misbruk) | Cloudflare, på vegne av Anthropic | Ja, plassering kan leses av per header |
| Internt nettverk til backend | Cloudflare og Anthropic | Nei |
| Inferens (modellen beregner) | Anthropic, på datakraft fra AWS, Google Cloud og egne datasentre | Nei, kun grov angivelse `us` eller `global` |

Vertsnavnene `claude.ai` og `api.anthropic.com` peker begge til adressen `160.79.104.10`. Den tilhører prefikset `160.79.104.0/23`, som er registrert på Anthropic (AS399358) og ifølge RPKI er gyldig fra Anthropic. I global ruting kunngjøres prefikset imidlertid utelukkende via Cloudflare (AS13335): Anthropic bringer sine egne IP-adresser inn i Cloudflares Anycast-nettverk. Den samme adressen besvares dermed verden over fra hvert Cloudflare-sted, og forbindelsen lander på nærmeste Edge-node.

## Bestem Edge-noden selv

Cloudflare oppgir betjenende sted på en diagnoseside og i headeren `cf-ray`. Forkortelsen etter Ray-ID-en er en IATA-flyplasskode: `ZRH` står for Zürich, `GVA` for Genève, `FRA` for Frankfurt, `MRS` for Marseille.

```bash
curl -s https://api.anthropic.com/cdn-cgi/trace
```

De relevante linjene i utdataene er `colo=` (Edge-sted) og `loc=` (landet Cloudflare tilordner avsenderadressen din). I Windows gir PowerShell det samme:

```powershell
$trace = Invoke-WebRequest -Uri "https://api.anthropic.com/cdn-cgi/trace" -UseBasicParsing
$trace.Content
```

Du kan lese headeren med en HEAD-forespørsel:

```bash
curl -sI https://api.anthropic.com/ \
  | grep -i -E '^(server|cf-ray):'
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `-s` | Undertrykker fremdriftsvisning og feilmeldinger fra curl |
| `-I` | Sender en HEAD-forespørsel og viser bare svar-headerne |
| `grep -i` | Søker uten å skille mellom store og små bokstaver |
| `-E '^(server\|cf-ray):'` | Utvidet regulært uttrykk: bare linjer som begynner med `server:` eller `cf-ray:` |

</details>

Fra et sveitsisk nettverk kan du normalt forvente `ZRH` eller `GVA`. Hvis Internett-utgangen er et annet sted, for eksempel i et bedriftsnettverk med sentral utgang i utlandet, svarer Edge-noden der, for eksempel med `colo=FRA` eller `colo=MRS`. Edge-stedet følger altså nettverksutgangen din, ikke behandlingsstedet.

## Hvorfor traceroute stopper ved Edge

En traceroute viser veien frem til Edge-noden og ikke lenger:

```powershell
Test-NetConnection -ComputerName api.anthropic.com -TraceRoute -Hops 20
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `-ComputerName api.anthropic.com` | Målhodeverten for målingen |
| `-TraceRoute` | Finner ruterhoppene på veien til målet |
| `-Hops 20` | Maksimalt antall hopp som undersøkes |

</details>

På Linux og macOS tilsvarer dette `traceroute -n api.anthropic.com` (`-n` undertrykker DNS-oppløsning av hoppene). I målingen lå de siste hoppene før `160.79.104.10` i adresseområdet `162.158.0.0/15`, som tilhører Cloudflare. Deretter stopper sporet: Forbindelsen termineres ved Edge-noden, og videresendingen til backend skjer i en ny, intern forbindelse som ikke er synlig utenfra.

RIPEstat viser hvem prefikset tilhører og hvilket nettverk det kunngjøres gjennom:

```bash
curl -s "https://stat.ripe.net/data/prefix-overview/data.json?resource=160.79.104.10"
curl -s "https://stat.ripe.net/data/bgp-state/data.json?resource=160.79.104.0/23"
```

I det andre resultatet ender alle AS-stier på `13335 399358`: Cloudflare er den eneste upstream-leverandøren for Anthropic-prefikset.

Etter min undersøkelse finnes det ingen offentlig måling som sporer veien helt til et inferens-datasenter, og på grunn av termineringen ved Edge er det heller ikke mulig med nettverksverktøy. Måletjenester som llmlatency.dev måler bare svartiden til første byte ved Edge (per september 2026: median 96 ms fra US-Central, 199 ms fra Tyskland). Tiden til første token (Time to First Token) kunne riktignok sammenlignes fra flere steder, men den avhenger mer av modell, promptlengde og belastning enn av avstand. En slutning om et sted er derfor ikke pålitelig.

## Hva som er kjent om Anthropics datasentre

Anthropic driver foreløpig ikke et eget, offentlig tilordnet inferens-datasenter med adresse. Datakraften kommer fra partnerskap som Tim Cadenbach har samlet i TCDEV-bloggen:

| Sted / partner | Nøkkeltall | Kjent rolle |
|---|---|---|
| Project Rainier, New Carlisle (Indiana, USA), Amazon Web Services | Åpnet i oktober 2025, rundt 11 mrd. USD, omkring 500 000 Trainium2-brikker, over 2,2 GW planlagt | Anthropic er hovedleietaker; primært trening |
| Google Cloud | Avtale i oktober 2025 om opptil 1 mill. TPU-er, over 1 GW fra 2026 | Regioner ikke offentliggjort |
| Egne datasentre med Fluidstack | Annonsert i november 2025, 50 mrd. USD, steder i Texas og New York, oppstart fra 2026 | Fortsatt under oppbygging |

Trening og inferens skjer ikke nødvendigvis på samme sted. Inferens kan kjøre i vilkårlige AWS- eller Google Cloud-regioner, og Anthropic opplyser ikke hvilket sted som beregner svaret for en bruker i Sveits. Det eneste som er sikkert, er det personvernerklæringen fastslår: Avtalepart og behandlingsansvarlig for kunder fra EØS, Storbritannia og Sveits er Anthropic Ireland, Limited i Dublin; data overføres til servere i USA eller andre land utenfor EØS, basert på standard kontraktsklausuler.

## Styring direkte hos Anthropic: bare USA eller globalt

Claude API har siden modellene i generasjon 4.6 parameteren `inference_geo`. Den har nøyaktig to verdier:

| Verdi | Virkning | Pris |
|---|---|---|
| `global` (standard) | Inferens i en hvilken som helst tilgjengelig region | Listepris |
| `us` | Inferens utelukkende i USA | Listepris × 1,1 |

Det finnes ingen verdi for EU eller Sveits. Også lagringsstedet for Workspace (`workspace geo`) kan for tiden bare settes til `us`. Svaret oppgir i feltet `usage.inference_geo` hvilken innstilling som ble brukt, men ingen konkret region. For Haiku 4.5 og eldre modeller avviser API-et parameteren med feil 400.

For claude.ai og Claude Code med abonnement (Pro, Max, Team, Enterprise) finnes det ingen innstilling for behandlingssted. Gjennom Anthropics direkte tilbud kan behandlingen dermed verken begrenses til EU eller Sveits. Det er bare mulig gjennom skyleverandørene som driver Claude i sine egne regioner.

## AWS Bedrock: området EU pluss Sveits fra Zürich

AWS Bedrock driver Claude på AWS-infrastruktur. Ifølge AWS har Anthropic ingen tilgang til kundenes prompter, svar eller logger. Du velger regionen via kilderegionen for API-kallet og via Inference Profile:

| Endepunktstype | Prefiks for modell-ID | Behandlingssted |
|---|---|---|
| Global Cross-Region Inference | `global.` | Vilkårlig kommersiell AWS-region verden over |
| Geographic Cross-Region Inference | `eu.` | Bare regioner innenfor geografien |
| In-Region | uten prefiks | Bare den påkalte regionen |

For Sveits er regionen `eu-central-2` (Zürich) avgjørende. Der er Opus 5.5, Sonnet 5.5 og Haiku 4.5 tilgjengelige, men bare som Global- eller EU-profil og ikke In-Region. En profil som utelukkende beregner i Zürich, finnes ikke. Dersom EU-profilen kalles fra Zürich, fordeler AWS ifølge modellkortet (dokumentert for Haiku 4.5 og Sonnet 4.6) forespørslene på disse regionene:

| Region | Sted |
|---|---|
| `eu-central-2` | Zürich |
| `eu-central-1` | Frankfurt |
| `eu-north-1` | Stockholm |
| `eu-south-1` | Milano |
| `eu-south-2` | Spania |
| `eu-west-1` | Irland |
| `eu-west-3` | Paris |

Zürich er bare et mulig mål dersom forespørselen kommer fra Zürich. Behandlingen forblir dermed i området EU pluss Sveits. For Opus 5.5 og Sonnet 5.5 angir ikke modellkortet målregionene; API-et returnerer den faktiske listen:

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
| `--inference-profile-identifier` | ID-en til EU-profilen for Sonnet 5.5 |
| `--query "models[].modelArn"` | Viser bare modell-ARN-ene; regionen står i hvert ARN |

</details>

Data lagres som standard bare i kilderegionen. Et unntak er innhold som beholdes for misbruksdeteksjon: Det lagres i målregionen. Transport mellom regionene skjer kryptert over AWS-nettverket.

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
| `cloudtrail lookup-events` | Søker i CloudTrail-hendelser fra de siste 90 dagene |
| `--region eu-central-2` | Kilderegionen der kallene logges |
| `--lookup-attributes AttributeKey=EventSource,AttributeValue=bedrock.amazonaws.com` | Filtrerer på hendelser fra Bedrock |
| `--max-results 20` | Begrenser utdataene til 20 hendelser |
| `--query "Events[].CloudTrailEvent"` | Viser bare hele hendelsen som JSON-tekst |
| `--output text` | Utdata uten JSON-innpakning, én hendelse per linje |
| `jq -r '… \| @tsv'` | Henter ut tidspunkt, handling og behandlingsregion som tabulatorseparert linje |

</details>

I tillegg kan en Service Control Policy (SCP) i AWS Organizations blokkere alle regioner utenfor EU og Sveits. Hvis en målregion for en profil er blokkert, mislykkes forespørselen i stedet for å falle tilbake til en annen region. Med denne kombinasjonen er behandlingsstedet teknisk håndhevet og kan dokumenteres per forespørsel.

### Claude Code via Bedrock i EU

Claude Code kan også kjøres via Bedrock. I en `eu-*`-region velger Claude Code prefikset `eu.` automatisk:

```bash
export CLAUDE_CODE_USE_BEDROCK=1
export AWS_REGION=eu-central-2
claude
```

| Variabel | Virkning |
|---|---|
| `CLAUDE_CODE_USE_BEDROCK=1` | Bytter Claude Code fra Anthropic API til Bedrock |
| `AWS_REGION=eu-central-2` | Kilderegion Zürich; dette medfører EU-profilen |

Med `ANTHROPIC_BEDROCK_REGION_PREFIX` kan prefikset overstyres. AWS-pålogging skjer via de vanlige mekanismene (profil, SSO, miljøvariabler). Faktureringen skjer via AWS-kontoen, ikke via et Claude-abonnement.

## Google Vertex AI: bare EU, uttrykkelig uten Sveits

På Google Cloud (Vertex AI, nå under navnet Gemini Enterprise Agent Platform) er situasjonen mindre gunstig for Sveits:

| Endepunkt | Modeller (utvalg) | Behandlingssted |
|---|---|---|
| `global` | alle | Vilkårlig Google Cloud-region, uten garanti |
| Multiregion `eu` (`aiplatform.eu.rep.googleapis.com`) | Opus 5.5, Sonnet 5.5, Opus 5, Sonnet 5, Opus 4.8, Opus 4.7 | Bare EU-medlemsstater |
| Region `europe-west1` (Belgia) | Haiku 4.5, Sonnet 4.6, Opus 4.6 og eldre | Ifølge modellsiden multiregion Europa |
| Region `europe-west6` (Zürich) | ingen | Claude ikke tilgjengelig |

Google utelukker uttrykkelig Sveits fra EU-multiregion-endepunktet: Det dekker bare EU-medlemsstater, Storbritannia og Sveits er ikke inkludert. Flere sveitsiske veiledningssider nevner `europe-west6` som en vei til Claude i Zürich; ifølge Googles offisielle stedstabell er ingen Claude-modell tilgjengelig der. Google dokumenterer ikke et loggfelt med den faktiske behandlingsregionen per forespørsel. For det globale endepunktet fastslår Google at regionen for behandlingen verken kan styres eller fastslås.

Vertex AI egner seg dermed for behandling utelukkende i EU, men ikke for behandling i Sveits.

## Microsoft Foundry: for tiden ingen EU-løsning

Microsoft tilbyr Claude i Foundry i to varianter. «Hosted on Azure» beregner på Azure-infrastruktur, enten som Global Standard eller Data Zone Standard; Data Zone finnes for Claude bare i USA. «Hosted on Anthropic» beregner på Anthropics infrastruktur, og Microsoft opplyser at data kan behandles utenfor Azure og utenfor den valgte regionen. Anthropic oppgir Foundry for Europa som «Coming soon». I Microsoft 365 Copilot og Copilot Studio er Anthropic-modellene unntatt fra EU Data Boundary og som standard deaktivert i EU, EFTA (også i Sveits) og Storbritannia.

## Oversikt: hvilken vei garanterer hvilket behandlingssted

| Vei | Bare Sveits | Bare EU | Området EU pluss Sveits | Region per forespørsel dokumenterbar |
|---|---|---|---|---|
| claude.ai, Claude Code (abonnement) | nei | nei | nei | nei |
| Claude API med `inference_geo` | nei | nei | nei (bare `us`) | bare `us`/`global` |
| AWS Bedrock, kilde `eu-central-2`, profil `eu.` | nei | nei | ja | ja (CloudTrail) |
| AWS Bedrock, kilde i EU, profil `eu.` | nei | ja | ja | ja (CloudTrail) |
| Google Vertex AI, endepunkt `eu` | nei | ja | (bare EU-delen) | nei |
| Microsoft Foundry | nei | nei | nei | nei |

Ingen leverandør tilbyr for tiden en vei som garanterer at Claudes behandling begrenses til Sveits. AWS Bedrock med kilderegion Zürich og EU-profil kommer nærmest. Dersom det i tillegg må utelukkes at data forlater EU (for eksempel på grunn av kontraktsforpliktelser overfor EU-kunder), kan EU-profilen kalles fra en EU-region som Frankfurt; Zürich er da ikke lenger et mål.

## Anonymisering før modellen: pilotselskaper søkes

Behandlingsstedet er i de fleste tilfeller bare relevant fordi prompten inneholder personopplysninger eller fortrolig informasjon. Det er her et nytt produkt fra selskapet mitt, adeptio gmbh, kommer inn: Det fungerer som et mellomlag mellom applikasjonene i Sveits og modellene fra Anthropic og OpenAI og anonymiserer forespørslene før de forlater Sveits. Modellen mottar bare innhold uten personhenførbarhet, uansett hvilket datasenter den beregner i.

**Sveits-fokus:** Opplysninger som ikke lenger kan knyttes til en bestemt eller identifiserbar person, er ikke personopplysninger i betydningen av art. 5 bokstav a revDSG. Reglene for overføring til utlandet (art. 16 flg. revDSG) gjelder ikke for slikt innhold. Om en anonymisering når denne terskelen, avhenger av gjennomføringen og de konkrete dataene; det er nettopp dette vi vil dokumentere i live-drift.

For innføringen søker vi virksomheter som bruker Claude eller modeller fra OpenAI i produksjon, eller planlegger innføringen, og som vil bruke produktet som pilotselskap i hverdagen. Begge parter drar nytte av dette:

| Din virksomhet | adeptio |
|---|---|
| Personopplysninger og fortrolig informasjon forlater Sveits bare anonymisert. Begrunnelsen overfor personvernrådgivning, kunder og revisjon blir enklere. | Innsikt fra ekte prompter og arbeidsprosesser i stedet for testdata: hvilke opplysninger som må gjenkjennes, og hvor anonymiseringen fjerner for mye eller for lite. |
| Valget mellom Anthropic, OpenAI, Bedrock eller Vertex AI baseres igjen på modellutvalg og pris i stedet for bare behandlingsstedet. | Måledata om ventetid og svarkvalitet under reell belastning. |
| Tilgang til produktet før generell tilgjengelighet og innflytelse på hvilke datatyper og bruksområder det først dekker. | Tilbakemeldinger fra personvern og revisjon, som produktets dokumentasjon kan innrettes etter. |
| Direkte kontakt med utviklerne i stedet for billettbasert support. | Referanser fra det sveitsiske markedet for generell tilgjengelighet. |

<aside class="offer-box">
  <span class="offer-box__tag">Pilotselskaper søkes</span>
  <p><strong>Bruker du Claude eller OpenAI med personopplysninger?</strong> Ta i bruk adeptios anonymiseringslag i live-drift, og vær med på å bestemme videreutviklingen.</p>
  <a class="offer-box__cta" href="/kontakt">Ta kontakt</a>
</aside>

## Hva regional binding koster

Alle de tre skyleverandørene og Anthropic selv krever et påslag på 10 % sammenlignet med det globale endepunktet for binding til en region eller geografi. Hos Bedrock bestemmes prisene av kilderegionen; Zürich, Frankfurt og North Virginia koster det samme. Ruting mellom regioner koster ikke ekstra.

Listepriser i USD per 1 mill. token (input / output):

| Modell | Anthropic API, `global` | Anthropic API, `us` | Bedrock eller Vertex, globalt | Bedrock `eu.` eller Vertex `eu` |
|---|---|---|---|---|
| Opus 5.5 | 4.00 / 20.00 | 4.40 / 22.00 | 4.00 / 20.00 | 4.40 / 22.00 |
| Sonnet 5.5 | 2.00 / 10.00 | 2.20 / 11.00 | 2.00 / 10.00 | 2.20 / 11.00 |
| Haiku 4.5 | 1.00 / 5.00 | ikke tilgjengelig | 1.00 / 5.00 | 1.10 / 5.50 |
| Sonnet 4.6 | 3.00 / 15.00 | 3.30 / 16.50 | 3.00 / 15.00 | 3.30 / 16.50 |

Hos Vertex AI gjelder prisen i EU-kolonnen for Haiku 4.5 og Sonnet 4.6 i regionen `europe-west1`. Et regneeksempel for en intern applikasjon med 50 mill. input- og 10 mill. output-token per måned:

| Modell | Globalt | EU-bundet | Ekstrakostnad per måned |
|---|---|---|---|
| Sonnet 5.5 | 50 × 2 + 10 × 10 = 200 USD | 220 USD | 20 USD |
| Opus 5.5 | 50 × 4 + 10 × 20 = 400 USD | 440 USD | 40 USD |

Den regionale bindingen koster altså 10 % mer enn det globale endepunktet. Sammenlignet med listeprisen direkte hos Anthropic med `inference_geo: "us"` koster en EU-profil på Bedrock det samme. Det som heller veier tungt ved valget, er tilleggskostnadene: En AWS- eller Google Cloud-konto med organisasjonspolicyer, logging og budsjettvarsler må bygges opp og driftes, og de faste månedsprisene for Claude-abonnementene erstattes av ren forbruksavregning. Om byttet lønner seg, avhenger av volumet; et abonnement har en fast pris per person, men gir ingen kontroll over behandlingsstedet.

## Oppbevaring og trening

Ved siden av stedet teller også hvor lenge data lagres:

- **Anthropic API:** Prompter og svar brukes ikke til trening så lenge det ikke foreligger uttrykkelig samtykke. Zero Data Retention (ZDR) er tilgjengelig på forespørsel per organisasjon og kan kombineres med `inference_geo`. Innhold som misbruksdeteksjonen flagger, lagres i opptil to år også med ZDR. For modellene Fable og Mythos er oppbevaring i 30 dager obligatorisk.
- **AWS Bedrock:** Innhold sendes ikke til Anthropic. Oppbevaring kan velges per modell mellom `none`, `default` og `aws_review`; Fable krever `aws_review` med opptil 30 dagers lagring for kontroll av AWS.
- **Google Vertex AI:** Google bruker dataene bare med forhåndssamtykke til trening eller finjustering.

## Kilder

1.  [Claude Platform Docs: Data residency](https://platform.claude.com/docs/en/manage-claude/data-residency): Parameter `inference_geo`, verdier `us` og `global`, Workspace geo, støttede modeller, felt `usage.inference_geo`.

2.  [Claude Platform Docs: Pricing](https://platform.claude.com/docs/en/about-claude/pricing): Listepriser for Anthropic API og faktor 1,1 for `inference_geo: "us"`.

3.  [Claude Platform Docs: API and data retention](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention): Zero Data Retention, unntak for flagget innhold og for Fable/Mythos.

4.  [Anthropic: Privacy Policy](https://www.anthropic.com/legal/privacy): Anthropic Ireland som behandlingsansvarlig for EØS, UK og Sveits, overføring til USA på grunnlag av standard kontraktsklausuler.

5.  [Anthropic: Regional compliance](https://claude.com/regional-compliance): Oversikt over hvilken plattform som tilbyr dataresidens i hvilken region; Foundry Europa som «Coming soon».

6.  [Claude Platform Docs: Claude in Amazon Bedrock](https://platform.claude.com/docs/en/build-with-claude/claude-in-amazon-bedrock): Endepunktstyper per region, 10 % påslag for regionale endepunkter.

7.  [AWS: Modellkort Claude Sonnet 4.6](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-4-6.html): Målregioner for EU-profilen per kilderegion, inkludert Zürich.

8.  [AWS: Modellkort Claude Sonnet 5.5](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-5-5.html): Tilgjengelighet i `eu-central-2` bare som Geo- og Global-profil.

9.  [AWS: Geographic cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/geographic-cross-region-inference.html): Data forblir innenfor geografien, lagringssted, blokkering av målregioner med SCP.

10.  [AWS: Cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html): Felt `additionalEventData.inferenceRegion` i CloudTrail, pris etter kilderegion.

11.  [AWS: Data protection in Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/data-protection.html): Modellleverandørene har ingen tilgang til prompter, svar og logger.

12.  [AWS: Amazon Bedrock Pricing](https://aws.amazon.com/bedrock/pricing/): Listepriser for Global- og Geo-profiler, identiske for Zürich, Frankfurt og North Virginia.

13.  [AWS Alps Blog: Cross-region inference for EU data processing in Switzerland](https://aws.amazon.com/blogs/alps/unlocking-ai-flexibility-in-switzerland-a-guide-to-cross-region-inference-for-eu-data-processing-and-model-access/): Veiledning fra AWS Sveits om EU-profilen fra Zürich.

14.  [Claude Code Docs: Amazon Bedrock](https://code.claude.com/docs/en/amazon-bedrock): Miljøvariabler og automatisk prefiks `eu.` i EU-regioner.

15.  [Google Cloud: Generative AI locations](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/locations): Stedstabell for Claude-modellene, ingen tilgjengelighet i `europe-west6`.

16.  [Google Cloud: Data residency](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/data-residency): EU-multiregion kun for EU-medlemsstater, Sveits og UK utelukket.

17.  [Google Cloud: Vertex AI Pricing](https://cloud.google.com/vertex-ai/generative-ai/pricing): Listepriser globalt, EU-multiregion og `europe-west1`.

18.  [Microsoft Learn: Claude models hosting comparison](https://learn.microsoft.com/en-us/azure/foundry/foundry-models/concepts/claude-models-hosting-comparison): Hosted on Azure og Hosted on Anthropic, Data Zone kun USA.

19.  [Microsoft Learn: Anthropic as AI subprocessor in Microsoft 365 Copilot](https://learn.microsoft.com/en-us/microsoft-365/copilot/connect-to-ai-subprocessor): Unntak fra EU Data Boundary, standardinnstilling i EU/EFTA.

20.  [TCDEV Blog: Where Are Claude's Data Centers?](https://www.tcdev.de/blog/where-are-claudes-data-centers/): Tim Cadenbachs oversikt over Project Rainier, Google-TPU-avtalen og egne datasentre med Fluidstack.

21.  [Anthropic: Investering på 50 mrd. USD i amerikansk infrastruktur](https://www.anthropic.com/news/anthropic-invests-50-billion-in-american-ai-infrastructure): Annonsering av egne datasentre i Texas og New York med Fluidstack.

22.  [Claude Platform Docs: IP addresses](https://platform.claude.com/docs/en/api/ip-addresses): Inngående adresseområde `160.79.104.0/23`.

23.  [RIPEstat](https://stat.ripe.net/): Prefiksoversikt og BGP-status for `160.79.104.0/23` (AS399358, upstream AS13335).

24.  [llmlatency.dev: Anthropic](https://llmlatency.dev/provider/anthropic): Svartider ved Edge etter målested.

25.  [Fedlex: Personvernforordning (DSV), vedlegg 1](https://www.fedlex.admin.ch/eli/cc/2022/568/de): Liste over stater med tilstrekkelig personvern, inkludert EU/EØS og USA for sertifiserte selskaper.
