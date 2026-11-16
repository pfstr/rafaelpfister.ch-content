---
title: "S/MIME con le banche: certificato Silver o Gold di SwissSign?"
navTitle: "S/MIME Silver/Gold"
description: "Per le e-mail crittografate con una banca, un certificato Silver di SwissSign è tecnicamente sufficiente. Gold conferma inoltre il nome e l'organizzazione del mittente. Quale livello verifica cosa, cosa contiene il certificato e quali requisiti legali vigono in Svizzera."
date: "2026-11-16"
kategorie: "Crittografia delle e-mail"
timeToRead: "8 min di lettura"
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
url: https://rafaelpfister.ch/it/blog/s-mime-con-le-banche-certificato-silver-o-gold-di-swisssign
translationSourceHash: cbcb890de323f2b9db40a5682192296524250e7bc8c23a67c6f643bcba605a14
translationModel: gpt-5.6-terra
translatedAt: 2026-11-16T14:04:11.522Z
translationReview: required
---

# S/MIME con le banche: certificato Silver o Gold di SwissSign?

In breve: né una legge svizzera né la FINMA prescrivono un determinato livello di certificato per S/MIME. Per crittografare, un certificato Silver è tecnicamente del tutto sufficiente. Gold conferma inoltre il nome e l'organizzazione del mittente ed è la scelta appropriata se la banca non fornisce indicazioni.

La domanda se per lo scambio di e-mail con una banca sia necessario un certificato Silver o Gold di SwissSign ricorre regolarmente nei miei progetti. Di solito la banca non dispone di una scheda informativa al riguardo e la consulente o il consulente alla clientela non sa rispondere alla domanda. La risposta dipende dall'uso del certificato: per la crittografia o come prova di chi ha inviato un'e-mail.

**Focus Svizzera:** questo articolo tratta la situazione giuridica in Svizzera. Laddove le norme UE divergono, è riportata una nota contrassegnata.

## Cosa verifica SwissSign per Silver e Gold

Dal 1° settembre 2023, per i certificati S/MIME pubblicamente attendibili si applicano i `S/MIME Baseline Requirements` del CA/Browser Forum. Essi distinguono quattro tipi di validazione, a seconda di ciò che l'autorità di certificazione verifica prima dell'emissione. SwissSign associa i propri prodotti a questi tipi nei profili di certificato pubblicati (CPR S/MIME):

| Prodotto | Tipo secondo il CA/B Forum | Cosa viene verificato | Cosa contiene il certificato |
|---|---|---|---|
| E-Mail ID Silver | `Mailbox-validated` | Controllo della casella e-mail | solo l'indirizzo e-mail |
| E-Mail ID Gold | `Sponsor-validated` | Organizzazione e persona | Nome e cognome (o pseudonimo), organizzazione, identificativo dell'organizzazione, Paese, indirizzo e-mail |
| Organization Validated | `Organization-validated` | Organizzazione | nome dell'organizzazione, nessuna persona |

Per Silver, SwissSign invia un codice di conferma all'indirizzo richiesto. Chi può ricevere l'e-mail ottiene il certificato. L'identità della persona non viene verificata e l'indirizzo compare nel certificato anche come `Common Name`.

Gold è gestito da SwissSign esclusivamente come `Sponsor-validated`. Vengono verificati l'organizzazione e l'associazione della persona a essa: la sua identità viene verificata da SwissSign oppure dall'organizzazione stessa in qualità di ente di registrazione, ad esempio nell'ambito di una Managed PKI. Nei profili di certificato non è elencato un profilo per privati senza organizzazione (`Individual-validated`).

Per caselle funzionali quali `support@` o `buchhaltung@` è previsto il profilo `Organization-validated`. Le richieste Gold per tali caselle possono fallire perché SwissSign rifiuta termini generici quali «Support» o «Helpdesk» nel campo `GivenName`.

