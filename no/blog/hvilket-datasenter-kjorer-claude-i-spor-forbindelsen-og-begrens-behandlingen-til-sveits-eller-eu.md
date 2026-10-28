---
title: "Hvilket datasenter kjører Claude i? Spor forbindelsen og begrens behandlingen til Sveits eller EU"
navTitle: "Claude-datasenter"
description: "Hvilke stasjoner en forespørsel til Claude passerer, hvorfor sporet ender ved Cloudflare-kanten, og hva Anthropic opplyser om sine datasentre. I tillegg sammenlignes alternativer med fast behandlingssted (Anthropic API, AWS Bedrock, Google Vertex AI, Microsoft Foundry), begrensningene ved drift kun i Sveits og kostnaden ved regional binding."
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
translationSourceHash: 2d2a3c4b5820e6ec99b710813e209ed3b2e6f6fa750028d120a7d5e4bf4e0c17
translationModel: gpt-5.6-terra
translatedAt: 2026-10-27T09:10:10.999Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/no/blog/hvilket-datasenter-kjorer-claude-i-spor-forbindelsen-og-begrens-behandlingen-til-sveits-eller-eu
---

Når du bruker Claude via claude.ai, Claude Code eller API-et, får du ikke vite hvilket datasenter som behandler forespørselen. Bare den første delen av forbindelsen til nærmeste Edge-node er synlig. For virksomheter som overfører personopplysninger eller fortrolig innhold til en språkmodell, er dette for lite: De må kunne dokumentere overfor kunder, personvernrådgivere eller revisjonen hvor behandlingen finner sted.

Dette innlegget viser hvor langt selve forbindelsen kan spores, hva som er offentlig kjent om Anthropics datasentre, og hvilke veier som gjør det mulig å begrense behandlingen bindende til EU eller området EU pluss Sveits. Prisene og regionene er oppdatert per 25. oktober 2026.

**Sveits-fokus:** Det reviderte personvernlovverket (revDSG) er avgjørende. Overføring av personopplysninger til utlandet er tillatt etter art. 16 revDSG dersom Forbundsrådet bekrefter at mottakerlandet har tilstrekkelig beskyttelse (vedlegg 1 til personvernforordningen, DSV), eller dersom det finnes egnede garantier som standard kontraktsklausuler. EU- og EØS-statene står på denne listen, mens USA kun gjør det for selskaper som er sertifisert etter Swiss-U.S. Data Privacy Framework. Et behandlingssted i EU forenkler derfor begrunnelsen betydelig, men erstatter ikke de øvrige pliktene (databehandleravtale, informasjon til de registrerte, datasikkerhet).

**EU-merknad:** For virksomheter med hovedsete i EU gjelder i stedet art. 44 ff. GDPR. EU-kommisjonen har bekreftet at Sveits har et tilstrekkelig personvernnivå (tilstrekkelighetsbeslutning 2000/518/EF, bekreftet i januar 2024); behandling i Zürich er dermed ikke en problematisk tredjelandsoverføring for EU-virksomheter.

<aside class="pilot-card">
  <span class="pilot-card__tag">Nytt fra adeptio · Pilotvirksomheter søkes</span>
  <p class="pilot-card__titel">Bruk Claude og OpenAI uten at personopplysninger forlater Sveits</p>
  <p class="pilot-card__lead">Mellomlaget vårt står mellom applikasjonene dine i Sveits og modellene fra Anthropic og OpenAI, og anonymiserer hver forespørsel før den forlater landet. For live-drift søker vi virksomheter som bruker produktet i hverdagen.</p>
  <div class="pilot-card__spalten">
    <div class="pilot-card__spalte">
      <p class="pilot-card__spalte-titel">Din virksomhet</p>
      <ul>
        <li>Personopplysninger forlater Sveits kun i anonymisert form</li>
        <li>Enklere begrunnelse overfor personvern og revisjon</li>
        <li>Tilgang før generell tilgjengelighet og innflytelse på funksjonene</li>
        <li>Direkte kontakt med utviklerne</li>
      </ul>
    </div>
    <div class="pilot-card__spalte">
      <p class="pilot-card__spalte-titel">adeptio</p>
      <ul>
        <li>Innsikt fra virkelige prompter i stedet for testdata</li>
        <li>Måleverdier for forsinkelse og svarkvalitet under reell belastning</li>
        <li>Referanser fra det sveitsiske markedet</li>
      </ul>
    </div>
  </div>
  <div class="pilot-card__aktionen">
    <a class="pilot-card__cta" href="/kontakt">Meld deg som pilotvirksomhet</a>
    <a class="pilot-card__mehr" href="#anonymisierung-vor-dem-modell-pilotunternehmen-gesucht">Detaljer og juridisk vurdering</a>
  </div>
