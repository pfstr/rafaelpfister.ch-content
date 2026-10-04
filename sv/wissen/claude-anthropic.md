---
title: "Claude och Claude Code: arkitektur, API och säker agentdrift"
blatt: "claude"
description: "Claude och Claude Code för plattforms-, säkerhets- och automationsadministratörer: modell- och API-gränser, tillståndslösa Messages, Content Blocks, streaming, Tool Use och MCP, lokal agentkörning, Workspaces och nycklar, Rate Limits, caching, batcher, observability, dataskydd, Prompt Injection och återställning."
fakten:
  - label: Systemroll
    wert: Claude är en familj av generativa språkmodeller; applikationer tillhandahåller kontext och bearbetar probabilistiskt genererade Content Blocks
    href: https://platform.claude.com/docs/en/about-claude/models/overview
  - label: Leverantör
    wert: Anthropic utvecklar modellerna, det direkta Claude API:t, Claude-applikationer och Claude Code
    href: https://www.anthropic.com/news/introducing-claude
  - label: Kärn-API
    wert: POST /v1/messages bearbetar strukturerade meddelanden; samtalstillstånd skickas på nytt av klienten
    href: https://platform.claude.com/docs/en/api/messages/create
  - label: Transport
    wert: HTTPS/JSON; streaming använder Server-Sent Events utan att ändra det semantiska Message-kontraktet
    href: https://platform.claude.com/docs/en/build-with-claude/streaming
  - label: Utdata
    wert: ett Message innehåller typade Content Blocks, användningsvärden och stop_reason i stället för garanterad fri text
    href: https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons
  - label: Modellbindning
    wert: Modell-ID:n är pinnade kontrakt; funktioner och gränser fastställs via Models API och dokumentation
    href: https://platform.claude.com/docs/en/about-claude/models/model-ids-and-versions
  - label: Verktygskontrakt
    wert: Claude genererar tool_use med JSON-argument; klientkod utför och skickar tillbaka tool_result
    href: https://platform.claude.com/docs/en/agents-and-tools/tool-use/how-tool-use-works
  - label: MCP
    wert: öppet klient-server-protokoll för verktyg, resurser och kontext; varje server utökar data- och åtgärdsbehörigheter
    href: https://docs.anthropic.com/en/docs/mcp
  - label: Claude Code
    wert: lokal agent för repository, filer, shell och verktyg; API-inferens förblir en extern tjänst
    href: https://docs.anthropic.com/en/docs/claude-code/getting-started
  - label: Tenantmodell
    wert: Organisation → Workspace → medlemmar, nycklar, gränser och workspacebundna resurser
    href: https://platform.claude.com/docs/en/manage-claude/workspaces
  - label: Kapacitet
    wert: Spend Limits samt Request-, Input- och Output-Token-gränser; 429 och retry-after styr backoff
    href: https://platform.claude.com/docs/en/api/rate-limits
  - label: Datalagring
    wert: Lagringstid beror på produkt, Workspace, funktion, avtal och säkerhetsklassificering och granskas före användning
    href: https://privacy.claude.com/en/articles/7996866-how-long-do-you-store-my-organization-s-data
werbung:
  - newsletter
ctaThemen:
  - claude
translationSourceHash: a5189670b409d16f452c042199e3237347d4a9a1483be7c38ebbc07a8d640c5f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-04T10:08:39.571Z
translationReview: automatic
---

# Claude och Claude Code: arkitektur, API och säker agentdrift

Claude avser en familj av stora generativa språkmodeller från Anthropic samt flera produkter som använder dessa modeller. Den viktigaste administrativa distinktionen är: **modell, API, applikation och agentkörning är inte samma system**. Modellen genererar en probabilistisk följd av utdata-token från indata-token. Messages API paketerar denna process som ett HTTPS-kontrakt. Claude-applikationer tillför konton, samtalslagring och användargränssnitt. Claude Code tillför en lokal körning på en endpoint med åtkomst till filer, shell och verktyg.

