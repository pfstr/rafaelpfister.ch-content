---
title: "Cisco Secure Email: Gateway, AsyncOS e SMA"
blatt: "cisco"
description: "Cisco Secure Email per amministratori di messaggistica: pipeline di posta SEG/ESA, listener, HAT e RAT, Work Queue e Delivery, Mail Policies, AsyncOS, servizi SMA, limiti del cluster, stack tecnologico, monitoraggio, ripristino e diagnostica."
fakten:
  - label: Ruoli del prodotto
    wert: Secure Email Gateway (SEG/ESA) · Secure Email and Web Manager (SMA)
    href: https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/datasheet-c78-742868.html
  - label: Ruolo del sistema
    wert: Gateway di posta SMTP davanti o tra sistemi di posta
    href: https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html
  - label: Sistema operativo
    wert: Cisco AsyncOS
    href: https://www.cisco.com/c/en/us/support/security/email-security-appliance/series.html
  - label: Pipeline
    wert: Receipt → Work Queue → Delivery
    href: https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html
  - label: Accettazione
    wert: Listener · HAT · Sender Groups · RAT · LDAP
    href: https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html
  - label: Elaborazione
    wert: Message Filters · Mail Policies · Content Filters · Scan-Engines
    href: https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html
  - label: Servizi centrali
    wert: Tracking · Reporting · quarantene spam e policy su SMA
    href: https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0100.html
  - label: Cluster di configurazione
    wert: peer-to-peer; nessuna HA per code o traffico
    href: https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0101000.html
  - label: Fattori di forma
    wert: appliance virtuale · Public Cloud · Cisco Cloud Gateway
    href: https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/datasheet-c78-742868.html
  - label: Amministrazione
    wert: interfaccia web · CLI tramite SSH · REST API
    href: https://docs.ces.cisco.com/docs/api
  - label: Stati primari
    wert: configurazione · coda · quarantene · tracking/reporting · chiavi e certificati
    href: https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0100010.html
  - label: Origine
    wert: tecnologia IronPort; acquisita da Cisco nel 2007
    href: https://newsroom.cisco.com/c/r/newsroom/en/us/a/y2007/m01/cisco-announces-agreement-to-acquire-ironport.html
werbung:
  - tools
  - newsletter
ctaThemen:
  - cisco-esa-sma
translationSourceHash: c796a2c50226bbdcf5a0d7a7562913a5126b28cdbd05a5b72065afcf531bfd42
translationModel: gpt-5.6-terra
translatedAt: 2026-10-03T09:42:38.035Z
translationReview: required
---

# Cisco Secure Email: Gateway, AsyncOS e SMA

**Cisco Secure Email Gateway**, abbreviato SEG e storicamente **Email Security Appliance** o ESA, è un gateway di posta SMTP con stato [](/kb/smtp). Termina le sessioni SMTP in entrata, decide l'accettazione, elabora i messaggi in una Work Queue interna e apre una nuova sessione SMTP per la consegna. Il confine della responsabilità tecnica non coincide quindi con il corretto handshake TCP o TLS, bensì con la risposta SMTP positiva dopo il contenuto del messaggio: da quel momento il gateway deve consegnare oppure generare un errore conforme allo standard ([](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html), [](https://datatracker.ietf.org/doc/html/rfc5321)).

Il secondo ruolo classico è il **Cisco Secure Email and Web Manager**, SMA. Normalmente non opera come MTA regolare nel percorso produttivo della posta. Gestisce dati centralizzati di tracking e reporting nonché, a seconda del design, quarantene per spam, policy, virus e outbreak di più gateway. Un guasto dell'SMA può pertanto lasciare inalterata la consegna sui nodi SEG, interrompendo al contempo la ricerca, la quarantena dell'utente finale o la gestione dei messaggi trattenuti ([](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0100.html), [](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0110.html)).