Le due categorie non differiscono dal punto di vista crittografico. Tutti i profili utilizzano chiavi RSA da almeno 2048 bit, l'uso della chiave `digitalSignature` e `keyEncipherment` nonché l'uso esteso `id-kp-emailProtection`. Un'e-mail crittografata con Silver è protetta tanto bene quanto una crittografata con Gold.

## Cosa significa la differenza nella pratica

| | Silver | Gold |
|---|---|---|
| Crittografare e decrittografare e-mail | sì | sì |
| Visualizzazione della firma in Outlook | Indirizzo e-mail | Nome e organizzazione |
| Significato della firma | Il mittente ha accesso alla casella e-mail | Il mittente è una persona verificata di questa organizzazione |
| Impegno per la richiesta | Conferma via e-mail | Prova dell'organizzazione e verifica dell'identità |

La differenza emerge in caso di compromissione di un account. Chi controlla una casella e-mail può richiedere un certificato Silver per tale indirizzo e inviare così e-mail con firma valida. Un certificato Gold richiede invece che l'organizzazione e la persona siano state verificate. Per una banca che, ad esempio, riceve ordini di pagamento o corrispondenza contrattuale via e-mail, una firma Gold ha quindi maggiore valore informativo.

Per la crittografia questo non ha alcuna rilevanza. La banca necessita soltanto della chiave pubblica del destinatario, che è equivalente per entrambi i livelli.

## Requisiti legali in Svizzera

Nessuna legge svizzera prescrive S/MIME e nessuna distingue tra livelli di certificato. Esistono obblighi generali di protezione dei dati, la cui attuazione tecnica resta aperta.

| Normativa | Contenuto | Riferimento a S/MIME |
|---|---|---|
| LPD art. 8 e OPDa art. 1-6 | Sicurezza dei dati adeguata al rischio | La crittografia è uno strumento abituale per dati personali sensibili; non è prescritto un metodo specifico |
| LBCR art. 47 | Segreto bancario | Obbliga la banca, non la sua clientela |
| Circ. FINMA 2023/1 | Protezione dei dati critici presso le banche | Tecnologicamente neutrale, nessun requisito relativo ai certificati |
| FiEle e CO art. 14 cpv. 2bis | Solo la firma elettronica qualificata equivale alla firma autografa | Silver e Gold non sono firme qualificate |
| OCEI-CE | Invii elettronici a tribunali e uffici di esecuzione | Sono necessari una firma qualificata e una piattaforma di recapito riconosciuta; S/MIME non è sufficiente |

### Protezione dei dati: LPD e OPDa

L'art. 8 LPD obbliga i titolari del trattamento e i responsabili del trattamento a garantire una sicurezza dei dati adeguata al rischio. L'ordinanza sulla protezione dei dati precisa tale obbligo con obiettivi di protezione quali la riservatezza e l'integrità, ma non indica metodi specifici. Se utilizzare S/MIME, un gateway con recapito via webmail o una piattaforma di scambio dipende dalla valutazione del singolo caso.

### Segreto bancario e FINMA

Il segreto bancario ai sensi dell'art. 47 LBCR e la Circolare FINMA 2023/1 «Rischi operativi e resilienza: banche» si rivolgono alla banca. La circolare richiede alle banche di proteggere i loro dati critici per l'intero ciclo di vita, anche durante la trasmissione. Le banche stabiliscono nelle proprie direttive come attuare tali requisiti. L'accettazione di Silver è quindi una questione di policy interna della rispettiva banca.

### Firma elettronica: FiEle e CO

La FiEle distingue tra firma elettronica, firma elettronica avanzata, firma elettronica regolamentata e firma elettronica qualificata. Ai sensi dell'art. 14 cpv. 2bis CO, solo la firma elettronica qualificata con marca temporale qualificata equivale alla firma autografa. I certificati S/MIME di SwissSign sono emessi secondo le regole del CA/Browser Forum e le policy ETSI LCP (Silver) rispettivamente NCP (Gold) e non sono certificati qualificati ai sensi della FiEle.

