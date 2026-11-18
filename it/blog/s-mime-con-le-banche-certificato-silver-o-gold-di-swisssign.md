---
title: "S/MIME nelle banche: quadro giuridico in Svizzera e scelta tra Silver e Gold"
navTitle: "S/MIME Silver/Gold"
description: "Quali requisiti giuridici si applicano quando una banca svizzera firma e cifra e-mail ai clienti: segreto bancario, Circolare FINMA 2023/1, LPD, ZertES, diritto probatorio e conservazione. Inoltre, quale livello di certificato SwissSign ne deriva."
date: "2026-11-16"
kategorie: "Crittografia e-mail"
timeToRead: "12 min di lettura"
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
translationSourceHash: f4b52127b6883bb31850668cc18a27ebba43f75200bfdc605d199f427d602ddf
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T03:26:28.212Z
translationReview: required
url: https://rafaelpfister.ch/it/blog/s-mime-con-le-banche-certificato-silver-o-gold-di-swisssign
---

# S/MIME nelle banche: quadro giuridico in Svizzera e scelta tra Silver e Gold

In sintesi: nessuna norma svizzera impone a una banca di utilizzare S/MIME o un determinato livello di certificato. Il segreto bancario, la Circolare FINMA 2023/1 e la legge sulla protezione dei dati richiedono tuttavia che i dati dei clienti siano adeguatamente protetti durante l'invio e che la banca sia in grado di motivare e documentare la propria soluzione. Una firma S/MIME non sostituisce una firma autografa, neppure con un certificato Gold. Come mezzo di prova e attestazione dell'origine, Gold è nettamente superiore al certificato Silver, poiché soltanto Gold contiene il nome verificato della banca.

La questione di quale certificato SwissSign necessiti una banca per l'invio di e-mail ricorre regolarmente nei miei progetti con istituti finanziari. Dietro alla questione tecnica vi è generalmente una questione giuridica: che cosa deve fare la banca per adempiere ai propri obblighi nell'invio di dati dei clienti via e-mail e quale effetto produce un'e-mail firmata?

**Focus Svizzera:** questo articolo tratta il quadro giuridico in Svizzera. Qualora le norme UE divergano, viene riportata un'apposita nota. Non sostituisce una consulenza giuridica nel singolo caso.

## Panoramica delle basi giuridiche

| Normativa | Contenuto | Rilevanza per l'invio di e-mail |
|---|---|---|
| LBCR art. 47 | Segreto bancario, penalmente sanzionato anche in caso di negligenza | I dati dei clienti non possono essere divulgati senza autorizzazione |
| Circ. FINMA 2023/1, marg. 73-82 | Protezione dei dati critici durante l'intero ciclo di vita | Protezione anche durante l'invio, neutrale dal punto di vista tecnologico |
| LPD art. 8 e 24, OPDa art. 1-6 | Sicurezza dei dati, notifica di violazioni | Misure adeguate, notifica all'IFPDT in caso di rischio elevato |
| ZertES e CO art. 14 cpv. 2bis | Tipi di firme elettroniche, validità formale | Le firme S/MIME non sostituiscono una firma autografa |
| CPC art. 157 e 177 | Mezzi di prova e libero apprezzamento delle prove | Le e-mail firmate sono documenti il cui valore probatorio è liberamente apprezzato |
| CO art. 958f, OGeB | Conservazione dei documenti aziendali | Le e-mail cifrate devono rimanere leggibili |

Nessuna di queste norme menziona S/MIME o un livello di certificato. Esse stabiliscono obiettivi di protezione e conseguenze giuridiche; spetta alla banca decidere come raggiungerli.

## Segreto bancario

L'art. 47 LBCR punisce chi, in qualità di organo, impiegato o incaricato di una banca, divulga un segreto affidatogli in tale qualità. È punibile anche la commissione per negligenza. La norma si rivolge alle persone fisiche; alla sua base vi è tuttavia il rapporto contrattuale tra la cliente o il cliente, quale titolare del segreto, e la banca.