Entrambi i ruoli operano su **AsyncOS**, una piattaforma software gestita da Cisco come unità appliance. Gli amministratori non gestiscono i pacchetti sottostanti come su un server Linux generico; l'interfaccia tecnica affidabile è costituita da configurazione AsyncOS, CLI, interfaccia web, REST API, sottoscrizioni ai log, MIB, canali di aggiornamento e integrazioni documentate. Le panoramiche Cisco sull'open source attestano numerosi componenti integrati, ma non un elenco pubblicamente manutenibile della pipeline di posta proprietaria. Le singole librerie non devono quindi essere equiparate all'architettura complessiva ([](https://www.cisco.com/c/dam/en_us/about/doing_business/open_source/docs/CiscoEmailSecurityAppliance-AsyncOS1253-1688323737.pdf), [](https://docs.ces.cisco.com/docs/api)).

La spiegazione segue un messaggio attraverso Cisco Secure Email: dal listener, tramite HAT, RAT e Work Queue, fino alla consegna. Seguono SMA, funzionamento in cluster, dipendenze, diagnostica e ripristino.

## Ruoli del prodotto e limiti di fiducia

Un tipico design on-premises colloca almeno due nodi SEG nella DMZ e un SMA in una rete di gestione interna. Il DNS MX o un servizio a monte distribuiscono le connessioni in entrata ai gateway; in uscita, i connettori smarthost del sistema di posta determinano il percorso gateway. Più nodi SEG sono ad alta disponibilità solo quando DNS, load balancer o l'MTA mittente possono utilizzare destinazioni alternative. Il solo cluster di configurazione AsyncOS non assume questo instradamento del traffico ([](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0101000.html), [](https://datatracker.ietf.org/doc/html/rfc5321)).

Cisco documenta appliance virtuali, deployment in Public Cloud e un Secure Email Cloud Gateway gestito. Queste varianti condividono termini di prodotto, ma spostano le responsabilità: per l'appliance virtuale, il cliente è responsabile di hypervisor, rete, capacità e ripristino; per il Cloud Gateway, Cisco fornisce l'infrastruttura del gateway. **Secure Email Threat Defense** è invece una piattaforma cloud-native di analisi e protezione, integrabile tramite gateway, journaling o API Microsoft. Non è né un sinonimo della Work Queue locale né un termine sostitutivo per l'SMA ([](https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/datasheet-c78-742868.html), [](https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/secure-email-threat-defense-ds.html)).

| Ruolo | Nel percorso SMTP | Stato persistente | Effetto del guasto |
|---|---:|---|---|
| Secure Email Gateway | sì | Coda, quarantene locali, configurazione, certificati, log | Accettazione o consegna compromessa su questo nodo |
| Secure Email and Web Manager | normalmente no | Tracking, reporting, quarantene centralizzate, Safe-/Blocklist, configurazione propria | Visibilità e servizi di quarantena centralizzati compromessi |
| Email Threat Defense | dipendente dall'integrazione | Telemetria, indagine e policy lato cloud | Analisi o remediation aggiuntive compromesse |
| Sistema di posta | prima o dopo il gateway | Caselle postali, code di trasporto, connettori | Accesso degli utenti o consegna end-to-end compromessi |

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-cisco.svg?v=20260813" title="Interaktive Infografik: Cisco Secure Email mit SEG-Mailpipeline, Listener, HAT und RAT, Work Queue, Delivery, SMA-Diensten, Konfigurationscluster und Admin-Kontrollpunkten" loading="lazy">
  <a href="/images/kb-interaktiv-cisco.svg?v=20260813">Apri direttamente il grafico interattivo</a>.
</iframe>

## Receipt: listener, HAT e RAT

Un **listener** associa SMTP a un'interfaccia IP e costituisce il primo limite di policy. I listener pubblici accettano tipicamente traffico Internet per domini locali; i listener privati ricevono messaggi in uscita da reti controllate. Questi ruoli sono configurazioni, non proprietà di fiducia intrinseche della porta. Un listener privato con autorizzazione relay troppo ampia è un open relay, anche se denominato interno.

La **Host Access Table**, HAT, assegna gli host che si connettono a Sender Groups. Le relative Mail Flow Policies stabiliscono, tra l'altro, se una connessione viene accettata, rifiutata, limitata o elaborata senza singole scansioni. La **Recipient Access Table**, RAT, definisce i domini destinatari locali per i messaggi in entrata. Facoltativamente, [](/kb/ldap) verifica destinatari specifici durante la sessione SMTP o successivamente nella Work Queue; in alternativa, SMTP Call-Ahead può interrogare il server a valle. Cisco separa così quattro identità che non devono essere confuse in caso di malfunzionamento: IP di origine, mittente envelope, destinatario envelope e identità delle intestazioni ([](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html)).

