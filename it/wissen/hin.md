---
title: "HIN: spazio di fiducia, Mail Gateway e Stargate"
blatt: "hin"
description: "HIN per amministratori di messaggistica e piattaforme: modello di fiducia e identità, HIN Mail, posta classica e Access Gateway, HIN Client, PKI, percorsi SMTP e del portale, Mailstore, architettura Stargate, migrazione, monitoraggio, ripristino e diagnosi."
fakten:
  - label: Piattaforma
    wert: Spazio di fiducia per il settore sanitario svizzero
    href: https://www.hin.ch/de/services/hin-mail/hin-mail.cfm
  - label: Gestore
    wert: Health Info Net AG · fondata nel 1996
    href: https://www.hin.ch/de/ueber-hin/unternehmen/geschichte.cfm
  - label: Modello di posta
    wert: HIN verso HIN automaticamente · esterno con contrassegno di riservatezza
    href: https://support.hin.ch/de/service/hin-mail-und-mobile.cfm
  - label: Edge classico
    wert: Mail e Access come appliance virtuali
    href: https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm
  - label: Architettura di destinazione
    wert: Postfix → MXEngine → Postfix
    href: https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf
  - label: Stack principale
    wert: OPA/Rego · PostgreSQL · Vault · MinIO
    href: https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf
  - label: Trasporto mesh
    wert: WireGuard · IDAgent · Port 19818
    href: https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf
  - label: Ancoraggio di fiducia
    wert: HIN Identità · S/MIME · chiavi e CSR
    href: https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf
  - label: Accesso web
    wert: HIN Client · Access Gateway · SAML · OAuth 2.0
    href: https://download.hin.ch/oauth2/doku/de/
  - label: Mailstore
    wert: separato dal trasporto del gateway · IMAP/POP/Webmail
    href: https://support.hin.ch/de/service/hin-gateway.cfm
  - label: Distribuzione
    wert: Linux · Docker Compose · immagini VM
    href: https://health-info-net-ag.github.io/Stargate-deployment/de/
  - label: Osservabilità
    wert: Prometheus · Promtail/Loki · Node Exporter
    href: https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf
werbung:
  - stargate
  - newsletter
ctaThemen:
  - hin-gateway
translationSourceHash: 4b3c48b30e1dfb939e082bba0637c401f51d48a6dc5ed8e71b90c9d8114b2c8c
translationModel: gpt-5.6-terra
translatedAt: 2026-10-06T11:19:28.424Z
translationReview: automatic
---

# HIN: spazio di fiducia, Mail Gateway e Stargate

HIN non è un singolo programma di cifratura, bensì uno spazio di fiducia settoriale composto da identità verificate, servizi di piattaforma centrali e componenti di accesso o gateway. **HIN Mail** protegge la comunicazione e-mail, **HIN Access** media l'accesso ad applicazioni web protette e una **HIN Identità** collega la persona o l'organizzazione autorizzata al materiale crittografico delle chiavi. Nelle istituzioni, i componenti Mail e Access si trovano al confine tra l'infrastruttura propria e la piattaforma HIN ([HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm), [HIN adesione collettiva con gateway](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm)).

Per gli amministratori di messaggistica occorre distinguere quattro spazi di stato. Il server di posta locale possiede mailbox, connettori e code. Il gateway assume le decisioni di trasporto, fiducia e protezione. La piattaforma HIN fornisce servizi di directory, identità, chiavi, posta e accesso. Per i destinatari esterni alla HIN Community si aggiunge un percorso web e di autenticazione. Un guasto in uno spazio non è automaticamente un guasto in tutti gli altri; una porta SMTP raggiungibile, ad esempio, non dimostra né un'identità HIN valida né una consegna riuscita tramite portale ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm), [HIN Mail a non membri](https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm)).

Anche il nome del prodotto necessita di un inquadramento temporale. La generazione classica di gateway comprende un Mail Gateway, MGW, e, a seconda del contratto, un Access Gateway, AGW. HIN descrive come successore il nodo mesh abilitato alla posta sviluppato nel progetto **Stargate**. Rimane compatibile con SMTP rispetto ai sistemi di posta locali, ma modifica l'architettura, la distribuzione delle chiavi, la distribuzione e il trasporto tra organizzazioni. Le affermazioni sulla piattaforma di destinazione non devono quindi essere trasferite senza verifica a un MGW esistente, e viceversa ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm)).

La spiegazione segue un messaggio dal mittente, attraverso l'identità HIN e il gateway, fino al destinatario. In seguito vengono trattati i cambiamenti di piattaforma, le dipendenze, la diagnosi, il materiale delle chiavi e il ripristino.

## Approccio architetturale: spazio di fiducia con componenti edge

Il modello collettivo classico fornisce HIN Mail e HIN Access come appliance virtuali. HIN documenta la cifratura e la firma S/MIME a livello di dominio di posta nonché un audit trail; l'Access Gateway opera come provider di identità locale, assegna eID HIN alle richieste verificate e può integrare servizi di directory e autenticazione esistenti ([HIN adesione collettiva con gateway](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm)).