Per l'invio di e-mail questo significa che un'e-mail non cifrata con dati dei clienti, inviata a un indirizzo errato o letta durante il transito, può costituire una divulgazione negligente. La banca deve pertanto adottare misure organizzative e tecniche affinché i collaboratori inviino dati dei clienti soltanto in modo protetto.

Il cliente, quale titolare del segreto, può acconsentire alla divulgazione. Per questo motivo molte banche si fanno confermare nelle condizioni generali o in un accordo separato che il cliente conosce e desidera i rischi della comunicazione non cifrata via e-mail. Un tale consenso copre soltanto la comunicazione con quel cliente e soltanto nella misura concordata. Per documenti che riguardano terzi o per la comunicazione con partner commerciali, non sostituisce la cifratura.

## Circolare FINMA 2023/1: dati critici

La Circolare FINMA 2023/1 «Rischi operativi e resilienza: banche» è in vigore dal 1° gennaio 2024 e ha sostituito la Circolare 2008/21. La FINMA ha esteso l'attenzione precedentemente posta sulla riservatezza dei dati di identificazione dei clienti (CID) ai dati critici in generale. Ogni banca determina autonomamente, in base al rischio, che cosa è critico; di norma i dati dei clienti vi rientrano sotto il profilo della riservatezza.

Per l'invio di e-mail sono rilevanti soprattutto i seguenti numeri marginali:

| N. marg. | Contenuto | Riferimento all'invio di e-mail |
|---|---|---|
| 73 | Identificare i dati critici e classificarli in base alla criticità | Stabilire quali dati possono essere inviati soltanto in forma protetta |
| 74, 75 | Garantire riservatezza, integrità e disponibilità durante l'intero ciclo di vita | L'invio fa parte del ciclo di vita |
| 76 | Proteggere i dati critici dall'accesso di persone non autorizzate | Protezione dell'e-mail durante il trasporto e presso il destinatario |
| 79 | Limitare i rischi elevati in caso di conservazione o accesso dall'estero | Riguarda gateway e piattaforme e-mail ubicati all'estero |
| 81 | Notificare senza indugio alla FINMA gli incidenti rilevanti | Ad esempio, un invio massivo di dati dei clienti a destinatari errati |
| 82 | Selezionare e sorvegliare accuratamente i prestatori di servizi che trattano dati critici | Riguarda i fornitori di gateway di cifratura e le autorità di certificazione con PKI gestita |

La circolare è neutrale dal punto di vista tecnologico e non menziona né S/MIME né livelli di certificato. Nelle sue spiegazioni relative alla consultazione, la FINMA cita la cifratura dei dati come esempio di misura speciale per proteggere i dati critici. La banca deve definire la propria soluzione in direttive interne e indicare anche quale livello di certificato utilizza e perché.

## Protezione dei dati: LPD e OPDa

L'art. 8 LPD obbliga i titolari del trattamento a garantire una sicurezza dei dati adeguata al rischio. L'ordinanza sulla protezione dei dati concretizza tale obbligo mediante obiettivi di protezione quali riservatezza, integrità e tracciabilità, senza tuttavia prescrivere una procedura. S/MIME, un gateway con consegna tramite portale web o una piattaforma di scambio soddisfano parimenti il requisito se utilizzati in modo adeguato al rischio.

Se una violazione della sicurezza dei dati comporta presumibilmente un rischio elevato per la personalità o i diritti fondamentali delle persone interessate, la banca deve notificarla all'IFPDT quanto prima ai sensi dell'art. 24 LPD. Tale obbligo di notifica si aggiunge a quello nei confronti della FINMA. Se l'e-mail era cifrata, in caso di invio errato il rischio è generalmente basso e non occorre effettuare una notifica.

## Effetti giuridici di una firma S/MIME

### Nessuna firma ai sensi del CO