</aside>

## Hvilke stasjoner en forespørsel til Claude passerer

En forespørsel til `claude.ai` eller `api.anthropic.com` går via tre stasjoner:

| Stasjon | Hvem drifter den | Synlig for deg? |
|---|---|---|
| Edge-node (TLS-terminering, beskyttelse mot misbruk) | Cloudflare, på vegne av Anthropic | Ja, plassering kan leses av i headeren |
| Internt nettverk til backend | Cloudflare og Anthropic | Nei |
| Inferens (modellen beregner) | Anthropic, med datakraft fra AWS, Google Cloud og egne datasentre | Nei, kun den grove angivelsen `us` eller `global` |

Vertsnavnene `claude.ai` og `api.anthropic.com` peker begge til adressen `160.79.104.10`. Den tilhører prefikset `160.79.104.0/23`, som er registrert på Anthropic (AS399358) og ifølge RPKI er gyldig fra Anthropic. I global ruting kunngjøres prefikset imidlertid utelukkende via Cloudflare (AS13335): Anthropic tar sine egne IP-adresser inn i Cloudflares Anycast-nettverk. Den samme adressen besvares dermed fra alle Cloudflare-lokasjoner verden over, og forbindelsen havner hos nærmeste Edge-node.

## Bestem Edge-noden selv

Cloudflare oppgir den betjenende lokasjonen på en diagnoseside og i headeren `cf-ray`. Forkortelsen etter Ray-ID-en er en IATA-flyplasskode: `ZRH` står for Zürich, `GVA` for Genève, `FRA` for Frankfurt og `MRS` for Marseille.

```bash
curl -s https://api.anthropic.com/cdn-cgi/trace
```

