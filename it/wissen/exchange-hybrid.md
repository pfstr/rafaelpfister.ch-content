---
title: "Exchange Hybrid: identità, coesistenza e gestione"
blatt: "exchange-hybrid"
description: "Exchange Hybrid spiegato in modo chiaro: prerequisiti, sincronizzazione delle directory, autorità sui destinatari, Hybrid Configuration Wizard, OAuth, relazioni tra organizzazioni, Autodiscover, spostamenti delle cassette postali e gestione."
fakten:
  - label: Scopo
    wert: Coesistenza di Exchange Server ed Exchange Online
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: Spazio dei nomi condiviso
    wert: Le cassette postali su entrambi i lati possono utilizzare gli stessi domini SMTP
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: Sincronizzazione directory
    wert: Microsoft Entra Connect Sync o Cloud Sync secondo il modello supportato
    href: https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites
  - label: Strumento di configurazione
    wert: Hybrid Configuration Wizard
    href: https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard
  - label: Configurazione locale
    wert: Oggetto HybridConfiguration in Active Directory
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: Configurazione cloud
    wert: Connettori, relazioni tra organizzazioni e attendibilità OAuth
    href: https://learn.microsoft.com/en-us/exchange/hybrid-deployment/deploy-hybrid
  - label: Modello dei destinatari
    wert: Remote Mailbox locale, cassetta postale di Exchange Online nel cloud
    href: https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox
  - label: Funzioni di coesistenza
    wert: Disponibilità, MailTips, archivio, ricerca e spostamenti delle cassette postali a seconda della configurazione
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: Migrazione delle cassette postali
    wert: Mailbox Replication Service ed endpoint di migrazione
    href: https://learn.microsoft.com/en-us/exchange/mailbox-migration/migrate-mailboxes-across-tenants
  - label: Trasporto della posta
    wert: SMTP/TLS basato su certificati tra le due organizzazioni
    href: https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites
  - label: Gestione dei destinatari
    wert: Exchange Management Tools o trasferimento Cloud SoA supportato
    href: https://learn.microsoft.com/en-us/exchange/manage-hybrid-exchange-recipients-with-management-tools
  - label: Diagnostica
    wert: Log HCW, stato della sincronizzazione Entra, configurazione OAuth, organizzativa e del trasporto
    href: https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard
werbung:
  - tools
  - newsletter
ctaThemen:
  - exchange-onprem-hybrid
  - microsoft-365-exchange
translationSourceHash: c35b1646509133dd8975f96d030b2990225af14469d14a61e4fd16a4e2737f08
translationModel: gpt-5.6-terra
translatedAt: 2026-10-05T11:19:28.181Z
translationReview: automatic
---

# Exchange Hybrid: identità, coesistenza e gestione

**Exchange Hybrid** collega un'organizzazione Exchange locale con Exchange Online. Gli utenti possono avere cassette postali su entrambi i lati e utilizzare comunque gli stessi domini SMTP, una rubrica condivisa e funzioni selezionate tra organizzazioni. Hybrid è quindi più di una coppia di connettori: collega dati di directory, destinatari, autenticazione, Autodiscover, funzioni di calendario, migrazione e trasporto della posta ([Distribuzioni ibride di Exchange](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)).

Chiarite innanzitutto tre domande: **Dove si trova la cassetta postale?** **Dove viene gestito il relativo oggetto destinatario?** E **quale servizio esegue l'operazione richiesta?** Quando queste tre risposte sono definite, i numerosi componenti Hybrid diventano una catena comprensibile.

## Cosa Hybrid unifica per gli utenti

Senza Hybrid, l'organizzazione Exchange locale ed Exchange Online sono due sistemi separati. Hybrid crea un'esperienza utente condivisa. Le cassette postali possono usare lo stesso dominio SMTP primario. Le informazioni della rubrica vengono sincronizzate. Le richieste di disponibilità e i MailTips possono funzionare tra organizzazioni. Le cassette postali possono essere spostate con Remote Move supportati ([Distribuzioni ibride di Exchange](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)).

Queste funzioni non condividono tuttavia un unico archivio dati comune. Una cassetta postale locale rimane in un database ESE locale; una cassetta postale cloud rimane in Exchange Online. Active Directory ed Entra ID mantengono ciascuno oggetti di directory. Le relazioni tra organizzazioni e OAuth consentono richieste selezionate oltre il confine. I connettori SMTP trasportano i messaggi. L'apparente «unico Exchange» nasce da collegamenti coordinati.