La ZertES distingue tra firma elettronica, firma elettronica avanzata, firma elettronica regolamentata e firma elettronica qualificata, nonché, per le imprese, sigillo elettronico regolamentato. Ai sensi dell'art. 14 cpv. 2bis CO, soltanto la firma elettronica qualificata con marca temporale qualificata è equiparata alla firma autografa.

I certificati S/MIME di SwissSign vengono emessi secondo le regole del CA/Browser Forum e le policy ETSI LCP (Silver) e NCP (Gold). Non sono certificati qualificati ai sensi della ZertES. Pertanto una banca non può rendere in forma valida tramite e-mail firmata S/MIME dichiarazioni per le quali la legge o un contratto richiedono la forma scritta, neppure con Gold.

### Firma personale e firma del gateway

Una firma avanzata presuppone, tra l'altro, che la chiave di firma sia sotto il controllo esclusivo della persona firmataria. Se un gateway firma centralmente tutte le e-mail in uscita con chiavi situate sul gateway, tale presupposto non è soddisfatto. La firma attesta allora che l'e-mail proviene dal sistema della banca e non è stata modificata, ma non che una determinata persona l'abbia approvata. Nel contenuto equivale quindi piuttosto a un sigillo dell'organizzazione, senza soddisfare i requisiti di un sigillo regolamentato ai sensi della ZertES.

### Valore probatorio

Ai sensi dell'art. 177 CPC, un'e-mail è un documento e il giudice ne apprezza liberamente il valore probatorio ai sensi dell'art. 157 CPC. Una firma S/MIME valida mostra che l'e-mail non è stata modificata dalla firma e che è stata firmata con la chiave di un determinato certificato. Quanto ciò dica sul mittente dipende da cosa l'autorità di certificazione abbia verificato prima dell'emissione. È qui che risiede la differenza giuridicamente rilevante tra Silver e Gold.

## Silver o Gold dal punto di vista giuridico

Dal 1° settembre 2023, per i certificati S/MIME pubblicamente attendibili si applicano i `S/MIME Baseline Requirements` del CA/Browser Forum. SwissSign assegna i propri prodotti a questi tipi di validazione nei profili dei certificati (CPR S/MIME):

| Prodotto | Tipo secondo il CA/B Forum | Cosa verifica SwissSign | Cosa figura nel certificato | Cosa attesta la firma |
|---|---|---|---|---|
| E-Mail ID Silver | `Mailbox-validated` | Controllo sulla casella e-mail | Solo l'indirizzo e-mail | Qualcuno con accesso a questa casella e-mail |
| E-Mail ID Gold | `Sponsor-validated` | Organizzazione e persona | Nome, organizzazione, identificativo dell'organizzazione, Paese, indirizzo e-mail | Una persona verificata di questa banca |
| Organization Validated | `Organization-validated` | Organizzazione | Nome dell'organizzazione, nessuna persona | Una casella e-mail di questa banca |

Dal punto di vista crittografico, i livelli sono equivalenti. Per la cifratura e quindi per gli obblighi di protezione derivanti dal segreto bancario, dalla Circolare FINMA e dalla LPD, il livello non ha pertanto alcuna rilevanza.

Per la firma, la differenza è considerevole. Un certificato Silver viene rilasciato a chiunque possa ricevere e-mail a un indirizzo. Chi registra un dominio dal nome simile, ad esempio `musterbank-ch.com` anziché `musterbank.ch`, può così inviare e-mail firmate validamente quanto le e-mail autentiche della banca. In una controversia sull'effettiva provenienza di un'e-mail dalla banca, una firma Silver prova quindi poco. Un certificato Gold o Organization Validated contiene invece il nome della banca verificato da SwissSign. Con Gold si aggiunge l'attribuzione a una persona, la cui identità è confermata dalla banca in qualità di organismo di registrazione nell'ambito della PKI gestita.

## Conservazione: CO e OGeB

