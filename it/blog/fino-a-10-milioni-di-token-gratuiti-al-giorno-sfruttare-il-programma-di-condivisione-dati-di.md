---
title: "Fino a 10 milioni di token gratuiti al giorno: sfruttare il programma di condivisione dati di OpenAI con guardrail sui costi"
navTitle: "Token gratuiti OpenAI"
description: "OpenAI accredita alle organizzazioni che condividono il proprio traffico API per l'addestramento una quota giornaliera gratuita: a seconda del tier, fino a 10 milioni di token. Con credito prepagato, limiti di progetto e un budget di token nel codice, l'utilizzo rimane gratuitamente sostenibile nel tempo."
date: "2026-08-27"
kategorie: "API OpenAI"
timeToRead: "9 min di lettura"
themen:
  - openai-api
produkte:
  - "openai"
protokolle:
  - "apis"
  - "lizenzierung"
slug: "fino-a-10-milioni-di-token-gratuiti-al-giorno-sfruttare-il-programma-di-condivisione-dati-di"
translationId: "article-dde41cbe2dd858e6"
aiPrompt: |
  Du bist mein Assistent für die OpenAI-Plattform. Prüfe mit mir Schritt für Schritt, ob mein OpenAI-Konto für das Data-Sharing-Programm mit Gratis-Tokens sauber abgesichert ist: 1) Billing: Prepaid-Guthaben statt Rechnung, Auto-Reload aus. 2) Data controls → Sharing: "Share inputs and outputs" nur für ein dediziertes Projekt aktiviert, Enrollment-Hinweis sichtbar. 3) Projekt: eigenes Spend-Limit gesetzt, nur ein restricted API-Key. 4) Limits: Spend-Alerts konfiguriert. 5) Code: tägliches Token-Budget deutlich unter Gratis-Kontingent und Tages-Rate-Limit. Frage mich nach meinem Usage-Tier und Modell und rechne mir mein Gratis-Kontingent aus.
translationOf: openai-gratis-tokens-data-sharing
url: https://rafaelpfister.ch/it/blog/fino-a-10-milioni-di-token-gratuiti-al-giorno-sfruttare-il-programma-di-condivisione-dati-di
translationSourceHash: 0f0fef78a8ab264b755061045a34cc765916b1f1b433473f99a5eb6e0538a6b0
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:43:26.816Z
translationReview: automatic
---

# Fino a 10 milioni di token gratuiti al giorno: sfruttare il programma di condivisione dati di OpenAI con guardrail sui costi

OpenAI paga i dati di addestramento con capacità di calcolo anziché denaro: dal dicembre 2024, le organizzazioni che condividono input e output della propria API per l'addestramento ricevono una quota giornaliera di token gratuiti. A seconda dell'Usage Tier e del gruppo di modelli, si tratta di un valore compreso tra 250'000 e 10 milioni di token al giorno. Per molte piccole automazioni è più che sufficiente: una traduzione batch notturna, un cron job di classificazione o l'assegnazione automatica di tag a un archivio pubblico restano così gratuiti in modo permanente.

Perché resti gratuito, servono limiti, nei punti giusti. Un contatore di token nel proprio codice è una funzione di comodità; sono vincolanti solo i limiti applicati da OpenAI stessa.

## Il programma: token in cambio di dati di addestramento

La partecipazione avviene tramite l'impostazione **Share inputs and outputs with OpenAI** in *Settings → Data controls → Sharing*. Può modificarla solo l'Organization Owner, per l'intera organizzazione oppure per singoli progetti. Chi è idoneo al programma vede in questa pagina l'avviso "You're eligible for free daily usage on traffic shared with OpenAI"; dopo l'attivazione, questo cambia in "You're enrolled for complimentary daily tokens". Se l'avviso non è presente, l'organizzazione non è attualmente idonea alla partecipazione. Gli account con Zero Data Retention e i contratti Enterprise sono esclusi dalla condivisione di input e output.

La quota dipende dall'Usage Tier dell'organizzazione e viene calcolata per gruppo di modelli:

| Gruppo di modelli | Tier 1–2 | Tier 3–5 |
|---|---|---|
| Modelli grandi (tra cui gpt-5.6-sol, gpt-5.x, serie o, gpt-4.1, gpt-4o) | 250'000 token/giorno | 1 milione di token/giorno |
| Modelli piccoli (tra cui gpt-5.6-terra, gpt-5.6-luna, varianti mini e nano) | 2,5 milioni di token/giorno | 10 milioni di token/giorno |

