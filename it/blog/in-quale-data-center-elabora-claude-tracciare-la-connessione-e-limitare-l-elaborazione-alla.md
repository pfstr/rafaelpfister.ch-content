---
title: "In quale data center viene elaborato Claude? Tracciare la connessione e limitare l'elaborazione alla Svizzera o all'UE"
navTitle: "Data center di Claude"
description: "Quali passaggi compie una richiesta a Claude, perché la traccia termina al bordo Cloudflare e cosa rivela Anthropic sui propri data center. Include il confronto tra percorsi con luogo di elaborazione fisso (Anthropic API, AWS Bedrock, Google Vertex AI, Microsoft Foundry), i limiti per un utilizzo esclusivamente svizzero e il costo del vincolo regionale."
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
translationSourceHash: 2d2a3c4b5820e6ec99b710813e209ed3b2e6f6fa750028d120a7d5e4bf4e0c17
translationModel: gpt-5.6-terra
translatedAt: 2026-10-26T09:35:38.804Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/in-quale-data-center-elabora-claude-tracciare-la-connessione-e-limitare-l-elaborazione-alla
---

Chi utilizza Claude tramite claude.ai, Claude Code o l'API non sa in quale data center il modello elabori la richiesta. È visibile soltanto il primo tratto della connessione fino al nodo edge più vicino. Per le aziende che trasmettono dati personali o contenuti riservati a un modello linguistico, questo non è sufficiente: devono poter dimostrare a clienti, consulenti per la protezione dei dati o revisori dove avviene l'elaborazione.

Questo articolo mostra fino a che punto sia possibile tracciare la connessione, cosa si sa pubblicamente dei data center di Anthropic e attraverso quali modalità l'elaborazione possa essere limitata in modo vincolante all'UE o all'area UE più Svizzera. I prezzi e le regioni sono aggiornati al 25 ottobre 2026.

