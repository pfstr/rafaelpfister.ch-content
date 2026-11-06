---
title: "Microsoft Exchange: famiglia di prodotti e modelli operativi"
blatt: "exchange"
description: "Microsoft Exchange come famiglia di prodotti: concetti e protocolli comuni, differenze tra Exchange Online e Exchange Server, nonché i compiti della modalità ibrida e del flusso di posta ibrido."
fakten:
  - label: Famiglia di prodotti
    wert: Exchange Online ed Exchange Server
    href: https://learn.microsoft.com/en-us/exchange/
  - label: Funzioni principali
    wert: E-mail, calendario, contatti, rubrica e criteri
    href: https://learn.microsoft.com/en-us/exchange/recipients/recipients
  - label: Servizio cloud
    wert: Exchange Online all'interno di Microsoft 365
    href: https://learn.microsoft.com/en-us/exchange/exchange-online
  - label: Gestione in proprio
    wert: Exchange su Windows Server e Active Directory
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
    wert: Caselle di posta, gruppi, contatti, utenti di posta e risorse
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
translationSourceHash: 120d6e74efed65ae14cfc80b5e66597bd4ea19799d6b0c4d9e22fce0152b8d30
translationModel: gpt-5.6-terra
translatedAt: 2026-11-06T03:30:13.184Z
translationReview: automatic
---

# Microsoft Exchange: famiglia di prodotti e modelli operativi

