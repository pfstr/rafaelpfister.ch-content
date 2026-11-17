---
title: "S/MIME nelle banche: quadro giuridico in Svizzera e scelta tra Silver e Gold"
navTitle: "S/MIME Silver/Gold"
description: "Quali requisiti giuridici si applicano quando una banca svizzera firma e crittografa e-mail ai clienti: segreto bancario, Circolare FINMA 2023/1, LPD, ZertES, diritto probatorio e conservazione. Inoltre, quale livello di certificato SwissSign ne deriva."
date: "2026-11-16"
kategorie: "Crittografia e-mail"
timeToRead: "11 min di lettura"
themen:
  - e-mail-verschluesselung
produkte: []
protokolle:
  - "verschluesselung"
  - "tls"
hauptthema: "e-mail-verschluesselung"
startseiteAb: "2099-12-31"
slug: "s-mime-con-le-banche-certificato-silver-o-gold-di-swisssign"
translationId: "article-5c2fd8d42f088dcb"
translationOf: smime-silver-gold-schweiz
translationSourceHash: 60de430124ebe20a3001bd48fc55affcb921141039e7c85303da9c7216b01e89
translationModel: gpt-5.6-terra
translatedAt: 2026-11-16T15:21:07.305Z
translationReview: required
url: https://rafaelpfister.ch/it/blog/s-mime-con-le-banche-certificato-silver-o-gold-di-swisssign
---

# S/MIME nelle banche: quadro giuridico in Svizzera e scelta tra Silver e Gold

In sintesi: nessuna norma svizzera impone a una banca l'uso di S/MIME o di un determinato livello di certificato. Tuttavia, il segreto bancario, la Circolare FINMA 2023/1 e la legge sulla protezione dei dati richiedono che i dati dei clienti siano adeguatamente protetti durante l'invio, e la banca deve poter motivare e documentare la propria soluzione. Una firma S/MIME non sostituisce una firma autografa, neppure con un certificato Gold. Come mezzo di prova e attestazione della provenienza, Gold è nettamente superiore al certificato Silver, poiché solo Gold contiene il nome verificato della banca.

La domanda su quale certificato SwissSign sia necessario a una banca per l'invio di e-mail ricorre regolarmente nei miei progetti con istituti finanziari. Dietro la questione tecnica vi è solitamente una questione giuridica: cosa deve fare la banca per adempiere ai propri obblighi nell'invio di dati dei clienti via e-mail e quale effetto produce un'e-mail firmata?

**Focus Svizzera:** questo articolo tratta il quadro giuridico in Svizzera. Laddove le norme UE differiscono, è presente un'avvertenza contrassegnata. Non sostituisce una consulenza giuridica nel singolo caso.

## Panoramica delle basi giuridiche

| Norma | Contenuto | Rilevanza per l'invio di e-mail |
|---|---|---|
| LBCR art. 47 | Segreto bancario, penalmente sanzionato anche in caso di negligenza | I dati dei clienti non possono essere divulgati senza autorizzazione |
| Circ.-FINMA 2023/1, n. marg. 73-82 | Protezione dei dati critici durante l'intero ciclo di vita | Protezione anche durante l'invio, neutrale dal punto di vista tecnologico |
| LPD art. 8 e 24, OPDa art. 1-6 | Sicurezza dei dati, notifica delle violazioni | Misure adeguate, notifica all'IFPDT in caso di rischio elevato |
| ZertES e CO art. 14 cpv. 2bis | Tipi di firme elettroniche, validità formale | Le firme S/MIME non sostituiscono una firma autografa |
| CPC art. 157 e 177 | Mezzi di prova e libera valutazione delle prove | Le e-mail firmate sono titoli con forza probatoria liberamente valutabile |
| CO art. 958f, OGeB | Conservazione dei documenti aziendali | Le e-mail crittografate devono rimanere leggibili |

Nessuna di queste norme menziona S/MIME o un livello di certificato. Esse definiscono obiettivi di protezione e conseguenze giuridiche; la banca decide autonomamente come raggiungerli.

## Segreto bancario

L'art. 47 LBCR punisce chiunque, quale organo, impiegato o mandatario di una banca, divulghi un segreto affidatogli in tale qualità. È punibile anche la commissione per negligenza. La norma si rivolge alle persone fisiche; alla base vi è tuttavia il rapporto contrattuale tra la cliente o il cliente, quale titolare del segreto, e la banca.

