---
title: "Il fornitore di servizi per casse pensioni attaccato non ha alcun record DMARC"
navTitle: "DMARC casse pensioni"
description: "PK Softech, il fornitore di servizi per casse pensioni attaccato alla fine di settembre 2026, non dispone tuttora di alcun record DMARC. Un controllo passivo di 66 istituti di previdenza e 30 fornitori di servizi mostra che non si tratta di un caso isolato: soprattutto i produttori di software amministrativo sono privi di protezione."
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
translationSourceHash: 357621c8e85052ed11d792c9e6f911df15e6e32a0ad4cf0c5d92ba1fdbd9f906
translationModel: gpt-5.6-terra
translatedAt: 2026-10-25T19:34:03.505Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/il-fornitore-di-servizi-per-casse-pensioni-attaccato-non-ha-alcun-record-dmarc
---

Alla fine di settembre 2026 è stata attaccata PK Softech AG, un produttore di software la cui soluzione previdenziale è utilizzata da diverse casse pensioni svizzere. Attraverso questa via sono fuoriusciti dati. Secondo le notizie, sono coinvolte tra le altre la cassa pensioni federale Publica, la cassa pensioni Migros e la cassa pensioni della Città di Zurigo, nonché altre istituzioni. A seconda della cassa, si tratta di nome, numero AVS, stato civile, indirizzo e IBAN, nonché dati salariali e previdenziali.

Questa fuoriuscita è già di per sé grave. Riguarda dati identificativi e finanziari di un'ampia parte della popolazione attiva e, attraverso Publica, di grandi parti dell'Amministrazione federale. Dati di questa quantità e qualità non rappresentano un problema solo per le singole persone. Consentono furti d'identità, inganni mirati ed estorsioni su larga scala e costituiscono quindi un pericolo per l'ordine pubblico e la sicurezza nazionale. Gli averi previdenziali stessi non sono stati toccati, ma la piattaforma informativa Cybercrimepolice.ch aveva già messo in guardia da chiamate fraudolente con i dati sottratti.

A ciò si aggiunge una constatazione difficile da spiegare per un fornitore di servizi di questa categoria: **PK Softech AG non ha tuttora alcun record DMARC.** Il suo dominio ha un record SPF rigoroso, ma nessuna politica DMARC (verificato il 18.10.2026). Di conseguenza, è possibile falsificare l'indirizzo del mittente del fornitore di servizi attaccato senza che una piattaforma di posta ricevente lo rifiuti in modo affidabile. Dopo una fuoriuscita di dati, questa è la situazione iniziale più sfavorevole immaginabile. Chi conosce nome, indirizzo e coordinate bancarie di una persona assicurata e può inoltre falsificare un'e-mail a nome del fornitore di servizi o della cassa può scrivere un messaggio difficilmente distinguibile da uno autentico, ad esempio chiedendo di aggiornare l'IBAN per il prossimo pagamento della rendita. È proprio da questa falsificazione che protegge DMARC.

PK Softech non è un caso isolato, ma l'esempio più evidente di una tendenza. Ho verificato quanto siano preparati gli istituti di previdenza svizzeri e i loro tipici fornitori di servizi in materia di autenticazione delle e-mail.

## Come è stato verificato

L'analisi comprende 66 istituti di previdenza del secondo pilastro, dalle grandi casse di diritto pubblico alle fondazioni collettive degli assicuratori, fino a piccoli istituti comuni. Sono stati verificati esclusivamente record DNS pubblici: SPF, DKIM, DMARC, record MX, nonché DNSSEC e DANE (record TLSA dei server di posta, verificati il 19.10.2026). Si tratta delle stesse informazioni che ogni sistema di posta ricevente consulta per ogni messaggio in arrivo.

La verifica è stata puramente passiva. Sono stati interrogati solo DNS pubblici, senza stabilire connessioni ai sistemi degli istituti, senza tentare accessi e senza raccogliere dati personali. L'analisi mostra soltanto ciò che è comunque leggibile da fonti pubbliche per chiunque. Le fondazioni collettive di gruppo sono riunite sotto il dominio principale dell'ente promotore; singole fondazioni affiliate possono differire.

## Il risultato

| Caratteristica | Quota dei 66 istituti |
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

Tre aspetti risaltano.

**Solo il 20 per cento è protetto efficacemente con DMARC `p=reject`.** Solo questa politica istruisce una piattaforma di posta ricevente a rifiutare effettivamente i messaggi falsificati. In tutti gli altri casi, una falsificazione finisce in caso di dubbio nella casella di posta della persona assicurata.

