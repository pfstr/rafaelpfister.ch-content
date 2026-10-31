---
title: "Cosa possiamo imparare dalle scienze naturali per la risoluzione dei problemi IT"
navTitle: "Esperimenti controllati"
description: "Falsificabilità, gruppo di controllo, variabili confondenti e distorsione del campione: il metodo con cui le scienze naturali lavorano da secoli risolve proprio i problemi in cui la risoluzione dei problemi IT fallisce regolarmente, illustrato con esempi dal flusso di posta."
date: "2026-08-11"
kategorie: "SMTP / flusso di posta"
timeToRead: "15 min di lettura"
themen:
  - smtp-mailflow
  - testing
  - exchange-onprem-hybrid
hauptthema: "smtp-mailflow"
produkte:
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
translationSourceHash: 1759c709160723c161883fede66d48cd968bb6a587cb49c73781b3a39498cd66
translationModel: gpt-5.6-terra
translatedAt: 2026-10-30T03:29:52.721Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/cosa-possiamo-imparare-dalle-scienze-naturali-per-la-ricerca-guasti-in-ambito-it
---

# Cosa possiamo imparare dalle scienze naturali per la risoluzione dei problemi IT

Un messaggio non arriva. Il protocollo restituisce un messaggio di errore che suggerisce subito una spiegazione. Verificate questa spiegazione, trovate degli indizi e, dopo due ore, scoprite che la spiegazione era sbagliata e gli indizi erano casuali.

Non è un errore da principianti, ma la regola. Ed è notevole che il nostro settore raramente disponga di un metodo per questo problema, sebbene ne esista uno da secoli e funzioni estremamente bene. Le scienze naturali hanno esattamente lo stesso compito: dedurre le cause dalle osservazioni, in sistemi che non si possono comprendere completamente.

Questo articolo trasferisce cinque principi del metodo scientifico alla risoluzione dei problemi nel flusso di posta. Gli esempi provengono dalla pratica, ma il procedimento non è specifico della posta.

## Perché la risoluzione dei problemi IT è sistematicamente vulnerabile

Il flusso di posta è una catena di sistemi, ciascuno con una propria visione dello stesso messaggio: il gateway, il livello di filtraggio, il server di trasporto locale, il servizio cloud, la casella di posta di destinazione. Ogni messaggio è scritto dalla prospettiva di un solo livello.

Inoltre, i testi degli errori sono termini collettivi. La stessa formulazione descrive spesso situazioni completamente diverse, perché il sistema che rifiuta conosce soltanto una griglia approssimativa. I codici di stato estesi sono stati creati proprio per formare classi, non per identificare singoli casi.

Un esempio: un servizio cloud ha rifiutato un messaggio indicando che il mittente non era autorizzato alla consegna in uscita. La stessa formulazione si è verificata nello stesso ambiente in due configurazioni completamente diverse. In un caso, un sistema tentava di consegnare tramite il servizio a un destinatario esterno, quindi un vero tentativo di inoltro verso l'esterno. Nell'altro caso, il destinatario era una normale casella di posta del servizio e veniva contestato esclusivamente il dominio del mittente.

Chi interpreta il testo alla lettera cerca la stessa cosa in entrambi i casi. E poiché contiene la parola «uscita», si cerca prima dalla parte sbagliata.

## Principio 1: un'ipotesi deve escludere qualcosa

Karl Popper ha arricchito la filosofia della scienza con un'intuizione immediatamente pratica per la risoluzione dei problemi: **un'affermazione è utile solo se è falsificabile.** Una spiegazione che si adatta a qualsiasi possibile risultato dell'osservazione non spiega nulla.

In pratica: formulate la vostra supposizione in modo che contenga una **previsione** che possa rivelarsi falsa. Non «c'è qualcosa che non va con il dominio del mittente», ma «se invio lo stesso messaggio con un altro dominio mittente attraverso lo stesso percorso, verrà consegnato».

La seconda formulazione vale qualcosa perché può essere confutata in cinque minuti. La prima può essere alimentata per ore con indizi senza diventare mai più chiara.

Un buon test: prima della prova, chiedetevi quale risultato **confuterebbe** la vostra ipotesi. Se non ve ne viene in mente nessuno, non avete un'ipotesi, ma un'impressione.

## Principio 2: una variabile, altrimenti tutto uguale

Il nucleo dell'esperimento è il controllo delle variabili confondenti. Nella pratica accade regolarmente il contrario: si confrontano due casi disponibili per caso. E quasi sempre differiscono contemporaneamente in diverse caratteristiche.

Da un caso reale: i messaggi provenienti da `example-test.com` venivano rifiutati, quelli provenienti da `partner.example` venivano consegnati. I due domini differivano in almeno quattro caratteristiche: appartenenza all'organizzazione, dove è ospitata la posta, presenza di una rigorosa politica di autenticazione e percorso di inoltro. Da due punti dati con quattro differenze non si può dedurre esattamente nulla. Tutte e quattro le spiegazioni sono compatibili.