**Focus Svizzera:** È determinante la legge federale sulla protezione dei dati riveduta (revDSG). Una comunicazione di dati personali all'estero è ammessa ai sensi dell'art. 16 revDSG se il Consiglio federale attesta al Paese destinatario un livello di protezione adeguato (allegato 1 dell'ordinanza sulla protezione dei dati, DSV) oppure se esistono garanzie adeguate, quali le clausole contrattuali standard. Gli Stati UE e SEE figurano in tale elenco; gli USA solo per le aziende certificate secondo lo Swiss-U.S. Data Privacy Framework. Un luogo di elaborazione nell'UE semplifica quindi notevolmente la motivazione, ma non sostituisce gli altri obblighi (contratto di trattamento su incarico, informazione delle persone interessate, sicurezza dei dati).

**Nota UE:** Per le aziende con sede nell'UE si applicano invece gli artt. 44 e segg. GDPR. La Commissione europea ha attestato alla Svizzera un livello adeguato di protezione dei dati (decisione di adeguatezza 2000/518/CE, confermata nel gennaio 2024); per le aziende UE, un'elaborazione a Zurigo non costituisce dunque un trasferimento verso un Paese terzo in senso problematico.

<aside class="pilot-card">
  <span class="pilot-card__tag">Novità da adeptio · Cerchiamo aziende pilota</span>
  <p class="pilot-card__titel">Utilizzare Claude e OpenAI senza che i dati personali lascino la Svizzera</p>
  <p class="pilot-card__lead">Il nostro livello intermedio si colloca tra le vostre applicazioni in Svizzera e i modelli di Anthropic e OpenAI, anonimizzando ogni richiesta prima che lasci il Paese. Per l'esercizio in produzione cerchiamo aziende che utilizzino il prodotto nella quotidianità.</p>
  <div class="pilot-card__spalten">
    <div class="pilot-card__spalte">
      <p class="pilot-card__spalte-titel">La vostra azienda</p>
      <ul>
        <li>I dati personali lasciano la Svizzera solo anonimizzati</li>
        <li>Motivazione più semplice nei confronti della protezione dei dati e della revisione</li>
        <li>Accesso prima della disponibilità generale e partecipazione alle decisioni sulle funzionalità</li>
        <li>Contatto diretto con gli sviluppatori</li>
      </ul>
    </div>
    <div class="pilot-card__spalte">
      <p class="pilot-card__spalte-titel">adeptio</p>
      <ul>
        <li>Conoscenze ricavate da prompt reali anziché da dati di test</li>
        <li>Valori misurati su latenza e qualità delle risposte sotto carico reale</li>
        <li>Referenze dal mercato svizzero</li>
      </ul>
    </div>
  </div>
  <div class="pilot-card__aktionen">
    <a class="pilot-card__cta" href="/kontakt">Candidarsi come azienda pilota</a>
    <a class="pilot-card__mehr" href="#anonymisierung-vor-dem-modell-pilotunternehmen-gesucht">Dettagli e inquadramento giuridico</a>
  </div>
</aside>

## Quali passaggi compie una richiesta a Claude

Una richiesta a `claude.ai` o `api.anthropic.com` passa attraverso tre stazioni:

| Stazione | Chi la gestisce | Visibile per voi? |
|---|---|---|
| Nodo edge (terminazione TLS, protezione dagli abusi) | Cloudflare, per conto di Anthropic | Sì, posizione leggibile nell'header |
| Rete interna fino al backend | Cloudflare e Anthropic | No |
| Inferenza (il modello elabora) | Anthropic, su capacità di calcolo di AWS, Google Cloud e propri data center | No, soltanto l'indicazione generica `us` o `global` |

I nomi host `claude.ai` e `api.anthropic.com` puntano entrambi all'indirizzo `160.79.104.10`. Esso appartiene al prefisso `160.79.104.0/23`, registrato a nome di Anthropic (AS399358) e proveniente validamente da Anthropic secondo RPKI. Nel routing globale, tuttavia, il prefisso viene annunciato esclusivamente tramite Cloudflare (AS13335): Anthropic inserisce i propri indirizzi IP nella rete anycast di Cloudflare. Di conseguenza, lo stesso indirizzo riceve risposta in tutto il mondo da ogni sede Cloudflare e la connessione giunge al nodo edge più vicino.

## Determinare il nodo edge

Cloudflare rivela la sede che serve la richiesta in una pagina diagnostica e nell'header `cf-ray`. L'abbreviazione dopo il Ray-ID è un codice aeroportuale IATA: `ZRH` indica Zurigo, `GVA` Ginevra, `FRA` Francoforte, `MRS` Marsiglia.

```bash
curl -s https://api.anthropic.com/cdn-cgi/trace
```

Nell'output sono rilevanti le righe `colo=` (sede edge) e `loc=` (Paese al quale Cloudflare associa il vostro indirizzo sorgente). In Windows, PowerShell fornisce lo stesso risultato:

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
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-s` | Sopprime l'indicatore di avanzamento e i messaggi di errore di curl |
| `-I` | Invia una richiesta HEAD e restituisce solo gli header di risposta |
| `grep -i` | Cerca senza considerare maiuscole e minuscole |
| `-E '^(server\|cf-ray):'` | Espressione regolare estesa: solo righe che iniziano con `server:` o `cf-ray:` |

</details>

Da una rete svizzera, in genere ci si attende `ZRH` o `GVA`. Se l'uscita Internet si trova altrove, ad esempio in una rete aziendale con uscita centralizzata all'estero, il nodo edge risponde lì, per esempio con `colo=FRA` o `colo=MRS`. La sede edge segue quindi la vostra uscita di rete, non il luogo di elaborazione.

## Perché il traceroute termina all'edge

Un traceroute mostra il percorso fino al nodo edge e non oltre:

```powershell
Test-NetConnection -ComputerName api.anthropic.com -TraceRoute -Hops 20
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-ComputerName api.anthropic.com` | Host di destinazione della misurazione |
| `-TraceRoute` | Determina gli hop dei router lungo il percorso verso la destinazione |
| `-Hops 20` | Numero massimo di hop da verificare |

</details>

In Linux e macOS corrisponde a `traceroute -n api.anthropic.com` (`-n` sopprime la risoluzione DNS degli hop). Nella misurazione, gli ultimi hop prima di `160.79.104.10` erano nell'intervallo di indirizzi `162.158.0.0/15`, che appartiene a Cloudflare. Dopodiché la traccia termina: la connessione viene terminata al nodo edge, mentre l'inoltro verso il backend avviene in una nuova connessione interna, non visibile dall'esterno.

RIPEstat mostra a chi appartiene il prefisso e tramite quale rete viene annunciato:

```bash
curl -s "https://stat.ripe.net/data/prefix-overview/data.json?resource=160.79.104.10"
curl -s "https://stat.ripe.net/data/bgp-state/data.json?resource=160.79.104.0/23"
```

Nel secondo risultato, tutti i percorsi AS terminano su `13335 399358`: Cloudflare è l'unico upstream del prefisso Anthropic.

Secondo le mie ricerche, non esiste una misurazione pubblica che consenta di tracciare il percorso fino a un data center di inferenza e, data la terminazione all'edge, ciò non è possibile nemmeno con strumenti di rete. Servizi di misurazione come llmlatency.dev rilevano solo il tempo di risposta fino al primo byte all'edge (a settembre 2026: mediana di 96 ms da US-Central, 199 ms dalla Germania). Il tempo fino al primo token (Time to First Token) potrebbe essere confrontato da più sedi, ma dipende più dal modello, dalla lunghezza del prompt e dal carico che dalla distanza. Non è quindi possibile dedurne in modo affidabile una sede.

## Cosa si sa dei data center di Anthropic

Finora Anthropic non gestisce un proprio data center di inferenza con indirizzo pubblicamente attribuito. La capacità di calcolo proviene da partnership raccolte da Tim Cadenbach nel blog TCDEV:

| Sede / partner | Dati principali | Ruolo noto |
|---|---|---|
| Project Rainier, New Carlisle (Indiana, USA), Amazon Web Services | Aperto nell'ottobre 2025, circa 11 miliardi di USD, circa 500 000 chip Trainium2, oltre 2,2 GW pianificati | Anthropic è il principale locatario; soprattutto addestramento |
| Google Cloud | Accordo dell'ottobre 2025 per fino a 1 milione di TPU, oltre 1 GW dal 2026 | Regioni non pubblicate |
| Propri data center con Fluidstack | Annunciati nel novembre 2025, 50 miliardi di USD, sedi in Texas e New York, messa in servizio dal 2026 | Ancora in costruzione |

Addestramento e inferenza non avvengono necessariamente nello stesso luogo. L'inferenza può essere eseguita in qualunque regione AWS o Google Cloud e Anthropic non rivela quale sede elabori la risposta per un utente in Svizzera. È certo solo quanto stabilisce l'informativa sulla privacy: il partner contrattuale e titolare del trattamento per i clienti dello SEE, del Regno Unito e della Svizzera è Anthropic Ireland, Limited a Dublino; i dati sono trasferiti su server negli USA o in altri Paesi al di fuori dello SEE sulla base di clausole contrattuali standard.

## Il controllo direttamente presso Anthropic: solo USA o globale

Dalla generazione 4.6, la Claude API supporta il parametro `inference_geo`. Esso ha esattamente due valori:

| Valore | Effetto | Prezzo |
|---|---|---|
| `global` (predefinito) | Inferenza in qualsiasi regione disponibile | Prezzo di listino |
| `us` | Inferenza esclusivamente negli USA | Prezzo di listino × 1,1 |

Non esiste un valore per l'UE o la Svizzera. Anche la posizione di archiviazione del workspace (`workspace geo`) può al momento essere impostata soltanto su `us`. Nel campo `usage.inference_geo`, la risposta comunica quale impostazione è stata applicata, ma non una regione concreta. Per Haiku 4.5 e i modelli precedenti, l'API risponde al parametro con un errore 400.

Per claude.ai e Claude Code con un abbonamento (Pro, Max, Team, Enterprise) non esiste un'impostazione per il luogo di elaborazione. Tramite le offerte dirette di Anthropic non è quindi possibile limitare l'elaborazione né all'UE né alla Svizzera. Ciò è possibile solo tramite i fornitori cloud, che eseguono Claude nelle proprie regioni.

## AWS Bedrock: area UE più Svizzera a partire da Zurigo

AWS Bedrock esegue Claude su infrastruttura AWS. Secondo AWS, Anthropic non ha accesso a prompt, risposte o log dei clienti. Scegliete la regione tramite la regione sorgente della chiamata API e l'Inference Profile:

| Tipo di endpoint | Prefisso dell'ID modello | Luogo di elaborazione |
|---|---|---|
| Global Cross-Region Inference | `global.` | Qualsiasi regione AWS commerciale nel mondo |
| Geographic Cross-Region Inference | `eu.` | Solo regioni all'interno della geografia |
| In-Region | senza prefisso | Solo la regione chiamata |

Per la Svizzera è determinante la regione `eu-central-2` (Zurigo). Qui sono disponibili Opus 5.5, Sonnet 5.5, Haiku 4.5 e, dal 7 ottobre 2026, Haiku 5.5, ma soltanto come profilo Global o UE e non In-Region. Non esiste un profilo che elabori esclusivamente a Zurigo. Quando il profilo UE viene chiamato da Zurigo, AWS distribuisce le richieste, secondo la scheda del modello (documentato per Opus 5.5, Sonnet 5.5, Haiku 5.5, Haiku 4.5 e Sonnet 4.6), tra queste regioni:

| Regione | Sede |
|---|---|
| `eu-central-2` | Zurigo |
| `eu-central-1` | Francoforte |
| `eu-north-1` | Stoccolma |
| `eu-south-1` | Milano |
| `eu-south-2` | Spagna |
| `eu-west-1` | Irlanda |
| `eu-west-3` | Parigi |

Zurigo è una possibile destinazione solo se la richiesta proviene da Zurigo. L'elaborazione rimane quindi nell'area UE più Svizzera. È vincolante l'elenco che l'API restituisce per la propria regione sorgente:

```bash
aws bedrock get-inference-profile \
  --region eu-central-2 \
  --inference-profile-identifier eu.anthropic.claude-sonnet-5-5 \
  --query "models[].modelArn"
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `bedrock get-inference-profile` | Legge la definizione di un Inference Profile |
| `--region eu-central-2` | Regione sorgente Zurigo; l'elenco delle destinazioni dipende dalla regione sorgente |
| `--inference-profile-identifier` | ID del profilo UE per Sonnet 5.5 |
| `--query "models[].modelArn"` | Restituisce solo gli ARN dei modelli; la regione è indicata in ciascun ARN |

</details>

Per impostazione predefinita, i dati vengono memorizzati solo nella regione sorgente. Fanno eccezione i contenuti trattenuti per il rilevamento degli abusi: vengono conservati nella regione di destinazione. Il trasporto tra le regioni avviene cifrato attraverso la rete AWS.

### Dimostrare la regione di elaborazione per richiesta

Bedrock è l'unica delle modalità descritte che registra l'effettiva regione di elaborazione per ogni richiesta. CloudTrail la scrive nella regione sorgente nel campo `additionalEventData.inferenceRegion`:

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
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `cloudtrail lookup-events` | Cerca gli eventi CloudTrail degli ultimi 90 giorni |
| `--region eu-central-2` | Regione sorgente nella quale vengono registrate le chiamate |
| `--lookup-attributes AttributeKey=EventSource,AttributeValue=bedrock.amazonaws.com` | Filtra gli eventi di Bedrock |
| `--max-results 20` | Limita l'output a 20 eventi |
| `--query "Events[].CloudTrailEvent"` | Restituisce solo l'evento completo come testo JSON |
| `--output text` | Output senza involucro JSON, un evento per riga |
| `jq -r '… \| @tsv'` | Estrae istante, azione e regione di elaborazione in una riga separata da tabulazioni |

</details>

Inoltre, una Service Control Policy (SCP) in AWS Organizations può bloccare tutte le regioni al di fuori dell'UE e della Svizzera. Se una regione di destinazione di un profilo è bloccata, la richiesta fallisce invece di passare a un'altra regione. Con questa combinazione, il luogo di elaborazione è imposto tecnicamente e dimostrabile per ciascuna richiesta.

### Claude Code tramite Bedrock nell'UE

Anche Claude Code può essere utilizzato tramite Bedrock. In una regione `eu-*`, Claude Code seleziona automaticamente il prefisso `eu.`:

```bash
export CLAUDE_CODE_USE_BEDROCK=1
export AWS_REGION=eu-central-2
claude
```

| Variabile | Effetto |
|---|---|
| `CLAUDE_CODE_USE_BEDROCK=1` | Passa Claude Code dalla Anthropic API a Bedrock |
| `AWS_REGION=eu-central-2` | Regione sorgente Zurigo; ne risulta il profilo UE |

Con `ANTHROPIC_BEDROCK_REGION_PREFIX` è possibile sovrascrivere il prefisso. L'autenticazione AWS avviene tramite i meccanismi usuali (profilo, SSO, variabili d'ambiente). La fatturazione avviene tramite l'account AWS, non tramite un abbonamento Claude.

