---
title: "Crittografia di dominio S/MIME anziché TLS pinning: sicurezza e quadro giuridico in Svizzera"
navTitle: "Dominio anziché pinning"
description: "Perché la crittografia di dominio S/MIME protegge il traffico e-mail tra due organizzazioni meglio del TLS pinning, cosa può fare un aggressore con un messaggio S/MIME intercettato, come LPD, OPDa e segreti professionali influenzano la scelta e perché MTA-STS e DANE assumono la protezione del trasporto."
date: "2026-11-16"
kategorie: "Crittografia e-mail"
timeToRead: "10 min di lettura"
themen:
  - e-mail-verschluesselung
produkte: []
protokolle:
  - "verschluesselung"
  - "tls"
  - "dns"
hauptthema: "e-mail-verschluesselung"
startseiteAb: "2099-12-31"
slug: "crittografia-di-dominio-s-mime-anziche-tls-pinning-sicurezza-e-quadro-giuridico-in-svizzera"
translationId: "article-92db7a4587660dec"
translationOf: domain-verschluesselung-statt-tls-pinning
url: https://rafaelpfister.ch/it/blog/crittografia-di-dominio-s-mime-anziche-tls-pinning-sicurezza-e-quadro-giuridico-in-svizzera
translationSourceHash: 8eeb8d5552f8680381710c1a052edd3e62b69b34e1af9fd032c4bd481e8d65a1
translationModel: gpt-5.6-terra
translatedAt: 2026-11-16T15:58:36.433Z
translationReview: required
---

# Crittografia di dominio S/MIME anziché TLS pinning: sicurezza e quadro giuridico in Svizzera

In breve: il TLS pinning protegge la connessione tra due server di posta, la crittografia di dominio S/MIME protegge il contenuto del messaggio lungo l'intero percorso tra due organizzazioni. Se occorre scegliere tra le due, la crittografia di dominio è la scelta migliore dal punto di vista tecnico e giuridico. MTA-STS e DANE assumono la protezione del trasporto finora garantita dal pinning.

In un progetto attuale, una connessione e-mail esistente tra due organizzazioni, finora protetta con TLS obbligatorio e pinning dei certificati, deve essere convertita alla crittografia di dominio S/MIME tramite un servizio Secure Mail. Per la stessa connessione, il servizio non può offrire entrambe le opzioni contemporaneamente: si applica il pinning oppure la crittografia di dominio. La domanda se questo comporti una perdita di sicurezza è legittima. La risposta è chiara non appena si considera cosa proteggono i due metodi.

**Focus Svizzera:** la classificazione giuridica segue il diritto svizzero, in particolare LPD, OPDa e CP. Laddove le norme UE differiscono, viene fornita un'indicazione contrassegnata. L'articolo non sostituisce una consulenza legale per il singolo caso.

## Cosa protegge il TLS pinning

Senza misure aggiuntive, TLS tra server di posta è opportunistico. Ne derivano tre possibilità di attacco:

- **Downgrade:** un aggressore nel percorso di trasmissione rimuove l'offerta STARTTLS dalla sessione SMTP. Il server mittente effettua quindi la consegna in chiaro.
- **Certificato qualsiasi:** i server di posta mittenti non verificano per impostazione predefinita il certificato della controparte. Viene accettato un server con un certificato autofirmato.
- **MX falsificato:** chi manipola la risposta DNS reindirizza l'e-mail a un proprio server che offre TLS valido.

Il pinning chiude tutte e tre le lacune per un partner specifico. Il gateway mittente impone TLS e accetta solo un certificato o una chiave prestabiliti. Tuttavia, la protezione termina con la connessione. Sul server ricevente, nelle code, nei filtri antispam e antivirus, nei sistemi di journaling e archiviazione, il messaggio è disponibile in chiaro. Se attraversa più server, il pinning si applica solo al tratto per il quale è stato configurato.

Si aggiunge l'aspetto operativo. Il pin è una regola manuale presso il mittente. Se il destinatario rinnova il proprio certificato o modifica il routing, ad esempio tramite un nuovo servizio di filtraggio o un MX di backup, la consegna si interrompe finché la regola non viene aggiornata. In caso di guasto, la regola viene quindi spesso allentata temporaneamente e non sempre riattivata.

## Cosa protegge la crittografia di dominio S/MIME

Con la crittografia di dominio, il gateway del mittente cifra il messaggio con la chiave pubblica contenuta nel certificato di dominio del dominio destinatario. Solo il gateway del destinatario lo decifra nuovamente. Tutti i sistemi intermedi vedono solo il contenuto cifrato, indipendentemente dal numero di server attraversati dal messaggio e da dove viene memorizzato temporaneamente. Inoltre, il gateway del mittente firma il messaggio con il proprio certificato di dominio.