Un contratto per il quale è prescritta la forma scritta non può quindi essere concluso validamente nella forma con un'e-mail firmata S/MIME, neppure con Gold. In caso di controversia, una firma Gold può comunque servire come mezzo di prova; il tribunale ne valuta liberamente il peso.

### Invii a tribunali e autorità

Per gli invii elettronici nei procedimenti civili, penali, di esecuzione e fallimentari, l'OCEI-CE richiede una firma elettronica qualificata e la trasmissione tramite una piattaforma di recapito riconosciuta. Un'e-mail S/MIME non soddisfa questi requisiti.

**Nota UE:** anche nell'UE, ai sensi dell'art. 25 par. 2 eIDAS, solo la firma elettronica qualificata equivale alla firma autografa.

## Se la banca non fornisce indicazioni

Se manca una scheda informativa, la situazione può essere chiarita con tre domande alla banca:

1.  La banca richiede per le e-mail firmate un certificato con identità verificata oppure è sufficiente una mera validazione dell'indirizzo e-mail?

2.  La banca utilizza un gateway di crittografia (ad es. SEPPmail o Kiteworks/Totemomail) e un certificato di dominio, oppure certificati personali dei propri collaboratori?

3.  Come mette a disposizione la banca le proprie chiavi pubbliche?

Una breve richiesta al servizio di consulenza alla clientela, che può essere inoltrata all'IT:

> Per lo scambio di e-mail crittografate con voi, intendo acquistare un certificato S/MIME. Vi prego di comunicarmi se richiedete un certificato con identità verificata (ad es. SwissSign E-Mail ID Gold con nome e organizzazione) o se è sufficiente un certificato con mera validazione dell'indirizzo e-mail (Silver). Vi chiedo inoltre il vostro certificato pubblico oppure l'indicazione se utilizzate un gateway con certificato di dominio.

Molte banche inviano comunque e-mail riservate tramite un gateway con recapito via webmail. La destinataria o il destinatario accede quindi al portale e non necessita di un proprio certificato. Se entrambe le parti lavorano con un gateway, spesso sono sufficienti certificati di dominio o TLS forzato tra i server di posta, e non sono necessari certificati personali.

Se la questione resta aperta, Gold è la scelta appropriata per la corrispondenza aziendale. Gold copre la crittografia e la firma, e un successivo passaggio da Silver a Gold comporta una nuova richiesta e una nuova distribuzione della chiave pubblica a tutte le controparti. Silver è sufficiente se la banca conferma che effettua solo la crittografia e non valuta le firme. Per le caselle funzionali è previsto un certificato `Organization-validated`.

## Fonti

1.  [SwissSign CPR S/MIME: Certificate, CRL and OCSP Profiles for S/MIME Certificates](https://repository.swisssign.com/SwissSign_CPR_SMIME_R14.pdf): profili di certificato di SwissSign (stato: 18.09.2025), associazione di Silver, Gold e Organization Validated ai tipi di validazione, campi Subject e OID delle policy.

2.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): regole vincolanti per certificati S/MIME pubblicamente attendibili, definizione dei quattro tipi di validazione.

3.  [Legge federale sulla protezione dei dati (LPD)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): art. 8 sulla sicurezza dei dati.

4.  [Ordinanza sulla protezione dei dati (OPDa)](https://www.fedlex.admin.ch/eli/cc/2022/568/de): art. 1-6 con principi e obiettivi di protezione della sicurezza dei dati.

5.  [Legge sulle banche (LBCR)](https://www.fedlex.admin.ch/eli/cc/51/117_121_129/de): art. 47 sul segreto bancario.

6.  [FINMA: circolari](https://www.finma.ch/de/dokumentation/rundschreiben/): panoramica con la Circolare 2023/1 «Rischi operativi e resilienza: banche», in vigore dal 1° gennaio 2024.

7.  [Legge federale sulla firma elettronica (FiEle)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): tipi di firma elettronica e requisiti per i certificati qualificati.

8.  [Codice delle obbligazioni (CO)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): art. 14 cpv. 2bis sull'equivalenza della firma elettronica qualificata alla firma autografa.