Il nome **Microsoft Exchange** indica oggi due piattaforme strettamente correlate, ma gestite in modo diverso. Con **Exchange Online**, Microsoft gestisce i server, le copie dei database e l'infrastruttura di trasporto interna. Con **Exchange Server**, host, Active Directory, database, code, certificati e ripristino sono sotto la responsabilità dell'organizzazione. **Exchange Hybrid** collega entrambe le organizzazioni quando le caselle di posta o le funzioni sono distribuite tra i due lati ([Microsoft Learn: Exchange](https://learn.microsoft.com/en-us/exchange/), [Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)).

Questa distinzione è il punto di partenza per ogni altra domanda. Una casella di posta può trovarsi nel cloud o nel proprio data center. Il mittente visibile, il dominio SMTP e la rubrica possono comunque essere utilizzati congiuntamente. Solo quando sono noti la posizione della casella di posta, l'origine dei suoi attributi destinatario e l'effettivo percorso del messaggio è possibile analizzare in modo sensato il recapito, le autorizzazioni e gli errori.

## Prima chiarire: dove si trova la casella di posta?

Exchange gestisce e-mail, calendari, contatti, attività, oggetti della rubrica e diritti di accesso. Per gli utenti questo appare sostanzialmente uguale. Per gli amministratori, invece, la posizione della casella di posta cambia quasi ogni strumento e ogni responsabilità.

| Domanda | Exchange Online | Exchange On-Premises | Exchange Hybrid |
|---|---|---|---|
| Chi gestisce i server delle caselle di posta? | Microsoft | la propria organizzazione | entrambi i lati per le rispettive caselle di posta |
| Dove vengono gestiti i destinatari? | Exchange Online ed Entra ID | Exchange Server e Active Directory | generalmente creati in locale e sincronizzati con Entra ID; il modello esatto deve essere documentato |
| Dove viene tracciato un messaggio? | Message Trace | Message Tracking Logs e code | su entrambi i lati, collegati tramite ora, mittente, destinatario e ID messaggio |
| Chi può attivare copie di database? | Microsoft | la propria amministrazione Exchange | il gestore del lato interessato |
| Cosa collega entrambi i lati? | non applicabile | non applicabile | sincronizzazione delle directory, relazioni organizzative, OAuth, Autodiscover e connettori SMTP |

La tabella offre solo una panoramica. I quattro approfondimenti trattano separatamente i modelli operativi:

- [Exchange Online](/kb/exchange-online) spiega gli oggetti tenant, EOP, i connettori, Message Trace, la conservazione e la gestione di un servizio cloud.
- [Exchange On-Premises](/kb/exchange-on-premises) segue la pipeline di trasporto, i database ESE, le DAG, Active Directory e il recovery nel proprio data center.
- [Exchange Hybrid](/kb/exchange-hybrid) tratta la sincronizzazione delle directory, la gestione dei destinatari, OAuth, le relazioni organizzative, Autodiscover e Hybrid Configuration Wizard.
- [Flusso di posta ibrido](/kb/hybrid-mailfluss) segue i messaggi tra Internet, Exchange Online, organizzazione locale e gateway di posta opzionale.

## Struttura tecnica comune: ciò che rimane uguale in tutte le varianti di Exchange

Dopo aver chiarito il luogo operativo, vale la pena esaminare il modello funzionale comune. Exchange conosce **destinatari**, **messaggi**, **caselle di posta**, **regole di trasporto**, **domini** e **connettori**. Questi concetti compaiono sia nel cloud sia sui server propri, anche se i sistemi sottostanti sono accessibili in modo diverso ([Microsoft Learn: Recipients](https://learn.microsoft.com/en-us/exchange/recipients/recipients), [Mail flow and the transport pipeline](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow)).

Un destinatario è innanzitutto un oggetto di directory abilitato alla posta. Possiede indirizzi e un tipo, ad esempio casella di posta utente, casella condivisa, gruppo, contatto o utente di posta. L'oggetto risponde alla domanda su **chi** rappresenta un indirizzo e **dove** Exchange deve recapitare. La casella di posta memorizza quindi gli elementi effettivi. Per questo un oggetto destinatario difettoso e una casella di posta integra possono esistere contemporaneamente, o viceversa.

Anche il percorso del messaggio segue gli stessi passaggi generali su entrambe le piattaforme: Exchange accetta un messaggio SMTP, risolve i destinatari, applica regole di trasporto e funzioni di protezione, seleziona la destinazione successiva e recapita in una casella di posta oppure a un ulteriore hop SMTP. L'implementazione precisa differisce. Sui server propri l'amministratore può vedere code e log di tracciamento locali; in Exchange Online sono disponibili Message Trace, report e Service Health ([Exchange Server mail flow](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow), [Trace an email message in Exchange Online](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-exchange.svg?v=20260813" title="Interaktive Infografik: Exchange-Produktfamilie mit Transport, Postfächern, Exchange Online und Hybridverbindungen" loading="lazy">
  <a href="/images/kb-interaktiv-exchange.svg?v=20260813">Aprire direttamente la panoramica interattiva di Exchange</a>.
</iframe>

## Dall'indirizzo al percorso del messaggio

Il dominio SMTP comune porta spesso all'errata supposizione che tutti i messaggi seguano lo stesso percorso. In realtà, una combinazione di DNS, Accepted Domains, oggetto destinatario, connettori e regole determina dove un messaggio viene inviato successivamente.

Un'**Accepted Domain** indica a Exchange come trattare un dominio. Per un dominio autoritativo, Exchange si aspetta tutti i destinatari validi nella propria directory. Per un dominio Internal Relay, i destinatari sconosciuti possono essere inoltrati a un altro sistema. Questa impostazione non è quindi una riga di inventario, ma parte della decisione di recapito ([Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains), [Accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains)).

I **connettori** determinano quindi da quali sistemi Exchange accetta messaggi e a quali sistemi li invia. In Exchange Server, i connettori di ricezione e invio operano con associazioni locali, spazi indirizzi, server di origine, smart host e autorizzazioni. Exchange Online utilizza connettori in ingresso e in uscita per relazioni con la propria infrastruttura o con partner ([Connectors on Exchange servers](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors), [Set up connectors to route mail](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail)).

Solo a questo punto diventano importanti le varianti ibride. Centralized Mail Transport, un gateway a monte o un dominio destinatari condiviso modificano hop aggiuntivi e responsabilità. Per questo sono trattati nell'articolo dedicato [Flusso di posta ibrido](/kb/hybrid-mailfluss), non tra gli argomenti relativi a identità o client.

## Dall'accesso alla casella di posta

Il percorso del messaggio non spiega ancora come Outlook trovi la propria casella di posta. A questo scopo Exchange utilizza **Autodiscover**. Un client parte dall'identità dell'utente e determina il punto di accesso al servizio appropriato. In locale possono essere coinvolti i Service Connection Point di Active Directory e il DNS; in Exchange Online gli endpoint di Microsoft 365 conducono al servizio cloud ([Microsoft Learn: Autodiscover](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)).

Dopo la determinazione dell'endpoint, l'accesso moderno di Outlook avviene tramite HTTPS, in particolare con MAPI over HTTP. Anche Outlook sul Web, Exchange ActiveSync e varie API utilizzano HTTPS, ma ciascuno con protocolli applicativi e autorizzazioni propri. Un accesso riuscito al portale Microsoft 365 non dimostra quindi automaticamente che Autodiscover, il protocollo Outlook o l'accesso alla casella di posta specifica funzionino ([Client Access protocol architecture](https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access), [MAPI over HTTP](https://learn.microsoft.com/en-us/exchange/clients/mapi-over-http/mapi-over-http)).

Nella modalità ibrida si aggiunge un'ulteriore decisione: la casella di posta è locale o online? Autodiscover e gli attributi destinatario devono guidare il client verso il lato corretto. Solo in seguito entrano in gioco le funzioni tra organizzazioni, come libero/occupato o gli spostamenti delle caselle di posta. Questa sequenza viene trattata passo dopo passo nell'articolo [Exchange Hybrid](/kb/exchange-hybrid).

## Amministrazione e autorizzazioni

Una volta compresi i percorsi dei dati e di accesso, si pone la domanda su chi può modificarli. Exchange utilizza il controllo degli accessi basato sui ruoli. I ruoli contengono cmdlet e parametri, i gruppi di ruoli o le assegnazioni di ruolo li collegano agli amministratori e gli scope ne limitano l'ambito. Exchange Online ed Exchange Server possiedono concetti RBAC correlati, ma configurazioni separate ([Permissions in Exchange Server](https://learn.microsoft.com/en-us/exchange/permissions/permissions), [Permissions in Exchange Online](https://learn.microsoft.com/en-us/exchange/permissions-exo/permissions-exo)).

Nella pratica quotidiana ciò significa: un ruolo di amministratore Entra, un gruppo di ruoli Exchange e un'autorizzazione per una casella di posta non sono la stessa cosa. **Full Access** consente di aprire una casella di posta, **Send As** di inviare come destinatario e **Send on Behalf** di inviare in modo riconoscibile per conto di un altro soggetto. Nessuna di queste autorizzazioni spiega da sola se un'applicazione può accedere tramite Microsoft Graph o EWS ([Manage permissions for recipients in Exchange Online](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/manage-permissions-for-recipients)).

La domanda da esperti non è quindi «L'utente è amministratore?», bensì: quale identità effettua l'accesso, quale ruolo si applica in quale organizzazione Exchange, quale oggetto viene indirizzato e quale ulteriore autorizzazione per casella di posta o applicazione viene verificata?

## Gestione operativa e risoluzione dei problemi iniziano dal lato corretto

Una diagnosi sensata inizia con tre informazioni: **utente o destinatario interessato, momento esatto e posizione della casella di posta**. Successivamente, il percorso viene seguito nell'ordine DNS o Autodiscover, accesso, endpoint Exchange, risoluzione del destinatario, evento di trasporto e recapito nella casella di posta.

Per Exchange Online, Message Trace e Microsoft 365 Service Health forniscono lo stato del servizio visibile al cliente. Per Exchange Server si aggiungono code locali, Message Tracking Logs, Health Sets, Event Logs e copie di database. Nella modalità ibrida, le evidenze di entrambi i lati vengono riunite su una linea temporale comune ([Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq), [Server health and performance](https://learn.microsoft.com/en-us/exchange/server-health/server-health)).

Gli articoli di approfondimento contengono ciascuno blocchi di diagnostica Windows/Unix adatti e la documentazione ufficiale degli strumenti utilizzati. Qui è sufficiente la regola operativa più importante: prima determinare il luogo e il percorso, poi scegliere lo strumento.

## Archiviazione dei dati, disponibilità e ripristino

La differenza tra cloud e gestione in proprio risulta più evidente nel recovery. Exchange Server memorizza le caselle di posta in database ESE con log delle transazioni. I Database Availability Groups replicano le copie dei database e consentono l'attivazione su altri server. L'organizzazione rimane comunque responsabile della strategia di backup, della possibilità di ripristino e della dipendenza da Active Directory ([Database availability groups](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups), [Backup, restore and disaster recovery](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery)).

Exchange Online gestisce la ridondanza dei database e del trasporto come parte del servizio. Gli amministratori del tenant lavorano invece con elementi eliminati, Single Item Recovery, conservazione, hold e, se necessario, requisiti di backup esterni. La resilienza del servizio Microsoft e una regola di conservazione funzionale rispondono a domande diverse: la prima protegge il servizio in esecuzione, l'altra stabilisce quali contenuti devono essere mantenuti dopo l'eliminazione o per finalità di conformità ([Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency), [Retention policies for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange)).

Nella modalità ibrida entrambi i modelli di recovery devono essere documentati parallelamente. Inoltre, sono necessari sincronizzazione delle directory, certificati, configurazione OAuth e connettori per ristabilire la connessione dopo un guasto. Queste configurazioni non contengono contenuti delle caselle di posta, ma determinano se le due organizzazioni Exchange possono tornare a collaborare.

## Evoluzione tecnica

Exchange 4.0 apparve nel 1996 come successore delle precedenti piattaforme di posta Microsoft. Le prime versioni utilizzavano una propria directory, MAPI e la famiglia di database ESE; i protocolli Internet acquisirono gradualmente maggiore importanza. Con Exchange 2000, Active Directory e SMTP divennero componenti centrali. Exchange 2007 introdusse ruoli server distinti e Exchange Management Shell, Exchange 2010 il Database Availability Group ([Exchange Team: A brief history of time](https://techcommunity.microsoft.com/blog/exchange/a-brief-history-of-time---exchange-server-way/589388), [Exchange Team: Exchange Server 2007 transport redesign](https://techcommunity.microsoft.com/blog/exchange/motivations-for-the-exchange-server-2007-transport-redesign/600465)).

Parallelamente, Microsoft sviluppò le offerte Exchange ospitate fino a Exchange Online. Di conseguenza, Hybrid non nacque come singolo prodotto, ma come collegamento tra due organizzazioni Exchange indipendenti. A partire dal 2025, Exchange Server Subscription Edition prosegue questa linea di sviluppo locale nel Modern Lifecycle. Le informazioni concrete su build e aggiornamenti vengono verificate prima delle modifiche nella documentazione Microsoft costantemente aggiornata ([Exchange Server SE release notes](https://learn.microsoft.com/en-us/exchange/release-notes), [Exchange Server Subscription Edition lifecycle](https://learn.microsoft.com/en-us/lifecycle/products/exchange-server-subscription-edition)).

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