Per l'invio di e-mail ciò significa che un'e-mail non crittografata contenente dati dei clienti, inviata a un indirizzo errato o letta durante il transito, può costituire una divulgazione per negligenza. La banca deve pertanto garantire, con misure organizzative e tecniche, che i collaboratori inviino i dati dei clienti solo in forma protetta.

Il cliente, quale titolare del segreto, può acconsentire alla divulgazione. Molte banche fanno quindi confermare nelle CG o in un accordo separato che il cliente conosce i rischi della comunicazione non crittografata via e-mail e la desidera. Un tale consenso copre soltanto la comunicazione con quel cliente e solo nella misura concordata. Per documenti che riguardano terzi o per la comunicazione con partner commerciali, non sostituisce la crittografia.

## Circolare FINMA 2023/1: dati critici

La Circolare FINMA 2023/1 «Rischi operativi e resilienza: banche» è in vigore dal 1° gennaio 2024 e ha sostituito la Circolare 2008/21. La FINMA ha esteso il precedente focus sulla riservatezza dei dati di identificazione dei clienti (CID) ai dati critici in generale. Ogni banca stabilisce autonomamente, in base al rischio, quali dati siano critici; di norma i dati dei clienti rientrano in questa categoria per quanto riguarda la riservatezza.

Per l'invio di e-mail sono particolarmente rilevanti i seguenti numeri marginali:

| N. marg. | Contenuto | Riferimento all'invio di e-mail |
|---|---|---|
| 73 | Identificare i dati critici e classificarli secondo la criticità | Stabilire quali dati possono essere inviati solo in forma protetta |
| 74, 75 | Garantire riservatezza, integrità e disponibilità durante l'intero ciclo di vita | L'invio fa parte del ciclo di vita |
| 76 | Proteggere i dati critici dall'accesso di persone non autorizzate | Protezione dell'e-mail durante il trasporto e presso il destinatario |
| 79 | Limitare i rischi maggiori in caso di conservazione o accesso dall'estero | Riguarda gateway e piattaforme e-mail situati all'estero |
| 81 | Notificare immediatamente alla FINMA gli incidenti significativi | Ad esempio, un invio di massa di dati dei clienti a destinatari errati |
| 82 | Selezionare e sorvegliare accuratamente i fornitori di servizi che trattano dati critici | Riguarda i fornitori di gateway di crittografia e le autorità di certificazione con Managed PKI |

La circolare è neutrale dal punto di vista tecnologico e non menziona né S/MIME né livelli di certificato. Nelle sue spiegazioni relative alla consultazione, la FINMA cita la crittografia dei dati come esempio di misura speciale per proteggere i dati critici. La banca deve definire la propria soluzione in direttive interne e indicare anche quale livello di certificato utilizza e perché.

## Protezione dei dati: LPD e OPDa

L'art. 8 LPD obbliga i titolari del trattamento a garantire una sicurezza dei dati adeguata al rischio. L'ordinanza sulla protezione dei dati concretizza tale obbligo mediante obiettivi di protezione quali riservatezza, integrità e tracciabilità, ma non prescrive alcuna procedura. S/MIME, un gateway con recapito tramite portale web o una piattaforma di scambio soddisfano il requisito allo stesso modo, se utilizzati in modo adeguato al rischio.

Se una violazione della sicurezza dei dati comporta presumibilmente un rischio elevato per la personalità o i diritti fondamentali delle persone interessate, la banca deve notificarla all'IFPDT il più rapidamente possibile ai sensi dell'art. 24 LPD. Tale obbligo di notifica si aggiunge a quello nei confronti della FINMA. Se l'e-mail era crittografata, il rischio in caso di invio errato è di norma basso e non è necessaria una notifica.

## Effetti giuridici di una firma S/MIME

### Nessuna firma ai sensi del CO

La ZertES distingue tra firma elettronica, firma elettronica avanzata, firma elettronica regolamentata e firma elettronica qualificata, nonché, per le imprese, sigillo elettronico regolamentato. Ai sensi dell'art. 14 cpv. 2bis CO, solo la firma elettronica qualificata con marca temporale qualificata è equiparata alla firma autografa.

I certificati S/MIME di SwissSign sono emessi secondo le regole del CA/Browser Forum e le policy ETSI LCP (Silver) o NCP (Gold). Non sono certificati qualificati ai sensi della ZertES. Pertanto, una banca non può rendere validamente in forma scritta, tramite e-mail firmata S/MIME, dichiarazioni per le quali la legge o un contratto richiedono la forma scritta, neppure con Gold.

