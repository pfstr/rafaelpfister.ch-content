---
title: "Il fornitore di servizi per casse pensioni colpito dall'attacco non ha alcun record DMARC"
navTitle: "DMARC casse pensioni"
description: "PK Softech, il fornitore di servizi per casse pensioni attaccato alla fine di settembre 2026, non dispone tuttora di alcun record DMARC. Un'analisi passiva di 66 istituti di previdenza e 30 fornitori di servizi mostra che non si tratta di un caso isolato: soprattutto i produttori di software amministrativo sono privi di protezione."
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
translationSourceHash: a673909df1426a76c58ced5aaa46d839e480ae867b2ffad468c903d65919e0c3
translationModel: gpt-5.6-terra
translatedAt: 2026-10-26T09:15:28.163Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/il-fornitore-di-servizi-per-casse-pensioni-attaccato-non-ha-alcun-record-dmarc
---

Alla fine di settembre 2026 è stata attaccata PK Softech AG, un produttore di software la cui soluzione previdenziale è utilizzata da diverse casse pensioni svizzere. Attraverso questa via sono fuoriusciti dati. Secondo le notizie, sono coinvolte tra le altre la cassa pensioni della Confederazione Publica, la cassa pensioni di Migros e la cassa pensioni della Città di Zurigo, nonché altre istituzioni. A seconda della cassa, si tratta di nome, numero AVS, stato civile, indirizzo e IBAN, oltre a dati salariali e previdenziali.

Questa fuoriuscita è già di per sé grave. Riguarda dati identificativi e finanziari di un'ampia parte della popolazione attiva e, attraverso Publica, di vaste parti dell'Amministrazione federale. Dati di questa quantità e qualità non sono un problema soltanto per le singole persone. Consentono furti d'identità, inganni mirati ed estorsioni su larga scala e costituiscono quindi un pericolo per l'ordine pubblico e la sicurezza nazionale. I patrimoni previdenziali stessi sono rimasti intatti, ma la piattaforma informativa Cybercrimepolice.ch aveva già avvertito di chiamate fraudolente con i dati sottratti.

A ciò si aggiunge una constatazione difficile da spiegare per un fornitore di servizi di questa categoria: **PK Softech AG non dispone tuttora di alcun record DMARC.** Il suo dominio ha un record SPF rigoroso, ma nessuna policy DMARC (verificato il 18.10.2026). Di conseguenza, è possibile falsificare l'indirizzo del mittente del fornitore attaccato senza che una piattaforma di posta ricevente lo rifiuti in modo affidabile. Dopo una fuoriuscita di dati, questa è la situazione iniziale più sfavorevole immaginabile. Chi conosce nome, indirizzo e coordinate bancarie di una persona assicurata e può inoltre falsificare un'e-mail a nome del fornitore o della cassa può scrivere un messaggio quasi indistinguibile da uno autentico, ad esempio la richiesta di aggiornare l'IBAN per il prossimo pagamento della rendita. DMARC protegge proprio da questa falsificazione.

PK Softech non è un caso isolato, ma l'esempio più evidente di un modello ricorrente. Ho verificato quanto siano preparati gli istituti di previdenza svizzeri e i loro tipici fornitori di servizi in materia di autenticazione e-mail.

## Come è stata effettuata la verifica

L'analisi comprende 66 istituti di previdenza del secondo pilastro, dalle grandi casse di diritto pubblico alle fondazioni collettive degli assicuratori fino a istituzioni collettive più piccole. Sono stati verificati esclusivamente record DNS pubblici: SPF, DKIM, DMARC, record MX nonché DNSSEC e DANE (record TLSA dei server di posta, verificati il 19.10.2026). Si tratta degli stessi dati che ogni sistema di posta ricevente interroga per ogni messaggio in arrivo.

La verifica è stata esclusivamente passiva. Sono stati interrogati soltanto DNS pubblici, senza stabilire connessioni ai sistemi delle istituzioni, tentare accessi o raccogliere dati personali. L'analisi rappresenta solo ciò che è comunque consultabile da chiunque tramite fonti pubbliche. Le fondazioni collettive di gruppo sono state raggruppate sotto il dominio principale dell'ente promotore; singole fondazioni affiliate possono differire.

## Il risultato

| Caratteristica | Quota delle 66 istituzioni |
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

Emergono tre punti.

**Solo il 20 per cento è protetto efficacemente con DMARC `p=reject`.** Solo questa policy istruisce una piattaforma di posta ricevente a rifiutare effettivamente i messaggi falsificati. In tutti gli altri casi, in caso di dubbio una falsificazione finisce nella casella di posta della persona assicurata.

