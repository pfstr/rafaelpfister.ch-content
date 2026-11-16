---
title: "S/MIME per le banche: certificato Silver o Gold di SwissSign?"
navTitle: "S/MIME Silver/Gold"
description: "Una banca che firma e cifra le e-mail dei clienti con S/MIME necessita di certificati Gold per le caselle personali e di certificati Organization Validated per le caselle funzionali. Silver cifra con la stessa robustezza, ma non attesta la banca come mittente. Cosa verifica SwissSign, cosa figura nel certificato e quali requisiti vigono in Svizzera."
date: "2026-11-16"
kategorie: "Crittografia e-mail"
timeToRead: "9 min di lettura"
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
translationSourceHash: f95529dd7c72d4d783714affbcf18d96e37adf66a52379b94212034dda610b74
translationModel: gpt-5.6-terra
translatedAt: 2026-11-16T14:48:31.327Z
translationReview: required
url: https://rafaelpfister.ch/it/blog/s-mime-con-le-banche-certificato-silver-o-gold-di-swisssign
---

# S/MIME per le banche: certificato Silver o Gold di SwissSign?

In breve: una banca che firma e cifra con S/MIME le e-mail destinate a clienti e partner commerciali dovrebbe dotare le caselle personali di certificati Gold e le caselle funzionali di certificati Organization Validated di SwissSign. Silver cifra con la stessa robustezza, ma nel certificato figura soltanto l'indirizzo e-mail e non il nome della banca. Né la legge né la FINMA prescrivono un determinato livello di certificato.

La questione di quale livello di certificato sia necessario per l'invio di e-mail emerge regolarmente nei miei progetti con istituti finanziari. Solitamente si pone all'inizio di un progetto di gateway oppure quando clienti aziendali richiedono che i documenti riservati siano inviati esclusivamente in forma cifrata. Per rispondere, occorre distinguere due funzioni di S/MIME: la cifratura, che protegge il contenuto, e la firma, che dimostra al destinatario chi ha inviato l'e-mail.

**Focus Svizzera:** questo articolo tratta il quadro giuridico svizzero. Qualora le norme UE differiscano, è presente un'avvertenza contrassegnata.

## Cosa verifica SwissSign per Silver e Gold

Dal 1° settembre 2023, per i certificati S/MIME pubblicamente attendibili si applicano i `S/MIME Baseline Requirements` del CA/Browser Forum. Essi distinguono quattro tipi di validazione, in base a ciò che l'autorità di certificazione verifica prima dell'emissione. SwissSign assegna i propri prodotti a tali tipi nei profili di certificato pubblicati (CPR S/MIME):

| Prodotto | Tipo secondo il CA/B Forum | Cosa viene verificato | Cosa figura nel certificato |
|---|---|---|---|
| E-Mail ID Silver | `Mailbox-validated` | Controllo della casella di posta | solo l'indirizzo e-mail |
| E-Mail ID Gold | `Sponsor-validated` | Organizzazione e persona | Nome e cognome (o pseudonimo), organizzazione, identificativo dell'organizzazione, paese, indirizzo e-mail |
| Organization Validated | `Organization-validated` | Organizzazione | Nome dell'organizzazione, nessuna persona |

Per Silver, SwissSign invia un codice di conferma all'indirizzo richiesto. Chi può ricevere l'e-mail ottiene il certificato. L'identità della persona e l'organizzazione non vengono verificate e nel certificato l'indirizzo figura anche come `Common Name`.

Gold è gestito da SwissSign esclusivamente come `Sponsor-validated`. SwissSign verifica l'organizzazione e la persona le è associata. L'identità della persona viene verificata da SwissSign oppure dall'organizzazione stessa in qualità di ente di registrazione, di norma nell'ambito di una Managed PKI. Per caselle funzionali come `kundendienst@` o `zahlungsverkehr@` è previsto il profilo `Organization-validated`; in questo caso la banca figura nel certificato come organizzazione, ma non una persona.

Dal punto di vista crittografico, i livelli non differiscono. Tutti i profili utilizzano chiavi RSA da 2048 bit in su, gli usi della chiave `digitalSignature` e `keyEncipherment` nonché l'uso esteso `id-kp-emailProtection`.

## Cosa significa la differenza per una banca in qualità di mittente

| | Silver | Gold | Organization Validated |
|---|---|---|---|
| Cifrare e decifrare e-mail | sì | sì | sì |
| La firma mostra nel programma e-mail | Indirizzo e-mail | Nome e banca | Banca |
| Attestazione della firma | Il mittente ha accesso alla casella di posta | Persona verificata di questa banca | Casella di posta di questa banca |
| Impiego presso una banca | Non adatto alla comunicazione con i clienti | Caselle personali | Caselle funzionali |

