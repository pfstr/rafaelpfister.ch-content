---
title: "Cosa possiamo imparare dalle scienze naturali per la ricerca guasti nell'IT"
navTitle: "Esperimenti controllati"
description: "Falsificabilità, gruppo di controllo, variabili confondenti e distorsione del campione: il metodo con cui le scienze naturali lavorano da secoli risolve proprio i problemi sui quali la ricerca guasti IT fallisce regolarmente, illustrato con esempi del flusso di posta."
date: "2026-08-11"
kategorie: "SMTP / flusso di posta"
timeToRead: "15 min di lettura"
themen:
  - smtp-mailflow
  - testing
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
translationSourceHash: e3fff70bc1386c28d78713ec89a35b4d6c29b7f16e809e8a84bd9850a40a261c
translationModel: gpt-5.6-terra
translatedAt: 2026-08-31T10:16:22.067Z
translationReview: automatic
url: https://rafaelpfister.ch/it/blog/cosa-possiamo-imparare-dalle-scienze-naturali-per-la-ricerca-guasti-in-ambito-it
---

# Cosa possiamo imparare dalle scienze naturali per la ricerca guasti nell'IT

Un messaggio non arriva. Il protocollo fornisce un messaggio di errore che suggerisce subito una spiegazione. Verificate questa spiegazione, trovate delle prove e, dopo due ore, emerge che la spiegazione era sbagliata e le prove erano casuali.

Non è un errore da principianti, ma la regola. Ed è sorprendente che il nostro settore raramente disponga di un metodo per questo problema, benché ne esista uno da secoli e funzioni straordinariamente bene. Le scienze naturali hanno esattamente lo stesso compito: dedurre le cause dalle osservazioni, in sistemi che non si possono comprendere completamente.

Questo articolo applica cinque principi del metodo scientifico alla ricerca guasti nel flusso di posta. Gli esempi provengono dalla pratica, ma l'approccio non è specifico della posta elettronica.

## Perché la ricerca guasti IT è sistematicamente vulnerabile

Il flusso di posta è una catena di sistemi, ciascuno con una propria visione dello stesso messaggio: il gateway, il livello di filtraggio, il server di trasporto locale, il servizio cloud, la casella di posta di destinazione. Ogni messaggio è scritto dalla prospettiva di un solo livello.

Inoltre, i testi di errore sono termini generici. Spesso la stessa formulazione descrive situazioni completamente diverse, perché il sistema che rifiuta conosce solo una classificazione approssimativa. I codici di stato estesi sono concepiti proprio per creare classi, non per indicare singoli casi.

Un esempio: un servizio cloud ha rifiutato un messaggio indicando che il mittente non era autorizzato alla consegna in uscita. La stessa formulazione si è presentata nello stesso ambiente in due contesti completamente diversi. Una volta un sistema tentava di consegnare, tramite il servizio, a un destinatario esterno: un vero tentativo di inoltro verso l'esterno. L'altra volta il destinatario era una normale casella di posta del servizio e veniva contestato esclusivamente il dominio del mittente.

Chi prende il testo alla lettera cerca la stessa cosa in entrambi i casi. E poiché contiene la parola «uscita», si cerca prima dalla parte sbagliata.

## Principio 1: un'ipotesi deve escludere qualcosa

Karl Popper ha arricchito la filosofia della scienza con un'intuizione immediatamente pratica per la ricerca guasti: **un'affermazione è utile solo se è falsificabile.** Una spiegazione che si adatta a ogni possibile risultato dell'osservazione non spiega nulla.

Applicato al caso concreto, significa: formulate la vostra supposizione in modo che contenga una **previsione** che possa risultare falsa. Non «c'è qualcosa che non va con il dominio del mittente», bensì «se invio lo stesso messaggio con un altro dominio mittente lungo lo stesso percorso, arriverà».

La seconda formulazione vale qualcosa perché può essere confutata in cinque minuti. La prima può essere alimentata per ore con prove senza diventare mai più chiara.

Un buon test: prima del tentativo, chiedetevi quale risultato **confuterebbe** la vostra ipotesi. Se non ve ne viene in mente nessuno, non avete un'ipotesi, ma un'impressione.

## Principio 2: una variabile, tutto il resto uguale

Il nucleo dell'esperimento è il controllo delle variabili confondenti. Nella pratica accade regolarmente il contrario: si confrontano due casi disponibili per caso. E quasi sempre differiscono contemporaneamente in più caratteristiche.

Da un caso reale: i messaggi da `example-test.com` venivano rifiutati, quelli da `partner.example` arrivavano. I due domini differivano in almeno quattro caratteristiche: appartenenza all'organizzazione, dove è ospitata la posta, presenza di una politica di autenticazione rigorosa e percorso di invio. Da due punti dati con quattro differenze non si può dedurre esattamente nulla. Tutte e quattro le spiegazioni sono compatibili.