Le regole più importanti nel dettaglio:

- Vengono conteggiati insieme i token di input e output, condivisi tra tutti i modelli di un gruppo. Il contatore viene azzerato ogni giorno alle 00:00 UTC.
- I modelli sottoposti a fine-tuning, l'addestramento di fine-tuning, le valutazioni e l'uso di tool sono esclusi.
- L'account deve avere un saldo positivo, altrimenti nemmeno i token gratuiti funzionano.
- OpenAI si riserva il diritto di terminare il programma con 30 giorni di preavviso.

La regola di fatturazione più importante: la richiesta che supera la quota giornaliera viene fatturata **interamente** alla tariffa standard, non soltanto per la parte eccedente. Chi si trova a 975'000 token su 1 milione e invia una richiesta da 30'000 token paga tutti i 30'000 token. Per la pianificazione del proprio budget significa: prevedere un margine di sicurezza, non ottimizzare fino al limite della quota.

## Cosa si cede in cambio

La contropartita è inequivocabile: tutti gli input e gli output dei progetti condivisi vanno a OpenAI e possono essere utilizzati per addestrare modelli futuri. Questo esclude intere categorie di casi d'uso. Dati dei clienti, ticket di supporto, documenti interni, codice con dettagli di configurazione e qualsiasi dato personale non devono finire in un progetto condiviso; per le aziende svizzere, la LPD rev. pone già qui il limite, prima ancora di affrontare la questione della riservatezza verso i clienti.

Sono adatti i carichi di lavoro basati su dati già pubblici. Un esempio è la traduzione notturna di un blog pubblico in più lingue: gli articoli sono online, ogni crawler può già leggerli oggi e anche le traduzioni vengono pubblicate. In un caso simile, la condivisione non rivela nulla che non sia già stato reso pubblico. Altri candidati sono testi alternativi per un archivio fotografico pubblico, l'assegnazione di tag alla documentazione open source o riassunti di note di rilascio pubbliche per un changelog.

## Configurare i guardrail sui costi nell'account OpenAI

L'ordine è intenzionale: prima vengono i limiti che OpenAI applica lato server. Intervengono anche se il proprio codice contiene un errore, un cron job viene eseguito due volte o una chiave finisce nelle mani sbagliate.

**Credito prepagato, Auto-Reload disattivato.** Impostare la fatturazione su "Pay as you go" con credito prepagato e disattivare la ricarica automatica. In questo modo il danno massimo è limitato al credito residuo: una volta esaurito, l'API rifiuta ulteriori richieste. Poiché il programma richiede un saldo positivo, serve una piccola base; bastano da 5 a 10 dollari e, con un funzionamento corretto, rimangono intatti. Questo è l'unico passaggio che, nel peggiore dei casi, ferma davvero tutto, perciò è al primo posto.

**Un progetto dedicato al traffico condiviso.** Impostare la condivisione su "Enabled for selected projects" e abilitare soltanto un progetto creato appositamente. Tutti gli altri progetti dell'organizzazione restano esclusi dall'addestramento e il traffico accidentale proveniente da altre applicazioni non finisce né nel set di dati di addestramento né nel budget sbagliato.

**Impostare un limite di spesa del progetto basso.** I progetti hanno un proprio limite mensile di spesa, ed è rigido: le richieste falliscono non appena viene raggiunto. Per un progetto che dovrebbe costare 0 dollari, può essere molto basso; 5 dollari sono sufficienti come riserva nel caso in cui una singola esecuzione superi la quota gratuita. Il limite a livello di organizzazione è invece pensato come tetto massimo con avvisi; le soglie di avvertimento, ad esempio al 90 e al 100 per cento, attivano e-mail.

**Una restricted key per progetto, solo come secret CI.** La chiave API viene creata nel progetto, non a livello di organizzazione, e riceve solo le autorizzazioni necessarie al carico di lavoro. Per un workflow CI significa: esattamente una chiave con diritti limitati, memorizzata come secret nell'ambiente CI. Non compare in alcun repository, shell locale o secondo servizio.

**Scegliere un modello del gruppo economico.** La differenza tra i gruppi è un fattore 10. Chi lavora nel Tier 1 dispone, con un modello del gruppo piccolo, di 2,5 milioni di token al giorno anziché 250'000. Per attività strutturate come traduzione, classificazione o estrazione, il gruppo piccolo è di norma sufficiente.

