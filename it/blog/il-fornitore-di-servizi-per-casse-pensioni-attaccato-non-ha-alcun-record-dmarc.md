---
title: "Il fornitore di servizi per casse pensioni attaccato non ha alcun record DMARC"
navTitle: "DMARC casse pensioni"
description: "Il fornitore di servizi per casse pensioni PK Softech, attaccato alla fine di settembre 2026, non dispone tuttora di alcun record DMARC. Un controllo passivo di 66 istituti di previdenza e 30 fornitori di servizi mostra che non si tratta di un caso isolato: in particolare i produttori di software di gestione sono privi di protezione."
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
aiPrompt: |
  Du bist mein Berater für E-Mail-Sicherheit. Prüfe für meine Domain, ob SPF, DKIM, DMARC und MTA-STS korrekt gesetzt sind, erkläre mir den aktuellen Stand in einfachen Worten und zeige mir Schritt für Schritt, wie ich DMARC gefahrlos von p=none über p=quarantine auf p=reject anhebe.
translationOf: dmarc-pensionskassen-schweiz
url: https://rafaelpfister.ch/it/blog/il-fornitore-di-servizi-per-casse-pensioni-attaccato-non-ha-alcun-record-dmarc
translationSourceHash: 64ec035a358c39b09487102da66cf7aca02df2c567e29e343895906a4490d205
translationModel: gpt-5.6-terra
translatedAt: 2026-10-18T14:53:32.024Z
translationReview: automatic
---

Alla fine di settembre 2026 è stata attaccata PK Softech AG, un produttore di software la cui soluzione previdenziale è utilizzata da diverse casse pensioni svizzere. Attraverso questa via sono fuoriusciti dati. Secondo le notizie, tra gli enti colpiti figurano la cassa pensioni della Confederazione Publica, la cassa pensioni Migros e la cassa pensioni della Città di Zurigo, nonché altri istituti. A seconda della cassa, sono coinvolti nome, numero AVS, stato civile, indirizzo e IBAN, nonché dati salariali e previdenziali.

Questa fuoriuscita è di per sé grave. Riguarda dati identificativi e finanziari di una larga parte della popolazione attiva e, tramite Publica, di ampie parti dell'Amministrazione federale. Dati di questa quantità e qualità non costituiscono solo un problema per le singole persone. Consentono furti d'identità, inganni mirati ed estorsioni su larga scala e rappresentano quindi un pericolo per l'ordine pubblico e la sicurezza nazionale. Gli averi previdenziali stessi non sono stati toccati, ma la polizia contro la criminalità informatica aveva già avvertito di chiamate fraudolente con i dati sottratti.

A ciò si aggiunge una constatazione difficilmente spiegabile per un fornitore di servizi di questa categoria: **PK Softech AG non dispone tuttora di alcun record DMARC.** Il suo dominio presenta un record SPF rigoroso, ma nessuna policy DMARC (verificato il 18.10.2026). Di conseguenza, l'indirizzo del mittente del fornitore di servizi attaccato può essere falsificato senza che una piattaforma di posta ricevente lo rifiuti in modo affidabile. Dopo una fuoriuscita di dati, è la situazione iniziale più sfavorevole immaginabile. Chi conosce nome, indirizzo e coordinate bancarie di una persona assicurata e può inoltre falsificare un'e-mail a nome del fornitore di servizi o della cassa può scrivere un messaggio quasi indistinguibile da uno reale, ad esempio chiedendo di aggiornare l'IBAN per il prossimo pagamento della rendita. È proprio da questa falsificazione che protegge DMARC.

PK Softech non è un caso isolato, bensì l'esempio più evidente di un modello ricorrente. Ho verificato quanto siano preparati gli istituti di previdenza svizzeri e i loro tipici fornitori di servizi in materia di autenticazione delle e-mail.

## Come è stato effettuato il controllo

L'analisi comprende 66 istituti di previdenza del secondo pilastro, dalle grandi casse di diritto pubblico alle fondazioni collettive degli assicuratori fino ai più piccoli istituti comuni. Sono stati controllati esclusivamente record DNS pubblici: SPF, DKIM, DMARC, MTA-STS, TLS-RPT e record MX. Si tratta delle stesse informazioni che ogni sistema di posta ricevente consulta per ogni messaggio in entrata.

Il controllo è stato puramente passivo. Sono stati interrogati solo DNS pubblici, senza stabilire connessioni ai sistemi degli istituti, senza tentare alcun accesso e senza raccogliere dati personali. L'analisi riflette solo ciò che è comunque visibile a chiunque da fonti pubbliche. Le fondazioni collettive di gruppo sono ricondotte al dominio principale dell'ente promotore; singole fondazioni affiliate possono differire.

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
| DKIM riconoscibile | 55 % |
| MTA-STS | 2 % |
| TLS-RPT | 5 % |

