---
title: "Claude e Claude Code: architettura, API e gestione sicura degli agenti"
blatt: "claude"
description: "Claude e Claude Code per amministratori di piattaforme, sicurezza e automazione: limiti di modelli e API, messaggi stateless, Content Blocks, streaming, Tool Use e MCP, runtime locale degli agenti, workspace e chiavi, rate limit, caching, batch, osservabilità, protezione dei dati, prompt injection e recovery."
fakten:
  - label: Ruolo del sistema
    wert: Claude è una famiglia di modelli linguistici generativi; le applicazioni forniscono il contesto ed elaborano Content Blocks generati probabilisticamente
    href: https://platform.claude.com/docs/en/about-claude/models/overview
  - label: Fornitore
    wert: Anthropic sviluppa i modelli, l’API Claude diretta, le applicazioni Claude e Claude Code
    href: https://www.anthropic.com/news/introducing-claude
  - label: API principale
    wert: POST /v1/messages elabora messaggi strutturati; lo stato della conversazione viene reinviato dal client
    href: https://platform.claude.com/docs/en/api/messages/create
  - label: Trasporto
    wert: HTTPS/JSON; lo streaming utilizza Server-Sent Events senza modificare il contratto semantico del messaggio
    href: https://platform.claude.com/docs/en/build-with-claude/streaming
  - label: Output
    wert: un messaggio contiene Content Blocks tipizzati, metriche di utilizzo e stop_reason anziché testo libero garantito
    href: https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons
  - label: Associazione del modello
    wert: gli ID dei modelli sono contratti fissati; capacità e limiti vengono determinati tramite Models API e documentazione
    href: https://platform.claude.com/docs/en/about-claude/models/model-ids-and-versions
  - label: Contratto degli strumenti
    wert: Claude genera tool_use con argomenti JSON; il codice client esegue e rimanda tool_result
    href: https://platform.claude.com/docs/en/agents-and-tools/tool-use/how-tool-use-works
  - label: MCP
    wert: protocollo client-server aperto per strumenti, risorse e contesto; ogni server estende diritti relativi a dati e azioni
    href: https://docs.anthropic.com/en/docs/mcp
  - label: Claude Code
    wert: agente locale per repository, file, shell e strumenti; l’inferenza API rimane un servizio esterno
    href: https://docs.anthropic.com/en/docs/claude-code/getting-started
  - label: Modello multi-tenant
    wert: Organizzazione → Workspace → membri, chiavi, limiti e risorse associate al workspace
    href: https://platform.claude.com/docs/en/manage-claude/workspaces
  - label: Capacità
    wert: Spend Limits e limiti di richieste, token di input e token di output; 429 e retry-after regolano il backoff
    href: https://platform.claude.com/docs/en/api/rate-limits
  - label: Conservazione dei dati
    wert: la conservazione dipende da prodotto, workspace, funzionalità, contratto e classificazione di sicurezza e viene verificata prima dell’uso
    href: https://privacy.claude.com/en/articles/7996866-how-long-do-you-store-my-organization-s-data
werbung:
  - newsletter
ctaThemen:
  - claude
translationSourceHash: a5189670b409d16f452c042199e3237347d4a9a1483be7c38ebbc07a8d640c5f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-04T10:03:23.341Z
translationReview: automatic
---

# Claude e Claude Code: architettura, API e gestione sicura degli agenti

Claude indica una famiglia di grandi modelli linguistici generativi di Anthropic nonché vari prodotti che utilizzano tali modelli. La distinzione amministrativa più importante è: **modello, API, applicazione e runtime dell’agente non sono lo stesso sistema**. Il modello genera dai token di input una sequenza probabilistica di token di output. La Messages API incapsula questo processo come contratto HTTPS. Le applicazioni Claude aggiungono account, memoria delle conversazioni e interfacce utente. Claude Code aggiunge su un endpoint un runtime locale con accesso a file, shell e strumenti.