| Protezione | TLS pinning | Crittografia di dominio S/MIME |
|---|---|---|
| Corpo dell'e-mail e allegati | solo durante la trasmissione sul tratto con pinning | lungo l'intero percorso, anche su server intermedi e nelle code |
| Oggetto, mittente, destinatario, data | cifrati durante la trasmissione | leggibili, salvo sia attiva la protezione degli header |
| Envelope, dimensione, momento | cifrati durante la trasmissione | leggibili |
| Autenticità del server ricevente | sì, un server errato viene rifiutato | no, l'e-mail viene consegnata al server indicato dal routing MX |
| Forward Secrecy | sì, con scambio di chiavi ECDHE | no |
| Integrità e origine dopo la consegna | no | sì, tramite firma |
| Dipendenza da routing e infrastruttura | elevata, il pin vale per un server o una chiave | ridotta, la protezione dipende dal certificato di dominio |

Il pinning presenta quindi tre caratteristiche che la crittografia di dominio non offre: protegge i metadati, impedisce che un server errato accetti l'e-mail e offre Forward Secrecy. Per il contenuto, la crittografia di dominio è chiaramente superiore.

## Cosa può fare un aggressore con un messaggio S/MIME intercettato

Supponiamo che un aggressore inserisca un proprio server di posta nel percorso di trasmissione, tramite record MX falsificati, BGP hijacking o un relay compromesso.

Non può fare quanto segue:

- **Leggere il contenuto:** testo e allegati sono cifrati con la chiave pubblica del dominio destinatario. Senza la chiave privata del gateway del destinatario, rimangono illeggibili.
- **Modificare il contenuto senza essere rilevato:** se il messaggio è firmato e il gateway destinatario richiede una firma valida, ogni modifica viene rilevata.
- **Generare messaggi a nome del mittente:** gli manca la chiave privata di firma del dominio mittente.

Può fare quanto segue:

- **Analizzare i metadati:** mittente, destinatario, momento, dimensione e, di norma, l'oggetto sono leggibili. Un oggetto come «Rapporto di dimissione Max Muster» costituisce già una comunicazione di dati personali.
- **Memorizzare messaggi e decifrarli in seguito:** S/MIME non offre Forward Secrecy. Se la chiave privata di dominio viene compromessa successivamente, tutti i messaggi memorizzati diventano leggibili. Lo stesso vale a lungo termine qualora RSA diventasse attaccabile mediante computer quantistici.
- **Sopprimere o ritardare messaggi:** il mittente non necessariamente se ne accorge. È interessata la disponibilità, il che è rilevante per le scadenze.
- **Modificare header non protetti:** può ad esempio riscrivere l'oggetto o `Reply-To` per deviare le risposte. Il contenuto cifrato rimane invariato.
- **Manipolare selettivamente il testo cifrato:** nei messaggi solo cifrati e non firmati che usano AES-CBC, il testo cifrato può essere modificato in modo che parti del testo in chiaro possano fuoriuscire dopo la decifratura. È il principio degli attacchi EFAIL del 2018. Una verifica obbligatoria della firma o la cifratura autenticata con AES-GCM secondo S/MIME 4.0 lo impediscono.

Un testo in chiaro intercettato con la sola crittografia del trasporto costituisce una fuoriuscita completa di dati. Un messaggio S/MIME intercettato riguarda essenzialmente metadati e disponibilità.

## Classificazione giuridica

Nessuna norma svizzera prescrive S/MIME, TLS pinning o un altro metodo specifico. Le prescrizioni sono tecnologicamente neutrali e definiscono obiettivi di protezione. La scelta del metodo deve tuttavia essere valutata rispetto a tali obiettivi di protezione e documentata.

| Norma | Contenuto | Rilevanza per la scelta del metodo |
|---|---|---|
| LPD art. 8, OPDa art. 1-3 | sicurezza adeguata dei dati, obiettivi di protezione, controllo del trasporto | protezione nella comunicazione, commisurata al rischio e allo stato della tecnica |
| LPD art. 61 lett. c | multa per violazione dei requisiti minimi di sicurezza dei dati | responsabilità penale personale della persona responsabile |
| LPD art. 9 | trattamento da parte di responsabili del trattamento | deve essere considerato ogni sistema con accesso al testo in chiaro |
| LPD art. 16 e 17 | comunicazione all'estero | trattamento del testo in chiaro su sistemi esteri |
| LPD art. 24 | notifica di violazioni della sicurezza dei dati | il rischio per le persone interessate dipende dalla leggibilità dei dati fuoriusciti |
| CP art. 321, LPD art. 62, LPGA art. 33 | segreto professionale, obbligo di discrezione professionale, obbligo del segreto nelle assicurazioni sociali | il contenuto deve essere protetto da ogni presa di conoscenza non autorizzata |
| CPC art. 177 | file elettronici quali documenti | la firma rafforza la prova dell'origine e dell'integrità |

