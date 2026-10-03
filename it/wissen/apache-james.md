---
title: "Apache James: server di posta modulare e piattaforma Mailet"
blatt: "apache-james"
description: "Apache James inquadrato dal punto di vista tecnico: protocolli e ruoli della posta, architettura basata su componenti, coda e pipeline Mailet, modello di mailbox e storage, varianti operative da PostgreSQL a Cassandra e sviluppo dal progetto Java Apache alla piattaforma di posta JVM."
fakten:
  - label: Nome completo
    wert: Java Apache Mail Enterprise Server
    href: https://james.apache.org/
  - label: Categoria
    wert: MTA, MDA, server di mailbox e piattaforma applicativa per la posta
    href: https://james.apache.org/documentation.html
  - label: Progetto
    wert: Apache Software Foundation
    href: https://projects.apache.org/committee.html?james
  - label: Runtime
    wert: JVM · Java 21 dalla versione 3.9
    href: https://james.apache.org/james/update/2025/09/25/james-3.9.0.html
  - label: Linguaggi
    wert: prevalentemente Java, singoli moduli in Scala
    href: https://github.com/apache/james-project
  - label: Protocolli
    wert: SMTP, LMTP, IMAP, POP3, ManageSieve, JMAP
    href: https://james.apache.org/server/feature-protocols.html
  - label: Stile architetturale
    wert: modulare, basato su componenti, Inversion of Control, guidato dagli eventi
    href: https://james.apache.org/
  - label: Backend
    wert: PostgreSQL/JPA o Cassandra · OpenSearch · RabbitMQ · S3
    href: https://james.apache.org/download.cgi
  - label: Configurazione
    wert: conf/*.xml e *.properties · variabili d'ambiente
    href: https://james.apache.org/server/config.html
  - label: Pacchettizzazione
    wert: distribuzioni ZIP e immagini Docker ufficiali
    href: https://james.apache.org/download.cgi
  - label: Amministrazione
    wert: WebAdmin REST API, CLI e metriche
    href: https://james.apache.org/server/manage-webadmin.html
  - label: Monitoraggio
    wert: Health Checks, Prometheus, JMX, log e Grafana
    href: https://james.apache.org/server/metrics.html
  - label: Licenza
    wert: Apache License 2.0
    href: https://www.apache.org/licenses/LICENSE-2.0
werbung:
  - newsletter
ctaThemen:
  - smtp-mailflow
translationSourceHash: adbfb005b83b16086ba55e53dd469f3aff1e5642364da5ab8b2da5d265a1ce51
translationModel: gpt-5.6-terra
translatedAt: 2026-10-03T09:23:44.236Z
translationReview: required
---

# Apache James: server di posta modulare e piattaforma Mailet

Apache James è un server di posta open source e al tempo stesso un toolkit per applicazioni la cui logica aziendale si basa sulle e-mail. Il nome significa **Java Apache Mail Enterprise Server**. James può ricevere e inoltrare messaggi tramite SMTP, gestire mailbox locali, renderle disponibili tramite IMAP, POP3 o JMAP e controllare l'intero flusso dei messaggi tramite componenti di elaborazione liberamente combinabili. Per questo il progetto si descrive non solo come server, ma anche come **piattaforma di Inversion of Control componibile modularmente sulla JVM** ([Panoramica del progetto Apache James](https://james.apache.org/)).

Questo duplice ruolo distingue James dai classici Mail Transfer Agent come Postfix e dalle appliance di sicurezza già pronte. Un amministratore può utilizzare James come puro relay SMTP, come server mailbox completo o come motore di posta integrato in un prodotto. Il controllo antispam, la crittografia, l'archiviazione o il routing specifico del dominio non derivano da un blocco funzionale rigido, bensì da una pipeline di **Matchers** e **Mailets**. Questo rende James eccezionalmente adattabile, ma trasferisce una parte della responsabilità di prodotto dal produttore all'organizzazione che lo gestisce.

La spiegazione segue un messaggio attraverso James: dai server di protocollo alla coda e alla pipeline Mailet, fino allo storage delle mailbox. Su questa base si sviluppano le varianti operative, la diagnostica e infine l'evoluzione tecnica del progetto.

## Inquadramento: MTA, MDA e piattaforma applicativa

In un sistema e-mail, non ogni componente svolge lo stesso ruolo. Un **Mail User Agent** (MUA) è il client dell'utente, ad esempio Thunderbird. Un **Mail Transfer Agent** (MTA) trasporta i messaggi tra sistemi. Un **Mail Delivery Agent** (MDA) deposita un messaggio nella mailbox di destinazione. James può essere contemporaneamente MTA e MDA; tramite i suoi moduli di protocollo e mailbox fornisce inoltre servizi lato server per i MUA. La panoramica ufficiale dei componenti elenca progetti separati per server, protocolli, Mailets, mailbox e test ([Apache James – Software Components](https://james.apache.org/documentation.html)).

| Ruolo | Implementazione in James | Punto di consegna |
|---|---|---|
| Trasporto dei messaggi | Server SMTP e LMTP, coda, Mailet di consegna remota | altri MTA, relay e gateway |
| Consegna locale | Pipeline Mailet e Mailbox API | utenti, domini e quote |
| Accesso alla mailbox | IMAP, POP3 e JMAP | client di posta e applicazioni web |
| Logica di filtro | Matchers, Mailets, Processors e Sieve | regole interne e servizi di controllo esterni |
| Amministrazione | WebAdmin REST API, CLI, Health Checks e metriche | automazione e monitoraggio |

James non è quindi **un client di posta** né un Secure Mail Gateway preconfigurato. Fornisce componenti per trasporto, consegna, archiviazione ed elaborazione. Se ne risulti un semplice relay, un servizio di posta multi-tenant o un gateway specifico per un prodotto dipende dalla distribuzione e dalla configurazione scelte.

## Protocolli, TLS e porte

James fornisce SMTP, LMTP, IMAP, POP3 e ManageSieve come servizi basati su TCP; JMAP e WebAdmin utilizzano HTTP ([Apache James – Protocol Servers](https://james.apache.org/server/feature-protocols.html)). A seconda del listener, TLS protegge una connessione cifrata fin dall'inizio oppure viene inserito in una sessione esistente tramite StartTLS. Il DNS non fa parte del processo James, ma è indispensabile per un MTA pubblico: i record MX determinano la destinazione, i record A e AAAA i relativi indirizzi e i record PTR influenzano la reputazione delle connessioni in uscita.

Il solo numero di porta non descrive ancora la semantica di sicurezza. La porta 25 è destinata al trasporto server-to-server; l'invio autenticato da parte dei client deve avvenire, secondo [RFC 6409](https://datatracker.ietf.org/doc/html/rfc6409), sulla porta 587. La porta 465 è nuovamente registrata da [RFC 8314](https://datatracker.ietf.org/doc/html/rfc8314) per Message Submission con crittografia implicita. Per IMAP e POP3 valgono gli stessi due modelli: connessione in chiaro con possibile StartTLS oppure instaurazione immediata di TLS.

| Servizio | Porte tipiche | Standard | Significato in James |
|---|---:|---|---|
| SMTP | 25, 587, 465 | [RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321) | ricezione, relay e submission |
| LMTP | configurabile, registrata 24 | [RFC 2033](https://datatracker.ietf.org/doc/html/rfc2033) | consegna locale con stato per destinatario |
| IMAP4rev2 | 143, 993 | [RFC 9051](https://datatracker.ietf.org/doc/html/rfc9051) | accesso sincrono alla mailbox |
| POP3 | 110, 995 | [RFC 1939](https://datatracker.ietf.org/doc/html/rfc1939) | recupero semplice dei messaggi |
| ManageSieve | 4190 | [RFC 5804](https://datatracker.ietf.org/doc/html/rfc5804) | gestione delle regole Sieve specifiche dell'utente |
| JMAP Mail | generalmente 443 | [RFC 8621](https://datatracker.ietf.org/doc/html/rfc8621) | accesso alla mailbox basato su HTTP per client moderni |

Le porte sono configurabili; è vincolante la combinazione di listener, protocollo, modalità TLS e autenticazione. La [IANA Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) rimane il riferimento per le assegnazioni registrate.

## Approccio architetturale

James segue un'**architettura basata su componenti**. Server di protocollo, coda, logica di elaborazione, mailbox, gestione degli utenti, indice di ricerca e amministrazione sono separati tra loro tramite API e composti mediante dependency injection. Le distribuzioni documentate per James 3.9 si basano su Google Guice; l'architettura Spring appartiene a una generazione precedente. Il disaccoppiamento non è solo un'organizzazione del codice: consente di utilizzare la stessa [Mailbox API](https://james.apache.org/mailbox/mailbox-api.html) con diversi livelli di persistenza e la stessa logica Mailet in profili server molto differenti.

Il percorso centrale dei dati è asincrono. Un listener SMTP non deve consegnare completamente un messaggio accettato prima di rispondere alla connessione. Inserisce un oggetto mail in una coda; uno **Spooler** lo estrae successivamente e lo fa passare attraverso il container Mailet. La coda separa così il carico di ricezione, il tempo di elaborazione e la disponibilità dei sistemi a valle. La documentazione operativa distribuita la definisce pertanto una componente obbligatoria di un server SMTP ([Apache James – Distributed Server Operations](https://james.apache.org/server/manage-guice-distributed-james.html)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 976" src="/images/apache-james-architektur.svg?v=20260813" title="Interaktive Infografik: technische Architektur und Nachrichtenfluss von Apache James" loading="lazy">
  <a href="/images/apache-james-architektur.svg?v=20260813">Apri l'infografica sull'architettura tecnica</a>
</iframe>

### Il percorso di elaborazione di un messaggio

1. **Ricezione del protocollo:** SMTP o LMTP verifica sessione, autenticazione, mittente dell'envelope e destinatari. Dopo la fine di `DATA` viene creato un oggetto interno `Mail` con envelope, contenuto MIME e attributi.
2. **Coda:** L'oggetto viene accodato in modo persistente o volatile. Solo da questo punto ricezione ed elaborazione sono disaccoppiate.
3. **Spooler:** I worker estraggono le voci dalla coda e le passano al container Mailet.
4. **Processor:** Un Processor nominato contiene un elenco ordinato di coppie Matcher/Mailet. Il Processor obbligatorio `root` costituisce il punto di ingresso.
5. **Matcher:** Un Matcher non modifica il messaggio, ma restituisce il sottoinsieme dei destinatari per i quali una condizione è soddisfatta.
6. **Mailet:** Il Mailet associato modifica il messaggio o l'envelope, attiva un effetto collaterale, consegna localmente o remotamente oppure dirama verso un altro Processor.
7. **Risultato:** Il messaggio finisce nella mailbox di un utente, nella consegna in uscita, in un Mail Repository per un trattamento successivo oppure è completato dopo un'azione riuscita.

Un dettaglio importante è la **suddivisione per destinatario**. Se un Matcher corrisponde solo a una parte dei destinatari, il container divide l'elaborazione in gruppi di destinatari corrispondenti e non corrispondenti. Le regole non si applicano quindi necessariamente a un intero messaggio MIME. Un Mailet può inoltre passare direttamente a un altro Processor tramite `ToProcessor`; la pipeline è quindi più simile a un grafo di elaborazione diretto che a un unico elenco lineare. La [documentazione ufficiale del container Mailet](https://james.apache.org/server/feature-mailetcontainer.html) descrive esattamente questo modello.

Un modello minimo e semplificato è il seguente:

```xml
<processor state="root" enableJmx="true">
  <mailet match="RelayLimit=30" class="ToRepository">
    <repositoryPath>cassandra://var/mail/relay-denied/</repositoryPath>
  </mailet>
  <mailet match="RecipientIsLocal" class="LocalDelivery" />
  <mailet match="All" class="RemoteDelivery" />
</processor>
```

L'ordine fa parte della semantica. Una regola con corrispondenza ampia all'inizio può rendere irraggiungibili le regole successive; un ciclo infinito tra Processors può occupare lo Spooler. James offre pertanto un comportamento di errore configurabile per ogni Matcher e Mailet, nonché Processors di errore dedicati ([Mailet Container Configuration](https://james.apache.org/server/config-mailetcontainer.html)).

L'architettura a componenti diventa concreta non appena un messaggio raggiunge la coda. A quel punto Processor, Matcher e Mailet determinano quali passaggi di elaborazione seguiranno e dove arriverà il risultato.

## Struttura tecnica

L'architettura descrive il percorso del messaggio; per l'installazione e l'esercizio occorre ora ricavarne un quadro concreto dei componenti. È decisivo quali runtime, storage e servizi aggiuntivi richieda effettivamente il profilo James scelto.

### Stack tecnologico e panoramica amministrativa

Per una prima classificazione del prodotto, sono più importanti dei nomi delle classi i confini operativi. La seguente panoramica condensa lo stack nelle domande da chiarire prima dell'installazione, dell'integrazione o della presa in carico di un ambiente esistente:

| Area | Tecnologia o artefatto | Cosa deve sapere l'amministratore |
|---|---|---|
| Runtime | Java 21, JVM; codice sorgente prevalentemente Java, singoli moduli Scala | heap, garbage collection, threading e patch JVM fanno parte dell'esercizio del server |
| Build e pacchetto | progetto Maven multimodulo; ZIP e immagini Docker | i Mailet personalizzati devono essere compatibili con la generazione James, Java e Jakarta |
| Wiring | Guice nella generazione 3.9; Spring nelle installazioni meno recenti | la distribuzione scelta determina i moduli e i file di configurazione disponibili |
| Configurazione | `conf/*.xml`, `conf/*.properties`, variabili d'ambiente | particolarmente importanti: `smtpserver.xml`, `mailetcontainer.xml`, `webadmin.properties`, file JMAP e backend |
| Elaborazione | MailQueue, Spooler, Processor, Matcher, Mailet | ricezione, elaborazione e consegna finale sono stati separati |
| Dati | PostgreSQL/JPA o Cassandra; opzionalmente S3, OpenSearch, RabbitMQ | origine, proiezione, coda e contenuto blob richiedono piani di ripristino separati |
| Amministrazione | WebAdmin REST API e `james-cli` | REST è più potente; la CLI è inclusa in ogni variante di wiring |
| Osservabilità | Health Checks, Dropwizard Metrics, Prometheus, JMX, log, Grafana | coda, Mailets, Matchers, protocolli e backend dispongono di metriche proprie |
| Sicurezza | keystore TLS, SMTP AUTH, JWT per WebAdmin, segmentazione di rete | WebAdmin senza JWT attivato non è protetto per impostazione predefinita |

Secondo il progetto, tutti i file di configurazione si trovano in `conf` o `conf/META-INF`; quali siano effettivamente applicati dipende dal wiring e dal backend. I valori possono essere ottenuti dall'ambiente con `${env:VARIABLE}` ([Apache James – Configuration](https://james.apache.org/server/config.html)). Questo è pratico per i container, ma non sostituisce la gestione dei segreti: certificati, chiavi private, chiavi JWT e password dei database dovrebbero essere forniti come secret montati o tramite la piattaforma di orchestrazione.

### Livello dei protocolli

Il progetto Protocols fornisce implementazioni estensibili di server per SMTP, LMTP, IMAP, POP3, ManageSieve e JMAP ([James Protocols](https://james.apache.org/server/feature-protocols.html)). I listener non sono cablati in modo fisso a uno storage specifico. IMAP e JMAP accedono tramite Mailbox API; SMTP passa i messaggi accettati alla coda e al container Mailet. In questo modo i protocolli possono essere scalati o disattivati indipendentemente dalla topologia del backend.

### Mailbox, Mail Repository e storage Blob

James distingue tre concetti di storage che non dovrebbero essere confusi nell'esercizio:

| Storage | Contenuto | Visibilità | Ripristino tipico |
|---|---|---|---|
| **Mailbox** | cartelle, messaggi, flag, UID, ACL e quote di un utente | IMAP/JMAP/POP3 | restore o replica del backend mailbox |
| **Mail Repository** | messaggi provenienti da percorsi di elaborazione come `error`, `relay-denied` o quarantena | solo amministrazione | correggere la causa e rielaborare il messaggio |
| **Blob Store** | contenuto MIME binario o oggetti di grandi dimensioni | referenziato indirettamente tramite metadati | backup coerente con metadati e riferimenti |

La [documentazione sulla persistenza](https://james.apache.org/server/feature-persistence.html) sottolinea che un Mail Repository **non** è la mailbox dell'utente. Questa separazione è preziosa per l'Incident Response: un messaggio difettoso può essere isolato, analizzato e reimmesso nella pipeline dopo una correzione, senza aggirare il modello mailbox.

### Event bus, ricerca e proiezioni

Le operazioni sulle mailbox generano eventi, ad esempio `MailboxAdded`, `MessageMoveEvent`, `FlagsUpdated` o modifiche delle quote. I listener aggiornano da questi eventi quote, indici di ricerca e altre proiezioni. Nel profilo distribuito RabbitMQ gestisce la comunicazione, OpenSearch la ricerca e Cassandra i metadati; i contenuti binari risiedono in un Object Store compatibile con S3. Questa scomposizione permette la scalabilità orizzontale, ma genera **consistenza eventuale** tra origine e proiezioni. Gli eventi dei listener non riusciti finiscono in un Event Dead Letter e devono essere monitorati e, se necessario, riconsegnati ([Distributed James – Mailbox Event Bus](https://james.apache.org/server/manage-guice-distributed-james.html#Mailbox_Event_Bus)).

### MIME, Sieve e autenticazione dei mittenti

Il progetto James comprende più del server. **Apache Mime4J** analizza strutture MIME in streaming o come modello a oggetti; **jSieve** implementa il linguaggio di filtro Sieve; **jSPF** e **jDKIM** forniscono librerie Java rispettivamente per il controllo del mittente e per la firma e verifica DKIM. Questi moduli sono progetti indipendenti e possono essere utilizzati anche al di fuori di un server James completo ([Apache James – Componenti](https://james.apache.org/documentation.html)).

Quali di questi componenti siano eseguiti su un nodo o distribuiti non è una mera questione di prestazioni. La scelta determina anche coerenza, riavvio e il numero di backend da monitorare.

## Varianti operative e scalabilità

Per James 3.9.0 Apache documenta diversi profili. Non si tratta semplicemente di installer differenti, ma di diversi modelli di consistenza, scalabilità ed esercizio. In questo stato la variante JPA è esplicitamente indicata come **legacy**; sono inoltre disponibili una distribuzione PostgreSQL e una distribuita ([Apache James – Downloads](https://james.apache.org/download.cgi)). I punti indicati nel grafico come **derivazione operativa** sono raccomandazioni dedotte e non dichiarazioni letterali del produttore.

<iframe class="kb-infographic" style="aspect-ratio: 1280 / 956" src="/images/apache-james-betriebsmodelle.svg?v=20260813" title="Interaktive Infografik: Apache-James-Betriebsmodelle und Technologiestacks" loading="lazy">
  <a href="/images/apache-james-betriebsmodelle.svg?v=20260813">Apri l'infografica di confronto dei modelli operativi</a>
</iframe>

| Profilo | Persistenza e servizi | Adatto a | Conseguenza operativa |
|---|---|---|---|
| JPA/Guice (legacy) | database H2 integrato o database SQL esterno; modello classico a server singolo | laboratorio, migrazione di installazioni meno recenti, piccole soluzioni speciali | pochi componenti, ma percorso strategico limitato e scalabilità verticale |
| PostgreSQL | PostgreSQL come nucleo; opzionalmente OpenSearch, RabbitMQ e storage compatibile con S3 | nuove installazioni a nodo singolo o multinodo con base relazionale | backup e HA sono ben noti; introdurre servizi aggiuntivi solo quando serve scalabilità |
| Distributed/Guice | Cassandra, RabbitMQ, OpenSearch e Object Store compatibile con S3 | grandi servizi scalabili orizzontalmente | più domini di errore, proiezioni, Dead Letters e controlli di coerenza più complessi |
| Memory | componenti In-Memory volatili | test e sviluppo | nessuna conservazione dei dati in produzione |

La versione 3.9 evidenzia l'efficiente implementazione PostgreSQL come novità importante e la descrive come adatta sia allo standalone sia alla scalabilità tramite RabbitMQ, OpenSearch e S3 ([Apache James 3.9.0](https://james.apache.org/james/update/2025/09/25/james-3.9.0.html)). Per le nuove installazioni questo è in genere il punto di partenza più comprensibile: prima consistenza relazionale e procedure di backup note, quindi servizi aggiuntivi solo per requisiti concretamente misurati.

## Modello di sicurezza

James fornisce TLS, autenticazione SMTP, controlli di protocollo e Mailet crittografici. Ciò non implica automaticamente un esercizio di produzione sicuro. La cifratura del trasporto protegge un hop; non sostituisce né la crittografia end-to-end né una verifica vincolante del destinatario. La [configurazione TLS](https://james.apache.org/server/config-ssl-tls.html) separa keystore, Cipher Suites attive, StartTLS e TLS implicito per listener. Un cambio di certificato deve quindi essere verificato separatamente per SMTP, IMAP, POP3 e HTTP.

Particolare attenzione merita **WebAdmin**. La REST API può modificare domini, utenti, mailbox, code, repository, quote e task di manutenzione. Secondo la [documentazione WebAdmin](https://james.apache.org/server/manage-webadmin.html) l'autenticazione JWT è disattivata per impostazione predefinita; senza protezioni aggiuntive l'API non deve quindi mai essere raggiungibile da una rete non controllata. Gli endpoint di salute e la documentazione API possono inoltre trovarsi deliberatamente al di fuori dell'autenticazione.

Un hardening minimo per la produzione comprende:

- vincolare WebAdmin a una rete di gestione, attivare JWT e limitare ulteriormente l'accesso tramite firewall o reverse proxy;
- impedire relay aperti mediante regole esplicite per relay, autenticazione e destinatari;
- gestire Submission e SMTP server-to-server su listener separati con policy differenti;
- rimuovere domini demo, utenti di esempio e password predefinite dalle immagini container prima del primo avvio esterno;
- gestire le chiavi private al di fuori del layer container e monitorarne le scadenze;
- trattare i Mailet personalizzati come codice applicativo: verificare le dipendenze, eseguire test e limitare i privilegi di runtime;
- progettare consapevolmente il controllo antispam e antimalware. James è una piattaforma; scanner esterni e servizi di reputazione vengono integrati tramite Mailets o passaggi di protocollo.

Per la ricerca guasti, il percorso del messaggio viene verificato nuovamente nello stesso ordine: listener, coda, pipeline Mailet, repository, mailbox e consegna in uscita.

## Esercizio e ricerca guasti

In un server di posta modulare, «il servizio è in esecuzione» non è una descrizione di stato sufficiente. I WebAdmin Health Checks distinguono `healthy`, `degraded` e `unhealthy`; in modalità rigorosa, già un componente degradato comporta HTTP 503. A seconda del profilo vengono verificati, tra gli altri, JPA o Cassandra, OpenSearch, RabbitMQ, il ciclo di vita Guice, Event Dead Letters e una consegna di test completa ([WebAdmin Health Checks](https://james.apache.org/server/manage-webadmin.html#HealthCheck)).

Per la diagnosi, un percorso a livelli è più efficiente di una ricerca globale nei log:

1. **Connessione:** Il client raggiunge il listener corretto e TLS riesce con il certificato e il nome host attesi?
2. **Transazione SMTP:** Quale codice di risposta è stato restituito per `MAIL FROM`, `RCPT TO` e `DATA`? Un `250` dopo `DATA` indica accettazione, non necessariamente consegna finale.
3. **Coda:** Cresce il numero di voci in attesa, aumenta la loro età o si ripete lo stesso errore remoto?
4. **Pipeline Mailet:** Quale Processor e quale coppia Matcher/Mailet ha elaborato il messaggio? L'ID mail funge da chiave di correlazione.
5. **Repository:** Il messaggio è in `error`, `address-error`, `relay-denied` o in un repository personalizzato? Correggere prima la causa, poi rielaborare.
6. **Mailbox ed eventi:** Il messaggio è presente nello store mailbox autorevole, ma manca nell'indice di ricerca o in JMAP? In tal caso listener, Dead Letters e reindicizzazione sono più rilevanti di SMTP.
7. **Remote Delivery:** Per la consegna in uscita verificare separatamente DNS, rotta, TLS, codice della controparte, piano di retry e generazione dei bounce.

Un controllo sintetico compatto può collegare il piano di amministrazione e quello dei dati:

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für den Health Check">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$headers = @{ Authorization = "Bearer $env:JAMES_ADMIN_JWT" }
Invoke-RestMethod `
  -Uri "https://james-admin.example.net/healthcheck?strict" `
  -Headers $headers</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">curl --fail --silent \
  -H "Authorization: Bearer $JAMES_ADMIN_JWT" \
  "https://james-admin.example.net/healthcheck?strict"</code></pre>
  </div>
</div>

In Windows, [`Invoke-RestMethod`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod) richiama l'endpoint REST; in Linux e Unix [`curl`](https://curl.se/docs/manpage.html) esegue lo stesso controllo HTTP. Entrambi i comandi testano qui esclusivamente il WebAdmin Health Check documentato e non sostituiscono una transazione SMTP o mailbox sintetica.

Inoltre dovrebbero essere configurati allarmi almeno per profondità ed età della coda, repository di errore, Event Dead Letters, ritardo di indicizzazione OpenSearch, latenze backend, classi di risposta SMTP, memoria JVM e validità dei certificati. Nella variante distribuita, un processo James verde con RabbitMQ o OpenSearch guasti rappresenta solo un successo parziale.

### Strumenti per la postazione dell'amministratore

James include un client da riga di comando per domini, utenti, mailbox, mapping, quote e reindicizzazione; nei container Guice è disponibile come `james-cli` ([James CLI](https://james.apache.org/server/manage-cli.html)). Per una diagnostica affidabile, alla postazione dell'amministratore devono affiancarsi alcuni strumenti indipendenti dal protocollo:

| Strumento | Impiego con James |
|---|---|
| [`swaks`](https://www.jetmore.org/john/code/swaks/) | transazione SMTP e Submission completa con AUTH, TLS, envelope e header impostati liberamente |
| [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) | verificare catena di certificati, SNI, cipher e StartTLS su SMTP, IMAP o POP3 |
| [`curl`](https://curl.se/docs/manpage.html) e [`jq`](https://jqlang.org/manual/) | interrogare automaticamente WebAdmin, Health Checks, task e metriche |
| [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) o [`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) | controllare MX, A/AAAA, PTR, SPF, DKIM e DMARC |
| [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) o [Wireshark](https://www.wireshark.org/docs/wsug_html_chunked/) | distinguere handshake, ritrasmissioni, interruzioni di connessione e dialoghi di protocollo |
| [Prometheus](https://prometheus.io/docs/introduction/overview/) e [Grafana](https://grafana.com/docs/grafana/latest/) | osservare metriche di coda e protocolli, percentili di latenza, tempi di esecuzione di Mailet/Matcher e stati backend |
| [JMX](https://docs.oracle.com/en/java/javase/21/management/java-management-extensions-jmx-user-guide.html), [VisualVM](https://visualvm.github.io/documentation.html) e [`jcmd`](https://docs.oracle.com/en/java/javase/21/docs/specs/man/jcmd.html) | analizzare heap, thread, garbage collection e metriche interne della JVM |

La [documentazione nativa delle metriche](https://james.apache.org/server/metrics.html) elenca tra l'altro connessioni SMTP, IMAP e LMTP attive, voci in coda, messaggi inviati e consegnati, tempi di risposta per protocollo e tempi di esecuzione di singoli Mailet e Matcher. Queste metriche sono più significative di un unico uptime del processo, perché rappresentano il percorso di un messaggio attraverso l'architettura.

## Storia tecnica

James non è nato come porting di un MTA Unix esistente. Le più antiche pagine di progetto conservate, del **1997/1998**, descrivono inizialmente un server Java pianificato e non ancora utilizzabile, basato su pacchetti condivisi del Java Apache Project. Erano previste un'interfaccia di protocollo comune, storage JDBC e un'interfaccia **MailServlet** ispirata ai Servlet; come lavoro tecnico preliminare fungeva l'infrastruttura dell'ambiente Apache JServ ([Archivio James-1.0](https://svn.apache.org/repos/asf/james/server/tags/james_1_0/docs/index.html?p=1400000)). La successiva Mailet API ha conservato l'idea di piccoli componenti di elaborazione distribuibili, senza entrare nella specifica Java Servlet.

| Periodo | Passo dello sviluppo tecnico |
|---|---|
| 1997–1998 | Progettazione nel Java Apache Project: server interamente Java, interfacce comuni per protocolli e risorse, idea MailServlet |
| Febbraio 2001 | Migrazione dal Java Apache Project al progetto Jakarta ([Jakarta News 2001](https://jakarta.apache.org/site/news/news-2001.html#20010311.1)) |
| James 1.x/2.x | server SMTP/POP3 stabile, per un periodo NNTP; motore Mailet, storage su file e RDBMS; container di componenti Avalon/Phoenix ([Archivio della documentazione](https://james.apache.org/server/archive/document_archive.html)) |
| primi anni 2000 | passaggio da sottoprogetto Jakarta a progetto Top-Level indipendente della Apache Software Foundation ([James 2.1.3 – pagina di progetto archiviata](https://svn.apache.org/repos/asf/james/server/tags/deprecated/build_2_2_0_RC1/www/index.html?p=1400000)) |
| 2010 | James 3.0 M1 con supporto IMAP completo, SMTP/LMTP, Mailet API rivista e storage Maildir, JPA e JCR ([Annuncio di rilascio](https://james.apache.org/james/update/2010/11/05/james-3.0-M1.html)) |
| James 3.x | sostituzione di Avalon/Phoenix con Spring e successivo orientamento strategico verso Guice; ampliamento di IMAP, JMAP, amministrazione REST e backend distribuiti |
| Settembre 2025 | James 3.9.0: passaggio da `javax` a `jakarta`, Java 21 e nuova implementazione PostgreSQL ([Annuncio di rilascio](https://james.apache.org/james/update/2025/09/25/james-3.9.0.html)) |

Il codice sorgente risiede nel repository ufficiale [apache/james-project](https://github.com/apache/james-project). La generazione 3.9 qui considerata è composta prevalentemente da Java; alcuni moduli utilizzano Scala. Il build viene eseguito come grande progetto Maven multimodulo. La lunga storia di sviluppo spiega perché nella documentazione e nelle installazioni siano visibili più generazioni affiancate: termini Phoenix e Spring nei testi meno recenti, Guice nella documentazione 3.x, JPA come percorso legacy e profili PostgreSQL o Cassandra per deployment distribuiti.

## Idoneità e limiti

James è particolarmente adatto quando l'e-mail è **parte di un'applicazione** anziché semplice infrastruttura: elaborazione basata su regole, Mailet personalizzati, protocolli aperti, JMAP, gestione controllabile dei dati o scalabilità orizzontale senza un nucleo server proprietario. Le API pubbliche consentono di evolvere separatamente trasporto, mailbox e logica aziendale.

James è meno adatto a organizzazioni che si aspettano un'appliance chiavi in mano con GUI completa, difesa antispam e antimalware preconfigurata, SLA del produttore e un unico oggetto di backup. La libertà modulare crea lavoro di integrazione. Il profilo distribuito, in particolare, richiede esperienza operativa con più sistemi di dati e una definizione chiara di origine, proiezione, ricostruzione e recovery point.

La domanda architetturale decisiva è quindi: **si vuole gestire l'e-mail come sistema di protocolli configurabile o come prodotto finito?** Nel primo caso James offre un toolkit aperto e insolitamente profondo. Nel secondo caso, un prodotto più preconfigurato è spesso più conveniente.

## Fonti

- [Apache Projects – James Committee](https://projects.apache.org/committee.html?james)
- [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0)
- [Microsoft Learn – Invoke-RestMethod](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod)
- [curl – Manpage](https://curl.se/docs/manpage.html)
- [SWAKS – Swiss Army Knife for SMTP](https://www.jetmore.org/john/code/swaks/)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [jq – Manual](https://jqlang.org/manual/)
- [BIND 9 – dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)
- [tcpdump – Manpage](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [Wireshark – User’s Guide](https://www.wireshark.org/docs/wsug_html_chunked/)
- [Prometheus – Overview](https://prometheus.io/docs/introduction/overview/)
- [Grafana – Documentation](https://grafana.com/docs/grafana/latest/)
- [Oracle – JMX User Guide](https://docs.oracle.com/en/java/javase/21/management/java-management-extensions-jmx-user-guide.html)
- [VisualVM – Documentation](https://visualvm.github.io/documentation.html)
- [Oracle – jcmd](https://docs.oracle.com/en/java/javase/21/docs/specs/man/jcmd.html)
- [Apache James – Panoramica del progetto](https://james.apache.org/) – autodescrizione, JVM, protocolli, moduli e obiettivi architetturali.
- [Apache James – Software Components](https://james.apache.org/documentation.html) – sottoprogetti server, Mailet, mailbox, Protocols e altri.
- [Apache James – Protocol Servers](https://james.apache.org/server/feature-protocols.html) – servizi di protocollo supportati.
- [RFC 6409](https://datatracker.ietf.org/doc/html/rfc6409)
- [RFC 8314](https://datatracker.ietf.org/doc/html/rfc8314)
- [IETF: SMTP](https://datatracker.ietf.org/doc/html/rfc5321), [LMTP](https://datatracker.ietf.org/doc/html/rfc2033), [Message Submission](https://datatracker.ietf.org/doc/html/rfc6409), [IMAP4rev2](https://datatracker.ietf.org/doc/html/rfc9051), [POP3](https://datatracker.ietf.org/doc/html/rfc1939), [ManageSieve](https://datatracker.ietf.org/doc/html/rfc5804) e [JMAP Mail](https://datatracker.ietf.org/doc/html/rfc8621) – standard normativi dei protocolli.
- [RFC 2033](https://datatracker.ietf.org/doc/html/rfc2033)
- [RFC 9051](https://datatracker.ietf.org/doc/html/rfc9051)
- [RFC 1939](https://datatracker.ietf.org/doc/html/rfc1939)
- [RFC 5804](https://datatracker.ietf.org/doc/html/rfc5804)
- [RFC 8621](https://datatracker.ietf.org/doc/html/rfc8621)
- [IANA Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) – porte registrate.
- [Mailbox API](https://james.apache.org/mailbox/mailbox-api.html)
- [Apache James – Managing Distributed James](https://james.apache.org/server/manage-guice-distributed-james.html) – Cassandra, S3, OpenSearch, RabbitMQ, event bus ed esercizio.
- [Apache James – Mailet Container](https://james.apache.org/server/feature-mailetcontainer.html) – Matchers, Mailets, Processors, Spooler e suddivisione dei destinatari.
- [Apache James – Mailet Container Configuration](https://james.apache.org/server/config-mailetcontainer.html) – configurazione e gestione degli errori della pipeline.
- [Apache James – Configuration](https://james.apache.org/server/config.html) – directory di configurazione, file e variabili d'ambiente.
- [Apache James – Persistence](https://james.apache.org/server/feature-persistence.html) – distinzione tra mailbox e Mail Repository.
- [Apache James – Downloads](https://james.apache.org/download.cgi) – profili server e download ufficiali.
- [Apache James Server 3.9.0](https://james.apache.org/james/update/2025/09/25/james-3.9.0.html) – Java 21, passaggio a Jakarta e implementazione PostgreSQL.
- [Apache James – SSL/TLS Configuration](https://james.apache.org/server/config-ssl-tls.html) – modalità TLS e configurazione dei listener.
- [Apache James – WebAdmin](https://james.apache.org/server/manage-webadmin.html) – amministrazione REST, avviso JWT e Health Checks.
- [Apache James – Command Line](https://james.apache.org/server/manage-cli.html) – CLI per domini, utenti, mailbox, mapping, quote e reindicizzazione.
- [Apache James – Metrics](https://james.apache.org/server/metrics.html) – Prometheus, JMX e metriche operative disponibili.
- [Archivio James-1.0 del Java Apache Project](https://svn.apache.org/repos/asf/james/server/tags/james_1_0/docs/index.html?p=1400000) – pianificazione iniziale dell'architettura e di MailServlet.
- [Jakarta Project News 2001](https://jakarta.apache.org/site/news/news-2001.html) – migrazione del progetto James a Jakarta.
- [Apache James Document Archive](https://james.apache.org/server/archive/document_archive.html) – documentazione delle versioni 1.x e 2.x.
- [James 2.1.3 – pagina di progetto archiviata](https://svn.apache.org/repos/asf/james/server/tags/deprecated/build_2_2_0_RC1/www/index.html?p=1400000)
- [Apache James 3.0 M1](https://james.apache.org/james/update/2010/11/05/james-3.0-M1.html) – IMAP, profili di storage e Mailet API della generazione 3.x.
- [Apache James – Repository GitHub](https://github.com/apache/james-project) – codice sorgente, build e struttura dei moduli.