Costruite quindi voi stessi il confronto. Stesso punto di invio, stesso destinatario, stesso percorso, stesso momento e **una sola** caratteristica modificata. Se sospettate il dominio del mittente, cambiate solo quello.

## Principio 3: senza un test di controllo il risultato non vale nulla

Questa è la parte che si preferisce omettere, ed è la più importante. Nella ricerca clinica il gruppo di controllo è scontato; nell'IT di solito vi si rinuncia e poi ci si meraviglia dei risultati contraddittori.

**La vostra configurazione di test deve prima riprodurre l'errore.** Se non potete generare il caso di errore con i vostri mezzi, un tentativo contrario riuscito non dimostra nulla. Forse il messaggio di test funziona solo perché lo immettete in un altro punto rispetto al sistema originale, oppure perché un controllo non interviene affatto lungo il vostro percorso.

Un test utile consiste quindi di almeno due messaggi:

| | Scopo | Aspettativa |
|---|---|---|
| Tentativo 1 | Controllo, replica il caso originale | **deve fallire** |
| Tentativo 2 | Ipotesi, una variabile modificata | dovrebbe riuscire |

Se il tentativo 1 non fallisce, la configurazione non è rappresentativa. Allora non avete appreso nulla sul caso originale, ma solo sulla vostra configurazione di test, e dovete immettere il messaggio più vicino all'originale.

## Un esempio completo

Torniamo al caso precedente, anonimizzato. I messaggi di un sistema non raggiungevano i destinatari nel cloud, mentre altri messaggi agli stessi destinatari arrivavano senza problemi. Tre tentativi lungo lo stesso percorso, allo stesso destinatario, a pochi minuti di distanza:

| Tentativo | Dominio del mittente | Ipotesi verificata | Risultato |
|---|---|---|---|
| 1 (controllo) | `example-test.com` | La configurazione è rappresentativa | Rifiuto, identico all'originale |
| 2 | `example.com`, dominio proprio della destinazione | Dipende dal dominio del mittente | consegnato |
| 3 | `other-test.com`, dominio esterno della stessa organizzazione | Dipende dall'appartenenza all'organizzazione | consegnato |

Il tentativo 1 ha riprodotto l'errore, quindi la configurazione era valida. Il tentativo 2 ha mostrato che dipende dal dominio del mittente e non da destinatario, casella di posta, instradamento o autorizzazioni. Il tentativo 3 è stato quello davvero elegante: ha verificato in modo mirato la spiegazione alternativa più ovvia e **l'ha confutata**, poiché `other-test.com` apparteneva alla stessa organizzazione eppure è passato.

Tre messaggi, dieci minuti, e la causa era dimostrata anziché supposta. Prima, diverse ore erano confluite in tentativi di spiegazione, nessuno dei quali alla fine ha retto.

## Principio 4: confutare è il vero progresso

Un'ipotesi confutata sembra un passo indietro. In realtà è l'unica cosa che sapete con certezza. Le conferme sono deboli, poiché un'osservazione può adattarsi a più spiegazioni. Una confutazione rigorosa elimina un intero ramo dallo spazio di ricerca, in modo permanente.

È proprio qui che il bias di conferma agisce più intensamente. Se avete una supposizione, trovate quasi sempre qualcosa che vi si adatta. Nell'analisi descritta sopra c'era una correlazione tra il rifiuto e la questione di dove il dominio del mittente ospitasse la propria posta. Sembrava convincente, ma si basava su due punti dati che differivano in più caratteristiche. Il terzo tentativo l'ha smentita.

Annotate quindi le spiegazioni confutate insieme al motivo per cui sono state scartate. Non è altro che un quaderno di laboratorio. Ha due effetti: chi prenderà in carico il caso in seguito non percorrerà gli stessi vicoli ciechi. E voi stessi noterete quando state pensando in cerchio, perché un'idea già scartata torna sotto un altro nome.

Nella documentazione, i punti confutati devono figurare esplicitamente accanto a quelli dimostrati. Un rapporto che contiene solo la risposta corretta nasconde metà del lavoro e invita a ripeterlo.

## Principio 5: conoscete il vostro campione

La fonte di errore più sottile è la distorsione del campione, e nell'IT colpisce soprattutto le query che restituiscono risultati pagina per pagina.

Interrogate sette giorni di tracciamento dei messaggi, filtrate localmente per una caratteristica e non ottenete alcun risultato. La conclusione ovvia è che quel traffico non sia esistito. In realtà avete filtrato solo la prima pagina, che in presenza di un volume elevato copre pochi minuti.