### Sicurezza dei dati secondo LPD e OPDa

L'art. 8 LPD richiede una sicurezza dei dati adeguata al rischio. L'OPDa concretizza gli obiettivi di protezione della riservatezza, disponibilità, integrità e tracciabilità e cita all'art. 3, quale misura tra le altre, il controllo del trasporto: i dati personali non devono poter essere letti, copiati, modificati o cancellati senza autorizzazione durante la comunicazione. Per i dati personali degni di particolare protezione ai sensi dell'art. 5 lett. c LPD, come i dati sanitari o i dati relativi a misure di assistenza sociale, si applicano requisiti di conseguenza più elevati.

Il TLS pinning soddisfa il controllo del trasporto per un tratto della trasmissione. Tuttavia, il messaggio viene memorizzato su più sistemi prima di raggiungere la destinazione e lì è leggibile in chiaro. Con la crittografia di dominio, il contenuto è protetto per l'intero percorso tra le due organizzazioni. La firma copre inoltre l'obiettivo di protezione dell'integrità, che TLS non può più dimostrare dopo la consegna.

### Responsabilità penale personale

Ai sensi dell'art. 61 lett. c LPD, chiunque intenzionalmente non rispetta i requisiti minimi di sicurezza dei dati secondo l'art. 8 cpv. 3 LPD è punito, a querela di parte, con una multa fino a CHF 250'000. La multa è rivolta alla persona fisica responsabile, non all'impresa. Una cifratura del contenuto conforme allo stato attuale della tecnica, documentata in una valutazione del rischio, rappresenta per queste persone una prova più solida di una regola di trasporto per singoli server.

**Nota UE:** anche il GDPR richiede all'art. 32 misure tecniche e organizzative adeguate e menziona espressamente la cifratura. Tuttavia, le sanzioni ai sensi dell'art. 83 GDPR sono rivolte all'impresa e, per violazioni dell'art. 32, ammontano fino a 10 milioni di euro o al 2% del fatturato annuo mondiale.

### Responsabili del trattamento e comunicazione all'estero

Ogni sistema sul percorso che tratta il testo in chiaro deve essere considerato sotto il profilo giuridico: servizi di filtraggio, relay, gateway cloud, servizi di archiviazione. Se tali sistemi sono gestiti da terzi, di norma si tratta di trattamento da parte di responsabili ai sensi dell'art. 9 LPD, con i relativi contratti e obblighi di controllo. Se i sistemi si trovano all'estero, si aggiungono le norme sulla comunicazione all'estero secondo gli art. 16 e 17 LPD.

Con la crittografia di dominio, questi sistemi trattano solo testo cifrato. La cerchia dei soggetti che possono accedere ai contenuti si limita ai due gateway. I metadati restano tuttavia dati personali e gli obblighi citati continuano ad applicarsi.

### Obbligo di notifica delle violazioni della sicurezza dei dati

Una violazione della sicurezza dei dati che comporta presumibilmente un rischio elevato per le persone interessate deve essere notificata all'IFPDT ai sensi dell'art. 24 LPD. Se messaggi fuoriescono tramite un server intermedio compromesso, la valutazione dipende dalla leggibilità dei contenuti. Se sono fortemente cifrati e le chiavi non sono coinvolte, il rischio per le persone interessate è di norma ridotto, purché l'oggetto non contenga indicazioni sensibili. Con la sola cifratura del trasporto, i contenuti sono disponibili all'aggressore in chiaro.

### Segreti professionali e obblighi del segreto

Laddove i dati sono soggetti a un segreto professionale ai sensi dell'art. 321 CP, ad esempio per medici, avvocati o i loro ausiliari, o a un obbligo legale del segreto quale l'art. 33 LPGA nell'ambito delle assicurazioni sociali, il contenuto deve essere protetto da ogni presa di conoscenza non autorizzata. Ciò include anche i gestori di sistemi intermedi. L'art. 62 LPD punisce inoltre la violazione dell'obbligo di discrezione professionale da parte di persone non soggette all'art. 321 CP. Questo requisito è soddisfatto solo dalla cifratura del contenuto.

### Valore probatorio della firma

