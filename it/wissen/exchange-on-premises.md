---
title: "Exchange On-Premises: architettura e gestione del server"
blatt: "exchange-on-premises"
description: "Exchange nel proprio data center: ruoli Mailbox ed Edge, pipeline di trasporto, Active Directory, database ESE, DAG, accesso client, sicurezza, monitoraggio, backup e ripristino."
fakten:
  - label: Ruolo del prodotto
    wert: Piattaforma di messaggistica e groupware gestita in proprio
    href: https://learn.microsoft.com/en-us/exchange/exchange-server
  - label: Ruoli del server
    wert: Mailbox e Edge Transport opzionale
    href: https://learn.microsoft.com/en-us/exchange/architecture/architecture
  - label: Sistema operativo
    wert: Windows Server secondo i requisiti di sistema di Exchange
    href: https://learn.microsoft.com/en-us/exchange/plan-and-deploy/system-requirements
  - label: Directory
    wert: Active Directory Domain Services
    href: https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory
  - label: Archiviazione delle cassette postali
    wert: Database ESE, log delle transazioni e checkpoint
    href: https://learn.microsoft.com/en-us/exchange/client-developer/backup-restore/transaction-logs-and-checkpoint-files-for-backup-and-restore-in-exchange
  - label: Alta disponibilità
    wert: Database Availability Group e copie del database
    href: https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups
  - label: Trasporto
    wert: Frontend Transport, Transport Service e Mailbox Transport
    href: https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow
  - label: Resilienza del trasporto
    wert: Shadow Redundancy e Safety Net
    href: https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/transport-high-availability
  - label: Accesso client
    wert: HTTPS, MAPI/HTTP, Outlook sul web, EWS e ActiveSync
    href: https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access
  - label: Amministrazione
    wert: Exchange Admin Center e Exchange Management Shell
    href: https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/admin-interface
  - label: Monitoraggio
    wert: Managed Availability, Health Sets, registri eventi e indicatori di prestazione
    href: https://learn.microsoft.com/en-us/exchange/high-availability/managed-availability/managed-availability
  - label: Ripristino
    wert: Server Recovery, ripristino del database e Recovery Database
    href: https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery
werbung:
  - tools
  - newsletter
ctaThemen:
  - exchange-onprem-hybrid
  - smtp-mailflow
translationSourceHash: 45315e2f767bcdfb1b554eea6df80335a5ca86bbc102a345b83dd0e54cb0693b
translationModel: gpt-5.6-terra
translatedAt: 2026-11-06T03:33:59.297Z
translationReview: automatic
---

# Exchange On-Premises: architettura e gestione del server

**Exchange On-Premises** significa che l'organizzazione gestisce Exchange nella propria infrastruttura. Controlla host Windows, Active Directory, certificati, servizi di trasporto, code, database delle cassette postali e ripristino. Microsoft fornisce il codice del prodotto, la documentazione e gli aggiornamenti; la disponibilità e la manutenzione sicura restano compito del gestore ([Documentazione di Exchange Server](https://learn.microsoft.com/en-us/exchange/exchange-server), [Architettura di Exchange Server](https://learn.microsoft.com/en-us/exchange/architecture/architecture)).

La differenza pratica rispetto a Exchange Online emerge subito in caso di guasto. Un amministratore On-Prem può analizzare una coda di trasporto su un server specifico, verificare lo stato di una copia del database e passare in modo controllato a un'altra copia. Tuttavia, deve anche comprendere come interagiscono SMTP, Active Directory, ESE, Windows Failover Clustering, IIS e i servizi Exchange.

## Il server Mailbox è il componente centrale

I moderni server Exchange utilizzano il **server Mailbox** come componente comune. Include servizi Client Access che accettano e inoltrano connessioni, servizi di trasporto per il flusso di messaggi e l'Information Store con i database delle cassette postali. Un'installazione può iniziare in piccolo; più server e copie del database estendono lo stesso modello di base per l'alta disponibilità ([Architettura di Exchange Server](https://learn.microsoft.com/en-us/exchange/architecture/architecture)).