Emergono tre punti.

**Solo il 20 per cento è effettivamente protetto con DMARC `p=reject`.** Solo questa policy indica a una piattaforma di posta ricevente di rifiutare realmente i messaggi falsificati. In tutti gli altri casi, in caso di dubbio una falsificazione arriva nella casella di posta della persona assicurata.

**Il 41 per cento non ha DMARC.** Il dominio del mittente può essere usato senza problemi per e-mail fraudolente e l'istituto non ne viene nemmeno a conoscenza, poiché senza DMARC manca anche la funzione di reporting. Colpisce il fatto che tra questi vi siano anche casse che gestiscono la posta con Microsoft 365, dove DMARC può essere configurato con poco sforzo.

**DMARC `p=none` non è una protezione.** Il 14 per cento di questo gruppo ha impostato una policy DMARC, ma l'ha lasciata al livello di monitoraggio. Fornisce report, ma non rifiuta neppure un singolo messaggio falsificato. Nella pratica, è un passaggio intermedio che spesso rimane tale per mesi o anni.

A ciò si aggiunge una constatazione costante: con il 2 per cento, MTA-STS è praticamente inesistente. Senza MTA-STS, la cifratura del trasporto delle e-mail in entrata non è imposta, il che favorisce un aggressore con accesso al percorso di trasporto.

## Tutti i dati da esplorare

La seguente infografica mostra lo stato verificato di tutti i 66 istituti di previdenza e 30 fornitori di servizi. I filtri consentono di restringere la ricerca per stato DMARC, categoria e provider di posta, e ogni voce può essere espansa per visualizzare i risultati DNS completi.

<div data-dmarc-infographic></div>

## Confronto: le maggiori aziende svizzere

Il [Mail-Sicherheitsindex Schweiz](/open-data/mail-security-index) mostra come gli istituti di previdenza si posizionino rispetto al resto dell'economia. Con lo stesso metodo passivo, esamina quotidianamente oltre 500 delle maggiori aziende svizzere, con una mappa interattiva per cantone, confronti per settore e fornitore di posta e una pagina di dettaglio per ogni azienda. Tra queste figurano anche le banche, gli assicuratori e i fornitori di servizi IT con cui collaborano gli istituti di previdenza.

La differenza è netta: delle 518 aziende nell'indice, il 18.10.2026 il 38 per cento utilizza DMARC con `p=reject`, mentre tra gli istituti di previdenza è il 20 per cento. Il 5 per cento delle aziende è completamente privo di DMARC, contro il 41 per cento degli istituti di previdenza.

## I fornitori sono il tema più importante

L'incidente di fine settembre non ha colpito una cassa, bensì un fornitore di servizi comune. Per questo ho verificato, con lo stesso metodo passivo, 30 fornitori tipici: produttori di software di gestione, aziende per la gestione esternalizzata, attuariato e consulenza, banche depositarie e fornitori di servizi IT.

In media, i fornitori di servizi si trovano in una posizione leggermente migliore rispetto alle casse: il 40 per cento utilizza DMARC con `p=reject`, contro il 20 per cento degli istituti. Determinante è però la distribuzione tra le categorie.

| Categoria | con DMARC `p=reject` |
|---|---|
| Attuariato e consulenza | buono |
| Banche depositarie | buono |
| Fornitori di servizi IT | misto |
| Gestione esternalizzata | debole |
| Software di gestione | i più deboli |

I produttori di software di gestione sono il gruppo più debole. Nessuno dei fornitori di software esaminati utilizza DMARC con piena efficacia, e il fornitore attaccato PK Softech non ha alcun record DMARC. Si tratta proprio della categoria che tratta dati previdenziali ed è profondamente collegata alle casse tramite interfacce, nonché della categoria coinvolta nell'incidente attuale.

Da ciò deriva il punto essenziale per la diligenza: una cassa con `p=reject` è ben protetta finché l'aggressore tenta di falsificare il suo dominio. Se invece può falsificare il dominio di un fornitore di servizi scarsamente protetto, attraverso il quale avviene comunque la comunicazione con gli assicurati, l'e-mail risulta altrettanto credibile. Proteggere il proprio dominio è quindi solo il primo passo. Il secondo consiste nell'obbligare i principali fornitori di servizi a raggiungere lo stesso livello, idealmente per contratto.