Ai sensi dell'art. 958f CO, i libri di commercio e i documenti giustificativi devono essere conservati per dieci anni e l'ordinanza sulla tenuta e la conservazione dei libri di commercio (OGeB) richiede che i documenti conservati elettronicamente possano essere resi leggibili durante tale periodo. Per le banche si aggiungono obblighi di conservazione di natura prudenziale e contrattuale. Da ciò derivano due requisiti per l'esercizio di S/MIME:

- Le e-mail cifrate devono poter essere decifrate per l'intero periodo di conservazione. La banca deve pertanto conservare in modo sicuro le chiavi private dei certificati scaduti oppure decifrare le e-mail prima dell'archiviazione.
- Se una firma deve ancora fungere da prova dopo anni, deve rimanere verificabile che il certificato fosse valido al momento della firma. A tal fine, i risultati della verifica o le informazioni di revoca al momento della firma devono essere archiviati.

**Nota UE:** nell'UE, ai sensi dell'art. 25 par. 2 eIDAS, soltanto la firma elettronica qualificata è equiparata alla firma autografa. Per le imprese finanziarie nell'UE, dal 17 gennaio 2025 si applica inoltre il regolamento DORA, con propri requisiti per la gestione dei rischi ICT.

## Esempio: un gateway per tutti i collaboratori

Molte banche utilizzano un gateway di cifratura centrale, come PrivaSphere, SEPPmail o Totemomail, affinché tutti i collaboratori possano cifrare e firmare e-mail senza dover gestire personalmente i certificati. Le chiavi sono collocate sul gateway e il gateway firma e cifra per conto delle singole caselle e-mail. Da quanto precede risulta la seguente scelta per questa configurazione:

| Casella e-mail | Livello | Motivo |
|---|---|---|
| Caselle e-mail personali | E-Mail ID Gold | Identifica la banca come mittente mediante nome e identificativo dell'organizzazione verificati |
| Caselle e-mail funzionali (servizio clienti, traffico pagamenti) | Organization Validated | Identifica la banca senza menzionare una singola persona |
| Nessuna | E-Mail ID Silver | Attesta soltanto l'accesso a una casella e-mail; le e-mail provenienti da domini dal nome simile sono firmate in modo altrettanto valido |

Poiché firma il gateway e non la singola persona, le chiavi non sono sotto il controllo esclusivo dei collaboratori. La firma ha giuridicamente l'effetto di un sigillo della banca e non di una firma personale. È pertanto determinante che l'organizzazione sia indicata nel certificato come verificata, cosa garantita soltanto da Gold e Organization Validated. Per la cifratura stessa, il livello non è rilevante.

Prima dell'introduzione, occorre chiarire i seguenti punti con il fornitore del gateway e l'autorità di certificazione:

- **Emissione e rinnovo:** se il gateway ottiene e rinnova automaticamente certificati Gold tramite la PKI gestita dell'autorità di certificazione. Con Gold, la banca conferma l'identità di ogni persona in qualità di organismo di registrazione; è quindi necessario un processo per assunzioni e uscite, cambi di nome e revoche.
- **Verifica dei prestatori di servizi:** il fornitore del gateway e la PKI gestita trattano dati critici ai sensi del n. marg. 82 della Circolare FINMA 2023/1. L'ubicazione delle chiavi e dei dati deve essere documentata; in caso di conservazione all'estero o accesso dall'estero, si applica inoltre il n. marg. 79.
- **Archiviazione:** le chiavi private dei certificati scaduti e i dati di verifica relativi al momento della firma devono rimanere disponibili per l'intero periodo di conservazione.
- **Direttiva:** i livelli scelti e le procedure autorizzate devono essere inclusi nella direttiva interna, che serve come attestazione nei confronti della FINMA e della revisione.

## Raccomandazione