**Il 41 per cento non ha DMARC.** Il loro dominio mittente può essere usato facilmente per e-mail fraudolente e l'istituto non ne viene nemmeno a conoscenza, poiché senza DMARC manca anche la funzione di reporting. Colpisce che tra queste vi siano anche casse che gestiscono la posta con Microsoft 365, dove DMARC può essere configurato con poco sforzo.

**DMARC `p=none` non è una protezione.** Il 14 per cento di questo gruppo ha impostato una politica DMARC, ma l'ha lasciata al livello di monitoraggio. Questo fornisce report, ma non rifiuta neppure un singolo messaggio falsificato. Nella pratica è un passaggio intermedio che spesso rimane tale per mesi o anni.

Per la cifratura del trasporto emerge un quadro simile: solo 6 dei 66 istituti (9 per cento) proteggono i propri server di posta con DANE. In questo modo, un server di posta mittente verifica mediante un record firmato con DNSSEC se è connesso in modo cifrato al destinatario corretto. Il requisito è DNSSEC, già attivato dal 38 per cento degli istituti. Per questi, DANE è il naturale passo successivo.

## Tutti i dati da esplorare

La seguente infografica mostra lo stato verificato di tutti i 66 istituti di previdenza e 30 fornitori di servizi. I filtri permettono di restringere la ricerca per stato DMARC, categoria e provider di posta, e ogni voce può essere aperta per visualizzare i risultati DNS completi.

<div data-dmarc-infographic></div>

## Confronto: le maggiori aziende svizzere

Il [Mail-Sicherheitsindex Schweiz](/open-data/mail-security-index) mostra come gli istituti di previdenza si posizionino rispetto al resto dell'economia. Con lo stesso metodo passivo, verifica quotidianamente oltre 1000 delle maggiori aziende svizzere, nonché Confederazione, Cantoni e Comuni, con una mappa interattiva per Cantone, confronti per settore e fornitore di posta e una pagina di dettaglio per ogni organizzazione. Tutti gli istituti di previdenza e fornitori di servizi qui verificati vi sono inclusi con un proprio profilo, riuniti nel settore [istituti di previdenza](/open-data/mail-security-index/branche/vorsorgeeinrichtungen) e aggiornati ogni giorno. Nell'infografica sopra, il link nella voce aperta conduce direttamente al rispettivo profilo.

La distanza è netta: delle 518 aziende nell'indice, il 18.10.2026 il 38 per cento utilizza DMARC con `p=reject`, mentre tra gli istituti di previdenza è il 20 per cento. Il 5 per cento delle aziende non ha affatto DMARC, contro il 41 per cento degli istituti di previdenza.

## I fornitori sono il tema più rilevante

L'incidente di fine settembre non ha colpito una cassa, bensì un fornitore di servizi comune. Per questo ho verificato con lo stesso metodo passivo 30 fornitori tipici: produttori di software amministrativo, società per la gestione aziendale esternalizzata, attuari e consulenti, banche depositarie e fornitori IT.

In media, i fornitori di servizi sono posizionati un po' meglio delle casse: il 40 per cento utilizza DMARC con `p=reject`, contro il 20 per cento degli istituti. Determinante è però la distribuzione tra le categorie.

| Categoria | con DMARC `p=reject` |
|---|---|
| Attuariato e consulenza | buono |
| Banche depositarie | buono |
| Fornitori IT | misto |
| Gestione aziendale esternalizzata | debole |
| Software amministrativo | più debole |

I produttori di software amministrativo sono il gruppo più debole. Nessuno dei fornitori software verificati utilizza DMARC con piena efficacia e il fornitore attaccato PK Softech non ha alcun record DMARC. Si tratta proprio della categoria che tratta dati previdenziali ed è profondamente connessa alle casse tramite interfacce, nonché della categoria coinvolta nell'incidente attuale.

Da ciò deriva il punto essenziale in termini di diligenza: una cassa con `p=reject` è ben protetta finché l'aggressore vuole falsificare il suo dominio. Se invece può falsificare il dominio di un fornitore di servizi poco protetto, attraverso il quale avviene già la comunicazione con gli assicurati, l'e-mail è altrettanto credibile. Proteggere il proprio dominio è quindi solo il primo passo. Il secondo è obbligare i principali fornitori di servizi a raggiungere lo stesso livello, idealmente per contratto.

## Cosa significa per il consiglio di fondazione

