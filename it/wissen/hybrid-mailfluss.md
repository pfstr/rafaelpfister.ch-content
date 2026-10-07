---
title: "Flusso di posta ibrido: routing tra Exchange Online e ambiente locale"
blatt: "hybrid-mailfluss"
description: "Flusso di posta ibrido passo dopo passo: recapito diretto e centralizzato, connettori, certificati TLS, domini remoti, domini SMTP condivisi, gateway di posta, Message Trace e Message Tracking."
fakten:
  - label: Funzione
    wert: Routing SMTP tra Exchange Online e organizzazione Exchange locale
    href: https://learn.microsoft.com/en-us/exchange/transport-routing
  - label: Configurazione
    wert: Hybrid Configuration Wizard crea e gestisce la configurazione di trasporto
    href: https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard
  - label: Trasporto
    wert: SMTP su TCP 25 con TLS
    href: https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites
  - label: Verifica della controparte
    wert: Nome del certificato e condizioni del connettore
    href: https://learn.microsoft.com/en-us/exchange/hybrid-deployment/hybrid-mail-flow
  - label: Connettori cloud
    wert: Connettori Inbound e Outbound
    href: https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail
  - label: Connettori locali
    wert: Connettori Receive e Send
    href: https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors
  - label: Routing del destinatario
    wert: Remote Mailbox, Target Address e dominio di coesistenza
    href: https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox
  - label: Routing standard
    wert: L'organizzazione cloud e quella locale possono inviare direttamente la posta Internet
    href: https://learn.microsoft.com/en-us/exchange/transport-routing
  - label: Centralized Mail Transport
    wert: La posta Internet di Exchange Online passa attraverso l'organizzazione locale
    href: https://learn.microsoft.com/en-us/exchange/transport-routing
  - label: Gateway esterni
    wert: Catena aggiuntiva di connettori e filtri davanti o dietro Microsoft 365
    href: https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud
  - label: Diagnostica cloud
    wert: Message Trace e convalida dei connettori
    href: https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message
  - label: Diagnostica locale
    wert: Message Tracking, code e registri di protocollo
    href: https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking
werbung:
  - tools
  - newsletter
ctaThemen:
  - exchange-onprem-hybrid
  - smtp-mailflow
translationSourceHash: 442ec532c64e034403d3449ffe1b46d4b6de05a2337d3c3210da2d83cb5f618f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-07T11:03:20.306Z
translationReview: automatic
---

# Flusso di posta ibrido: routing tra Exchange Online e ambiente locale

Il **flusso di posta ibrido** è il percorso SMTP tra un'organizzazione Exchange locale e Exchange Online. Consente alle cassette postali su entrambi i lati di utilizzare lo stesso dominio SMTP e garantisce comunque che i messaggi arrivino nel posto corretto. Hybrid Configuration Wizard configura a questo scopo connettori e parametri TLS ([Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing), [Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)).

Il flusso di posta è solo una parte di Exchange Hybrid. La sincronizzazione delle directory, libero/occupato, OAuth e gli spostamenti delle cassette postali utilizzano altri percorsi. Questo articolo si concentra quindi deliberatamente su una sola domanda: **quali hop SMTP attraversa un messaggio concreto e quale decisione viene presa a ogni hop?**

Lo **stack di protocolli** è chiaro: DNS indica le destinazioni raggiungibili pubblicamente, SMTP su TCP 25 trasporta il messaggio, TLS protegge e identifica la connessione e i connettori Exchange stabiliscono quale controparte viene utilizzata per quale dominio. Gli oggetti destinatario forniscono l'indirizzo di routing; Message Trace e i registri di tracking locali mostrano successivamente che cosa ogni organizzazione ha fatto con il messaggio ([Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing), [Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)).

## Il modello di base: due organizzazioni Exchange, uno spazio di indirizzamento