## Google Vertex AI: solo UE, esplicitamente senza Svizzera

Su Google Cloud (Vertex AI, nel frattempo denominato Gemini Enterprise Agent Platform), la situazione è meno favorevole per la Svizzera:

| Endpoint | Modelli (selezione) | Luogo di elaborazione |
|---|---|---|
| `global` | tutti | Qualsiasi regione Google Cloud, senza garanzia |
| Multi-regione `eu` (`aiplatform.eu.rep.googleapis.com`) | Opus 5.5, Sonnet 5.5, Haiku 5.5, Opus 5, Sonnet 5, Opus 4.8, Opus 4.7 | Solo Stati membri UE |
| Regione `europe-west1` (Belgio) | Haiku 4.5, Sonnet 4.6, Opus 4.6 e precedenti | Secondo la pagina del modello, multi-regione Europa |
| Regione `europe-west6` (Zurigo) | nessuno | Claude non disponibile |

Google esclude espressamente la Svizzera dall'endpoint multi-regione UE: esso copre solo gli Stati membri UE; il Regno Unito e la Svizzera non vi rientrano. Diverse pagine svizzere di consulenza indicano `europe-west6` come percorso verso Claude a Zurigo; secondo la tabella ufficiale delle sedi di Google, tuttavia, nessun modello Claude è disponibile lì. Google non documenta un campo di log con l'effettiva regione di elaborazione per richiesta. Per l'endpoint globale, Google precisa che non è possibile né controllare né determinare la regione di elaborazione.