Costruite quindi voi stessi il confronto. Stesso punto di invio, stesso destinatario, stesso percorso, stesso momento e **una sola** caratteristica modificata. Se sospettate il dominio del mittente, modificate solo quello.

## Principio 3: senza prova di controllo il risultato non vale nulla

Questa è la parte che si preferisce omettere, ed è la più importante. Nella ricerca clinica il gruppo di controllo è ovvio; nell'IT di solito vi si rinuncia e poi ci si stupisce dei risultati contraddittori.

**La vostra configurazione di test deve prima riprodurre l'errore.** Se non riuscite a generare il caso di errore con i vostri mezzi, una prova contraria riuscita non dice nulla. Forse il vostro messaggio di test funziona solo perché lo immettete in un punto diverso dal sistema originale, oppure perché sul vostro percorso un controllo non viene affatto applicato.

Un test utilizzabile consiste quindi di almeno due messaggi:

| | Scopo | Aspettativa |
|---|---|---|
| Prova 1 | Controllo, replica il caso originale | **deve fallire** |
| Prova 2 | Ipotesi, una variabile modificata | dovrebbe riuscire |

Se la prova 1 non fallisce, la vostra configurazione non è rappresentativa. Allora non avete imparato nulla sul caso originale, ma solo sulla vostra configurazione di test, e dovete inviare più vicino all'originale.

## Un esempio completo

Torniamo al caso sopra, anonimizzato. I messaggi di un sistema non raggiungevano i destinatari nel cloud, mentre altri messaggi agli stessi destinatari arrivavano senza problemi. Tre prove attraverso lo stesso percorso, allo stesso destinatario, a pochi minuti di distanza:

| Prova | Dominio del mittente | Ipotesi verificata | Risultato |
|---|---|---|---|
| 1 (controllo) | `example-test.com` | La configurazione è rappresentativa | Rifiuto, identico all'originale |
| 2 | `example.com`, dominio proprio della destinazione | Il problema è il dominio del mittente | consegnato |
| 3 | `other-test.com`, dominio esterno della stessa organizzazione | Il problema è l'appartenenza all'organizzazione | consegnato |

La prova 1 ha riprodotto l'errore, quindi la configurazione era valida. La prova 2 ha mostrato che dipende dal dominio del mittente e non dal destinatario, dalla casella di posta, dal routing o dalle autorizzazioni. La prova 3 è stata quella davvero elegante: ha verificato miratamente la spiegazione alternativa più ovvia e l'ha **confutata**, perché `other-test.com` apparteneva alla stessa organizzazione eppure è passato.

Tre messaggi, dieci minuti, e la causa era dimostrata anziché supposta. Prima, diverse ore erano state dedicate a tentativi di spiegazione, nessuno dei quali ha retto alla fine.

## Principio 4: confutare è il vero progresso

Un'ipotesi confutata sembra un passo indietro. In realtà è l'unica cosa che sapete con certezza. Le conferme sono deboli, perché un'osservazione può adattarsi a più spiegazioni. Una confutazione pulita elimina un intero ramo dallo spazio di ricerca, e lo fa definitivamente.

È proprio qui che il bias di conferma agisce più fortemente. Quando avete una supposizione, trovate quasi sempre qualcosa che vi si adatta. Nell'analisi descritta sopra c'era una correlazione tra il rifiuto e il luogo in cui il dominio del mittente ospitava la propria posta. Sembrava convincente, ma si basava su due punti dati che differivano in più caratteristiche. La terza prova l'ha indebolita.

Annotate quindi le spiegazioni confutate insieme al motivo per cui sono state scartate. Non è altro che un quaderno di laboratorio. Ha due effetti: chi prende in carico il caso in seguito non percorre gli stessi vicoli ciechi. E voi stessi vi accorgete quando pensate in cerchio, perché un'idea già scartata ritorna con un nuovo nome.

Nella documentazione, i punti confutati devono essere esplicitamente accanto a quelli dimostrati. Un rapporto che contiene solo la risposta corretta nasconde metà del lavoro e invita a ripeterla.

## Principio 5: conoscete il vostro campione

La fonte di errore più sottile è la distorsione del campione, e nell'IT colpisce soprattutto le query che restituiscono risultati a pagine.

Interrogate sette giorni di tracciamento dei messaggi, filtrate localmente per una caratteristica e non ottenete alcun risultato. La conclusione ovvia è che quel traffico non sia esistito. In realtà avete filtrato solo la prima pagina, che con un volume elevato copre pochi minuti.