Questa unificazione non significa che tutte le funzioni abbiano lo stesso stato. Un frontend HTTPS può essere raggiungibile anche se il database interessato non è montato. SMTP può accettare connessioni mentre un messaggio resta successivamente in una coda. La diagnosi segue quindi il percorso effettivo e non soltanto lo stato generale del server.

Il **ruolo Edge Transport** opzionale è in genere collocato nella rete perimetrale ed elabora esclusivamente traffico SMTP. EdgeSync trasferisce informazioni selezionate su destinatari e configurazione a un'istanza AD LDS locale. Edge non contiene database delle cassette postali e non sostituisce i server Mailbox interni ([Server Edge Transport](https://learn.microsoft.com/en-us/exchange/architecture/edge-transport-servers/edge-transport-servers)).

## Stack tecnologico e dipendenze

Dal componente server deriva lo stack tecnologico. Exchange viene eseguito su versioni supportate di Windows Server e utilizza Active Directory per la configurazione di organizzazione, server e destinatari. IIS fornisce endpoint HTTP. PowerShell costituisce l'interfaccia di amministrazione. ESE archivia dati delle cassette postali e delle code in database separati ([Requisiti di sistema di Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/system-requirements), [Active Directory in Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)).

| Tecnologia | Compito nella gestione di Exchange | Domanda amministrativa importante |
|---|---|---|
| Windows Server | Processi, servizi, rete, archivio certificati e registri eventi | L'host è integro e correttamente aggiornato? |
| Active Directory | Organizzazione Exchange, server, destinatari, RBAC e informazioni di routing | La modifica corretta è visibile nei controller di dominio utilizzati? |
| IIS e HTTPS | Outlook sul web, EAC, EWS, ActiveSync, Autodiscover e frontend MAPI/HTTP | Nome, certificato, autenticazione e route di backend corrispondono? |
| SMTP e TLS | Accettazione e inoltro dei messaggi | Quale connettore ha accettato e quale hop successivo è stato scelto? |
| ESE | Database delle cassette postali, coda di trasporto e registri delle transazioni | Quale database e quale sequenza di log appartengono insieme? |
| PowerShell | Amministrazione tramite cmdlet e RBAC | Quali ruolo, ambito e contesto server si applicano? |

Per gli esperti è particolarmente importante la dipendenza da Active Directory. Il programma di installazione di Exchange estende lo schema e scrive la configurazione dell'organizzazione nella Configuration Partition. Gli attributi dei destinatari risiedono nella partizione del dominio. Un ritardo di replica o un controller di dominio non raggiungibile può quindi influire in modo diverso sulle varie funzioni.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-exchange-onprem.svg?v=20260813" title="Interaktive Infografik: Exchange-On-Premises-Pfad von Client und SMTP über Mailboxserver, Transport, Active Directory, ESE und DAG" loading="lazy">
  <a href="/images/kb-interaktiv-exchange-onprem.svg?v=20260813">Apri direttamente il grafico interattivo di Exchange On-Premises</a>.
</iframe>

## La pipeline di trasporto passo per passo

Con le basi tecniche è possibile leggere più precisamente il percorso dei messaggi. Una connessione SMTP in ingresso raggiunge dapprima il Front End Transport Service. Accetta il dialogo e lo inoltra al Transport Service; non deposita autonomamente il messaggio in una cassetta postale ([Flusso di posta e pipeline di trasporto](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow)).

Il **Transport Service** memorizza il messaggio nel proprio database delle code. Poi lo categorizza: i destinatari vengono risolti, vengono eseguite regole e agenti di trasporto, e il routing determina l'hop successivo. Per una cassetta postale locale, Mailbox Transport Delivery consegna il messaggio allo Store. Un messaggio inviato dalla cassetta postale ritorna al trasporto tramite Mailbox Transport Submission.

Questa sequenza spiega osservazioni tipiche. Un test SMTP riuscito dimostra solo l'accettazione da parte del frontend. Un evento `RECEIVE` nel Message Tracking non dimostra ancora la consegna. Solo gli eventi successivi, la coda ed eventualmente lo stato dello Store mostrano dove il processo si è interrotto ([Tracciamento dei messaggi](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking)).