Per gli amministratori ne deriva una regola importante: un mail flow funzionante non dimostra che la disponibilità funzioni, e una richiesta di disponibilità riuscita non dimostra che sia possibile un Remote Move. Ogni funzione ha il proprio percorso e le proprie verifiche.

## I componenti nell'ordine corretto

Una distribuzione Hybrid inizia dai prerequisiti, non dal wizard. L'organizzazione Exchange locale deve essere a un livello supportato. Nomi pubblici, certificati, DNS, raggiungibilità HTTPS e SMTP devono essere corretti. Un tenant Microsoft 365 con Exchange Online e una sincronizzazione delle directory supportata collegano poi le identità ([Prerequisiti per la distribuzione ibrida](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)).

Su questa base opera **Hybrid Configuration Wizard**, HCW. Legge la configurazione desiderata, scrive un oggetto `HybridConfiguration` nell'Active Directory locale e configura impostazioni appropriate in locale e in Exchange Online. Possono includere relazioni tra organizzazioni, OAuth, Intra-Organization Connectors e connettori di trasporto ([Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard), [Creare una distribuzione ibrida](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/deploy-hybrid)).

| Componente | Compito principale | Cosa verificare per primo in caso di guasto |
|---|---|---|
| Active Directory | Attributi locali di utenti ed Exchange | Oggetto, tipo di destinatario, indirizzi proxy e ora della modifica |
| Sincronizzazione Entra | Trasferisce gli attributi di identità e destinatari supportati | Errori di esportazione, stato della sincronizzazione e oggetto cloud |
| Exchange Online | Cassetta postale cloud e configurazione cloud | Tipo di destinatario, licenza, stato della cassetta postale e RBAC |
| Configurazione HCW | Coordina le due organizzazioni Exchange | Log HCW, parametri selezionati e oggetti modificati successivamente |
| Relazione organizzativa e OAuth | Funzioni tra organizzazioni | URI di destinazione, Autodiscover, certificati e flusso di token |
| Connettori SMTP | Messaggi tra i due lati | Nome del certificato, host origine/destinazione, TLS e Message Trace |

La tabella mostra anche perché «eseguire nuovamente HCW» non è una riparazione universale. Il wizard può riallineare gli oggetti Hybrid documentati. Non ripara una zona DNS errata, un percorso firewall bloccato o un oggetto destinatario gestito in modo errato.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-exchange-hybrid.svg?v=20260813" title="Interaktive Infografik: Exchange-Hybrid-Verbindungen für Verzeichnissync, Empfänger, HCW, OAuth, Frei-Gebucht, Mailboxverschiebung und SMTP" loading="lazy">
  <a href="/images/kb-interaktiv-exchange-hybrid.svg?v=20260813">Apri direttamente la grafica interattiva di Exchange Hybrid</a>.
</iframe>

## Stack tecnologico: protocolli e strumenti di gestione

Hybrid non è un processo server Exchange aggiuntivo, bensì un collegamento tra sistemi esistenti. Active Directory ed Entra ID mantengono identità e attributi dei destinatari. La sincronizzazione Entra trasferisce i valori supportati. HTTPS trasporta Autodiscover, disponibilità, chiamate di servizio protette da OAuth e spostamenti delle cassette postali. SMTP con TLS trasporta i messaggi. PowerShell, Exchange Admin Center e HCW gestiscono gli oggetti coinvolti ([Prerequisiti per la distribuzione ibrida](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites), [Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)).

Questa suddivisione determina anche l'ordine per la risoluzione dei problemi. Un errore dell'oggetto viene cercato nella directory e nella sincronizzazione, un problema del calendario nel percorso HTTPS/OAuth, un problema di posta in SMTP e nei connettori. In questo modo gli strumenti restano legati alla funzione interessata.

## Sincronizzazione delle directory e autorità sui destinatari

Non appena le piattaforme sono collegate, l'origine dei dati dei destinatari diventa la principale questione operativa. Negli ambienti Hybrid classici, un utente viene creato nell'Active Directory locale. Gli strumenti Exchange scrivono gli attributi relativi alla posta. Entra Connect sincronizza l'oggetto nel cloud, dove Exchange Online fornisce l'oggetto cloud corrispondente e, se necessario, una cassetta postale ([Prerequisiti per la distribuzione ibrida](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)).