Un ambiente ibrido dispone di almeno due organizzazioni di trasporto. L'organizzazione Exchange locale conosce le cassette postali locali e gli oggetti Remote Mailbox. Exchange Online conosce le cassette postali cloud e le rappresentazioni sincronizzate dei destinatari locali. Entrambi i lati possono utilizzare lo stesso dominio primario, ad esempio `example.com` ([Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)).

Affinché un messaggio non termini nel posto sbagliato, ogni lato necessita di un'indicazione della posizione effettiva del destinatario. Per una cassetta postale cloud, l'oggetto Remote Mailbox locale contiene un indirizzo di routing remoto nel dominio di coesistenza, tipicamente `tenant.mail.onmicrosoft.com`. Viceversa, Exchange Online conosce i destinatari locali sincronizzati come oggetti abilitati alla posta ([Enable-RemoteMailbox](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox)).

Il flusso normale è quindi semplice:

1. La prima organizzazione Exchange accetta il messaggio.
2. Risolve il destinatario nella propria directory.
3. L'oggetto destinatario indica se la cassetta postale è locale o si trova sull'altro lato.
4. Il connettore ibrido appropriato invia tramite SMTP/TLS all'altra organizzazione.
5. Lì il destinatario viene nuovamente risolto e il messaggio viene recapitato.

Gli esperti verificano inoltre se le regole di trasporto modificano il percorso, se è inserito un gateway e quale priorità di dominio o connettore spiega il next hop selezionato.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-hybrid-mailfluss.svg?v=20260813" title="Interaktive Infografik: direkter und zentraler Hybrid-Mailfluss zwischen Internet, Exchange Online, Exchange On-Premises und Mail-Gateway" loading="lazy">
  <a href="/images/kb-interaktiv-hybrid-mailfluss.svg?v=20260813">Aprire direttamente il grafico interattivo del flusso di posta ibrido</a>.
</iframe>

## Il percorso standard senza trasporto Internet centralizzato

Nel consueto modello decentralizzato, ogni lato invia autonomamente la propria posta Internet. Una cassetta postale locale utilizza l'organizzazione di trasporto Exchange locale per la posta Internet in uscita. Una cassetta postale cloud invia tramite Exchange Online Protection. Solo i messaggi tra cassette postali locali e cloud attraversano i connettori ibridi ([Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing)).

La posta Internet in entrata segue l'MX pubblicato. Se l'MX punta a Exchange Online, EOP accetta prima il messaggio. Per una cassetta postale cloud, Exchange Online recapita localmente; per un destinatario locale sincronizzato, utilizza il connettore Outbound ibrido. Se invece l'MX punta all'ambiente locale o a un gateway a monte, la prima decisione sul destinatario avviene lì.

Questo modello mantiene brevi i percorsi Internet, ma comporta più possibili IP di uscita e punti di filtro. SPF, DKIM, DMARC, allowlisting e regole dei partner devono considerare entrambi i percorsi di uscita. Non è un errore del modello ibrido, bensì una conseguenza del recapito distribuito.

## Inquadrare consapevolmente Centralized Mail Transport

**Centralized Mail Transport**, CMT, modifica esattamente questo percorso in uscita. I messaggi dalle cassette postali di Exchange Online verso Internet vengono prima inviati all'organizzazione Exchange locale. Solo lì lasciano l'organizzazione. In questo modo il lato locale può continuare a utilizzare regole di trasporto centrali, appliance o IP di uscita fissi ([Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing)).

Il vantaggio è un controllo comune delle comunicazioni in uscita. Il prezzo consiste in hop e dipendenze aggiuntivi. Se il trasporto locale o la relativa connessione Internet non sono disponibili, ciò influisce ora anche sulla posta cloud in uscita. Latenza, posizione della coda, IP di uscita e punto dell'ultimo filtraggio cambiano.