**Il 41 per cento non ha DMARC.** Il loro dominio mittente può essere usato senza difficoltà per e-mail fraudolente, e l'istituzione non ne viene nemmeno a conoscenza, poiché senza DMARC manca anche la funzione di reporting. Colpisce il fatto che tra queste vi siano anche casse che gestiscono la posta con Microsoft 365, dove DMARC può essere configurato con poco sforzo.

**DMARC `p=none` non è una protezione.** Il 14 per cento di questo gruppo ha impostato una policy DMARC, ma l'ha lasciata al livello di monitoraggio. Questo fornisce report, ma non rifiuta neppure un singolo messaggio falsificato. Nella pratica è un passaggio intermedio che spesso rimane tale per mesi o anni.

Per la cifratura del trasporto emerge un quadro simile: solo 6 delle 66 istituzioni (9 per cento) proteggono i propri server di posta con DANE. In questo modo, un server di posta mittente verifica tramite un record firmato con DNSSEC se è connesso in modo cifrato al destinatario corretto. Il requisito è DNSSEC, che il 38 per cento delle istituzioni ha già attivato. Per queste, DANE è il successivo passo naturale.

## Tutti i dati da esplorare

La seguente infografica mostra lo stato verificato di tutti i 66 istituti di previdenza e 30 fornitori di servizi. I filtri consentono di restringere la selezione per stato DMARC, categoria e provider di posta, e ogni voce può essere aperta per visualizzare i risultati DNS completi.

<div data-dmarc-infographic></div>

## Confronto: le maggiori aziende svizzere

Il [Mail-Sicherheitsindex Schweiz](/open-data/mail-security-index) mostra come gli istituti di previdenza si posizionino rispetto al resto dell'economia. Con lo stesso metodo passivo verifica quotidianamente oltre 1000 delle maggiori aziende svizzere, nonché Confederazione, cantoni e comuni, con una mappa interattiva per cantoni, confronti per settore e provider di posta e una pagina dettagliata per ogni organizzazione. Tutti gli istituti di previdenza e fornitori di servizi qui verificati sono inclusi con un proprio profilo, raggruppati nel settore [istituti di previdenza](/open-data/mail-security-index/branche/vorsorgeeinrichtungen) e aggiornati quotidianamente. Nell'infografica sopra, il link nella voce aperta conduce direttamente al relativo profilo.

La differenza è netta: delle 518 aziende nell'indice, il 18.10.2026 il 38 per cento utilizza DMARC con `p=reject`, contro il 20 per cento degli istituti di previdenza. Il 5 per cento delle aziende è completamente privo di DMARC, contro il 41 per cento degli istituti di previdenza.

## I fornitori sono il problema maggiore

L'incidente di fine settembre non ha colpito una cassa, bensì un fornitore di servizi comune. Per questo ho verificato con lo stesso metodo passivo 30 fornitori tipici: produttori di software amministrativo, aziende per la gestione esternalizzata, attuariato e consulenza, banche depositarie e fornitori IT.

In media, i fornitori di servizi sono messi leggermente meglio delle casse: il 40 per cento utilizza DMARC con `p=reject`, contro il 20 per cento delle istituzioni. Decisiva, tuttavia, è la distribuzione tra le categorie.

| Categoria | con DMARC `p=reject` |
|---|---|
| Attuariato e consulenza | buono |
| Banche depositarie | buono |
| Fornitori IT | misto |
| Gestione esternalizzata | debole |
| Software amministrativo | i più deboli |

I produttori di software amministrativo sono il gruppo più debole. Nessuno dei fornitori software verificati utilizza DMARC con piena efficacia, e il fornitore attaccato PK Softech non ha alcun record DMARC. Si tratta proprio della categoria che tratta dati previdenziali ed è profondamente collegata alle casse tramite interfacce, nonché della categoria coinvolta nell'incidente attuale.

Da ciò deriva il punto essenziale per la diligenza: una cassa con `p=reject` è ben protetta finché l'aggressore vuole falsificare il suo dominio. Se invece può falsificare il dominio di un fornitore scarsamente protetto, attraverso il quale avviene comunque la comunicazione con gli assicurati, l'e-mail è altrettanto credibile. Proteggere il proprio dominio è quindi solo il primo passo. Il secondo è obbligare i fornitori più importanti a raggiungere lo stesso livello, preferibilmente per contratto.

## Cosa significa per il consiglio di fondazione

La responsabilità della sicurezza delle informazioni spetta all'organo supremo dell'istituto di previdenza. Una fuoriuscita di dati comporta inoltre obblighi concreti ai sensi della legge sulla protezione dei dati riveduta: una violazione della sicurezza dei dati deve essere notificata all'IFPDT se è presumibile che comporti un rischio elevato per le persone interessate, e le persone interessate devono essere informate nella misura necessaria alla loro protezione.