I utdataene er linjene `colo=` (Edge-lokasjon) og `loc=` (landet Cloudflare tilordner avsenderadressen din) relevante. I Windows gir PowerShell det samme:

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
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-s` | Undertrykker fremdriftsvisning og feilmeldinger fra curl |
| `-I` | Sender en HEAD-forespørsel og skriver kun ut responsheaderne |
| `grep -i` | Søker uten hensyn til store og små bokstaver |
| `-E '^(server\|cf-ray):'` | Utvidet regulært uttrykk: kun linjer som begynner med `server:` eller `cf-ray:` |

</details>

Fra et sveitsisk nettverk kan man normalt forvente `ZRH` eller `GVA`. Dersom Internett-utgangen er et annet sted, for eksempel i et bedriftsnettverk med sentral utgang i utlandet, svarer Edge-noden der, for eksempel med `colo=FRA` eller `colo=MRS`. Edge-lokasjonen følger altså nettverksutgangen din, ikke behandlingsstedet.

## Hvorfor traceroute ender ved Edge

En traceroute viser veien til Edge-noden, men ikke lenger:

```powershell
Test-NetConnection -ComputerName api.anthropic.com -TraceRoute -Hops 20
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-ComputerName api.anthropic.com` | Målingens målvert |
| `-TraceRoute` | Finner ruterhoppene på veien til målet |
| `-Hops 20` | Maksimalt antall hopp som kontrolleres |

</details>

I Linux og macOS tilsvarer dette `traceroute -n api.anthropic.com` (`-n` undertrykker DNS-oppløsning av hoppene). I målingen lå de siste hoppene før `160.79.104.10` i adresseområdet `162.158.0.0/15`, som tilhører Cloudflare. Deretter ender sporet: Forbindelsen termineres ved Edge-noden, og videresendingen til backend skjer i en ny intern forbindelse som ikke er synlig utenfra.

RIPEstat viser hvem prefikset tilhører og hvilket nettverk det kunngjøres via:

```bash
curl -s "https://stat.ripe.net/data/prefix-overview/data.json?resource=160.79.104.10"
curl -s "https://stat.ripe.net/data/bgp-state/data.json?resource=160.79.104.0/23"
```

I det andre resultatet ender alle AS-stier på `13335 399358`: Cloudflare er den eneste upstream-leverandøren til Anthropic-prefikset.

Ifølge min undersøkelse finnes det ingen offentlig måling som sporer veien helt til et inferensdatasenter, og på grunn av termineringen ved Edge er dette heller ikke mulig med nettverksverktøy. Måletjenester som llmlatency.dev registrerer kun responstiden frem til første byte ved Edge (per september 2026: 96 ms median fra US-Central, 199 ms fra Tyskland). Tiden til første token (Time to First Token) kunne riktignok sammenlignes fra flere steder, men den avhenger mer av modell, promptlengde og belastning enn av avstand. En slutning om en lokasjon er derfor ikke robust.

## Hva som er kjent om Anthropics datasentre

Anthropic driver foreløpig ikke et eget inferensdatasenter med offentlig oppgitt adresse. Datakraften kommer fra partnerskap som Tim Cadenbach har samlet i TCDEV-bloggen:

| Lokasjon / partner | Nøkkeltall | Kjent rolle |
|---|---|---|
| Project Rainier, New Carlisle (Indiana, USA), Amazon Web Services | Åpnet oktober 2025, rundt 11 mrd. USD, omtrent 500 000 Trainium2-brikker, over 2,2 GW planlagt | Anthropic er hovedleietaker; hovedsakelig trening |
| Google Cloud | Avtale fra oktober 2025 om opptil 1 mill. TPU-er, over 1 GW fra 2026 | Regioner ikke offentliggjort |
| Egne datasentre med Fluidstack | Annonsert november 2025, 50 mrd. USD, lokasjoner i Texas og New York, igangsetting fra 2026 | Fortsatt under oppbygging |

Trening og inferens foregår ikke nødvendigvis på samme sted. Inferens kan kjøre i vilkårlige AWS- eller Google Cloud-regioner, og Anthropic opplyser ikke hvilken lokasjon som beregner svaret for en bruker i Sveits. Bare det som fremgår av personvernerklæringen, er sikkert: Avtalepart og behandlingsansvarlig for kunder i EØS, Storbritannia og Sveits er Anthropic Ireland, Limited i Dublin; data overføres til servere i USA eller andre land utenfor EØS basert på standard kontraktsklausuler.

## Styring direkte hos Anthropic: bare USA eller globalt

Claude API har siden modellene i generasjon 4.6 parameteren `inference_geo`. Den har nøyaktig to verdier:

| Verdi | Virkning | Pris |
|---|---|---|
| `global` (standard) | Inferens i enhver tilgjengelig region | Listepris |
| `us` | Inferens utelukkende i USA | Listepris × 1,1 |

Det finnes ingen verdi for EU eller Sveits. Også lagringsstedet for arbeidsområdet (`workspace geo`) kan foreløpig kun settes til `us`. Svaret oppgir i feltet `usage.inference_geo` hvilken innstilling som ble brukt, men ingen konkret region. For Haiku 4.5 og eldre modeller avviser API-et parameteren med feil 400.

For claude.ai og Claude Code med abonnement (Pro, Max, Team, Enterprise) finnes det ingen innstilling for behandlingsstedet. Gjennom Anthropics direkte tilbud kan behandlingen dermed verken begrenses til EU eller Sveits. Dette er bare mulig via skyleverandørene som driver Claude i sine egne regioner.

## AWS Bedrock: området EU pluss Sveits fra Zürich

AWS Bedrock driver Claude på AWS-infrastruktur. Ifølge AWS har Anthropic ikke tilgang til kundenes prompter, svar eller logger. Du velger regionen via kilderegionen for API-kallet og via Inference Profile:

| Endepunktstype | Prefiks for modell-ID | Behandlingssted |
|---|---|---|
| Global Cross-Region Inference | `global.` | Enhver kommersiell AWS-region verden over |
| Geographic Cross-Region Inference | `eu.` | Kun regioner innenfor geografien |
| In-Region | uten prefiks | Kun den forespurte regionen |

For Sveits er regionen `eu-central-2` (Zürich) avgjørende. Der er Opus 5.5, Sonnet 5.5, Haiku 4.5 og siden 7. oktober 2026 Haiku 5.5 tilgjengelige, men kun som Global- eller EU-profil og ikke In-Region. Det finnes ingen profil som beregner utelukkende i Zürich. Hvis EU-profilen kalles fra Zürich, fordeler AWS forespørslene i henhold til modellkortet (dokumentert for Opus 5.5, Sonnet 5.5, Haiku 5.5, Haiku 4.5 og Sonnet 4.6) på disse regionene:

| Region | Lokasjon |
|---|---|
| `eu-central-2` | Zürich |
| `eu-central-1` | Frankfurt |
| `eu-north-1` | Stockholm |
| `eu-south-1` | Milano |
| `eu-south-2` | Spania |
| `eu-west-1` | Irland |
| `eu-west-3` | Paris |

Zürich er bare et mulig mål dersom forespørselen kommer fra Zürich. Behandlingen forblir dermed i området EU pluss Sveits. Listen som API-et returnerer for din egen kilderegion, er bindende:

```bash
aws bedrock get-inference-profile \
  --region eu-central-2 \
  --inference-profile-identifier eu.anthropic.claude-sonnet-5-5 \
  --query "models[].modelArn"
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `bedrock get-inference-profile` | Leser definisjonen av en Inference Profile |
| `--region eu-central-2` | Kilderegion Zürich; mållisten avhenger av kilderegionen |
| `--inference-profile-identifier` | ID-en til EU-profilen for Sonnet 5.5 |
| `--query "models[].modelArn"` | Skriver kun ut modell-ARN-ene; regionen står i hvert ARN |