Ai sensi dell'art. 177 CPC, i file elettronici sono documenti, il cui valore probatorio è liberamente valutato dal giudice. Una firma S/MIME valida del dominio mittente dimostra che il messaggio proviene da tale dominio e non è stato modificato dopo la firma. Ciò rafforza la prova in caso di scadenze, impegni o istruzioni. Non sostituisce una firma autografa; i requisiti formali del CO sono soddisfatti solo da una firma elettronica qualificata secondo ZertES. Con il TLS pinning non è possibile dimostrare né l'origine né l'integrità dopo la consegna.

## MTA-STS e DANE in sostituzione del TLS pinning

Se il pinning viene eliminato, vengono meno le caratteristiche offerte solo dalla protezione del trasporto: TLS obbligatorio, un server ricevente autenticato e quindi la protezione dei metadati contro l'intercettazione e il reindirizzamento. MTA-STS e DANE forniscono proprio queste caratteristiche, come standard che il destinatario pubblica una volta per tutti i mittenti.

**MTA-STS (RFC 8461):** il destinatario pubblica sotto `https://mta-sts.<domain>/.well-known/mta-sts.txt` una policy con i nomi host MX consentiti e la modalità `enforce`. Un mittente che supporta MTA-STS recupera la policy, la memorizza e consegna solo tramite TLS con certificato valido a uno degli MX indicati. Un MX falsificato o un server senza certificato valido per il nome MX viene rifiutato.

**DANE (RFC 7672):** il destinatario pubblica nella propria zona firmata con DNSSEC un record TLSA con l'hash del proprio certificato o della propria chiave. Il mittente verifica il certificato dell'MX rispetto a tale record. Tecnicamente, DANE è un pinning che il destinatario stesso pubblica nel DNS e protegge con DNSSEC.

| | TLS pinning | MTA-STS | DANE |
|---|---|---|---|
| Gestito da | mittente, manualmente per ciascun partner | destinatario, una volta per tutti i mittenti | destinatario, una volta per tutti i mittenti |
| Efficace per | un partner configurato | tutti i mittenti con supporto MTA-STS | tutti i mittenti con supporto DANE |
| Rinnovo del certificato | il pin deve essere aggiornato presso il mittente | senza adattamenti, finché il nuovo certificato è valido | senza adattamenti con la stessa chiave, altrimenti pubblicare prima un nuovo record TLSA |
| Base di fiducia | esattamente questo certificato o questa chiave | autorità di certificazione pubbliche | DNSSEC |
| Punto debole | onere operativo, allentamento in caso di guasti | primo recupero e cache scaduta (Trust on First Use), emissione errata da parte di una CA | richiede DNSSEC |
| Rapporti sugli errori | nessuno | TLS-RPT (RFC 8460) | TLS-RPT (RFC 8460) |