Gli agenti di trasporto e le regole di flusso della posta possono rifiutare, reindirizzare, copiare o modificare i messaggi. Poiché possono essere create più istanze di trasporto, la ricerca non dovrebbe basarsi solo sull'oggetto. Network Message ID, Internet Message ID, mittente, destinatario, orario e server costituiscono insieme una traccia più affidabile.

## Routing, domini e connettori

Dopo l'accettazione, Exchange deve sapere se un destinatario è locale o se il messaggio viene inoltrato. I **domini accettati** descrivono questa relazione. Un dominio autorevole prevede tutti i destinatari validi nella propria organizzazione. Un dominio Internal Relay consente l'inoltro di destinatari sconosciuti. External Relay consegna interamente il dominio a un altro server di posta ([Domini accettati in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)).

I Receive Connector classificano le sessioni in ingresso in base al binding locale, all'intervallo IP remoto, all'autenticazione e alle autorizzazioni. I Send Connector selezionano un percorso in uscita in base a spazio di indirizzi, costo, server di origine e routing DNS o smart host. Più connettori compatibili vengono valutati secondo le regole di routing documentate; il nome di un connettore non ne determina la selezione ([Connettori nei server Exchange](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors), [Routing della posta in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing)).

Per il normale esercizio è sufficiente un modello semplice: il Receive Connector spiega **come arriva un messaggio**; il dominio accettato e la risoluzione dei destinatari spiegano **se Exchange è responsabile**; il Send Connector e il routing spiegano **dove prosegue**. Gli esperti aggiungono siti AD, gruppi di consegna, appartenenza a DAG, ambito dei connettori e regole di trasporto.

## Database delle cassette postali, log e checkpoint

Quando il trasporto consegna allo Store, inizia un'altra parte del sistema. Exchange archivia le cassette postali in database ESE. Le modifiche vengono dapprima scritte nei log delle transazioni e successivamente riportate nel file `.edb`. Il file di checkpoint registra fino a quale posizione del log sono state scritte le pagine del database ([Log delle transazioni e file di checkpoint](https://learn.microsoft.com/en-us/exchange/client-developer/backup-restore/transaction-logs-and-checkpoint-files-for-backup-and-restore-in-exchange)).

Questa sequenza permette il crash recovery, ma richiede file correlati. Un file `.edb` copiato senza i log corrispondenti e senza uno stato di arresto noto non è automaticamente ripristinabile. Allo stesso modo, un backup non deve eliminare in modo incontrollato file di log che sono ancora necessari per il ripristino o la replica.

Anche la coda di trasporto utilizza ESE, ma è un database separato con propri log. Il database delle cassette postali e la coda vengono quindi monitorati e ripristinati separatamente. Un database delle cassette postali integro non elimina un hop SMTP successivo bloccato; una coda vuota non ripara una copia danneggiata del database delle cassette postali ([Code e database delle code](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues)).

## Database Availability Group e Active Manager

Un singolo server Mailbox spiega il funzionamento normale. Per l'alta disponibilità, più server vengono riuniti in una **Database Availability Group**, DAG. Ogni database delle cassette postali possiede esattamente una copia attiva e può avere copie passive su altri membri DAG. Le modifiche vengono trasferite tramite replica dei log e dei blocchi e riprodotte sulle copie passive ([Gruppi di disponibilità del database](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups), [Copie del database delle cassette postali](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-copies)).

L'**Active Manager** nel Microsoft Exchange Replication Service decide quale copia è attiva. Best Copy and Server Selection valuta, tra l'altro, lo stato di copia e riproduzione, i blocchi di attivazione e l'integrità del server. Una Copy Queue pari a zero è quindi utile, ma non costituisce una prova completa che una copia possa essere attivata immediatamente ([Active Manager](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/active-manager)).

L'alta disponibilità del trasporto protegge un'altra sezione del percorso. Shadow Redundancy conserva una copia aggiuntiva mentre il messaggio è in transito. Safety Net conserva messaggi già elaborati per una possibile nuova consegna dopo l'attivazione del database. DAG, Shadow Redundancy e Safety Net si completano a vicenda; nessuna delle tre funzioni sostituisce un backup contro eliminazioni accidentali o danneggiamenti prolungati non rilevati ([Alta disponibilità del trasporto](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/transport-high-availability)).

