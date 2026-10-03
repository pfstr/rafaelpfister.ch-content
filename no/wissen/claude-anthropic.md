---
title: "Claude og Claude Code: arkitektur, API og sikker agentdrift"
blatt: "claude"
description: "Claude og Claude Code for plattform-, sikkerhets- og automatiseringsadministratorer: modell- og API-grenser, tilstandsløse Messages, Content Blocks, strømming, Tool Use og MCP, lokal agentkjøretid, workspaces og nøkler, rate limits, caching, batches, observability, personvern, prompt injection og recovery."
fakten:
  - label: Systemrolle
    wert: Claude er en familie av generative språkmodeller; applikasjoner leverer kontekst og behandler probabilistisk genererte Content Blocks
    href: https://platform.claude.com/docs/en/about-claude/models/overview
  - label: Leverandør
    wert: Anthropic utvikler modellene, den direkte Claude API-en, Claude-applikasjoner og Claude Code
    href: https://www.anthropic.com/news/introducing-claude
  - label: Kjerne-API
    wert: POST /v1/messages behandler strukturerte meldinger; samtaletilstand sendes på nytt av klienten
    href: https://platform.claude.com/docs/en/api/messages/create
  - label: Transport
    wert: HTTPS/JSON; strømming bruker Server-Sent Events uten å endre den semantiske Message-kontrakten
    href: https://platform.claude.com/docs/en/build-with-claude/streaming
  - label: Utdata
    wert: en Message inneholder typede Content Blocks, bruksverdier og stop_reason i stedet for garantert fritekst
    href: https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons
  - label: Modellbinding
    wert: Modell-ID-er er festede kontrakter; egenskaper og grenser hentes via Models API og dokumentasjon
    href: https://platform.claude.com/docs/en/about-claude/models/model-ids-and-versions
  - label: Verktøykontrakt
    wert: Claude genererer tool_use med JSON-argumenter; klientkode utfører og sender tool_result tilbake
    href: https://platform.claude.com/docs/en/agents-and-tools/tool-use/how-tool-use-works
  - label: MCP
    wert: åpen klient-server-protokoll for verktøy, ressurser og kontekst; hver server utvider data- og handlingsrettigheter
    href: https://docs.anthropic.com/en/docs/mcp
  - label: Claude Code
    wert: lokal agent for repository, filer, shell og verktøy; API-inferens forblir en ekstern tjeneste
    href: https://docs.anthropic.com/en/docs/claude-code/getting-started
  - label: Leietakermodell
    wert: Organisasjon → Workspace → medlemmer, nøkler, grenser og workspace-bundne ressurser
    href: https://platform.claude.com/docs/en/manage-claude/workspaces
  - label: Kapasitet
    wert: Spend Limits samt Request-, Input- og Output-Token-Limits; 429 og retry-after styrer backoff
    href: https://platform.claude.com/docs/en/api/rate-limits
  - label: Datalagring
    wert: Oppbevaring avhenger av produkt, workspace, funksjon, kontrakt og sikkerhetsklassifisering og kontrolleres før bruk
    href: https://privacy.claude.com/en/articles/7996866-how-long-do-you-store-my-organization-s-data
werbung:
  - newsletter
ctaThemen:
  - claude
translationSourceHash: a5189670b409d16f452c042199e3237347d4a9a1483be7c38ebbc07a8d640c5f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-03T10:04:53.565Z
translationReview: automatic
---

# Claude og Claude Code: arkitektur, API og sikker agentdrift

Claude betegner en familie av store generative språkmodeller fra Anthropic samt flere produkter som bruker disse modellene. Det viktigste administrative skillet er: **modell, API, applikasjon og agentkjøretid er ikke det samme systemet**. Modellen genererer en probabilistisk sekvens av utdatatokener fra inndatatokener. Messages API pakker denne prosessen som en HTTPS-kontrakt. Claude-applikasjoner tilfører kontoer, samtalelagring og brukergrensesnitt. Claude Code legger til en lokal kjøretid med tilgang til filer, shell og verktøy på et endpoint.