Per la documentazione ai sensi dell'art. 8 LPD, MTA-STS e DANE presentano un ulteriore vantaggio: tramite TLS-RPT il destinatario riceve ogni giorno rapporti che indicano se le consegne sono avvenute in modo cifrato o sono fallite. Con il pinning non esiste questo riscontro. L'articolo [Record DNS per e-mail](https://rafaelpfister.ch/blog/dns-records-e-mail-stolpersteine) descrive la configurazione di entrambi i metodi nella sezione dedicata a MTA-STS, TLS-RPT e DANE.

## Chi deve verificare MTA-STS e DANE

Deve verificare sempre il sistema che consegna il messaggio direttamente all'MX del destinatario. I record devono essere pubblicati dal gestore dell'MX.

Con un servizio Secure Mail, il percorso in uscita è tipicamente il seguente:

```text
Eigener Mailserver -> Secure-Mail-Dienst (verschlüsselt, signiert) -> MX des Partners
```

Finora il pinning proteggeva l'ultimo tratto. Qui il servizio deve verificare MTA-STS e DANE del partner; il proprio server di posta non ha alcuna influenza su questo tratto. Il tratto dal proprio server di posta al servizio deve essere protetto autonomamente, con TLS obbligatorio e verifica del certificato sul connettore.

Sul lato ricevente dipende dall'MX. Se l'MX punta al servizio, quest'ultimo deve fornire certificati adeguati e si pubblicano record MTA-STS e TLSA adatti ai suoi server. Per DANE, ciò presuppone che il servizio annunci per tempo i cambi di chiave. Se l'MX punta al proprio gateway, ad esempio Exchange Online, si configurano MTA-STS e DANE lì. Se entrambe le organizzazioni utilizzano lo stesso servizio, quest'ultimo potrebbe consegnare il messaggio all'interno della propria piattaforma; in tal caso non vi è alcun tratto SMTP pubblico tra le due organizzazioni.

Se il servizio non supporta né MTA-STS né DANE per l'invio, il contenuto rimane protetto da S/MIME. Rimangono aperte la protezione dei metadati e l'autenticazione del server ricevente nell'ultimo tratto. Questo rischio residuo deve rientrare nella valutazione del rischio.

## Rischi residui e misure

| Rischio residuo | Misura |
|---|---|
| dati personali nell'oggetto leggibili | regola: nessun dato personale nell'oggetto, concordata per iscritto con il partner |
| assenza di Forward Secrecy | memorizzare la chiave privata di dominio in modo non esportabile, idealmente in un HSM; registrare gli accessi |
| manipolazione di messaggi non firmati | obbligo di firma su entrambi i lati, AES-GCM se entrambi i gateway lo supportano |
| consegna in chiaro in caso di errori | trattenere i messaggi in caso di cifratura non riuscita anziché consegnarli in chiaro |
| metadati nel percorso di trasporto | MTA-STS in modalità `enforce` e, dove è disponibile DNSSEC, DANE; analizzare TLS-RPT |
| certificati scaduti | notifica reciproca almeno 30 giorni prima della scadenza, confronto delle impronte digitali tramite un secondo canale |

La decisione a favore della crittografia di dominio e i rischi rimanenti devono essere inclusi nel concetto di sicurezza dei dati o nella documentazione delle attività di trattamento.

## Domande al fornitore del servizio gateway

1.  Il servizio verifica durante l'invio le policy MTA-STS in modalità `enforce` e i record DANE del dominio destinatario?

2.  Da quali server avviene la consegna ed è possibile consultare rapporti TLS-RPT o protocolli di consegna relativi alla cifratura del trasporto?

3.  Il servizio, in qualità di MX, supporta MTA-STS e DANE per i messaggi in entrata e come vengono annunciati i cambi di chiave?

4.  Il connettore dal proprio server di posta al servizio può essere gestito con TLS obbligatorio e verifica del certificato?

5.  Tra due clienti del servizio, la consegna avviene tramite SMTP pubblico o all'interno della piattaforma?

6.  Quali algoritmi vengono utilizzati per cifratura e firma e AES-GCM è supportato?

## Fonti

1.  [Legge federale sulla protezione dei dati (LPD)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): art. 8 sulla sicurezza dei dati, art. 9 sul trattamento da parte di responsabili, art. 16 e 17 sulla comunicazione all'estero, art. 24 sull'obbligo di notifica, art. 61 e 62 sulle disposizioni penali.

2.  [Ordinanza sulla protezione dei dati (OPDa)](https://www.fedlex.admin.ch/eli/cc/2022/568/de): art. 1-3 con principi, obiettivi di protezione e misure tecniche e organizzative, tra cui il controllo del trasporto.

3.  [Codice penale svizzero (CP)](https://www.fedlex.admin.ch/eli/cc/54/757_781_799/de): art. 321 sulla violazione del segreto professionale.

4.  [Legge federale sulla parte generale del diritto delle assicurazioni sociali (LPGA)](https://www.fedlex.admin.ch/eli/cc/2002/510/de): art. 33 sull'obbligo del segreto degli organi esecutivi.

5.  [Codice di procedura civile (CPC)](https://www.fedlex.admin.ch/eli/cc/2010/262/de): art. 177 sulla nozione di documento, compresi i file elettronici.

6.  [RFC 8461: SMTP MTA Strict Transport Security (MTA-STS)](https://www.rfc-editor.org/rfc/rfc8461): specifica della policy, delle regole di recupero e della modalità `enforce`.

7.  [RFC 7672: SMTP Security via Opportunistic DANE TLS](https://www.rfc-editor.org/rfc/rfc7672): DANE per SMTP, record TLSA e regole di verifica per i server mittenti.

8.  [RFC 8460: SMTP TLS Reporting](https://www.rfc-editor.org/rfc/rfc8460): formato e consegna dei rapporti TLS-RPT.

9.  [RFC 8551: S/MIME Version 4.0 Message Specification](https://www.rfc-editor.org/rfc/rfc8551): specifica S/MIME attuale con AuthEnvelopedData e AES-GCM.

10.  [EFAIL](https://efail.de/): descrizione degli attacchi pubblicati nel 2018 contro messaggi S/MIME e OpenPGP non firmati in modalità CBC.

11.  [Regolamento generale sulla protezione dei dati (GDPR)](https://eur-lex.europa.eu/legal-content/DE/TXT/?uri=CELEX:32016R0679): art. 32 sulla sicurezza del trattamento, art. 83 sulle sanzioni pecuniarie (nota UE).