## Cosa significa per il consiglio di fondazione

La responsabilità per la sicurezza delle informazioni spetta all'organo supremo dell'istituto di previdenza. Una fuoriuscita di dati comporta inoltre obblighi concreti ai sensi della legge sulla protezione dei dati riveduta: una violazione della sicurezza dei dati deve essere notificata all'IFPDT se è presumibile che comporti un rischio elevato per le persone interessate, e le persone interessate devono essere informate nella misura necessaria alla loro protezione.

In questa situazione, l'autenticazione delle e-mail è la singola misura più efficace con il minor impegno. SPF, DKIM e DMARC non richiedono licenze né nuovo software. Consistono in record DNS e in una configurazione corretta della piattaforma di posta esistente. La protezione agisce immediatamente per ogni persona assicurata, senza che debba fare nulla.

## Cosa fare

Per gli istituti senza una protezione efficace, un percorso chiaro porta all'obiettivo:

1. **Verificare SPF e impostarlo in modo rigoroso.** Il record SPF elenca i sistemi autorizzati all'invio e termina con `-all`, affinché tutti gli altri mittenti siano considerati non autorizzati.
2. **Attivare DKIM.** Ogni messaggio in uscita viene firmato crittograficamente. In Microsoft 365 e nei comuni gateway di posta si tratta di un'impostazione, non di un progetto.
3. **Introdurre DMARC e renderlo progressivamente più restrittivo.** Iniziare con `p=none` e reporting attivo, per rilevare il traffico e-mail legittimo. Poi `p=quarantine`, infine `p=reject`. L'intero percorso richiede alcune settimane a seconda dell'ambiente, non mesi.
4. **Valutare il reporting.** I report DMARC mostrano chi invia e-mail a nome del dominio e rilevano sia sistemi legittimi dimenticati sia abusi.
5. **Aggiungere MTA-STS e TLS-RPT.** Entrambi impongono e monitorano la cifratura del trasporto delle e-mail in entrata.
6. **Responsabilizzare i fornitori di servizi.** I fornitori di software, la gestione esternalizzata e i partner IT dovrebbero dimostrare lo stesso livello in SPF, DKIM e DMARC. Ciò deve essere incluso nei contratti e nella verifica periodica dei fornitori.

## Conclusione

Gli attacchi recenti hanno preso di mira un fornitore di servizi, non le casse stesse. I danni conseguenti si verificano però laddove un dominio del mittente non è protetto, tanto presso la cassa quanto presso i suoi fornitori. Quattro dei cinque istituti di previdenza esaminati non sono completamente protetti dai mittenti falsificati, due su cinque non hanno affatto DMARC e proprio i produttori di software di gestione ottengono i risultati peggiori. La lacuna è nota, poco costosa da colmare e ha effetto immediato. Dopo una fuoriuscita di dati che riguarda nome, indirizzo e coordinate bancarie, non è una questione tecnica secondaria, bensì parte del dovere di diligenza dell'organo supremo, lungo l'intera catena di fornitura.

## Fonti

1.  [SRF: Cassa pensioni della Confederazione, come hanno potuto dati sensibili finire presso un'azienda esterna di software](https://www.srf.ch/news/schweiz/pensionskasse-des-bundes-wie-konnten-heikle-daten-bei-externer-softwarefirma-landen): Contesto dell'incidente, tipi di dati coinvolti e ruolo del fornitore di software.

2.  [cash.ch (AWP): Attacco a un fornitore di Publica, potrebbero essere stati coinvolti dati salariali](https://www.cash.ch/news/top-news/angriff-auf-publica-lieferanten-lohndaten-konnten-betroffen-sein-975247): Publica indica ad AWP PK Softech AG come azienda attaccata.

3.  [Netzwoche: La cassa pensioni della Confederazione Publica segnala una fuoriuscita di dati](https://www.netzwoche.ch/news/2026-10-08/bundespensionskasse-publica-meldet-datenabfluss): Notizia sull'attacco al fornitore di software e sulla fuoriuscita di dati.

4.  [Mimikama: La polizia svizzera avverte di possibili chiamate di investimento dopo una fuoriuscita di dati](https://www.mimikama.org/?p=436628): Contestualizzazione dell'avvertimento sulle chiamate fraudolente con i dati fuoriusciti.

5.  [IFPDT: Notifica di violazioni della sicurezza dei dati](https://www.edoeb.admin.ch/de/meldung-von-verletzungen-der-datensicherheit): Obbligo di notifica secondo la LPD riveduta e procedura in caso di violazioni della sicurezza dei dati.