Questa costruzione è un **modello di fiducia edge**: l'organizzazione controlla server di posta, DNS, firewall, consegna interna e risorsa gateway locale; HIN gestisce i servizi di fiducia e piattaforma sovraordinati. Il completamento SMTP positivo trasferisce la responsabilità di un messaggio concreto all'hop successivo; non afferma che il successivo percorso HIN, del portale o del destinatario sia già concluso. Per il nuovo stack gateway, HIN attribuisce esplicitamente al cliente DNS e reputazione della posta ([HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf), [RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321)).

Stargate sposta questo confine verso un nodo mesh riferito all'organizzazione. HIN descrive un'architettura cloud-native a microservizi con API REST, identità decentralizzate, gestione distribuita delle chiavi e policy programmabili. Per organizzazione è prevista un'istanza dedicata; HIN indica immagini virtuali e container come forme di distribuzione, nonché ambienti OpenShift e Kubernetes nella descrizione del prodotto. Si tratta di un'architettura di destinazione pubblicata, non della prova che ogni organizzazione HIN esistente sia già gestita in questo modo ([HIN Gateway: architettura e distribuzione](https://support.hin.ch/de/service/hin-gateway.cfm), [Descrizione del prodotto HIN Gateway](https://www.hin.ch/de/services/hin-mail/hin-gateway.cfm)).

## Stack tecnologico e responsabilità

Lo stack pubblicamente documentato è composto da più generazioni e non deve essere mescolato in un unico monolite:

| Livello | Implementazione del nuovo gateway | Ambito di stato e guasto | Evidenza per l'amministratore |
|---|---|---|---|
| Perimetro SMTP | **Postfix Relay** per accettazione, retry, routing DNS e consegna | Il trasporto è separato dalla decisione sul contenuto | Risposta finale SMTP, ID coda, next hop, MX e PTR |
| Elaborazione | **MXEngine** per ingest HTTP/SMTP, trasformazione e Delivery Strategy | Gli errori di elaborazione restano distinguibili dai retry di Postfix | Message-ID, evento MXEngine, trasformazione, restituzione a Postfix |
| Policy | **Open Policy Agent con Rego**, opzionalmente sincronizzato tramite Git | Lo stato delle regole è uno stato di dati e versioni | Revisione della policy, input, risultato, approvazione e rollback |
| Identità e crittografia | **S/MIME Keys Client**, **IDAgent**, Issuer e Verifier | CSR, certificato, chiave privata e identità peer sono oggetti separati | Fingerprint, titolare, scadenza, Issuer, peer e rotazione |
| Persistenza | **PostgreSQL** per servizio, **Vault** per segreti, **MinIO** per messaggi e allegati | Database, Secret Store e archiviazione a oggetti hanno confini di ripristino propri | Volume, ora del backup, test di ripristino e verifica di coerenza per servizio |
| Trasporto mesh | **WireGuard** tramite IDAgent | Lo stato del canale non coincide con la consegna SMTP | Chiave peer, endpoint, handshake, porta 19818 ed evento SMTP successivo |
| Osservabilità | **Promtail → Loki**, **Node Exporter**, metriche compatibili con Prometheus e Version Collector | Trasporto log, metriche host e salute del servizio possono guastarsi separatamente | Liveness, età dello scrape, arrivo log, risorse host e base temporale |
| Distribuzione | Linux, **Docker Compose**, volumi Docker persistenti; immagini VM come modalità di installazione | Host, container, immagini e volumi hanno cicli di vita diversi | Immagine approvata, configurazione Compose, inventario dei volumi e test di riavvio |

HIN documenta esplicitamente il flusso dei messaggi come `External SMTP → Postfix → MXEngine → Postfix → External SMTP`. Il passaggio forzato a MXEngine impedisce un bypass delle policy; Postfix rimane responsabile della consegna e dei retry. OPA/Rego mantiene le regole aziendali fuori dal codice applicativo. PostgreSQL, Vault e MinIO memorizzano classi di stato diverse e non devono essere trattati nel backup come un unico file system ([HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf), [HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf)).

La documentazione tecnica d'installazione menziona distribuzioni Linux delle famiglie compatibili con RHEL, nonché Ubuntu e Debian, il funzionamento Docker Compose su un singolo host e immagini VM per più piattaforme. Questa dichiarazione di supporto dipende dalla versione e dal rollout; fa fede la documentazione HIN approvata al momento dell'installazione, non un elenco di distribuzioni riportato staticamente ([HIN Stargate Deployment](https://health-info-net-ag.github.io/Stargate-deployment/de/)).

Il Mailstore rimane un confine distinto: secondo HIN, Stargate agisce come Mail Transport Agent, mentre IMAP continua a essere mappato sulla piattaforma Zimbra esistente. HIN Access costituisce inoltre un percorso autonomo di autenticazione e autorizzazione tramite client, AGW o Access Control Service ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm), [Manuale HIN Client 3](https://download.hin.ch/documentation/HIN_Client3_Handbuch_de_v1.1.pdf)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-hin.svg?v=20260813" title="Interaktive Infografik: HIN Vertrauensraum mit lokalem Mailserver, klassischem Mail und Access Gateway, HIN Identität, Mailplattform, Nichtmitglieder-Portal sowie Stargate-Zielarchitektur und Betriebssignalen" loading="lazy">
  <a href="/images/kb-interaktiv-hin.svg?v=20260813">Aprire direttamente il grafico interattivo</a>.
</iframe>

## Identità e PKI con autorizzazione

Un'identità HIN è più di un indirizzo e-mail. Durante l'attivazione, HIN Client genera una coppia di chiavi; la password sblocca il materiale locale delle chiavi. Nel nuovo gateway, S/MIME Keys Client genera chiavi crittografiche e Certificate Signing Requests, mentre Vault conserva chiavi private, credenziali e configurazioni sensibili. Un certificato associa una chiave pubblica a un'identità denominata, ma non sostituisce un'autorizzazione riferita all'applicazione ([Manuale HIN Client 3](https://download.hin.ch/documentation/HIN_Client3_Handbuch_de_v1.1.pdf), [HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf), [RFC 5280](https://datatracker.ietf.org/doc/html/rfc5280)).

Il ciclo di vita fa parte del runbook IAM: registrazione, attivazione, cambio dispositivo, modifica di ruolo o nome, blocco, nuova registrazione in caso di sospetto sulla chiave e uscita. Dopo l'ordine, HIN richiede una verifica dell'identità e distingue le eID personali dall'ID dell'organizzazione o del dispositivo nel modello collettivo. Un trasporto di posta funzionante non deve essere considerato prova che un'identità precedente o assegnata erroneamente non disponga più dell'accesso ([HIN Identità](https://support.hin.ch/de/service/hin-identitaet.cfm), [HIN adesione collettiva con gateway](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm)).

HIN Access e HIN Mail condividono lo spazio di fiducia, ma non lo stesso flusso di protocollo. Access Control Service può richiedere un livello di autenticazione tramite SAML `RequestedAuthnContext`; HIN documenta profili per password e MFA. Per le integrazioni, HIN fornisce inoltre flussi OAuth 2.0 per Authorization Code e Client Credentials. Autenticazione, emissione del token e autorizzazione da parte dell'applicazione di destinazione devono essere registrate separatamente ([HIN Authentication Context](https://support.hin.ch/de/thema/hin-access/pwd.cfm), [Integrazione OAuth2 HIN](https://download.hin.ch/oauth2/doku/de/), [SAML 2.0 Core](https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf), [RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749)).

Solo dopo aver chiarito identità e PKI è possibile valutare il percorso della posta. Gateway e policy decidono, in base a mittente, destinatario e destinazione, quale metodo di protezione applicare.

## Percorsi della posta e decisioni di protezione

Considerando i componenti HIN coinvolti, ora è possibile spiegare il percorso concreto del messaggio. I casi seguenti si distinguono in base alla posizione di mittente e destinatario e al servizio che prende la decisione di protezione.

### Tra partecipanti HIN

HIN descrive i messaggi tra indirizzi HIN come trasmessi automaticamente in modo conforme alla protezione dei dati. La piattaforma contrassegna lo stato di integrità nell'oggetto con `[HIN secured]` oppure `[Not secured by HIN]`. Questo contrassegno è un segnale per l'utente, ma non una correlazione tecnica sufficiente: per un incidente occorrono inoltre mittente e destinatario dell'envelope, Internet `Message-ID`, catena `Received`, evento gateway e finestra temporale ([HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm), [HIN Mail e Mobile](https://support.hin.ch/de/service/hin-mail-und-mobile.cfm), [RFC 5322](https://datatracker.ietf.org/doc/html/rfc5322)).

Un percorso organizzativo classico può essere descritto come `Mailserver → SMTP → MGW → HIN → MGW → SMTP → Mailserver`. Ogni stazione termina una sessione e può possedere una propria coda. HIN descrive il modello collettivo come cifratura e firma S/MIME a livello di dominio di posta; la consegna locale prima e dopo questo confine gateway rimane un ambito di protezione e operativo distinto ([HIN adesione collettiva con gateway](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm), [RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321)).

### A persone senza adesione HIN

Per destinatari privi di indirizzo HIN, secondo la documentazione HIN il mittente deve contrassegnare esplicitamente il messaggio come riservato, ad esempio con `(Vertraulich)` nell'oggetto. Il destinatario apre il contenuto protetto tramite un percorso web e si autentica con numero di cellulare e codice SMS; è possibile una risposta sicura. Si creano così ulteriori stati: e-mail di notifica, oggetto del portale, registrazione del destinatario, secondo fattore, conservazione e canale di risposta ([HIN Mail a non membri](https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm), [HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm)).

Una notifica consegnata non equivale a un messaggio letto. I test sintetici dovrebbero pertanto coprire l'intero percorso fino al login, all'apertura, all'allegato e alla risposta. Gli inoltri dal portale possono uscire dal percorso di protezione; HIN segnala espressamente che determinati tipi di inoltro vengono inviati senza cifratura. Tali azioni dell'utente rientrano nella formazione, nel modello DLP e nell'analisi degli incidenti ([HIN Mail a non membri](https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm)).

### Dispositivi, applicazioni e invii di massa

Il Mail Gateway classico può fungere da interfaccia SMTP per dispositivi interni; HIN documenta questa capacità anche per Stargate. La pagina dei servizi HIN Mail indica che l'invio di sistema tramite gateway richiede una licenza separata. Scanner, sistemi informativi clinici, applicazioni di laboratorio e processi batch necessitano quindi di un percorso documentato dedicato per mittente, relay, quantità ed errori, anziché di un uso silenzioso del flusso di posta degli utenti ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm), [HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm)).

## Routing SMTP e limiti di accettazione

Per ogni direzione, il gateway deve sapere con precisione quali domini siano locali, autorevoli, da inoltrare o da rifiutare. Una responsabilità poco chiara tra Exchange, cloud connector, Secure Mail Gateway e HIN edge genera bypass o loop. [SMTP](/kb/smtp) prescrive righe di trace e descrive il rilevamento dei loop; il trasferimento effettivo di responsabilità avviene solo con una risposta positiva dopo il contenuto completo del messaggio ([RFC 5321, informazioni di trace](https://datatracker.ietf.org/doc/html/rfc5321#section-4.4), [RFC 5321, DATA](https://datatracker.ietf.org/doc/html/rfc5321#section-4.1.1.4)).

Per ogni route, la documentazione operativa deve contenere almeno i seguenti valori:

- listener locale, porta e reti sorgenti o identità peer attese;
- nome EHLO, domini envelope e aree relay autorizzate;
- sequenza di filtro antispam/antimalware, gateway HIN e sistema di posta interno;
- hop successivo, risoluzione DNS o Smarthost e requisito [TLS](/kb/tls);
- comportamento in caso di piattaforma HIN, componente di identità o componente di chiavi non raggiungibile;
- età della coda, piano di retry, numero massimo di hop e responsabilità dei bounce;
- percorsi eccezionali per dispositivi, invio di sistema e coesistenza durante la migrazione.

Per Exchange Online si tratta di una questione di connettori e non solo di DNS. Microsoft documenta il routing verso gateway di terze parti e condizioni dei connettori basate su certificati o IP. Le FAQ HIN confermano il supporto di base per architetture ibride e vicine a Microsoft 365, ma per la configurazione concreta rimandano a documentazione di migrazione specifica del cliente ([Microsoft: flusso di posta tramite connettori](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow), [Microsoft: flusso di posta cloud di terze parti](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud), [HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm)).

## Mailstore e accesso client con token

L'accesso alle mailbox e il trasporto gateway sono modelli operativi diversi. HIN pubblica per gli account personali HIN Mail IMAP sulla porta 993 con TLS e Message Submission sulla porta 587 con STARTTLS; come password viene utilizzato un Mail Token generato. Anche POP sulla porta 995 è documentato, ma in genere scarica i messaggi localmente e può rimuoverli dal server. I ruoli dei protocolli corrispondono a [IMAP](/kb/apache-james#protokolle-tls-und-ports), POP3 e Message Submission, non al trasporto da gateway a gateway ([HIN Mail Token Service](https://support.hin.ch/de/service/hin-mail-und-mobile/token-service-mail-client.cfm), [Configurazione POP HIN](https://support.hin.ch/de/service/hin-mail-und-mobile/mail-clients-einrichten-pop.cfm), [RFC 9051](https://datatracker.ietf.org/doc/html/rfc9051), [RFC 1939](https://datatracker.ietf.org/doc/html/rfc1939), [RFC 6409](https://datatracker.ietf.org/doc/html/rfc6409)).

Il manuale HIN Client documenta la sostituzione dello storico proxy di posta locale con l'accesso basato su token. Per i server terminali, HIN indica espressamente che, con proxy disattivato, invio e ricezione avvengono tramite Mail Token. Ciò è rilevante per la storia tecnica: il client era originariamente un intermediario di comunicazione per HTTP, SMTP, POP e IMAP; i modelli operativi successivi separano maggiormente l'accesso del client di posta dal client di identità HIN ([Manuale HIN Client 3](https://download.hin.ch/documentation/HIN_Client3_Handbuch_de_v1.1.pdf), [HIN Client su server terminali](https://support.hin.ch/de/thema/hin-client/hin-client-auf-terminalserver.cfm)).

Nelle FAQ Stargate, HIN descrive il Mail Storage Agent esistente come basato su Zimbra e inizialmente non interessato dal nuovo trasporto gateway. Un flusso di posta Stargate riuscito non dimostra pertanto né la disponibilità IMAP né la coerenza delle mailbox. Viceversa, Webmail può funzionare mentre il connettore organizzativo o il trasporto mesh è guasto ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm)).

Il percorso Mailstore classico non è l'unica forma operativa. Stargate sposta funzioni e responsabilità e deve pertanto essere compreso come un percorso autonomo per messaggi e amministrazione.

## Stargate come architettura di destinazione

Stargate mira a mantenere la compatibilità e-mail al perimetro e al contempo a consentire uno scambio di dati decentralizzato più generale. HIN menziona Self-Sovereign Identity, Data Mesh, microservizi, API RESTful, componenti open source e un nodo mesh abilitato alla posta. Per il canale tra istanze Stargate è annunciato un trasporto applicazione-applicazione basato su [WireGuard](https://www.wireguard.com/protocol/); HIN lo distingue espressamente da un tunnel VPN generale ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm)).

Per gli amministratori ne deriva una traccia suddivisa in tre parti:

1. Localmente, SMTP con risposta di accettazione, coda e connettore rimane il confine verificabile.
2. Tra i nodi mesh si aggiungono stati di identità, discovery, chiavi e canale WireGuard.
3. Alla destinazione si crea nuovamente un percorso SMTP locale verso il sistema di posta ricevente.

Le panoramiche di sistema pubblicate documentano Postfix, MXEngine, OPA/Rego, PostgreSQL, Vault, MinIO, IDAgent, Promtail/Loki e metriche compatibili con Prometheus. Non vi sono indicate né un linguaggio di programmazione né un albero sorgente completo dei servizi del prodotto; pertanto questa informazione non viene dedotta da immagini container o progetti esterni con lo stesso nome ([HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf), [HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf)).

I confini di rete del nuovo gateway sono documentati pubblicamente in modo insolitamente concreto:

| Porta e direzione | Ruolo | Significato operativo |
|---|---|---|
| TCP 25 in ingresso e in uscita | Accettazione SMTP e consegna basata su MX | Esposizione Internet, reputazione, coda e routing next-hop |
| TCP 8084 in ingresso | Callback HTTP del Sealer remoto | secondo HIN intenzionalmente senza ulteriore livello TLS, poiché il payload stesso è cifrato |
| TCP e UDP 19818 in entrambe le direzioni | WireGuard tra IDAgent | Verificare insieme chiave peer, endpoint, NAT e firewall |
| TCP 443 e 4433 in uscita | Registry, CA S/MIME, Sealer, Issuer, logging e Verifier | Dipendenza dalla piattaforma nonostante il funzionamento gateway locale |
| TCP e UDP 53 in uscita | Risoluzione MX, SPF, A/AAAA e PTR | Routing e decisioni di sicurezza dipendono da [DNS](/kb/dns) |

Le porte locali di diagnosi e servizio esposte, inclusi PostgreSQL, Vault, MinIO ed endpoint delle metriche, secondo HIN non devono essere raggiungibili da Internet pubblico. Gli accessi amministrativi alle porte 22, 443, 8180 o 8190 devono appartenere a una rete di gestione definita e non a un'abilitazione generalizzata su Internet ([HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf), [HIN Stargate Deployment](https://health-info-net-ag.github.io/Stargate-deployment/de/)).

HIN descrive la migrazione come una configurazione parallela del nuovo gateway. Solo dopo configurazione, test e conferma della prontezza operativa, il cliente decide il passaggio. Gli indirizzi IP esistenti possono in linea di principio essere riutilizzati, ma la configurazione deve essere tradotta. Un runbook di cutover sicuro contiene quindi almeno inventario, esportazione, route parallela, matrice di test, criterio di commutazione, percorso di ritorno, gestione della coda e rollback chiaro ([HIN Gateway: migrazione](https://support.hin.ch/de/service/hin-gateway.cfm)).

## Alta disponibilità con backup e ripristino

HIN documenta per il proprio lato Stargate un cluster OpenShift ridondante, ma non pianifica automaticamente due macchine virtuali ridondanti lato cliente. Queste affermazioni non devono essere riunite in una disponibilità elevata end-to-end generale. Un servizio di piattaforma ridondante non protegge da un singolo hypervisor locale, una regola di connettore errata, una chiave scaduta o un firewall bloccato ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm)).

Nel nuovo gateway, i servizi persistono tramite volumi Docker. Vault si sigilla automaticamente dopo il riavvio di un container e richiede un processo di unseal controllato. La panoramica tecnica indica come standard backup giornalieri di database, chiavi e segreti Vault, file di configurazione e certificati; responsabilità e conservazione devono essere concordate con il cliente ([HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf)).

Un inventario di ripristino dovrebbe contenere almeno, per ogni generazione:

| Oggetto | Effetto della perdita | Passaggio di ripristino verificabile |
|---|---|---|
| Host, definizione Compose e container | Il nodo edge non si avvia | Fornire una nuova destinazione dall'immagine approvata e creare una runtime deterministica |
| Configurazione cliente e policy OPA/Rego | Route, policy o dominio errati | Ripristinare la versione, caricare la policy ed eseguire la matrice di test |
| Database PostgreSQL | Manca lo stato di policy, metadati o agenti | Eseguire il restore del database per servizio e la verifica referenziale |
| Chiavi Vault, segreti e certificati | Manca la decifratura o l'identità peer o organizzativa | Eseguire restore approvato, unseal e test funzionale crittografico |
| Messaggi e allegati MinIO | Manca il messaggio o l'oggetto d'archivio | Chiarire separatamente con HIN portata e conservazione e testare il ripristino dell'oggetto |
| Connettori e DNS | Bypass, loop o mancata consegna | Verificare la route in entrambe le direzioni con Message-ID univoco |
| Coda o prova di trasferimento | Duplicati o perdita di messaggi | Chiarire la responsabilità aperta per messaggio prima della commutazione |
| Mailbox e Mail Token | Accesso client interrotto | Validare separatamente tramite Webmail e IMAP/Submission |
| Log di audit e operativi | Impossibile ricostruire l'incidente | Testare base temporale, esportazione, conservazione e ingresso SIEM |

L'elenco pubblico dei backup cita database, Vault, configurazione e certificati, ma non menziona espressamente messaggi e allegati MinIO. Da ciò non si può dedurre né il loro backup né la loro esclusione intenzionale; proprio questo punto deve essere incluso per iscritto nell'accordo di retention, backup e restore prima della messa in produzione. Inoltre, uno snapshot VM generico non dimostra né uno stato PostgreSQL coerente né un Vault ripristinabile ([HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf), [HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf)).

In caso di guasto, il messaggio viene seguito dall'accettazione attraverso la decisione di identità e policy fino al percorso di consegna scelto; solo successivamente vengono riavviati o aggirati singoli componenti.

## Monitoraggio e triage degli incidenti

Un quadro operativo utile combina segnali locali e centrali:

- disponibilità ed età della coda per ogni SMTP next-hop;
- tassi di accettazione, inoltro e bounce con Message-ID correlabile;
- errori HIN, del portale, di identità, delle chiavi e delle policy separati;
- scadenza dei certificati, stato di registrazione e token;
- accesso alla mailbox tramite Webmail e IMAP indipendentemente dal gateway;
- risoluzione DNS e firewall tramite nomi anziché indirizzi IP HIN cablati;
- messaggi della piattaforma su [HIN Status](https://status.hin.ch/) più telemetria locale.

Il nuovo stack fornisce metriche di servizio compatibili con Prometheus, metriche host tramite Node Exporter, inoltro centrale dei log tramite Promtail a Loki e un Version Collector che interroga endpoint di liveness. Per l'allerta, almeno scrape mancante, ingresso log assente, età della coda Postfix, errori MXEngine, stato di seal Vault, capacità PostgreSQL e MinIO, stato peer WireGuard e scadenza dei certificati devono essere trattati separatamente ([HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf), [HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf)).

La documentazione firewall HIN raccomanda nomi DNS poiché gli indirizzi IP possono cambiare e cita tra l'altro percorsi HTTPS e SMTP per i servizi client. Una pagina di stato globale non può rilevare un guasto locale di DNS, NAT, MTU, connettore o chiavi. Il triage inizia pertanto dalla portata: un utente, un'identità, un dominio, una direzione, un gateway o la piattaforma ([Adattamenti firewall HIN](https://support.hin.ch/de/thema/hin-client/anpassung-firewall-hin-client.cfm), [HIN Status](https://status.hin.ch/)).

L'offerta collettiva classica menziona un audit trail per il flusso di posta; il nuovo gateway aggiunge log centrali strutturati. Per un'analisi completa della posta, tali evidenze devono essere correlate con i log SMTP locali e del sistema di posta. La sincronizzazione temporale e fusi orari uniformi sono requisiti operativi, non impostazioni estetiche ([HIN adesione collettiva con gateway](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm), [HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf)).

## Strumenti di diagnosi

La diagnosi inizia dal nome pubblico o interno e segue poi il percorso effettivo della posta. Solo quando DNS, connessione e certificato sono corretti vengono valutati lo stato del gateway, la coda e gli eventi specifici HIN.

### DNS ed endpoint HIN

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für HIN-DNS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName gateway.hin.ch -Type A
Resolve-DnsName gateway.hin.ch -Type AAAA
Resolve-DnsName smtp.mail.hin.ch -Type A
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig +short A gateway.hin.ch
dig +short AAAA gateway.hin.ch
dig +short A smtp.mail.hin.ch
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) e [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) mostrano se i nomi documentati siano risolvibili dalla prospettiva del resolver effettivamente utilizzata. Ciò è più importante di un valore IP copiato, specialmente con split DNS e proxy; HIN raccomanda espressamente nomi DNS anziché indirizzi fissati a lungo termine ([Adattamenti firewall HIN](https://support.hin.ch/de/thema/hin-client/anpassung-firewall-hin-client.cfm)).

### Raggiungibilità TCP e TLS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für HIN-TCP- und TLS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Test-NetConnection hin-gateway.example.ch -Port 25 -InformationLevel Detailed
Test-NetConnection hin-gateway.example.ch -Port 19818 -InformationLevel Detailed
curl.exe --verbose --ssl-reqd smtp://hin-gateway.example.ch:25
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nc -vz hin-gateway.example.ch 25
nc -vz hin-gateway.example.ch 19818
nc -vzu hin-gateway.example.ch 19818
openssl s_client -starttls smtp -connect hin-gateway.example.ch:25 \
  -servername hin-gateway.example.ch -showcerts
```

  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) e [`nc`](https://man.openbsd.org/nc) documentano il percorso TCP. La chiamata UDP di `nc` può al massimo suggerire la raggiungibilità; poiché WireGuard scarta senza risposta i pacchetti non autorizzati, solo l'handshake peer autenticato costituisce una prova affidabile per la porta 19818. Windows-[`curl.exe`](https://curl.se/docs/manpage.html) e [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) verificano il perimetro SMTP/STARTTLS. Un handshake riuscito non dimostra ancora l'elaborazione della policy o la consegna del messaggio ([HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf), [RFC 8446](https://datatracker.ietf.org/doc/html/rfc8446)).

### Transazione SMTP controllata

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für einen autorisierten HIN-Gateway-SMTP-Test">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
curl.exe --verbose --url smtp://hin-gateway.intern.example:25 `
  --mail-from hin-test@example.ch `
  --mail-rcpt test-recipient@example.net `
  --upload-file .\hin-test.eml
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
swaks --server hin-gateway.intern.example --port 25 \
  --from hin-test@example.ch --to test-recipient@example.net \
  --data hin-test.eml
```

  </div>
</div>

[`curl`](https://curl.se/docs/manpage.html) e [`swaks`](https://jetmore.org/john/code/swaks/) devono essere utilizzati solo contro un listener e un destinatario di test espressamente autorizzati. Vengono rilevati la risposta finale dopo `DATA`, l'ID coda locale, l'evento gateway, il percorso di protezione selezionato, l'hop successivo e l'arrivo effettivo. Un `250` su `RCPT TO` non costituisce ancora l'accettazione del contenuto del messaggio ([RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321)).

### Stati socket locali

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für lokale HIN-Gateway-Socketdiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-NetTCPConnection -State Listen,Established |
  Where-Object LocalPort -In 25,443,587,993
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
ss -tanp '( sport = :25 or sport = :443 or sport = :587 or sport = :993 )'
```

  </div>
</div>

[`Get-NetTCPConnection`](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection) e [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) mostrano listener locali e sessioni TCP stabilite. Sono utili solo laddove l'amministratore abbia accesso all'host interessato; un prodotto appliance o container gestito non deve essere modificato tramite accessi shell non documentati.

### Percorso dei pacchetti nel punto di misura corretto

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für HIN-Paketerfassung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
pktmon filter remove
pktmon filter add HIN-SMTP -p 25
pktmon start --capture --pkt-size 0 --file-name hin.etl
pktmon stop
pktmon pcapng hin.etl -o hin.pcapng
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
tcpdump -ni any -s 0 -w hin.pcap \
  'tcp port 25 or tcp port 443 or tcp port 587 or tcp port 993'
```

  </div>
</div>

[`pktmon`](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon) e [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) vedono solo il traffico nel punto di misura selezionato. Per Stargate, un trace SMTP locale non può spiegare completamente il canale mesh; sono inoltre necessari eventi gateway e di piattaforma. Le catture possono contenere indirizzi, oggetti o parti di protocollo non cifrate e devono essere trattate come dati operativi sensibili.

## Storia tecnica

FMH e Ärztekasse fondarono Health Info Net AG nel 1996, quando l'e-mail iniziava a diffondersi nel settore sanitario e l'invio di dati sensibili tramite la posta Internet ordinaria era riconosciuto come insufficientemente protetto. HIN iniziò quindi come fornitore di comunicazione e-mail protetta per i medici e si sviluppò in uno spazio di fiducia e accesso più ampio ([Storia aziendale HIN](https://www.hin.ch/de/ueber-hin/unternehmen/geschichte.cfm)).

L'architettura client mostra il cambiamento tecnico. HIN Client 1 e 2 sono stati sostituiti da HIN Client 3. Per ragioni di compatibilità, il client ha continuato a operare come proxy locale per browser e programmi di posta; per l'accesso web HIN documenta in seguito Challenge/Response, per gli account di posta il passaggio a token indipendenti su porte standard. La storia spiega perché le istruzioni d'installazione più vecchie menzionano porte proxy locali, mentre la documentazione più recente usa endpoint IMAP, POP e Submission diretti ([Manuale HIN Client 3](https://download.hin.ch/documentation/HIN_Client3_Handbuch_de_v1.1.pdf), [HIN Mail Token Service](https://support.hin.ch/de/service/hin-mail-und-mobile/token-service-mail-client.cfm)).

Il modello collettivo HIN classico raggruppava appliance Mail e Access nella rete del cliente. L'attuale pagina dei servizi documenta per questa generazione S/MIME a livello di dominio di posta, audit trail, un provider di identità locale e il collegamento di servizi di autenticazione e directory esistenti. Queste funzioni spiegano la separazione sviluppatasi tra trasporto della posta e accesso web ([HIN adesione collettiva con gateway](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm)).

Dal 2025 HIN ha introdotto una nuova consegna ai non membri; parallelamente sono state rinnovate l'infrastruttura di piattaforma e di accesso. La documentazione gateway pubblicata nel 2026 descrive Stargate come il successivo cambio generazionale: da un gateway di cifratura della posta esclusivo a un nodo decentralizzato cloud-native per posta e scambio strutturato di dati sanitari. La documentazione tecnica concretizza questo cambiamento con Postfix, MXEngine, OPA/Rego, PostgreSQL, Vault, MinIO e funzionamento containerizzato. Per un piano di migrazione non conta quindi soltanto la sostituzione di una VM, ma la nuova misurazione di identità, chiavi, trasporto, osservabilità e ripristino ([HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm), [HIN Access](https://support.hin.ch/de/thema/hin-access.cfm), [HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm), [HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf)).

## Fonti

- [HIN – HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm)
- [HIN – Adesione collettiva con gateway](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm)
- [Supporto HIN – HIN Gateway e Stargate](https://support.hin.ch/de/service/hin-gateway.cfm)
- [Supporto HIN – HIN Mail a non membri](https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm)
- [HIN – Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf)
- [RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321)
- [HIN – Descrizione del prodotto HIN Gateway](https://www.hin.ch/de/services/hin-mail/hin-gateway.cfm)
- [HIN – Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf)
- [HIN – Stargate Deployment](https://health-info-net-ag.github.io/Stargate-deployment/de/)
- [HIN – Manuale HIN Client 3](https://download.hin.ch/documentation/HIN_Client3_Handbuch_de_v1.1.pdf)
- [RFC 5280 – Internet X.509 PKI](https://datatracker.ietf.org/doc/html/rfc5280)
- [Supporto HIN – HIN Identità](https://support.hin.ch/de/service/hin-identitaet.cfm)
- [Supporto HIN – SAML Authentication Context](https://support.hin.ch/de/thema/hin-access/pwd.cfm)
- [HIN – Integrazione OAuth2](https://download.hin.ch/oauth2/doku/de/)
- [OASIS – SAML 2.0 Core](https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf)
- [RFC 6749 – OAuth 2.0](https://datatracker.ietf.org/doc/html/rfc6749)
- [Supporto HIN – HIN Mail e Mobile](https://support.hin.ch/de/service/hin-mail-und-mobile.cfm)
- [RFC 5322 – Internet Message Format](https://datatracker.ietf.org/doc/html/rfc5322)
- [Microsoft – Flusso di posta tramite connettori](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow)
- [Microsoft – Flusso di posta tramite un servizio cloud di terze parti](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud)
- [Supporto HIN – HIN Mail Token Service](https://support.hin.ch/de/service/hin-mail-und-mobile/token-service-mail-client.cfm)
- [Supporto HIN – Configurazione POP](https://support.hin.ch/de/service/hin-mail-und-mobile/mail-clients-einrichten-pop.cfm)
- [RFC 9051 – IMAP4rev2](https://datatracker.ietf.org/doc/html/rfc9051)
- [RFC 1939 – POP3](https://datatracker.ietf.org/doc/html/rfc1939)
- [RFC 6409 – Message Submission](https://datatracker.ietf.org/doc/html/rfc6409)
- [Supporto HIN – HIN Client su server terminali](https://support.hin.ch/de/thema/hin-client/hin-client-auf-terminalserver.cfm)
- [WireGuard – Protocol and Cryptography](https://www.wireguard.com/protocol/)
- [HIN Status](https://status.hin.ch/)
- [Supporto HIN – Adattamenti firewall per HIN Client](https://support.hin.ch/de/thema/hin-client/anpassung-firewall-hin-client.cfm)
- [Microsoft – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [ISC BIND – manuale dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – manuale nc](https://man.openbsd.org/nc)
- [curl – manuale della riga di comando](https://curl.se/docs/manpage.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [RFC 8446 – TLS 1.3](https://datatracker.ietf.org/doc/html/rfc8446)
- [swaks – Swiss Army Knife per SMTP](https://jetmore.org/john/code/swaks/)
- [Microsoft – Get-NetTCPConnection](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection)
- [Linux man-pages – ss](https://man7.org/linux/man-pages/man8/ss.8.html)
- [Microsoft – Packet Monitor](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon)
- [tcpdump – tcpdump(1)](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [HIN – Storia aziendale](https://www.hin.ch/de/ueber-hin/unternehmen/geschichte.cfm)
- [Supporto HIN – HIN Access](https://support.hin.ch/de/thema/hin-access.cfm)