Per gli amministratori avanzati, la decisione non è quindi «CMT attivato o disattivato», ma: quale criterio concreto richiede l'hop locale, quale capacità deve sostenere e come viene effettuato il routing in caso di guasto? Gli esperti documentano inoltre la protezione dai loop, i requisiti TLS, la priorità del connettore e la prova che ogni messaggio previsto segue realmente il percorso centrale.

Con questo il tema CMT è concluso. L'autenticazione dei client Outlook o Hybrid Modern Authentication non rientra qui, poiché non seleziona alcun hop SMTP.

## Come i connettori riconoscono la controparte

Dopo aver scelto il percorso, ogni lato deve poter considerare attendibile la controparte. Hybrid Configuration Wizard crea connettori Send/Receive locali e i corrispondenti connettori Inbound/Outbound in Exchange Online. Il trasporto avviene tramite SMTP su TCP 25 e utilizza TLS ([Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites), [Hybrid mail flow](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/hybrid-mail-flow)).

Il connettore Send locale determina la destinazione e i requisiti TLS per il dominio di coesistenza. Il connettore Outbound cloud descrive l'organizzazione locale come destinazione. Nella direzione opposta, il connettore Receive o Inbound accetta il traffico in base a condizioni documentate, tra cui l'identità del certificato e l'origine.

Il certificato svolge un compito concreto: identifica l'endpoint SMTP nell'handshake TLS. Subject o Subject Alternative Name, parametri del connettore, catena di certificati presentata e nome host effettivo devono corrispondere. Un certificato valido presente nell'archivio certificati non è sufficiente se il servizio di trasporto ne presenta un altro.

Gli esperti verificano quindi separatamente entrambe le direzioni. La direzione A può funzionare mentre la direzione B fallisce a causa di un altro connettore, una diversa destinazione DNS o un differente nome nel certificato.

## Routing del destinatario e domini condivisi

Un connettore funzionante non indica ancora quali messaggi lo utilizzano. Questa decisione inizia con l'oggetto destinatario. Una Remote Mailbox locale punta al cloud. Un oggetto cassetta postale locale sincronizzato in Exchange Online punta di nuovo all'organizzazione locale.

Gli Accepted Domains determinano inoltre se un'organizzazione è responsabile di tutti i destinatari di un dominio o se può inoltrare destinatari sconosciuti. Nei domini condivisi, una configurazione Internal Relay è sicura solo se il next hop conosce o rifiuta correttamente i destinatari sconosciuti ([Accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains), [Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)).

Un oggetto Remote Mailbox obsoleto può pertanto causare un routing errato nonostante TLS sia integro. Viceversa, un Target Address corretto non aiuta se il connettore cloud è disattivato. La diagnosi collega oggetto e trasporto, anziché considerare soltanto un lato.

Per gli esperti diventano importanti inoltri, contatti di posta, espansione dei gruppi di distribuzione e regole di trasporto. Possono modificare l'indirizzo destinatario originale o generare destinatari aggiuntivi. Ogni messaggio risultante riceve la propria decisione di routing.

## Gateway di posta davanti o dietro Exchange Online

Molte organizzazioni integrano l'ambiente ibrido con un Secure Email Gateway o una piattaforma di filtro cloud. Ciò aggiunge almeno un ulteriore hop SMTP. Il percorso deve essere tracciato separatamente per i messaggi in entrata e in uscita ([Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud)).

Se il gateway si trova davanti a Exchange Online, l'MX punta al gateway. EOP vede quindi inizialmente il suo IP di origine. Enhanced Filtering for Connectors può includere l'informazione originale del mittente nella valutazione del filtro Microsoft se il connettore e gli IP ignorati sono configurati correttamente ([Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)).

In uscita deve essere stabilito se Exchange Online invia direttamente, attraverso il gateway oppure, con CMT, prima in locale e poi attraverso il gateway. Più percorsi consentiti possono aggirare i criteri e generare firme DKIM, IP di uscita e protocolli diversi.