</details>

Data lagres som standard bare i kilderegionen. Et unntak er innhold som beholdes for misbruksdeteksjon: Det ligger i målregionen. Transporten mellom regionene er kryptert over AWS-nettverket.

### Dokumenter behandlingsregionen per forespørsel

Bedrock er den eneste av de beskrevne alternativene som logger den faktiske behandlingsregionen per forespørsel. CloudTrail skriver den i kilderegionen i feltet `additionalEventData.inferenceRegion`:

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
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `cloudtrail lookup-events` | Søker gjennom CloudTrail-hendelser fra de siste 90 dagene |
| `--region eu-central-2` | Kilderegionen der kallene logges |
| `--lookup-attributes AttributeKey=EventSource,AttributeValue=bedrock.amazonaws.com` | Filtrerer på hendelser fra Bedrock |
| `--max-results 20` | Begrenser utdataene til 20 hendelser |
| `--query "Events[].CloudTrailEvent"` | Skriver bare ut hele hendelsen som JSON-tekst |
| `--output text` | Utdata uten JSON-omslag, én hendelse per linje |
| `jq -r '… \| @tsv'` | Henter ut tidspunkt, handling og behandlingsregion som en tabulatorseparert linje |

</details>

I tillegg kan en Service Control Policy (SCP) i AWS Organizations sperre alle regioner utenfor EU og Sveits. Dersom en målregion i en profil er sperret, mislykkes forespørselen i stedet for å falle tilbake til en annen region. Med denne kombinasjonen er behandlingsstedet teknisk påtvunget og kan dokumenteres per forespørsel.

### Claude Code via Bedrock i EU

Claude Code kan også kjøres via Bedrock. I en `eu-*`-region velger Claude Code automatisk prefikset `eu.`:

```bash
export CLAUDE_CODE_USE_BEDROCK=1
export AWS_REGION=eu-central-2
claude
```

