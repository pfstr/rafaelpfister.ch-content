---
title: "Cosa possiamo imparare dalle scienze naturali per la ricerca guasti in ambito IT"
navTitle: "Esperimenti controllati"
description: "Falsificabilità, gruppo di controllo, variabili di disturbo e distorsione del campione: il metodo con cui le scienze naturali lavorano da secoli risolve esattamente i problemi nei quali la ricerca guasti IT fallisce regolarmente. Con esempi dettagliati sul flusso di posta."
date: "2026-08-11"
kategorie: "SMTP / Flusso di posta"
timeToRead: "15 min di lettura"
themen:
  - smtp-mailflow
  - exchange-onprem-hybrid
hauptthema: "smtp-mailflow"
produkte:
  - "uebergreifend"
  - "exchange"
protokolle:
  - "smtp"
  - "troubleshooting"
related:
  - exchange-message-tracking-und-receive-connectoren-analysieren
  - einliefernde-ip-adressen-aggregieren
  - typische-ursachen-fuer-mail-loops-und-deren-behebung
slug: "cosa-possiamo-imparare-dalle-scienze-naturali-per-la-ricerca-guasti-in-ambito-it"
translationId: "article-098ed40e6d027b8b"
draft: false
translationOf: mailflow-fehlersuche-kontrollierte-experimente
url: https://rafaelpfister.ch/it/blog/cosa-possiamo-imparare-dalle-scienze-naturali-per-la-ricerca-guasti-in-ambito-it
translationSourceHash: d2466d0e63e5b08052fe7a47766ec2500b94c84097bfcfe91f8f6348cd6d1cc2
translationModel: gpt-5.6-terra
translatedAt: 2026-08-12T05:20:10.604Z
translationReview: automatic
---

# Cosa possiamo imparare dalle scienze naturali per la ricerca guasti in ambito IT

Un messaggio non arriva. Il protocollo fornisce un messaggio di errore che suggerisce subito una spiegazione. Verificate questa spiegazione, trovate delle prove e, dopo due ore, scoprite che la spiegazione era sbagliata e le prove erano casuali.

Non è un errore da principianti, bensì la regola. Ed è notevole che il nostro settore raramente disponga di un metodo per questo problema, nonostante ne esista uno da secoli e funzioni straordinariamente bene. Le scienze naturali hanno esattamente lo stesso compito: dedurre le cause dalle osservazioni, in sistemi che non si possono comprendere interamente.

Questo articolo trasferisce cinque principi del metodo scientifico alla ricerca guasti nel flusso di posta. Gli esempi provengono dalla pratica, ma il procedimento non è specifico delle email.

## Perché la ricerca guasti IT è sistematicamente vulnerabile

Il flusso di posta è una catena di sistemi, ciascuno dei quali ha una propria visione dello stesso messaggio: il gateway, il livello di filtraggio, il server di trasporto locale, il servizio cloud, la casella di posta di destinazione. Ogni messaggio è scritto dalla prospettiva di un solo livello.

Inoltre, i testi degli errori sono termini generici. La stessa formulazione descrive spesso situazioni completamente diverse, perché il sistema che rifiuta conosce solo una griglia approssimativa. I codici di stato estesi sono concepiti proprio per formare classi, non per identificare singoli casi.

Un esempio: un servizio cloud ha rifiutato un messaggio indicando che il mittente non era autorizzato per il recapito in uscita. La stessa formulazione si è presentata, nello stesso ambiente, in due configurazioni del tutto diverse. In un caso, un sistema cercava di recapitare tramite il servizio a un destinatario esterno, quindi un reale tentativo di inoltro verso l'esterno. Nell'altro, il destinatario era una normale casella di posta del servizio e veniva contestato esclusivamente il dominio del mittente.

Chi prende il testo alla lettera cerca la stessa cosa in entrambi i casi. E poiché contiene la parola «in uscita», si cerca prima dal lato sbagliato.

## Principio 1: un'ipotesi deve escludere qualcosa

Karl Popper ha arricchito la filosofia della scienza con un'intuizione immediatamente pratica per la ricerca guasti: **un'affermazione è utile solo se è confutabile.** Una spiegazione che si adatta a ogni possibile risultato dell'osservazione non spiega nulla.

In pratica, formulate la vostra supposizione in modo che contenga una **previsione** che possa risultare falsa. Non «c'è qualcosa che non va con il dominio del mittente», ma «se invio lo stesso messaggio con un altro dominio del mittente lungo lo stesso percorso, verrà recapitato».

La seconda formulazione ha valore perché può essere distrutta in cinque minuti. La prima può essere alimentata per ore con prove senza diventare mai più chiara.

Un buon test: prima della prova, chiedetevi quale risultato **confuterebbe** la vostra ipotesi. Se non ve ne viene in mente nessuno, non avete un'ipotesi, ma un'impressione.

## Principio 2: una variabile, tutto il resto uguale

