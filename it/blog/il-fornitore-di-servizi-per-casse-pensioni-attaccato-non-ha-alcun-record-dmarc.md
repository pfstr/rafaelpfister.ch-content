---
title: "Il fornitore di servizi per casse pensioni colpito non ha alcun record DMARC"
navTitle: "DMARC casse pensioni"
description: "Il fornitore di servizi per casse pensioni PK Softech, colpito a fine settembre 2026, non dispone tuttora di un record DMARC. Un controllo passivo di 66 istituti di previdenza e 30 fornitori di servizi mostra che non si tratta di un caso isolato: in particolare i produttori di software gestionale sono privi di protezione."
date: "2026-10-18"
kategorie: "SMTP e flusso di posta"
timeToRead: "9 min di lettura"
themen:
  - smtp-mailflow
produkte: []
protokolle:
  - "mail-auth"
  - "dns"
slug: "il-fornitore-di-servizi-per-casse-pensioni-attaccato-non-ha-alcun-record-dmarc"
translationId: "article-a34017b39fecf271"
zielgruppe: "entscheider"
translationOf: dmarc-pensionskassen-schweiz
translationSourceHash: 57506056bfd8dcc96818df43dc71d22a158978bcd7432c91c56eb92698e52fe1
translationModel: gpt-5.6-terra
translatedAt: 2026-10-20T08:42:17.842Z
translationReview: automatic
url: https://rafaelpfister.ch/it/blog/il-fornitore-di-servizi-per-casse-pensioni-attaccato-non-ha-alcun-record-dmarc
---

A fine settembre 2026 è stata colpita PK Softech AG, un produttore di software la cui soluzione previdenziale è utilizzata da diverse casse pensioni svizzere. Attraverso questa via sono fuoriusciti dati. Secondo le notizie, sono interessate tra l'altro la cassa pensioni della Confederazione Publica, la cassa pensioni Migros e la cassa pensioni della Città di Zurigo, oltre ad altri enti. A seconda della cassa, si tratta di nome, numero AVS, stato civile, indirizzo e IBAN, nonché di dati salariali e previdenziali.

Questa fuoriuscita è già di per sé grave. Riguarda dati identificativi e finanziari di una grande parte della popolazione attiva e, tramite Publica, di ampie parti dell'amministrazione federale. Dati di tale quantità e qualità non costituiscono solo un problema per la singola persona. Consentono furti d'identità, inganni mirati ed estorsioni su larga scala e rappresentano quindi un pericolo per l'ordine pubblico e la sicurezza nazionale. Gli averi previdenziali stessi non sono stati toccati, ma la piattaforma informativa Cybercrimepolice.ch aveva già messo in guardia da chiamate fraudolente con i dati sottratti.

A ciò si aggiunge una constatazione difficile da spiegare per un fornitore di servizi di questa categoria: **PK Softech AG non ha tuttora alcun record DMARC.** Il suo dominio dispone di un record SPF rigoroso, ma di nessuna policy DMARC (verificato il 18.10.2026). Di conseguenza, è possibile falsificare l'indirizzo del mittente del fornitore di servizi colpito senza che una piattaforma di posta ricevente lo rifiuti in modo affidabile. Dopo una fuoriuscita di dati, questa è la situazione iniziale più sfavorevole immaginabile. Chi conosce nome, indirizzo e coordinate bancarie di una persona assicurata e può inoltre falsificare un'e-mail a nome del fornitore di servizi o della cassa può scrivere un messaggio difficilmente distinguibile da uno autentico, ad esempio la richiesta di aggiornare l'IBAN per il prossimo pagamento della rendita. È proprio da questa falsificazione che protegge DMARC.

PK Softech non è un caso isolato, ma l'esempio più evidente di un modello ricorrente. Ho verificato quanto siano preparati gli istituti di previdenza svizzeri e i loro fornitori di servizi tipici sul fronte dell'autenticazione delle e-mail.

## Come è stata effettuata la verifica

L'analisi comprende 66 istituti di previdenza del secondo pilastro, dalle grandi casse di diritto pubblico alle fondazioni collettive degli assicuratori fino a enti collettivi più piccoli. Sono stati verificati esclusivamente record DNS pubblici: SPF, DKIM, DMARC, record MX, nonché DNSSEC e DANE (record TLSA dei server di posta, verificati il 19.10.2026). Si tratta delle stesse informazioni che ogni sistema di posta ricevente consulta per ogni messaggio in entrata.

La verifica è stata puramente passiva. Sono stati interrogati solo DNS pubblici, senza stabilire connessioni ai sistemi degli enti, senza tentare accessi e senza raccogliere dati personali. L'analisi rappresenta soltanto ciò che è comunque consultabile da chiunque tramite fonti pubbliche. Le fondazioni collettive di gruppo sono state riunite sotto il dominio principale dell'ente gestore; singole fondazioni affiliate possono differire.

## Il risultato