Il risultato corretto è: non trovato nell'estratto. Non è: non esiste. La differenza è la stessa tra «nel nostro studio non è dimostrabile alcun effetto» e «non esiste alcun effetto».

Funzionano due vie d'uscita. Riducete la finestra temporale fino a quando una pagina la copre completamente, riconoscibile dall'assenza dell'avviso di ulteriori risultati. Oppure scorrete tutte le pagine e poi valutate i dati.

E una terza, spesso trascurata: per la domanda se qualcosa non si verifichi **mai**, un controllo della configurazione è superiore a qualsiasi osservazione. Se un sistema non possiede una route verso una destinazione, non può consegnarvi messaggi, indipendentemente da qualsiasi finestra di osservazione. È la differenza tra un argomento empirico e uno strutturale e, quando potete disporre di quello strutturale, sceglietelo.

## Il trasferimento: collegare l'onere della prova alla reversibilità

Qui finisce l'analogia con la scienza e subentra la prospettiva ingegneristica. La ricerca vuole la verità, l'esercizio vuole un impianto funzionante. Ne deriva un criterio che la scienza non conosce: **lo sforzo richiesto per la dimostrazione dipende dalla reversibilità dell'intervento.**

Disattivare un connettore è un comando, e anche annullarlo lo è. A tale scopo bastano indizi fondati, perché un errore viene corretto in un minuto ed è subito evidente. Eliminare lo stesso connettore non è reversibile; per questo vale la pena ottenere un'ulteriore prova tramite la configurazione della controparte o un rapporto di utilizzo lato server.

Lo stesso vale per le modifiche alle regole. Potete introdurre con una base fattuale limitata un livello di sola osservazione che registra e non reindirizza nulla. Non ha conseguenze e raccoglie proprio i dati che mancano per il passo decisivo. Solo il cambiamento che può trattenere i messaggi richiede prove solide.

Chi non applica questo criterio commette regolarmente entrambi gli errori nello stesso tempo: richiede settimane di prove per una modifica che si potrebbe annullare in pochi secondi e attiva senza garanzie qualcosa che può fermare il traffico di posta.

## Quando potete fermarvi

C'è un punto in cui continuare a scavare non aggiunge più valore: quando la correzione è chiara, ma il meccanismo resta incerto.

Nell'esempio precedente, dopo tre tentativi era dimostrato che il dominio del mittente era l'innesco, che tutto il resto del percorso di posta funzionava e che non esisteva un problema più ampio. Il motivo per cui il servizio cloud decidesse internamente proprio in quel modo rimase aperto. Per la correzione non aveva importanza, perché spettava all'applicazione mittente.

Separate quindi consapevolmente due domande. Cosa devo cambiare affinché funzioni? E perché il sistema si comporta così? Alla prima dovete rispondere, la seconda potete affidarla al produttore. Un caso di supporto con tre tentativi controllati, timestamp, identificativi dei messaggi e un controesempio funzionante è comunque di gran lunga più prezioso di una descrizione del sintomo.

Questo è anche il punto in cui scienza ed esercizio possono essere distinti chiaramente. La scienza non può abbandonare la domanda sul meccanismo. L'esercizio deve stabilirne la priorità.

## In breve

Formulate le ipotesi in modo che possano fallire e chiedetevi prima quale risultato le confuterebbe. Non confrontate mai due casi disponibili per caso, ma costruite il confronto con una sola variabile modificata. Riproducete l'errore nel test di controllo prima di credere al tentativo contrario. Considerate le confutazioni un progresso e annotatele per iscritto. A ogni query, verificate se vedete l'intero insieme o un campione. E adeguate il livello di prova richiesto alla facilità con cui l'intervento pianificato può essere annullato.

Le query concrete sono disponibili in [Analizzare il flusso di posta di Exchange: Message Tracking, protocolli SMTP e connettori di ricezione](https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren). Chi preferisce comporre i comandi con un'interfaccia anziché digitarli trova tutto nel [Generatore di comandi](https://rafaelpfister.ch/tools/command-builder).

## Fonti

1.  [Karl Popper: Logica della scoperta scientifica](https://www.mohrsiebeck.com/buch/logik-der-forschung-9783161584350): origine del principio di falsificazione, secondo cui un'affermazione è scientifica solo se rimane confutabile.

2.  [RFC 3463: Enhanced Mail System Status Codes](https://www.rfc-editor.org/rfc/rfc3463): spiega perché i codici di stato estesi sono deliberatamente classi approssimative e consentono lo stesso codice per cause diverse.

3.  [Message tracking in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): tipi di eventi e campi, base per determinare l'ultimo passaggio di elaborazione.

4.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2): logica di paginazione del tracciamento dei messaggi, che favorisce errori di campionamento.