| Variabel | Virkning |
|---|---|
| `CLAUDE_CODE_USE_BEDROCK=1` | Bytter Claude Code fra Anthropic API til Bedrock |
| `AWS_REGION=eu-central-2` | Kilderegion Zürich; dette gir EU-profilen |

Med `ANTHROPIC_BEDROCK_REGION_PREFIX` kan prefikset overstyres. AWS-autentiseringen skjer via de vanlige mekanismene (profil, SSO, miljøvariabler). Faktureringen skjer via AWS-kontoen, ikke via et Claude-abonnement.

## Google Vertex AI: kun EU, uttrykkelig uten Sveits

På Google Cloud (Vertex AI, nå under navnet Gemini Enterprise Agent Platform) er situasjonen mindre gunstig for Sveits:

| Endepunkt | Modeller (utvalg) | Behandlingssted |
|---|---|---|
| `global` | alle | Enhver Google Cloud-region, uten garanti |
| Multiregion `eu` (`aiplatform.eu.rep.googleapis.com`) | Opus 5.5, Sonnet 5.5, Haiku 5.5, Opus 5, Sonnet 5, Opus 4.8, Opus 4.7 | Kun EU-medlemsstater |
| Region `europe-west1` (Belgia) | Haiku 4.5, Sonnet 4.6, Opus 4.6 og eldre | Ifølge modellsiden multiregion Europa |
| Region `europe-west6` (Zürich) | ingen | Claude er ikke tilgjengelig |

Google utelukker uttrykkelig Sveits fra EU-multiregion-endepunktet: Det dekker kun EU-medlemsstater; Storbritannia og Sveits er ikke inkludert. Flere sveitsiske veiledningssider oppgir `europe-west6` som vei til Claude i Zürich; ifølge Googles offisielle lokasjonstabell er ingen Claude-modell tilgjengelig der. Google dokumenterer ikke et loggfelt med den faktiske behandlingsregionen per forespørsel. For det globale endepunktet fastslår Google at behandlingsregionen verken kan styres eller fastslås.

Vertex AI egner seg dermed for behandling utelukkende i EU, men ikke for behandling i Sveits.

## Microsoft Foundry: foreløpig ingen EU-løsning

Microsoft tilbyr Claude i Foundry i to varianter. «Hosted on Azure» kjører på Azure-infrastruktur, som Global Standard eller Data Zone Standard; Data Zone finnes for Claude kun for USA. «Hosted on Anthropic» kjører på Anthropic-infrastruktur, og Microsoft opplyser at data kan behandles utenfor Azure og utenfor den valgte regionen. Anthropic oppfører Foundry for Europa som «Coming soon». I Microsoft 365 Copilot og Copilot Studio er Anthropic-modellene unntatt fra EU Data Boundary og som standard deaktivert i EU, EFTA (også Sveits) og Storbritannia.

## Oversikt: hvilken vei garanterer hvilket behandlingssted

| Alternativ | Kun Sveits | Kun EU | Området EU pluss Sveits | Region per forespørsel kan dokumenteres |
|---|---|---|---|---|
| claude.ai, Claude Code (abonnement) | nei | nei | nei | nei |
| Claude API med `inference_geo` | nei | nei | nei (kun `us`) | kun `us`/`global` |
| AWS Bedrock, kilde `eu-central-2`, profil `eu.` | nei | nei | ja | ja (CloudTrail) |
| AWS Bedrock, kilde i EU, profil `eu.` | nei | ja | ja | ja (CloudTrail) |
| Google Vertex AI, endepunkt `eu` | nei | ja | (kun EU-delen) | nei |
| Microsoft Foundry | nei | nei | nei | nei |

Ingen leverandør tilbyr for tiden en vei som garanterer at behandlingen av Claude begrenses til Sveits. AWS Bedrock med kilderegion Zürich og EU-profil kommer nærmest. Dersom det i tillegg må utelukkes at data forlater EU, for eksempel på grunn av kontraktsmessige forpliktelser overfor EU-kunder, kan EU-profilen kalles fra en EU-region som Frankfurt; Zürich er da ikke lenger et mål.