## Accesso client e Autodiscover

Il database può essere integro e un utente può comunque non riuscire ad aprire Outlook. I Client Access Services accettano connessioni HTTPS e le inoltrano al backend sul server con il database attivo. Un load balancer richiede pertanto più di una porta TCP aperta: nome, certificato, endpoint del protocollo e integrità del backend devono corrispondere ([Architettura del protocollo Client Access](https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access)).

Autodiscover fornisce al client le impostazioni appropriate. I client interni al dominio possono utilizzare i Service Connection Point in Active Directory; i client esterni e altri client seguono procedure DNS e HTTPS. Gli errori sono spesso causati da SCP obsoleti, risposte DNS contraddittorie, nomi dei certificati errati o un frontend che inoltra al backend sbagliato ([Servizio Autodiscover](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)).

MAPI over HTTP è il trasporto Outlook tipico. Anche Outlook sul web, EWS e ActiveSync utilizzano HTTPS, ma dispongono di proprie directory virtuali, autenticazione e caratteristiche applicative. Un test OWA riuscito non dimostra quindi automaticamente l'integrità di una sessione MAPI/HTTP ([MAPI over HTTP](https://learn.microsoft.com/en-us/exchange/clients/mapi-over-http/mapi-over-http)).

## Active Directory e destinatari

Dopo il trasporto e l'accesso client, la directory rimane la base comune. Exchange archivia in Active Directory la configurazione dell'organizzazione e dei server nonché gli attributi dei destinatari. I cmdlet non scrivono questi dati in un database Exchange privato, bensì in AD tramite la logica di Exchange ([Active Directory in Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)).

Un problema relativo al destinatario viene quindi esaminato attraverso tre domande: l'oggetto corretto esiste? Il tipo, l'indirizzo primario, gli indirizzi proxy e gli attributi di destinazione sono corretti? La modifica ha raggiunto il controller di dominio utilizzato dal servizio Exchange interessato? Solo dopo vale la pena cercare nel trasporto.

Per gli esperti si aggiungono cataloghi globali, siti AD, Recipient Update, Address Book Policies e attributi ibridi. Le modifiche dirette con strumenti AD generici aggirano la convalida di Exchange e possono creare configurazioni sintatticamente presenti ma tecnicamente incoerenti.

## Sicurezza e controllo amministrativo

