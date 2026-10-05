---
title: "Microsoft Exchange: famiglia di prodotti e modelli operativi"
blatt: "exchange"
description: "Microsoft Exchange come famiglia di prodotti: termini e protocolli comuni, differenze tra Exchange Online e Exchange Server, nonché i compiti della modalità ibrida e del flusso di posta ibrido."
fakten:
  - label: Famiglia di prodotti
    wert: Exchange Online ed Exchange Server
    href: https://learn.microsoft.com/en-us/exchange/
  - label: Funzioni principali
    wert: E-mail, calendario, contatti, rubrica e criteri
    href: https://learn.microsoft.com/en-us/exchange/recipients/recipients
  - label: Gestione cloud
    wert: Exchange Online all'interno di Microsoft 365
    href: https://learn.microsoft.com/en-us/exchange/exchange-online
  - label: Gestione in proprio
    wert: Exchange Server su Windows Server e Active Directory
    href: https://learn.microsoft.com/en-us/exchange/exchange-server
  - label: Coesistenza
    wert: Exchange Hybrid collega l'organizzazione locale ed Exchange Online
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: Trasporto della posta
    wert: SMTP, connettori, regole, code e recapito
    href: https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow
  - label: Accesso client
    wert: HTTPS, MAPI/HTTP, Outlook sul Web e client mobili
    href: https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access
  - label: Destinatari
    wert: Cassette postali, gruppi, contatti, utenti di posta e risorse
    href: https://learn.microsoft.com/en-us/exchange/recipients/recipients
  - label: Directory
    wert: Active Directory in locale, Microsoft Entra ID nel cloud
    href: https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory
  - label: Amministrazione
    wert: Exchange Admin Center, PowerShell e autorizzazioni basate sui ruoli
    href: https://learn.microsoft.com/en-us/exchange/permissions/permissions
  - label: Diagnostica cloud
    wert: Message Trace, report e Microsoft 365 Service Health
    href: https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message
  - label: Diagnostica server
    wert: Code, Message Tracking, Health Sets e copie di database
    href: https://learn.microsoft.com/en-us/exchange/server-health/server-health
werbung:
  - tools
  - newsletter
ctaThemen:
  - microsoft-365-exchange
  - exchange-onprem-hybrid
translationSourceHash: f76fa79338367d134f158edf1b05fbee051e68cffacc657f421ca0bbdd03dc2f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-05T11:15:16.843Z
translationReview: automatic
---

# Microsoft Exchange: famiglia di prodotti e modelli operativi