- **Definire una direttiva:** la banca stabilisce in una direttiva interna quali dati possono essere inviati soltanto in forma cifrata, quali procedure sono ammesse a tale scopo e quali livelli di certificato utilizza. Questa è la base per dimostrare i requisiti della Circolare FINMA e della LPD.
- **Gold per caselle e-mail personali, Organization Validated per caselle e-mail funzionali:** soltanto questi livelli identificano la banca nel certificato come mittente e hanno quindi un valore probatorio che va oltre l'accesso a una casella e-mail. Silver non è adatto alla comunicazione con i clienti.
- **Chiarire i consensi:** chi comunica con i clienti via e-mail non cifrata necessita del loro consenso documentato. Esso non sostituisce la cifratura nei confronti di terzi e partner commerciali.
- **Non inviare via e-mail dichiarazioni soggette a forma:** per dichiarazioni che richiedono la forma scritta, è necessaria una firma elettronica qualificata o una firma autografa.
- **Verificare i prestatori di servizi:** il fornitore del gateway e la PKI gestita dell'autorità di certificazione rientrano nel n. marg. 82 della Circolare FINMA; per sedi all'estero, anche nel n. marg. 79.
- **Archiviare chiavi e dati di verifica:** le chiavi private per la decifratura e i dati per la verifica della firma devono rimanere disponibili per il periodo di conservazione.
- **Collegare i processi di notifica:** un invio errato di dati dei clienti può attivare contemporaneamente una notifica alla FINMA (n. marg. 81) e all'IFPDT (art. 24 LPD). Entrambi gli obblighi possono essere gestiti in un unico processo per gli incidenti.

## Fonti

1.  [Circolare FINMA 2023/1 «Rischi operativi e resilienza: banche»](https://www.finma.ch/de/~/media/finma/dokumente/dokumentencenter/myfinma/rundschreiben/finma-rs-2023-01-20221207.pdf): n. marg. 73-82 sui dati critici, in vigore dal 1° gennaio 2024.

2.  [datenrecht.ch: FINMA-RS 2023/1 con spiegazioni](https://datenrecht.ch/wp-content/uploads/FINMA-RS-OpRisk-%E2%80%93-Text-revidiert-1.1.2024.pdf): testo rivisto con le osservazioni della FINMA emerse dalla consultazione, tra l'altro sulla cifratura dei dati quale misura di protezione.

3.  [Legge sulle banche (LBCR)](https://www.fedlex.admin.ch/eli/cc/51/117_121_129/de): art. 47 sul segreto bancario.

4.  [Swissbanking: segreto bancario e cloud, parere 2019](https://www.swissbanking.ch/_Resources/Persistent/b/1/3/3/b13339fe5dcd344e715dbe0a899afe3c8c56ed0b/SBVg_Bankgeheimnis_und_Cloud_Gutachten_LLAG_2019_DE.pdf): parere giuridico sull'interpretazione dell'art. 47 LBCR, tra l'altro sul consenso del cliente.

5.  [Legge federale sulla protezione dei dati (LPD)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): art. 8 sulla sicurezza dei dati, art. 24 sulla notifica delle violazioni della sicurezza dei dati.

6.  [Ordinanza sulla protezione dei dati (OPDa)](https://www.fedlex.admin.ch/eli/cc/2022/568/de): art. 1-6 con principi e obiettivi di protezione della sicurezza dei dati.

7.  [Legge federale sulla firma elettronica (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): tipi di firme elettroniche e sigilli, requisiti per i certificati qualificati.

8.  [Codice delle obbligazioni (CO)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): art. 14 cpv. 2bis sull'equiparazione della firma elettronica qualificata alla firma autografa, art. 958f sulla conservazione.

9.  [Codice di procedura civile (CPC)](https://www.fedlex.admin.ch/eli/cc/2010/262/de): art. 157 sul libero apprezzamento delle prove, art. 177 sulla nozione di documento.

10.  [SwissSign CPR S/MIME: Certificate, CRL and OCSP Profiles for S/MIME Certificates](https://repository.swisssign.com/SwissSign_CPR_SMIME_R14.pdf): profili di certificato di SwissSign (aggiornati al 18.09.2025), assegnazione di Silver, Gold e Organization Validated ai tipi di validazione e campi Subject.

11.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): regole vincolanti per certificati S/MIME pubblicamente attendibili, definizione dei tipi di validazione.