Exchange espone servizi SMTP e HTTPS ed elabora dati altamente privilegiati di directory e cassette postali. La base comprende Security Updates tempestivi, endpoint raggiungibili ridotti al minimo, certificati adeguati, account amministrativi protetti e modifiche tracciabili ([Security Updates di Exchange Server](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates), [Certificati TLS in Exchange Server](https://learn.microsoft.com/en-us/exchange/architecture/client-access/certificates)).

RBAC separa i compiti tramite ruoli, gruppi di ruoli e ambiti. I diritti delle cassette postali quali Full Access o Send As rimangono separati. Administrator Audit Logging registra le modifiche ai cmdlet, ma non sostituisce i registri del sistema operativo, di Active Directory e di sicurezza ([Autorizzazioni in Exchange Server](https://learn.microsoft.com/en-us/exchange/permissions/permissions), [Registrazione di controllo dell'amministratore](https://learn.microsoft.com/en-us/exchange/policy-and-compliance/admin-audit-logging/admin-audit-logging)).

Per gli esperti, l'interfaccia di amministrazione stessa fa parte del modello di protezione. EAC, Exchange Management Shell, Remote PowerShell, WinRM, RDP e l'accesso all'hypervisor dispongono di diritti e protocolli diversi. Un amministratore server compromesso può eseguire misure al di fuori di Exchange-RBAC; la separazione in livelli e account privilegiati distinti rimangono quindi importanti.

## Gestione: dal sintomo al server concreto

Managed Availability esegue probe, monitor e responder. Gli Health Set raggruppano questi risultati per funzione e possono attivare azioni di ripristino automatiche. Sono un buon punto di partenza, ma non una verifica end-to-end completa ([Managed Availability](https://learn.microsoft.com/en-us/exchange/high-availability/managed-availability/managed-availability)).

Per il flusso di posta, la diagnosi locale inizia con [`Get-Queue`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue) e [`Get-MessageTrackingLog`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog). Numero di code, hop successivo, ora di ripetizione e `LastError` vanno considerati insieme. Per i database seguono [`Get-MailboxDatabaseCopyStatus`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-mailboxdatabasecopystatus) e [`Test-ReplicationHealth`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/test-replicationhealth). [`Get-ServerHealth`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-serverhealth) mostra Health Set e monitor.

Questi cmdlet vengono eseguiti nella Exchange Management Shell su Windows Server supportati. I test di rete e DNS possono invece essere eseguiti da entrambe le piattaforme amministrative. [`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) verifica un endpoint TCP in Windows; [`nc`](https://man.openbsd.org/nc) esegue lo stesso test di porta in Unix. [`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) e [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) verificano il DNS. Per SMTP con STARTTLS è adatto [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/), per un dialogo SMTP controllato [`swaks`](https://jetmore.org/john/code/swaks/).

La sequenza diagnostica è: risolvere il nome pubblico o interno, verificare la connessione al frontend corretto, confermare l'accettazione nel log del protocollo, seguire gli eventi di tracking, verificare la coda e l'hop successivo e solo in caso di consegna locale esaminare Store e database.

## Backup e ripristino

L'alta disponibilità mantiene il servizio disponibile in caso di singoli guasti; il ripristino recupera uno stato precedente desiderato o perso. Exchange documenta Server Recovery, ripristino del database e Recovery Database come procedure diverse ([Backup, ripristino e disaster recovery](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery)).

Un inventario ripristinabile comprende almeno Active Directory, organizzazione Exchange e configurazione del server, certificati e chiavi private, database delle cassette postali con log, configurazione di connettori e regole, nonché parametri documentati di installazione e ripristino. Il Recovery Database consente di montare isolatamente un database ripristinato e trasferirne il contenuto alle cassette postali attive ([Ripristinare dati tramite un recovery database](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/restore-data-using-recovery-dbs)).

Gli esperti non verificano soltanto se un processo di backup ha avuto successo. Misurano quanto tempo richiede effettivamente il ripristino di Active Directory, di un server guasto, di un database e di singoli contenuti delle cassette postali. Vengono inoltre verificati quali sequenze di log sono necessarie, quali dipendenze DNS e dei certificati esistono e se, dopo il ripristino, i percorsi client e SMTP funzionano di nuovo.

## Evoluzione tecnica e limiti

Exchange 4.0 è apparso nel 1996. Le prime versioni utilizzavano una directory propria, MAPI ed ESE; SMTP e Active Directory sono diventati componenti centrali della piattaforma con Exchange 2000. Exchange 2007 ha introdotto ruoli server ed Exchange Management Shell. Exchange 2010 ha sostituito i vecchi modelli di clustering con la Database Availability Group ([Exchange Team: una breve storia del tempo](https://techcommunity.microsoft.com/blog/exchange/a-brief-history-of-time---exchange-server-way/589388), [Exchange Server 2007: riprogettazione del trasporto](https://techcommunity.microsoft.com/blog/exchange/motivations-for-the-exchange-server-2007-transport-redesign/600465)).

Le versioni successive hanno nuovamente riunito le funzioni Client Access e Mailbox in un componente server comune. Exchange Server Subscription Edition ha proseguito la linea di prodotti locali nel 2025 con il Modern Lifecycle. Numeri di build, percorsi di aggiornamento supportati e Security Updates vengono verificati nella documentazione corrente di Microsoft prima di ogni modifica ([Note sulla versione di Exchange Server SE](https://learn.microsoft.com/en-us/exchange/release-notes), [Numeri di build e date di rilascio di Exchange Server](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates)).

Exchange On-Premises è adatto quando l'organizzazione necessita di controllo sulla gestione del database, sui percorsi di rete e sull'integrazione locale ed è in grado di sostenere l'operatività 24/7 necessaria. Il rovescio della medaglia sono dipendenze complesse, manutenzione continua della sicurezza e responsabilità del ripristino. Un singolo server può sembrare semplice; un servizio Exchange affidabile è sempre anche un progetto di Active Directory, rete, certificati, storage e gestione.

## Fonti

- [Microsoft Learn – Documentazione di Exchange Server](https://learn.microsoft.com/en-us/exchange/exchange-server)
- [Microsoft Learn – Architettura di Exchange Server](https://learn.microsoft.com/en-us/exchange/architecture/architecture)
- [Microsoft Learn – Requisiti di sistema di Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/system-requirements)
- [Microsoft Learn – Active Directory in Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)
- [Microsoft Learn – Server Edge Transport](https://learn.microsoft.com/en-us/exchange/architecture/edge-transport-servers/edge-transport-servers)
- [Microsoft Learn – Flusso di posta e pipeline di trasporto](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow)
- [Microsoft Learn – Tracciamento dei messaggi](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking)
- [Microsoft Learn – Domini accettati in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)
- [Microsoft Learn – Connettori nei server Exchange](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors)
- [Microsoft Learn – Routing della posta in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing)
- [Microsoft Learn – Log delle transazioni e file di checkpoint](https://learn.microsoft.com/en-us/exchange/client-developer/backup-restore/transaction-logs-and-checkpoint-files-for-backup-and-restore-in-exchange)
- [Microsoft Learn – Code e database delle code](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues)
- [Microsoft Learn – Gruppi di disponibilità del database](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups)
- [Microsoft Learn – Monitorare i gruppi di disponibilità del database](https://learn.microsoft.com/en-us/exchange/high-availability/manage-ha/monitor-dags)
- [Microsoft Learn – Copie del database delle cassette postali](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-copies)
- [Microsoft Learn – Active Manager](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/active-manager)
- [Microsoft Learn – Alta disponibilità del trasporto](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/transport-high-availability)
- [Microsoft Learn – Architettura del protocollo Client Access](https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access)
- [Microsoft Learn – Servizio Autodiscover](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)
- [Microsoft Learn – MAPI over HTTP](https://learn.microsoft.com/en-us/exchange/clients/mapi-over-http/mapi-over-http)
- [Microsoft Learn – Interfacce di amministrazione di Exchange](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/admin-interface)
- [Microsoft Learn – Certificati TLS in Exchange Server](https://learn.microsoft.com/en-us/exchange/architecture/client-access/certificates)
- [Microsoft Learn – Autorizzazioni in Exchange Server](https://learn.microsoft.com/en-us/exchange/permissions/permissions)
- [Microsoft Learn – Registrazione di controllo dell'amministratore](https://learn.microsoft.com/en-us/exchange/policy-and-compliance/admin-audit-logging/admin-audit-logging)
- [Microsoft Learn – Managed Availability](https://learn.microsoft.com/en-us/exchange/high-availability/managed-availability/managed-availability)
- [Microsoft Learn – Get-Queue](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue)
- [Microsoft Learn – Get-MessageTrackingLog](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog)
- [Microsoft Learn – Get-MailboxDatabaseCopyStatus](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-mailboxdatabasecopystatus)
- [Microsoft Learn – Test-ReplicationHealth](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/test-replicationhealth)
- [Microsoft Learn – Get-ServerHealth](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-serverhealth)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc(1)](https://man.openbsd.org/nc)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [BIND 9 – Manuale di dig](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Swaks – Strumento di test SMTP](https://jetmore.org/john/code/swaks/)
- [Microsoft Learn – Backup, ripristino e disaster recovery](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery)
- [Microsoft Learn – Ripristinare dati tramite un recovery database](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/restore-data-using-recovery-dbs)
- [Exchange Team – Una breve storia del tempo](https://techcommunity.microsoft.com/blog/exchange/a-brief-history-of-time---exchange-server-way/589388)
- [Exchange Team – Exchange Server 2007: riprogettazione del trasporto](https://techcommunity.microsoft.com/blog/exchange/motivations-for-the-exchange-server-2007-transport-redesign/600465)
- [Microsoft Learn – Note sulla versione di Exchange Server SE](https://learn.microsoft.com/en-us/exchange/release-notes)
- [Microsoft Learn – Numeri di build e date di rilascio di Exchange Server](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates)