Una documentazione pulita dei listener comprende per ciascuna direzione almeno IP e porta di binding, reti sorgenti consentite, nomi EHLO attesi, modalità TLS, requisiti per il certificato client, ordine HAT, domini RAT, verifica dei destinatari, dimensione massima del messaggio, rate limit e profilo di bounce. L'ordine è particolarmente critico: un rifiuto HAT anticipato non genera un record Message-ID come un messaggio accettato successivamente; l'helpdesk non può quindi trovarlo con la stessa ricerca.

Dopo l'accettazione SMTP inizia la vera elaborazione di contenuto e policy. Il suo ordine è importante, poiché un risultato anticipato può influenzare verifiche successive, gruppi di destinatari o percorsi di consegna.

## Work Queue: ordine, splintering e policy

Dopo l'accettazione, il messaggio entra nella **Work Queue**. Cisco documenta routing e masquerading, Message Filters, Safe-/Blocklist, anti-spam, anti-virus, Graymail, reputazione e analisi dei file, Content Filters, Outbreak Filters e quarantene. L'ordine fa parte del modello di sicurezza. Una modifica di policy in genere non agisce retroattivamente sui messaggi già inseriti; l'attivazione successiva di uno scanner non ripara quindi automaticamente un bypass precedente ([](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html)).

I **Message Filters** operano prima della Mail Policy basata sui destinatari e possono modificare, archiviare, mettere in quarantena, respingere o scartare messaggi in base a envelope, intestazioni, contenuto, allegati o dati di connessione. Successivamente AsyncOS può effettuare lo **splintering** di un messaggio con più destinatari: vengono creati Message ID separati per policy destinatario diverse e quindi stati finali differenti. Un singolo valore inject o ICID originale può pertanto ramificarsi in più MID e risultati di consegna. Il tracking deve mostrare questo albero, non limitarsi alla ricerca per oggetto.

Le **Mail Policies** controllano filtri di scansione e contenuto basati su destinatario o mittente. Secondo Cisco, DLP è limitato all'elaborazione in uscita. Licenze, stato di aggiornamento dei motori e connettività cloud determinano inoltre quali controlli vengono effettivamente eseguiti. Per ogni policy, un test affidabile richiede un caso positivo innocuo, un caso negativo mirato e lo stato finale atteso: consegna, modifica, quarantena, drop o bounce.

## Delivery: SMTP Routes, Destination Controls e coda