Il risultato corretto è: non trovato nell'estratto. Non è: non esiste. La differenza è la stessa tra «nel nostro studio non è dimostrabile alcun effetto» e «non esiste alcun effetto».

Due vie d'uscita funzionano. Riducete la finestra temporale fino a quando una pagina la copre completamente, riconoscibile dall'assenza dell'avviso di ulteriori risultati. Oppure sfogliate tutte le pagine e poi valutate.

E una terza, spesso trascurata: per la domanda se qualcosa non si verifichi **mai**, una verifica della configurazione è superiore a qualsiasi osservazione. Se un sistema non possiede alcuna route verso una destinazione, non può consegnarvi, indipendentemente da qualsiasi finestra di osservazione. Questa è la differenza tra un argomento empirico e uno strutturale; dove potete usare quello strutturale, usatelo.

## Il trasferimento: legare l'onere della prova alla reversibilità

Qui termina l'analogia con la scienza e subentra la prospettiva ingegneristica. La ricerca vuole la verità, l'esercizio vuole un impianto funzionante. Da ciò deriva un criterio che la scienza non conosce: **l'impegno richiesto per la dimostrazione dipende dalla reversibilità dell'intervento.**

Disattivare un connettore è un comando, e anche ripristinarlo lo è. A questo scopo bastano indizi motivati, perché un errore si corregge in un minuto e si nota subito. Eliminare lo stesso connettore non è reversibile; per questo vale la pena fornire l'ulteriore prova tramite la configurazione della controparte o un rapporto di utilizzo lato server.

Lo stesso vale per le modifiche alle regole. Potete introdurre con una base fattuale ridotta una fase di sola osservazione, che registra e non reindirizza nulla. Non ha conseguenze e raccoglie esattamente i dati che mancano per il passo decisivo. Solo la modifica che può trattenere i messaggi richiede prove solide.

Chi non applica questo criterio commette regolarmente entrambi gli errori contemporaneamente: richiede settimane di prove per una modifica che potrebbe annullare in pochi secondi e attiva senza garanzie qualcosa che può fermare il traffico di posta.

## Quando potete fermarvi

C'è un punto in cui scavare ulteriormente non crea più valore: quando la soluzione è chiara, ma il meccanismo rimane poco chiaro.

Nell'esempio sopra, dopo tre prove era dimostrato che il dominio del mittente era il fattore scatenante, che tutto il resto del percorso di posta funzionava e che non c'era un problema più ampio. Il motivo per cui il servizio cloud prende internamente quella decisione precisa è rimasto aperto. Per la correzione era irrilevante, perché spettava all'applicazione mittente.

Separate quindi consapevolmente due domande. Cosa devo modificare perché funzioni? E perché il sistema si comporta così? Alla prima dovete rispondere; la seconda potete rivolgerla al produttore. Un caso di supporto con tre prove controllate, timestamp, identificatori dei messaggi e un controesempio funzionante è comunque di gran lunga più prezioso di una descrizione del sintomo.

Questo è anche il punto in cui scienza ed esercizio possono essere distinti chiaramente. La scienza non può abbandonare la domanda sul meccanismo. L'esercizio deve stabilirne le priorità.

## La versione breve

Formulate le ipotesi in modo che possano fallire e chiedetevi prima quale risultato le confuterebbe. Non confrontate mai due casi disponibili per caso, ma costruite il confronto con una sola variabile modificata. Riproducete l'errore nella prova di controllo prima di credere alla prova contraria. Trattate le confutazioni come progressi e registratele per iscritto. Per ogni query, verificate se state vedendo l'intera popolazione o un campione. E adeguate il livello di prova richiesto alla facilità con cui l'intervento pianificato può essere annullato.

Le query concrete sono disponibili in [Analizzare il flusso di posta di Exchange: Message Tracking, protocolli SMTP e connettori di ricezione](https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren). Chi preferisce comporre i comandi con clic anziché digitarli trova tutto nel [Generatore di comandi](https://rafaelpfister.ch/tools/command-builder).

## Cronologia delle modifiche

- **25 ottobre 2026:** aggiornati i link alle fonti con i nuovi indirizzi di Mohr Siebeck e Microsoft Learn.

## Fonti

1.  [Karl Popper: Logica della ricerca](https://www.mohrsiebeck.com/buch/logik-der-forschung-9783161484100/): origine del principio di falsificazione, secondo cui un'affermazione è scientifica solo se resta confutabile.

2.  [RFC 3463: Enhanced Mail System Status Codes](https://www.rfc-editor.org/rfc/rfc3463): spiega perché i codici di stato estesi sono deliberatamente classi generiche e consentono lo stesso codice per cause diverse.

3.  [Message tracking in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): tipi di evento e campi, base per determinare l'ultimo passaggio di elaborazione.

4.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2): logica di paginazione del tracciamento dei messaggi, che favorisce gli errori di campionamento.