La responsabilità per la sicurezza delle informazioni spetta all'organo supremo dell'istituto di previdenza. Una fuoriuscita di dati comporta inoltre obblighi concreti ai sensi della legge sulla protezione dei dati riveduta: una violazione della sicurezza dei dati deve essere notificata all'IFPDT se è probabile che comporti un rischio elevato per le persone interessate, e le persone interessate devono essere informate nella misura necessaria alla loro protezione.

In questa situazione, l'autenticazione delle e-mail è la misura singola più efficace con il minor sforzo. SPF, DKIM e DMARC non richiedono licenze né nuovo software. Consistono in record DNS e in una configurazione corretta della piattaforma di posta esistente. La protezione agisce immediatamente per ogni persona assicurata, senza che quest'ultima debba fare nulla.

## Cosa fare

Per gli istituti privi di una protezione efficace, un percorso chiaro conduce all'obiettivo:

1. **Verificare SPF e impostarlo in modo rigoroso.** Il record SPF elenca i sistemi autorizzati all'invio e termina con `-all`, affinché tutti gli altri mittenti siano considerati non autorizzati.
2. **Attivare DKIM.** Ogni messaggio in uscita viene firmato crittograficamente. In Microsoft 365 e nei comuni gateway di posta si tratta di un'impostazione, non di un progetto.
3. **Introdurre DMARC e renderlo progressivamente più rigoroso.** Si inizia con `p=none` e reporting attivo, per rilevare il traffico e-mail legittimo. Poi `p=quarantine`, infine `p=reject`. A seconda dell'ambiente, l'intero percorso richiede alcune settimane, non mesi.
4. **Valutare il reporting.** I report DMARC mostrano chi invia e-mail a nome del dominio e individuano sia sistemi legittimi dimenticati sia abusi.
5. **Almeno verificare DANE e analizzare la cifratura della posta.** DANE protegge la cifratura del trasporto tra server di posta tramite DNSSEC. Se la propria piattaforma di posta e il provider DNS lo supportano può essere chiarito con poco sforzo. Per i messaggi con dati salariali e previdenziali occorre inoltre disciplinare come inviarli cifrati end-to-end, ad esempio tramite un gateway di posta con S/MIME o una consegna sicura attraverso un portale web.
6. **Responsabilizzare i fornitori di servizi.** I fornitori software, la gestione aziendale esternalizzata e i partner IT dovrebbero dimostrare lo stesso livello in SPF, DKIM e DMARC. Questo deve figurare nei contratti e nella verifica periodica dei fornitori.

## Conclusione

Gli attacchi recenti hanno preso di mira un fornitore di servizi, non le casse stesse. I danni conseguenti sorgono però ovunque un dominio mittente sia privo di protezione, sia presso la cassa sia presso i suoi fornitori. Quattro istituti di previdenza verificati su cinque non sono completamente protetti contro mittenti falsificati, due su cinque non hanno affatto DMARC e proprio i produttori di software amministrativo ottengono i risultati peggiori. La lacuna è nota, economica da colmare e produce effetto immediato. Dopo una fuoriuscita di dati che riguarda nome, indirizzo e coordinate bancarie, non è una questione tecnica secondaria, ma parte del dovere di diligenza dell'organo supremo, lungo l'intera catena di fornitura.

## Fonti

1.  [SRF: Cassa pensioni della Confederazione, come hanno potuto dati sensibili finire presso una società di software esterna](https://www.srf.ch/news/schweiz/pensionskasse-des-bundes-wie-konnten-heikle-daten-bei-externer-softwarefirma-landen): Contesto dell'incidente, tipi di dati coinvolti e ruolo del fornitore software.

2.  [cash.ch (AWP): Attacco a un fornitore di Publica, i dati salariali potrebbero essere stati coinvolti](https://www.cash.ch/news/top-news/angriff-auf-publica-lieferanten-lohndaten-konnten-betroffen-sein-975247): Publica indica a AWP PK Softech AG come società attaccata.

3.  [Netzwoche: La cassa pensioni federale Publica segnala una fuoriuscita di dati](https://www.netzwoche.ch/news/2026-10-08/bundespensionskasse-publica-meldet-datenabfluss): Notizia sull'attacco al fornitore software e sulla fuoriuscita di dati.

4.  [Mimikama: La polizia svizzera mette in guardia dopo una fuoriuscita di dati da possibili chiamate relative a investimenti](https://www.mimikama.org/?p=436628): Inquadramento dell'avvertimento sulle chiamate fraudolente con i dati fuoriusciti.

5.  [IFPDT: DataBreach](https://www.edoeb.admin.ch/de/databreach): Obbligo di notifica ai sensi dell'art. 24 LPD, portale di notifica e guida sulla procedura in caso di violazioni della sicurezza dei dati.