Nella fase di Delivery, AsyncOS seleziona route, interfaccia sorgente e destinazione. Le **SMTP Routes** sostituiscono la normale risoluzione MX per i domini configurati; i **Destination Controls** limitano le connessioni parallele e i destinatari per destinazione. I Virtual Gateways possono fornire indirizzi IP sorgente, hostname e code di consegna differenti. Queste impostazioni influenzano reputazione, SPF, allowlist della controparte e il punto in cui un messaggio resta in attesa ([](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html), [](https://datatracker.ietf.org/doc/html/rfc7208)).

TLS in uscita è hop-by-hop. AsyncOS può usare [](/kb/tls) con le controparti; il successo protegge questo tratto di trasporto, ma non dice nulla sui hop precedenti o successivi. Per policy obbligatorie devono essere documentati pattern di destinazione, verifica dei certificati, relazione con il nome e comportamento in caso di errore. TLS opportunistico può ricadere in testo in chiaro in caso di errore di handshake; una policy obbligatoria deve invece accodare o fallire ([](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118844-technote-esa-00.html), [](https://datatracker.ietf.org/doc/html/rfc3207)).

L'età della coda è più importante della sola lunghezza della coda. Un volume elevato può essere sano con throughput elevato; pochi messaggi molto vecchi indicano un blocco persistente di destinazione, DNS, TLS o policy. Per ogni route, l'immagine dell'incidente deve includere il messaggio più vecchio, il motivo del retry, il prossimo tentativo, la risposta della destinazione e la controparte responsabile.

Non appena l'ESA ha inoltrato o messo in quarantena un messaggio, parte della visibilità amministrativa si sposta sull'SMA. Tuttavia, non sostituisce i dati locali di coda e sistema dell'ESA.

## SMA: tracking, reporting e quarantene

L'SMA raccoglie dati di tracking e reporting da più nodi SEG. Il Message Tracking può mostrare stati finali quali `Delivered`, `Dropped`, `Bounced`, `Quarantined`, `Queued`, `Processing` e `Splintered`. Tuttavia è un indice derivato: se mancano dati di esportazione, un servizio è in ritardo o il messaggio è fuori dal periodo di conservazione, un risultato vuoto non dimostra che il messaggio non sia mai stato elaborato. La prova primaria restano i Mail Logs pertinenti e la catena MID sul SEG ([](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0100.html)).

La quarantena spam e le quarantene per policy/virus/outbreak sono servizi separati con utenti, percorsi di rilascio e conservazione differenti. Le quarantene centralizzate memorizzano i messaggi sull'SMA dietro il firewall e possono essere incluse nel relativo backup standard. Al 75, 85 e 95 per cento di occupazione, AsyncOS genera soglie di allarme documentate. Se un servizio di quarantena centralizzato diventa irraggiungibile, l'esercizio richiede una decisione testata in anticipo: accodare temporaneamente, riconfigurare l'elaborazione locale o disabilitare in modo controllato la policy associata ([](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0110.html), [](https://www.cisco.com/c/en/us/td/docs/security/esa/esa13-7-0/user_guide/b_ESA_Admin_Guide_13-7/b_ESA_Admin_Guide_12_1_chapter_0101011.html)).

## Il cluster di configurazione non è Mail HA

AsyncOS può collegare più gateway in un **cluster di configurazione** peer-to-peer. Le impostazioni possono essere mantenute a livello di cluster, gruppo o macchina; non esiste un nodo cluster primario. I membri devono usare una versione AsyncOS compatibile e comunicano tramite SSH o Cluster Communication Service. Il cluster replica la configurazione, non le sessioni SMTP attive, il contenuto delle code, le quarantene locali o lo stato di avanzamento della consegna ([](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0101000.html)).

Esistono quindi tre meccanismi separati:

- **Distribuzione del traffico:** più destinazioni MX, load balancer o failover smarthost;
- **Coerenza della configurazione:** cluster AsyncOS con override chiari di cluster, gruppo e macchina;
- **Disponibilità dei dati:** stato della coda per SEG nonché dati di tracking e quarantena sull'SMA.

La perdita di un nodo dopo l'accettazione SMTP positiva può interessare messaggi presenti solo nella relativa coda locale. Il mittente non deve semplicemente inviarli di nuovo finché lo stato di consegna originale non è chiaro; altrimenti si generano duplicati. Un test di recovery deve quindi non solo caricare la configurazione, ma seguire messaggi di test accettati durante un guasto controllato del nodo.

## Stack tecnologico e superfici di amministrazione

AsyncOS è una piattaforma appliance chiusa. Cisco pubblica avvisi open source per i componenti forniti, ma non un piano completo dei sorgenti o dei linguaggi dei servizi proprietari. Affermazioni come «scritto in Python» o «basato su FreeBSD» non sono informazioni operative affidabili senza una prova del produttore specifica per versione. Per gli amministratori è più rilevante il seguente stack verificabile ([](https://www.cisco.com/c/dam/en_us/about/doing_business/open_source/docs/CiscoEmailSecurityAppliance-AsyncOS1253-1688323737.pdf)):

| Livello | Tecnologia verificabile | Rilevanza operativa |
|---|---|---|
| Trasporto posta | Listener SMTP, Receipt, Work Queue, Delivery Queue | Limite di accettazione, ordine delle policy, retry e bounce |
| Policy e analisi | HAT/RAT, Message Filters, Mail Policies, Content Filters, Scan-Engines | Ordine, licenze, aggiornamenti dei motori, splintering |
| Dati e ricerca | Code/quarantene locali; tracking, reporting e quarantene centralizzate SMA | Capacità, conservazione, backup e protezione dei dati |
| Gestione | GUI HTTPS, CLI interattiva tramite SSH, configurazione XML | Change, commit, export, restore e audit |
| Automazione | RESTful AsyncOS API con Swagger | Reporting, tracking e accesso alla quarantena; nessuna configurazione completa non verificata |
| Telemetria | Mail Logs, ulteriori Log Subscriptions, Syslog, Alerts, SNMP/MIB, API | Correlazione tramite ICID/MID/DCID e stato delle risorse |
| Piattaforma | Appliance hardware, virtuale e cloud | Responsabilità per compute, storage, rete e lifecycle |

Le modifiche CLI seguono un modello transazionale: i comandi modificano inizialmente una configurazione in esecuzione, `commit` la attiva, `clearchanges` la scarta. Un runbook deve indicare il dialogo completo e la modalità di configurazione; semplici frammenti copy-and-paste sono pericolosi a causa delle differenze tra release e cluster. La REST API offre accesso autenticato in modo sicuro a report, contatori, dati di tracking e quarantena; la sua interfaccia Swagger locale documenta l'effettivo ambito API installato ([](https://docs.ces.cisco.com/docs/api), [](https://www.cisco.com/c/en/us/support/security/email-security-appliance/series.html)).

## Dipendenze di rete, identità e tempo

Dopo aver definito la pipeline della posta, è possibile verificare le sue connessioni esterne. Ogni riga mostra quale sistema avvia una connessione, a cosa serve e come si manifesta un errore.

| Connessione | Porta tipica | Iniziatore | Scopo e sintomo di errore |
|---|---:|---|---|
| SMTP | TCP 25 | MTA esterno, sistema di posta interno o SEG | Accettazione e inoltro; timeout, 4xx/5xx, crescita della coda |
| HTTPS | TCP 443 o configurato | Admin, utente finale o client API | GUI, API, quarantena; verificare separatamente certificato, SSO e ruoli |
| SSH | TCP 22 o configurato | Admin o membro SEG | CLI e comunicazione cluster facoltativa |
| CCS | TCP 2222 per impostazione predefinita, configurabile | Membro SEG | Cluster di configurazione; nessun flusso di posta |
| DNS | UDP/TCP 53 | SEG/SMA | MX, A/AAAA, PTR, reputazione e aggiornamenti |
| LDAP/LDAPS | TCP 389/636 | SEG/SMA | Destinatari, routing, gruppi e autenticazione admin |
| Syslog | UDP/TCP 514 o TLS 6514 secondo il design | SEG/SMA | Trasporto log esterno; definire il modello di perdita e backpressure |
| SNMP | UDP 161/162 | Monitoraggio o appliance | Query di stato e trap; preferire SNMPv3 |

I soli numeri di porta non dimostrano una funzione attiva. Le assegnazioni provengono da [](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml); Cisco documenta CCS e la sua configurabilità nel capitolo sui cluster. I firewall dovrebbero contenere sorgente, destinazione, direzione, protocollo, requisiti TLS o di autenticazione e scopo aziendale.

[](/kb/ldap) può alimentare l'accettazione dei destinatari, il routing, l'appartenenza ai gruppi e l'autenticazione admin. Queste query hanno schemi, timeout e conseguenze degli errori differenti. Se Recipient Acceptance non funziona, a seconda della configurazione il sistema può effettuare un bounce ritardato o scartare; un errore di autenticazione della GUI non prova quindi un difetto della verifica SMTP dei destinatari. Account di servizio, Base DN, filtri, comportamento dei referral, catena di certificati e ordine di failover devono essere documentati per ciascuna query ([](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html), [](https://datatracker.ietf.org/doc/html/rfc4511)).

DNS e orario corretto sono dipendenze di sistema. La risoluzione MX e host controlla la consegna e la raggiungibilità del cluster; PTR e reputazione influenzano la classificazione. NTP mantiene correlabili gli orari dei log, Received e tracking. Cisco richiede hostname risolvibili o indirizzi IP usati in modo coerente per i cluster e descrive l'ora di sistema e NTP come parte della configurazione di base ([](https://www.cisco.com/c/en/us/td/docs/security/esa/esa13-7-0/user_guide/b_ESA_Admin_Guide_13-7/b_ESA_Admin_Guide_12_1_chapter_010.html)).

In caso di guasti, si segue lo stesso percorso a ritroso: stato di delivery, decisione della Work Queue, policy di receipt, listener e dipendenze di rete.

## Monitoraggio e triage degli incidenti

La domanda centrale è: **il gateway ha accettato il messaggio, lo ha elaborato e a quale hop lo ha consegnato?** A tal fine, gli ID di connessione, messaggio e delivery dei Mail Logs vengono concatenati. Il Message Tracking sull'SMA accelera la ricerca, ma non sostituisce i log grezzi. Segnali tecnici utili sono:

- tasso di accettazione, risposte 4xx/5xx e connessioni rifiutate per listener e Sender Group;
- Work Queue e Delivery Queue, età del messaggio più vecchio e risposte ricorrenti delle destinazioni;
- durata dell'elaborazione e dello splintering, errori dei motori di scansione ed età degli aggiornamenti;
- occupazione delle quarantene locali e centralizzate, eventi di rilascio e cancellazione;
- valore Resource Conservation, CPU, memoria, occupazione disco e alert critici;
- raggiungibilità e latenza di DNS, LDAP, SMA, servizi di aggiornamento e cloud;
- coerenza del cluster e override della macchina non intenzionali;
- scadenza e utilizzo di ogni certificato TLS nonché modifiche al truststore.

In **Resource Conservation Mode**, AsyncOS riduce gradualmente l'accettazione affinché la consegna possa smaltire l'arretrato; in caso di estrema carenza di risorse non vengono accettati nuovi messaggi. Il sintomo è quindi spesso una riduzione del throughput in ingresso, mentre la causa effettiva è una route di destinazione lenta o una risorsa esaurita. Cisco fornisce stato e alert in GUI/CLI; SNMPv3 e `ASYNCOS-MAIL-MIB` consentono il monitoraggio esterno ([](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/117834-qanda-esa-00.html), [](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/117831-qanda-esa-00.html)).

## Backup, recovery e upgrade

Un file di configurazione XML esportato è necessario, ma non costituisce un backup completo del sistema. Cisco documenta `saveconfig`, `mailconfig` e `loadconfig`; le passphrase mascherate non possono essere ricaricate. Certificati e chiavi, stato del cluster, Feature Keys, code locali, quarantene locali, dati SMA e dipendenze esterne richiedono prove proprie ([](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0100010.html), [](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118403-technote-esa-00.html)).

| Oggetto di recovery | Backup o ricostruzione | Test di accettazione |
|---|---|---|
| Configurazione SEG | Export non mascherato, archiviato in modo protetto, con passphrase documentate | Caricare su istanza sostitutiva, diff e test listener/policy |
| Certificati e chiavi private | Backup crittografato delle chiavi, catena CA e matrice dei ruoli | Handshake HTTPS e SMTP-TLS con verifica del nome |
| Coda locale | Normalmente non ricostruibile da un backup di configurazione | Guasto del nodo con mail di test accettata e controllo dei duplicati |
| Dati SMA | Backup SMA per tracking, reporting, quarantene e liste | Verificare ricerca, rilascio di una mail di test e conservazione |
| Cluster | Export per livello più override della macchina documentati | Ricollegare il membro e controllare la coerenza |
| Servizi esterni | Configurazione DNS, LDAP, Syslog, NTP, aggiornamenti e cloud | Verifica sintetica end-to-end |

Gli upgrade sono migrazioni dell'appliance. Prima vanno verificati percorso di destinazione, stati intermedi compatibili, requisiti dell'hypervisor o del cloud, modifiche delle funzionalità, ordine del cluster, spazio libero, downtime e limite di rollback. Le categorie di release Cisco GD e MD non sono una raccomandazione automatica per ogni ambiente; fanno fede i Security Advisories, la matrice di supporto e l'ambito delle policy proprie testato. La pagina di supporto e le spiegazioni sul lifecycle devono far parte della procedura di patch, non essere riportate come numero di versione statico nell'articolo ([](https://www.cisco.com/c/en/us/support/security/email-security-appliance/products-release-notes-list.html), [](https://www.cisco.com/c/dam/en/us/td/docs/security/esa/lifecycle_support_statement/Secure_Email_Gateway_Software_Lifecycle_Support_Statement.pdf)).

## Strumenti diagnostici

La ricerca dei guasti segue il percorso del messaggio dall'esterno verso l'interno. Prima vengono verificati nomi e raggiungibilità, poi accettazione SMTP, eventi della pipeline, coda e, se necessario, la valutazione SMA.

### Risoluzione DNS, MX e destinazione

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Cisco-Secure-Email-DNS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName -Type MX example.ch
Resolve-DnsName seg1.example.ch -Type A,AAAA
Resolve-DnsName 192.0.2.25 -Type PTR
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig +short MX example.ch
dig +short A seg1.example.ch
dig +short AAAA seg1.example.ch
dig +short -x 192.0.2.25
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) e [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) mostrano la risoluzione MX, forward e reverse. La query va ripetuta dalla prospettiva dei resolver interni ed esterni; le SMTP Routes di AsyncOS possono sostituire il risultato MX visibile.

### TCP e SMTP-TLS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Cisco-Secure-Email-TLS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Test-NetConnection seg1.example.ch -Port 25 -InformationLevel Detailed
curl.exe --verbose --ssl-reqd smtp://seg1.example.ch:25
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nc -vz seg1.example.ch 25
openssl s_client -starttls smtp -connect seg1.example.ch:25 \
  -servername seg1.example.ch -showcerts
```

  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) e [`nc`](https://man.openbsd.org/nc) dimostrano solo il percorso TCP. [`curl`](https://curl.se/docs/manpage.html) e [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) richiedono STARTTLS e mostrano handshake e catena di certificati; solo la verifica prevista del nome e della fiducia dimostra la policy TLS configurata.

### Messaggio SMTP di test autorizzato

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Cisco-Secure-Email-SMTP-Test">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
curl.exe --verbose --ssl-reqd --url smtp://seg1.example.ch:25 `
  --mail-from test-sender@example.ch `
  --mail-rcpt test-recipient@example.net `
  --upload-file .\seg-test.eml
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
swaks --server seg1.example.ch --port 25 --tls \
  --from test-sender@example.ch --to test-recipient@example.net \
  --data seg-test.eml
```

  </div>
</div>

[`curl`](https://curl.se/docs/manpage.html) e [`swaks`](https://jetmore.org/john/code/swaks/) inviano un messaggio di test controllato. Mittente, destinatario e destinazione devono essere autorizzati. Occorre registrare risposta SMTP finale, ICID/MID, MID di splintering, policy, stato di quarantena o delivery e arrivo effettivo.

### Interfaccia web e REST API

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Cisco-AsyncOS-API-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Invoke-WebRequest -Method Head -Uri https://sma.example.ch/
Invoke-WebRequest -Method Head -Uri https://seg1.example.ch/swagger
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
curl --head --verbose https://sma.example.ch/
curl --head --verbose https://seg1.example.ch/swagger
```

  </div>
</div>

[`Invoke-WebRequest`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest) e [`curl`](https://curl.se/docs/manpage.html) verificano raggiungibilità HTTP e TLS. Un codice di stato non dimostra né accesso né autorizzazione di ruolo, importazione del tracking o funzionamento della quarantena. La pagina Swagger descrive solo l'API dell'istanza contattata; i test API produttivi usano un account minimo in sola lettura e non salvano token nella cronologia della shell.

### Percorso pacchetti presso un punto di misurazione autorizzato

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Cisco-Secure-Email-Paketerfassung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
pktmon filter remove
pktmon filter add SEG-SMTP -p 25
pktmon start --capture --pkt-size 0 --file-name seg.etl
pktmon stop
pktmon pcapng seg.etl -o seg.pcapng
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
tcpdump -ni any -s 0 -w seg.pcap 'tcp port 25 or tcp port 443'
```

  </div>
</div>

[`pktmon`](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon) e [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) vedono solo il traffico nel punto di misurazione scelto. Un client admin non osserva automaticamente il percorso tra load balancer, SEG, SMA e MTA di destinazione. I dati dei pacchetti possono contenere contenuti SMTP prima di STARTTLS e metadati personali e devono essere protetti di conseguenza.

## Storia tecnica

IronPort Systems sviluppò gateway di messaggistica specializzati e la linea di prodotti AsyncOS. Cisco annunciò l'acquisizione dell'azienda nel gennaio 2007 e assegnò la tecnologia di sicurezza email e web al proprio portafoglio di sicurezza ([](https://newsroom.cisco.com/c/r/newsroom/en/us/a/y2007/m01/cisco-announces-agreement-to-acquire-ironport.html)). In seguito, i nomi dei prodotti passarono da Cisco IronPort Email Security Appliance a Cisco Email Security Appliance fino a **Cisco Secure Email Gateway**; termini storici quali ESA, C-Series e M-Series restano visibili in runbook, messaggi di log, licenze e percorsi della documentazione.

L'idea architetturale è rimasta riconoscibile attraverso queste ridenominazioni: un gateway specializzato con la pipeline a tre stadi Receipt, Work Queue e Delivery e un sistema di gestione separato per dati aggregati e quarantene. In seguito si sono aggiunte appliance virtuali e Public Cloud, Cloud Gateway, REST API e servizi di analisi basati sul cloud. Email Threat Defense amplia il portafoglio con modelli API, journaling e gateway; non modifica retroattivamente i limiti di stato di un'installazione ESA/SMA esistente ([](https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/datasheet-c78-742868.html), [](https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/secure-email-threat-defense-ds.html)).

Il nome di un'appliance installata non è quindi sufficiente come informazione sul lifecycle. Modello hardware, piattaforma virtuale, ramo AsyncOS, licenze attivate, aggiornamenti di motori e regole nonché servizi cloud dipendenti hanno cicli di vita propri. Le pagine Cisco di supporto, release ed end-of-life sono fonti operative dinamiche; un articolo statico dovrebbe collegarle, ma non fissare un presunto stato di versione sempre aggiornato ([](https://www.cisco.com/c/en/us/products/security/email-security-appliance/eos-eol-notice-listing.html), [](https://www.cisco.com/c/en/us/support/security/email-security-appliance/series.html)).

## Fonti

- [](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html)
- [](https://datatracker.ietf.org/doc/html/rfc5321)
- [](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0100.html)
- [](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0110.html)
- [](https://www.cisco.com/c/dam/en_us/about/doing_business/open_source/docs/CiscoEmailSecurityAppliance-AsyncOS1253-1688323737.pdf)
- [](https://docs.ces.cisco.com/docs/api)
- [](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0101000.html)
- [](https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/datasheet-c78-742868.html)
- [](https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/secure-email-threat-defense-ds.html)
- [](https://datatracker.ietf.org/doc/html/rfc7208)
- [](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118844-technote-esa-00.html)
- [](https://datatracker.ietf.org/doc/html/rfc3207)
- [](https://www.cisco.com/c/en/us/td/docs/security/esa/esa13-7-0/user_guide/b_ESA_Admin_Guide_13-7/b_ESA_Admin_Guide_12_1_chapter_0101011.html)
- [](https://www.cisco.com/c/en/us/support/security/email-security-appliance/series.html)
- [](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml)
- [](https://datatracker.ietf.org/doc/html/rfc4511)
- [](https://www.cisco.com/c/en/us/td/docs/security/esa/esa13-7-0/user_guide/b_ESA_Admin_Guide_13-7/b_ESA_Admin_Guide_12_1_chapter_010.html)
- [](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/117834-qanda-esa-00.html)
- [](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/117831-qanda-esa-00.html)
- [](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0100010.html)
- [](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118403-technote-esa-00.html)
- [](https://www.cisco.com/c/en/us/support/security/email-security-appliance/products-release-notes-list.html)
- [](https://www.cisco.com/c/dam/en/us/td/docs/security/esa/lifecycle_support_statement/Secure_Email_Gateway_Software_Lifecycle_Support_Statement.pdf)
- [](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [](https://bind9.readthedocs.io/en/latest/manpages.html)
- [](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [](https://man.openbsd.org/nc)
- [](https://curl.se/docs/manpage.html)
- [](https://docs.openssl.org/master/man1/openssl-s_client/)
- [](https://jetmore.org/john/code/swaks/)
- [](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest)
- [](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon)
- [](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [](https://newsroom.cisco.com/c/r/newsroom/en/us/a/y2007/m01/cisco-announces-agreement-to-acquire-ironport.html)
- [](https://www.cisco.com/c/en/us/products/security/email-security-appliance/eos-eol-notice-listing.html)