Den första Claude-versionen presenterades 2023 som chatt- och API-tjänst ([Introducing Claude](https://www.anthropic.com/news/introducing-claude)). Anthropic beskriver Claude som en assistent inriktad på hjälpsamt, ärligt och ofarligt beteende. Denna inriktning, ett stort kontextfönster eller en övertygande formulerad text är dock inga bevis på faktisk korrekthet, auktorisering eller säker körning. Ett produktionssystem måste behandla modellsvar som ej betrodd indata som kontrolleras mot schema och policy.

Förklaringen börjar med en begäran till modellen och följer svaret via Messages API, Content Blocks och Tool-anrop. Först när detta flöde är tydligt fördjupas Claude Code, MCP, behörigheter, Prompt Injection, drift och återställning.

Claude är inte ett autonomt operativsystem utan en inferenstjänst. Den omgivande applikationen tillhandahåller kontext, kontrollerar svar, kör godkända verktyg och ansvarar för identiteter, dataåtkomst och revision.

## Arkitekturansats: inferenstjänst plus kontrollerande applikation

Claude körs normalt som en extern inferenstjänst. En klient skickar systeminstruktioner, samtalsmeddelanden, Content Blocks, verktygsdefinitioner och genereringsparametrar. Tjänsten autentiserar och begränsar begäran, tokeniserar kontexten, utför inferens och levererar ett Message eller en händelseström. Applikationen avgör därefter om text ska visas, JSON valideras, ett verktyg köras, ett resultat skickas tillbaka eller körningen avbrytas.

Det direkta API:t är inte den enda leveransvägen. Claude-modeller erbjuds även via molnplattformar; identitet, endpoint, regioner, kvoter, loggning och avtalsvillkor skiljer sig där. Applikationsarkitekturen håller leverantörsadaptrar och verksamhetsflöden åtskilda. Ett modellbyte är ingen enkel DNS-omkoppling när verktygstyper, Context Limits, Stop Reasons eller leverantörsfunktioner skiljer sig åt.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1116" src="/images/kb-interaktiv-claude.svg?v=20260813" title="Interaktive Infografik: Claude von Benutzer, Anwendung und Workspace über Messages API, Tokenisierung und Modellinferenz bis zu Content Blocks, Tool Use, MCP, Claude Code, Rate Limits, Logging, Datenschutz und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-claude.svg?v=20260813">Öppna interaktiv grafik direkt</a>.
</iframe>

## Modellskikt och probabilistisk semantik

Claude-modeller bearbetar text, kod och beroende på modell ytterligare modaliteter. [Modellöversikten](https://platform.claude.com/docs/en/about-claude/models/overview) dokumenterar modellfamiljer, kontextfönster och funktioner. Sådana värden ska inte anges som permanent frysta fakta i en statisk artikel. Före utrullning frågar en klient via Models API eller vid build-tid efter dokumenterade funktioner och sparar den faktiskt använda modellbeteckningen med varje resultat.

En språkmodell är varken en databas eller en Rules Engine. Samma indata kan generera olika utdata beroende på sampling, systemkontext, modellrevision och verktygsresultat. Även vid låg temperatur kvarstår följande egenskaper:

- Fakta kan saknas, vara föråldrade eller uppdiktade.
- Instruktioner kan viktas tvetydigt.
- långa kontexter kan dölja relevanta detaljer.
- strukturliknande text är inget giltigt objekt utan schemaavslutning.
- en rimlig förklaring bevisar inte att ett verktyg faktiskt kördes.
- en modell känner inte till någon behörighet utöver informationen och verktygsgränserna som applikationen upprätthåller.

Den tekniska acceptansen använder därför utvärderingsfall, förväntade scheman, deterministiska kontroller och verksamhetskällor. Ett lyckat modellbenchmark ersätter inte applikationsspecifik mätning av noggrannhet, latens, kostnad och skada vid fel.

## Träning, Constitution och ansvarsområden

Anthropic utvecklade **Constitutional AI** som ett komplement till övervakat lärande och Reinforcement Learning. Modellen kritiserar och reviderar svar utifrån explicita principer; ett andra träningssteg använder AI-genererad feedback. Anthropic förklarar ansatsen och dess gränser i [Claude’s Constitution](https://www.anthropic.com/research/claudes-constitution). [Claude-3-modellkortet](https://assets.anthropic.com/m/61e7d27f8c8f5919/original/Claude-3-Model-Card.pdf) dokumenterar träning, utvärdering och säkerhetsåtgärder för en konkret modellgeneration.

För administratörer är denna historia relevant eftersom den förklarar beteendet, inte för att den ersätter runtime-kontroll. Safety Training kan minska skadliga svar, men kan varken tvinga fram tenantseparation eller verktygsauktorisering. Refusals är normala möjliga modellutdata. Applikationen måste behandla dem som `stop_reason` respektive Content-typ och får inte dra slutsatsen att en åtgärd är säker av att en avvisning saknas.

Först genom Messages API blir den probabilistiska modellen en administrerbar tjänst. Varje begäran överför den nödvändiga samtalskontexten på nytt och får strukturerade Content Blocks som svar.

## Messages API: tillståndslöst samtalskontrakt

`POST /v1/messages` tar emot en lista av `user`- och `assistant`-meddelanden och genererar nästa Assistant-turn. En systemprompt ligger i det separata toppnivåfältet `system`; en `system`-roll i `messages` finns inte. Content kan skickas som sträng eller som en lista av typade block ([Create a Message](https://platform.claude.com/docs/en/api/messages/create)).

API:t är **tillståndslöst** utan ytterligare trådlagring. För en dialog med flera turer skickar klienten den nödvändiga historiken på nytt. Det får direkta följder:

1. Applikationen äger Conversation ID, ordning och lagringstid.
2. Att förkorta, sammanfatta eller utelämna ändrar modellkontexten.
3. Systemprompt, verktygsdefinitioner och historik räknas till inmatningsbudgeten.
4. Ett Provider-Request-ID ersätter inte ett verksamhetsmässigt Job-ID.
5. En retry av samma begäran kan skapa nya utdata och ytterligare kostnader.

Den verksamhetsmässiga operationen får därför ett eget korrelations- och idempotens-ID. Före en retry kontrollerar orkestratorn om ett resultat eller en irreversibel verktygseffekt redan finns.

## Content Blocks och Stop Reasons

Ett lyckat svar är inte nödvändigtvis en enskild text. `content` är en ordnad lista av typade block, exempelvis text, Tool Use, Thinking- eller serververktygsblock. `usage` rapporterar tokenklasser. `stop_reason` beskriver varför genereringen avslutades. Den officiella [Stop-Reason-dokumentationen](https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons) skiljer bland annat mellan `end_turn`, `max_tokens`, `stop_sequence`, `tool_use`, `pause_turn`, `refusal` och `model_context_window_exceeded`.

Endast `end_turn` betyder en naturligt avslutad tur; det bevisar inte verksamhetsmässig fullständighet. `max_tokens` och `model_context_window_exceeded` markerar potentiellt avkortade data. `tool_use` är en uppmaning till orkestratorn, inte en utförd effekt. `pause_turn` kräver en protokollenlig fortsättning. Nya enumvärden kan tillkomma, varför parsrar måste avvisa okända typer synligt eller ignorera dem säkert i stället för att hamna i en standardframgång.

## API-version och modellversion

Varje direkt API-begäran har en `anthropic-version`-header. Denna API-version stabiliserar fält och streamingsemantik, men får inte förhindra nya valfria indata, utdatafält eller enumvarianter ([API versioning](https://platform.claude.com/docs/en/api/versioning)). Klientparsningen byggs därför framåtkompatibelt och loggar okända block.

Skilt från detta finns **modell-ID:t**. Anthropic garanterar en konstant modellversion under livslängden för ett pinnat ID; bekvämlighetsalias kan ha andra regler ([Model IDs and versioning](https://platform.claude.com/docs/en/about-claude/models/model-ids-and-versions)). För revision och reproduktion lagrar ett system:

- API-version och Betaheader,
- modell-ID i stället för enbart visningsnamn,
- systemprompt-/policyversion,
- Toolset- och JSON-schemaversion,
- Retrieval- och dokumentversioner,
- genereringsparametrar,
- Request-, Job- och användarkorrelation,
- Stop Reason, Usage och valideringsresultat.

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

[`Invoke-RestMethod`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod) och [`ConvertTo-Json`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertto-json) hanterar Windows; [`curl`](https://curl.se/docs/manpage.html) och [`jq`](https://jqlang.org/manual/) hanterar Unix. Nyckeln kommer från en Secrets-hanterare och skrivs varken till kommandoraden eller ut.

## Tokenisering, kontext och kostnader

Kontextfönster mäts i token, inte tecken eller filer. Verktygsscheman, systemprompt, meddelanden, bilder, dokument och verktygsresultat bidrar till indata; genererad text och eventuellt Thinking till utdata. [Token Counting API](https://platform.claude.com/docs/en/build-with-claude/token-counting) tar samma indatastruktur som Messages och ger en förhandsuppskattning.

En stor kontext är inget arkiv. Ju mer irrelevant data klienten skickar, desto högre blir kostnaderna, latensen och risken för motstridiga instruktioner. Produktionssystem bygger därför en **Context Assembly Pipeline**:

1. Kontrollera användar- och tenantbehörigheter före Retrieval.
2. Hämta dokument utifrån stabila ID:n och versioner.
3. Märk untrusted content som data, inte som systeminstruktioner.
4. Begränsa datamängd, filtyper och tokenbudget.
5. Bifoga källmetadata och hashar.
6. Validera svaret mot samma källor och scheman.

Search-Result-Content-Blocks kan överföra RAG-källor med titel och ursprung så att Claude genererar citat ([Search results](https://platform.claude.com/docs/en/build-with-claude/search-results)). Dessa citat är bara lika tillförlitliga som Retrieval, dokumentidentiteten och den överförda metadatan.

## Prompt Caching

Prompt Caching lagrar återanvändbara prefix och minskar bearbetningskostnader och latens. Cachehierarkin följer `tools` → `system` → `messages`; ändringar i en tidigare del invaliderar den och efterföljande nivåer. Den officiella [Prompt-Caching-dokumentationen](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) beskriver automatiska och explicita breakpoints samt korta TTL:er.

En cache hit bevisar endast att ett identiskt prefix återanvändes. Den garanterar varken aktuella källdata eller identiska utdata. Cachemätvärden registreras separat för Creation och Read. Workspaces isolerar Prompt Caches på det direkta Claude API:t ([Workspaces](https://platform.claude.com/docs/en/manage-claude/workspaces)). Hemligheter hör inte hemma i prompter trots kort TTL; cache och retention är olika mekanismer.

## Streaming med Server-Sent Events

Med `stream: true` levererar API:t inkrementella Server-Sent Events. Content Blocks påbörjas, utökas med deltan och avslutas; avslutande Message- och Usage-information krävs fortfarande för korrekt bearbetning. [Streaming-dokumentationen](https://platform.claude.com/docs/en/build-with-claude/streaming) beskriver händelsetyper och SDK-ackumulatorer.

En synlig textdelta är ingen commit. Vid anslutningsavbrott kan användaren redan ha sett deltext medan klienten saknar ett komplett Message. JSON- eller verktygsargument får användas först efter ett komplett block och schemavalidering. Gateways och proxies måste hantera långa HTTP-anslutningar, buffering, timeouts och backpressure korrekt.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Claude-Messages und Streaming">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$body = @{
  model = $env:CLAUDE_MODEL
  max_tokens = 256
  messages = @(@{ role = 'user'; content = 'Antworte mit einem Satz.' })
} | ConvertTo-Json -Depth 8
curl.exe --no-buffer --fail-with-body https://api.anthropic.com/v1/messages `
  -H "x-api-key: $env:ANTHROPIC_API_KEY" `
  -H 'anthropic-version: 2023-06-01' -H 'content-type: application/json' `
  --data-raw $body</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">jq -n --arg model "$CLAUDE_MODEL" '{
  model: $model, max_tokens: 256,
  messages: [{role: "user", content: "Antworte mit einem Satz."}]
}' | curl --no-buffer --fail-with-body https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H 'anthropic-version: 2023-06-01' -H 'content-type: application/json' \
  --data-binary @-</code></pre>
  </div>
</div>

Exempelbegäran använder medvetet ett modell-ID som läses från konfiguration. För SSE anges dessutom `stream: true` och den namngivna händelseströmmen parsas protokollenligt; enbart radläsning räcker inte för produktionskod.

Textutdata förändrar ännu inget system. Först Tool Use kopplar ett modellförslag till en åtgärd – och just där måste applikation, behörighet och mänskligt godkännande träda in.

## Tool Use: modellen begär, applikationen agerar

Tool Use är ett kontrakt mellan modell och orkestrator. Applikationen beskriver ett verktyg med namn, syfte och JSON Schema. Claude kan därefter skapa ett `tool_use`-block med argument. Klienten validerar namn och indata, auktoriserar det konkreta anropet, kör det och skickar tillbaka ett `tool_result`-block med samma Tool-Use-ID. Först en ytterligare modelltur kan skapa ett svar av detta ([How tool use works](https://platform.claude.com/docs/en/agents-and-tools/tool-use/how-tool-use-works)).

[Verktygsöversikten](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview) skiljer mellan:

- **Client Tools:** Den egna applikationen kör dem.
- **Anthropic-Schema-Tools:** Anthropic definierar schemat, men klienten kör dem ändå.
- **Server Tools:** Anthropic kör dem i sin infrastruktur och levererar resultatblock.
- **MCP Connector:** API:t ansluter till en fjärransluten MCP-server.

Denna skillnad bestämmer nätverksväg, hemligheter, loggning, dataskydd och felområde. `strict: true` tvingar fram schemaöverensstämmelse, inte verksamhetsmässig korrekthet eller behörighet. Ett syntaktiskt giltigt anrop `delete_user(id)` kan fortfarande radera fel användare.

### Produktionsklar verktygsslinga

En säker orkestrator utför följande steg per Tool Use:

1. Kontrollera verktygsnamn mot allowlist och arbetsflödesfas.
2. Validera JSON Schema strikt och avvisa okända fält.
3. Kontrollera på nytt användar-, tenant- och objektbehörighet på serversidan.
4. Normalisera indata; begränsa sökvägar, URL:er, ID:n och storlekar.
5. Klassificera Read, Write, External Message och Destructive Action.
6. Kräv verksamhetsmässigt godkännande eller fyrögonsprincip vid hög risk.
7. Kör med kortlivad, minimal identitet i isolerad körmiljö.
8. Ange timeout, outputgräns och idempotensnyckel.
9. Rensa resultatet från hemligheter och untrusted instructions.
10. Revidera Tool Use, beslut, effekt och resultat oföränderligt.

Slingan begränsar turer, parallella verktyg, kumulativa kostnader och upprepade fel. Ett verktygsresultat är i sin tur untrusted content. En databasrad eller webbsida kan innehålla Prompt Injection och får inte skriva över orkestratorns policy.

## MCP som protokollgräns

Model Context Protocol standardiserar anslutningen mellan AI-applikationer och verktyg, resurser samt prompter. Anthropic beskriver MCP som ett öppet klient-server-protokoll ([MCP-översikt](https://docs.anthropic.com/en/docs/mcp)); den normativa [MCP-specifikationen](https://modelcontextprotocol.io/specification/2025-06-18) definierar meddelanden och funktioner.

MCP gör integrationer utbytbara, men inte automatiskt tillförlitliga. En server kan läsa data, utlösa åtgärder, leverera mycket stora resultat eller returnera innehåll från tredje part. Att ta upp den i en konfigurationsfil är därför ett mjukvaru- och behörighetsbeslut. Operatör, transport, endpoint, autentisering, verktyg, resurser, scopes, dataklasser, version, timeout, outputgräns och återkallelse inventeras.

[Messages-API-MCP-Connector](https://platform.claude.com/docs/en/agents-and-tools/mcp-connector) är ett serververktyg i Anthropics infrastruktur. En stdio-server som startats lokalt i Claude Code kör däremot på endpointen. Samma protokoll, andra datavägar och felområden.

## Claude Code: lokal agentkörning

Claude Code är en agent för terminal och repository. Den samlar projektkontext, skickar den till en modellendpoint, tolkar modellförslag och använder lokala verktyg för att läsa, ändra och köra. Den officiella [installationen](https://docs.anthropic.com/en/docs/claude-code/getting-started) dokumenterar Windows via WSL eller Git Bash samt macOS och Linux. [CLI-referensen](https://docs.anthropic.com/en/docs/claude-code/cli-usage) beskriver interaktiv drift, Print Mode, JSON-utdata, sessionsfortsättning, modellval och verktygsregler.

Den centrala förtroendegränsen är den lokala processen. Claude Code kan endast agera med operativsystemrättigheterna och nåbara credentials för den startade användaren, men dessa rättigheter kan vara mycket omfattande: repository, SSH-agent, Cloud-CLI, paketregister, Kubernetes, webbläsarcookies, miljövariabler eller production-tunnlar. ”Agenten frågade” är ingen isolering.

[Claude-Codes säkerhetsdokumentation](https://docs.anthropic.com/en/docs/claude-code/security) beskriver skrivskyddade standardvärden, Permission Prompts, projektgränser och skydd mot Prompt Injection. Den fastslår samtidigt att inget system är helt immunt. Icke-interaktiv drift behöver striktare policies eftersom repositoryinnehåll, Issue-text, buildloggar och verktygsutdata kan innehålla angriparinstruktioner.

### Körnings- och behörighetsprofil

En hanterad endpoint eller CI-runner använder:

- ett dedikerat användar- eller arbetslastkonto,
- en projektspecifik arbetskatalog,
- minimala filsystems- och nätverksrättigheter,
- kortlivade credentials begränsade till mål och åtgärd,
- tillåtna och förbjudna verktyg från central policy,
- Container/VM/Sandbox för untrusted builds,
- Secret Scanning före kontextinläsning,
- begränsade turer, tid, output och kostnader,
- Git-diff, tester och godkännande före commit/deploy,
- session-, verktygs- och leverantörskorrelation i auditlogg.

Växeln `--dangerously-skip-permissions` är ingen automatiseringsstrategi. Den tar bort ett skyddsskikt och är endast försvarbar i en föregående sandbox med egen policy.

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

[`Get-Command`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/get-command), [`Get-FileHash`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash) och [`Get-ChildItem`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-childitem) inventerar Windows. POSIX [`command`](https://pubs.opengroup.org/onlinepubs/9799919799/utilities/command.html), [`file`](https://man7.org/linux/man-pages/man1/file.1.html), [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) och [`find`](https://man7.org/linux/man-pages/man1/find.1.html) hanterar Unix. En hash bedöms endast mot en betrodd release- eller paketkälla.

## Icke-interaktiv drift och CI-drift

Print Mode gör Claude Code skriptbar. `--output-format json` respektive `stream-json` ger maskinläsbara resultat; `--max-turns` begränsar agentkörningen. Indata och exitkod förblir en del av jobbloggen. Fritextutdata utvärderas inte som shellscript.

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

[`Get-Content`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-content), [`Tee-Object`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/tee-object) och [`ConvertFrom-Json`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json) hanterar Windows. [`cat`](https://www.gnu.org/software/coreutils/manual/html_node/cat-invocation.html) och [`tee`](https://www.gnu.org/software/coreutils/manual/html_node/tee-invocation.html) hanterar Unix. Ett CI-jobb får dessutom fasta regler för arbetskatalog, nätverk, hemligheter och branch.

## Settings, Hooks och Policies

Claude Code kombinerar användar-, projekt- och hanterade inställningar. Projektfiler kan ligga i repositoryt och ingår därmed i Code Review. Hooks kör kommandon vid definierade livscykelpunkter; de är körbar kod, inte harmlös promptkonfiguration. MCP-servrar utökar de nåbara systemen. En Enterprisepolicy måste därför gemensamt kontrollera Settings, Hooks, Plugins, Skills, MCP och tillåtna shellmönster.

CLI:t kan tillåta behörigheter en gång eller permanent. Breda wildcards snabbar upp arbetet men ökar blast radius. En bra regel tillåter inte ”Bash”, utan en begränsad läsfunktion eller ett föregående typat verktyg. För skrivoperationer kontrolleras målsökväg, branch och diff. För deployer eller externa meddelanden krävs separat godkännande utanför modellen.

## Identitet, organisation och Workspaces

Claude Platform kopplar användning till en organisation och Workspaces. Workspace-nycklar är begränsade till resurser och användning i denna Workspace; medlemmar har Workspace-roller. Separata Workspaces för utveckling, test och produktion separerar nycklar, gränser, batcher, Files och Prompt Caches ([Workspaces](https://platform.claude.com/docs/en/manage-claude/workspaces)).

Admin API och Inference API använder olika nyckeltyper. [Admin-API-dokumentationen](https://platform.claude.com/docs/en/manage-claude/admin-api) beskriver medlemmar, inbjudningar, Workspaces och API Keys. En adminnyckel hör aldrig hemma i en applikation som endast behöver skicka Messages. Offboarding återkallar användaråtkomst, personliga Claude Code-nycklar och vid behov separat skapade tjänstenycklar.

Claude Code kan autentiseras via Console/OAuth, Claude-planer eller Enterpriseleverantörer. Avgörande är den **faktiska billing- och datavägen** för sessionen. Personliga konton i ett företagsrepository kringgår annars Workspace-kontroller, retention, kostnadsställen eller revision.

## Rate Limits, Spend och Backpressure

Anthropic skiljer mellan Spend Limits och Rate Limits. Messages begränsas efter Requests per Minute, Input Tokens per Minute och Output Tokens per Minute. Gränserna använder Token Buckets; korta bursts kan därför utlösa 429 trots ett till synes passande minutmedel. `retry-after` och responseheaders ger ramen för backoff ([Rate limits](https://platform.claude.com/docs/en/api/rate-limits)).

En gateway implementerar:

- kö och prioritet per tenant/arbetsflöde,
- tokenräkning före godkännande av stora jobb,
- exponentiell backoff med jitter och `retry-after`,
- globala och workspacebaserade parallellitetsgränser,
- Circuit Breaker vid 5xx och nätverksfel,
- budgetvarning och hård kostnadsgräns,
- separata mätvärden för Cache Read, Cache Write, Input och Output,
- kontrollerad fallback med dokumenterad kvalitetsförändring.

[API-feldokumentationen](https://platform.claude.com/docs/en/api/errors) skiljer mellan 400, 401, 402, 403, 404, 413, 429, 500 och 529 och levererar `request_id`. SDK:er upprepar vissa transienta fel automatiskt. Dessa retries räknas med vid kapacitets- och kostnadsplanering.

## Batcher och asynkron bearbetning

Message Batches API bearbetar många oberoende begäranden asynkront. Batcher är workspacebundna; enskilda resultat kan lyckas eller misslyckas. Resultattillgänglighet och lagring på serversidan skiljer sig från den synkrona vägen. [Batch-dokumentationen](https://platform.claude.com/docs/en/build-with-claude/batch-processing) anger gränser för storlek, körtid och hämtning.

Ett batchjobb har ett `custom_id` per verksamhetsobjekt, ett indatamanifest, en hämtningscursor och en resultatkontroll. ”Batch completed” betyder endast att alla poster har ett slutligt tillstånd. Importen bearbetar varje resultat idempotent, identifierar saknade ID:n och arkiverar modell-, prompt- och schemaversion. För personuppgifter eller reglerade data kontrolleras om batchvägen alls är tillåten enligt avtal och datalagringsmodell.

Efter funktion och skalning följer datafrågan: Vilket innehåll lämnar det egna systemet, hur länge lagras det och vilka ytterligare operatörer är involverade på molnplattformar?

## Datalagring, Privacy och tredjepartsplattformar

Datalagring beror på gränssnitt och avtal. Anthropic beskriver för kommersiell API-användning en standardradering av indata och utdata inom 30 dagar, men nämner undantag för funktioner med längre lagring, avvikande överenskommelser, Safety Enforcement och rättsliga skyldigheter ([Lagring av kommersiella data](https://privacy.claude.com/en/articles/7996866-how-long-do-you-store-my-organization-s-data)). Claude-applikationer lagrar samtal för produktfunktioner enligt egna regler.

Zero Data Retention är ingen global knapp för varje produkt. [ZDR-dokumentationen](https://privacy.claude.com/en/articles/8956058-i-have-a-zero-data-retention-agreement-with-anthropic-what-products-does-it-apply-to) beskriver behöriga API:er, organisationer och undantag. Vid åtkomst via molnleverantör gäller dessutom dess dataväg, regioner, nyckelhantering och avtal. MCP, Web Search eller externa verktyg kan överföra data till ytterligare ansvariga parter.

Före produktionsgodkännande upprättas en dataflödesmatris:

| Väg | Data | Ansvarig kontroll |
|---|---|---|
| Klient → modellendpoint | Prompt, filer, bilder, verktygsschema | Klassificering, minimering, avtal, region |
| Modellendpoint → klient | Content Blocks, Usage, Request ID | Validering, redaction, loggning |
| Klient → Tool/MCP | Verktygsargument, användarkontext | Auktorisering, scope, DPA, revision |
| Tool → modell | Resultat och untrusted content | Filter, outputgräns, skydd mot injection |
| Claude Code lokalt | Repository, shell, credentials | Endpointpolicy, sandbox, hemligheter |
| Loggar/tracing | Prompter, resultat, metadata | Redaction, åtkomst, lagring |

## Prompt Injection och untrusted content

Prompt Injection uppstår när data försöker bli instruktioner. Angreppsytor är webbsidor, e-post, dokument, Issues, källkodskommentarer, MCP-resurser, verktygsresultat och terminalutdata. Angreppet behöver inte ”övertyga” modellen om applikationen ändå ger ett okontrollerat Tool Use långtgående rättigheter.

Försvaret är flerskiktat:

1. Upprätthåll systempolicy och behörighet utanför modelltexten.
2. Filtrera Retrieval efter användar- och tenantbehörigheter.
3. Märk datablock med ursprung, Trust Level och syfte.
4. Auktorisera verktyg minimalt, typat och objektbaserat.
5. Godkänn skriv-, sändnings-, betalnings- och raderingsåtgärder separat.
6. Sandboxa körningens nätverk och filsystem.
7. Lägg inte hemligheter i kontext, verktygsresultat eller synliga loggar.
8. Validera output och verktygsargument mot schema och verksamhetsregler.
9. Begränsa agentslingor efter tid, turer, kostnader och effekter.
10. Genomför adversariala tester med indirekt injection i verkliga datavägar.

En Human-in-the-Loop är endast effektiv när personen ser målet, effekten och relevanta data. En generisk ”Tillåt?”-uppmaning leder till Approval Fatigue. Högriskåtgärder visar normaliserade parametrar och kontrolleras av en oberoende Policy Enforcement Point.

## Nätverksväg och proxydrift

Claude API och Claude Code behöver HTTPS-åtkomst. En företagsproxy kan hantera autentisering, TLS Inspection, egresskontroll och loggning, men blir därmed en del av kedjan för konfidentialitet och tillgänglighet. [Proxy-dokumentationen för Claude Code](https://docs.anthropic.com/en/docs/claude-code/corporate-proxy) anger proxyvariabler, CA-bundles och nödvändiga mål som stöds.

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

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname), [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) och [`Get-NetTCPConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection) kontrollerar Windows. [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility), [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) och [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) kontrollerar Unix. En 401 utan nyckel bekräftar att DNS, TCP, TLS och HTTP når den förväntade API-vägen; det är inget lyckat inferenstest.

Säker agentdrift måste inte bara räkna modellanrop, utan göra hela vägen från indata via verktygsbeslut till extern effekt spårbar.

## Observability och revision

En produktiv begäran skapar verksamhetsmässiga, tekniska och kostnadsrelaterade mätvärden. Minst följande registreras:

- tid, Workspace, applikation, arbetsflöde och pseudonym användarkorrelation,
- modell-ID, API-version, prompt-/Toolset-/schemaversion,
- latens till header, första token och komplett Message,
- Input-, Cache-Creation-, Cache-Read- och Output-token,
- Stop Reason, Content-Block-typer och valideringsstatus,
- felklass, antal retries, `request_id` och `retry-after`,
- verktygsnamn, auktoriseringsbeslut, varaktighet, effekt och resultatklass,
- uppskattade och fakturerade kostnader,
- Redaction- och lagringsklass.

Prompter och modellsvar loggas inte reflexmässigt i sin helhet. De kan innehålla personuppgifter, hemligheter, källkod eller angriparpayloads. Auditmetadata och debuginnehåll får separata lagringar, roller och raderingsfrister. [Usage Report API](https://platform.claude.com/docs/en/api/admin/usage_report) tillhandahåller aggregerbara API- och Claude Code-användningsdata; lokala verktygseffekter måste ändå revideras i det egna systemet.

## Felhantering och återstart

En modellbegäran är inte automatiskt idempotent. Vid timeout kan leverantören ha slutfört genereringen trots att klienten inte fick något svar. Vid Tool Use kan den externa effekten redan ha inträffat. Orkestratorn lagrar därför tillståndsövergångar:

`accepted → context_built → request_sent → response_received → validated → tool_authorized → tool_executed → result_returned → completed`

För varje övergång finns korrelations-ID, tid, versioner och checkpoint. En retry före `tool_executed` kan upprepa modellturnen; en retry efteråt frågar först av den verksamhetsmässiga effekten. Externa verktyg stöder Idempotency Keys eller bevis med Read-after-Write. Vid okänt tillstånd eskaleras ärendet i stället för att köras blint på nytt.

Fallbackmodeller ändrar kvalitet, kostnad, Context Limit och verktygsbeteende. De är en egen testad väg. Applikationen döljer inte en fallback om den ändrar utsagekraft eller compliance.

## Backup och Recovery

Själva modellen säkerhetskopieras inte av kunden eftersom den är en extern tjänst. Det egna kontrollskiktet samt lagrade data och konfigurationer måste kunna återställas:

- systemprompter, policies och utvärderingsuppsättningar,
- verktygsscheman, implementationer och behörighetsregler,
- MCP-inventarier och godkända serverversioner,
- Workspace- och nyckelprovisionering som kod eller runbook,
- Retrievalkällor, embeddings/index och dokumentversioner,
- Conversation-/jobstatus och idempotensdata,
- Redaction-, audit- och kostnadsmetadata,
- Claude Code-projektinställningar, Hooks och granskade Skills,
- leverantörsadaptrar och testade modellmigreringar.

API Keys återställs inte från backuper, utan återkallas och provisioneras på nytt. Ett Disaster-Recovery-test bygger upp en ny körmiljö, tilldelar minimala identiteter, skickar ett känt testfall, validerar schema och källor, kör ett harmlöst testverktyg och korrelerar leverantörs-, gateway- och verktygsloggar. Grunder finns under [Backup och Disaster Recovery](/kb/backup-dr), [API:er](/kb/apis), [Härdning](/kb/haertung) och [Felsökning](/kb/troubleshooting).

## Teknisk historia

Anthropic grundades 2021 som ett forsknings- och produktföretag för säkra AI-system. Claude presenterades i mars 2023 efter en sluten partnerfas som chatt- och API-assistent ([Introducing Claude](https://www.anthropic.com/news/introducing-claude)). Claude 2 utökade kontext och offentlig tillgänglighet i juli 2023 ([Claude 2](https://www.anthropic.com/news/claude-2)).

År 2024 etablerade Claude-3-familjen beteckningarna Haiku, Sonnet och Opus som nivåer för hastighet, kostnad och kapacitet ([Claude 3](https://www.anthropic.com/news/claude-3-family)). Dessa produktnamn utgör ingen stabil kapabilitetskontroll; applikationer binder pinnade ID:n och frågar gränser separat.

I februari 2025 släpptes Claude Code som Research Preview tillsammans med en Hybrid-Reasoning-modell ([Claude 3.7 och Claude Code](https://www.anthropic.com/news/claude-3-7-sonnet)). I maj 2025 blev Claude Code allmänt tillgängligt; samtidigt kompletterade Anthropic API:t med agentorienterade byggstenar som Code Execution, MCP Connector och längre Prompt Caching ([Claude 4](https://www.anthropic.com/news/claude-4)). Därmed försköts driftfrågan från ”Hur konsumerar jag text?” till ”Vilka data och effekter får ett probabilistiskt planeringsskikt nå?”

Plattformen utvecklade därefter starkare modellversionering, Workspaces, Usage-/Cost-API:er, verktygsscheman, caching, batcher och agent-SDK:er. Den bestående arkitekturprincipen är: Modellen föreslår; API:er och agentramverk transporterar; den kontrollerande applikationen auktoriserar, kör, validerar och reviderar.

## Admin-checklista

De många gränssnitten i Claude blir hanterbara när modell, API, verktyg och lokal process inventeras separat. Checklistan sammanfattar de kontroller som bör kunna påvisas före produktiv användning.

- **Skikt:** Inventera modell, API, Claude-applikation, gateway, verktyg och Claude Code-endpoint separat.
- **Kontrakt:** Hantera API-version, pinnat modell-ID, Content Blocks, Stop Reasons och okända typer.
- **Kontext:** Minimera data, auktorisera Retrieval, räkna token och versionera källor.
- **Verktyg:** Tvinga fram schema, verksamhetsbehörighet, idempotens, timeout, outputgräns och revision.
- **MCP:** Kontrollera serverursprung, transport, scopes, dataklasser och risk för Prompt Injection.
- **Claude Code:** Begränsa användarrättigheter, hemligheter, nätverk, sandbox, Settings, Hooks och Toolpolicy.
- **Identitet:** Separera organisation, Workspace, användar-, API- och adminnycklar tydligt.
- **Kapacitet:** Övervaka RPM, ITPM, OTPM, Spend, cache och retrykostnader.
- **Privacy:** Bedöm produktgränssnitt, leverantör, retention, ZDR, tredjepartsverktyg och loggar var för sig.
- **Kvalitet:** Använd evals, schema- och källkontroll samt verksamhetsgodkännande före effekt.
- **Recovery:** Håll prompter, policies, verktyg, Retrieval, jobb och provisionering reproducerbara.
- **Bevis:** Utför ett komplett test från autentisering via Message och Tool till revision och kostnadsrapport.

## Källor

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
- [curl – Handbok](https://curl.se/docs/manpage.html)
- [jq – Handbok](https://jqlang.org/manual/)
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
- [Anthropic – Installera Claude Code](https://docs.anthropic.com/en/docs/claude-code/getting-started)
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
- [Anthropic Privacy Center – Lagring av kommersiella data](https://privacy.claude.com/en/articles/7996866-how-long-do-you-store-my-organization-s-data)
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
