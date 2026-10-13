---
title: "SMTP: trasporto, coda e responsabilità di consegna"
blatt: "smtp"
description: "Simple Mail Transfer Protocol dal punto di vista dell’amministratore: ruoli di MUA, MSA, MTA e MDA, envelope e contenuto del messaggio, macchina a stati SMTP, confine di accettazione, routing MX, coda e retry, codici di risposta, estensioni ESMTP, submission, policy TLS e diagnosi a livelli."
fakten:
  - label: Nome completo
    wert: Simple Mail Transfer Protocol
    href: https://datatracker.ietf.org/doc/html/rfc5321
  - label: Modello di base
    wert: Command/Reply e Store-and-Forward
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-2.3.1
  - label: Oggetto di trasporto
    wert: Envelope più contenuto del messaggio
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-2.3.1
  - label: Trasporto
    wert: flusso di dati affidabile e ordinato; su Internet TCP
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-1.1
  - label: Server a server
    wert: SMTP su TCP 25
    href: https://datatracker.ietf.org/doc/html/rfc6409#section-1
  - label: Submission
    wert: 587 con STARTTLS · 465 con TLS implicito
    href: https://datatracker.ietf.org/doc/html/rfc8314#section-3
  - label: Ruoli principali
    wert: MUA · MSA · MTA · MDA oppure LMTP
    href: https://datatracker.ietf.org/doc/html/rfc5598#section-4
  - label: Funzionalità
    wert: EHLO negozia le estensioni ESMTP per ogni sessione
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-2.2
  - label: Confine di accettazione
    wert: 250 dopo End-of-Data trasferisce la responsabilità
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-4.1.1.4
  - label: Classi di errore
    wert: 4xx temporaneo · 5xx permanente · x.y.z come dettaglio
    href: https://datatracker.ietf.org/doc/html/rfc3463#section-2
  - label: Dipendenze
    wert: DNS · TCP · TLS · SASL · MIME · coda persistente
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-2.3
  - label: Storia normativa
    wert: RFC 821 → RFC 2821 → RFC 5321
    href: https://datatracker.ietf.org/doc/html/rfc5321#appendix-C
werbung:
  - tools
  - newsletter
ctaThemen:
  - smtp-mailflow
translationSourceHash: bd51241025647a47c1c52f82747a0f7eee7d7b8e42d57f52057e7123d6bc3825
translationModel: gpt-5.6-terra
translatedAt: 2026-10-13T11:14:22.988Z
translationReview: required
---

# SMTP: trasporto, coda e responsabilità di consegna

