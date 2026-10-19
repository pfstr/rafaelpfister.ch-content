---
title: "In quale data center viene elaborato Claude? Tracciare la connessione e limitare l'elaborazione alla Svizzera o all'UE"
navTitle: "Data center di Claude"
description: "Quali tappe percorre una richiesta a Claude, perché la traccia termina all'edge di Cloudflare e cosa rende noto Anthropic sui suoi data center. Include il confronto tra i percorsi con sede di elaborazione fissa (API Anthropic, AWS Bedrock, Google Vertex AI, Microsoft Foundry), i limiti di un funzionamento esclusivamente in Svizzera e il costo del vincolo regionale."
date: "2026-10-02"
kategorie: "Claude"
timeToRead: "12 min di lettura"
themen:
  - claude
produkte:
  - "claude"
protokolle:
  - "apis"
  - "tcp"
slug: "in-quale-data-center-elabora-claude-tracciare-la-connessione-e-limitare-l-elaborazione-alla"
translationId: "article-d8b7299589ea96f4"
translationOf: claude-rechenzentrum-schweiz-eu
translationSourceHash: 93ccc2c90b4f24a12ada6d9fd9417465b4e821e9d371fa5d045ed64cfd70e07c
translationModel: gpt-5.6-terra
translatedAt: 2026-10-19T14:51:00.669Z
translationReview: required
url: https://rafaelpfister.ch/it/blog/in-quale-data-center-elabora-claude-tracciare-la-connessione-e-limitare-l-elaborazione-alla
---

Chi utilizza Claude tramite claude.ai, Claude Code o l'API non sa in quale data center il modello elabora la richiesta. È visibile solo il primo tratto della connessione fino al nodo edge più vicino. Per le aziende che trasmettono dati personali o contenuti riservati a un modello linguistico, questo non è sufficiente: devono poter dimostrare a clienti, consulenti per la protezione dei dati o revisori dove avviene l'elaborazione.

Questo articolo mostra fino a che punto sia possibile tracciare autonomamente la connessione, cosa si sappia pubblicamente dei data center di Anthropic e attraverso quali percorsi sia possibile limitare in modo vincolante l'elaborazione all'UE o all'area UE più Svizzera. I prezzi e le regioni sono aggiornati al 2 ottobre 2026.