## Anonymisering før modellen: pilotvirksomheter søkes

Behandlingsstedet er i de fleste tilfeller bare relevant fordi prompten inneholder personopplysninger eller fortrolige opplysninger. Her kommer et nytt produkt fra selskapet mitt, adeptio gmbh, inn: Det fungerer som et mellomlag mellom applikasjonene i Sveits og modellene fra Anthropic og OpenAI, og anonymiserer forespørslene før de forlater Sveits. Modellen mottar dermed bare innhold uten personreferanse, uansett hvilket datasenter den kjører i.

**Sveits-fokus:** Opplysninger som ikke lenger kan knyttes til en bestemt eller identifiserbar person, er ikke personopplysninger i betydningen av art. 5 bokstav a revDSG. Reglene om overføring til utlandet (art. 16 f. revDSG) gjelder ikke for slikt innhold. Om en anonymisering når denne terskelen, avhenger av gjennomføringen og de konkrete dataene; nettopp dette vil vi dokumentere i live-drift.

For lanseringen søker vi virksomheter som bruker Claude eller modeller fra OpenAI i produksjon, eller planlegger innføringen, og som tar produktet i bruk som pilotvirksomhet i hverdagen. Begge parter drar nytte av dette:

| Din virksomhet | adeptio |
|---|---|
| Personopplysninger og fortrolige opplysninger forlater Sveits kun i anonymisert form. Begrunnelsen overfor personvernrådgivning, kunder og revisjon blir enklere. | Innsikt fra virkelige prompter og arbeidsprosesser i stedet for testdata: hvilke opplysninger som må gjenkjennes, og hvor anonymiseringen fjerner for mye eller for lite. |
| Valget mellom Anthropic, OpenAI, Bedrock eller Vertex AI styres igjen av modellutvalg og pris, ikke bare av behandlingsstedet. | Måleverdier for forsinkelse og svarkvalitet under reell belastning. |
| Tilgang til produktet før generell tilgjengelighet og innflytelse på hvilke datatyper og brukstilfeller det først dekker. | Tilbakemeldinger fra personvern og revisjon som produktets dokumentasjon kan innrettes etter. |
| Direkte kontakt med utviklerne i stedet for ticketsupport. | Referanser fra det sveitsiske markedet for generell tilgjengelighet. |

<aside class="offer-box">
  <span class="offer-box__tag">Pilotvirksomheter søkes</span>
  <p><strong>Bruker du Claude eller OpenAI med personopplysninger?</strong> Ta i bruk adeptios anonymiseringslag i live-drift og vær med på å styre videreutviklingen.</p>
  <a class="offer-box__cta" href="/kontakt">Ta kontakt</a>
</aside>

## Hva regional binding koster

Alle de tre skyleverandørene og Anthropic selv krever et påslag på 10 % sammenlignet med det globale endepunktet for binding til en region eller geografi. Prisene hos Bedrock bestemmes av kilderegionen; Zürich, Frankfurt og North Virginia koster det samme. Ruting mellom regioner koster ikke ekstra.

Listepriser i USD per 1 mill. token (input / output):

| Modell | Anthropic API, `global` | Anthropic API, `us` | Bedrock eller Vertex, globalt | Bedrock `eu.` eller Vertex `eu` |
|---|---|---|---|---|
| Opus 5.5 | 4.00 / 20.00 | 4.40 / 22.00 | 4.00 / 20.00 | 4.40 / 22.00 |
| Sonnet 5.5 | 2.00 / 10.00 | 2.20 / 11.00 | 2.00 / 10.00 | 2.20 / 11.00 |
| Haiku 5.5 (prompter opptil 100 000 token) | 0.10 / 0.50 | 0.11 / 0.55 | 0.10 / 0.50 | 0.11 / 0.55 |
| Haiku 4.5 | 1.00 / 5.00 | ikke tilgjengelig | 1.00 / 5.00 | 1.10 / 5.50 |
| Sonnet 4.6 | 3.00 / 15.00 | 3.30 / 16.50 | 3.00 / 15.00 | 3.30 / 16.50 |

