---
title: "Exchange On-Premises: architettura e gestione del server"
blatt: "exchange-on-premises"
description: "Exchange Server nel proprio data center: ruoli Mailbox ed Edge, pipeline di trasporto, Active Directory, database ESE, DAG, accesso client, sicurezza, monitoraggio, backup e ripristino."
fakten:
  - label: Ruolo del prodotto
    wert: Piattaforma di messaggistica e groupware gestita autonomamente
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
    wert: Database Availability Group e copie di database
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
translationSourceHash: 8d30441ccb794fc2e8228dfe1fa084ee38a525dcb5e59222e182819a74596bbc
translationModel: gpt-5.6-terra
translatedAt: 2026-10-05T11:23:35.380Z
translationReview: automatic
---

# Exchange On-Premises: architettura e gestione del server

**Exchange On-Premises** significa che l'organizzazione gestisce Exchange Server nella propria infrastruttura. Controlla host Windows, Active Directory, certificati, servizi di trasporto, code, database delle cassette postali e ripristino. Microsoft fornisce il codice del prodotto, la documentazione e gli aggiornamenti; la disponibilità e la manutenzione sicura restano responsabilità dell'operatore ([Documentazione di Exchange Server](https://learn.microsoft.com/en-us/exchange/exchange-server), [Architettura di Exchange Server](https://learn.microsoft.com/en-us/exchange/architecture/architecture)).

La differenza pratica rispetto a Exchange Online emerge subito in caso di guasto. Un amministratore On-Prem può esaminare una coda di trasporto su un server specifico, verificare lo stato di una copia del database e passare in modo controllato a un'altra copia. Tuttavia, deve anche comprendere come interagiscono SMTP, Active Directory, ESE, Windows Failover Clustering, IIS e i servizi Exchange.

## Il server Mailbox è il componente centrale

I server Exchange moderni utilizzano il **server Mailbox** come componente comune. Include i servizi Client Access, che accettano e inoltrano le connessioni, i servizi di trasporto per il flusso dei messaggi e l'Information Store con i database delle cassette postali. Un'installazione può iniziare in piccolo; più server e copie di database estendono lo stesso modello di base per l'alta disponibilità ([Architettura di Exchange Server](https://learn.microsoft.com/en-us/exchange/architecture/architecture)).

Questa unificazione non significa che tutte le funzioni abbiano lo stesso stato. Un frontend HTTPS può essere raggiungibile anche se il database interessato non è montato. SMTP può accettare connessioni mentre un messaggio attende successivamente in una coda. La diagnosi segue quindi il percorso effettivo e non solo lo stato complessivo del server.

Il **ruolo Edge Transport** opzionale si trova tipicamente nella rete perimetrale ed elabora esclusivamente traffico SMTP. EdgeSync trasferisce informazioni selezionate su destinatari e configurazione in un'istanza AD LDS locale. Edge non contiene database delle cassette postali e non sostituisce i server Mailbox interni ([Server Edge Transport](https://learn.microsoft.com/en-us/exchange/architecture/edge-transport-servers/edge-transport-servers)).

## Stack tecnologico e dipendenze

Il componente server determina lo stack tecnologico. Exchange viene eseguito su versioni supportate di Windows Server e utilizza Active Directory per la configurazione dell'organizzazione, dei server e dei destinatari. IIS fornisce gli endpoint HTTP. PowerShell costituisce l'interfaccia di amministrazione. ESE memorizza i dati delle cassette postali e delle code in database separati ([Requisiti di sistema di Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/system-requirements), [Active Directory in Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)).

| Tecnologia | Compito nel funzionamento di Exchange | Domanda amministrativa importante |
|---|---|---|
| Windows Server | Processi, servizi, rete, archivio certificati e registri eventi | L'host è integro e aggiornato correttamente? |
| Active Directory | Organizzazione Exchange, server, destinatari, RBAC e informazioni di routing | La modifica corretta è visibile nei controller di dominio utilizzati? |
| IIS e HTTPS | Frontend per Outlook sul web, EAC, EWS, ActiveSync, Autodiscover e MAPI/HTTP | Nome, certificato, autenticazione e route di backend corrispondono? |
| SMTP e TLS | Accettazione e inoltro dei messaggi | Quale connettore ha accettato e quale hop successivo è stato scelto? |
| ESE | Database delle cassette postali, coda di trasporto e registri delle transazioni | Quale database e quale sequenza di log appartengono insieme? |
| PowerShell | Amministrazione mediante cmdlet e RBAC | Quali ruolo, ambito e contesto server si applicano? |

Per gli esperti, la dipendenza da Active Directory è particolarmente importante. Il programma di installazione di Exchange estende lo schema e scrive la configurazione dell'organizzazione nella partizione Configuration. Gli attributi dei destinatari si trovano nella partizione di dominio. Un ritardo di replica o un controller di dominio non raggiungibile può quindi influire diversamente sulle varie funzioni.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-exchange-onprem.svg?v=20260813" title="Interaktive Infografik: Exchange-On-Premises-Pfad von Client und SMTP über Mailboxserver, Transport, Active Directory, ESE und DAG" loading="lazy">
  <a href="/images/kb-interaktiv-exchange-onprem.svg?v=20260813">Apri direttamente il grafico interattivo di Exchange On-Premises</a>.
</iframe>

## La pipeline di trasporto passo dopo passo

Con le basi tecniche è possibile leggere più precisamente il percorso del messaggio. Una connessione SMTP in entrata raggiunge anzitutto il Front End Transport Service. Accetta il dialogo e lo inoltra al Transport Service; non consegna autonomamente il messaggio in una cassetta postale ([Flusso della posta e pipeline di trasporto](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow)).

Il **Transport Service** memorizza il messaggio nel proprio database delle code. Quindi lo categorizza: i destinatari vengono risolti, vengono eseguite regole e agenti di trasporto, e il routing determina l'hop successivo. Per una cassetta postale locale, Mailbox Transport Delivery consegna il messaggio allo Store. Un messaggio inviato dalla cassetta postale torna al trasporto tramite Mailbox Transport Submission.

Questa sequenza spiega osservazioni tipiche. Un test SMTP riuscito dimostra solo l'accettazione sul frontend. Un evento `RECEIVE` nel tracciamento dei messaggi non prova ancora la consegna. Solo gli eventi successivi, la coda e, se necessario, lo stato dello Store mostrano dove si è concluso il processo ([Tracciamento dei messaggi](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking)).

Gli agenti di trasporto e le regole di flusso della posta possono rifiutare, reindirizzare, copiare o modificare i messaggi. Poiché possono originarsi più istanze di trasporto, la ricerca non dovrebbe basarsi soltanto sull'oggetto. Network Message ID, Internet Message ID, mittente, destinatario, momento e server forniscono insieme una traccia più affidabile.

## Routing, domini e connettori

Dopo l'accettazione, Exchange deve sapere se un destinatario è locale o se il messaggio deve essere inoltrato. I **domini accettati** descrivono questa relazione. Un dominio autorevole prevede tutti i destinatari validi nella propria organizzazione. Un dominio relay interno consente l'inoltro di destinatari sconosciuti. External Relay consegna interamente il dominio a un altro server di posta ([Domini accettati in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)).

I connettori di ricezione classificano le sessioni in entrata in base al binding locale, all'intervallo di IP remoti, all'autenticazione e alle autorizzazioni. I connettori di invio selezionano un percorso in uscita in base a spazio indirizzi, costo, server di origine e routing DNS o smart host. Più connettori corrispondenti vengono valutati secondo le regole di routing documentate; il nome di un connettore non ne determina la selezione ([Connettori nei server Exchange](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors), [Routing della posta in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing)).

Per il normale funzionamento basta un modello semplice: il connettore di ricezione spiega **come un messaggio entra**; il dominio accettato e la risoluzione del destinatario spiegano **se Exchange è responsabile**; il connettore di invio e il routing spiegano **dove prosegue**. Gli esperti aggiungono siti AD, Delivery Groups, appartenenza alla DAG, ambito dei connettori e regole di trasporto.

## Database delle cassette postali, log e checkpoint

Quando il trasporto consegna allo Store, inizia un'altra parte del sistema. Exchange memorizza le cassette postali in database ESE. Le modifiche vengono prima scritte nei log delle transazioni e successivamente trasferite nel file `.edb`. Il file di checkpoint registra fino a quale posizione del log sono state scritte le pagine del database ([Log delle transazioni e file di checkpoint](https://learn.microsoft.com/en-us/exchange/client-developer/backup-restore/transaction-logs-and-checkpoint-files-for-backup-and-restore-in-exchange)).

Questa sequenza permette il crash recovery, ma richiede file correlati. Un file `.edb` copiato senza i log corrispondenti e senza uno stato di arresto noto non è automaticamente ripristinabile. Allo stesso modo, un backup non deve eliminare in modo incontrollato file di log ancora necessari per il ripristino o la replica.

Anche la coda di trasporto utilizza ESE, ma è un database separato con log propri. Il database delle cassette postali e la coda vengono pertanto monitorati e ripristinati separatamente. Un database delle cassette postali integro non elimina un hop SMTP successivo bloccato; una coda vuota non ripara una copia di cassetta postale danneggiata ([Code e database delle code](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues)).

## Database Availability Group e Active Manager

Un singolo server Mailbox spiega il funzionamento normale. Per l'alta disponibilità, più server vengono collegati in una **Database Availability Group**, DAG. Ogni database delle cassette postali possiede esattamente una copia attiva e può avere copie passive su altri membri della DAG. Le modifiche vengono trasferite mediante replica dei log e dei blocchi e riprodotte sulle copie passive ([Gruppi di disponibilità del database](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups), [Copie di database delle cassette postali](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-copies)).

L'**Active Manager** nel Microsoft Exchange Replication Service decide quale copia è attiva. Best Copy and Server Selection valuta, tra l'altro, lo stato di copia e riproduzione, i blocchi di attivazione e l'integrità del server. Una Copy Queue pari a zero è quindi utile, ma non dimostra completamente che una copia possa essere attivata immediatamente ([Active Manager](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/active-manager)).

L'alta disponibilità del trasporto protegge un'altra sezione del percorso. Shadow Redundancy conserva una copia aggiuntiva finché il messaggio è in transito. Safety Net conserva messaggi già elaborati per una possibile nuova consegna dopo l'attivazione del database. DAG, Shadow Redundancy e Safety Net si integrano a vicenda; nessuna delle tre funzioni sostituisce un backup contro l'eliminazione accidentale o un danneggiamento prolungato e non rilevato ([Alta disponibilità del trasporto](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/transport-high-availability)).

## Accesso client e Autodiscover

Il database può essere integro e un utente può comunque non riuscire ad aprire Outlook. I Client Access Services accettano connessioni HTTPS e le inoltrano al backend sul server con il database attivo. Un bilanciatore di carico necessita quindi di più di una porta TCP aperta: nome, certificato, endpoint del protocollo e integrità del backend devono corrispondere ([Architettura del protocollo Client Access](https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access)).

Autodiscover fornisce al client le impostazioni corrette. I client interni al dominio possono utilizzare i Service Connection Point in Active Directory; i client esterni e gli altri client seguono procedure DNS e HTTPS. Gli errori sono spesso causati da SCP obsoleti, risposte DNS contraddittorie, nomi di certificato errati o un frontend che inoltra al backend sbagliato ([Servizio Autodiscover](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)).

MAPI over HTTP è il tipico trasporto di Outlook. Anche Outlook sul web, EWS e ActiveSync utilizzano HTTPS, ma dispongono di directory virtuali, autenticazione e caratteristiche applicative proprie. Un test OWA riuscito non prova quindi automaticamente una sessione MAPI/HTTP integra ([MAPI over HTTP](https://learn.microsoft.com/en-us/exchange/clients/mapi-over-http/mapi-over-http)).

## Active Directory e destinatari

Dopo il trasporto e l'accesso client, la directory rimane la base comune. Exchange memorizza la configurazione dell'organizzazione e dei server nonché gli attributi dei destinatari in Active Directory. I cmdlet non scrivono questi dati in un database Exchange privato, ma in AD attraverso la logica di Exchange ([Active Directory in Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)).

Un problema di destinatario viene quindi esaminato seguendo tre domande: esiste l'oggetto corretto? Il tipo, l'indirizzo principale, gli indirizzi proxy e gli attributi di destinazione sono corretti? La modifica ha raggiunto il controller di dominio utilizzato dal servizio Exchange interessato? Solo dopo conviene cercare nel trasporto.

Per gli esperti si aggiungono cataloghi globali, siti AD, Recipient Update, Address Book Policies e attributi ibridi. Le modifiche dirette con strumenti AD generici aggirano la convalida di Exchange e possono creare configurazioni sintatticamente presenti ma tecnicamente incoerenti.

## Sicurezza e controllo amministrativo

Exchange pubblica servizi SMTP e HTTPS ed elabora dati di directory e cassette postali con privilegi elevati. La base comprende Security Updates tempestivi, endpoint raggiungibili ridotti al minimo, certificati adeguati, account amministrativi protetti e modifiche tracciabili ([Security Updates di Exchange Server](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates), [Certificati TLS in Exchange Server](https://learn.microsoft.com/en-us/exchange/architecture/client-access/certificates)).

RBAC separa le attività mediante ruoli, gruppi di ruoli e ambiti. I diritti sulle cassette postali quali Full Access o Send As restano separati. Administrator Audit Logging registra le modifiche ai cmdlet, ma non sostituisce i registri del sistema operativo, di Active Directory e di sicurezza ([Autorizzazioni in Exchange Server](https://learn.microsoft.com/en-us/exchange/permissions/permissions), [Registrazione di controllo degli amministratori](https://learn.microsoft.com/en-us/exchange/policy-and-compliance/admin-audit-logging/admin-audit-logging)).

Per gli esperti, l'interfaccia di amministrazione stessa fa parte del modello di protezione. EAC, Exchange Management Shell, Remote PowerShell, WinRM, RDP e l'accesso all'hypervisor hanno diritti e protocolli diversi. Un amministratore del server compromesso può eseguire azioni al di fuori di Exchange-RBAC; restano quindi importanti il tiering e account privilegiati separati.

## Gestione: dal sintomo al server concreto

Managed Availability esegue probe, monitor e responder. Gli Health Set raggruppano questi risultati per funzione e possono attivare azioni di ripristino automatiche. Sono un buon punto di partenza, ma non una verifica end-to-end completa ([Managed Availability](https://learn.microsoft.com/en-us/exchange/high-availability/managed-availability/managed-availability)).

Per il flusso della posta, la diagnosi locale inizia con [`Get-Queue`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue) e [`Get-MessageTrackingLog`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog). Numero di code, hop successivo, ora di nuovo tentativo e `LastError` vanno considerati insieme. Per i database seguono [`Get-MailboxDatabaseCopyStatus`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-mailboxdatabasecopystatus) e [`Test-ReplicationHealth`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/test-replicationhealth). [`Get-ServerHealth`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-serverhealth) mostra Health Set e monitor.

Questi cmdlet vengono eseguiti nella Exchange Management Shell su Windows Server supportati. I test di rete e DNS possono invece essere eseguiti da entrambe le piattaforme di amministrazione. [`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) verifica un endpoint TCP su Windows; [`nc`](https://man.openbsd.org/nc) esegue lo stesso test di porta in ambiente Unix. [`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) e [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) verificano il DNS. Per SMTP con STARTTLS è adatto [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/), per un dialogo SMTP controllato [`swaks`](https://jetmore.org/john/code/swaks/).

L'ordine della diagnosi è: risolvere il nome pubblico o interno, verificare la connessione al frontend corretto, confermare l'accettazione nel registro del protocollo, seguire gli eventi di tracciamento, verificare coda e hop successivo e, solo in caso di consegna locale, esaminare Store e database.

## Backup e ripristino

L'alta disponibilità mantiene disponibile il servizio in caso di singoli guasti; il ripristino ripristina uno stato precedente desiderato o perduto. Exchange documenta Server Recovery, ripristino del database e Recovery Database come procedure distinte ([Backup, ripristino e disaster recovery](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery)).

Un inventario ripristinabile comprende almeno Active Directory, organizzazione Exchange e configurazione dei server, certificati e chiavi private, database delle cassette postali con log, configurazione di connettori e regole nonché parametri documentati di installazione e ripristino. Il Recovery Database consente di montare isolatamente un database ripristinato e trasferire il contenuto nelle cassette postali attive ([Ripristinare dati mediante un database di ripristino](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/restore-data-using-recovery-dbs)).

Gli esperti non verificano solo che un processo di backup sia riuscito. Misurano quanto tempo richiede effettivamente il ripristino di Active Directory, di un server guasto, di un database e di singoli contenuti delle cassette postali. Vengono inoltre verificati le sequenze di log necessarie, le dipendenze DNS e dei certificati e il corretto funzionamento dei percorsi client e SMTP dopo il ripristino.

## Evoluzione tecnica e limiti

Exchange 4.0 è apparso nel 1996. Le prime versioni utilizzavano una directory propria, MAPI ed ESE; SMTP e Active Directory sono diventati componenti centrali della piattaforma con Exchange 2000. Exchange 2007 ha introdotto i ruoli server e Exchange Management Shell. Exchange 2010 ha sostituito i precedenti modelli di cluster con la Database Availability Group ([Exchange Team: una breve storia del tempo](https://techcommunity.microsoft.com/blog/exchange/a-brief-history-of-time---exchange-server-way/589388), [Exchange Server 2007: riprogettazione del trasporto](https://techcommunity.microsoft.com/blog/exchange/motivations-for-the-exchange-server-2007-transport-redesign/600465)).

Le versioni successive hanno nuovamente riunito le funzioni Client Access e Mailbox in un componente server comune. Exchange Server Subscription Edition ha proseguito la linea di prodotti locale nel Modern Lifecycle nel 2025. Prima di ogni modifica vengono verificati nella documentazione Microsoft aggiornata numeri di build, percorsi di aggiornamento supportati e Security Updates ([Note sulla versione di Exchange Server SE](https://learn.microsoft.com/en-us/exchange/release-notes), [Numeri di build e date di rilascio di Exchange Server](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates)).

Exchange On-Premises è adatto quando l'organizzazione necessita di controllo sul funzionamento dei database, sui percorsi di rete e sull'integrazione locale, e può garantire l'operatività necessaria 24/7. Il rovescio della medaglia sono dipendenze complesse, manutenzione continua della sicurezza e responsabilità del ripristino. Un singolo server può sembrare semplice; un servizio Exchange affidabile è sempre anche un progetto di Active Directory, rete, certificati, storage e gestione operativa.

## Fonti

- [Microsoft Learn – Documentazione di Exchange Server](https://learn.microsoft.com/en-us/exchange/exchange-server)
- [Microsoft Learn – Architettura di Exchange Server](https://learn.microsoft.com/en-us/exchange/architecture/architecture)
- [Microsoft Learn – Requisiti di sistema di Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/system-requirements)
- [Microsoft Learn – Active Directory in Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)
- [Microsoft Learn – Server Edge Transport](https://learn.microsoft.com/en-us/exchange/architecture/edge-transport-servers/edge-transport-servers)
- [Microsoft Learn – Flusso della posta e pipeline di trasporto](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow)
- [Microsoft Learn – Tracciamento dei messaggi](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking)
- [Microsoft Learn – Domini accettati in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)
- [Microsoft Learn – Connettori nei server Exchange](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors)
- [Microsoft Learn – Routing della posta in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing)
- [Microsoft Learn – Log delle transazioni e file di checkpoint](https://learn.microsoft.com/en-us/exchange/client-developer/backup-restore/transaction-logs-and-checkpoint-files-for-backup-and-restore-in-exchange)
- [Microsoft Learn – Code e database delle code](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues)
- [Microsoft Learn – Gruppi di disponibilità del database](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups)
- [Microsoft Learn – Monitorare i gruppi di disponibilità del database](https://learn.microsoft.com/en-us/exchange/high-availability/manage-ha/monitor-dags)
- [Microsoft Learn – Copie di database delle cassette postali](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-copies)
- [Microsoft Learn – Active Manager](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/active-manager)
- [Microsoft Learn – Alta disponibilità del trasporto](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/transport-high-availability)
- [Microsoft Learn – Architettura del protocollo Client Access](https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access)
- [Microsoft Learn – Servizio Autodiscover](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)
- [Microsoft Learn – MAPI over HTTP](https://learn.microsoft.com/en-us/exchange/clients/mapi-over-http/mapi-over-http)
- [Microsoft Learn – Interfacce di amministrazione di Exchange](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/admin-interface)
- [Microsoft Learn – Certificati TLS in Exchange Server](https://learn.microsoft.com/en-us/exchange/architecture/client-access/certificates)
- [Microsoft Learn – Autorizzazioni in Exchange Server](https://learn.microsoft.com/en-us/exchange/permissions/permissions)
- [Microsoft Learn – Registrazione di controllo degli amministratori](https://learn.microsoft.com/en-us/exchange/policy-and-compliance/admin-audit-logging/admin-audit-logging)
- [Microsoft Learn – Managed Availability](https://learn.microsoft.com/en-us/exchange/high-availability/managed-availability/managed-availability)
- [Microsoft Learn – Get-Queue](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue)
- [Microsoft Learn – Get-MessageTrackingLog](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog)
- [Microsoft Learn – Get-MailboxDatabaseCopyStatus](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-mailboxdatabasecopystatus)
- [Microsoft Learn – Test-ReplicationHealth](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/test-replicationhealth)
- [Microsoft Learn – Get-ServerHealth](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-serverhealth)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc(1)](https://man.openbsd.org/nc)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [BIND 9 – manuale di dig](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Swaks – strumento di test SMTP](https://jetmore.org/john/code/swaks/)
- [Microsoft Learn – Backup, ripristino e disaster recovery](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery)
- [Microsoft Learn – Ripristinare dati mediante un database di ripristino](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/restore-data-using-recovery-dbs)
- [Exchange Team – Una breve storia del tempo](https://techcommunity.microsoft.com/blog/exchange/a-brief-history-of-time---exchange-server-way/589388)
- [Exchange Team – Exchange Server 2007: riprogettazione del trasporto](https://techcommunity.microsoft.com/blog/exchange/motivations-for-the-exchange-server-2007-transport-redesign/600465)
- [Microsoft Learn – Note sulla versione di Exchange Server SE](https://learn.microsoft.com/en-us/exchange/release-notes)
- [Microsoft Learn – Numeri di build e date di rilascio di Exchange Server](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates)