| Caratteristica | Quota dei 66 enti |
|---|---|
| SPF presente | 88 % |
| SPF rigoroso (`-all`) | 59 % |
| DMARC presente | 59 % |
| DMARC `p=reject` (protezione completa) | 20 % |
| DMARC `p=quarantine` | 26 % |
| DMARC `p=none` (solo monitoraggio) | 14 % |
| nessun DMARC | 41 % |
| DKIM rilevabile | 55 % |
| DNSSEC | 38 % |
| DANE | 9 % |

Tre punti risaltano.

**Solo il 20 per cento è protetto efficacemente con DMARC `p=reject`.** Solo questa policy istruisce una piattaforma di posta ricevente a rifiutare effettivamente i messaggi falsificati. In tutti gli altri casi, una falsificazione finisce in caso di dubbio nella casella di posta della persona assicurata.

**Il 41 per cento non ha DMARC.** Il loro dominio mittente può essere utilizzato senza difficoltà per e-mail fraudolente e l'ente non ne viene nemmeno a conoscenza, perché senza DMARC manca anche la funzione di reporting. Colpisce il fatto che tra questi vi siano anche casse che gestiscono la posta con Microsoft 365, dove DMARC può essere configurato con poco sforzo.

**DMARC `p=none` non è una protezione.** Il 14 per cento di questo gruppo ha impostato una policy DMARC, ma l'ha lasciata al livello di monitoraggio. Ciò fornisce report, ma non rifiuta un singolo messaggio falsificato. Nella pratica è una fase intermedia che spesso rimane in vigore per mesi o anni.

Anche per la crittografia del trasporto emerge un quadro simile: solo 6 dei 66 enti (9 per cento) proteggono i loro server di posta con DANE. In questo modo, un server di posta mittente verifica tramite un record firmato con DNSSEC se è connesso in modo cifrato al destinatario corretto. Il prerequisito è DNSSEC, che il 38 per cento degli enti ha già attivato. Per questi, DANE è il passo successivo naturale.

## Tutti i dati da esplorare

La seguente infografica mostra lo stato verificato di tutti i 66 istituti di previdenza e 30 fornitori di servizi. I filtri consentono di restringere la ricerca per stato DMARC, categoria e provider di posta, e ogni voce può essere espansa per visualizzare i risultati DNS completi.

<div data-dmarc-infographic></div>

## Confronto: le maggiori aziende svizzere

Il [Mail-Sicherheitsindex Schweiz](/open-data/mail-security-index) mostra come si posizionano gli istituti di previdenza rispetto al resto dell'economia. Con lo stesso metodo passivo, verifica ogni giorno oltre 1000 delle maggiori aziende svizzere, nonché Confederazione, cantoni e comuni, con una mappa interattiva per cantone, confronti per settore e provider di posta e una pagina di dettaglio per ogni organizzazione. Tutti gli istituti di previdenza e i fornitori di servizi qui verificati sono inclusi con un proprio profilo, raggruppati nel settore [istituti di previdenza](/open-data/mail-security-index/branche/vorsorgeeinrichtungen) e aggiornati quotidianamente. Nell'infografica sopra, il link nella voce espansa porta direttamente al rispettivo profilo.

La differenza è netta: delle 518 aziende nell'indice, il 18.10.2026 il 38 per cento utilizza DMARC con `p=reject`, contro il 20 per cento degli istituti di previdenza. Il 5 per cento delle aziende non ha affatto DMARC, contro il 41 per cento degli istituti di previdenza.

## I fornitori sono il tema più importante

L'incidente di fine settembre non ha colpito una cassa, bensì un fornitore di servizi comune. Per questo ho verificato, con lo stesso metodo passivo, 30 fornitori tipici: produttori di software gestionale, aziende che forniscono gestione esternalizzata, servizi attuariali e di consulenza, banche depositarie e fornitori IT.

In media, i fornitori di servizi sono messi leggermente meglio delle casse: il 40 per cento usa DMARC con `p=reject`, contro il 20 per cento degli enti. Determinante è però la distribuzione tra le categorie.

| Categoria | con DMARC `p=reject` |
|---|---|
| Servizi attuariali e consulenza | buono |
| Banche depositarie | buono |
| Fornitori IT | misto |
| Gestione esternalizzata | debole |
| Software gestionale | il più debole |

I produttori di software gestionale sono il gruppo più debole. Nessuno dei fornitori di software verificati usa DMARC con efficacia completa e il fornitore colpito PK Softech non ha alcun record DMARC. Si tratta proprio della categoria che tratta dati previdenziali ed è profondamente connessa alle casse tramite interfacce, nonché della categoria coinvolta nell'incidente attuale.

Da ciò deriva il punto centrale in termini di diligenza: una cassa con `p=reject` è ben protetta finché l'aggressore tenta di falsificarne il dominio. Se invece può falsificare il dominio di un fornitore di servizi scarsamente protetto, attraverso il quale avvengono comunque comunicazioni con gli assicurati, l'e-mail è altrettanto credibile. Proteggere il proprio dominio è quindi solo il primo passo. Il secondo consiste nell'obbligare i fornitori più importanti a raggiungere lo stesso livello, preferibilmente per contratto.