Una **Remote Mailbox** è un oggetto locale abilitato alla posta che fa riferimento a una cassetta postale di Exchange Online. Attributi come `remoteRoutingAddress`, `proxyAddresses` e il tipo di destinatario aiutano l'organizzazione locale a indirizzare messaggi e gestione verso il lato cloud. [`Enable-RemoteMailbox`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox) crea o abilita questa rappresentazione locale; la cassetta postale cloud viene creata solo tramite sincronizzazione e assegnazione della licenza.

La normale domanda amministrativa è quindi: dove devo modificare questo valore? La domanda degli esperti è: quale sistema è autorevole per **questo singolo attributo** e quale ciclo di sincronizzazione lo trasferisce? Un portale cloud può visualizzare un valore sincronizzato senza poterlo modificare in modo permanente.

Microsoft supporta scenari in cui rimangono solo gli Exchange Management Tools per gli attributi locali dei destinatari. Per determinati ambienti esiste inoltre una procedura per trasferire nel cloud la gestione degli attributi Exchange. Si tratta di modelli operativi diversi con prerequisiti propri; spegnere l'ultimo server da solo non trasferisce l'autorità sui dati ([Gestire i destinatari con Exchange Management Tools](https://learn.microsoft.com/en-us/exchange/manage-hybrid-exchange-recipients-with-management-tools), [Disattivazione dopo il trasferimento dell'autorità di origine](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/decommission-last-exchange-server)).

## Autodiscover e il percorso del client

Se i destinatari sono corretti, un client deve trovare la posizione della cassetta postale. Autodiscover risponde a questa domanda. Gli endpoint Exchange locali possono reindirizzare un client per una cassetta postale cloud a Exchange Online; gli endpoint cloud forniscono le impostazioni per la cassetta postale online ([Autodiscover nelle distribuzioni ibride di Exchange](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)).

Un problema di Autodiscover Hybrid si manifesta quindi spesso come posizione errata: l'utente può in linea di principio accedere, ma arriva all'endpoint locale, riceve un reindirizzamento inatteso oppure ottiene impostazioni per una cassetta postale che non esiste più. DNS, SCP, directory virtuali, certificati e attributi dei destinatari vengono verificati in questo ordine.

Solo dopo che il client ha raggiunto il corretto servizio di cassetta postale hanno senso le questioni relative a protocollo e autorizzazioni. In questo modo la diagnosi resta chiara: prima trovare, poi autenticarsi, quindi autorizzare.

## Disponibilità e altre funzioni tra organizzazioni

Una rubrica condivisa non basta per le richieste di calendario. La disponibilità richiede relazioni tra organizzazioni, Autodiscover raggiungibile e una configurazione di attendibilità o OAuth funzionante. Exchange richiede le informazioni sul lato opposto invece di copiare completamente i dati del calendario nel proprio sistema ([Condivisione nelle distribuzioni ibride di Exchange](https://learn.microsoft.com/en-us/exchange/sharing/sharing)).

Lo stesso schema di base vale per altre funzioni Hybrid: un componente locale invia una richiesta, la controparte la autentica, autorizza l'operazione e restituisce un risultato limitato. Nella ricerca degli errori vengono quindi annotati cassetta postale di origine, cassetta postale di destinazione, direzione ed endpoint. «La disponibilità non funziona» è troppo generico senza queste indicazioni.

Per gli esperti diventano rilevanti token e URI di destinazione. HCW configura relazioni tra organizzazioni, ma cambiamenti dei certificati, modifiche manuali o endpoint obsoleti possono compromettere il funzionamento successivo. La configurazione di entrambi i lati viene sempre esportata insieme.

## OAuth tra le organizzazioni Exchange

Una volta chiarito quali richieste tra organizzazioni vengono effettuate, è possibile inquadrare la relativa autenticazione. Exchange può utilizzare OAuth affinché un'organizzazione dimostri all'altra una chiamata di servizio. Ciò riguarda funzioni Hybrid quali disponibilità tra organizzazioni e operazioni selezionate di archiviazione, ricerca o migrazione; l'utilizzo esatto dipende dalla versione e dalla configurazione ([Configurare l'autenticazione OAuth](https://learn.microsoft.com/en-us/exchange/configure-oauth-authentication-between-exchange-and-exchange-online-organizations-exchange-2013-help)).

Il flusso di token non sostituisce SMTP-TLS. OAuth protegge le chiamate applicative, mentre il trasporto di posta Hybrid utilizza connettori e verifiche dei certificati propri. Questa separazione evita il salto che crea confusione in molte spiegazioni: prima si determina la funzione, poi il suo protocollo e solo dopo l'autenticazione.

Per gli esperti, AuthConfig, AuthServer, PartnerApplication, Intra-Organization Connector e Organization Relationship fanno parte di un quadro di verifica comune. Un singolo oggetto può essere presente sintatticamente mentre certificato, realm o URI di destinazione non corrispondono più alla controparte.

## Hybrid Modern Authentication è un tema client distinto

**Hybrid Modern Authentication**, HMA, viene affrontata solo ora perché non spiega né il trasporto della posta né la sincronizzazione dei destinatari. HMA consente alle risorse locali supportate di Exchange e Skype for Business di utilizzare Microsoft Entra ID per l'autenticazione moderna dei client. Il client ottiene un token Entra e lo utilizza verso il servizio locale ([Panoramica dell'autenticazione moderna ibrida](https://learn.microsoft.com/en-us/microsoft-365/enterprise/hybrid-modern-auth-overview)).

HMA estende quindi l'accesso dei client con una dipendenza dal cloud. Raggiungibilità di Entra, URL pubblicati, Service Principal Names registrati e configurazione Exchange locale devono corrispondere. Un mail flow Hybrid funzionante non dice nulla su questo percorso di token.

Gli esperti trattano quindi HMA in un runbook separato con versioni supportate, esclusioni, gruppi di distribuzione e piano di ripristino. La funzione non viene aggiunta incidentalmente a un'opzione di routing.

## Spostamenti delle cassette postali

La coesistenza viene spesso configurata per spostare gradualmente le cassette postali. Un Remote Move copia i dati della cassetta postale tramite Mailbox Replication Service, mantiene sincronizzate le modifiche e trasferisce in modo controllato la cassetta postale al lato di destinazione. Gli attributi dei destinatari e il routing vengono mantenuti o adattati ([Spostare cassette postali tra ambiente locale ed Exchange Online](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/move-mailboxes)).

Per gli amministratori avanzati, il processo consiste in preparazione, avvio, sincronizzazione, completamento e verifica successiva. Prima del completamento vengono controllati volume dei dati, elementi difettosi, deleghe, archivi, accesso client e mail flow. Dopo il completamento, Autodiscover, licenza, indirizzo di destinazione e oggetto Remote Mailbox locale devono corrispondere.

Gli esperti pianificano dimensioni dei batch, throughput di rete, limitazione MRS, Bad Item Limits, oggetti di grandi dimensioni e migrazione di ritorno. Il solo valore tecnico di avanzamento non costituisce un collaudo: ne fanno parte accesso utente, deleghe, client mobili e funzioni tra organizzazioni.

## Il flusso di posta Hybrid rimane un percorso separato

Hybrid richiede SMTP tra l'organizzazione locale ed Exchange Online. Questo percorso della posta utilizza connettori, TLS e certificati. È sufficientemente importante da meritare un articolo dedicato, poiché la posta Internet, Centralized Mail Transport, gateway di posta e domini condivisi formano diverse varianti ([Prerequisiti per la distribuzione ibrida](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)).

L'articolo [Flusso di posta Hybrid](/kb/hybrid-mailfluss) parte da un messaggio concreto e segue ogni hop. Solo lì vengono confrontati Centralized Mail Transport, IP di uscita, posizione del filtro e code aggiuntive. Questo articolo si concentra su identità e coesistenza.

## Sicurezza e gestione

Hybrid amplia i sistemi raggiungibili. Endpoint HTTPS e SMTP pubblici, certificati, sincronizzazione Entra, account privilegiati e oggetti di attendibilità tra organizzazioni devono essere inventariati congiuntamente. HCW richiede autorizzazioni estese su entrambi i lati; il suo utilizzo e i relativi log devono essere protetti e archiviati in modo tracciabile ([Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)).

Nella pratica quotidiana, ogni funzione Hybrid dovrebbe avere un responsabile e un test: sincronizzazione dei destinatari, disponibilità in entrambe le direzioni, Remote Move, Autodiscover e SMTP in entrambe le direzioni. Un test sintetico regolare individua certificati scaduti o endpoint modificati silenziosamente prima di un progetto di migrazione.

In caso di guasti, è utile una cronologia comune. Eventi di sincronizzazione Entra, log HCW, registri eventi Exchange, test OAuth, Message Tracking e Message Trace non vengono raccolti casualmente, bensì assegnati alla funzione interessata. Ciò abbrevia la diagnosi ed evita che un test riuscito di un'altra funzione venga frainteso come prova.

## Backup, ricostruzione e dismissione

I dati delle cassette postali vengono protetti sul lato in cui risiedono: database locali con ripristino locale, cassette postali cloud con funzioni di Exchange Online e Purview. Inoltre, il collegamento deve essere ripristinabile. Ne fanno parte l'oggetto HybridConfiguration locale, certificati e chiavi private, configurazione dei connettori e dell'organizzazione, regole di sincronizzazione Entra e decisioni HCW documentate.

Una ricostruzione inizia con identità e risoluzione dei nomi, seguite da raggiungibilità HTTPS e SMTP, configurazione HCW e test delle funzioni. Il wizard può rigenerare la configurazione, ma senza certificati, DNS e oggetti destinatario appropriati non si crea un sistema complessivo funzionante.

Durante la dismissione viene prima chiarito quali funzioni Hybrid sono ancora utilizzate. Microsoft distingue tra un server rimanente, i soli Management Tools e il trasferimento nel cloud della gestione degli attributi Exchange. Solo dopo questa decisione vengono rimossi in modo controllato connettori, relazioni tra organizzazioni, endpoint e server ([Gestire i destinatari con Exchange Management Tools](https://learn.microsoft.com/en-us/exchange/manage-hybrid-exchange-recipients-with-management-tools), [Disattivazione dopo il trasferimento dell'autorità di origine](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/decommission-last-exchange-server)).

## Evoluzione tecnica e limiti

Hybrid è nato con Exchange Online come modo per estendere in modo controllato le organizzazioni locali al servizio cloud. Le generazioni precedenti utilizzavano maggiormente Federation Trust; le versioni Exchange e i flussi HCW più recenti usano OAuth e Intra-Organization Connectors per molte funzioni tra organizzazioni ([Creare una distribuzione ibrida](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/deploy-hybrid)).

Il modello è potente perché consente migrazione e coesistenza permanente. È impegnativo perché devono essere gestite entrambe le organizzazioni Exchange e il relativo collegamento. Chi non dispone più di cassette postali locali dopo una migrazione dovrebbe quindi decidere consapevolmente quale funzione di gestione o coesistenza giustifichi ancora Hybrid.

## Fonti

- [Microsoft Learn – Distribuzioni ibride di Exchange](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)
- [Microsoft Learn – Active Directory in Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)
- [Microsoft Learn – Destinatari in Exchange Online](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/recipients-in-exchange-online)
- [Microsoft Learn – Prerequisiti per la distribuzione ibrida](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)
- [Microsoft Learn – Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)
- [Microsoft Learn – Creare una distribuzione ibrida](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/deploy-hybrid)
- [Microsoft Learn – Enable-RemoteMailbox](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox)
- [Microsoft Learn – Gestire i destinatari con Exchange Management Tools](https://learn.microsoft.com/en-us/exchange/manage-hybrid-exchange-recipients-with-management-tools)
- [Microsoft Learn – Disattivazione dopo il trasferimento dell'autorità di origine](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/decommission-last-exchange-server)
- [Microsoft Learn – Servizio Autodiscover](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)
- [Microsoft Learn – Condivisione in Exchange](https://learn.microsoft.com/en-us/exchange/sharing/sharing)
- [Microsoft Learn – Configurare l'autenticazione OAuth](https://learn.microsoft.com/en-us/exchange/configure-oauth-authentication-between-exchange-and-exchange-online-organizations-exchange-2013-help)
- [Microsoft Learn – Panoramica dell'autenticazione moderna ibrida](https://learn.microsoft.com/en-us/microsoft-365/enterprise/hybrid-modern-auth-overview)
- [Microsoft Learn – Spostare cassette postali tra ambiente locale ed Exchange Online](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/move-mailboxes)
- [Microsoft Learn – Migrazione delle cassette postali tra tenant](https://learn.microsoft.com/en-us/exchange/mailbox-migration/migrate-mailboxes-across-tenants)