In questa situazione, l'autenticazione e-mail è la misura singola più efficace con il minor impegno. SPF, DKIM e DMARC non richiedono licenze né nuovo software. Consistono in record DNS e in una configurazione corretta della piattaforma di posta esistente. La protezione ha effetto immediato per ogni persona assicurata, senza che questa debba fare nulla.

## Cosa fare

Per le istituzioni prive di una protezione efficace, un percorso chiaro porta all'obiettivo:

1. **Verificare SPF e impostarlo in modo rigoroso.** Il record SPF elenca i sistemi autorizzati all'invio e termina con `-all`, affinché tutti gli altri mittenti siano considerati non autorizzati.
2. **Attivare DKIM.** Ogni messaggio in uscita viene firmato crittograficamente. In Microsoft 365 e nei gateway di posta più diffusi si tratta di un'impostazione, non di un progetto.
3. **Introdurre DMARC e irrigidirlo gradualmente.** Iniziare con `p=none` e reporting attivo per rilevare il traffico e-mail legittimo. Successivamente `p=quarantine`, infine `p=reject`. L'intero percorso richiede alcune settimane a seconda dell'ambiente, non mesi.
4. **Analizzare il reporting.** I report DMARC mostrano chi invia e-mail a nome del dominio e rilevano sia sistemi legittimi dimenticati sia abusi.
5. **Verificare almeno DANE e valutare la cifratura della posta.** DANE protegge la cifratura del trasporto tra server di posta tramite DNSSEC. È possibile chiarire con poco impegno se la propria piattaforma di posta e il provider DNS lo supportano. Per i messaggi contenenti dati salariali e previdenziali va inoltre definito come inviarli cifrati end-to-end, ad esempio tramite un gateway di posta con S/MIME o una consegna sicura tramite portale web.
6. **Responsabilizzare i fornitori di servizi.** I fornitori software, i gestori esternalizzati e i partner IT dovrebbero dimostrare lo stesso livello in SPF, DKIM e DMARC. Questo deve rientrare nei contratti e nella verifica periodica dei fornitori.

## Conclusione

Gli attacchi recenti hanno preso di mira un fornitore di servizi, non le casse stesse. I danni conseguenti si verificano però laddove un dominio mittente è privo di protezione, sia presso la cassa sia presso i suoi fornitori. Quattro dei cinque istituti di previdenza verificati non sono completamente protetti contro mittenti falsificati, due su cinque non hanno affatto DMARC e proprio i produttori di software amministrativo ottengono i risultati peggiori. La lacuna è nota, economica da colmare e ha effetto immediato. Dopo una fuoriuscita di dati che riguarda nome, indirizzo e coordinate bancarie, non è un dettaglio tecnico, ma parte del dovere di diligenza dell'organo supremo lungo l'intera catena di fornitura.

## Cronologia delle modifiche

- **25 ottobre 2026:** aggiornato il link alla fonte con il nuovo indirizzo dell'IFPDT.

## Fonti

1.  [SRF: Cassa pensioni della Confederazione, come hanno potuto dati sensibili finire presso un'azienda software esterna](https://www.srf.ch/news/schweiz/pensionskasse-des-bundes-wie-konnten-heikle-daten-bei-externer-softwarefirma-landen): contesto dell'incidente, tipi di dati coinvolti e ruolo del fornitore software.

2.  [cash.ch (AWP): Attacco al fornitore di Publica, potrebbero essere stati coinvolti dati salariali](https://www.cash.ch/news/top-news/angriff-auf-publica-lieferanten-lohndaten-konnten-betroffen-sein-975247): Publica identifica presso AWP PK Softech AG come l'azienda attaccata.

3.  [Netzwoche: La cassa pensioni della Confederazione Publica segnala una fuoriuscita di dati](https://www.netzwoche.ch/news/2026-10-08/bundespensionskasse-publica-meldet-datenabfluss): notizia sull'attacco al fornitore software e sulla fuoriuscita di dati.

4.  [Mimikama: Polizia svizzera avverte dopo una fuoriuscita di dati di possibili chiamate di investimento](https://www.mimikama.org/?p=436628): inquadramento dell'avvertimento relativo a chiamate fraudolente con i dati fuoriusciti.

5.  [IFPDT: DataBreach](https://www.edoeb.admin.ch/de/databreach): obbligo di notifica ai sensi dell'art. 24 LPD, portale di notifica e guida alla procedura in caso di violazioni della sicurezza dei dati.