Den første Claude-versjonen ble presentert som chat- og API-tjeneste i 2023 ([Introducing Claude](https://www.anthropic.com/news/introducing-claude)). Anthropic beskriver Claude som en assistent innrettet mot hjelpsom, ærlig og harmløs atferd. Denne innretningen, et stort kontekstvindu eller overbevisende formulert tekst er imidlertid ikke bevis på faktisk riktighet, autorisasjon eller sikker utførelse. Et produksjonssystem må behandle modellsvar som ikke-klarert input som kontrolleres mot skjema og policy.

Forklaringen begynner med en forespørsel til modellen og følger svaret via Messages API, Content Blocks og verktøykall. Først når dette forløpet er klart, utdypes Claude Code, MCP, rettigheter, prompt injection, drift og recovery.

Claude er ikke et autonomt handlende operativsystem, men en inferenstjeneste. Den omkringliggende applikasjonen leverer kontekst, kontrollerer svar, utfører godkjente verktøy og har ansvaret for identiteter, datatilgang og revisjon.

## Arkitekturtilnærming: inferenstjeneste pluss kontrollerende applikasjon

Claude kjører normalt som en ekstern inferenstjeneste. En klient sender systeminstruksjoner, samtalemeldinger, Content Blocks, verktøydefinisjoner og genereringsparametere. Tjenesten autentiserer og begrenser forespørselen, tokeniserer konteksten, utfører inferens og leverer en Message eller en hendelsesstrøm. Applikasjonen avgjør deretter om tekst vises, JSON valideres, et verktøy utføres, et resultat sendes tilbake eller kjøringen avbrytes.

Direkte API er ikke den eneste leveringsveien. Claude-modeller tilbys også via skyplattformer; identitet, endpoint, regioner, kvoter, logging og kontraktsvilkår er forskjellige der. Applikasjonsarkitekturen holder leverandøradaptere og faglig arbeidsflyt atskilt. Et modellbytte er ikke en enkel DNS-omkobling når verktøytyper, kontekstgrenser, Stop Reasons eller leverandørfunksjoner avviker.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1116" src="/images/kb-interaktiv-claude.svg?v=20260813" title="Interaktive Infografik: Claude von Benutzer, Anwendung und Workspace über Messages API, Tokenisierung und Modellinferenz bis zu Content Blocks, Tool Use, MCP, Claude Code, Rate Limits, Logging, Datenschutz und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-claude.svg?v=20260813">Åpne interaktiv grafikk direkte</a>.
</iframe>

## Modellag og probabilistisk semantikk

Claude-modeller behandler tekst, kode og, avhengig av modell, flere modaliteter. [Modelloversikten](https://platform.claude.com/docs/en/about-claude/models/overview) dokumenterer modellfamilier, kontekstvinduer og egenskaper. Slike verdier bør ikke stå som en permanent frosset faktaopplysning i en statisk artikkel. Før utrulling henter en klient dokumenterte egenskaper via Models API, eller ved byggetid, og lagrer den faktisk brukte modellbetegnelsen med hvert resultat.

En språkmodell er ingen database og ingen regelmotor. Like inndata kan gi ulike utdata avhengig av sampling, systemkontekst, modellrevisjon og verktøyresultater. Selv ved lav temperatur gjelder følgende egenskaper:

- Fakta kan mangle, være foreldet eller oppdiktet.
- Instruksjoner kan vektes tvetydig.
- Lange kontekster kan skjule relevante detaljer.
- Struktur-lignende tekst er ikke et gyldig objekt uten skjemaavslutning.
- En plausibel forklaring beviser ikke at et verktøy faktisk ble utført.
- En modell kjenner ingen rettighet utover informasjonen og verktøygrensene som applikasjonen håndhever.

Teknisk godkjenning bruker derfor evalueringscaser, forventede skjemaer, deterministiske etterkontroller og faglige kilder. En vellykket modellbenchmark erstatter ikke applikasjonsspesifikk måling av nøyaktighet, latenstid, kostnader og skade ved feil.

## Trening, Constitution og ansvarsområder

Anthropic utviklet **Constitutional AI** som et supplement til overvåket læring og reinforcement learning. Modellen kritiserer og reviderer svar ved hjelp av eksplisitte prinsipper; et andre treningstrinn bruker AI-generert tilbakemelding. Anthropic forklarer denne tilnærmingen og dens grenser i [Claude’s Constitution](https://www.anthropic.com/research/claudes-constitution). [Claude-3-modellkortet](https://assets.anthropic.com/m/61e7d27f8c8f5919/original/Claude-3-Model-Card.pdf) dokumenterer trening, evaluering og sikkerhetstiltak for en konkret modellgenerasjon.

For administratorer er denne historien relevant fordi den forklarer atferd, ikke fordi den erstatter kjøretidskontroll. Safety Training kan redusere skadelige svar, men kan verken håndheve leietakerisolasjon eller verktøyautorisasjon. Refusals er normale mulige modellutdata. Applikasjonen må behandle dem som `stop_reason` eller Content Type og må ikke utlede at en handling er sikker fordi en avvisning mangler.

Først gjennom Messages API blir den probabilistiske modellen en administrerbar tjeneste. Hver forespørsel overfører nødvendig samtalekontekst på nytt og mottar strukturerte Content Blocks som svar.

## Messages API: tilstandsløs samtalekontrakt

`POST /v1/messages` mottar en liste med `user`- og `assistant`-meldinger og genererer neste Assistant-turn. En systemprompt ligger i det separate toppnivåfeltet `system`; en `system`-rolle i `messages` finnes ikke. Content kan sendes som streng eller som en liste over typede blokker ([Create a Message](https://platform.claude.com/docs/en/api/messages/create)).

API-et er **tilstandsløst** uten ekstra trådlager. For en flerturssamtale sender klienten nødvendig historikk på nytt. Dette har direkte konsekvenser:

1. Applikasjonen eier Conversation ID, rekkefølge og oppbevaring.
2. Forkorting, oppsummering eller utelatelse endrer modellkonteksten.
3. Systemprompt, verktøydefinisjoner og historikk teller med i inputbudsjettet.
4. En provider-request-ID erstatter ikke en faglig job-ID.
5. Et retry av samme forespørsel kan generere nytt utdata og ekstrakostnader.

Den faglige operasjonen får derfor en egen korrelasjons- og idempotens-ID. Før et retry kontrollerer orkestratoren om et resultat eller en irreversibel verktøyeffekt allerede foreligger.

## Content Blocks og Stop Reasons

Et vellykket svar er ikke nødvendigvis én enkelt tekst. `content` er en ordnet liste over typede blokker, for eksempel tekst, Tool Use, Thinking- eller serververktøyblokker. `usage` rapporterer tokenklasser. `stop_reason` beskriver hvorfor genereringen avsluttet. Den offisielle [Stop Reason-dokumentasjonen](https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons) skiller blant annet mellom `end_turn`, `max_tokens`, `stop_sequence`, `tool_use`, `pause_turn`, `refusal` og `model_context_window_exceeded`.

Bare `end_turn` betyr en naturlig avsluttet tur; det beviser ikke faglig fullstendighet. `max_tokens` og `model_context_window_exceeded` markerer potensielt avkortede data. `tool_use` er en oppfordring til orkestratoren, ikke en utført effekt. `pause_turn` krever en protokollkorrekt fortsettelse. Nye enumverdier kan tilkomme, og parseren må derfor synlig avvise eller trygt ignorere ukjente typer i stedet for å falle tilbake til standardsuksess.

## API-versjon og modellversjon

Hver direkte API-forespørsel har en `anthropic-version`-header. Denne API-versjonen stabiliserer felt og strømme-semantikk, men kan ikke hindre nye valgfrie inndata, utdatafelt eller enumvarianter ([API versioning](https://platform.claude.com/docs/en/api/versioning)). Klientparseren bygges derfor fremoverkompatibelt og logger ukjente blokker.

Adskilt fra dette er **modell-ID-en**. Anthropic garanterer en konstant modellversjon for en festet ID gjennom dens levetid; bekvemmelighetsaliaser kan ha andre regler ([Model IDs and versioning](https://platform.claude.com/docs/en/about-claude/models/model-ids-and-versions)). For revisjon og reproduksjon lagrer et system:

- API-versjon og betaheader,
- modell-ID i stedet for bare visningsnavn,
- systemprompt-/policyversjon,
- toolset- og JSON-schemaversjon,
- retrieval- og dokumentversjoner,
- genereringsparametere,
- request-, job- og brukerkorrelasjon,
- Stop Reason, Usage og valideringsresultat.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Claude-Modellinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$headers = @{
  'x-api-key' = $env:ANTHROPIC_API_KEY
  'anthropic-version' = '2023-06-01'
}
Invoke-RestMethod -Headers $headers -Uri 'https://api.anthropic.com/v1/models' |
  ConvertTo-Json -Depth 12</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">curl --fail --silent --show-error https://api.anthropic.com/v1/models \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H 'anthropic-version: 2023-06-01' | jq .</code></pre>
  </div>
</div>

[`Invoke-RestMethod`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod) og [`ConvertTo-Json`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertto-json) behandler Windows; [`curl`](https://curl.se/docs/manpage.html) og [`jq`](https://jqlang.org/manual/) behandler Unix. Nøkkelen kommer fra en secrets-lagring og skrives verken til kommandolinjen eller ut.

## Tokenisering, kontekst og kostnader

Kontekstvinduer måles i tokens, ikke tegn eller filer. Verktøyskjemaer, systemprompt, meldinger, bilder, dokumenter og verktøyresultater bidrar til input; generert tekst og eventuelt Thinking bidrar til output. [Token Counting API](https://platform.claude.com/docs/en/build-with-claude/token-counting) mottar samme inputstruktur som Messages og leverer et forhåndsanslag.

En stor kontekst er ikke et arkiv. Jo mer irrelevant data klienten sender med, desto høyere blir kostnadene, latenstiden og risikoen for motstridende instruksjoner. Produksjonssystemer bygger derfor en **Context Assembly Pipeline**:

1. Kontroller bruker- og leietakerrettigheter før retrieval.
2. Hent dokumenter basert på stabile ID-er og versjoner.
3. Merk untrusted content som data, ikke som systeminstruksjoner.
4. Begrens datamengde, filtyper og tokenbudsjett.
5. Send med kildemetadata og hasher.
6. Valider svaret mot de samme kildene og skjemaene.

Search-result-content-blocks kan overføre RAG-kilder med tittel og opphav, slik at Claude genererer sitater ([Search results](https://platform.claude.com/docs/en/build-with-claude/search-results)). Disse sitatene er bare like pålitelige som retrieval, dokumentidentitet og overførte metadata.

## Prompt Caching

Prompt Caching lagrer gjenbrukbare prefikser og reduserer behandlingskostnader og latenstid. Cachehierarkiet følger `tools` → `system` → `messages`; endringer i en tidligere del invaliderer den og etterfølgende nivåer. Den offisielle [Prompt Caching-dokumentasjonen](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) beskriver automatiske og eksplisitte breakpoints samt korte TTL-er.

Et cache hit beviser bare at et identisk prefiks ble gjenbrukt. Det garanterer verken oppdaterte kildedata eller identisk utdata. Cachemetrikker registreres separat for Creation og Read. Workspaces isolerer Prompt Caches i den direkte Claude API-en ([Workspaces](https://platform.claude.com/docs/en/manage-claude/workspaces)). Hemmeligheter hører ikke hjemme i prompter, selv ved kort TTL; cache og oppbevaring er ulike mekanismer.

## Strømming med Server-Sent Events

Med `stream: true` leverer API-et inkrementelle Server-Sent Events. Content Blocks startes, utvides via deltaer og avsluttes; avsluttende Message- og Usage-informasjon er fortsatt nødvendig for korrekt behandling. [Streaming-dokumentasjonen](https://platform.claude.com/docs/en/build-with-claude/streaming) beskriver hendelsestyper og SDK-akkumulatorer.

En synlig tekstdelta er ingen commit. Ved forbindelsesavbrudd kan brukeren allerede ha sett delvis tekst, mens klienten ikke har en fullstendig Message. JSON- eller verktøyargumenter må først brukes etter fullstendig blokk og skjemavalidering. Gatewayer og proxier må behandle lange HTTP-forbindelser, buffering, timeouts og backpressure korrekt.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Claude-Messages und Streaming">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$body = @{
  model = $env:CLAUDE_MODEL
  max_tokens = 256
  messages = @(@{ role = 'user'; content = 'Svar med én setning.' })
} | ConvertTo-Json -Depth 8
curl.exe --no-buffer --fail-with-body https://api.anthropic.com/v1/messages `
  -H "x-api-key: $env:ANTHROPIC_API_KEY" `
  -H 'anthropic-version: 2023-06-01' -H 'content-type: application/json' `
  --data-raw $body</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">jq -n --arg model "$CLAUDE_MODEL" '{
  model: $model, max_tokens: 256,
  messages: [{role: "user", content: "Svar med én setning."}]
}' | curl --no-buffer --fail-with-body https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H 'anthropic-version: 2023-06-01' -H 'content-type: application/json' \
  --data-binary @-</code></pre>
  </div>
</div>

Eksempelforespørselen bruker bevisst en modell-ID lest fra konfigurasjon. For SSE settes i tillegg `stream: true`, og den navngitte hendelsesstrømmen parses protokollkorrekt; ren linjelesing er ikke nok for produksjonskode.

Tekstutdata endrer ennå ikke noe system. Først Tool Use kobler et modellforslag til en handling – og nettopp her må applikasjon, autorisasjon og menneskelig godkjenning gripe inn.

## Tool Use: modellen ber om det, applikasjonen handler

Tool Use er en kontrakt mellom modell og orkestrator. Applikasjonen beskriver et verktøy med navn, formål og JSON Schema. Claude kan deretter generere en `tool_use`-blokk med argumenter. Klienten validerer navn og input, autoriserer det konkrete kallet, utfører det og sender en `tool_result`-blokk med samme Tool Use-ID tilbake. Først en ytterligere modellturn kan danne et svar av dette ([How tool use works](https://platform.claude.com/docs/en/agents-and-tools/tool-use/how-tool-use-works)).

[Verktøyoversikten](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview) skiller mellom:

- **Client Tools:** Den egne applikasjonen utfører.
- **Anthropic Schema Tools:** Anthropic definerer skjemaet, men klienten utfører likevel.
- **Server Tools:** Anthropic utfører på sin infrastruktur og leverer resultatblokker.
- **MCP Connector:** API-et kobler seg til en ekstern MCP-server.

Dette skillet bestemmer nettverksvei, hemmeligheter, logging, personvern og feilområde. `strict: true` håndhever skjemaoverensstemmelse, ikke faglig riktighet eller autorisasjon. Et syntaktisk gyldig kall `delete_user(id)` kan fortsatt slette feil bruker.

### Produksjonsklar verktøysløyfe

En sikker orkestrator utfører følgende trinn for hver Tool Use:

1. Kontroller verktøynavn mot allowlist og arbeidsflytfase.
2. Valider JSON Schema strengt og avvis ukjente felt.
3. Kontroller bruker-, leietaker- og objektrettigheter på nytt serverside.
4. Normaliser inndata; begrens stier, URL-er, ID-er og størrelser.
5. Klassifiser Read, Write, External Message og Destructive Action.
6. Krev faglig godkjenning eller fireøyneprinsipp ved høy risiko.
7. Utfør med kortlivet, minimal identitet i isolert kjøretid.
8. Sett timeout, outputgrense og idempotensnøkkel.
9. Rens resultatet for hemmeligheter og untrusted instructions.
10. Revider Tool Use, beslutning, effekt og resultat uforanderlig.

Sløyfen begrenser turns, parallelle verktøy, akkumulerte kostnader og gjentatte feil. Et verktøyresultat er igjen untrusted content. En databaselinje eller nettside kan inneholde prompt injection og må ikke overskrive orkestratorens policy.

## MCP som protokollgrense

Model Context Protocol standardiserer forbindelsen mellom AI-applikasjoner og verktøy, ressurser og prompter. Anthropic beskriver MCP som en åpen klient-server-protokoll ([MCP-oversikt](https://docs.anthropic.com/en/docs/mcp)); den normative [MCP-spesifikasjonen](https://modelcontextprotocol.io/specification/2025-06-18) definerer meldinger og egenskaper.

MCP gjør integrasjoner utskiftbare, men ikke automatisk pålitelige. En server kan lese data, utløse handlinger, levere svært store resultater eller returnere innhold fra en tredjepart. Inkludering i en konfigurasjonsfil er derfor en programvare- og rettighetsbeslutning. Operatør, transport, endpoint, autentisering, verktøy, ressurser, scopes, dataklasser, versjon, timeout, outputgrense og tilbakekalling inventariseres.

[Messages API MCP Connector](https://platform.claude.com/docs/en/agents-and-tools/mcp-connector) er et serververktøy på Anthropic-infrastruktur. En stdio-server som startes lokalt i Claude Code, kjører derimot på endpointet. Samme protokoll, andre dataveier og feilområder.

## Claude Code: lokal agentkjøretid

Claude Code er en agent for terminal og repository. Den samler prosjektkontekst, sender den til et modellendpoint, tolker modellforslag og bruker lokale verktøy for å lese, endre og utføre. Den offisielle [installasjonen](https://docs.anthropic.com/en/docs/claude-code/getting-started) dokumenterer Windows via WSL eller Git Bash samt macOS og Linux. [CLI-referansen](https://docs.anthropic.com/en/docs/claude-code/cli-usage) beskriver interaktiv drift, Print Mode, JSON-utdata, sesjonsfortsettelse, modellvalg og verktøyregler.

Den sentrale tillitsgrensen er den lokale prosessen. Claude Code kan bare handle med operativsystemrettighetene og tilgjengelige credentials til den startede brukeren, men disse rettighetene kan være svært omfattende: repository, SSH-agent, sky-CLI, pakkeregistre, Kubernetes, nettlesercookies, miljøvariabler eller production-tunneler. «Agenten spurte» er ingen isolasjon.

[Claude Code-sikkerhetsdokumentasjonen](https://docs.anthropic.com/en/docs/claude-code/security) beskriver skrivebeskyttede standardinnstillinger, Permission Prompts, prosjektgrenser og beskyttelse mot Prompt Injection. Den presiserer samtidig at ingen systemer er fullstendig immune. Ikke-interaktiv drift krever strengere policyer fordi repositoryinnhold, issue-tekst, byggelogger og verktøyutdata kan inneholde angriperinstruksjoner.

### Kjøretids- og rettighetsprofil

Et administrert endpoint eller en CI-runner bruker:

- en dedikert bruker- eller workloadkonto,
- en prosjektspesifikk arbeidskatalog,
- minimale filsystem- og nettverksrettigheter,
- kortlivede credentials begrenset til mål og handling,
- tillatte og forbudte verktøy fra sentral policy,
- Container/VM/Sandbox for untrusted builds,
- Secret Scanning før kontekstinntak,
- begrensede turns, tid, output og kostnader,
- Git-diff, tester og godkjenning før commit/deploy,
- sesjons-, verktøy- og leverandørkorrelasjon i revisjonsloggen.

Bryteren `--dangerously-skip-permissions` er ingen automatiseringsstrategi. Den fjerner et beskyttelseslag og er bare forsvarlig i en forhåndsoppsatt sandbox med egen policy.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Claude-Code-Inventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-Command claude | Format-List Source,Version
claude doctor
Get-FileHash (Get-Command claude).Source -Algorithm SHA256
Get-ChildItem .\.claude -Force -Recurse | Select-Object FullName,Length</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">command -v claude
claude doctor
file "$(command -v claude)"
sha256sum "$(command -v claude)"
find ./.claude -maxdepth 3 -type f -printf '%p %s\n'</code></pre>
  </div>
</div>

[`Get-Command`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/get-command), [`Get-FileHash`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash) og [`Get-ChildItem`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-childitem) inventariserer Windows. POSIX [`command`](https://pubs.opengroup.org/onlinepubs/9799919799/utilities/command.html), [`file`](https://man7.org/linux/man-pages/man1/file.1.html), [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) og [`find`](https://man7.org/linux/man-pages/man1/find.1.html) håndterer Unix. En hash vurderes bare mot en pålitelig release- eller pakkekilde.

## Ikke-interaktiv drift og CI-drift

Print Mode gjør Claude Code skriptbar. `--output-format json` eller `stream-json` leverer maskinlesbare resultater; `--max-turns` begrenser agentkjøringen. Inndata og exitkode forblir en del av jobbloggen. Fritekstutdata evalueres ikke som shellscript.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für begrenzten Claude-Code-Print-Mode">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-Content .\review-request.txt -Raw |
  claude -p --max-turns 3 --output-format json `
    --allowedTools 'Read' --disallowedTools 'Bash' 'Edit' |
  Tee-Object -FilePath .\claude-review.json |
  ConvertFrom-Json</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">cat ./review-request.txt |
  claude -p --max-turns 3 --output-format json \
    --allowedTools Read --disallowedTools Bash Edit |
  tee ./claude-review.json | jq .</code></pre>
  </div>
</div>

[`Get-Content`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-content), [`Tee-Object`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/tee-object) og [`ConvertFrom-Json`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json) behandler Windows. [`cat`](https://www.gnu.org/software/coreutils/manual/html_node/cat-invocation.html) og [`tee`](https://www.gnu.org/software/coreutils/manual/html_node/tee-invocation.html) håndterer Unix. En CI-jobb får i tillegg faste regler for arbeidskatalog, nettverk, hemmeligheter og branch.

## Settings, Hooks og Policies

Claude Code kombinerer bruker-, prosjekt- og administrerte innstillinger. Prosjektfiler kan ligge i repositoryet og inngår dermed i Code Review. Hooks utfører kommandoer ved definerte livssykluspunkter; de er kjørbar kode, ikke harmløs promptkonfigurasjon. MCP-servere utvider systemene som kan nås. En enterprisepolicy må derfor kontrollere Settings, Hooks, Plugins, Skills, MCP og tillatte shellmønstre samlet.

CLI-en kan tillate rettigheter én gang eller varig. Brede wildcards fremskynder arbeidet, men øker blast radius. En god regel tillater ikke «Bash», men en begrenset lesefunksjon eller et forhåndskoblet, typet verktøy. For skriveoperasjoner kontrolleres målsti, branch og diff. For deployer eller eksterne meldinger beholdes en separat godkjenning utenfor modellen.

## Identitet, organisasjon og Workspaces

Claude Platform knytter bruk til en organisasjon og Workspaces. Workspace-nøkler er begrenset til ressursene og bruken i dette workspace-et; medlemmer har Workspace-roller. Separate Workspaces for utvikling, test og produksjon skiller nøkler, grenser, batches, filer og Prompt Caches ([Workspaces](https://platform.claude.com/docs/en/manage-claude/workspaces)).

Admin API og Inference API bruker ulike nøkkeltyper. [Admin API-dokumentasjonen](https://platform.claude.com/docs/en/manage-claude/admin-api) beskriver medlemmer, invitasjoner, Workspaces og API Keys. En adminnøkkel skal aldri ligge i en applikasjon som bare må sende Messages. Offboarding tilbakekaller brukertilgang, personlige Claude Code-nøkler og eventuelt separat opprettede tjenestenøkler.

Claude Code kan autentiseres via Console/OAuth, Claude-abonnementer eller Enterpriseprovider. Avgjørende er den **faktiske fakturerings- og dataruten** for økten. Personlige kontoer i et virksomhetsrepository omgår ellers workspacekontroller, oppbevaring, kostnadssteder eller revisjon.

## Rate Limits, forbruk og Backpressure

Anthropic skiller mellom Spend Limits og Rate Limits. Messages begrenses etter Requests per Minute, Input Tokens per Minute og Output Tokens per Minute. Grensene bruker Token Buckets; korte bursts kan derfor utløse 429 selv ved tilsynelatende passende minuttgjennomsnitt. `retry-after` og responseheadere angir rammen for backoff ([Rate limits](https://platform.claude.com/docs/en/api/rate-limits)).

En gateway implementerer:

- kø og prioritet per leietaker/arbeidsflyt,
- tokenopptelling før store jobber aksepteres,
- eksponentiell backoff med jitter og `retry-after`,
- globale og workspace-relaterte parallellitetsgrenser,
- Circuit Breaker ved 5xx og nettverksfeil,
- budsjettvarsling og hard kostnadsgrense,
- separate metrikker for Cache Read, Cache Write, Input og Output,
- kontrollert fallback med dokumentert kvalitetsendring.

[API-feildokumentasjonen](https://platform.claude.com/docs/en/api/errors) skiller mellom 400, 401, 402, 403, 404, 413, 429, 500 og 529 og leverer `request_id`. SDK-er gjentar bestemte transiente feil automatisk. Disse retryene regnes med i kapasitets- og kostnadsplanleggingen.

## Batches og asynkron behandling

Message Batches API behandler mange uavhengige forespørsler asynkront. Batches er bundet til workspace; enkeltresultater kan lykkes eller feile. Resultattilgjengelighet og oppbevaring på serversiden er forskjellig fra den synkrone veien. [Batch-dokumentasjonen](https://platform.claude.com/docs/en/build-with-claude/batch-processing) angir grenser for størrelse, kjøretid og henting.

En batchjobb har en `custom_id` per fagobjekt, et inputmanifest, en hentecursor og resultatkontroll. «Batch completed» betyr bare at alle oppføringer har en sluttstatus. Importen behandler hvert resultat idempotent, oppdager manglende ID-er og arkiverer modell-, prompt- og skjemaversjon. For personopplysninger eller regulerte data vurderes det om batchveien i det hele tatt er tillatt under kontrakten og datalagringsmodellen.

Etter funksjon og skala følger dataspørsmålet: Hvilket innhold forlater eget system, hvor lenge beholdes det, og hvilke ytterligere operatører er involvert på skyplattformer?

## Datalagring, personvern og tredjepartsplattformer

Datalagring avhenger av grensesnitt og kontrakt. Anthropic beskriver for kommersiell API-bruk standard sletting av input og output innen 30 dager, men nevner unntak for funksjoner med lengre lagring, avvikende avtaler, Safety Enforcement og juridiske forpliktelser ([Oppbevaring av kommersielle data](https://privacy.claude.com/en/articles/7996866-how-long-do-you-store-my-organization-s-data)). Claude-applikasjoner lagrer samtaler for produktfunksjoner etter egne regler.

Zero Data Retention er ingen global bryter for ethvert produkt. [ZDR-dokumentasjonen](https://privacy.claude.com/en/articles/8956058-i-have-a-zero-data-retention-agreement-with-anthropic-what-products-does-it-apply-to) beskriver kvalifiserte API-er, organisasjoner og unntak. Ved tilgang via skyleverandør gjelder i tillegg dens datarute, regioner, nøkkelhåndtering og kontrakt. MCP, Web Search eller eksterne verktøy kan overføre data til flere behandlingsansvarlige.

Før produksjonsgodkjenning opprettes en dataflytmatrise:

| Sti | Data | Ansvarlig kontroll |
|---|---|---|
| Klient → modellendpoint | Prompt, filer, bilder, verktøyskjema | Klassifisering, minimering, kontrakt, region |
| Modellendpoint → klient | Content Blocks, Usage, Request ID | Validering, redaction, logging |
| Klient → Tool/MCP | Verktøyargumenter, brukerkontekst | Autorisasjon, scope, DPA, audit |
| Tool → modell | Resultat og untrusted content | Filter, outputgrense, injeksjonsbeskyttelse |
| Claude Code lokalt | Repository, shell, credentials | Endpointpolicy, sandbox, secrets |
| Logs/Tracing | Prompter, resultater, metadata | Redaction, tilgang, oppbevaring |

## Prompt Injection og untrusted content

Prompt Injection oppstår når data forsøker å bli instruksjoner. Angrepsflater er nettsider, e-poster, dokumenter, issues, kildekodekommentarer, MCP-ressurser, verktøyresultater og terminalutdata. Angrepet trenger ikke «overbevise» modellen dersom applikasjonen uansett gir en ukontrollert Tool Use vidtrekkende rettigheter.

Forsvaret er flerlaget:

1. Håndhev systempolicy og rettigheter utenfor modellteksten.
2. Filtrer retrieval etter bruker- og leietakerrettigheter.
3. Merk datablokker med opprinnelse, Trust Level og formål.
4. Autoriser verktøy minimalt, typet og objektspesifikt.
5. Godkjenn skrive-, sende-, betalings- og slettehandlinger separat.
6. Sandboks nettverk og filsystem for utførelsen.
7. Ikke legg hemmeligheter i kontekst, verktøyresultat eller synlige logger.
8. Valider output og verktøyargumenter mot skjema og fagregler.
9. Begrens agentsløyfer etter tid, turns, kostnader og effekter.
10. Utfør adversariale tester med indirekte injection i reelle dataveier.

En Human-in-the-Loop er bare effektiv hvis personen ser mål, effekt og relevante data. En generisk «Tillate?»-melding fører til Approval Fatigue. Høyrisiko-handlinger viser normaliserte parametere og kontrolleres av et uavhengig Policy Enforcement Point.

## Nettverksvei og proxydrift

Claude API og Claude Code trenger HTTPS-tilgang. En virksomhetsproxy kan håndtere autentisering, TLS Inspection, egresskontroll og logging, men blir dermed en del av konfidensialitets- og tilgjengelighetskjeden. [Proxydokumentasjonen for Claude Code](https://docs.anthropic.com/en/docs/claude-code/corporate-proxy) angir støttede proxyvariabler, CA-bundles og nødvendige mål.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Claude-DNS-, TCP- und TLS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName api.anthropic.com
Test-NetConnection api.anthropic.com -Port 443
curl.exe -sS -D - -o NUL https://api.anthropic.com/v1/models
Get-NetTCPConnection -State Established | Where-Object RemotePort -eq 443</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig +short api.anthropic.com A api.anthropic.com AAAA
curl -sS -D - -o /dev/null https://api.anthropic.com/v1/models
ss -ntp state established '( dport = :443 )'
openssl s_client -connect api.anthropic.com:443 -servername api.anthropic.com -brief</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname), [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) og [`Get-NetTCPConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection) kontrollerer Windows. [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility), [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) og [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) kontrollerer Unix. En 401 uten nøkkel bekrefter at DNS, TCP, TLS og HTTP når den forventede API-veien; det er ingen vellykket inferenstest.

Sikker agentdrift må ikke bare telle modellkall, men gjøre hele veien fra input via verktøybeslutning til ekstern effekt sporbar.

## Observability og audit

En produktiv forespørsel genererer faglige, tekniske og kostnadsmetrikker. Minst følgende registreres:

- tid, workspace, applikasjon, arbeidsflyt og pseudonym brukerkorrelasjon,
- modell-ID, API-versjon, prompt-/toolset-/skemaversjon,
- latenstid til header, første token og fullstendig Message,
- input-, cache-creation-, cache-read- og outputtokens,
- Stop Reason, Content Block-typer og valideringsstatus,
- feilklasse, antall retryer, `request_id` og `retry-after`,
- verktøynavn, autorisasjonsbeslutning, varighet, effekt og resultatklasse,
- estimerte og fakturerte kostnader,
- redaction- og oppbevaringsklasse.

Prompter og modellsvar logges ikke refleksmessig i sin helhet. De kan inneholde personopplysninger, hemmeligheter, kildekode eller angriperpayloads. Auditmetadata og debuginnhold får separate lagre, roller og slettefrister. [Usage Report API](https://platform.claude.com/docs/en/api/admin/usage_report) leverer aggregerbare API- og Claude Code-bruksdata; lokale verktøyeffekter må likevel revideres i eget system.

## Feilhåndtering og gjenopptakelse

En modellforespørsel er ikke automatisk idempotent. Ved timeout kan leverandøren ha fullført genereringen selv om klienten ikke mottok svar. Ved Tool Use kan den eksterne effekten allerede ha inntruffet. Orkestratoren lagrer derfor tilstandsovergangene:

`accepted → context_built → request_sent → response_received → validated → tool_authorized → tool_executed → result_returned → completed`

For hver overgang finnes korrelasjons-ID, tid, versjoner og checkpoint. Et retry før `tool_executed` kan gjenta modellturnen; et retry etterpå spør først etter den faglige effekten. Eksterne verktøy støtter Idempotency Keys eller et Read-after-Write-bevis. Ved ukjent tilstand eskaleres saken, den utføres ikke blindt på nytt.

Fallbackmodeller endrer kvalitet, kostnader, Context Limit og verktøyatferd. De er en testet, egen vei. Applikasjonen skjuler ikke en fallback når den endrer utsagnskraft eller compliance.

## Backup og recovery

Selve modellen sikres ikke av kunden som ekstern tjeneste. Egne kontroller og lagrede data og konfigurasjoner må kunne gjenopprettes:

- systemprompter, policyer og evalueringssett,
- verktøyskjemaer, implementasjoner og autorisasjonsregler,
- MCP-inventar og godkjente serverversjoner,
- workspace- og nøkkelprovisjonering som kode eller runbook,
- retrievalkilder, embeddings/indekser og dokumentversjoner,
- conversation-/jobbstatus og idempotensdata,
- redaction-, audit- og kostnadsmetadata,
- Claude Code-prosjektinnstillinger, Hooks og kontrollerte Skills,
- leverandøradaptere og testede modellmigreringer.

API Keys gjenopprettes ikke fra backuper, men tilbakekalles og provisjoneres på nytt. En disaster-recovery-test setter opp en ny kjøretid, tilordner minimale identiteter, sender en kjent testcase, validerer skjema og kilder, utfører et harmløst testverktøy og korrelerer provider-, gateway- og verktøylogger. Grunnleggende informasjon finnes under [Backup og Disaster Recovery](/kb/backup-dr), [API-er](/kb/apis), [Herding](/kb/haertung) og [Feilsøking](/kb/troubleshooting).

## Teknisk historie

Anthropic ble grunnlagt i 2021 som et forsknings- og produktselskap for sikre AI-systemer. Claude ble presentert i mars 2023 som chat- og API-assistent etter en lukket partnerfase ([Introducing Claude](https://www.anthropic.com/news/introducing-claude)). Claude 2 utvidet kontekst og offentlig tilgjengelighet i juli 2023 ([Claude 2](https://www.anthropic.com/news/claude-2)).

I 2024 etablerte Claude-3-familien betegnelsene Haiku, Sonnet og Opus som graderinger av hastighet, kostnad og ytelse ([Claude 3](https://www.anthropic.com/news/claude-3-family)). Disse produktnavnene er ingen stabil egenskapskontroll; applikasjoner binder festede ID-er og spør separat etter grenser.

I februar 2025 kom Claude Code som Research Preview sammen med en hybrid-resonneringsmodell ([Claude 3.7 og Claude Code](https://www.anthropic.com/news/claude-3-7-sonnet)). I mai 2025 ble Claude Code generelt tilgjengelig; samtidig utvidet Anthropic API-et med agentorienterte komponenter som Code Execution, MCP Connector og lengre Prompt Caching ([Claude 4](https://www.anthropic.com/news/claude-4)). Dermed flyttet driftsspørsmålet seg fra «Hvordan bruker jeg tekst?» til «Hvilke data og effekter kan et probabilistisk planleggingslag nå?»

Plattformen utviklet deretter sterkere modellversjonering, Workspaces, Usage-/Cost-API-er, verktøyskjemaer, caching, batches og agent-SDK-er. Det varige arkitekturpunktet er: Modellen foreslår; API-er og agentrammer transporterer; den kontrollerende applikasjonen autoriserer, utfører, validerer og reviderer.

## Admin-sjekkliste

De mange grensesnittene til Claude blir håndterbare når modell, API, verktøy og lokal prosess registreres hver for seg. Sjekklisten oppsummerer kontrollene som bør være dokumentert før produktiv bruk.

- **Lag:** Inventariser modell, API, Claude-applikasjon, gateway, verktøy og Claude Code-endpoint separat.
- **Kontrakter:** Håndter API-versjon, festet modell-ID, Content Blocks, Stop Reasons og ukjente typer.
- **Kontekst:** Minimer data, autoriser retrieval, tell tokens og versjoner kilder.
- **Verktøy:** Håndhev skjema, faglig autorisasjon, idempotens, timeout, outputgrense og audit.
- **MCP:** Kontroller serveropprinnelse, transport, scopes, dataklasser og risiko for Prompt Injection.
- **Claude Code:** Begrens brukerrettigheter, hemmeligheter, nettverk, sandbox, Settings, Hooks og verktøypolicy.
- **Identitet:** Skill organisasjon, workspace, bruker-, API- og adminnøkler tydelig.
- **Kapasitet:** Overvåk RPM, ITPM, OTPM, Spend, Cache og retrykostnader.
- **Personvern:** Vurder produktgrensesnitt, leverandør, oppbevaring, ZDR, tredjepartsverktøy og logger individuelt.
- **Kvalitet:** Bruk evals, skjema- og kildekontroll samt faglig godkjenning før effekt.
- **Recovery:** Hold prompter, policyer, verktøy, retrieval, jobber og provisjonering reproduserbare.
- **Bevis:** Utfør en fullstendig test fra autentisering via Message og Tool til audit og kostnadsrapport.

## Kilder

- [Anthropic – Introducing Claude](https://www.anthropic.com/news/introducing-claude)
- [Claude Platform – Models overview](https://platform.claude.com/docs/en/about-claude/models/overview)
- [Anthropic – Claude’s Constitution](https://www.anthropic.com/research/claudes-constitution)
- [Anthropic – Claude 3 Model Card](https://assets.anthropic.com/m/61e7d27f8c8f5919/original/Claude-3-Model-Card.pdf)
- [Claude API – Create a Message](https://platform.claude.com/docs/en/api/messages/create)
- [Claude Platform – Stop reasons](https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons)
- [Claude Platform – API versioning](https://platform.claude.com/docs/en/api/versioning)
- [Claude Platform – Model IDs and versions](https://platform.claude.com/docs/en/about-claude/models/model-ids-and-versions)
- [Microsoft – Invoke-RestMethod](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod)
- [Microsoft – ConvertTo-Json](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertto-json)
- [curl – Håndbok](https://curl.se/docs/manpage.html)
- [jq – Håndbok](https://jqlang.org/manual/)
- [Claude Platform – Token counting](https://platform.claude.com/docs/en/build-with-claude/token-counting)
- [Claude Platform – Search results and citations](https://platform.claude.com/docs/en/build-with-claude/search-results)
- [Claude Platform – Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [Claude Platform – Workspaces](https://platform.claude.com/docs/en/manage-claude/workspaces)
- [Claude Platform – Streaming messages](https://platform.claude.com/docs/en/build-with-claude/streaming)
- [Claude Platform – How tool use works](https://platform.claude.com/docs/en/agents-and-tools/tool-use/how-tool-use-works)
- [Claude Platform – Tool use overview](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview)
- [Anthropic – Model Context Protocol](https://docs.anthropic.com/en/docs/mcp)
- [Model Context Protocol – Specification](https://modelcontextprotocol.io/specification/2025-06-18)
- [Claude Platform – MCP connector](https://platform.claude.com/docs/en/agents-and-tools/mcp-connector)
- [Anthropic – Sette opp Claude Code](https://docs.anthropic.com/en/docs/claude-code/getting-started)
- [Anthropic – Claude Code CLI reference](https://docs.anthropic.com/en/docs/claude-code/cli-usage)
- [Anthropic – Claude Code security](https://docs.anthropic.com/en/docs/claude-code/security)
- [Microsoft – Get-Command](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/get-command)
- [Microsoft – Get-FileHash](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash)
- [Microsoft – Get-ChildItem](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-childitem)
- [POSIX – command](https://pubs.opengroup.org/onlinepubs/9799919799/utilities/command.html)
- [Linux man-pages – file(1)](https://man7.org/linux/man-pages/man1/file.1.html)
- [GNU Coreutils – sha2 utilities](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Linux man-pages – find(1)](https://man7.org/linux/man-pages/man1/find.1.html)
- [Microsoft – Get-Content](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-content)
- [Microsoft – Tee-Object](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/tee-object)
- [Microsoft – ConvertFrom-Json](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json)
- [GNU Coreutils – cat](https://www.gnu.org/software/coreutils/manual/html_node/cat-invocation.html)
- [GNU Coreutils – tee](https://www.gnu.org/software/coreutils/manual/html_node/tee-invocation.html)
- [Claude Platform – Admin API](https://platform.claude.com/docs/en/manage-claude/admin-api)
- [Claude Platform – Rate limits](https://platform.claude.com/docs/en/api/rate-limits)
- [Claude Platform – API errors](https://platform.claude.com/docs/en/api/errors)
- [Claude Platform – Batch processing](https://platform.claude.com/docs/en/build-with-claude/batch-processing)
- [Anthropic Privacy Center – Oppbevaring av kommersielle data](https://privacy.claude.com/en/articles/7996866-how-long-do-you-store-my-organization-s-data)
- [Anthropic Privacy Center – Zero Data Retention](https://privacy.claude.com/en/articles/8956058-i-have-a-zero-data-retention-agreement-with-anthropic-what-products-does-it-apply-to)
- [Anthropic – Claude Code corporate proxy](https://docs.anthropic.com/en/docs/claude-code/corporate-proxy)
- [Microsoft – Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)
- [Microsoft – Test-NetConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection)
- [Microsoft – Get-NetTCPConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection)
- [BIND 9 – dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Linux man-pages – ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Claude API – Usage Report](https://platform.claude.com/docs/en/api/admin/usage_report)
- [Anthropic – Claude 2](https://www.anthropic.com/news/claude-2)
- [Anthropic – Claude 3 family](https://www.anthropic.com/news/claude-3-family)
- [Anthropic – Claude 3.7 and Claude Code](https://www.anthropic.com/news/claude-3-7-sonnet)
- [Anthropic – Claude 4 and Claude Code GA](https://www.anthropic.com/news/claude-4)