Il nucleo dell'esperimento è il controllo delle variabili di disturbo. Nella pratica accade regolarmente il contrario: si confrontano due casi disponibili per caso. E quasi sempre differiscono contemporaneamente in più caratteristiche.

Da un caso reale: i messaggi provenienti da `example-test.com` venivano rifiutati, quelli provenienti da `partner.example` venivano recapitati. I due domini differivano in almeno quattro caratteristiche: appartenenza all'organizzazione, dove è ospitata la posta, se è configurata una policy di autenticazione rigorosa e il percorso di invio. Da due punti dati con quattro differenze non si può dedurre esattamente nulla. Tutte e quattro le spiegazioni sono compatibili.

Costruite quindi voi stessi il confronto. Stesso punto di invio, stesso destinatario, stesso percorso, stesso momento e **una sola** caratteristica modificata. Se sospettate il dominio del mittente, modificate solo quello.

## Principio 3: senza una prova di controllo, il risultato non vale nulla

Questa è la parte che si preferisce omettere, ed è la più importante. Nella ricerca clinica il gruppo di controllo è scontato; nell'IT di solito vi si rinuncia e ci si meraviglia poi dei risultati contraddittori.

**La configurazione del test deve prima riprodurre l'errore.** Se non riuscite a generare il caso di errore con i vostri mezzi, un tentativo alternativo riuscito non dice nulla. Forse il vostro messaggio di prova funziona solo perché lo immettete in un punto diverso dal sistema originale, oppure perché lungo il vostro percorso un controllo non viene nemmeno applicato.

Un test utile consiste quindi in almeno due messaggi:

| | Scopo | Aspettativa |
|---|---|---|
| Tentativo 1 | Controllo, replica il caso originale | **deve fallire** |
| Tentativo 2 | Ipotesi, una variabile modificata | dovrebbe riuscire |

Se il tentativo 1 non fallisce, la vostra configurazione non è rappresentativa. Allora non avete imparato nulla sul caso originale, ma solo sulla vostra configurazione di test, e dovete immettere il messaggio più vicino all'origine.

## Un esempio dettagliato

Torniamo al caso precedente, anonimizzato. I messaggi di un sistema non raggiungevano i destinatari nel cloud, mentre altri messaggi diretti agli stessi destinatari arrivavano senza problemi. Tre tentativi lungo lo stesso percorso, verso lo stesso destinatario, a distanza di pochi minuti:

| Tentativo | Dominio del mittente | Ipotesi verificata | Risultato |
|---|---|---|---|
| 1 (controllo) | `example-test.com` | La configurazione è rappresentativa | Rifiuto, identico all'originale |
| 2 | `example.com`, dominio proprio della destinazione | Il problema è il dominio del mittente | recapitato |
| 3 | `other-test.com`, dominio esterno della stessa organizzazione | Il problema è l'appartenenza all'organizzazione | recapitato |

Il tentativo 1 ha riprodotto l'errore, quindi la configurazione era valida. Il tentativo 2 ha mostrato che dipende dal dominio del mittente e non dal destinatario, dalla casella di posta, dal routing o dalle autorizzazioni. Il tentativo 3 è stato quello davvero elegante: ha verificato in modo mirato la spiegazione alternativa più ovvia e **l'ha confutata**, perché `other-test.com` apparteneva alla stessa organizzazione eppure è passato.

Tre messaggi, dieci minuti, e la causa era dimostrata anziché solo ipotizzata. Prima erano state spese diverse ore in tentativi di spiegazione, nessuno dei quali alla fine ha retto.

## Principio 4: confutare è il vero progresso

Un'ipotesi confutata dà l'impressione di un passo indietro. In realtà è l'unica cosa che sapete con certezza. Le conferme sono deboli, poiché un'osservazione può adattarsi a più spiegazioni. Una confutazione pulita elimina un intero ramo dallo spazio di ricerca, e lo fa in modo permanente.

È proprio qui che il bias di conferma agisce con maggiore forza. Se avete una supposizione, trovate quasi sempre qualcosa che le si adatta. Nell'analisi descritta sopra, vi era una correlazione tra il rifiuto e il luogo in cui il dominio del mittente ospita la propria posta. Sembrava convincente, ma si basava su due punti dati che differivano in più caratteristiche. Il terzo tentativo l'ha smontata.

Annotate quindi le spiegazioni confutate insieme al motivo per cui sono state scartate. Non è altro che un quaderno di laboratorio. Ha due effetti: chi riprende il caso in seguito non percorre gli stessi vicoli ciechi. E voi stessi vi accorgete quando pensate in cerchio, perché un'idea già scartata ritorna con un altro nome.

Nella documentazione, i punti confutati devono essere riportati esplicitamente accanto a quelli dimostrati. Un rapporto che contiene solo la risposta corretta nasconde metà del lavoro e invita a ripeterlo.

## Principio 5: conoscete il vostro campione

La fonte di errore più sottile è la distorsione del campione, e nell'IT colpisce soprattutto le query che restituiscono risultati pagina per pagina.