Vertex AI è quindi adatto a un'elaborazione esclusivamente nell'UE, non a un'elaborazione in Svizzera.

## Microsoft Foundry: attualmente nessuna opzione UE

Microsoft offre Claude in Foundry in due varianti. “Hosted on Azure” elabora su infrastruttura Azure, come Global Standard o Data Zone Standard; la Data Zone è disponibile per Claude solo negli USA. “Hosted on Anthropic” elabora su infrastruttura Anthropic e Microsoft avverte che i dati possono essere elaborati al di fuori di Azure e al di fuori della regione selezionata. Anthropic indica Foundry per l'Europa come “Coming soon”. In Microsoft 365 Copilot e Copilot Studio, i modelli Anthropic sono esclusi dalla EU Data Boundary e sono disattivati per impostazione predefinita nell'UE, nell'EFTA (quindi anche in Svizzera) e nel Regno Unito.

## Panoramica: quale modalità garantisce quale luogo di elaborazione

| Modalità | Solo Svizzera | Solo UE | Area UE più Svizzera | Regione dimostrabile per richiesta |
|---|---|---|---|---|
| claude.ai, Claude Code (abbonamento) | no | no | no | no |
| Claude API con `inference_geo` | no | no | no (solo `us`) | solo `us`/`global` |
| AWS Bedrock, sorgente `eu-central-2`, profilo `eu.` | no | no | sì | sì (CloudTrail) |
| AWS Bedrock, sorgente nell'UE, profilo `eu.` | no | sì | sì | sì (CloudTrail) |
| Google Vertex AI, endpoint `eu` | no | sì | (solo parte UE) | no |
| Microsoft Foundry | no | no | no | no |

