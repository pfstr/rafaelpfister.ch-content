---
title: "Totemomail e Kiteworks EPG: gateway e-mail e cambio di piattaforma"
blatt: "totemomail"
description: "Totemomail e il suo successore Kiteworks Email Protection Gateway per amministratori di messaggistica: architettura SMTP inline, policy, S/MIME, OpenPGP e TLS, WebMail, stack crittografico Java, stato delle chiavi, identità, licenze, migrazione, alta disponibilità, monitoraggio e ripristino."
fakten:
  - label: Linea di prodotti
    wert: totemomail → Kiteworks Email Protection Gateway
    href: https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf
  - label: Ruolo di sistema
    wert: gateway di crittografia e-mail basato su policy
    href: https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf
  - label: Trasporto
    wert: SMTP nel percorso del messaggio
    href: https://marketplace.microsoft.com/en/marketplace/apps/totemo.totemo-azr-tm7?tab=overview
  - label: Procedure
    wert: S/MIME · OpenPGP · TLS
    href: https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf
  - label: Senza chiave del destinatario
    wert: recapito tramite WebMail o portale
    href: https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf
  - label: Decisione
    wert: policy centrali e selezione automatica delle chiavi
    href: https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf
  - label: Nucleo crittografico legacy
    wert: Totemo Security Platform · TCM basato su Java
    href: https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/3500
  - label: Piattaforma EPG
    wert: Kiteworks Hardened Virtual Appliance
    href: https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf
  - label: Archiviazione dati EPG
    wert: database integrato non amministrato direttamente dal cliente
    href: https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf
  - label: Osservabilità
    wert: prova del mail flow · Syslog · audit della piattaforma
    href: https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf
  - label: Modalità operative EPG
    wert: on-premises · risorse cloud proprie · hosting single-tenant
    href: https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf
  - label: Origine
    wert: totemo AG, Svizzera · fondata nel 2001 · acquisita da Kiteworks nel 2022
    href: https://www.kiteworks.com/sites/default/files/resources/on-the-hunt-kiteworks-secure-content-strategy.pdf
werbung:
  - tools
  - newsletter
ctaThemen:
  - totemomail
translationSourceHash: d949453b9a80abb64f76a8ecdee768813548ae2987b18b071cc23d33b0c5df21
translationModel: gpt-5.6-terra
translatedAt: 2026-10-15T08:54:09.227Z
translationReview: automatic
---

# Totemomail e Kiteworks EPG: gateway e-mail e cambio di piattaforma

Totemomail era una famiglia di prodotti della svizzera totemo AG per la crittografia centralizzata delle e-mail. Dopo l'acquisizione da parte di Kiteworks, secondo il produttore il codice è stato integrato nella piattaforma Kiteworks e proseguito come **Email Protection Gateway**, EPG. I nomi non designano quindi due prodotti del tutto indipendenti, ma nemmeno lo stesso ambiente operativo: un'appliance Totemomail esistente e un EPG all'interno della Kiteworks Hardened Virtual Appliance presentano limiti diversi per piattaforma, database, amministrazione e ripristino ([Kiteworks: FAQ for totemomail customers](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf), [Kiteworks: EPG capability brief](https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf)).

Dal punto di vista funzionale, il ruolo centrale rimane quello di un gateway e-mail: elabora le e-mail secondo direttive centrali e utilizza S/MIME, OpenPGP o TLS; per i destinatari senza chiavi adeguate è disponibile una soluzione WebMail o portale. La documentazione del produttore descrive la gestione automatica di chiavi e certificati, nonché l'utilizzo con normali client di posta senza plug-in client specifico per EPG come modello di base ([Kiteworks: Enterprise packaging and EPG features](https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf), [Microsoft Marketplace: totemomail Encryption Gateway](https://marketplace.microsoft.com/en/marketplace/apps/totemo.totemo-azr-tm7?tab=overview)).

Per gli amministratori di messaggistica, la linea di prodotti è soprattutto una catena di confini di fiducia. Il gateway accetta un messaggio tramite SMTP, classifica direzione e identità, determina una policy, cerca chiavi o un canale alternativo, eventualmente modifica il contenuto MIME e consegna il messaggio al successivo hop. Con WebMail si aggiunge uno stato proprio di web, account, chiavi e archiviazione. L'affermazione «il messaggio è crittografato» è incompleta senza indicare metodo, tratto protetto, titolare della chiave e momento ([RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321), [RFC 8551](https://datatracker.ietf.org/doc/html/rfc8551), [RFC 9580](https://datatracker.ietf.org/doc/html/rfc9580)).

La spiegazione segue un messaggio attraverso le regole EPG, la selezione delle chiavi e il metodo di crittografia fino al recapito diretto o a WebMail. Il passaggio da Totemomail a Kiteworks, l'esercizio, la diagnosi e il ripristino vengono poi trattati separatamente.