La differenza riguarda la firma. Una banca firma le e-mail in uscita affinché i destinatari possano verificare che il messaggio provenga effettivamente da essa e non sia stato modificato durante il transito. Con Silver ciò è possibile solo in misura limitata:

- Chi registra un dominio dal nome simile, ad esempio `musterbank-ch.com` anziché `musterbank.ch`, ottiene senza ulteriori verifiche un certificato Silver per indirizzi di tale dominio. Le sue e-mail di phishing risultano quindi firmate validamente proprio come le e-mail autentiche della banca e, in entrambi i casi, nel certificato figura soltanto un indirizzo e-mail.
- Chi si impossessa di una casella di posta della banca può richiedere a sua volta un certificato Silver per tale indirizzo.

Un certificato Gold o Organization Validated contiene il nome verificato della banca nel campo `Organization`. Questo dato è ottenibile solo superando la verifica dell'organizzazione effettuata da SwissSign. Per i destinatari che esaminano la firma, questa è la prova rilevante.

Per la cifratura, il livello non ha alcuna importanza. Un'e-mail cifrata con Silver è protetta tanto bene quanto una cifrata con Gold.

## La cifratura richiede i certificati dei destinatari

Per cifrare un'e-mail con S/MIME, la banca necessita della chiave pubblica del destinatario. Il certificato della banca non viene utilizzato a questo scopo; serve per la firma e per ricevere risposte cifrate.

La maggior parte dei clienti privati non dispone di un certificato S/MIME. Le banche recapitano quindi le e-mail riservate di norma tramite un gateway di cifratura, come SEPPmail (GINA) o Kiteworks/Totemomail, che rende disponibile il messaggio in un portale web o come allegato cifrato. La cifratura S/MIME viene impiegata soprattutto con clienti aziendali e partner che gestiscono essi stessi certificati o un gateway. Se entrambe le parti lavorano con un gateway, spesso sono sufficienti certificati di dominio o TLS forzato tra i server di posta.

## Certificati operativi: Managed PKI e gateway

Per più di una manciata di caselle di posta, una banca ottiene i propri certificati tramite la Managed PKI di SwissSign. La banca agisce in questo caso come ente di registrazione e conferma essa stessa l'identità dei collaboratori; ciò corrisponde al tipo `Sponsor-validated`. Gateway come SEPPmail e Kiteworks/Totemomail possono, tramite un'interfaccia verso la Managed PKI, richiedere automaticamente i certificati, rinnovarli e associarli alle caselle di posta.

Dai progetti sono noti tre aspetti che causano problemi nell'esercizio:

- Le richieste Gold per caselle funzionali falliscono perché il gateway riprende nome e cognome dalla directory e SwissSign rifiuta denominazioni generiche quali «Support» o «Helpdesk» nel campo `GivenName`. Per queste caselle è possibile configurare nel gateway il profilo `Organization-validated`.
- I rinnovi automatici possono fallire per settimane senza che nessuno se ne accorga, finché i primi certificati non scadono e i clienti non possono più inviare e-mail cifrate. Occorre pertanto monitorare le date di scadenza dei certificati emessi e i messaggi di errore delle procedure di rinnovo.
- Il gateway firma le proprie richieste alla Managed PKI con un certificato proprio dell'ente di registrazione. Se questo scade, si interrompe l'intera automazione.

## Requisiti legali in Svizzera

Nessuna legge svizzera prescrive S/MIME e nessuna distingue tra livelli di certificato. Gli obblighi di una banca riguardano la protezione dei dati dei clienti; è la banca stessa a decidere come attuarla.

| Norma | Contenuto | Rilevanza per l'invio di e-mail della banca |
|---|---|---|
| Legge sulle banche (LBCR) art. 47 | Segreto bancario | I dati dei clienti non possono essere divulgati senza autorizzazione; l'invio non cifrato via Internet è un rischio che la banca deve valutare |
| Circ. FINMA 2023/1 | Protezione dei dati critici | Protezione anche durante la trasmissione, neutrale dal punto di vista tecnologico, nessun requisito sui certificati |
| LPD art. 8 e OPDa art. 1-6 | Sicurezza dei dati adeguata al rischio | La cifratura è uno strumento abituale, ma non è prescritto un procedimento specifico |
| ZertES e CO art. 14 cpv. 2bis | Solo la firma elettronica qualificata equivale alla firma autografa | Le firme S/MIME non sostituiscono una firma |

### Segreto bancario e FINMA