## Cosa significa per il consiglio di fondazione

La responsabilità della sicurezza delle informazioni spetta all'organo supremo dell'istituto di previdenza. Una fuoriuscita di dati comporta inoltre obblighi concreti ai sensi della legge sulla protezione dei dati riveduta: una violazione della sicurezza dei dati deve essere notificata all'IFPDT se è probabile che comporti un rischio elevato per le persone interessate, e gli interessati devono essere informati nella misura necessaria alla loro protezione.

In questa situazione, l'autenticazione delle e-mail è la singola misura più efficace con il minimo sforzo. SPF, DKIM e DMARC non richiedono licenze né nuovo software. Consistono in record DNS e in una configurazione corretta della piattaforma di posta esistente. La protezione agisce subito per ogni persona assicurata, senza che questa debba fare nulla.

## Cosa fare

Per gli enti privi di una protezione efficace, un percorso chiaro porta all'obiettivo:

1. **Verificare SPF e impostarlo in modo rigoroso.** Il record SPF elenca i sistemi di invio autorizzati e termina con `-all`, affinché tutti gli altri mittenti siano considerati non autorizzati.
2. **Attivare DKIM.** Ogni messaggio in uscita viene firmato crittograficamente. In Microsoft 365 e nei comuni mail gateway è un'impostazione, non un progetto.
3. **Introdurre DMARC e renderlo gradualmente più restrittivo.** Iniziare con `p=none` e reporting attivo per rilevare il traffico di posta legittimo. Poi `p=quarantine`, infine `p=reject`. L'intero percorso richiede alcune settimane a seconda dell'ambiente, non mesi.
4. **Analizzare i report.** I report DMARC mostrano chi invia e-mail a nome del dominio e rivelano sia sistemi legittimi dimenticati sia abusi.
5. **Almeno verificare DANE e valutare la crittografia della posta.** DANE protegge la crittografia del trasporto tra server di posta tramite DNSSEC. Con poco sforzo è possibile chiarire se la propria piattaforma di posta e il provider DNS lo supportano. Per i messaggi contenenti dati salariali e previdenziali, occorre inoltre definire come inviarli con crittografia end-to-end, ad esempio tramite un mail gateway con S/MIME o una consegna sicura via portale web.
6. **Responsabilizzare i fornitori di servizi.** I fornitori di software, i gestori esternalizzati e i partner IT dovrebbero dimostrare di avere lo stesso livello per SPF, DKIM e DMARC. Questo va previsto nei contratti e nella verifica periodica dei fornitori.

## Conclusione

Gli attacchi degli ultimi tempi hanno preso di mira un fornitore di servizi, non le casse stesse. I danni conseguenti si verificano però ovunque un dominio mittente sia privo di protezione, tanto presso la cassa quanto presso i suoi fornitori. Quattro dei cinque istituti di previdenza verificati non sono completamente protetti contro mittenti falsificati, due su cinque non hanno affatto DMARC e proprio i produttori di software gestionale ottengono i risultati peggiori. La lacuna è nota, economica da colmare e produce effetto immediato. Dopo una fuoriuscita di dati che riguarda nome, indirizzo e coordinate bancarie, non è una questione tecnica secondaria, bensì parte dell'obbligo di diligenza dell'organo supremo, lungo l'intera catena di fornitura.

## Fonti

1.  [SRF: Cassa pensioni della Confederazione, come hanno potuto dati sensibili finire presso un'azienda esterna di software](https://www.srf.ch/news/schweiz/pensionskasse-des-bundes-wie-konnten-heikle-daten-bei-externer-softwarefirma-landen): approfondimento sull'incidente, sui tipi di dati interessati e sul ruolo del fornitore di software.

2.  [cash.ch (AWP): Attacco al fornitore di Publica, potrebbero essere stati coinvolti dati salariali](https://www.cash.ch/news/top-news/angriff-auf-publica-lieferanten-lohndaten-konnten-betroffen-sein-975247): Publica identifica PK Softech AG nei confronti di AWP come azienda colpita.

3.  [Netzwoche: La cassa pensioni della Confederazione Publica segnala una fuoriuscita di dati](https://www.netzwoche.ch/news/2026-10-08/bundespensionskasse-publica-meldet-datenabfluss): notizia sull'attacco al fornitore di software e sulla fuoriuscita di dati.

4.  [Mimikama: La polizia svizzera mette in guardia da possibili chiamate di investimento dopo una fuoriuscita di dati](https://www.mimikama.org/?p=436628): inquadramento dell'avvertimento sulle chiamate fraudolente con i dati fuoriusciti.

5.  [IFPDT: Notifica di violazioni della sicurezza dei dati](https://www.edoeb.admin.ch/de/meldung-von-verletzungen-der-datensicherheit): obbligo di notifica ai sensi della LPD riveduta e procedura in caso di violazioni della sicurezza dei dati.