For Vertex AI gjelder prisen i EU-kolonnen for Haiku 4.5 og Sonnet 4.6 i regionen `europe-west1`. Et regneeksempel for en intern applikasjon med 50 mill. input- og 10 mill. output-token per måned:

| Modell | Globalt | EU-bundet | Ekstrakostnad per måned |
|---|---|---|---|
| Sonnet 5.5 | 50 × 2 + 10 × 10 = 200 USD | 220 USD | 20 USD |
| Opus 5.5 | 50 × 4 + 10 × 20 = 400 USD | 440 USD | 40 USD |

Regional binding koster altså 10 % mer enn det globale endepunktet. Sammenlignet med listeprisen direkte hos Anthropic med `inference_geo: "us"` koster en EU-profil hos Bedrock det samme. Det er heller tilleggskostnadene som veier tungt i valget: En AWS- eller Google Cloud-konto med organisasjonspolicyer, logging og budsjettvarsler må bygges opp og driftes, og de faste månedsprisene for Claude-abonnementer erstattes av ren forbruksbasert fakturering. Om byttet lønner seg, avhenger av volumet; et abonnement har fast pris per person, men gir ingen kontroll over behandlingsstedet.

## Lagring og trening

I tillegg til stedet teller hvor lenge data lagres:

- **Anthropic API:** Prompter og svar brukes ikke til trening uten uttrykkelig samtykke. Zero Data Retention (ZDR) er tilgjengelig på forespørsel per organisasjon og kan kombineres med `inference_geo`. Innhold som markeres av misbruksdeteksjonen, lagres i opptil to år selv med ZDR. For modellene Fable og Mythos er lagring i 30 dager obligatorisk.
- **AWS Bedrock:** Innhold sendes ikke til Anthropic. Lagring kan velges per modell mellom `none`, `default` og `aws_review`; Fable krever `aws_review` med opptil 30 dagers lagring for kontroll hos AWS.
- **Google Vertex AI:** Google bruker kun dataene til trening eller finjustering med forhåndssamtykke.

## Endringslogg

- **25. oktober 2026:** Lagt til: Haiku 5.5 (siden 7. oktober 2026) med tilgjengelighet i Zürich og i EU-multiregionen til Google Cloud samt priser; målregionene til EU-profilen er nå også dokumentert for Opus 5.5 og Sonnet 5.5, priser og regioner per 25. oktober 2026. Kildelenker oppdatert til de nye adressene i Google Cloud-dokumentasjonen.

## Kilder