Totemomail e Kiteworks EPG operano come gateway inline nel percorso SMTP. Un messaggio viene accettato, assegnato a un'identità e a una policy, protetto con una procedura appropriata e quindi reso disponibile tramite SMTP o portale. Il cambio di piattaforma modifica implementazione e gestione, non questo compito fondamentale.

## Approccio architetturale: gateway inline con policy centrale

Nel funzionamento come gateway, il sistema di posta interno invia messaggi in uscita tramite [SMTP](/kb/smtp) a Totemomail o EPG. I messaggi in entrata raggiungono il gateway direttamente tramite MX o attraverso un sistema di sicurezza e-mail a monte. Il gateway termina la sessione SMTP, elabora il messaggio e apre una nuova sessione per l'inoltro. Un handshake TCP o TLS riuscito non costituisce ancora accettazione del messaggio; il trasferimento di responsabilità avviene solo con la risposta SMTP positiva alla fine della fase DATA ([RFC 5321, sezione 4.1.1.4](https://datatracker.ietf.org/doc/html/rfc5321#section-4.1.1.4), [RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207)).

La descrizione pubblica del prodotto EPG distingue diversi percorsi di protezione. «Encryption Gateway» copre la crittografia e la decrittografia esterne del gateway. «Internal Encryption» protegge i messaggi durante il transito e l'archiviazione in una piattaforma e-mail cloud. Ulteriori componenti documentati sono Directory Proxy, Application Integration Proxy e un connettore per RMS o Microsoft Information Protection. Quali di questi ruoli siano concessi in licenza e realmente attivati deve risultare dalla configurazione e dal contratto propri; il solo nome del prodotto non lo dimostra ([Kiteworks: Enterprise packaging and licensing](https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf)).

Per Microsoft 365, l'integrazione del trasporto rimane una questione di connettori. Exchange Online deve instradare i messaggi in modo mirato verso il gateway e accettare il traffico di ritorno sulla base di un'identità affidabile della controparte. Un MX davanti a un gateway di terze parti modifica inoltre l'IP sorgente che Exchange Online vede per la valutazione di spam e spoofing. Microsoft documenta a tale scopo connettori e Enhanced Filtering; entrambi sono separati dalla cryptopolicy EPG ([Microsoft: Configure mail flow using connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow), [Microsoft: Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)).

## Stack tecnologico e limiti di prodotto verificabili