Il segreto bancario ai sensi dell'art. 47 LBCR obbliga la banca, i suoi organi e i suoi collaboratori a non divulgare senza autorizzazione i dati dei clienti. La Circolare FINMA 2023/1 «Rischi operativi e resilienza: banche» richiede che le banche proteggano i propri dati critici durante l'intero ciclo di vita, anche durante la trasmissione. Il modo in cui ciò avviene viene stabilito dalla banca nelle proprie direttive. La scelta del livello di certificato è quindi parte della policy interna e dovrebbe essere documentata in essa.

### Protezione dei dati: LPD e OPDa

L'art. 8 LPD obbliga i titolari del trattamento a garantire una sicurezza dei dati adeguata al rischio. L'ordinanza sulla protezione dei dati concretizza tale obbligo mediante obiettivi di protezione quali riservatezza e integrità, ma non indica procedure specifiche. S/MIME, un gateway con recapito via webmail o una piattaforma di scambio soddisfano in ugual misura questo requisito, purché siano impiegati in modo adeguato al rischio.

### Firma elettronica: ZertES e CO

La ZertES distingue tra firma elettronica, firma elettronica avanzata, firma elettronica disciplinata e firma elettronica qualificata. Ai sensi dell'art. 14 cpv. 2bis CO, solo la firma elettronica qualificata con marca temporale qualificata equivale alla firma autografa. I certificati S/MIME di SwissSign sono emessi secondo le regole del CA/Browser Forum e le policy ETSI LCP (Silver) o NCP (Gold) e non sono certificati qualificati ai sensi della ZertES. Contratti o dichiarazioni per i quali è prescritta la forma scritta non possono essere validamente rilasciati tramite un'e-mail firmata S/MIME, neppure con Gold.

**Avvertenza UE:** nell'UE, ai sensi dell'art. 25 cpv. 2 eIDAS, solo la firma elettronica qualificata equivale anch'essa alla firma autografa.

## Raccomandazione

- **Caselle personali** della consulenza alla clientela e di altri collaboratori a contatto con i clienti: Gold (`Sponsor-validated`) tramite la Managed PKI.
- **Caselle funzionali:** `Organization-validated`, affinché la banca figuri nel certificato come mittente.
- Non impiegare **Silver** per la comunicazione con i clienti, poiché la firma non attesta la banca come mittente.
- **Cifratura per clienti privati** tramite il recapito push o tramite portale del gateway; S/MIME per destinatari con certificato o gateway proprio.
- **Firma e verifica del mittente** si completano a vicenda: la firma S/MIME non sostituisce SPF, DKIM e DMARC, ma mostra al destinatario nel programma e-mail l'organizzazione verificata.
- **Scheda informativa per i clienti:** descrivete quali certificati utilizza la banca, come i clienti possono verificare una firma e quali livelli di certificato la banca accetta dai clienti che desiderano inviare autonomamente e-mail cifrate. Se manca una scheda di questo tipo, la domanda viene inoltrata alla consulenza alla clientela, che nella maggior parte dei casi non sa rispondere.

## Fonti

1.  [SwissSign CPR S/MIME: Certificate, CRL and OCSP Profiles for S/MIME Certificates](https://repository.swisssign.com/SwissSign_CPR_SMIME_R14.pdf): profili di certificato di SwissSign (aggiornati al 18.09.2025), attribuzione di Silver, Gold e Organization Validated ai tipi di validazione, campi Subject e OID delle policy.

2.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): regole vincolanti per i certificati S/MIME pubblicamente attendibili, definizione dei quattro tipi di validazione.

3.  [SEPPmail: Managed PKI SwissSign](https://docs.seppmail.com/en/07_mi_10_mpki_11_swisssign.html): collegamento del gateway SEPPmail alla Managed PKI di SwissSign.

4.  [Legge sulle banche (LBCR)](https://www.fedlex.admin.ch/eli/cc/51/117_121_129/de): art. 47 sul segreto bancario.

5.  [FINMA: Circolari](https://www.finma.ch/de/dokumentation/rundschreiben/): panoramica comprendente la Circolare 2023/1 «Rischi operativi e resilienza: banche», in vigore dal 1° gennaio 2024.

6.  [Legge federale sulla protezione dei dati (LPD)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): art. 8 sulla sicurezza dei dati.

7.  [Ordinanza sulla protezione dei dati (OPDa)](https://www.fedlex.admin.ch/eli/cc/2022/568/de): artt. 1-6 con principi e obiettivi di protezione della sicurezza dei dati.

8.  [Legge federale sulla firma elettronica (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): tipi di firma elettronica e requisiti per i certificati qualificati.

9.  [Codice delle obbligazioni (CO)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): art. 14 cpv. 2bis sull'equivalenza della firma elettronica qualificata alla firma autografa.