Interrogate sette giorni di tracciamento dei messaggi, filtrate localmente in base a una caratteristica e non ottenete risultati. La conclusione più ovvia è che quel traffico non sia esistito. In realtà avete filtrato solo la prima pagina, che con un volume elevato copre pochi minuti.

Il risultato corretto è: non trovato nell'estratto. Non è: non esiste. La differenza è la stessa tra «nel nostro studio non è dimostrabile alcun effetto» e «non esiste alcun effetto».

Funzionano due vie d'uscita. Riducete l'intervallo di tempo finché una pagina lo copre interamente, riconoscibile dall'assenza dell'indicazione di ulteriori risultati. Oppure sfogliate tutte le pagine e poi valutate.

E una terza, spesso trascurata: per la domanda se qualcosa non si verifichi **mai**, una verifica della configurazione è superiore a qualsiasi osservazione. Se un sistema non possiede una route verso una destinazione, non può recapitarvi messaggi, indipendentemente da qualsiasi finestra di osservazione. Questa è la differenza tra un argomento empirico e uno strutturale e, quando potete disporre di quello strutturale, sceglietelo.

## Il trasferimento: collegare l'onere della prova alla reversibilità

Qui termina l'analogia con la scienza e subentra la prospettiva ingegneristica. La ricerca vuole la verità, l'esercizio vuole un impianto funzionante. Ne deriva un criterio che la scienza non conosce: **lo sforzo richiesto per la dimostrazione dipende dalla reversibilità dell'intervento.**

Disattivare un connettore è un comando, e anche annullarlo lo è. A tal fine bastano indizi motivati, poiché un errore si corregge in un minuto e si nota immediatamente. Eliminare lo stesso connettore non è reversibile; per questo vale la pena fornire un'ulteriore prova tramite la configurazione della controparte o un rapporto di utilizzo lato server.

Lo stesso vale per le modifiche alle regole. Potete introdurre con una base fattuale limitata un livello di sola osservazione che registra e non reindirizza nulla. Non ha conseguenze e raccoglie proprio i dati che mancano per il passo decisivo. Solo la modifica che può trattenere i messaggi richiede prove solide.

Chi non applica questo criterio commette regolarmente entrambi gli errori contemporaneamente: richiede settimane di prove per una modifica che potrebbe annullare in pochi secondi e attiva senza protezioni qualcosa che può bloccare il traffico email.

## Quando potete smettere

C'è un punto in cui continuare a scavare non crea più valore: quando la correzione è chiara, ma il meccanismo resta oscuro.

Nell'esempio precedente, dopo tre tentativi era dimostrato che il dominio del mittente era l'elemento scatenante, che tutto il resto del percorso di posta funzionava e che non esisteva un problema più ampio. Il motivo per cui il servizio cloud decidesse internamente proprio in quel modo restava aperto. Per la correzione non aveva importanza, poiché spettava all'applicazione mittente.

Separate quindi consapevolmente due domande. Cosa devo modificare affinché funzioni? E perché il sistema si comporta così? Alla prima dovete rispondere, la seconda potete affidarla al produttore. Un caso di supporto con tre tentativi controllati, timestamp, identificativi dei messaggi e un controesempio funzionante è comunque di gran lunga più prezioso di una descrizione del sintomo.

Questo è anche il punto in cui scienza ed esercizio operativo possono essere distinti nettamente. La scienza non può abbandonare la domanda sul meccanismo. L'esercizio deve stabilirne le priorità.

## In breve

Formulate le ipotesi in modo che possano fallire e chiedetevi prima quale risultato le confuterebbe. Non confrontate mai due casi disponibili per caso, ma costruite il confronto con una sola variabile modificata. Riproducete l'errore nel tentativo di controllo prima di credere al tentativo alternativo. Considerate le confutazioni come progresso e documentatele per iscritto. Per ogni query, verificate se state vedendo l'intero insieme o un campione. E adeguate il livello di prova richiesto alla facilità con cui l'intervento previsto può essere annullato.

Le query concrete sono disponibili in [Analizzare il flusso di posta di Exchange: Message Tracking, protocolli SMTP e connettori di ricezione](https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren). Chi preferisce selezionare i comandi invece di digitarli può trovarli nel [Generatore di comandi](https://rafaelpfister.ch/tools/command-builder).

## Fonti

1.  [Karl Popper: Logica della ricerca](https://www.mohrsiebeck.com/buch/logik-der-forschung-9783161584350): origine del principio di falsificazione, secondo il quale un'affermazione è scientifica solo se resta confutabile.

2.  [RFC 3463: Enhanced Mail System Status Codes](https://www.rfc-editor.org/rfc/rfc3463): spiega perché i codici di stato estesi sono intenzionalmente classi approssimative e consentono lo stesso codice per cause diverse.

3.  [Message tracking in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): tipi di eventi e campi, base per determinare l'ultimo passaggio di elaborazione.

4.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2): logica di paginazione del tracciamento dei messaggi, che favorisce gli errori di campionamento.