Il controllo degli esperti è un grafo dei percorsi consentiti: ogni freccia indica iniziatore, destinazione, porta, verifica TLS, domini consentiti, funzione di filtro, proprietario della coda e fonte del protocollo. Un nome di gateway senza queste informazioni non è ancora un'architettura.

## Tracciare un messaggio end-to-end

La ricerca dei guasti inizia con un messaggio di test di cui sono noti mittente, destinatario e orario. Prima si verifica il percorso pubblico, quindi gli eventi di ogni lato Exchange coinvolto.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für DNS- und SMTP-Test">
    <button type="button" role="tab" data-os-tab="windows" aria-selected="true">Windows</button>
    <button type="button" role="tab" data-os-tab="unix" aria-selected="false">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName -Type MX example.com
Test-NetConnection mail.example.com -Port 25
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig MX example.com
nc -vz mail.example.com 25
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) e [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) mostrano la destinazione MX pubblicata. [`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) e [`nc`](https://man.openbsd.org/nc) verificano se TCP 25 è raggiungibile dal punto di misurazione. Ciò non dimostra ancora che la verifica TLS o del connettore abbia avuto esito positivo.

Segue quindi l'handshake SMTP. OpenSSL può essere utilizzato su entrambe le piattaforme amministrative.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-STARTTLS-Test">
    <button type="button" role="tab" data-os-tab="windows" aria-selected="true">Windows</button>
    <button type="button" role="tab" data-os-tab="unix" aria-selected="false">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
openssl s_client -starttls smtp -connect mail.example.com:25 `
  -servername mail.example.com -showcerts
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
openssl s_client -starttls smtp -connect mail.example.com:25 \
  -servername mail.example.com -showcerts
```

  </div>
</div>

[`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) mostra la catena di certificati, i nomi e la negoziazione TLS. Per un dialogo di test completo e autorizzato è adatto [`swaks`](https://jetmore.org/john/code/swaks/). I messaggi di produzione non vengono testati con mittenti inventati; l'identità di test e il percorso previsto vengono definiti in anticipo.

In Exchange Online, [`Get-MessageTraceV2`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2) fornisce gli eventi cloud. In locale, [`Get-MessageTrackingLog`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog) mostra l'elaborazione sui server Exchange e [`Get-Queue`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue) mostra i next hop in attesa. I timestamp vengono riportati in un fuso orario comune; Internet Message ID e Network Message ID aiutano a collegare le sezioni ([Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq), [Message tracking](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking)).

## Errori tipici senza cambiare argomento

Un **errore TLS** viene inizialmente trattato come problema di trasporto: quale host è stato connesso, quale certificato ha presentato e quale condizione del connettore si aspettava la controparte? La sincronizzazione dei destinatari diventa rilevante solo quando il messaggio viene instradato in modo errato dopo un'accettazione riuscita.

Un **NDR per destinatario sconosciuto**, invece, conduce innanzitutto all'oggetto destinatario e al tipo di Accepted Domain. Solo se l'oggetto è corretto viene verificato se il percorso scelto lo trasporta al lato corretto.

Una **coda in attesa** richiede next hop, ora di nuovo tentativo e `LastError`. La porta aperta dell'host di destinazione aiuta solo come test successivo. Un **loop** si manifesta in intestazioni `Received` ripetute, hop o eventi di tracking e si verifica solitamente quando entrambi i lati inoltrano reciprocamente destinatari sconosciuti ([RFC 5321: Trace information and loop detection](https://datatracker.ietf.org/doc/html/rfc5321#section-6.3)).

Una **connessione funzionante in una sola direzione** non è una contraddizione. Le direzioni opposte utilizzano mittenti, connettori e verifiche dei certificati differenti. Vengono acquisite e testate separatamente.

## Sicurezza, esercizio e modifiche

L'SMTP ibrido apre un percorso di trasporto esplicitamente consentito. Questo percorso dovrebbe essere limitato ai sistemi di origine e destinazione, ai certificati e ai domini documentati. Open relay, intervalli IP troppo ampi o connettori che classificano ogni messaggio come attendibile sono in contrasto con questo modello.

Le sostituzioni dei certificati vengono pianificate come modifiche di routing. Prima della scadenza si verificano su entrambi i lati il nuovo certificato, l'assegnazione al servizio, la catena presentata e l'aspettativa del connettore. Seguono messaggi di test in entrambe le direzioni e un piano di rollback controllato.

Per l'esercizio continuo vengono monitorati almeno i connettori ibridi, la scadenza dei certificati, la crescita delle code, gli errori di Message Trace, le destinazioni DNS e lo stato del gateway. Con Centralized Mail Transport si aggiunge la capacità del percorso di uscita locale.

## Backup e ricostruzione del percorso di posta

Durante un'interruzione, i messaggi SMTP rimangono nelle code dei sistemi rispettivamente responsabili. Un backup della configurazione non ripristina questi messaggi in attesa. Configurazione e stato di trasporto vengono quindi considerati separatamente.

La ricostruzione comprende i parametri dei connettori di entrambi i lati, Accepted e Remote Domains, regole di trasporto, record DNS pubblici, certificati con chiavi private, configurazione del gateway e selezioni HCW. I segreti vengono archiviati in modo protetto; le esportazioni leggibili documentano struttura e dipendenze.

Dopo una ricostruzione non viene testata soltanto una porta. Un messaggio contrassegnato percorre in ogni direzione il percorso previsto. Message Trace, registri di tracking locali, protocolli del gateway e cassetta postale di destinazione confermano ogni hop. Solo questo collaudo end-to-end dimostra che routing e filtraggio sono nuovamente corretti.

## Evoluzione tecnica e trade-off

Hybrid Configuration Wizard ha automatizzato, attraverso più generazioni di Exchange, la connessione a Exchange Online. Il trasporto è rimasto SMTP/TLS, mentre connettori cloud, certificati supportati e opzioni di routing sono stati ulteriormente sviluppati ([Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)).

Il trasporto Internet diretto mantiene brevi i percorsi e utilizza la rispettiva piattaforma nel luogo in cui risiede la cassetta postale. Centralized Mail Transport centralizza il controllo, ma rende la posta cloud dipendente dall'uscita locale. Un gateway di terze parti aggiunge filtraggio o crittografia specializzati, ma aumenta il numero di hop e fonti di protocollo. La scelta corretta deriva da un requisito dimostrabile, non dal desiderio che nel diagramma tutto passi attraverso la stessa casella.

## Fonti

- [Microsoft Learn – Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing)
- [Microsoft Learn – Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)
- [Microsoft Learn – Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)
- [Microsoft Learn – Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)
- [Microsoft Learn – Hybrid mail flow](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/hybrid-mail-flow)
- [Microsoft Learn – Enable-RemoteMailbox](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox)
- [Microsoft Learn – Set up connectors to route mail](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail)
- [Microsoft Learn – Connectors on Exchange servers](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors)
- [Microsoft Learn – Accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains)
- [Microsoft Learn – Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)
- [Microsoft Learn – Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud)
- [Microsoft Learn – Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)
- [Microsoft Learn – Trace an email message](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message)
- [Microsoft Learn – Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq)
- [Microsoft Learn – Message tracking](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking)
- [Microsoft Learn – Queues in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues)
- [Microsoft Learn – Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2)
- [Microsoft Learn – Get-MessageTrackingLog](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog)
- [Microsoft Learn – Get-Queue](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [BIND 9 – dig manual](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc(1)](https://man.openbsd.org/nc)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Swaks – SMTP test tool](https://jetmore.org/john/code/swaks/)
- [RFC 5321 – Trace information and loop detection](https://datatracker.ietf.org/doc/html/rfc5321#section-6.3)