### Firma personale e firma del gateway

Una firma avanzata presuppone tra l'altro che la chiave di firma sia sotto il controllo esclusivo della persona firmataria. Se un gateway firma centralmente tutte le e-mail in uscita con chiavi memorizzate sul gateway, questo requisito non è soddisfatto. La firma dimostra allora che l'e-mail proviene dal sistema della banca e non è stata modificata, ma non che una persona specifica l'abbia autorizzata. Dal punto di vista del contenuto, corrisponde quindi più a un sigillo dell'organizzazione, senza soddisfare i requisiti di un sigillo regolamentato ai sensi della ZertES.

### Valore probatorio

Ai sensi dell'art. 177 CPC, un'e-mail è un titolo, e il tribunale ne valuta liberamente la forza probatoria ai sensi dell'art. 157 CPC. Una firma S/MIME valida dimostra che l'e-mail non è stata modificata dalla firma e che è stata firmata con la chiave di un determinato certificato. Quanto ciò dica sul mittente dipende da ciò che l'autorità di certificazione ha verificato prima dell'emissione. È qui che risiede la differenza giuridicamente rilevante tra Silver e Gold.

## Silver o Gold dal punto di vista giuridico

Dal 1° settembre 2023, per i certificati S/MIME pubblicamente attendibili si applicano i `S/MIME Baseline Requirements` del CA/Browser Forum. SwissSign assegna i propri prodotti, nei profili di certificato (CPR S/MIME), a questi tipi di convalida:

| Prodotto | Tipo secondo il CA/B Forum | Cosa verifica SwissSign | Cosa figura nel certificato | Cosa attesta la firma |
|---|---|---|---|---|
| E-Mail ID Silver | `Mailbox-validated` | Controllo della casella di posta | Solo l'indirizzo e-mail | Qualcuno con accesso a questa casella di posta |
| E-Mail ID Gold | `Sponsor-validated` | Organizzazione e persona | Nome, organizzazione, identificativo dell'organizzazione, Paese, indirizzo e-mail | Una persona verificata di questa banca |
| Organization Validated | `Organization-validated` | Organizzazione | Nome dell'organizzazione, nessuna persona | Una casella di posta di questa banca |

Dal punto di vista crittografico, i livelli sono equivalenti. Per la crittografia e quindi per gli obblighi di protezione derivanti dal segreto bancario, dalla Circolare FINMA e dalla LPD, il livello non ha quindi alcuna rilevanza.

Per la firma, la differenza è significativa. Un certificato Silver viene rilasciato a chiunque possa ricevere e-mail a un indirizzo. Chi registra un dominio dal nome simile, ad esempio `musterbank-ch.com` anziché `musterbank.ch`, può così inviare e-mail firmate in modo altrettanto valido quanto le e-mail autentiche della banca. In una controversia sulla provenienza di un'e-mail dalla banca, una firma Silver dimostra quindi poco. Un certificato Gold o Organization Validated contiene invece il nome della banca verificato da SwissSign. Con Gold si aggiunge l'attribuzione a una persona, la cui identità è confermata dalla banca quale organismo di registrazione nell'ambito della Managed PKI.

## Conservazione: CO e OGeB

Ai sensi dell'art. 958f CO, i libri di commercio e i documenti contabili devono essere conservati per dieci anni, e l'ordinanza sulla conservazione dei libri di commercio (OGeB) richiede che i documenti conservati elettronicamente possano essere resi leggibili durante tale periodo. Per le banche si aggiungono obblighi di conservazione di natura prudenziale e contrattuale. Ne derivano due requisiti per l'esercizio di S/MIME:

- Le e-mail crittografate devono poter essere decrittografate per l'intero periodo di conservazione. La banca deve quindi conservare in modo sicuro le chiavi private dei certificati scaduti oppure decrittografare le e-mail prima dell'archiviazione.
- Se una firma deve ancora servire come prova dopo anni, deve rimanere verificabile che il certificato fosse valido al momento della firma. A tale scopo, i risultati della verifica o le informazioni di revoca al momento della firma devono essere archiviati insieme.

**Nota UE:** nell'UE, ai sensi dell'art. 25 par. 2 eIDAS, solo la firma elettronica qualificata è equiparata alla firma autografa. Per le imprese finanziarie nell'UE, dal 17 gennaio 2025 si applica inoltre il regolamento DORA con propri requisiti per la gestione del rischio ICT.