## La seconda linea di difesa nel codice

I limiti dell'account prevengono danni finanziari, ma causano errori rigidi: un limite di spesa raggiunto interrompe l'esecuzione a metà del batch. Chi vuole rimanere in modo affidabile entro la quota gratuita può quindi aggiungere un proprio conteggio. Si è dimostrato efficace un semplice contatore giornaliero, configurato ad esempio così:

```json
{
  "openai": {
    "model": "gpt-5.6-terra",
    "reasoningEffort": "none",
    "maxOutputTokens": 32000,
    "dailyTokenBudget": 1000000
  }
}
```

Il meccanismo si basa su quattro regole:

- Dopo ogni risposta, il job aggiunge a un contatore in un file di stato i valori `input_tokens` e `output_tokens` riportati dall'API. Non ci sono stime né una seconda richiesta, solo i dati di utilizzo della risposta stessa.
- Prima di ogni richiesta controlla il budget residuo. Se non è più sicuramente sufficiente per una risposta completa, l'esecuzione termina regolarmente con il motivo di arresto `token-budget` anziché con un errore.
- Il contatore lavora con giorni di calendario UTC ed è quindi sincronizzato con l'azzeramento della quota gratuita alle 00:00 UTC.
- Indipendentemente dal budget, il numero di chiamate API per esecuzione è limitato, affinché anche una serie di tentativi falliti non possa esaurire la quota. Gli errori di trasporto e di quota interrompono l'esecuzione senza ripetizione automatica.

Il budget di questo esempio, pari a 1 milione, è volutamente molto inferiore alla quota di 2,5 milioni. Il margine deriva da due particolarità della fatturazione. In primo luogo, il contatore non conosce in anticipo la dimensione della richiesta successiva; un budget calcolato al limite può quindi essere superato della dimensione di una richiesta, e proprio questa richiesta verrebbe fatturata interamente secondo la regola descritta sopra. In secondo luogo, i limiti di velocità giornalieri (TPD) sono, a seconda del tier e del modello, inferiori alla quota gratuita; un budget superiore al limite TPD non verrebbe mai raggiunto regolarmente, perché prima l'API rifiuterebbe con HTTP 429.

## Controllo: il dashboard deve mostrare 0.00

Se il calcolo funziona, lo mostra il dashboard Usage della piattaforma. Sono sufficienti due viste:

- La vista **Usage** conta tutti i token, compresi quelli fatturati gratuitamente. Qui appare il consumo totale del carico di lavoro.
- La vista **Costs** (e il campo "Monthly spend" nell'elenco dei progetti) mostra solo i token a pagamento. Qui deve risultare sempre 0.00.

Chi desidera maggiori dettagli può raggruppare la vista Usage per *Service tier*: i token fatturati gratuitamente vi compaiono come voce separata "data sharing incentive tier". Un promemoria in calendario impostato una volta al mese per dare un'occhiata al dashboard completa la catena di guardrail, poiché OpenAI può terminare il programma con un preavviso di 30 giorni e, da quel giorno, lo stesso carico di lavoro proseguirebbe alla tariffa standard.

## Fonti

1.  [OpenAI Help Center: Sharing feedback, evaluation and fine-tuning data, and API inputs and outputs](https://help.openai.com/en/articles/10306912-sharing-feedback-evaluation-and-fine-tuning-data-and-api-inputs-and-outputs-with-openai): descrizione autorevole del programma con gruppi di modelli, quote per tier, azzeramento UTC e regola di fatturazione per le richieste eccedenti.

2.  [OpenAI Developer Community: Extended: Free tokens on traffic shared with OpenAI](https://community.openai.com/t/good-news-extended-free-tokens-on-traffic-shared-with-openai/1241322): annuncio della proroga del programma nell'aprile 2025 con la garanzia del preavviso di 30 giorni.

3.  [OpenAI Platform: Data sharing settings](https://platform.openai.com/settings/organization/data-controls/sharing): interruttore di opt-in e stato di adesione della propria organizzazione (accesso richiesto).

4.  [OpenAI Platform: Rate limits guide](https://platform.openai.com/docs/guides/rate-limits): spiegazione dei limiti TPM, RPM e TPD, che si applicano oltre alla quota gratuita.

5.  [OpenAI Platform: Pricing](https://platform.openai.com/docs/pricing): tariffe standard alle quali vengono fatturati i superamenti della quota.