1.  [Claude Platform Docs: Data residency](https://platform.claude.com/docs/en/manage-claude/data-residency): Parameter `inference_geo`, verdiene `us` og `global`, Workspace geo, støttede modeller, feltet `usage.inference_geo`.

2.  [Claude Platform Docs: Pricing](https://platform.claude.com/docs/en/about-claude/pricing): Listepriser for Anthropic API og faktor 1,1 for `inference_geo: "us"`.

3.  [Claude Platform Docs: API and data retention](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention): Zero Data Retention, unntak for markert innhold og for Fable/Mythos.

4.  [Anthropic: Privacy Policy](https://www.anthropic.com/legal/privacy): Anthropic Ireland som behandlingsansvarlig for EØS, Storbritannia og Sveits, overføring til USA på grunnlag av standard kontraktsklausuler.

5.  [Anthropic: Regional compliance](https://claude.com/regional-compliance): Oversikt over hvilken plattform som tilbyr dataresidens i hvilken region; Foundry Europa som «Coming soon».

6.  [Claude Platform Docs: Claude in Amazon Bedrock](https://platform.claude.com/docs/en/build-with-claude/claude-in-amazon-bedrock): Endepunktstyper per region, 10 % påslag for regionale endepunkter.

7.  [AWS: Modellkort Claude Sonnet 4.6](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-4-6.html): Målregioner for EU-profilen per kilderegion, inkludert Zürich.

8.  [AWS: Modellkort Claude Sonnet 5.5](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-5-5.html): Tilgjengelighet i `eu-central-2` kun som geo- og global profil, målregioner for EU-profilen fra Zürich.

9.  [AWS: Geographic cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/geographic-cross-region-inference.html): Datalagring innen geografien, lagringssted, sperring av målregioner med SCP.

10.  [AWS: Cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html): Feltet `additionalEventData.inferenceRegion` i CloudTrail, pris etter kilderegion.

11.  [AWS: Data protection in Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/data-protection.html): Modellleverandørene har ikke tilgang til prompter, svar og logger.

12.  [AWS: Amazon Bedrock Pricing](https://aws.amazon.com/bedrock/pricing/): Listepriser for Global- og Geo-profiler, identiske for Zürich, Frankfurt og North Virginia.

13.  [AWS Alps Blog: Cross-region inference for EU data processing in Switzerland](https://aws.amazon.com/blogs/alps/unlocking-ai-flexibility-in-switzerland-a-guide-to-cross-region-inference-for-eu-data-processing-and-model-access/): Veiledning fra AWS Sveits om EU-profilen fra Zürich.

14.  [Claude Code Docs: Amazon Bedrock](https://code.claude.com/docs/en/amazon-bedrock): Miljøvariabler og automatisk prefiks `eu.` i EU-regioner.

15.  [Google Cloud: Generative AI locations](https://docs.cloud.google.com/gemini-enterprise-agent-platform/resources/locations): Lokasjonstabell for Claude-modellene, ingen tilgjengelighet i `europe-west6`.

16.  [Google Cloud: Data residency](https://docs.cloud.google.com/gemini-enterprise-agent-platform/resources/data-residency): EU-multiregion kun for EU-medlemsstater, Sveits og Storbritannia utelukket.

17.  [Google Cloud: Vertex AI Pricing](https://cloud.google.com/gemini-enterprise-agent-platform/generative-ai/pricing): Listepriser globalt, EU-multiregion og `europe-west1`.

18.  [Microsoft Learn: Claude models hosting comparison](https://learn.microsoft.com/en-us/azure/foundry/foundry-models/concepts/claude-models-hosting-comparison): Hosted on Azure og Hosted on Anthropic, Data Zone kun USA.

19.  [Microsoft Learn: Anthropic as AI subprocessor in Microsoft 365 Copilot](https://learn.microsoft.com/en-us/microsoft-365/copilot/connect-to-ai-subprocessor): Unntak fra EU Data Boundary, standardinnstilling i EU/EFTA.

20.  [TCDEV Blog: Where Are Claude's Data Centers?](https://www.tcdev.de/blog/where-are-claudes-data-centers/): Tim Cadenbachs oversikt over Project Rainier, Google-TPU-avtalen og egne datasentre med Fluidstack.

21.  [Anthropic: Investering på 50 mrd. USD i amerikansk infrastruktur](https://www.anthropic.com/news/anthropic-invests-50-billion-in-american-ai-infrastructure): Annonsering av egne datasentre i Texas og New York med Fluidstack.

22.  [Claude Platform Docs: IP addresses](https://platform.claude.com/docs/en/api/ip-addresses): Inngående adresseområde `160.79.104.0/23`.

23.  [RIPEstat](https://stat.ripe.net/): Prefiksoversikt og BGP-status for `160.79.104.0/23` (AS399358, upstream AS13335).

24.  [llmlatency.dev: Anthropic](https://llmlatency.dev/provider/anthropic): Responstider ved Edge etter målelokasjon.

25.  [Fedlex: Personvernforordning (DSV), vedlegg 1](https://www.fedlex.admin.ch/eli/cc/2022/568/de): Liste over stater med tilstrekkelig personvern, inkludert EU/EØS og USA for sertifiserte selskaper.

26.  [Anthropic: Claude Haiku 5.5](https://www.anthropic.com/news/claude-haiku-5-5): Lansering 7. oktober 2026, tilgjengelighet hos AWS, Google Cloud og Microsoft, priser med trinn fra 100 000 token.

27.  [AWS: Modellkort Claude Haiku 5.5](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-haiku-5-5.html): Tilgjengelighet i `eu-central-2` som geo- og global profil samt målregioner for EU-profilen fra Zürich.