Attualmente non esiste alcun fornitore che garantisca che l'elaborazione di Claude sia limitata alla Svizzera. AWS Bedrock con regione sorgente Zurigo e profilo UE è l'opzione più vicina. Se occorre inoltre escludere che i dati lascino l'UE, ad esempio per impegni contrattuali verso clienti UE, il profilo UE può essere chiamato da una regione UE come Francoforte; Zurigo non sarà allora più una destinazione.

## Anonimizzazione prima del modello: cerchiamo aziende pilota

Nella maggior parte dei casi, il luogo di elaborazione è rilevante solo perché il prompt contiene dati personali o informazioni riservate. È qui che interviene un nuovo prodotto della mia azienda, adeptio gmbh: si colloca come livello intermedio tra le applicazioni in Svizzera e i modelli di Anthropic e OpenAI, anonimizzando le richieste prima che lascino la Svizzera. Il modello riceve così solo contenuti privi di riferimenti personali, indipendentemente dal data center in cui elabora.

**Focus Svizzera:** Le informazioni che non sono più attribuibili a una persona determinata o determinabile non sono dati personali ai sensi dell'art. 5 lett. a revDSG. Le norme sulla comunicazione all'estero (art. 16 e segg. revDSG) non si applicano a tali contenuti. Se un'anonimizzazione raggiunga questa soglia dipende dall'implementazione e dai dati concreti; è proprio ciò che vogliamo dimostrare nell'esercizio in produzione.