Il nome **Microsoft Exchange** indica oggi due piattaforme strettamente correlate, ma gestite in modo diverso. Con **Exchange Online**, Microsoft gestisce i server, le copie dei database e l'infrastruttura di trasporto interna. Con **Exchange Server**, host, Active Directory, database, code, certificati e ripristino restano sotto la responsabilità dell'organizzazione. **Exchange Hybrid** collega entrambe le organizzazioni quando le cassette postali o le funzioni sono distribuite tra i due lati ([Microsoft Learn: Exchange](https://learn.microsoft.com/en-us/exchange/), [Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)).

Questa distinzione è il punto di partenza per ogni ulteriore domanda. Una cassetta postale può trovarsi nel cloud o nel proprio data center. Il mittente visibile, il dominio SMTP e la rubrica possono comunque essere utilizzati congiuntamente. Solo quando sono definiti il percorso della cassetta postale, l'origine dei relativi attributi del destinatario e il percorso effettivo del messaggio, è possibile analizzare in modo sensato il recapito, le autorizzazioni e gli errori.

## Chiarire innanzitutto: dove si trova la cassetta postale?

Exchange gestisce e-mail, calendari, contatti, attività, oggetti della rubrica e diritti di accesso. Per gli utenti, il funzionamento è in gran parte identico. Per gli amministratori, invece, la posizione della cassetta postale cambia quasi ogni strumento e responsabilità.

| Domanda | Exchange Online | Exchange On-Premises | Exchange Hybrid |
|---|---|---|---|
| Chi gestisce i server delle cassette postali? | Microsoft | l'organizzazione stessa | entrambi i lati per le rispettive cassette postali |
| Dove vengono gestiti i destinatari? | Exchange Online ed Entra ID | Exchange Server e Active Directory | di norma creati localmente e sincronizzati con Entra ID; il modello esatto deve essere documentato |
| Dove si traccia un messaggio? | Message Trace | Message Tracking Logs e code | su entrambi i lati, collegati tramite ora, mittente, destinatario e ID dei messaggi |
| Chi può attivare copie di database? | Microsoft | l'amministrazione Exchange interna | il gestore del lato interessato |
| Cosa collega entrambi i lati? | non applicabile | non applicabile | sincronizzazione delle directory, relazioni tra organizzazioni, OAuth, Autodiscover e connettori SMTP |

La tabella è solo una panoramica. I quattro approfondimenti trattano separatamente i modelli operativi:

- [Exchange Online](/kb/exchange-online) illustra gli oggetti del tenant, EOP, connettori, Message Trace, conservazione e la gestione di un servizio cloud.
- [Exchange On-Premises](/kb/exchange-on-premises) segue la pipeline di trasporto, i database ESE, i DAG, Active Directory e il ripristino nel proprio data center.
- [Exchange Hybrid](/kb/exchange-hybrid) tratta sincronizzazione delle directory, gestione dei destinatari, OAuth, relazioni tra organizzazioni, Autodiscover e Hybrid Configuration Wizard.
- [Flusso di posta ibrido](/kb/hybrid-mailfluss) segue i messaggi tra Internet, Exchange Online, organizzazione locale e gateway di posta facoltativo.

## Struttura tecnica comune: ciò che resta uguale in tutte le varianti di Exchange

Dopo aver chiarito la sede operativa, è utile esaminare il modello funzionale comune. Exchange conosce **destinatari**, **messaggi**, **cassette postali**, **regole di trasporto**, **domini** e **connettori**. Questi termini ricorrono sia nel cloud sia sui server propri, anche se i sistemi sottostanti sono accessibili in modo diverso ([Microsoft Learn: Recipients](https://learn.microsoft.com/en-us/exchange/recipients/recipients), [Mail flow and the transport pipeline](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow)).

Un destinatario è anzitutto un oggetto di directory abilitato alla posta. Possiede indirizzi e un tipo, ad esempio cassetta postale utente, cassetta postale condivisa, gruppo, contatto o utente di posta. L'oggetto risponde alla domanda **chi** rappresenta un indirizzo e **dove** Exchange deve recapitare. La cassetta postale memorizza quindi gli elementi effettivi. Per questo possono coesistere un oggetto destinatario difettoso e una cassetta postale integra, o viceversa.

Anche il percorso dei messaggi segue su entrambe le piattaforme gli stessi passaggi generali: Exchange accetta un messaggio SMTP, risolve i destinatari, applica regole di trasporto e funzioni di protezione, sceglie la destinazione successiva e recapita il messaggio in una cassetta postale o a un ulteriore hop SMTP. L'implementazione esatta differisce. Sui server propri l'amministratore può vedere code e log di tracciamento locali; in Exchange Online sono disponibili Message Trace, report e Service Health ([Exchange Server mail flow](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow), [Trace an email message in Exchange Online](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-exchange.svg?v=20260813" title="Interaktive Infografik: Exchange-Produktfamilie mit Transport, Postfächern, Exchange Online und Hybridverbindungen" loading="lazy">
  <a href="/images/kb-interaktiv-exchange.svg?v=20260813">Aprire direttamente la panoramica interattiva di Exchange</a>.
</iframe>

## Dall'indirizzo al percorso del messaggio

Il dominio SMTP comune porta spesso alla falsa supposizione che tutti i messaggi seguano lo stesso percorso. In realtà, una combinazione di DNS, Accepted Domains, oggetto destinatario, connettori e regole determina dove un messaggio viene inviato successivamente.

Un'**Accepted Domain** indica a Exchange come deve essere trattato un dominio. Per un dominio autorevole, Exchange si aspetta tutti i destinatari validi nella propria directory. Per un dominio Internal Relay, i destinatari sconosciuti possono essere inoltrati a un altro sistema. Questa impostazione non è quindi una semplice voce d'inventario, ma parte della decisione di recapito ([Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains), [Accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains)).

I **connettori** determinano quindi da quali sistemi Exchange accetta messaggi e a quali sistemi li invia. In Exchange Server, i connettori Receive e Send lavorano con associazioni locali, spazi di indirizzamento, server di origine, smart host e autorizzazioni. Exchange Online utilizza connettori Inbound e Outbound per le relazioni con la propria infrastruttura o con partner ([Connectors on Exchange servers](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors), [Set up connectors to route mail](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail)).

Solo a questo punto diventano importanti le varianti ibride. Centralized Mail Transport, un gateway a monte o un dominio destinatari condiviso modificano hop e responsabilità aggiuntivi. Per questo sono trattati nell'articolo dedicato [Flusso di posta ibrido](/kb/hybrid-mailfluss), non tra gli argomenti relativi a identità o client.

## Dall'accesso alla cassetta postale

Il percorso del messaggio non spiega ancora come Outlook trovi la sua cassetta postale. A questo scopo Exchange utilizza **Autodiscover**. Un client parte dall'identità dell'utente e determina il punto finale di servizio appropriato. In locale possono essere coinvolti Service Connection Point di Active Directory e DNS; in Exchange Online, gli endpoint di Microsoft 365 conducono al servizio cloud ([Microsoft Learn: Autodiscover](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)).

Dopo aver individuato l'endpoint, l'accesso moderno di Outlook avviene tramite HTTPS, in particolare con MAPI over HTTP. Anche Outlook sul Web, Exchange ActiveSync e diverse API utilizzano HTTPS, ma ciascuno con protocolli applicativi e autorizzazioni propri. Un accesso riuscito al portale Microsoft 365 non dimostra quindi automaticamente che Autodiscover, il protocollo Outlook o l'accesso alla specifica cassetta postale funzionino ([Client Access protocol architecture](https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access), [MAPI over HTTP](https://learn.microsoft.com/en-us/exchange/clients/mapi-over-http/mapi-over-http)).

Nella modalità ibrida si aggiunge un'ulteriore decisione: la cassetta postale è locale o online? Autodiscover e gli attributi del destinatario devono indirizzare il client al lato corretto. Solo dopo entrano in gioco funzioni tra organizzazioni, come disponibilità/occupato o spostamenti delle cassette postali. Questa sequenza è trattata passo dopo passo nell'articolo [Exchange Hybrid](/kb/exchange-hybrid).

## Amministrazione e autorizzazioni

Una volta compresi i percorsi dei dati e degli accessi, si pone la domanda su chi possa modificarli. Exchange utilizza il controllo degli accessi basato sui ruoli. I ruoli contengono cmdlet e parametri, i gruppi di ruoli o le assegnazioni di ruolo li collegano agli amministratori e gli ambiti ne limitano l'area di applicazione. Exchange Online ed Exchange Server dispongono di concetti RBAC correlati, ma di configurazioni separate ([Permissions in Exchange Server](https://learn.microsoft.com/en-us/exchange/permissions/permissions), [Permissions in Exchange Online](https://learn.microsoft.com/en-us/exchange/permissions-exo/permissions-exo)).

Nella pratica quotidiana, questo significa che un ruolo di amministratore Entra, un gruppo di ruoli Exchange e un'autorizzazione della cassetta postale non sono la stessa cosa. **Full Access** consente di aprire una cassetta postale, **Send As** di inviare come destinatario e **Send on Behalf** di inviare in modo riconoscibile per conto di qualcuno. Nessuna di queste autorizzazioni, da sola, spiega se un'applicazione può accedere tramite Microsoft Graph o EWS ([Manage permissions for recipients in Exchange Online](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/manage-permissions-for-recipients)).

La domanda per esperti non è quindi «L'utente è amministratore?», bensì: quale identità esegue l'accesso, quale ruolo si applica in quale organizzazione Exchange, quale oggetto viene contattato e quale autorizzazione aggiuntiva della cassetta postale o dell'applicazione viene verificata?

## Operatività e ricerca guasti iniziano dal lato corretto

Una diagnosi efficace inizia con tre informazioni: **utente o destinatario interessato, orario preciso e posizione della cassetta postale**. Successivamente si segue il percorso nell'ordine DNS o Autodiscover, accesso, endpoint Exchange, risoluzione del destinatario, evento di trasporto e recapito nella cassetta postale.

Per Exchange Online, Message Trace e Microsoft 365 Service Health forniscono lo stato del servizio visibile al cliente. Per Exchange Server si aggiungono code locali, Message Tracking Logs, Health Sets, Event Logs e copie di database. Nella modalità ibrida, le evidenze di entrambi i lati vengono riunite su una linea temporale comune ([Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq), [Server health and performance](https://learn.microsoft.com/en-us/exchange/server-health/server-health)).

Gli articoli di approfondimento contengono blocchi diagnostici Windows/Unix adeguati e la documentazione ufficiale degli strumenti utilizzati. Qui basta la regola operativa più importante: determinare prima il luogo e il percorso, poi scegliere lo strumento.

## Archiviazione dei dati, disponibilità e ripristino

La differenza tra cloud e gestione in proprio è più evidente nel ripristino. Exchange Server memorizza le cassette postali in database ESE con registri delle transazioni. I Database Availability Group replicano copie dei database e consentono attivazioni su altri server. L'organizzazione resta tuttavia responsabile della strategia di backup, della ripristinabilità e della dipendenza da Active Directory ([Database availability groups](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups), [Backup, restore and disaster recovery](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery)).

Exchange Online gestisce la ridondanza del database e del trasporto come parte del servizio. Gli amministratori del tenant lavorano invece con elementi eliminati, Single Item Recovery, conservazione, hold ed eventualmente requisiti di backup esterni. La resilienza del servizio Microsoft e una regola di conservazione funzionale rispondono a domande diverse: la prima protegge il servizio in esecuzione, l'altra determina quali contenuti restano conservati dopo l'eliminazione o a fini di conformità ([Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency), [Retention policies for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange)).

Nella modalità ibrida, entrambi i modelli di ripristino devono essere documentati parallelamente. Sono inoltre necessari sincronizzazione delle directory, certificati, configurazione OAuth e connettori per ristabilire il collegamento dopo un guasto. Queste configurazioni non contengono dati delle cassette postali, ma determinano se le due organizzazioni Exchange possano tornare a collaborare.

## Evoluzione tecnica

Exchange 4.0 apparve nel 1996 come successore delle precedenti piattaforme di posta Microsoft. Le prime versioni utilizzavano una directory proprietaria, MAPI e la famiglia di database ESE; i protocolli Internet acquisirono gradualmente maggiore importanza. Con Exchange 2000, Active Directory e SMTP divennero componenti centrali. Exchange 2007 introdusse ruoli server distinti e Exchange Management Shell, Exchange 2010 il Database Availability Group ([Exchange Team: A brief history of time](https://techcommunity.microsoft.com/blog/exchange/a-brief-history-of-time---exchange-server-way/589388), [Exchange Team: Exchange Server 2007 transport redesign](https://techcommunity.microsoft.com/blog/exchange/motivations-for-the-exchange-server-2007-transport-redesign/600465)).

Parallelamente, Microsoft ha sviluppato le offerte Exchange ospitate fino a Exchange Online. Hybrid non è quindi nato come prodotto singolo, bensì come collegamento di due organizzazioni Exchange autonome. Exchange Server Subscription Edition ha proseguito questa linea di sviluppo locale nel Modern Lifecycle dal 2025. Le informazioni specifiche su build e aggiornamenti vengono verificate prima delle modifiche nella documentazione Microsoft costantemente aggiornata ([Exchange Server SE release notes](https://learn.microsoft.com/en-us/exchange/release-notes), [Exchange Server Subscription Edition lifecycle](https://learn.microsoft.com/en-us/lifecycle/products/exchange-server-subscription-edition)).

## Fonti

- [Microsoft Learn – Exchange](https://learn.microsoft.com/en-us/exchange/)
- [Microsoft Learn – Exchange Online](https://learn.microsoft.com/en-us/exchange/exchange-online)
- [Microsoft Learn – Exchange Server](https://learn.microsoft.com/en-us/exchange/exchange-server)
- [Microsoft Learn – Exchange Server architecture](https://learn.microsoft.com/en-us/exchange/architecture/architecture)
- [Microsoft Learn – Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)
- [Microsoft Learn – Recipients](https://learn.microsoft.com/en-us/exchange/recipients/recipients)
- [Microsoft Learn – Mail flow and the transport pipeline](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow)
- [Microsoft Learn – Trace an email message](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message)
- [Microsoft Learn – Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq)
- [Microsoft Learn – Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)
- [Microsoft Learn – Accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains)
- [Microsoft Learn – Connectors on Exchange servers](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors)
- [Microsoft Learn – Set up connectors to route mail](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail)
- [Microsoft Learn – Autodiscover](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)
- [Microsoft Learn – Client Access protocol architecture](https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access)
- [Microsoft Learn – MAPI over HTTP](https://learn.microsoft.com/en-us/exchange/clients/mapi-over-http/mapi-over-http)
- [Microsoft Learn – Active Directory in Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)
- [Microsoft Learn – Permissions in Exchange Server](https://learn.microsoft.com/en-us/exchange/permissions/permissions)
- [Microsoft Learn – Permissions in Exchange Online](https://learn.microsoft.com/en-us/exchange/permissions-exo/permissions-exo)
- [Microsoft Learn – Manage permissions for recipients in Exchange Online](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/manage-permissions-for-recipients)
- [Microsoft Learn – Server health and performance](https://learn.microsoft.com/en-us/exchange/server-health/server-health)
- [Microsoft Learn – Database availability groups](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups)
- [Microsoft Learn – Backup, restore and disaster recovery](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery)
- [Microsoft Service Assurance – Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency)
- [Microsoft Purview – Retention policies for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange)
- [Exchange Team – A brief history of time](https://techcommunity.microsoft.com/blog/exchange/a-brief-history-of-time---exchange-server-way/589388)
- [Exchange Team – Exchange Server 2007 transport redesign](https://techcommunity.microsoft.com/blog/exchange/motivations-for-the-exchange-server-2007-transport-redesign/600465)
- [Microsoft Learn – Exchange Server SE release notes](https://learn.microsoft.com/en-us/exchange/release-notes)
- [Microsoft Lifecycle – Exchange Server Subscription Edition](https://learn.microsoft.com/en-us/lifecycle/products/exchange-server-subscription-edition)