**Focus Svizzera:** è determinante la legge federale sulla protezione dei dati riveduta (revLPD). Una comunicazione di dati personali all'estero è ammessa ai sensi dell'art. 16 revLPD se il Consiglio federale attesta un livello di protezione adeguato nello Stato destinatario (allegato 1 dell'ordinanza sulla protezione dei dati, OPDa) oppure se esistono garanzie adeguate quali le clausole contrattuali standard. Gli Stati UE e SEE figurano in questo elenco; gli USA vi figurano solo per le aziende certificate ai sensi dello Swiss-U.S. Data Privacy Framework. Un luogo di elaborazione nell'UE semplifica quindi notevolmente la motivazione, ma non sostituisce gli altri obblighi (contratto di trattamento su incarico, informazione delle persone interessate, sicurezza dei dati).

**Nota UE:** per le aziende con sede nell'UE si applicano invece gli artt. 44 e segg. GDPR. La Commissione europea ha attestato alla Svizzera un livello di protezione dei dati adeguato (decisione di adeguatezza 2000/518/CE, confermata nel gennaio 2024); per le aziende UE, un'elaborazione a Zurigo non costituisce quindi un trasferimento verso un paese terzo in senso problematico.

## Quali tappe percorre una richiesta a Claude

Una richiesta a `claude.ai` o `api.anthropic.com` passa attraverso tre tappe:

| Tappa | Chi la gestisce | Visibile per voi? |
|---|---|---|
| Nodo edge (terminazione TLS, protezione dagli abusi) | Cloudflare, per conto di Anthropic | Sì, la posizione è leggibile nell'header |
| Rete interna fino al backend | Cloudflare e Anthropic | No |
| Inferenza (il modello elabora) | Anthropic, su capacità di calcolo di AWS, Google Cloud e propri data center | No, solo indicazione generica `us` o `global` |

I nomi host `claude.ai` e `api.anthropic.com` puntano entrambi all'indirizzo `160.79.104.10`. Esso appartiene al prefisso `160.79.104.0/23`, registrato a nome di Anthropic (AS399358) e, secondo RPKI, validamente originato da Anthropic. Nel routing globale, tuttavia, il prefisso è annunciato esclusivamente tramite Cloudflare (AS13335): Anthropic porta i propri indirizzi IP nella rete Anycast di Cloudflare. Di conseguenza, lo stesso indirizzo riceve risposta da ogni sede Cloudflare nel mondo e la connessione raggiunge il nodo edge più vicino.

## Determinare il nodo edge

Cloudflare rivela la sede che serve la richiesta in una pagina diagnostica e nell'header `cf-ray`. L'abbreviazione dopo il Ray ID è un codice aeroportuale IATA: `ZRH` indica Zurigo, `GVA` Ginevra, `FRA` Francoforte, `MRS` Marsiglia.

```bash
curl -s https://api.anthropic.com/cdn-cgi/trace
```

Nell'output sono rilevanti le righe `colo=` (sede edge) e `loc=` (paese al quale Cloudflare associa il vostro indirizzo sorgente). In Windows PowerShell fornisce lo stesso risultato:

```powershell
$trace = Invoke-WebRequest -Uri "https://api.anthropic.com/cdn-cgi/trace" -UseBasicParsing
$trace.Content
```

Potete leggere l'header con una richiesta HEAD:

```bash
curl -sI https://api.anthropic.com/ \
  | grep -i -E '^(server|cf-ray):'
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-s` | Sopprime l'indicatore di avanzamento e i messaggi di errore di curl |
| `-I` | Invia una richiesta HEAD e mostra solo gli header della risposta |
| `grep -i` | Cerca senza distinguere tra maiuscole e minuscole |
| `-E '^(server\|cf-ray):'` | Espressione regolare estesa: solo le righe che iniziano con `server:` o `cf-ray:` |

</details>

Da una rete svizzera ci si può normalmente aspettare `ZRH` o `GVA`. Se l'uscita Internet si trova altrove, ad esempio in una rete aziendale con uscita centralizzata all'estero, risponde il nodo edge di quella località, per esempio con `colo=FRA` o `colo=MRS`. La sede edge segue quindi l'uscita della vostra rete, non il luogo di elaborazione.

## Perché il traceroute termina all'edge

Un traceroute mostra il percorso fino al nodo edge e non oltre:

```powershell
Test-NetConnection -ComputerName api.anthropic.com -TraceRoute -Hops 20
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-ComputerName api.anthropic.com` | Host di destinazione della misurazione |
| `-TraceRoute` | Determina gli hop dei router nel percorso verso la destinazione |
| `-Hops 20` | Numero massimo di hop da verificare |

</details>

Su Linux e macOS l'equivalente è `traceroute -n api.anthropic.com` (`-n` sopprime la risoluzione DNS degli hop). Nella misurazione, gli ultimi hop prima di `160.79.104.10` si trovavano nell'intervallo di indirizzi `162.158.0.0/15`, appartenente a Cloudflare. Dopodiché la traccia termina: la connessione viene terminata al nodo edge e l'inoltro al backend avviene in una nuova connessione interna non visibile dall'esterno.

RIPEstat mostra a chi appartiene il prefisso e attraverso quale rete viene annunciato:

```bash
curl -s "https://stat.ripe.net/data/prefix-overview/data.json?resource=160.79.104.10"
curl -s "https://stat.ripe.net/data/bgp-state/data.json?resource=160.79.104.0/23"
```

Nel secondo risultato, tutti i percorsi AS terminano con `13335 399358`: Cloudflare è l'unico upstream del prefisso Anthropic.

Secondo le mie ricerche, non esiste una misurazione pubblica che permetta di tracciare il percorso fino a un data center di inferenza e, a causa della terminazione all'edge, non è possibile farlo neppure con strumenti di rete. Servizi di misurazione come llmlatency.dev rilevano soltanto il tempo di risposta fino al primo byte all'edge (a settembre 2026: mediana di 96 ms da US-Central, 199 ms dalla Germania). Il tempo fino al primo token (Time to First Token) potrebbe essere confrontato da più località, ma dipende più dal modello, dalla lunghezza del prompt e dal carico che dalla distanza. Non consente quindi di dedurre in modo affidabile una sede.

## Cosa si sa dei data center di Anthropic

Finora Anthropic non gestisce un proprio data center di inferenza pubblicamente attribuito e dotato di indirizzo. La potenza di calcolo proviene da partnership raccolte da Tim Cadenbach nel blog TCDEV:

| Sede / partner | Dati principali | Ruolo noto |
|---|---|---|
| Project Rainier, New Carlisle (Indiana, USA), Amazon Web Services | Inaugurato nell'ottobre 2025, circa 11 miliardi di USD, circa 500.000 chip Trainium2, oltre 2,2 GW previsti | Anthropic è locatario principale; soprattutto addestramento |
| Google Cloud | Accordo dell'ottobre 2025 per fino a 1 milione di TPU, oltre 1 GW dal 2026 | Regioni non pubblicate |
| Propri data center con Fluidstack | Annunciati nel novembre 2025, 50 miliardi di USD, sedi in Texas e New York, messa in funzione dal 2026 | Ancora in fase di realizzazione |

Addestramento e inferenza non avvengono necessariamente nello stesso luogo. L'inferenza può essere eseguita in qualsiasi regione AWS o Google Cloud e Anthropic non comunica quale sede calcoli la risposta per un utente in Svizzera. È certo solo quanto stabilito dall'informativa sulla privacy: il contraente e titolare del trattamento per i clienti del SEE, del Regno Unito e della Svizzera è Anthropic Ireland, Limited a Dublino; i dati vengono trasferiti a server negli USA o in altri paesi al di fuori del SEE, sulla base di clausole contrattuali standard.

## Il controllo direttamente presso Anthropic: solo USA o globale

Dai modelli della generazione 4.6, l'API Claude conosce il parametro `inference_geo`. Ha esattamente due valori:

| Valore | Effetto | Prezzo |
|---|---|---|
| `global` (predefinito) | Inferenza in qualsiasi regione disponibile | Prezzo di listino |
| `us` | Inferenza esclusivamente negli USA | Prezzo di listino × 1,1 |

Non esiste un valore per l'UE o la Svizzera. Anche la sede di archiviazione del workspace (`workspace geo`) al momento può essere impostata solo su `us`. La risposta indica nel campo `usage.inference_geo` quale impostazione è stata applicata, ma non una regione concreta. Per Haiku 4.5 e i modelli meno recenti, l'API risponde al parametro con un errore 400.

Per claude.ai e Claude Code con un abbonamento (Pro, Max, Team, Enterprise) non esiste un'impostazione del luogo di elaborazione. Tramite le offerte dirette di Anthropic non è quindi possibile limitare l'elaborazione né all'UE né alla Svizzera. Ciò è possibile solo tramite i fornitori cloud che eseguono Claude nelle proprie regioni.

## AWS Bedrock: area UE più Svizzera da Zurigo

AWS Bedrock esegue Claude su infrastruttura AWS. Secondo AWS, Anthropic non ha accesso ai prompt, alle risposte né ai log dei clienti. Scegliete la regione tramite la regione sorgente della chiamata API e l'Inference Profile:

| Tipo di endpoint | Prefisso dell'ID del modello | Luogo di elaborazione |
|---|---|---|
| Global Cross-Region Inference | `global.` | Qualsiasi regione AWS commerciale nel mondo |
| Geographic Cross-Region Inference | `eu.` | Solo regioni all'interno della geografia |
| In-Region | senza prefisso | Solo la regione chiamata |

Per la Svizzera è determinante la regione `eu-central-2` (Zurigo). Qui sono disponibili Opus 5.5, Sonnet 5.5 e Haiku 4.5, tuttavia solo come profilo Global o UE e non In-Region. Non esiste un profilo che elabori esclusivamente a Zurigo. Se il profilo UE viene chiamato da Zurigo, secondo la scheda del modello AWS (documentato per Haiku 4.5 e Sonnet 4.6) distribuisce le richieste su queste regioni:

| Regione | Sede |
|---|---|
| `eu-central-2` | Zurigo |
| `eu-central-1` | Francoforte |
| `eu-north-1` | Stoccolma |
| `eu-south-1` | Milano |
| `eu-south-2` | Spagna |
| `eu-west-1` | Irlanda |
| `eu-west-3` | Parigi |

Zurigo è una possibile destinazione solo se la richiesta proviene da Zurigo. L'elaborazione rimane così nell'area UE più Svizzera. Per Opus 5.5 e Sonnet 5.5, la scheda del modello non indica le regioni di destinazione; l'elenco effettivo viene restituito dall'API:

```bash
aws bedrock get-inference-profile \
  --region eu-central-2 \
  --inference-profile-identifier eu.anthropic.claude-sonnet-5-5 \
  --query "models[].modelArn"
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `bedrock get-inference-profile` | Legge la definizione di un Inference Profile |
| `--region eu-central-2` | Regione sorgente Zurigo; l'elenco delle destinazioni dipende dalla regione sorgente |
| `--inference-profile-identifier` | ID del profilo UE per Sonnet 5.5 |
| `--query "models[].modelArn"` | Mostra solo gli ARN dei modelli; la regione è indicata in ciascun ARN |

</details>

Per impostazione predefinita, i dati vengono archiviati solo nella regione sorgente. Fanno eccezione i contenuti trattenuti per il rilevamento degli abusi: sono conservati nella regione di destinazione. Il trasporto tra le regioni avviene cifrato attraverso la rete AWS.

### Dimostrare la regione di elaborazione per ogni richiesta

Bedrock è l'unico dei percorsi descritti che registra l'effettiva regione di elaborazione per ogni richiesta. CloudTrail la scrive nella regione sorgente nel campo `additionalEventData.inferenceRegion`:

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
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `cloudtrail lookup-events` | Cerca gli eventi CloudTrail degli ultimi 90 giorni |
| `--region eu-central-2` | Regione sorgente in cui sono registrate le chiamate |
| `--lookup-attributes AttributeKey=EventSource,AttributeValue=bedrock.amazonaws.com` | Filtra gli eventi di Bedrock |
| `--max-results 20` | Limita l'output a 20 eventi |
| `--query "Events[].CloudTrailEvent"` | Restituisce solo l'evento completo come testo JSON |
| `--output text` | Output senza contenitore JSON, un evento per riga |
| `jq -r '… \| @tsv'` | Estrae momento, azione e regione di elaborazione come riga separata da tabulazioni |

</details>

Inoltre, una Service Control Policy (SCP) in AWS Organizations permette di bloccare tutte le regioni al di fuori di UE e Svizzera. Se una regione di destinazione di un profilo è bloccata, la richiesta fallisce invece di passare a un'altra regione. Con questa combinazione, il luogo di elaborazione è imposto tecnicamente e dimostrabile per ogni richiesta.

### Claude Code tramite Bedrock nell'UE

Anche Claude Code può essere eseguito tramite Bedrock. In una regione `eu-*`, Claude Code seleziona automaticamente il prefisso `eu.`:

```bash
export CLAUDE_CODE_USE_BEDROCK=1
export AWS_REGION=eu-central-2
claude
```

| Variabile | Effetto |
|---|---|
| `CLAUDE_CODE_USE_BEDROCK=1` | Passa Claude Code dall'API Anthropic a Bedrock |
| `AWS_REGION=eu-central-2` | Regione sorgente Zurigo; ne risulta il profilo UE |

Con `ANTHROPIC_BEDROCK_REGION_PREFIX` è possibile sovrascrivere il prefisso. L'autenticazione AWS avviene tramite i consueti meccanismi (profilo, SSO, variabili d'ambiente). La fatturazione avviene tramite l'account AWS, non tramite un abbonamento Claude.

## Google Vertex AI: solo UE, esplicitamente senza Svizzera

Su Google Cloud (Vertex AI, ora sotto il nome Gemini Enterprise Agent Platform), la situazione è meno favorevole per la Svizzera:

| Endpoint | Modelli (selezione) | Luogo di elaborazione |
|---|---|---|
| `global` | tutti | Qualsiasi regione Google Cloud, senza garanzia |
| Multi-regione `eu` (`aiplatform.eu.rep.googleapis.com`) | Opus 5.5, Sonnet 5.5, Opus 5, Sonnet 5, Opus 4.8, Opus 4.7 | Solo Stati membri dell'UE |
| Regione `europe-west1` (Belgio) | Haiku 4.5, Sonnet 4.6, Opus 4.6 e precedenti | Secondo la pagina del modello, multi-regione Europa |
| Regione `europe-west6` (Zurigo) | nessuno | Claude non disponibile |

Google esclude esplicitamente la Svizzera dall'endpoint multi-regione UE: esso copre solo gli Stati membri dell'UE; Regno Unito e Svizzera non ne fanno parte. Diverse pagine guida svizzere indicano `europe-west6` come percorso per Claude a Zurigo; secondo la tabella ufficiale delle sedi di Google, lì non è disponibile alcun modello Claude. Google non documenta un campo di log con l'effettiva regione di elaborazione per richiesta. Per l'endpoint globale, Google precisa che non è possibile né controllare né determinare la regione di elaborazione.

Vertex AI è quindi adatto a un'elaborazione esclusivamente nell'UE, non a un'elaborazione in Svizzera.

## Microsoft Foundry: al momento nessuna opzione UE

Microsoft offre Claude in Foundry in due varianti. “Hosted on Azure” elabora su infrastruttura Azure, come Global Standard o Data Zone Standard; la Data Zone per Claude esiste solo per gli USA. “Hosted on Anthropic” elabora su infrastruttura Anthropic e Microsoft avverte che i dati possono essere elaborati al di fuori di Azure e al di fuori della regione selezionata. Anthropic indica Foundry per l'Europa come “Coming soon”. In Microsoft 365 Copilot e Copilot Studio, i modelli Anthropic sono esclusi dall'EU Data Boundary e sono disattivati per impostazione predefinita in UE, EFTA (quindi anche in Svizzera) e Regno Unito.

## Panoramica: quale percorso garantisce quale luogo di elaborazione

| Percorso | Solo Svizzera | Solo UE | Area UE più Svizzera | Regione dimostrabile per richiesta |
|---|---|---|---|---|
| claude.ai, Claude Code (abbonamento) | no | no | no | no |
| API Claude con `inference_geo` | no | no | no (solo `us`) | solo `us`/`global` |
| AWS Bedrock, sorgente `eu-central-2`, profilo `eu.` | no | no | sì | sì (CloudTrail) |
| AWS Bedrock, sorgente nell'UE, profilo `eu.` | no | sì | sì | sì (CloudTrail) |
| Google Vertex AI, endpoint `eu` | no | sì | (solo parte UE) | no |
| Microsoft Foundry | no | no | no | no |

Attualmente non esiste alcun fornitore che offra un percorso in grado di garantire che l'elaborazione di Claude sia limitata alla Svizzera. AWS Bedrock con regione sorgente Zurigo e profilo UE è la soluzione più vicina. Se occorre inoltre escludere che i dati lascino l'UE, ad esempio per impegni contrattuali verso clienti UE, il profilo UE può essere chiamato da una regione UE come Francoforte; in tal caso Zurigo non è più una destinazione.

## Anonimizzazione prima del modello: cercasi aziende pilota

Nella maggior parte dei casi, il luogo di elaborazione è rilevante solo perché nel prompt figurano dati personali o informazioni riservate. È qui che interviene un nuovo prodotto della mia azienda, adeptio gmbh: si colloca come livello intermedio tra le applicazioni in Svizzera e i modelli di Anthropic e OpenAI e anonimizza le richieste prima che lascino la Svizzera. Il modello riceve così solo contenuti non riferibili a persone, indipendentemente dal data center in cui elabora.

**Focus Svizzera:** le informazioni che non possono più essere attribuite a una persona determinata o determinabile non sono dati personali ai sensi dell'art. 5 lett. a revLPD. Le norme sulla comunicazione all'estero (art. 16 e segg. revLPD) non si applicano a tali contenuti. Il raggiungimento di questa soglia tramite anonimizzazione dipende dall'implementazione e dai dati concreti; è proprio ciò che intendiamo dimostrare nell'esercizio operativo.

Per l'introduzione cerchiamo aziende che utilizzano Claude o modelli di OpenAI in produzione, oppure ne pianificano l'introduzione, e che impieghino il prodotto nella quotidianità come aziende pilota. Ne beneficiano entrambe le parti:

| La vostra azienda | adeptio |
|---|---|
| I dati personali e le informazioni riservate lasciano la Svizzera solo in forma anonimizzata. La motivazione nei confronti di consulenti per la protezione dei dati, clienti e revisori diventa più semplice. | Conoscenze tratte da prompt e flussi di lavoro reali anziché da dati di test: quali informazioni devono essere riconosciute e dove l'anonimizzazione rimuove troppo o troppo poco. |
| La scelta tra Anthropic, OpenAI, Bedrock o Vertex AI torna a dipendere dall'offerta di modelli e dal prezzo, invece che esclusivamente dal luogo di elaborazione. | Misurazioni di latenza e qualità delle risposte sotto carico reale. |
| Accesso al prodotto prima della disponibilità generale e influenza sui tipi di dati e casi d'uso che coprirà per primi. | Riscontri dalla protezione dei dati e dalla revisione, ai quali allineare le evidenze del prodotto. |
| Contatto diretto con gli sviluppatori invece di un supporto tramite ticket. | Referenze dal mercato svizzero per la disponibilità generale. |

<aside class="offer-box">
  <span class="offer-box__tag">Cercasi aziende pilota</span>
  <p><strong>Utilizzate Claude o OpenAI con dati personali?</strong> Utilizzate il livello di anonimizzazione di adeptio in esercizio operativo e contribuite a definirne l'evoluzione.</p>
  <a class="offer-box__cta" href="/kontakt">Contattateci</a>
</aside>

## Quanto costa il vincolo regionale

Tutti e tre i fornitori cloud e Anthropic stessa applicano, per il vincolo a una regione o geografia, un sovrapprezzo del 10% rispetto all'endpoint globale. In Bedrock, i prezzi dipendono dalla regione sorgente; Zurigo, Francoforte e North Virginia hanno lo stesso costo. Il routing tra regioni non viene addebitato separatamente.

Prezzi di listino in USD per 1 milione di token (input / output):

| Modello | API Anthropic, `global` | API Anthropic, `us` | Bedrock o Vertex, globale | Bedrock `eu.` o Vertex `eu` |
|---|---|---|---|---|
| Opus 5.5 | 4.00 / 20.00 | 4.40 / 22.00 | 4.00 / 20.00 | 4.40 / 22.00 |
| Sonnet 5.5 | 2.00 / 10.00 | 2.20 / 11.00 | 2.00 / 10.00 | 2.20 / 11.00 |
| Haiku 4.5 | 1.00 / 5.00 | non disponibile | 1.00 / 5.00 | 1.10 / 5.50 |
| Sonnet 4.6 | 3.00 / 15.00 | 3.30 / 16.50 | 3.00 / 15.00 | 3.30 / 16.50 |

In Vertex AI, il prezzo della colonna UE si applica a Haiku 4.5 e Sonnet 4.6 nella regione `europe-west1`. Un esempio di calcolo per un'applicazione interna con 50 milioni di token di input e 10 milioni di token di output al mese:

| Modello | Globale | Vincolato all'UE | Costi aggiuntivi al mese |
|---|---|---|---|
| Sonnet 5.5 | 50 × 2 + 10 × 10 = 200 USD | 220 USD | 20 USD |
| Opus 5.5 | 50 × 4 + 10 × 20 = 400 USD | 440 USD | 40 USD |

Il vincolo regionale costa quindi il 10% in più dell'endpoint globale. Rispetto al prezzo di listino direttamente presso Anthropic con `inference_geo: "us"`, un profilo UE su Bedrock ha lo stesso costo. Nella scelta pesano maggiormente i costi indiretti: occorre predisporre e gestire un account AWS o Google Cloud con policy organizzative, logging e avvisi di budget, e i prezzi mensili fissi degli abbonamenti Claude vengono sostituiti da una fatturazione puramente a consumo. La convenienza del passaggio dipende dal volume; un abbonamento ha un prezzo fisso per persona, ma non offre alcun controllo sul luogo di elaborazione.

## Conservazione e addestramento

Oltre al luogo, conta per quanto tempo vengono conservati i dati:

- **API Anthropic:** i prompt e le risposte non vengono utilizzati per l'addestramento, salvo consenso esplicito. Zero Data Retention (ZDR) è disponibile su richiesta per organizzazione e può essere combinata con `inference_geo`. I contenuti contrassegnati dal rilevamento degli abusi restano archiviati fino a due anni anche con ZDR. Per i modelli Fable e Mythos è obbligatoria una conservazione di 30 giorni.
- **AWS Bedrock:** i contenuti non vengono inviati ad Anthropic. La conservazione è selezionabile per modello tra `none`, `default` e `aws_review`; Fable richiede `aws_review` con archiviazione fino a 30 giorni per la verifica da parte di AWS.
- **Google Vertex AI:** Google utilizza i dati per addestramento o fine-tuning solo previo consenso.

## Fonti

1.  [Claude Platform Docs: Data residency](https://platform.claude.com/docs/en/manage-claude/data-residency): parametro `inference_geo`, valori `us` e `global`, geo del workspace, modelli supportati, campo `usage.inference_geo`.

2.  [Claude Platform Docs: Pricing](https://platform.claude.com/docs/en/about-claude/pricing): prezzi di listino dell'API Anthropic e fattore 1,1 per `inference_geo: "us"`.

3.  [Claude Platform Docs: API and data retention](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention): Zero Data Retention, eccezioni per i contenuti contrassegnati e per Fable/Mythos.

4.  [Anthropic: Privacy Policy](https://www.anthropic.com/legal/privacy): Anthropic Ireland quale titolare del trattamento per SEE, Regno Unito e Svizzera, trasferimento negli USA sulla base di clausole contrattuali standard.

5.  [Anthropic: Regional compliance](https://claude.com/regional-compliance): panoramica delle piattaforme che offrono la residenza dei dati in ciascuna regione; Foundry Europa come “Coming soon”.

6.  [Claude Platform Docs: Claude in Amazon Bedrock](https://platform.claude.com/docs/en/build-with-claude/claude-in-amazon-bedrock): tipi di endpoint per regione, sovrapprezzo del 10% per gli endpoint regionali.

7.  [AWS: Scheda del modello Claude Sonnet 4.6](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-4-6.html): regioni di destinazione del profilo UE per regione sorgente, inclusa Zurigo.

8.  [AWS: Scheda del modello Claude Sonnet 5.5](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-5-5.html): disponibilità in `eu-central-2` solo come profilo Geo e Global.

9.  [AWS: Geographic cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/geographic-cross-region-inference.html): permanenza dei dati nella geografia, luogo di archiviazione, blocco delle regioni di destinazione tramite SCP.

10.  [AWS: Cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html): campo `additionalEventData.inferenceRegion` in CloudTrail, prezzo per regione sorgente.

11.  [AWS: Data protection in Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/data-protection.html): nessun accesso dei fornitori di modelli a prompt, risposte e log.

12.  [AWS: Amazon Bedrock Pricing](https://aws.amazon.com/bedrock/pricing/): prezzi di listino per profili Global e Geo, identici per Zurigo, Francoforte e North Virginia.

13.  [AWS Alps Blog: Cross-region inference for EU data processing in Switzerland](https://aws.amazon.com/blogs/alps/unlocking-ai-flexibility-in-switzerland-a-guide-to-cross-region-inference-for-eu-data-processing-and-model-access/): guida di AWS Svizzera al profilo UE da Zurigo.

14.  [Claude Code Docs: Amazon Bedrock](https://code.claude.com/docs/en/amazon-bedrock): variabili d'ambiente e prefisso automatico `eu.` nelle regioni UE.

15.  [Google Cloud: Generative AI locations](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/locations): tabella delle sedi dei modelli Claude, nessuna disponibilità in `europe-west6`.

16.  [Google Cloud: Data residency](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/data-residency): multi-regione UE solo per Stati membri UE, Svizzera e Regno Unito esclusi.

17.  [Google Cloud: Vertex AI Pricing](https://cloud.google.com/vertex-ai/generative-ai/pricing): prezzi di listino globale, multi-regione UE e `europe-west1`.

18.  [Microsoft Learn: Claude models hosting comparison](https://learn.microsoft.com/en-us/azure/foundry/foundry-models/concepts/claude-models-hosting-comparison): Hosted on Azure e Hosted on Anthropic, Data Zone solo USA.

19.  [Microsoft Learn: Anthropic as AI subprocessor in Microsoft 365 Copilot](https://learn.microsoft.com/en-us/microsoft-365/copilot/connect-to-ai-subprocessor): eccezione dall'EU Data Boundary, impostazione predefinita in UE/EFTA.

20.  [TCDEV Blog: Where Are Claude's Data Centers?](https://www.tcdev.de/blog/where-are-claudes-data-centers/): panoramica di Tim Cadenbach su Project Rainier, il contratto Google TPU e i propri data center con Fluidstack.

21.  [Anthropic: Investimento di 50 miliardi di USD nell'infrastruttura statunitense](https://www.anthropic.com/news/anthropic-invests-50-billion-in-american-ai-infrastructure): annuncio dei propri data center in Texas e New York con Fluidstack.

22.  [Claude Platform Docs: IP addresses](https://platform.claude.com/docs/en/api/ip-addresses): intervallo di indirizzi in entrata `160.79.104.0/23`.

23.  [RIPEstat](https://stat.ripe.net/): panoramica del prefisso e stato BGP per `160.79.104.0/23` (AS399358, upstream AS13335).

24.  [llmlatency.dev: Anthropic](https://llmlatency.dev/provider/anthropic): tempi di risposta all'edge per sede di misurazione.

25.  [Fedlex: Ordinanza sulla protezione dei dati (OPDa), allegato 1](https://www.fedlex.admin.ch/eli/cc/2022/568/de): elenco degli Stati con protezione dei dati adeguata, inclusi UE/SEE e USA per le aziende certificate.