Per l'introduzione cerchiamo aziende che utilizzano Claude o modelli di OpenAI in produzione, oppure ne pianificano l'introduzione, e che impieghino il prodotto nella quotidianità come aziende pilota. Ne beneficiano entrambe le parti:

| La vostra azienda | adeptio |
|---|---|
| I dati personali e le informazioni riservate lasciano la Svizzera solo anonimizzati. La motivazione nei confronti di consulenti per la protezione dei dati, clienti e revisori risulta più semplice. | Conoscenze ricavate da prompt e flussi di lavoro reali anziché da dati di test: quali informazioni devono essere riconosciute e dove l'anonimizzazione rimuove troppo o troppo poco. |
| La scelta tra Anthropic, OpenAI, Bedrock o Vertex AI torna a dipendere dall'offerta di modelli e dal prezzo, anziché solo dal luogo di elaborazione. | Valori misurati su latenza e qualità delle risposte sotto carico reale. |
| Accesso al prodotto prima della disponibilità generale e influenza sui tipi di dati e casi d'uso che verranno coperti per primi. | Riscontri da protezione dei dati e revisione, in base ai quali orientare le prove del prodotto. |
| Contatto diretto con gli sviluppatori anziché supporto tramite ticket. | Referenze dal mercato svizzero per la disponibilità generale. |

<aside class="offer-box">
  <span class="offer-box__tag">Cerchiamo aziende pilota</span>
  <p><strong>Utilizzate Claude o OpenAI con dati personali?</strong> Impiegate il livello di anonimizzazione di adeptio nell'esercizio in produzione e contribuite a determinarne lo sviluppo.</p>
  <a class="offer-box__cta" href="/kontakt">Contattateci</a>
</aside>

## Quanto costa il vincolo regionale

Tutti e tre i fornitori cloud e Anthropic stessa applicano un supplemento del 10% rispetto all'endpoint globale per il vincolo a una regione o geografia. Per Bedrock, i prezzi dipendono dalla regione sorgente; Zurigo, Francoforte e North Virginia hanno lo stesso costo. Il routing tra le regioni non comporta costi aggiuntivi.

Prezzi di listino in USD per 1 milione di token (input / output):

| Modello | Anthropic API, `global` | Anthropic API, `us` | Bedrock o Vertex, globale | Bedrock `eu.` o Vertex `eu` |
|---|---|---|---|---|
| Opus 5.5 | 4.00 / 20.00 | 4.40 / 22.00 | 4.00 / 20.00 | 4.40 / 22.00 |
| Sonnet 5.5 | 2.00 / 10.00 | 2.20 / 11.00 | 2.00 / 10.00 | 2.20 / 11.00 |
| Haiku 5.5 (prompt fino a 100 000 token) | 0.10 / 0.50 | 0.11 / 0.55 | 0.10 / 0.50 | 0.11 / 0.55 |
| Haiku 4.5 | 1.00 / 5.00 | non disponibile | 1.00 / 5.00 | 1.10 / 5.50 |
| Sonnet 4.6 | 3.00 / 15.00 | 3.30 / 16.50 | 3.00 / 15.00 | 3.30 / 16.50 |