## Raccomandazione

- **Definire una direttiva:** la banca stabilisce in una direttiva interna quali dati possono essere inviati solo in forma crittografata, quali procedure sono autorizzate a tale scopo e quali livelli di certificato utilizza. Questa è la base per dimostrare la conformità ai requisiti della Circolare FINMA e della LPD.
- **Gold per le caselle personali, Organization Validated per le caselle funzionali:** solo questi livelli identificano la banca nel certificato quale mittente e hanno quindi un valore probatorio che va oltre l'accesso a una casella di posta. Silver non è adatto alla comunicazione con i clienti.
- **Chiarire i consensi:** chi comunica con i clienti via e-mail non crittografata necessita del loro consenso documentato. Esso non sostituisce la crittografia nei confronti di terzi e partner commerciali.
- **Non inviare via e-mail dichiarazioni soggette a forma:** per le dichiarazioni che richiedono la forma scritta, occorre una firma elettronica qualificata o una firma autografa.
- **Verificare i fornitori di servizi:** i fornitori di gateway e la Managed PKI dell'autorità di certificazione rientrano nel n. marg. 82 della Circolare FINMA e, in caso di sedi all'estero, anche nel n. marg. 79.
- **Archiviare chiavi e dati di verifica:** le chiavi private per la decrittografia e i dati per la verifica della firma devono rimanere disponibili per il periodo di conservazione.
- **Collegare i processi di notifica:** un invio errato di dati dei clienti può comportare contemporaneamente una notifica alla FINMA (n. marg. 81) e all'IFPDT (art. 24 LPD). Entrambi gli obblighi possono essere gestiti in un processo comune per gli incidenti.

## Fonti

1.  [Circolare FINMA 2023/1 «Rischi operativi e resilienza: banche»](https://www.finma.ch/de/~/media/finma/dokumente/dokumentencenter/myfinma/rundschreiben/finma-rs-2023-01-20221207.pdf): n. marg. 73-82 sui dati critici, in vigore dal 1° gennaio 2024.

2.  [datenrecht.ch: Circ.-FINMA 2023/1 con spiegazioni](https://datenrecht.ch/wp-content/uploads/FINMA-RS-OpRisk-%E2%80%93-Text-revidiert-1.1.2024.pdf): testo rivisto con le osservazioni della FINMA emerse dalla consultazione, tra cui quelle sulla crittografia dei dati quale misura di protezione.

3.  [Legge sulle banche (LBCR)](https://www.fedlex.admin.ch/eli/cc/51/117_121_129/de): art. 47 sul segreto bancario.

4.  [Swissbanking: segreto bancario e cloud, parere giuridico 2019](https://www.swissbanking.ch/_Resources/Persistent/b/1/3/3/b13339fe5dcd344e715dbe0a899afe3c8c56ed0b/SBVg_Bankgeheimnis_und_Cloud_Gutachten_LLAG_2019_DE.pdf): parere giuridico sull'interpretazione dell'art. 47 LBCR, tra l'altro sul consenso del cliente.

5.  [Legge federale sulla protezione dei dati (LPD)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): art. 8 sulla sicurezza dei dati, art. 24 sulla notifica delle violazioni della sicurezza dei dati.

6.  [Ordinanza sulla protezione dei dati (OPDa)](https://www.fedlex.admin.ch/eli/cc/2022/568/de): art. 1-6 con principi e obiettivi di protezione della sicurezza dei dati.

7.  [Legge federale sulla firma elettronica (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): tipi di firme elettroniche e sigilli, requisiti per i certificati qualificati.

8.  [Codice delle obbligazioni (CO)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): art. 14 cpv. 2bis sull'equiparazione della firma elettronica qualificata alla firma autografa, art. 958f sulla conservazione.

9.  [Codice di procedura civile (CPC)](https://www.fedlex.admin.ch/eli/cc/2010/262/de): art. 157 sulla libera valutazione delle prove, art. 177 sulla nozione di titolo.

10.  [SwissSign CPR S/MIME: Certificate, CRL and OCSP Profiles for S/MIME Certificates](https://repository.swisssign.com/SwissSign_CPR_SMIME_R14.pdf): profili di certificato SwissSign (aggiornati al 18.09.2025), attribuzione di Silver, Gold e Organization Validated ai tipi di convalida e campi Subject.

11.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): regole vincolanti per certificati S/MIME pubblicamente attendibili, definizione dei tipi di convalida.