Il **Simple Mail Transfer Protocol** (SMTP) porta un messaggio da un processo di posta al successivo. Il client si connette a un server, ne interroga le capacità, indica mittente e destinatari dell’envelope e solo dopo trasferisce il contenuto del messaggio. Il server risponde a ogni passaggio con un codice numerico. Da questa sequenza apparentemente semplice derivano routing, accodamento, ritentativi e la domanda decisiva su quale server detenga in quel momento la responsabilità del messaggio ([RFC 5321, sezioni 1 e 2](https://datatracker.ietf.org/doc/html/rfc5321#section-1)).

Per gli amministratori di messaggistica, la proprietà decisiva è il **trasferimento di responsabilità**. Una connessione TCP riuscita significa soltanto che un listener è raggiungibile. Un `RCPT TO` accettato riserva un destinatario nella transazione in corso. Solo una chiusura positiva dopo il contenuto completo del messaggio trasferisce al sistema SMTP ricevente la responsabilità di consegnare il messaggio oppure di segnalare in seguito un errore al mittente dell’envelope ([RFC 5321, sezioni 4.1.1.4 e 6.1](https://datatracker.ietf.org/doc/html/rfc5321#section-4.1.1.4)). ID della coda, codice di risposta, MTA remoto, stato del destinatario e traccia `Received` non sono quindi dettagli di log secondari, bensì la catena probatoria di una consegna.

La spiegazione segue un messaggio dal client che lo invia attraverso la sessione SMTP, l’envelope e la coda fino al server di posta successivo o alla casella di posta. DNS, retry, codici di stato, sicurezza e diagnosi vengono approfonditi nei punti in cui influenzano il percorso.

## Ruoli e responsabilità

SMTP conosce inizialmente soltanto **client** e **server**. In una catena di consegna reale, questi ruoli cambiano a ogni hop: un Mail Transfer Agent accetta un messaggio come server, lo memorizza nella propria coda e diventa esso stesso client al successivo tentativo di consegna. Solo da questo cambio di ruolo è possibile interpretare correttamente log e responsabilità ([RFC 5321, sezione 2.3.3](https://datatracker.ietf.org/doc/html/rfc5321#section-2.3.3), [RFC 5598, sezione 4](https://datatracker.ietf.org/doc/html/rfc5598#section-4), [RFC 6409, sezione 2.1](https://datatracker.ietf.org/doc/html/rfc6409#section-2.1)).

| Ruolo | Compito | Tipico confine di fiducia e stato |
|---|---|---|
| **MUA** – Message User Agent | crea un messaggio e visualizza i messaggi ricevuti | identità dell’utente, bozze locali, rubrica e visualizzazione; nessun servizio di relay pubblico |
| **MSA** – Message Submission Agent | accetta nuovi messaggi da utenti o applicazioni autorizzati | identità autenticata, autorizzazione del mittente, policy di submission e restituzione immediata degli errori |
| **MTA** – Message Transfer Agent | instrada e trasferisce messaggi accettati verso altri MTA | coda persistente, scheduler dei retry, scelta della destinazione DNS, policy TLS e responsabilità di consegna |
| **Gateway o relay** | converte tra zone di sicurezza, organizzazioni o sistemi di messaggistica | policy, riscrittura, ispezione del contenuto, conversione di protocollo e coda propria |
| **MDA** – Message Delivery Agent | trasferisce a mailbox, file, pipe o store proprietario | risoluzione finale del destinatario, quota e coerenza della mailbox |
| **Server LMTP** | accetta un messaggio dall’MTA per la consegna locale | segnala uno stato separato per ogni destinatario dopo il contenuto e non necessita di una coda relay a lungo termine |

L’MSA non è semplicemente un MTA su un’altra porta. Message Submission separa l’invio controllato di nuovi messaggi dal trasferimento aperto tra domini di posta. Un MSA può, tra l’altro, verificare i diritti del mittente, aggiungere campi obbligatori mancanti e adattare i messaggi alle regole locali; un MTA a valle non dovrebbe modificare arbitrariamente il contenuto di messaggi già completati ([RFC 6409, sezioni 1, 6 e 8](https://datatracker.ietf.org/doc/html/rfc6409#section-1)). Questo confine è particolarmente rilevante per le firme DKIM: qualsiasi modifica successiva a header firmati o al body può invalidare la firma.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-smtp-zustellpfad.svg?v=20260813" title="Interaktive Infografik: SMTP-Rollen, Sitzungsablauf, Envelope, Queue und Zustellverantwortung" loading="lazy">
  <a href="/images/kb-interaktiv-smtp-zustellpfad.svg?v=20260813">Aprire l’infografica sul percorso di consegna SMTP e sul confine di accettazione</a>
</iframe>

Il grafico mostra confini logici. MSA, MTA in uscita, gateway e MTA in entrata possono funzionare come servizi separati, coincidere su un unico sistema oppure essere nascosti da un fornitore cloud. Gli standard non prescrivono alcuna architettura di processo. Dal punto di vista amministrativo, tuttavia, la separazione resta necessaria, poiché ogni passaggio accettato può generare una nuova coda, un nuovo stato di retry e un nuovo responsabile.

## Envelope e contenuto del messaggio

Prima di esaminare il dialogo nel dettaglio, occorre separare due aree di dati: l’**envelope** per il trasporto e il **contenuto del messaggio** per destinatari e programmi di posta. L’envelope consiste in un Reverse Path da `MAIL FROM` e uno o più Forward Path da `RCPT TO`. Il contenuto segue dopo `DATA` e contiene una sezione di header secondo Internet Message Format e il body, spesso con struttura MIME ([RFC 5321, sezione 2.3.1](https://datatracker.ietf.org/doc/html/rfc5321#section-2.3.1), [RFC 5322, sezione 2.1](https://datatracker.ietf.org/doc/html/rfc5322#section-2.1)).

| Elemento | Oggetto di protocollo | Utilizzo effettivo |
|---|---|---|
| `MAIL FROM:<bounce@example.net>` | Envelope Reverse Path | destinazione per i rapporti di mancata consegna e input per SPF; può essere vuoto per DSN |
| `RCPT TO:<user@example.ch>` | Envelope Forward Path | routing effettivo e decisione sul destinatario; ripetibile per ogni destinatario |
| `From: Author <author@example.net>` | Header del messaggio | identità visibile dell’autore e dominio di riferimento DMARC; non è un comando di routing SMTP |
| `To:` e `Cc:` | Header del messaggio | indicazioni visibili della destinazione per persone e MUA; non devono corrispondere a tutti i destinatari dell’envelope |
| `Bcc:` | Informazione di creazione | i destinatari Bcc vengono consegnati tramite indirizzi dell’envelope, senza rivelare il loro indirizzo alle altre copie |
| `Message-ID:` | Header del messaggio | identificativo contenutistico del messaggio; non coincide con l’ID della coda o con l’ID dell’envelope DSN |
| `Return-Path:` | Header di traccia generato alla consegna finale | conserva l’ultimo Reverse Path utilizzato; non va adottato come riga mittente impostata liberamente da un messaggio in entrata |

I server SMTP non possono derivare il percorso finale da `To:`, `Cc:` o `From:`. Viceversa, gli indirizzi dell’envelope non devono comparire nell’header del messaggio. Per questo funzionano Bcc, mailing list, inoltri, SRS e indirizzi di bounce separati. La separazione spiega anche perché un messaggio nel client può apparire visivamente provenire da `ceo@example.ch`, mentre il mittente dell’envelope usa un altro dominio. SPF considera il percorso SMTP; DMARC stabilisce inoltre una relazione con il `From:` visibile. I dettagli di questi controlli sono trattati in [Autenticazione della posta](/kb/mail-auth) ([RFC 5321, appendice C](https://datatracker.ietf.org/doc/html/rfc5321#appendix-C), [RFC 5322, sezione 3.6](https://datatracker.ietf.org/doc/html/rfc5322#section-3.6)).

Un Reverse Path vuoto viene trasmesso come `MAIL FROM:<>`. Impedisce i cicli di rapporti d’errore ed è obbligatorio per le Delivery Status Notifications. Un server che rifiuta indiscriminatamente mittenti vuoti danneggia pertanto il percorso di risposta del sistema di posta ([RFC 5321, sezione 3.6.3](https://datatracker.ietf.org/doc/html/rfc5321#section-3.6.3)).

## Sessione e transazione SMTP

Su una [connessione TCP](/kb/tcp) viene eseguita una **sessione SMTP**; al suo interno possono seguire più messaggi come **transazioni di posta** separate. Il server inizia con un greeting `220`; il client si identifica con `EHLO` e riceve un elenco di capacità ESMTP relativo alla sessione. Una transazione è quindi composta da `MAIL`, almeno un `RCPT` accettato e `DATA` oppure un’alternativa negoziata come `BDAT` ([RFC 5321, sezioni 3.1 e 4.1](https://datatracker.ietf.org/doc/html/rfc5321#section-3.1)).

1. **Connessione e greeting:** il client si connette; il server risponde con `220`. Se ritarda o rifiuta già qui, non esiste ancora alcuna transazione di posta.
2. **Negoziazione delle capacità:** `EHLO client.example` porta entrambe le parti nello stato iniziale e fornisce estensioni quali `SIZE`, `STARTTLS`, `PIPELINING` o `SMTPUTF8`. Un client non può presumere che un altro nodo del cluster o una sessione successiva offra le stesse capacità.
3. **Upgrade TLS opzionale:** dopo un `STARTTLS` riuscito, lo stato SMTP e le capacità apprese in precedenza vengono scartati. Il client invia nuovamente `EHLO` nel canale protetto ([RFC 3207, sezione 4](https://datatracker.ietf.org/doc/html/rfc3207#section-4)).
4. **Mittente dell’envelope:** `MAIL FROM` apre una transazione e svuota i precedenti buffer di envelope e dati. Parametri ESMTP quali `SIZE`, `BODY` o `SMTPUTF8` sono associati a questo comando.
5. **Destinatari:** a ogni `RCPT TO` viene risposto singolarmente. Un server può accettare due destinatari e rifiutarne permanentemente un terzo; il client invia quindi il contenuto al sottoinsieme accettato.
6. **Contenuto:** dopo `DATA`, in caso di disponibilità segue `354`. Il client trasferisce header e body come righe terminate da CRLF, maschera i punti all’inizio delle righe e termina con una riga composta soltanto da `.`.
7. **Chiusura:** la risposta dopo questo marcatore End-of-Data decide se il server assume il messaggio nel suo insieme per i destinatari precedentemente accettati. `RSET` scarta la transazione in corso senza chiudere la sessione; `QUIT` la chiude ordinatamente.

SMTP è orientato al lockstep, ma `PIPELINING` di ESMTP può inviare gruppi di comandi consentiti senza attendere ogni singola risposta. Le risposte restano nell’ordine di ricezione e devono continuare a essere associate correttamente a ogni comando ([RFC 2920](https://datatracker.ietf.org/doc/html/rfc2920)). Una cattura di pacchetti con più righe `MAIL`-/`RCPT` immediatamente consecutive non costituisce quindi automaticamente una violazione del protocollo.

Il dialogo è così completo, ma la responsabilità della consegna non è ancora chiarita. Essa cambia solo con una risposta ben precisa del server ricevente.

## Confine di accettazione: 250 dopo End-of-Data

Il confine SMTP più importante si trova dopo il contenuto del messaggio. Un `250` a `MAIL FROM` accetta soltanto il Reverse Path. Un `250` a `RCPT TO` accetta un destinatario per la transazione in corso. Solo un `250` dopo il marcatore End-of-Data significa che il server ha assunto la responsabilità del messaggio. Un sistema corretto deve quindi consegnarlo oppure generare un rapporto di mancata consegna in caso di errore rilevato successivamente; il client mittente può rimuovere la propria copia dalla responsabilità attiva di consegna ([RFC 5321, sezioni 4.1.1.4 e 6.1](https://datatracker.ietf.org/doc/html/rfc5321#section-4.1.1.4)).

Questa semantica rende critica l’ultima parte del dialogo. Se la connessione si interrompe dopo il punto, ma prima di ricevere una risposta completa, il client non sa con certezza se il server ha memorizzato il messaggio. Deve poter ritentare la consegna; sono quindi possibili duplicati. SMTP non offre alcuna garanzia globale di exactly-once. I sistemi richiedono quindi un tracciamento idempotente tramite ID della coda, `Message-ID`, destinatario, ora, dimensione ed eventualmente un riferimento transazionale proprietario, senza equiparare tra loro questi identificativi.

Content filter, DLP, controllo malware o firma DKIM dovrebbero essere integrati nella pipeline di accettazione in modo che il server sappia, prima della risposta positiva finale, se può assumere una responsabilità persistente. Un proxy a monte che segnala prematuramente `250` e poi perde internamente il messaggio non può più restituire l’errore al client SMTP originario. Proprio qui un’architettura deve stabilire se l’accettazione venga confermata solo dopo una scrittura durevole, dopo la replica, dopo la scansione oppure già dopo il passaggio a un broker interno. Si tratta di una caratteristica del prodotto, non di una prescrizione dello standard SMTP, e deve essere comprovata con documentazione del produttore nonché con test di errore e ripristino.

Dopo l’accettazione, l’MTA deve decidere dove inviare il messaggio successivamente. Per la posta Internet, questa decisione inizia nel DNS.

## Routing tramite DNS e MX

Per la consegna diretta su Internet, l’MTA mittente determina la destinazione successiva dal dominio del destinatario dell’envelope. Interroga il [DNS](/kb/dns) per i record MX, ordina i candidati in base al valore di preference crescente e risolve i nomi Exchange selezionati in indirizzi A o AAAA. Più destinazioni con la stessa preference sono candidati equivalenti; un backup MX con valore più alto non è una replica della coda primaria, bensì un ricevente SMTP indipendente con policy e responsabilità di consegna proprie ([RFC 5321, sezione 5](https://datatracker.ietf.org/doc/html/rfc5321#section-5)).

Se manca un set di record MX, SMTP tratta storicamente il dominio come un MX implicito sul nome del dominio stesso e tenta i relativi indirizzi. Un **Null MX** esplicito con preference `0` e Exchange `.` dichiara invece che un dominio non accetta e-mail ([RFC 7505](https://datatracker.ietf.org/doc/html/rfc7505)). `NXDOMAIN`, `NODATA`, `SERVFAIL`, errori di validazione DNSSEC e un MX raggiungibile senza servizio SMTP funzionante sono stati diversi e devono restare distinguibili nel motivo della coda.

I percorsi di posta interni impiegano spesso next hop statici, smarthost, connettori o tabelle di routing invece della selezione MX pubblica. Il destinatario dell’envelope rimane comunque l’indirizzo di consegna; la policy di routing locale decide solo a quale sistema successivo affidarlo. In ogni diagnosi vanno quindi annotati tre nomi: dominio del destinatario, nome del next hop selezionato e indirizzo IP effettivamente connesso.

Anche quando l’MTA conosce la destinazione, la consegna non è sempre immediatamente possibile. Gli errori temporanei conducono quindi nella coda anziché alla perdita del messaggio.

## Coda, retry e DSN

Un MTA completo è un sistema store-and-forward. Se non può raggiungere subito l’hop successivo o se quest’ultimo risponde temporaneamente, l’MTA conserva messaggio ed envelope in una coda persistente, pianifica un nuovo tentativo e gestisce lo stato per destinatario. RFC 5321 richiede una strategia di retry, ma non prescrive un singolo algoritmo per tutti i prodotti. Raccomanda ritentativi ritardati, una notifica in caso di ritardo prolungato e una conservazione di più giorni prima dell’abbandono definitivo ([RFC 5321, sezione 4.5.4](https://datatracker.ietf.org/doc/html/rfc5321#section-4.5.4)). Valori del prodotto quali backoff, durata massima della coda, parallelismo e concorrenza di destinazione devono quindi essere letti dalla configurazione concreta dell’MTA.

Un record di coda sensato contiene almeno:

- contenuto del messaggio immutabile o un blob referenziato;
- mittente dell’envelope e stato di ogni destinatario dell’envelope;
- momento dell’accettazione, tentativo successivo e limite di scadenza;
- trasporto selezionato, next hop, ultimo indirizzo e decisione TLS/policy;
- ultimo codice SMTP e Enhanced Status Code con testo remoto;
- ID della coda locale e ID correlabili da hop a monte o a valle;
- stati per active, deferred, hold, delivered, bounced e quarantined;
- stato di replica, filesystem o database, quando l’accettazione deve essere ad alta disponibilità.

Gli errori temporanei vengono normalmente ritentati. Gli errori permanenti prima della propria accettazione vengono restituiti in modo sincrono al client SMTP mittente. Gli errori permanenti dopo la propria accettazione producono una **Delivery Status Notification** (DSN) verso il Reverse Path. RFC 3464 definisce a questo scopo un messaggio `multipart/report` leggibile dalle macchine con campi per messaggio e per destinatario quali `Reporting-MTA`, `Final-Recipient`, `Action`, `Status` e facoltativamente `Remote-MTA` o `Diagnostic-Code` ([RFC 3464](https://datatracker.ietf.org/doc/html/rfc3464)).

Un bounce non è automaticamente affidabile. Le DSN possono essere contraffatte, rivelare il contenuto originale o generare backscatter in caso di mittente dell’envelope impostato abusivamente. Un gateway dovrebbe rifiutare messaggi non validi possibilmente durante la sessione SMTP originaria, anziché accettarli e poi rimandarli a un indirizzo potenzialmente contraffatto.

## Codici di risposta e significato diagnostico

Le risposte SMTP dispongono di un codice base a tre cifre. L’intervallo della prima cifra descrive lo stato immediato; il testo è destinato alle persone e può essere specifico del prodotto ([RFC 5321, sezione 4.2](https://datatracker.ietf.org/doc/html/rfc5321#section-4.2)).

| Classe | Significato nel dialogo | Conseguenza per il processo mittente |
|---|---|---|
| `2xx` | completamento positivo | eseguire il passo successivo del protocollo; dopo End-of-Data registrare la responsabilità come trasferita |
| `3xx` | positivo intermedio | inviare altri dati, ad esempio il contenuto del messaggio dopo `354` |
| `4xx` | completamento negativo transitorio | mantenere il messaggio o il destinatario interessato nella coda e ritentare secondo la policy di retry |
| `5xx` | completamento negativo permanente | non ripetere senza modifiche; segnalare sincronicamente al client precedente oppure generare una DSN dopo precedente accettazione |

Gli **Enhanced Status Codes** integrano il formato `x.y.z`. `x` indica successo, temporaneità o permanenza; `y` indica l’area generale dell’errore e `z` il dettaglio. Così `5.1.1` appartiene alla classe degli indirizzi e indica una mailbox di destinazione inesistente, mentre `4.4.x` classifica problemi di rete e routing. I significati registrati sono riportati nel [registro IANA degli Enhanced Status Codes](https://www.iana.org/assignments/smtp-enhanced-status-codes/smtp-enhanced-status-codes.xhtml); un testo del produttore non deve essere valutato contro la semantica numerica ([RFC 3463](https://datatracker.ietf.org/doc/html/rfc3463), [RFC 5248](https://datatracker.ietf.org/doc/html/rfc5248)).

Lo stesso codice può riguardare aspetti diversi a seconda della fase del protocollo. `550` su `MAIL FROM` rifiuta il mittente o la sua autorizzazione, su `RCPT TO` un destinatario specifico e dopo il contenuto l’intera transazione. Un ticket senza host remoto, ora, fase, codici base, Enhanced Code e testo completo della risposta è quindi incompleto.

## ESMTP ed estensioni

Extended SMTP usa `EHLO` per negoziare le capacità per connessione. IANA gestisce il vincolante [registro delle SMTP Service Extensions](https://www.iana.org/assignments/smtp/smtp.xhtml). Un client può usare un’estensione solo se è stata offerta nella sessione corrente; dopo STARTTLS deve interrogare nuovamente l’elenco.

| Parola chiave EHLO | Effetto tecnico | Rilevanza amministrativa |
|---|---|---|
| `SIZE` | il server può indicare un limite di dimensione fisso; il client può dichiarare la dimensione prevista su `MAIL FROM` | evita trasferimenti di dati inutili, ma non garantisce ancora l’accettazione ([RFC 1870](https://datatracker.ietf.org/doc/html/rfc1870)) |
| `PIPELINING` | consente gruppi di comandi definiti senza round trip per comando | aumenta il throughput su percorsi con latenza; le risposte devono essere correlate esattamente nell’ordine ([RFC 2920](https://datatracker.ietf.org/doc/html/rfc2920)) |
| `8BITMIME` | consente il trasporto del body MIME a 8 bit con `BODY=8BITMIME` | non autorizza dati binari arbitrari e non modifica le regole degli header ([RFC 6152](https://datatracker.ietf.org/doc/html/rfc6152)) |
| `CHUNKING` | introduce `BDAT` con conteggi di byte espliciti come alternativa a `DATA` | nessuna dot transparency; prerequisito per `BINARYMIME` ([RFC 3030](https://datatracker.ietf.org/doc/html/rfc3030)) |
| `SMTPUTF8` | consente mailbox internazionalizzate e UTF-8 negli header definiti a tale scopo | ogni hop successivo richiesto deve supportare la capacità, altrimenti si verifica un errore di interoperabilità ([RFC 6531](https://datatracker.ietf.org/doc/html/rfc6531)) |
| `DSN` | trasporta parametri per le notifiche di consegna su `MAIL` e `RCPT` | differenzia rapporti di successo, ritardo ed errore desiderati ([RFC 3461](https://datatracker.ietf.org/doc/html/rfc3461)) |
| `STARTTLS` | passa dal dialogo in chiaro a un canale TLS | reimposta lo stato SMTP; per il trasferimento MTA la sicurezza dipende inoltre dalla policy ([RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207)) |
| `AUTH` | integra l’autenticazione SASL in SMTP ed elenca i meccanismi | previsto per la submission; i meccanismi offerti possono cambiare dopo STARTTLS ([RFC 4954](https://datatracker.ietf.org/doc/html/rfc4954)) |

Il capability drift tra nodi del cluster è un problema operativo frequente. Se solo alcuni listener offrono `SMTPUTF8`, lo stesso valore `SIZE` o meccanismi AUTH identici, si verificano errori di consegna apparentemente casuali. Il monitoraggio dovrebbe quindi verificare non solo un VIP, ma ogni percorso backend rilevante con lo stesso SNI, nome EHLO e rete sorgente.

Fin qui si è trattato del trasporto tra server. L’invio da parte di utenti e applicazioni è un servizio distinto, con altre porte, identità e rischi di abuso.

## Submission, relay e autenticazione

La porta **25** serve al trasferimento MTA-MTA. La porta **587** indica Message Submission; la porta **465** è registrata per la submission con TLS attivo fin dall’inizio come servizio `submissions`. La porta 465 non è una porta MX alternativa: la consegna MX pubblica non conosce indicazioni di porta e usa la porta 25 ([RFC 6409, sezioni 1 e 3](https://datatracker.ietf.org/doc/html/rfc6409#section-3.1), [RFC 8314, sezione 7.3](https://datatracker.ietf.org/doc/html/rfc8314#section-7.3), [IANA Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml)).

Un servizio di submission combina autenticazione e autorizzazione. `AUTH` usa SASL e può supportare meccanismi come PLAIN su TLS o bearer token basati su OAuth. L’identità autenticata, l’identità di autorizzazione richiesta e il mittente dell’envelope consentito sono grandezze distinte. Un login riuscito non autorizza automaticamente a usare qualsiasi dominio mittente o qualsiasi mittente dell’envelope ([RFC 4422, sezione 3.4](https://datatracker.ietf.org/doc/html/rfc4422#section-3.4), [RFC 4954](https://datatracker.ietf.org/doc/html/rfc4954), [RFC 7628](https://datatracker.ietf.org/doc/html/rfc7628)).

Un open relay si verifica quando un MTA inoltra messaggi da fonti non autorizzate verso destinazioni non locali. Una configurazione sicura decide pertanto separatamente su domini destinatari locali, reti sorgenti attendibili, submission autenticata e connettori espliciti. L’IP allowlisting è un’autorizzazione di rete e non sostituisce un’identità utente; con NAT, cloud egress o relay condivisi può abilitare una cerchia di mittenti molto più ampia del previsto.

Solo una volta stabiliti ruolo e porta è possibile determinare la policy TLS adatta: la submission deve essere protetta e autenticata, mentre il trasporto MTA aperto richiede policy di dominio aggiuntive.

## Policy TLS per SMTP

STARTTLS protegge la singola connessione quando entrambe le parti eseguono l’upgrade. Nel trasferimento MTA aperto, tuttavia, il solo STARTTLS resta opportunistico: un attaccante attivo può rimuovere l’offerta o manipolare le destinazioni DNS, e un mittente senza policy aggiuntiva può ricadere sul testo in chiaro. [DANE per SMTP](https://datatracker.ietf.org/doc/html/rfc7672) vincola i dati TLSA tramite DNSSEC alla selezione della destinazione MX. [MTA-STS](https://datatracker.ietf.org/doc/html/rfc8461) pubblica tramite DNS e HTTPS una policy MX e TLS basata su PKIX. [TLS-RPT](https://datatracker.ietf.org/doc/html/rfc8460) segnala errori aggregati per tali policy. L’articolo [TLS](/kb/tls) tratta dettagli crittografici, catene di certificati e procedure di test.

La submission è diversa: qui client e servizio sono amministrativamente associati e l’utente non deve ricadere silenziosamente sul testo in chiaro. RFC 8314 raccomanda TLS implicito per Mail Submission e Access e richiede che il testo in chiaro sia considerato obsoleto. SMTP AUTH con meccanismi a password o bearer richiede un canale adeguatamente protetto; un server non dovrebbe offrire affatto meccanismi deboli prima di STARTTLS ([RFC 8314, sezione 3](https://datatracker.ietf.org/doc/html/rfc8314#section-3), [RFC 4954, sezione 14](https://datatracker.ietf.org/doc/html/rfc4954#section-14)).

## Consegna locale con LMTP

Il **Local Mail Transfer Protocol** (LMTP) usa un dialogo simile a SMTP per il passaggio a un sistema senza una propria coda di posta a lungo termine, ad esempio un servizio mailbox. La differenza essenziale appare dopo il contenuto del messaggio: LMTP fornisce una risposta finale propria per ogni destinatario precedentemente accettato. In questo modo il mailstore può consegnare con successo a un destinatario e rifiutarne un altro per quota o errore locale, senza che l’MTA a monte debba trattare l’intera transazione come un’unità omogenea ([RFC 2033, sezione 4.2](https://datatracker.ietf.org/doc/html/rfc2033#section-4.2)).

LMTP non sposta la responsabilità fuori dal sistema. Se il mailstore rifiuta temporaneamente un destinatario, l’MTA a monte deve continuare a mantenere quel destinatario nella propria coda. Con socket Unix viene meno il percorso di rete IP, ma non la necessità di osservare autorizzazioni dei socket, backpressure, timeout, risposte per destinatario e confine di persistenza.

## Stack di implementazione e tecnologia

SMTP definisce il protocollo wire, non l’architettura software interna. Nella pratica, un MTA collega più livelli tecnici:

| Livello | Componenti tipici | Stato operativo critico |
|---|---|---|
| Selezione del servizio | resolver DNS, MX/A/AAAA, tabelle di routing statiche | cache, stato DNSSEC, selezione del next hop e IP connesso |
| Trasporto di rete | stack TCP, listener, load balancer, proxy | limiti di connessione, timeout, conservazione dell’IP sorgente e backpressure |
| Protezione del canale | libreria TLS, motore di certificati e policy | SNI, truststore, decisione DANE/MTA-STS e fallback |
| Frontend di protocollo | parser SMTP, macchina a stati, AUTH/SASL, rate limit | stato della sessione, capacità offerte e limiti contro gli abusi |
| Policy e contenuto | risoluzione indirizzi, anti-spam, malware, DLP, firma | ordine deterministico, timeout e comportamento in caso di guasto dello scanner |
| Persistenza | file di coda/spool, database o blob store | scrittura durevole prima di `250`, spazio libero, replica e recovery |
| Scheduler e routing | timer di retry, concorrenza di destinazione, tabelle di trasporto | equità, backoff, età della coda e domini bloccati |
| Adattatore di consegna | SMTP, LMTP, mailbox locale, file, pipe o API | stato per destinatario, idempotenza e confine di consegna |
| Osservabilità | log strutturati, ID della coda, metriche, trace e DSN | correlazione tra hop senza perdita di contenuti sensibili |

Le implementazioni realizzano questi livelli in modo diverso. [Postfix](https://www.postfix.org/OVERVIEW.html) suddivide ricezione, cleanup, Queue Manager e agenti di consegna in processi cooperanti e separa `incoming`, `active` e `deferred` come stati della coda. [Exim](https://www.exim.org/exim-html-current/doc/html/spec_html/ch-how_exim_receives_and_delivers_mail.html) memorizza envelope/header e body nello spool e lascia che i queue runner elaborino i tentativi di consegna dovuti. [Apache James](/kb/apache-james) gira sulla JVM e combina server di protocollo, coda e una pipeline Mailet con backend di archiviazione intercambiabili. Tutti e tre parlano SMTP, ma i loro modelli di processo, coda, estensioni e recovery non sono intercambiabili.

Linguaggio di programmazione e runtime influenzano packaging, processo di patch, profilo di memoria ed estensibilità, non la semantica del dialogo SMTP. Postfix viene compilato con una toolchain C nativa per sistemi di tipo Unix ([Postfix INSTALL](https://www.postfix.org/INSTALL.html)); la [documentazione Exim](https://www.exim.org/exim-html-2.00/doc/html/oview.html) indica ANSI C come linguaggio di implementazione. Apache James si descrive invece come piattaforma Java/Scala sulla JVM ([Apache James](https://james.apache.org/)). Un servizio cloud può rappresentare gli stessi ruoli RFC tramite microservizi proprietari, code distribuite e API. Per l’approvazione operativa devono pertanto essere documentati non solo le capacità EHLO, ma anche garanzia di persistenza, modello di retry, esportazione della coda, backup, gestione delle chiavi, limiti di scalabilità e domini di errore.

Dal processo di consegna deriva una procedura di troubleshooting fissa: determinare la destinazione, verificare connessione e dialogo SMTP, valutare la risposta e quindi seguire coda e next hop.

## Diagnosi dal DNS alla coda

Una diagnosi SMTP affidabile segue il percorso di consegna e mantiene separate le diverse osservazioni. I test devono essere eseguiti con domini di test e destinatari autorizzati; non bisogna abusare di server altrui per test di relay, convalida di indirizzi o carico.

### Determinare destinazione MX e indirizzi

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-DNS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) e [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) mostrano dati DNS pubblicati. L’output deve essere verificato per preference, Null MX, catene CNAME, IPv4/IPv6 e resolver effettivamente interrogato. Una risposta DNS non prova né la funzionalità del listener né quella TLS.

### Verificare la raggiungibilità TCP dell’hop selezionato

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-TCP-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) e [`nc`](https://man.openbsd.org/nc) verificano qui soltanto la risoluzione DNS e l’instaurazione TCP. Un successo sulla porta 25 non dice se il greeting SMTP arriva tempestivamente; un successo sulla porta 587 o 465 non dice nulla sulla policy TLS e AUTH.

### Verificare STARTTLS, EHLO e certificato

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-STARTTLS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) esegue con `-starttls smtp` il necessario dialogo in chiaro, imposta con `-name` il nome SMTP EHLO e con `-servername` SNI. `-verify_hostname` e `-verify_return_error` trasformano il mero handshake in un test esplicito del nome PKIX. Il risultato descrive OpenSSL con il proprio truststore, non automaticamente la policy dell’MTA produttivo.

### Testare in modo controllato envelope e fase di risposta

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für kontrollierten SMTP-Dialog">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`swaks`](https://jetmore.org/john/code/swaks/) registra comandi e risposte SMTP e può terminare intenzionalmente il test dopo `RCPT`, senza inviare dati del messaggio. Lo strumento non è un componente di sistema in Windows; `swaks.exe` o un’installazione Perl deve provenire da una fonte controllata. Un test RCPT deve essere effettuato solo verso sistemi e indirizzi propri o espressamente autorizzati, poiché attiva policy, directory e rate limit.

### Esaminare listener e pacchetti sul server

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-Listener- und Paketprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Get-NetTCPConnection`](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection) e [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) associano listener e stati TCP ai processi locali. [`pktmon`](https://learn.microsoft.com/windows-server/networking/technologies/pktmon/pktmon) e [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) acquisiscono pacchetti; le catture in produzione possono contenere indirizzi dell’envelope, contenuti non cifrati e dati di autenticazione e devono essere pertanto protette, limitate ed eliminate di conseguenza. [`Wireshark`](https://www.wireshark.org/docs/wsug_html/index.html) è adatto all’analisi di handshake TCP, SMTP e TLS, ma non sostituisce né i log di coda né quelli di policy.

### Assegnare l’errore a un confine

| Osservazione | Confine probabile | Verifica successiva affidabile |
|---|---|---|
| nessun MX o Null MX | pubblicazione DNS del dominio destinatario | risposta autorevole, stato DNSSEC, TTL della cache negativa |
| timeout TCP verso uno di più MX | percorso di rete o singolo MX | ogni indirizzo MX e famiglia IP separatamente, SYN/SYN-ACK e percorso di ritorno |
| connessione stabilita, nessun `220` | listener, proxy, tarpit o sovraccarico | latenza del greeting, limiti di connessione, log frontend e backend |
| `STARTTLS` manca solo su singole connessioni | capability drift, proxy o policy per rete sorgente | ogni nodo del cluster, EHLO prima/dopo TLS, percorso del load balancer |
| `4xx` con `MAIL FROM` | policy del mittente, rate limit o risorse | codice completo, IP sorgente, mittente dell’envelope, log di coda/policy |
| `5xx` con un `RCPT TO` | decisione sul destinatario o relay | destinatario dell’envelope interessato, Accepted Domains, edge della directory |
| `4xx` dopo End-of-Data | scanner, storage, coda o content policy | pipeline di accettazione locale, spazio spool, latenza scanner, nessuna presa in carico in coda |
| `250` dopo End-of-Data, DSN successiva | consegna a valle fallita dopo la propria accettazione | MTA segnalante, MTA remoto, ID della coda, stato DSN e destinatario originale |
| la coda cresce per molti domini | scheduler locale, DNS, rete o risorse | distribuzione dell’età, velocità di consegna, stati active/deferred, dipendenze globali |
| la coda cresce solo per un dominio | MX remoto, policy TLS o destination throttling | ultimo errore per MX/IP, DANE/MTA-STS, concorrenza e orario di retry |
| mailbox assente nonostante hop SMTP riuscito | MDA/LMTP o hop di routing interno | ultima riga `Received` attendibile, risposta LMTP per destinatario, log dello store |
| duplicati sporadici dopo timeout | risposta non chiara al confine di accettazione | momento dell’interruzione rispetto al `250` finale, correlazione di coda e Message-ID |

## Osservabilità e modello operativo

Un monitor SMTP non dovrebbe limitarsi a inviare un messaggio di test. Deve rendere visibili i singoli stati del sistema:

- latenze di greeting, EHLO, STARTTLS, AUTH e End-of-Data separatamente;
- connessioni, sessioni, transazioni attive e rifiuti per fase;
- capacità ESMTP offerte per listener e backend;
- tasso di accettazione, numero di destinatari, dimensione dei messaggi e latenza di scrittura durevole;
- volume della coda per età, destinazione, trasporto, stato e ultimo errore;
- tasso di retry, messaggio più vecchio, consegne in scadenza e generazione DSN;
- destinazione DNS, IP connesso, IPv4/IPv6, modalità TLS, cipher e risultato della policy;
- codici di risposta come campi strutturati anziché esclusivamente testo libero;
- correlazione di edge ID, ID della coda interna, ID della coda remota, `Message-ID` e destinatario;
- stato LMTP/mailbox per destinatario, nonché percorsi di quarantena e rilascio;
- capacità dello spool, latenza I/O, stato del database o broker e ritardo di replica;
- confini di protezione per log e catture, poiché envelope, header e contenuti contengono dati personali o riservati.

I backup della sola coda non sono sufficienti. Il ripristino deve considerare configurazione, tabelle di routing, certificati e chiavi, metadati della coda, blob dei messaggi, quarantena, stato delle firme e idempotenza. Dopo un restore, una copia del destinatario già consegnata non deve essere inviata nuovamente senza controllo; al contempo, i destinatari ancora aperti di un messaggio consegnato parzialmente non devono andare persi. RPO e RTO devono quindi essere definiti per ogni componente di stato e testati con una coda reale contenente più destinatari.

## Storia tecnica

SMTP è nato dalle precedenti procedure di posta ARPANET. Il [RFC 821](https://datatracker.ietf.org/doc/html/rfc821) di Jon Postel ha standardizzato nel 1982 il modello ancora oggi riconoscibile di sender SMTP, receiver SMTP, `MAIL`, `RCPT`, `DATA` e risposte a tre cifre. Il formato del messaggio venne descritto separatamente nello stesso anno in RFC 822. Questa separazione tra protocollo di trasferimento e formato del messaggio è rimasta.

Con MIME, SMTP è andato oltre il semplice testo a 7 bit. Il framework ESMTP introdusse `EHLO`, capacità registrate e parametri su `MAIL` e `RCPT`; la sua versione del 1995 era contenuta in [RFC 1869](https://datatracker.ietf.org/doc/html/rfc1869). Estensioni basate su di esso quali `SIZE`, `8BITMIME`, `PIPELINING`, `AUTH`, `STARTTLS`, `CHUNKING`, DSN e in seguito `SMTPUTF8` hanno evoluto il protocollo in modo incrementale, senza creare uno stack parallelo «SMTP 2».

[RFC 2821](https://datatracker.ietf.org/doc/html/rfc2821) ha consolidato SMTP nel 2001 e sostituito RFC 821 e diversi documenti complementari. [RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321) è seguito nel 2008 come ulteriore consolidamento; il relativo Internet Message Format è riportato in [RFC 5322](https://datatracker.ietf.org/doc/html/rfc5322). Message Submission fu inizialmente specificato separatamente e consolidato in RFC 6409. L’evoluzione tecnica è quindi avvenuta attraverso ruoli più chiari, registri di estensioni, indirizzi internazionalizzati, rapporti di consegna e policy di trasporto, mentre il dialogo di base con stato è rimasto compatibile.

Anche le architetture software si sono evolute in direzioni diverse. La compatibilità storica con Sendmail ha caratterizzato interfacce locali a riga di comando; Postfix ha puntato su processi cooperanti e separati per privilegi, nonché su code esplicite, Exim su un approccio altamente configurabile basato su router/transport e spool, e sistemi JVM quali Apache James hanno reso estensibile l’elaborazione mediante componenti e Mailet. Le piattaforme cloud distribuiscono gli stessi ruoli logici tra frontend, broker, servizi di policy e code regionali. Per gli amministratori, la semantica RFC resta quindi il denominatore comune, mentre persistenza, scalabilità e recovery devono sempre essere comprovati in modo specifico per il prodotto.

## Fonti

- [RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321) – modello SMTP, routing, comandi, risposte, coda, retry, traccia e responsabilità di consegna.
- [RFC 5598, sezione 4](https://datatracker.ietf.org/doc/html/rfc5598)
- [RFC 6409 – Message Submission for Mail](https://datatracker.ietf.org/doc/html/rfc6409) – separazione tra submission e relay, ruoli, porta 587 e policy MSA.
- [RFC 5322 – Internet Message Format](https://datatracker.ietf.org/doc/html/rfc5322) – header, body, campi degli indirizzi, Message-ID e campi di traccia.
- [RFC 3207 – SMTP Service Extension for Secure SMTP over TLS](https://datatracker.ietf.org/doc/html/rfc3207) – STARTTLS e reimpostazione dello stato SMTP.
- [RFC 2920 – SMTP Command Pipelining](https://datatracker.ietf.org/doc/html/rfc2920) – gruppi di comandi ordinati senza round trip per comando.
- [RFC 7505 – A Null MX No Service Resource Record](https://datatracker.ietf.org/doc/html/rfc7505) – indicazione esplicita di domini non abilitati alla posta.
- [RFC 3464 – Delivery Status Notification Format](https://datatracker.ietf.org/doc/html/rfc3464) – struttura DSN leggibile dalle macchine e stato del destinatario.
- [IANA – SMTP Enhanced Status Codes](https://www.iana.org/assignments/smtp-enhanced-status-codes/smtp-enhanced-status-codes.xhtml) – significati registrati dei codici di dettaglio.
- [RFC 3463 – Enhanced Mail System Status Codes](https://datatracker.ietf.org/doc/html/rfc3463) – struttura e classi dei codici di stato estesi.
- [RFC 5248 – SMTP Enhanced Status Code Registry](https://datatracker.ietf.org/doc/html/rfc5248) – modello di registrazione per Enhanced Codes.
- [IANA – Simple Mail Transfer Protocol Parameters](https://www.iana.org/assignments/smtp/smtp.xhtml) – comandi SMTP, estensioni e parole chiave EHLO.
- [RFC 1870 – SMTP SIZE Extension](https://datatracker.ietf.org/doc/html/rfc1870) – negoziazione della dimensione e rifiuto prima del contenuto.
- [RFC 6152 – SMTP Service Extension for 8-bit MIME Transport](https://datatracker.ietf.org/doc/html/rfc6152) – 8BITMIME e parametro BODY.
- [RFC 3030 – CHUNKING and BINARYMIME](https://datatracker.ietf.org/doc/html/rfc3030) – trasferimento BDAT e messaggi MIME binari.
- [RFC 6531 – SMTP Extension for Internationalized Email](https://datatracker.ietf.org/doc/html/rfc6531) – SMTPUTF8 e mailbox internazionali.
- [RFC 3461 – SMTP Service Extension for Delivery Status Notifications](https://datatracker.ietf.org/doc/html/rfc3461) – parametri envelope per richieste DSN.
- [RFC 4954 – SMTP Service Extension for Authentication](https://datatracker.ietf.org/doc/html/rfc4954) – comando AUTH e profilo SASL per SMTP.
- [RFC 8314, sezione 7.3](https://datatracker.ietf.org/doc/html/rfc8314)
- [IANA – Service Name and Transport Protocol Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) – assegnazione di SMTP, Submission e Submissions alle porte.
- [RFC 4422, sezione 3.4](https://datatracker.ietf.org/doc/html/rfc4422)
- [RFC 7628 – SASL Mechanisms for OAuth](https://datatracker.ietf.org/doc/html/rfc7628) – meccanismi SASL basati su OAuth per SMTP e altri protocolli.
- [RFC 7672 – SMTP Security via DANE TLS](https://datatracker.ietf.org/doc/html/rfc7672) – policy di trasporto SMTP vincolata a DNSSEC e TLSA.
- [RFC 8461 – SMTP MTA Strict Transport Security](https://datatracker.ietf.org/doc/html/rfc8461) – policy MX e TLS basata su PKIX.
- [RFC 8460 – SMTP TLS Reporting](https://datatracker.ietf.org/doc/html/rfc8460) – rapporti aggregati sugli errori TLS di trasporto.
- [RFC 2033, sezione 4.2](https://datatracker.ietf.org/doc/html/rfc2033)
- [Postfix – Architecture Overview](https://www.postfix.org/OVERVIEW.html) – ruoli dei processi, Queue Manager e agenti di consegna.
- [Exim – How Exim receives and delivers mail](https://www.exim.org/exim-html-current/doc/html/spec_html/ch-how_exim_receives_and_delivers_mail.html) – modello di spool, routing, trasporto e queue runner.
- [Postfix – Installation From Source Code](https://www.postfix.org/INSTALL.html) – requisiti per build nativa, piattaforma e toolchain.
- [Exim – panoramica tecnica](https://www.exim.org/exim-html-2.00/doc/html/oview.html) – ANSI C come linguaggio di implementazione e proprietà tecniche del prodotto.
- [Apache James Project](https://james.apache.org/) – architettura JVM, Java/Scala e componenti.
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) – query DNS in Windows.
- [BIND 9 – manuale dig](https://bind9.readthedocs.io/en/latest/manpages.html) – query DNS su sistemi Unix.
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) – raggiungibilità TCP in Windows.
- [OpenBSD – nc(1)](https://man.openbsd.org/nc) – connessioni TCP con Netcat.
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/) – diagnosi di STARTTLS, SNI e certificati.
- [Swaks – Swiss Army Knife for SMTP](https://jetmore.org/john/code/swaks/) – test di transazioni ed estensioni SMTP.
- [Microsoft Learn – Get-NetTCPConnection](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection) – listener e connessioni TCP locali.
- [Linux man-pages – ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html) – diagnosi di socket e stati TCP.
- [Microsoft Learn – Packet Monitor](https://learn.microsoft.com/windows-server/networking/technologies/pktmon/pktmon) – cattura pacchetti Windows e conversione PCAPNG.
- [tcpdump(1) manual](https://www.tcpdump.org/manpages/tcpdump.1.html) – diagnosi basata su pacchetti su sistemi Unix.
- [Wireshark User’s Guide](https://www.wireshark.org/docs/wsug_html/index.html) – analisi interattiva delle tracce di rete e protocollo.
- [RFC 821 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc821) – specifica SMTP storica del 1982.
- [RFC 1869 – SMTP Service Extensions](https://datatracker.ietf.org/doc/html/rfc1869) – framework ESMTP storico e modello EHLO.
- [RFC 2821 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc2821) – consolidamento SMTP del 2001.
- [RFC 3848 – ESMTP and LMTP Transmission Types](https://datatracker.ietf.org/doc/html/rfc3848) – valori `with` come ESMTPSA negli header Received.