In Vertex AI, il prezzo della colonna UE si applica a Haiku 4.5 e Sonnet 4.6 nella regione `europe-west1`. Un esempio di calcolo per un'applicazione interna con 50 milioni di token di input e 10 milioni di token di output al mese:

| Modello | Globale | Vincolato all'UE | Costi aggiuntivi mensili |
|---|---|---|---|
| Sonnet 5.5 | 50 × 2 + 10 × 10 = 200 USD | 220 USD | 20 USD |
| Opus 5.5 | 50 × 4 + 10 × 20 = 400 USD | 440 USD | 40 USD |

Il vincolo regionale costa quindi il 10% in più dell'endpoint globale. Rispetto al prezzo di listino direttamente presso Anthropic con `inference_geo: "us"`, un profilo UE su Bedrock costa uguale. Nella scelta pesano maggiormente i costi indiretti: occorre predisporre e gestire un account AWS o Google Cloud con policy organizzative, logging e avvisi di budget, mentre i prezzi mensili fissi degli abbonamenti Claude vengono sostituiti da una fatturazione puramente a consumo. Se il passaggio sia conveniente dipende dal volume; un abbonamento ha un prezzo fisso per persona, ma non offre alcun controllo sul luogo di elaborazione.

## Conservazione e addestramento

Oltre al luogo, conta anche la durata della conservazione dei dati:

- **Anthropic API:** prompt e risposte non vengono utilizzati per l'addestramento salvo consenso esplicito. Zero Data Retention (ZDR) è disponibile su richiesta per organizzazione e può essere combinata con `inference_geo`. I contenuti segnalati dal rilevamento degli abusi restano memorizzati fino a due anni anche con ZDR. Per i modelli Fable e Mythos è obbligatoria una conservazione di 30 giorni.
- **AWS Bedrock:** i contenuti non vengono inviati ad Anthropic. La conservazione è selezionabile per modello tra `none`, `default` e `aws_review`; Fable richiede `aws_review` con conservazione fino a 30 giorni per la verifica da parte di AWS.
- **Google Vertex AI:** Google utilizza i dati per addestramento o fine-tuning solo previo consenso.

## Cronologia delle modifiche

- **25 ottobre 2026:** Aggiunto: Haiku 5.5 (dal 7 ottobre 2026) con disponibilità a Zurigo e nella multi-regione UE di Google Cloud, nonché prezzi; le regioni di destinazione del profilo UE sono ora documentate anche per Opus 5.5 e Sonnet 5.5, stato di prezzi e regioni al 25 ottobre 2026. Aggiornati i link alle Fonti ai nuovi indirizzi della documentazione Google Cloud.

## Fonti