La prima versione di Claude è stata presentata nel 2023 come servizio chat e API ([Introduzione a Claude](https://www.anthropic.com/news/introducing-claude)). Anthropic descrive Claude come un assistente orientato a un comportamento utile, onesto e innocuo. Tuttavia, questo orientamento, un’ampia finestra di contesto o un testo formulato in modo convincente non sono prove di correttezza fattuale, autorizzazione o esecuzione sicura. Un sistema di produzione deve trattare le risposte del modello come input non attendibile, verificato rispetto a schema e policy.

La spiegazione parte da una richiesta al modello e segue la risposta attraverso Messages API, Content Blocks e chiamate di strumenti. Solo quando questo flusso è chiaro vengono approfonditi Claude Code, MCP, autorizzazioni, prompt injection, esercizio e recovery.

Claude non è un sistema operativo che agisce autonomamente, ma un servizio di inferenza. L’applicazione circostante fornisce il contesto, verifica le risposte, esegue gli strumenti approvati ed è responsabile di identità, accesso ai dati e audit.

## Approccio architetturale: servizio di inferenza più applicazione di controllo

Claude viene normalmente eseguito come servizio di inferenza esterno. Un client invia istruzioni di sistema, messaggi di conversazione, Content Blocks, definizioni di strumenti e parametri di generazione. Il servizio autentica e limita la richiesta, tokenizza il contesto, esegue l’inferenza e restituisce un messaggio o un flusso di eventi. L’applicazione decide quindi se visualizzare testo, validare JSON, eseguire uno strumento, rinviare un risultato o interrompere l’esecuzione.

L’API diretta non è l’unico percorso di distribuzione. I modelli Claude sono offerti anche tramite piattaforme cloud; identità, endpoint, regioni, quote, logging e condizioni contrattuali differiscono. L’architettura applicativa mantiene separati gli adattatori del provider e il workflow funzionale. Un cambio di modello non è una semplice commutazione DNS se differiscono tipi di strumenti, limiti di contesto, Stop Reasons o funzionalità del provider.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1116" src="/images/kb-interaktiv-claude.svg?v=20260813" title="Interaktive Infografik: Claude von Benutzer, Anwendung und Workspace über Messages API, Tokenisierung und Modellinferenz bis zu Content Blocks, Tool Use, MCP, Claude Code, Rate Limits, Logging, Datenschutz und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-claude.svg?v=20260813">Apri direttamente il grafico interattivo</a>.
</iframe>

## Livello del modello e semantica probabilistica

I modelli Claude elaborano testo, codice e, a seconda del modello, ulteriori modalità. La [panoramica dei modelli](https://platform.claude.com/docs/en/about-claude/models/overview) documenta famiglie di modelli, finestre di contesto e capacità. Tali valori non devono essere inseriti come fatti permanentemente congelati in un articolo statico. Prima del rollout, tramite Models API oppure al momento della build, un client interroga le capacità documentate e memorizza l’identificatore del modello effettivamente usato con ogni risultato.

Un modello linguistico non è né un database né una Rules Engine. Gli stessi input possono produrre output diversi in base al sampling, al contesto di sistema, alla revisione del modello e ai risultati degli strumenti. Anche a temperatura bassa, rimangono le seguenti caratteristiche:

- I fatti possono mancare, essere obsoleti o inventati.
- Le istruzioni possono essere ponderate in modo ambiguo.
- Contesti lunghi possono celare dettagli rilevanti.
- Un testo simile a una struttura non è un oggetto valido senza validazione dello schema.
- Una spiegazione plausibile non prova che uno strumento sia stato effettivamente eseguito.
- Un modello non conosce autorizzazioni al di fuori delle informazioni e dei limiti degli strumenti applicati dall’applicazione.

L’accettazione tecnica utilizza pertanto casi di valutazione, schemi attesi, verifiche deterministiche e fonti specialistiche. Un benchmark del modello riuscito non sostituisce la misurazione specifica dell’applicazione di accuratezza, latenza, costi e danni dovuti a errori.

## Addestramento, Constitution e responsabilità

Anthropic ha sviluppato **Constitutional AI** come complemento all’apprendimento supervisionato e al Reinforcement Learning. Il modello critica e rielabora le risposte in base a principi espliciti; una seconda fase di addestramento utilizza feedback generato dall’AI. Anthropic spiega questo approccio e i suoi limiti in [Claude’s Constitution](https://www.anthropic.com/research/claudes-constitution). La [model card di Claude 3](https://assets.anthropic.com/m/61e7d27f8c8f5919/original/Claude-3-Model-Card.pdf) documenta addestramento, valutazione e misure di sicurezza per una generazione concreta di modelli.

Per gli amministratori questa storia è rilevante perché spiega il comportamento, non perché sostituisce un controllo in fase di esecuzione. Il Safety Training può ridurre risposte dannose, ma non può imporre né separazione dei tenant né autorizzazione degli strumenti. I rifiuti sono possibili output normali del modello. L’applicazione deve trattarli come `stop_reason` oppure come tipo di contenuto e non deve dedurre dall’assenza di un rifiuto che un’azione sia sicura.

Dal modello probabilistico nasce un servizio amministrabile solo attraverso la Messages API. Ogni richiesta trasmette nuovamente il contesto conversazionale necessario e riceve Content Blocks strutturati in risposta.

## Messages API: contratto di conversazione stateless

`POST /v1/messages` accetta un elenco di messaggi `user` e `assistant` e genera il turno successivo dell’assistente. Un prompt di sistema si trova nel campo top-level separato `system`; un ruolo `system` in `messages` non esiste. Il contenuto può essere inviato come stringa oppure come elenco di blocchi tipizzati ([Create a Message](https://platform.claude.com/docs/en/api/messages/create)).

L’API è **stateless** senza memoria aggiuntiva dei thread. Per un dialogo multi-turno, il client rinvia la cronologia necessaria. Ciò ha conseguenze dirette:

1. L’applicazione è proprietaria di Conversation ID, ordine e conservazione.
2. Accorciare, riassumere o omettere modifica il contesto del modello.
3. Prompt di sistema, definizioni degli strumenti e cronologia contano nel budget di input.
4. Un ID richiesta del provider non sostituisce un ID job funzionale.
5. Un retry della stessa richiesta può produrre un nuovo output e costi aggiuntivi.

L’operazione funzionale riceve pertanto un proprio ID di correlazione e idempotenza. Prima di un retry, l’orchestratore verifica se esistono già un risultato o un effetto irreversibile dello strumento.

## Content Blocks e Stop Reasons

Una risposta riuscita non è necessariamente un unico testo. `content` è un elenco ordinato di blocchi tipizzati, ad esempio testo, Tool Use, Thinking o blocchi di strumenti server. `usage` riporta classi di token. `stop_reason` descrive il motivo per cui la generazione è terminata. La [documentazione delle Stop Reasons](https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons) distingue tra gli altri `end_turn`, `max_tokens`, `stop_sequence`, `tool_use`, `pause_turn`, `refusal` e `model_context_window_exceeded`.

Solo `end_turn` indica un turno concluso naturalmente; non prova la completezza funzionale. `max_tokens` e `model_context_window_exceeded` indicano dati potenzialmente troncati. `tool_use` è una richiesta all’orchestratore, non un effetto eseguito. `pause_turn` richiede una continuazione conforme al protocollo. Possono essere aggiunti nuovi valori enum; i parser devono pertanto rifiutare visibilmente o ignorare in sicurezza tipi sconosciuti, anziché ricadere in un successo predefinito.

## Versione API e versione del modello

Ogni richiesta API diretta include un header `anthropic-version` . Questa versione API stabilizza campi e semantica di streaming, ma non può impedire nuovi input opzionali, campi di output o varianti enum ([API versioning](https://platform.claude.com/docs/en/api/versioning)). Il parser client viene quindi progettato in modo compatibile con il futuro e registra blocchi sconosciuti.

Da ciò è distinto l’**ID del modello**. Anthropic garantisce per un ID fissato una versione costante del modello durante la sua vita utile; gli alias di convenienza possono avere altre regole ([Model IDs and versioning](https://platform.claude.com/docs/en/about-claude/models/model-ids-and-versions)). Per audit e riproduzione, un sistema memorizza:

- versione API e header beta,
- ID del modello anziché semplici nomi visualizzati,
- versione del prompt di sistema/policy,
- versione del toolset e dello schema JSON,
- versioni di retrieval e documenti,
- parametri di generazione,
- correlazione di richiesta, job e utente,
- Stop Reason, Usage e risultato della validazione.

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

[`Invoke-RestMethod`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod) e [`ConvertTo-Json`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertto-json) elaborano Windows; [`curl`](https://curl.se/docs/manpage.html) e [`jq`](https://jqlang.org/manual/) Unix. La chiave proviene da un archivio di segreti e non viene né scritta sulla riga di comando né emessa.

## Tokenizzazione, contesto e costi

Le finestre di contesto vengono misurate in token, non in caratteri o file. Schemi degli strumenti, prompt di sistema, messaggi, immagini, documenti e risultati degli strumenti contribuiscono all’input; testo generato e, se previsto, Thinking contribuiscono all’output. La [Token Counting API](https://platform.claude.com/docs/en/build-with-claude/token-counting) accetta la stessa struttura di input di Messages e fornisce una stima preventiva.

Un contesto ampio non è un archivio. Più dati irrilevanti invia il client, maggiori diventano costi, latenza e rischio di istruzioni contraddittorie. I sistemi di produzione costruiscono pertanto una **Context Assembly Pipeline**:

1. Verificare diritti dell’utente e del tenant prima del retrieval.
2. Recuperare documenti tramite ID e versioni stabili.
3. Contrassegnare untrusted content come dati, non come istruzioni di sistema.
4. Limitare quantità di dati, tipi di file e budget di token.
5. Includere metadati delle fonti e hash.
6. Validare la risposta rispetto alle stesse fonti e agli schemi.

I Search-Result-Content-Blocks possono trasmettere fonti RAG con titolo e provenienza, affinché Claude generi citazioni ([Search results](https://platform.claude.com/docs/en/build-with-claude/search-results)). Tali citazioni sono affidabili solo quanto il retrieval, l’identità del documento e i metadati trasmessi.

## Prompt Caching

Prompt Caching memorizza prefissi riutilizzabili e riduce costi di elaborazione e latenza. La gerarchia della cache segue `tools` → `system` → `messages`; modifiche a una parte precedente la invalidano insieme ai livelli successivi. La [documentazione di Prompt Caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) descrive breakpoint automatici ed espliciti nonché TTL brevi.

Un cache hit dimostra solo che è stato riutilizzato un prefisso identico. Non garantisce né dati delle fonti aggiornati né un output identico. Le metriche della cache vengono rilevate separatamente per Creation e Read. I workspace isolano le Prompt Cache nella Claude API diretta ([Workspaces](https://platform.claude.com/docs/en/manage-claude/workspaces)). I segreti non devono essere inseriti nei prompt neppure con TTL brevi; cache e retention sono meccanismi distinti.

## Streaming con Server-Sent Events

Con `stream: true` l’API fornisce Server-Sent Events incrementali. I Content Blocks vengono avviati, ampliati tramite delta e completati; per un’elaborazione corretta sono comunque necessarie le informazioni finali di Message e Usage. La [documentazione sullo streaming](https://platform.claude.com/docs/en/build-with-claude/streaming) descrive tipi di eventi e accumulatori SDK.

Un delta di testo visibile non è un commit. In caso di interruzione della connessione, l’utente potrebbe avere già visto testo parziale mentre il client non possiede un Message completo. Gli argomenti JSON o degli strumenti possono essere utilizzati solo dopo il blocco completo e la validazione dello schema. Gateway e proxy devono gestire correttamente connessioni HTTP lunghe, buffering, timeout e backpressure.

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

La richiesta di esempio utilizza deliberatamente un ID modello letto dalla configurazione. Per SSE viene inoltre impostato `stream: true` e il flusso di eventi nominati viene analizzato conformemente al protocollo; per il codice di produzione non basta leggere semplicemente le righe.

L’output testuale non modifica ancora alcun sistema. Solo Tool Use collega una proposta del modello a un’azione; ed è precisamente qui che applicazione, autorizzazione e approvazione umana devono intervenire.

## Tool Use: il modello richiede, l’applicazione agisce

Tool Use è un contratto tra modello e orchestratore. L’applicazione descrive uno strumento con nome, scopo e schema JSON. Claude può quindi generare un blocco `tool_use` con argomenti. Il client valida nome e input, autorizza la chiamata concreta, la esegue e rinvia un blocco `tool_result` con lo stesso ID Tool Use. Solo un ulteriore turno del modello può formare una risposta da ciò ([How tool use works](https://platform.claude.com/docs/en/agents-and-tools/tool-use/how-tool-use-works)).

La [panoramica degli strumenti](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview) distingue:

- **Client Tools:** eseguiti dall’applicazione proprietaria.
- **Anthropic-Schema-Tools:** Anthropic definisce lo schema, ma il client esegue comunque.
- **Server Tools:** Anthropic esegue nella propria infrastruttura e fornisce blocchi di risultato.
- **MCP Connector:** l’API si connette a un server MCP remoto.

Questa distinzione determina percorso di rete, segreti, logging, protezione dei dati e dominio di guasto. `strict: true` impone la conformità allo schema, non la correttezza funzionale o l’autorizzazione. Una chiamata sintatticamente valida `delete_user(id)` può comunque eliminare l’utente sbagliato.

### Ciclo degli strumenti pronto per la produzione

Un orchestratore sicuro esegue i seguenti passaggi per ogni Tool Use:

1. Verificare il nome dello strumento rispetto all’allowlist e alla fase del workflow.
2. Validare rigorosamente lo schema JSON e rifiutare campi sconosciuti.
3. Verificare di nuovo lato server autorizzazione di utente, tenant e oggetto.
4. Normalizzare gli input; limitare percorsi, URL, ID e dimensioni.
5. Classificare Read, Write, External Message e Destructive Action.
6. Per rischi elevati richiedere approvazione funzionale o principio dei quattro occhi.
7. Eseguire con identità minima e di breve durata in un runtime isolato.
8. Impostare timeout, limite di output e chiave di idempotenza.
9. Ripulire il risultato da segreti e untrusted instructions.
10. Sottoporre a audit immutabile Tool Use, decisione, effetto e risultato.

Il ciclo limita turni, strumenti paralleli, costi cumulativi ed errori ripetuti. Un risultato dello strumento è a sua volta untrusted content. Una riga di database o una pagina web può contenere prompt injection e non deve sovrascrivere la policy dell’orchestratore.

## MCP come confine di protocollo

Il Model Context Protocol standardizza il collegamento di applicazioni AI con strumenti, risorse e prompt. Anthropic descrive MCP come protocollo client-server aperto ([panoramica MCP](https://docs.anthropic.com/en/docs/mcp)); la [specifica MCP](https://modelcontextprotocol.io/specification/2025-06-18) normativa definisce messaggi e capacità.

MCP rende le integrazioni intercambiabili, ma non automaticamente affidabili. Un server può leggere dati, attivare azioni, fornire risultati molto grandi o restituire contenuti di terzi. L’inclusione in un file di configurazione è pertanto una decisione software e di autorizzazione. Devono essere inventariati gestore, trasporto, endpoint, autenticazione, strumenti, risorse, scope, classi di dati, versione, timeout, limite di output e revoca.

Il [connettore MCP della Messages API](https://platform.claude.com/docs/en/agents-and-tools/mcp-connector) è uno strumento server nell’infrastruttura Anthropic. Un server stdio avviato localmente in Claude Code viene invece eseguito sull’endpoint. Stesso protocollo, diversi percorsi dei dati e domini di guasto.

## Claude Code: runtime locale degli agenti

Claude Code è un agente per terminale e repository. Raccoglie il contesto del progetto, lo invia a un endpoint del modello, interpreta le proposte del modello e utilizza strumenti locali per leggere, modificare ed eseguire. La [configurazione](https://docs.anthropic.com/en/docs/claude-code/getting-started) ufficiale documenta Windows tramite WSL o Git Bash, nonché macOS e Linux. Il [riferimento CLI](https://docs.anthropic.com/en/docs/claude-code/cli-usage) descrive funzionamento interattivo, Print Mode, output JSON, ripresa della sessione, scelta del modello e regole degli strumenti.

Il confine di fiducia centrale è il processo locale. Claude Code può agire solo con i diritti del sistema operativo e le credenziali raggiungibili dell’utente che lo ha avviato, ma tali diritti possono essere molto estesi: repository, agente SSH, CLI cloud, registri di pacchetti, Kubernetes, cookie del browser, variabili d’ambiente o tunnel di produzione. «L’agente ha chiesto» non è isolamento.

La [documentazione di sicurezza di Claude Code](https://docs.anthropic.com/en/docs/claude-code/security) descrive default di sola lettura, prompt di autorizzazione, confini di progetto e protezione dalla prompt injection. Al contempo chiarisce che nessun sistema è completamente immune. Il funzionamento non interattivo richiede policy più rigorose, poiché contenuto del repository, testo delle issue, log di build e output degli strumenti possono contenere istruzioni di un attaccante.

### Profilo di runtime e autorizzazioni

Un endpoint gestito o un runner CI utilizza:

- un account utente o workload dedicato,
- una directory di lavoro specifica del progetto,
- diritti minimi su file system e rete,
- credenziali di breve durata limitate a destinazione e azione,
- strumenti consentiti e vietati da una policy centrale,
- Container/VM/Sandbox per build non attendibili,
- Secret Scanning prima dell’acquisizione del contesto,
- turni, tempo, output e costi limitati,
- Git-Diff, test e approvazione prima di commit/deploy,
- correlazione di sessione, strumenti e provider nel log di audit.

L’opzione `--dangerously-skip-permissions` non è una strategia di automazione. Rimuove un livello di protezione ed è giustificabile solo in una sandbox a monte con una policy propria.

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

[`Get-Command`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/get-command), [`Get-FileHash`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash) e [`Get-ChildItem`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-childitem) inventariano Windows. POSIX [`command`](https://pubs.opengroup.org/onlinepubs/9799919799/utilities/command.html), [`file`](https://man7.org/linux/man-pages/man1/file.1.html), [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) e [`find`](https://man7.org/linux/man-pages/man1/find.1.html) si occupano di Unix. Un hash viene valutato solo rispetto a una fonte di release o pacchetto attendibile.

## Funzionamento non interattivo e CI

Print Mode rende Claude Code scriptabile. `--output-format json` oppure `stream-json` fornisce risultati leggibili dalla macchina; `--max-turns` limita l’esecuzione dell’agente. Dati di input ed exit code restano parte del protocollo del job. L’output in testo libero non viene valutato come script shell.

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

[`Get-Content`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-content), [`Tee-Object`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/tee-object) e [`ConvertFrom-Json`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json) elaborano Windows. [`cat`](https://www.gnu.org/software/coreutils/manual/html_node/cat-invocation.html) e [`tee`](https://www.gnu.org/software/coreutils/manual/html_node/tee-invocation.html) si occupano di Unix. Un job CI riceve inoltre regole fisse per working directory, rete, segreti e branch.

## Settings, hook e policy

Claude Code combina impostazioni utente, di progetto e gestite. I file di progetto possono trovarsi nel repository e fanno quindi parte del code review. Gli hook eseguono comandi in punti definiti del ciclo di vita; sono codice eseguibile, non configurazione innocua di prompt. I server MCP ampliano i sistemi raggiungibili. Una policy aziendale deve pertanto controllare insieme settings, hook, plugin, skill, MCP e pattern shell consentiti.

La CLI può consentire autorizzazioni una sola volta o permanentemente. Wildcard ampie accelerano il lavoro, ma aumentano il blast radius. Una buona regola non consente «Bash», bensì una funzione di lettura limitata o uno strumento tipizzato a monte. Per operazioni di scrittura vengono verificati percorso di destinazione, branch e diff. Per deploy o messaggi esterni rimane un’approvazione separata esterna al modello.

## Identità, organizzazione e workspace

La Claude Platform assegna l’utilizzo a un’organizzazione e a workspace. Le chiavi del workspace sono limitate alle risorse e all’utilizzo di tale workspace; i membri hanno ruoli nel workspace. Workspace separati per sviluppo, test e produzione separano chiavi, limiti, batch, file e Prompt Cache ([Workspaces](https://platform.claude.com/docs/en/manage-claude/workspaces)).

Admin API e Inference API utilizzano tipi di chiave differenti. La [documentazione Admin API](https://platform.claude.com/docs/en/manage-claude/admin-api) descrive membri, inviti, workspace e API Keys. Una chiave amministrativa non deve mai trovarsi in un’applicazione che deve soltanto inviare Messages. L’offboarding revoca accesso utente, chiavi personali di Claude Code e, se necessario, service key create separatamente.

Claude Code può autenticarsi tramite Console/OAuth, piani Claude o provider enterprise. È determinante il **percorso effettivo di billing e dati** della sessione. Altrimenti gli account personali in un repository aziendale aggirano controlli del workspace, retention, centri di costo o audit.

## Rate limit, spesa e backpressure

Anthropic distingue Spend Limits e Rate Limits. I Messages sono limitati per richieste al minuto, token di input al minuto e token di output al minuto. I limiti utilizzano Token Buckets; brevi burst possono quindi generare 429 nonostante una media al minuto apparentemente adeguata. `retry-after` e gli header della risposta forniscono il quadro per il backoff ([Rate limits](https://platform.claude.com/docs/en/api/rate-limits)).

Un gateway implementa:

- coda e priorità per tenant/workflow,
- conteggio dei token prima di accettare job grandi,
- backoff esponenziale con jitter e `retry-after`,
- limiti di concorrenza globali e per workspace,
- Circuit Breaker in caso di 5xx ed errori di rete,
- avviso di budget e limite massimo rigido dei costi,
- metriche separate per Cache Read, Cache Write, Input e Output,
- fallback controllato con modifica della qualità documentata.

La [documentazione degli errori API](https://platform.claude.com/docs/en/api/errors) distingue 400, 401, 402, 403, 404, 413, 429, 500 e 529 e fornisce `request_id`. Gli SDK ripetono automaticamente determinati errori transitori. Questi retry vengono inclusi nella pianificazione di capacità e costi.

## Batch ed elaborazione asincrona

La Message Batches API elabora molte richieste indipendenti in modo asincrono. I batch sono associati al workspace; i singoli risultati possono riuscire o fallire. Disponibilità dei risultati e conservazione lato server differiscono dal percorso sincrono. La [documentazione sui batch](https://platform.claude.com/docs/en/build-with-claude/batch-processing) indica limiti di dimensione, durata e recupero.

Un job batch possiede un `custom_id` per oggetto funzionale, un manifesto di input, un cursore di recupero e un controllo dei risultati. «Batch completed» significa soltanto che tutte le voci hanno uno stato finale. L’import elabora ogni risultato in modo idempotente, rileva ID mancanti e archivia versione del modello, prompt e schema. Per dati personali o regolamentati viene verificato se il percorso batch sia ammesso dal contratto e dal modello di conservazione dei dati.

Dopo funzionalità e scalabilità segue la questione dei dati: quali contenuti lasciano il sistema proprio, per quanto tempo vengono conservati e quali ulteriori gestori sono coinvolti sulle piattaforme cloud?

## Conservazione dei dati, privacy e piattaforme terze

La conservazione dei dati dipende dall’interfaccia e dal contratto. Per l’utilizzo commerciale delle API, Anthropic descrive una cancellazione standard di input e output entro 30 giorni, ma cita eccezioni per funzionalità con conservazione più lunga, accordi differenti, Safety Enforcement e obblighi legali ([conservazione dei dati commerciali](https://privacy.claude.com/en/articles/7996866-how-long-do-you-store-my-organization-s-data)). Le applicazioni Claude memorizzano le conversazioni per funzioni del prodotto secondo regole proprie.

Zero Data Retention non è un interruttore globale per ogni prodotto. La [documentazione ZDR](https://privacy.claude.com/en/articles/8956058-i-have-a-zero-data-retention-agreement-with-anthropic-what-products-does-it-apply-to) descrive API, organizzazioni ed eccezioni idonee. Per l’accesso tramite cloud provider si applicano inoltre percorso dei dati, regioni, gestione delle chiavi e contratto del provider. MCP, Web Search o strumenti esterni possono trasferire dati ad altri responsabili.

Prima dell’approvazione per la produzione viene creata una matrice dei flussi di dati:

| Percorso | Dati | Controllo responsabile |
|---|---|---|
| Client → endpoint del modello | Prompt, file, immagini, schema degli strumenti | Classificazione, minimizzazione, contratto, regione |
| Endpoint del modello → client | Content Blocks, Usage, Request ID | Validazione, redazione, logging |
| Client → strumento/MCP | Argomenti dello strumento, contesto utente | Autorizzazione, scope, DPA, audit |
| Strumento → modello | Risultato e untrusted content | Filtro, limite di output, protezione injection |
| Claude Code locale | Repository, shell, credenziali | Policy endpoint, sandbox, segreti |
| Log/tracing | Prompt, risultati, metadati | Redazione, accesso, conservazione |

## Prompt injection e untrusted content

La prompt injection si verifica quando i dati tentano di diventare istruzioni. Le superfici di attacco sono pagine web, e-mail, documenti, issue, commenti nel codice sorgente, risorse MCP, risultati degli strumenti e output del terminale. L’attacco non deve «convincere» il modello se l’applicazione assegna comunque diritti estesi a un Tool Use non verificato.

La difesa è a più livelli:

1. Applicare system policy e autorizzazioni al di fuori del testo del modello.
2. Filtrare il retrieval secondo diritti di utente e tenant.
3. Contrassegnare i blocchi di dati con origine, Trust Level e scopo.
4. Autorizzare gli strumenti in modo minimo, tipizzato e riferito all’oggetto.
5. Approvare separatamente azioni di scrittura, invio, pagamento ed eliminazione.
6. Eseguire in sandbox rete e file system.
7. Non fornire segreti nel contesto, nei risultati degli strumenti o nei log visibili.
8. Validare output e argomenti degli strumenti rispetto a schema e regole funzionali.
9. Limitare i cicli degli agenti per tempo, turni, costi ed effetti.
10. Eseguire test avversariali con injection indiretta in percorsi dati reali.

Un Human-in-the-Loop è efficace solo se la persona vede obiettivo, effetto e dati rilevanti. Un generico messaggio «Consentire?» porta ad Approval Fatigue. Le azioni ad alto rischio mostrano parametri normalizzati e vengono controllate da un Policy Enforcement Point indipendente.

## Percorso di rete e funzionamento tramite proxy

Claude API e Claude Code richiedono accesso HTTPS. Un proxy aziendale può gestire autenticazione, TLS Inspection, controllo dell’egress e logging, diventando però parte della catena di riservatezza e disponibilità. La [documentazione proxy per Claude Code](https://docs.anthropic.com/en/docs/claude-code/corporate-proxy) indica variabili proxy supportate, CA bundle e destinazioni necessarie.

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

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname), [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) e [`Get-NetTCPConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection) verificano Windows. [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility), [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) e [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) verificano Unix. Un 401 senza chiave conferma che DNS, TCP, TLS e HTTP raggiungono il percorso API previsto; non è un test di inferenza riuscito.

Una gestione sicura degli agenti non deve limitarsi a contare le chiamate al modello, ma deve rendere tracciabile l’intero percorso dall’input, attraverso la decisione dello strumento, fino all’effetto esterno.

## Osservabilità e audit

Una richiesta di produzione genera metriche funzionali, tecniche e di costo. Vengono rilevati almeno:

- tempo, workspace, applicazione, workflow e correlazione utente pseudonima,
- ID del modello, versione API, versione di prompt/toolset/schema,
- latenza fino agli header, al primo token e al Message completo,
- token di input, Cache Creation, Cache Read e output,
- Stop Reason, tipi di Content Block e stato di validazione,
- classe di errore, numero di retry, `request_id` e `retry-after`,
- nome dello strumento, decisione di autorizzazione, durata, effetto e classe di risultato,
- costi stimati e fatturati,
- classe di redazione e conservazione.

Prompt e risposte del modello non vengono registrati integralmente in modo riflesso. Possono contenere dati personali, segreti, codice sorgente o payload di attaccanti. Metadati di audit e contenuti di debug ricevono archivi, ruoli e periodi di cancellazione separati. La [Usage Report API](https://platform.claude.com/docs/en/api/admin/usage_report) fornisce dati aggregabili di utilizzo API e Claude Code; gli effetti locali degli strumenti devono tuttavia essere sottoposti ad audit nel proprio sistema.

## Gestione degli errori e riavvio

Una richiesta al modello non è automaticamente idempotente. In caso di timeout, il provider può aver completato la generazione benché il client non abbia ricevuto risposta. Con Tool Use, l’effetto esterno può già essersi verificato. L’orchestratore memorizza pertanto transizioni di stato:

`accepted → context_built → request_sent → response_received → validated → tool_authorized → tool_executed → result_returned → completed`

Per ogni transizione esistono ID di correlazione, ora, versioni e checkpoint. Un retry prima di `tool_executed` può ripetere il turno del modello; dopo, prima interroga l’effetto funzionale. Gli strumenti esterni supportano Idempotency Keys o una prova Read-after-Write. In caso di stato sconosciuto si procede con un’escalation, non con una nuova esecuzione alla cieca.

I modelli di fallback modificano qualità, costi, Context Limit e comportamento degli strumenti. Sono un percorso autonomo testato. L’applicazione non nasconde un fallback se modifica il valore informativo o la compliance.

## Backup e recovery

Il modello stesso, come servizio esterno, non viene sottoposto a backup dal cliente. Devono essere ripristinabili i propri dati di controllo, dati memorizzati e configurazioni:

- prompt di sistema, policy e set di valutazione,
- schemi degli strumenti, implementazioni e regole di autorizzazione,
- inventario MCP e versioni server approvate,
- provisioning di workspace e chiavi come codice o runbook,
- fonti di retrieval, embedding/indici e versioni dei documenti,
- stato di conversazioni/job e dati di idempotenza,
- metadati di redazione, audit e costi,
- impostazioni di progetto Claude Code, hook e skill verificati,
- adattatori del provider e migrazioni di modelli testate.

Le API Keys non vengono «ripristinate» dai backup, ma revocate e nuovamente sottoposte a provisioning. Un test di disaster recovery crea un nuovo runtime, assegna identità minime, invia un caso di test noto, valida schema e fonti, esegue uno strumento di test innocuo e correla log di provider, gateway e strumenti. Le basi sono disponibili in [Backup e Disaster Recovery](/kb/backup-dr), [API](/kb/apis), [Hardening](/kb/haertung) e [Troubleshooting](/kb/troubleshooting).

## Storia tecnica

Anthropic è nata nel 2021 come azienda di ricerca e prodotto per sistemi AI sicuri. Claude è stato presentato nel marzo 2023, dopo una fase chiusa per partner, come assistente chat e API ([Introduzione a Claude](https://www.anthropic.com/news/introducing-claude)). Claude 2 ha ampliato nel luglio 2023 il contesto e la disponibilità pubblica ([Claude 2](https://www.anthropic.com/news/claude-2)).

Nel 2024 la famiglia Claude 3 ha stabilito le denominazioni Haiku, Sonnet e Opus come livelli di velocità, costi e capacità ([Claude 3](https://www.anthropic.com/news/claude-3-family)). Questi nomi di prodotto non sono una verifica stabile delle capacità; le applicazioni associano ID fissati e interrogano separatamente i limiti.

Nel febbraio 2025 è apparso Claude Code come Research Preview insieme a un modello di Hybrid Reasoning ([Claude 3.7 e Claude Code](https://www.anthropic.com/news/claude-3-7-sonnet)). Nel maggio 2025 Claude Code è diventato generalmente disponibile; contemporaneamente Anthropic ha integrato nell’API componenti orientati agli agenti come Code Execution, MCP Connector e Prompt Caching più lungo ([Claude 4](https://www.anthropic.com/news/claude-4)). La questione operativa si è così spostata da «Come consumo testo?» a «Quali dati ed effetti può raggiungere un livello di pianificazione probabilistico?»

Successivamente la piattaforma ha sviluppato versioning dei modelli più forte, workspace, Usage/Cost API, schemi degli strumenti, caching, batch e SDK per agenti. Il punto architetturale duraturo rimane: il modello propone; API e framework degli agenti trasportano; l’applicazione di controllo autorizza, esegue, valida e sottopone ad audit.

## Checklist per amministratori

Le molte interfacce di Claude diventano gestibili quando modello, API, strumento e processo locale vengono rilevati separatamente. La checklist riassume i controlli che dovrebbero essere dimostrati prima di un utilizzo produttivo.

- **Livelli:** inventariare separatamente modello, API, applicazione Claude, gateway, strumento ed endpoint Claude Code.
- **Contratti:** gestire versione API, ID modello fissato, Content Blocks, Stop Reasons e tipi sconosciuti.
- **Contesto:** minimizzare dati, autorizzare retrieval, contare token e versionare fonti.
- **Strumenti:** imporre schema, autorizzazione funzionale, idempotenza, timeout, limite di output e audit.
- **MCP:** verificare provenienza del server, trasporto, scope, classi di dati e rischio di prompt injection.
- **Claude Code:** limitare diritti utente, segreti, rete, sandbox, settings, hook e policy degli strumenti.
- **Identità:** separare correttamente organizzazione, workspace, chiavi utente, API e amministrative.
- **Capacità:** monitorare RPM, ITPM, OTPM, spesa, cache e costi di retry.
- **Privacy:** valutare singolarmente interfaccia del prodotto, provider, retention, ZDR, strumenti di terzi e log.
- **Qualità:** utilizzare eval, verifica di schema e fonti nonché approvazione funzionale prima dell’effetto.
- **Recovery:** mantenere riproducibili prompt, policy, strumenti, retrieval, job e provisioning.
- **Prova:** eseguire un test completo dall’autenticazione, attraverso Message e strumento, fino ad audit e report dei costi.

## Fonti

- [Anthropic – Introduzione a Claude](https://www.anthropic.com/news/introducing-claude)
- [Claude Platform – Panoramica dei modelli](https://platform.claude.com/docs/en/about-claude/models/overview)
- [Anthropic – Claude’s Constitution](https://www.anthropic.com/research/claudes-constitution)
- [Anthropic – Claude 3 Model Card](https://assets.anthropic.com/m/61e7d27f8c8f5919/original/Claude-3-Model-Card.pdf)
- [Claude API – Create a Message](https://platform.claude.com/docs/en/api/messages/create)
- [Claude Platform – Stop reasons](https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons)
- [Claude Platform – API versioning](https://platform.claude.com/docs/en/api/versioning)
- [Claude Platform – Model IDs and versions](https://platform.claude.com/docs/en/about-claude/models/model-ids-and-versions)
- [Microsoft – Invoke-RestMethod](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod)
- [Microsoft – ConvertTo-Json](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertto-json)
- [curl – Manuale](https://curl.se/docs/manpage.html)
- [jq – Manuale](https://jqlang.org/manual/)
- [Claude Platform – Token counting](https://platform.claude.com/docs/en/build-with-claude/token-counting)
- [Claude Platform – Search results and citations](https://platform.claude.com/docs/en/build-with-claude/search-results)
- [Claude Platform – Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [Claude Platform – Workspaces](https://platform.claude.com/docs/en/manage-claude/workspaces)
- [Claude Platform – Streaming messages](https://platform.claude.com/docs/en/build-with-claude/streaming)
- [Claude Platform – How tool use works](https://platform.claude.com/docs/en/agents-and-tools/tool-use/how-tool-use-works)
- [Claude Platform – Tool use overview](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview)
- [Anthropic – Model Context Protocol](https://docs.anthropic.com/en/docs/mcp)
- [Model Context Protocol – Specifica](https://modelcontextprotocol.io/specification/2025-06-18)
- [Claude Platform – MCP connector](https://platform.claude.com/docs/en/agents-and-tools/mcp-connector)
- [Anthropic – Configurare Claude Code](https://docs.anthropic.com/en/docs/claude-code/getting-started)
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
- [Anthropic Privacy Center – Conservazione dei dati commerciali](https://privacy.claude.com/en/articles/7996866-how-long-do-you-store-my-organization-s-data)
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
- [Anthropic – Famiglia Claude 3](https://www.anthropic.com/news/claude-3-family)
- [Anthropic – Claude 3.7 e Claude Code](https://www.anthropic.com/news/claude-3-7-sonnet)
- [Anthropic – Claude 4 e disponibilità generale di Claude Code](https://www.anthropic.com/news/claude-4)