Per la piattaforma storica Totemomail, il nucleo crittografico è ben documentato pubblicamente. La documentazione NIST-CMVP descrive il **Totemo Cryptographic Module**, TCM, come libreria Java condivisa della Totemo Security Platform, TSP. La convalida del 2019 cita Totemo Appliance OS 2.3, JRE 8 e un modulo crittografico software con AES, RSA, ECDSA, famiglia SHA, HMAC e generatore di numeri casuali; la voce ha ora status storico e pertanto non dimostra una fornitura oggi ammissibile né una modalità algoritmica concreta installata ([NIST CMVP Certificate 3500](https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/3500), [Totemo TCM Security Policy](https://csrc.nist.gov/CSRC/media/projects/cryptographic-module-validation-program/documents/security-policies/140sp3500.pdf)).

La policy di sicurezza NIST delimita esplicitamente TCM quale componente della TSP. Non documenta un flusso di posta Totemomail completo né un'implementazione generale di coda, database o MTA. Tali dettagli non devono quindi essere trasferiti da un'installazione Apache James estranea o da una singola rilevazione sul campo a tutte le versioni di Totemomail o EPG. Per un documento operativo sono determinanti la configurazione esportata del prodotto, la matrice di supporto, l'inventario di pacchetti o servizi e il manuale del produttore della generazione effettivamente installata.

Con EPG, il confine si sposta alla Kiteworks Hardened Virtual Appliance. Le FAQ sulla migrazione descrivono che sistema operativo, web server e database sono contenuti nell'appliance e che gli amministratori dei clienti non ottengono accesso diretto al sistema operativo o al database. Gli aggiornamenti vengono installati come unità appliance testata. Ciò riduce la gestione dei componenti da parte del cliente, ma rende procedure di backup, cluster, aggiornamento e supporto dell'appliance la superficie di controllo primaria ([Kiteworks: FAQ for totemomail customers](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf)).

| Spazio di stato | Totemomail legacy | Kiteworks EPG | Evidenza amministrativa |
|---|---|---|---|
| SMTP e routing | configurazione di gateway, MTA e destinazione dell'installazione | EPG più modello di rete e ruoli Kiteworks | e-mail di prova, risposta SMTP, ID di coda/mail flow, hop successivo |
| Policy | regole e trigger Totemomail | policy EPG integrata più controlli della piattaforma | policy esportata, priorità, stato attivo, matrice di test |
| Chiavi | TSP/TCM, certificati e chiavi OpenPGP | archivio migrato di chiavi e certificati EPG | fingerprint, titolare, scadenza, cronologia, test di decrittografia |
| Recapito web | Totemomail WebMail e account destinatari | portale EPG nella piattaforma Kiteworks | registrazione, login, conservazione, apertura di vecchio messaggio |
| Database | database esterno o locale dipendente dall'installazione | database integrato senza accesso diretto dell'amministratore | job di backup, procedura di ripristino, percorso di supporto del produttore |
| Log | log di prodotto, mail flow e sistema | log normalizzati della piattaforma e di EPG, esportazione Syslog | ID messaggio/evento correlabile, base temporale, ingresso SIEM |

Il confronto si basa sulle FAQ di migrazione del produttore e sulle descrizioni pubblicate delle funzioni EPG. Laddove i dettagli di Totemomail dipendono dall'installazione, ciò rimane volutamente un punto da verificare anziché essere presentato come caratteristica di prodotto inventata ([Kiteworks: totemomail-to-EPG migration FAQ](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf), [Kiteworks: Top reasons for EPG](https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-totemomail.svg?v=20260813" title="Interaktive Infografik: Totemomail- und Kiteworks-EPG-Mailflow, Policy, Schlüssel, WebMail, Plattformmigration, Monitoring und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-totemomail.svg?v=20260813">Apri direttamente il grafico interattivo</a>.
</iframe>

## Mail flow SMTP e rischio di loop

Un percorso gateway richiede una responsabilità univoca per ogni direzione. In uscita, il sistema di posta non deve poter recapitare alternativamente direttamente o tramite EPG se solo il percorso gateway esegue la policy prescritta. In entrata, deve essere stabilito se MX, filtri a monte o EPG autenticano il mittente originario e quale hop raggiunge il server di posta interno. Più connettori bidirezionali senza una chiara autorità su destinatari e domini possono far circolare i messaggi tra i sistemi ([RFC 5321: Trace information and loop detection](https://datatracker.ietf.org/doc/html/rfc5321#section-6.3), [Microsoft: Mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud)).

Per ogni route produttiva, gli amministratori dovrebbero documentare congiuntamente i seguenti valori:

- rete sorgente o identità del peer basata su certificato e nome EHLO atteso;
- indirizzo di accettazione locale, porta, requisiti STARTTLS e certificato client;
- responsabilità autorevole, relay o split-domain;
- direzione della policy, classificazione di utenti e destinatari;
- hop successivo, risoluzione DNS o smarthost e requisito TLS;
- comportamento in caso di componente di policy, chiave, directory o portale non raggiungibile;
- numero massimo di hop, età della coda, responsabilità per retry e bounce.

Lo standard SMTP richiede l'aggiunta di righe Received a ogni inoltro e descrive il rilevamento dei loop in base al percorso di traccia. Per diagnosi e audit, l'ID evento del gateway deve essere correlabile con Internet `Message-ID`, envelope, intervallo temporale e traccia a valle. Un oggetto visibile non è un identificatore stabile e può essere modificato dalla policy o dal recapito web ([RFC 5321, sezione 4.4](https://datatracker.ietf.org/doc/html/rfc5321#section-4.4), [RFC 5322: Message-ID](https://datatracker.ietf.org/doc/html/rfc5322#section-3.6.4)).

Dopo l'accettazione SMTP, il motore di policy decide, in base a mittente, destinatario, directory e chiavi disponibili, quale percorso di protezione sia consentito e possibile.

## Motore di policy e decisione sull'identità

Kiteworks descrive EPG come livello decisionale automatico e guidato da policy. A seconda dell'ambito di funzioni concesso in licenza, le regole possono basarsi su domini, utenti, luoghi, caratteristiche del contenuto o etichette Microsoft Information Protection. Tuttavia, l'automazione non elimina la necessità di fidarsi degli input: un'intestazione del mittente, un trigger nell'oggetto o un'etichetta impostata dall'esterno non devono avere senza verifica lo stesso effetto di un'identità interna autenticata ([Kiteworks: EPG capability brief](https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf), [Kiteworks: totemo and Kiteworks better together](https://www.kiteworks.com/sites/default/files/resources/kiteworks-product-brief-top-5-reasons-totemo-kiteworks-better-together.pdf)).

Un modello di regole affidabile separa quattro questioni:

1. **Direzione:** interno verso esterno, esterno verso interno, interno verso interno o applicazione verso esterno.
2. **Identità:** envelope SMTP, `From:` visibile, sessione autenticata, connettore, oggetto directory e tenant non sono la stessa cosa.
3. **Azione:** firmare, crittografare, ricrittografare, recapito tramite portale, inoltrare senza modifiche, rifiutare temporaneamente o generare un bounce permanente.
4. **Percorso di errore:** chiave mancante, catena non valida, directory non raggiungibile, errore di policy e hop successivo non raggiungibile richiedono risultati distinti.

Prima delle modifiche, dovrebbe essere eseguita una piccola matrice di test con esattamente un mittente e un destinatario per classe di identità. Per ogni riga si registrano regola attesa, procedura, mittente visibile, canale destinatario, indicatore di log e comportamento in caso di errore. Solo quando questi risultati sono riproducibili, la policy non è soltanto configurata, ma dimostrata.

## Metodi di protezione e ricrittografia

[S/MIME](https://datatracker.ietf.org/doc/html/rfc8551) utilizza CMS e certificati X.509. Per firma o decrittografia, il gateway necessita di una chiave privata e, per la crittografia del destinatario, di un certificato pubblico idoneo. Percorso del certificato, nome o indirizzo e-mail, Key Usage, validità e informazioni di revoca fanno parte della verifica ([RFC 5652](https://datatracker.ietf.org/doc/html/rfc5652), [RFC 5280](https://datatracker.ietf.org/doc/html/rfc5280), [RFC 6960](https://datatracker.ietf.org/doc/html/rfc6960)).

[OpenPGP](https://datatracker.ietf.org/doc/html/rfc9580) utilizza un modello autonomo di chiavi, firme e pacchetti. Fingerprint, User ID, capacità, scadenza e revoca devono essere inventariati indipendentemente da X.509. Una chiave pubblica appresa automaticamente è affidabile solo quanto il percorso documentato di apprendimento e conferma. La documentazione del prodotto cita S/MIME e OpenPGP uno accanto all'altro, ma non li rende modelli di fiducia intercambiabili ([Kiteworks: Enterprise EPG feature matrix](https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf)).

[SMTP-TLS](/kb/tls#tls-policy-bei-smtp) protegge una singola connessione di trasporto. Con STARTTLS opportunistico, il recapito può ripiegare sul testo in chiaro senza una policy aggiuntiva. A seconda dell'ambiente, un'identità peer vincolante può essere definita tramite verifica del certificato, DANE o MTA-STS; ciò non protegge comunque il messaggio dagli MTA coinvolti ([RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207), [RFC 7672](https://datatracker.ietf.org/doc/html/rfc7672), [RFC 8461](https://datatracker.ietf.org/doc/html/rfc8461)).

Nella ricrittografia, il gateway decrittografa un metodo in entrata e ne crea uno nuovo per il tratto successivo. Ciò consente l'ispezione del contenuto o il cambio di protocollo, ma crea un momento in chiaro entro il confine del gateway. Il catalogo delle funzioni EPG descrive esplicitamente scenari in cui messaggi protetti internamente vengono brevemente decrittografati al gateway per ispezionarli o convertirli nello standard del destinatario ([Kiteworks: Internal Encryption and Encryption Gateway](https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf)). DLP, antivirus, file temporanei, dump di memoria, log di debug e accesso amministrativo fanno allora parte del confine crittografico.

Se manca una chiave destinatario direttamente utilizzabile, il recapito tramite portale sposta l'accesso al testo in chiaro in un'applicazione web. Identità web, sessione e messaggio memorizzato diventano così parte del modello operativo.

## WebMail e recapito tramite portale

Le FAQ ufficiali sulla migrazione confermano che EPG fornisce una funzione corrispondente a Totemomail WebMail per destinatari senza crittografia e-mail adeguata. Le istruzioni pubbliche per operatori di Totemomail mostrano il tipico flusso utente: notifica, password iniziale o link di registrazione, cambio password, accesso al portale e scelta tra WebMail, S/MIME o OpenPGP ([Kiteworks: Migration FAQ, domanda 7](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf), [L-Bank: Sichere E-Mail-Kommunikation](https://www.l-bank.info/allgemein/rechtliches/e-secure/sichere-e-mail-kommunikation.html)).

Il recapito tramite portale è un percorso distinto di archiviazione e identità. L'e-mail di notifica non contiene necessariamente il messaggio riservato; il contenuto può trovarsi sul portale o in un envelope protetto da password. Per gli amministratori sono quindi rilevanti almeno hostname, certificato TLS, stato dell'account, procedure di password e reset, periodo di conservazione, chiavi del messaggio, cronologia di recapito e processo di cancellazione. La variante scelta deve derivare dalla policy propria; le istruzioni di operatori esterni dimostrano possibili funzioni Totemomail, ma non impostazioni predefinite globali.

Un controllo di integrità del portale non deve fermarsi a HTTP 200. Il test sintetico deve includere registrazione o account di test esistente, login, apertura di un messaggio noto in anticipo, download di un allegato, risposta e logout. Dopo un ripristino, occorre verificare in particolare che i contenuti più vecchi del portale continuino a essere leggibili con le rispettive chiavi storiche.

## Directory e chiavi con conteggio delle licenze

Kiteworks documenta l'integrazione LDAP/AD, i ruoli e le policy centrali a livello di piattaforma. Per un gateway, la directory può classificare mittenti e destinatari, fornire condizioni di gruppo e associare identità a certificati o chiavi. Non appena una regola dipende da una query di directory, [LDAP](/kb/ldap#bind-sasl-und-tls-vertrauensgrenzen)-endpoint, Search Base, filtro, account Bind, fiducia TLS, timeout, cache e failover fanno parte del percorso del messaggio ([Kiteworks: security and integration overview](https://www.kiteworks.com/sites/default/files/resources/accellion-brochure-external-file-sharing-security-and-governance-platform_redirected_20191210.pdf), [RFC 4511](https://datatracker.ietf.org/doc/html/rfc4511), [RFC 4513](https://datatracker.ietf.org/doc/html/rfc4513)).

L'archivio delle chiavi non è un sottoprodotto dell'importazione utenti. Devono essere inventariate chiavi private di utenti e domini, chiavi pubbliche dei destinatari, certificati CA e intermedi, stato di fiducia, informazioni di revoca, connessioni Keyserver o MPKI e associazione a messaggi storici. Una pulizia della directory non deve mai rimuovere senza verifica una chiave privata necessaria per vecchi messaggi.

Un documento di packaging Kiteworks pubblicato definiva le postazioni EPG in modo che un utente interno consumasse una postazione inviando o ricevendo un messaggio crittografato oppure inviando un messaggio firmato; gli utenti interni che transitavano senza crittografia e senza firma venivano trattati diversamente. Il documento è una descrizione contrattuale datata, non una garanzia di licenza senza tempo ([Kiteworks: Enterprise 8.0 licensing and packaging](https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf)). Per l'esercizio, devono pertanto essere gestiti separatamente tre valori: identità tecnicamente note, identità realmente attive dal punto di vista crittografico e metrica contrattualmente concessa in licenza del proprio ordine.

Nel passaggio a EPG, quindi, non devono essere trasferiti o ricostruiti consapevolmente solo i percorsi SMTP, ma anche regole, identità, chiavi, stati del portale e prove.

## Migrazione da Totemomail a EPG

Kiteworks non descrive la migrazione come una ridenominazione in-place. Un trasferimento automatizzato copia informazioni dal database Totemomail esistente al database EPG. Il vecchio database esterno deve essere dismesso dopo la disattivazione del sistema Totemomail, poiché successivamente non è più sincronizzato. EPG stesso contiene database, sistema operativo e servizi web nella Hardened Virtual Appliance ([Kiteworks: Migration FAQ](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf)).

Un perimetro di migrazione completo comprende almeno:

- domini, route, identità relay e connettori nonché policy TLS;
- regole attive, priorità, trigger, eccezioni e percorsi di errore;
- chiavi S/MIME, OpenPGP, di dominio, TLS e CA, inclusa la cronologia;
- account WebMail, messaggi del portale, conservazione, template e nome pubblico;
- integrazioni con directory, SSO, SIEM, Syslog, DLP, archivio e applicazioni;
- ruoli, amministratori e accessi di audit e supporto;
- metrica di licenza, parco utenti attivo e limiti contrattuali;
- backup, ripristino, HA, DNS, monitoraggio e runbook.

Il test di accettazione deve verificare dati storici oltre ai nuovi messaggi. Per ogni vecchio tipo di crittografia viene aperto o decrittografato un messaggio creato prima della migrazione. Per ogni direzione viene inviata una nuova e-mail di test con e senza chiave destinatario disponibile. Vengono inoltre verificati casi negativi quali certificato non valido, hop successivo non raggiungibile e directory non disponibile. Solo in seguito il vecchio gateway può essere rimosso da DNS, connettori, allowlist e monitoraggio.

## Alta disponibilità con backup e ripristino

Le FAQ di migrazione EPG indicano clustering integrato, backup/ripristino, disaster recovery, alta disponibilità e distribuzione come caratteristiche della Hardened Virtual Appliance. Tuttavia, non affermano che ogni singola funzione EPG riceva automaticamente lo stesso RPO e RTO. Coda di posta, database, chiavi, contenuto del portale, audit log, configurazione e archiviazione esterna di oggetti o file devono essere assegnati separatamente secondo l'architettura ([Kiteworks: Migration FAQ – Hardened Virtual Appliance](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf)).

| Oggetto di ripristino | Evidenza minima | Lacuna tipica |
|---|---|---|
| Configurazione gateway | stato target esportato o salvato e ripristino su sistema sostitutivo | configurazione disponibile, mancano segreti o certificati |
| Chiavi private | elenco fingerprint prima e dopo il ripristino; decrittografia storica | chiavi attuali disponibili, vecchia rotazione persa |
| Policy | stato attivato più matrice di test positiva e negativa | l'esportazione non corrisponde alla regola effettivamente attiva |
| Stato posta e portale | oggetti di test di coda e WebMail noti in anticipo dopo il ripristino | la piattaforma si avvia, il vecchio messaggio è illeggibile |
| Cluster | failover con test SMTP, HTTPS, chiavi e log | VIP cambia, stato backend o IP in uscita no |
| Audit e Syslog | copia esterna degli eventi con correlazione temporale e dei messaggi | il ripristino locale cancella la cronologia dell'indagine |

Con Totemomail legacy, la procedura di backup dipende da versione e installazione. L'affermazione EPG sul backup integrato non deve essere applicata retroattivamente a un vecchio database esterno o a un archivio WebMail separato. Il percorso di supporto del produttore, la documentazione installata e un test di ripristino isolato sono qui le fonti primarie.

## Monitoraggio e gestione della sicurezza

Kiteworks conferma per EPG l'esportazione verso una o più destinazioni Syslog nonché l'integrazione Splunk. La documentazione della piattaforma descrive log e metadati normalizzati relativi alle e-mail e ad altri canali di comunicazione. Per gli amministratori di messaggistica rimane decisiva la correlazione funzionale: audit della piattaforma, evento SMTP, decisione della policy, operazione sulla chiave, accesso al portale e Message Trace a valle devono essere riuniti tramite tempo, identità e identificatore del messaggio ([Kiteworks: Migration FAQ – syslog](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf), [Kiteworks: EPG logging and audit](https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf)).

Un dashboard amministrativo dovrebbe visualizzare almeno accettazioni e rifiuti, età della coda per hop successivo, errori di policy, numero e tipo di azioni crittografiche, chiavi mancanti o non valide, scadenza dei certificati, latenza LDAP, errori di login al portale, stato del disco e del database, replica del cluster, età del backup e perdita Syslog. Una coda vuota è un riscontro positivo solo se contemporaneamente i messaggi vengono accettati e consegnati con successo all'hop successivo.

Syslog via UDP non offre consegna garantita. Per dati di audit da proteggere vanno pianificati trasporto TLS, buffering, autenticazione della destinazione, orario corretto e un allarme di guasto ([RFC 5424](https://datatracker.ietf.org/doc/html/rfc5424), [RFC 5425](https://datatracker.ietf.org/doc/html/rfc5425)). I log possono contenere mittenti, destinatari, oggetti, informazioni sui certificati o attività del portale; minimizzazione dei dati e protezione dell'accesso fanno pertanto parte della policy di logging.

Le segnalazioni storiche di sicurezza mostrano che anche l'interfaccia web fa parte della superficie d'attacco. Le voci NVD documentano, tra l'altro, precedenti vulnerabilità CSRF, di accesso a oggetti e XSS nei componenti web Totemomail. Da ciò non deriva alcuna affermazione su uno stato di patch arbitrario e non verificato, ma resta l'obbligo permanente di includere avvisi del produttore, esposizione, stato di supporto e test del portale web nella gestione delle patch ([NVD: CVE-2018-6563](https://nvd.nist.gov/vuln/detail/CVE-2018-6563), [NVD: CVE-2020-7918](https://nvd.nist.gov/vuln/detail/CVE-2020-7918), [NVD: CVE-2024-28063](https://nvd.nist.gov/vuln/detail/CVE-2024-28063)).

La diagnosi segue un messaggio di test dall'accettazione, passando per la decisione di policy e chiave, fino all'uscita SMTP o all'accesso al portale.

## Strumenti di diagnosi

La diagnosi inizia dal percorso di posta pubblicato e segue poi gli eventi EPG fino al recapito o a WebMail. La migrazione della piattaforma e lo stato della licenza vengono verificati solo quando è stato identificato il passo di elaborazione interessato.

### DNS e percorsi di posta pubblici

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Totemomail- und EPG-DNS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName -Type MX example.ch
Resolve-DnsName epg.example.ch -Type A,AAAA
Resolve-DnsName _25._tcp.epg.example.ch -Type TLSA
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig +short MX example.ch
dig +short A epg.example.ch
dig +short AAAA epg.example.ch
dig +short TLSA _25._tcp.epg.example.ch
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) e [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) mostrano l'MX pubblico, l'indirizzo del gateway ed eventualmente DANE-TLSA. La vista del resolver interno ed esterno deve essere verificata separatamente. Con Microsoft 365, l'MX descrive solo il primo hop Internet; connettori e regole di trasporto determinano l'ulteriore percorso.

### Raggiungibilità TCP, SMTP e TLS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Totemomail- und EPG-TLS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Test-NetConnection epg.example.ch -Port 25 -InformationLevel Detailed
Test-NetConnection epg.example.ch -Port 443 -InformationLevel Detailed
curl.exe --verbose --ssl-reqd smtp://epg.example.ch:25
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nc -vz epg.example.ch 25
nc -vz epg.example.ch 443
openssl s_client -starttls smtp -connect epg.example.ch:25 \
  -servername epg.example.ch -showcerts
```

  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) e [`nc`](https://man.openbsd.org/nc) dimostrano solo il percorso TCP. [`curl.exe`](https://curl.se/docs/manpage.html) di Windows richiede SMTP-STARTTLS; [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) mostra capacità, handshake, SNI e catena di certificati. Solo una transazione SMTP completa con risposta DATA positiva dimostra l'accettazione.

### Messaggio di test SMTP autorizzato

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Totemomail- und EPG-SMTP-Test">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
curl.exe --verbose --ssl-reqd --url smtp://epg.example.ch:25 `
  --mail-from test-sender@example.ch `
  --mail-rcpt test-recipient@example.net `
  --upload-file .\epg-test.eml
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
swaks --server epg.example.ch --port 25 --tls \
  --from test-sender@example.ch --to test-recipient@example.net \
  --data epg-test.eml
```

  </div>
</div>

[`curl`](https://curl.se/docs/manpage.html) e [`swaks`](https://jetmore.org/john/code/swaks/) inviano qui un messaggio di test controllato. Mittente, destinatario e destinazione devono essere espressamente autorizzati a tale scopo. Devono essere registrati risposta SMTP finale, ID gateway, policy attivata, chiave utilizzata o percorso del portale, hop successivo e arrivo effettivo. Un `250` su `RCPT TO` non costituisce ancora accettazione del contenuto del messaggio.

### Percorso WebMail o portale

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Totemomail- und EPG-Portaldiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Invoke-WebRequest -Method Head -Uri https://securemail.example.ch/
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
curl --head --verbose https://securemail.example.ch/
openssl s_client -connect securemail.example.ch:443 \
  -servername securemail.example.ch -showcerts
```

  </div>
</div>

[`Invoke-WebRequest`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest), [`curl`](https://curl.se/docs/manpage.html) e [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) verificano l'interfaccia HTTP e TLS. Un codice di stato riuscito non dimostra né il login né l'accesso alle chiavi. Un test sintetico del browser deve coprire accesso, apertura di un messaggio noto, allegato e risposta.

### Percorso dei pacchetti nel punto di misurazione corretto

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Totemomail- und EPG-Paketerfassung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
pktmon filter remove
pktmon filter add EPG-SMTP -p 25
pktmon start --capture --pkt-size 0 --file-name epg.etl
pktmon stop
pktmon pcapng epg.etl -o epg.pcapng
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
tcpdump -ni any -s 0 -w epg.pcap 'tcp port 25 or tcp port 443'
```

  </div>
</div>

[`pktmon`](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon) e [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) vedono il traffico soltanto nel punto di misurazione selezionato. Un client amministrativo non osserva automaticamente il percorso tra Exchange Online, Load Balancer ed EPG. Per l'analisi sono adatti un host autorizzato, Port Mirror o Cloud Flow Logs; SMTP prima di STARTTLS e i metadati web devono essere conservati con adeguata protezione.

## Storia tecnica

Totemo è stata fondata in Svizzera nel 2001 e ha sviluppato soluzioni per crittografia e-mail, Managed File Transfer e comunicazione mobile. Le convalide NIST del 2012 e del 2019 documentano lo sviluppo del Totemo Cryptographic Module basato su Java e della Totemo Security Platform su sistemi operativi appliance proprietari ([Entrust Partner Directory: Totemo](https://www.entrust.com/partners/directory/totemo), [NIST: TCM Security Policy 2012](https://csrc.nist.gov/csrc/media/projects/cryptographic-module-validation-program/documents/security-policies/140sp1884.pdf), [NIST CMVP Certificate 3500](https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/3500)).

Kiteworks ha acquisito totemo nel gennaio 2022. La documentazione del produttore descrive l'integrazione del codice Totemomail nella piattaforma Kiteworks e la ridenominazione in Email Protection Gateway. EPG ha combinato le funzioni esistenti di crittografia e-mail con la Hardened Virtual Appliance, ruoli condivisi, log normalizzati e altri canali di comunicazione Kiteworks ([Kiteworks: acquisition and EPG integration](https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf), [Kiteworks: Migration FAQ](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf), [Kiteworks: January 2022 acquisition](https://www.kiteworks.com/sites/default/files/resources/on-the-hunt-kiteworks-secure-content-strategy.pdf)).

La transizione è tecnicamente rilevante: il vecchio prodotto poteva funzionare con un database esterno; EPG acquisisce i relativi dati in un database integrato e sposta la manutenzione del sistema operativo e del database entro il confine dell'appliance. Runbook, monitoraggio e ripristino non devono quindi limitarsi a sostituire termini di prodotto, ma devono riassegnare superfici di controllo, dati memorizzati e configurazioni.

## Fonti

- [Kiteworks – FAQ for totemomail Customers Updating to EPG](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf)
- [Kiteworks – Top 6 Reasons to Add Email Protection Gateway](https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf)
- [Kiteworks – Enterprise 8.0 Licensing and Packaging](https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf)
- [Microsoft Marketplace – totemomail Encryption Gateway](https://marketplace.microsoft.com/en/marketplace/apps/totemo.totemo-azr-tm7?tab=overview)
- [IETF RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321)
- [IETF RFC 8551 – S/MIME 4.0](https://datatracker.ietf.org/doc/html/rfc8551)
- [IETF RFC 9580 – OpenPGP](https://datatracker.ietf.org/doc/html/rfc9580)
- [IETF RFC 3207 – SMTP STARTTLS](https://datatracker.ietf.org/doc/html/rfc3207)
- [Microsoft – Configure mail flow using connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow)
- [Microsoft – Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)
- [NIST CMVP – Certificate 3500, Totemo Cryptographic Module](https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/3500)
- [NIST – Totemo Cryptographic Module Security Policy 2019](https://csrc.nist.gov/CSRC/media/projects/cryptographic-module-validation-program/documents/security-policies/140sp3500.pdf)
- [Microsoft – Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud)
- [RFC 5322: Message-ID](https://datatracker.ietf.org/doc/html/rfc5322)
- [Kiteworks – Top 5 Reasons totemo and Kiteworks Are Better Together](https://www.kiteworks.com/sites/default/files/resources/kiteworks-product-brief-top-5-reasons-totemo-kiteworks-better-together.pdf)
- [IETF RFC 5652 – Cryptographic Message Syntax](https://datatracker.ietf.org/doc/html/rfc5652)
- [IETF RFC 5280 – X.509 PKI Certificate and CRL Profile](https://datatracker.ietf.org/doc/html/rfc5280)
- [IETF RFC 6960 – Online Certificate Status Protocol](https://datatracker.ietf.org/doc/html/rfc6960)
- [IETF RFC 7672 – SMTP Security via DANE](https://datatracker.ietf.org/doc/html/rfc7672)
- [IETF RFC 8461 – MTA Strict Transport Security](https://datatracker.ietf.org/doc/html/rfc8461)
- [L-Bank – Comunicazione e-mail sicura con Totemomail](https://www.l-bank.info/allgemein/rechtliches/e-secure/sichere-e-mail-kommunikation.html)
- [Kiteworks – Security and Integration Overview](https://www.kiteworks.com/sites/default/files/resources/accellion-brochure-external-file-sharing-security-and-governance-platform_redirected_20191210.pdf)
- [IETF RFC 4511 – LDAP Protocol](https://datatracker.ietf.org/doc/html/rfc4511)
- [IETF RFC 4513 – LDAP Authentication Methods and Security Mechanisms](https://datatracker.ietf.org/doc/html/rfc4513)
- [IETF RFC 5424 – Syslog Protocol](https://datatracker.ietf.org/doc/html/rfc5424)
- [IETF RFC 5425 – TLS Transport Mapping for Syslog](https://datatracker.ietf.org/doc/html/rfc5425)
- [NVD – CVE-2018-6563](https://nvd.nist.gov/vuln/detail/CVE-2018-6563)
- [NVD – CVE-2020-7918](https://nvd.nist.gov/vuln/detail/CVE-2020-7918)
- [NVD – CVE-2024-28063](https://nvd.nist.gov/vuln/detail/CVE-2024-28063)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [ISC BIND 9 – pagina man di dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – pagina man di nc](https://man.openbsd.org/nc)
- [curl – pagina man della riga di comando](https://curl.se/docs/manpage.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [swaks – strumento di test SMTP](https://jetmore.org/john/code/swaks/)
- [Microsoft Learn – Invoke-WebRequest](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest)
- [Microsoft Learn – pktmon](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon)
- [tcpdump – pagina del manuale](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [Entrust Partner Directory – Totemo](https://www.entrust.com/partners/directory/totemo)
- [NIST – Totemo Cryptographic Module Security Policy 2012](https://csrc.nist.gov/csrc/media/projects/cryptographic-module-validation-program/documents/security-policies/140sp1884.pdf)
- [Kiteworks – The Deal: On the Hunt](https://www.kiteworks.com/sites/default/files/resources/on-the-hunt-kiteworks-secure-content-strategy.pdf)