1.  [Claude Platform Docs: Data residency](https://platform.claude.com/docs/en/manage-claude/data-residency): parametro `inference_geo`, valori `us` e `global`, geo del workspace, modelli supportati, campo `usage.inference_geo`.

2.  [Claude Platform Docs: Pricing](https://platform.claude.com/docs/en/about-claude/pricing): prezzi di listino della Anthropic API e fattore 1,1 per `inference_geo: "us"`.

3.  [Claude Platform Docs: API and data retention](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention): Zero Data Retention, eccezioni per contenuti segnalati e per Fable/Mythos.

4.  [Anthropic: Privacy Policy](https://www.anthropic.com/legal/privacy): Anthropic Ireland quale titolare del trattamento per SEE, Regno Unito e Svizzera, trasferimento negli USA sulla base di clausole contrattuali standard.

5.  [Anthropic: Regional compliance](https://claude.com/regional-compliance): panoramica delle piattaforme che offrono residenza dei dati in ciascuna regione; Foundry Europa come “Coming soon”.

6.  [Claude Platform Docs: Claude in Amazon Bedrock](https://platform.claude.com/docs/en/build-with-claude/claude-in-amazon-bedrock): tipi di endpoint per regione, supplemento del 10% per endpoint regionali.

7.  [AWS: Scheda del modello Claude Sonnet 4.6](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-4-6.html): regioni di destinazione del profilo UE per regione sorgente, inclusa Zurigo.

8.  [AWS: Scheda del modello Claude Sonnet 5.5](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-5-5.html): disponibilità in `eu-central-2` solo come profilo Geo e Global, regioni di destinazione del profilo UE da Zurigo.

9.  [AWS: Geographic cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/geographic-cross-region-inference.html): permanenza dei dati nella geografia, luogo di archiviazione, blocco delle regioni di destinazione tramite SCP.

10.  [AWS: Cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html): campo `additionalEventData.inferenceRegion` in CloudTrail, prezzo per regione sorgente.

11.  [AWS: Data protection in Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/data-protection.html): nessun accesso dei fornitori dei modelli a prompt, risposte e log.

12.  [AWS: Amazon Bedrock Pricing](https://aws.amazon.com/bedrock/pricing/): prezzi di listino per profili Global e Geo, identici per Zurigo, Francoforte e North Virginia.

13.  [AWS Alps Blog: Cross-region inference for EU data processing in Switzerland](https://aws.amazon.com/blogs/alps/unlocking-ai-flexibility-in-switzerland-a-guide-to-cross-region-inference-for-eu-data-processing-and-model-access/): guida di AWS Svizzera al profilo UE da Zurigo.

14.  [Claude Code Docs: Amazon Bedrock](https://code.claude.com/docs/en/amazon-bedrock): variabili d'ambiente e prefisso automatico `eu.` nelle regioni UE.

15.  [Google Cloud: Generative AI locations](https://docs.cloud.google.com/gemini-enterprise-agent-platform/resources/locations): tabella delle sedi dei modelli Claude, nessuna disponibilità in `europe-west6`.

16.  [Google Cloud: Data residency](https://docs.cloud.google.com/gemini-enterprise-agent-platform/resources/data-residency): multi-regione UE solo per Stati membri UE, Svizzera e Regno Unito esclusi.

17.  [Google Cloud: Vertex AI Pricing](https://cloud.google.com/gemini-enterprise-agent-platform/generative-ai/pricing): prezzi di listino globale, multi-regione UE e `europe-west1`.

18.  [Microsoft Learn: Claude models hosting comparison](https://learn.microsoft.com/en-us/azure/foundry/foundry-models/concepts/claude-models-hosting-comparison): Hosted on Azure e Hosted on Anthropic, Data Zone solo USA.

19.  [Microsoft Learn: Anthropic as AI subprocessor in Microsoft 365 Copilot](https://learn.microsoft.com/en-us/microsoft-365/copilot/connect-to-ai-subprocessor): eccezione dalla EU Data Boundary, impostazione predefinita in UE/EFTA.

20.  [TCDEV Blog: Where Are Claude's Data Centers?](https://www.tcdev.de/blog/where-are-claudes-data-centers/): panoramica di Tim Cadenbach su Project Rainier, il contratto Google TPU e i propri data center con Fluidstack.

21.  [Anthropic: Investimento di 50 miliardi di USD in infrastruttura USA](https://www.anthropic.com/news/anthropic-invests-50-billion-in-american-ai-infrastructure): annuncio dei propri data center in Texas e New York con Fluidstack.

22.  [Claude Platform Docs: IP addresses](https://platform.claude.com/docs/en/api/ip-addresses): intervallo di indirizzi in ingresso `160.79.104.0/23`.

23.  [RIPEstat](https://stat.ripe.net/): panoramica del prefisso e stato BGP per `160.79.104.0/23` (AS399358, upstream AS13335).

24.  [llmlatency.dev: Anthropic](https://llmlatency.dev/provider/anthropic): tempi di risposta all'edge per sede di misurazione.

25.  [Fedlex: Ordinanza sulla protezione dei dati (DSV), allegato 1](https://www.fedlex.admin.ch/eli/cc/2022/568/de): elenco degli Stati con protezione dei dati adeguata, inclusi UE/SEE e USA per aziende certificate.

26.  [Anthropic: Claude Haiku 5.5](https://www.anthropic.com/news/claude-haiku-5-5): pubblicazione il 7 ottobre 2026, disponibilità presso AWS, Google Cloud e Microsoft, prezzi con fascia a partire da 100 000 token.

27.  [AWS: Scheda del modello Claude Haiku 5.5](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-haiku-5-5.html): disponibilità in `eu-central-2` come profilo Geo e Global e regioni di destinazione del profilo UE da Zurigo.
