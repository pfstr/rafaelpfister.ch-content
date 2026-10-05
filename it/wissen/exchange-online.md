---
title: "Exchange Online: architettura, flusso di posta e gestione"
blatt: "exchange-online"
description: "Exchange Online per amministratori della messaggistica: modello di tenant e destinatari, trasporto EOP, connettori, accesso client, PowerShell e Graph, Message Trace, conservazione, sicurezza e ripristino."
fakten:
  - label: Ruolo del prodotto
    wert: Servizio cloud per e-mail, calendario e directory
    href: https://learn.microsoft.com/en-us/exchange/exchange-online
  - label: Piattaforma
    wert: Microsoft 365
    href: https://learn.microsoft.com/en-us/office365/servicedescriptions/exchange-online-service-description/exchange-online-service-description
  - label: Ricezione della posta
    wert: Exchange Online Protection e SMTP
    href: https://learn.microsoft.com/en-us/defender-office-365/eop-about
  - label: Destinatari
    wert: Cassette postali, gruppi, contatti, utenti di posta e risorse
    href: https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/recipients-in-exchange-online
  - label: Domini
    wert: Authoritative o Internal Relay
    href: https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains
  - label: Routing
    wert: Connettori in ingresso e in uscita, regole e MX
    href: https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail
  - label: Accesso client
    wert: Outlook, Outlook sul Web, ActiveSync e casi speciali IMAP/POP
    href: https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/clients-and-mobile-in-exchange-online
  - label: Identità
    wert: Microsoft Entra ID e autenticazione moderna
    href: https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/enable-or-disable-modern-auth-in-exchange-online
  - label: Amministrazione
    wert: Exchange Admin Center e Exchange Online PowerShell
    href: https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell
  - label: API
    wert: Microsoft Graph per funzioni di posta, calendario e amministrazione
    href: https://learn.microsoft.com/en-us/graph/api/resources/mail-api-overview
  - label: Diagnostica
    wert: Message Trace, report e Service Health
    href: https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message
  - label: Conservazione
    wert: Recoverable Items, Retention e Holds
    href: https://learn.microsoft.com/en-us/purview/retention-policies-exchange
werbung:
  - tools
  - newsletter
ctaThemen:
  - microsoft-365-exchange
  - smtp-mailflow
translationSourceHash: 5965bf4a9447505ffbe8b9a5d00c6f1abf629dfbde070d9c7700d9fdc9e4b603
translationModel: gpt-5.6-terra
translatedAt: 2026-10-05T11:28:22.700Z
translationReview: automatic
---

# Exchange Online: architettura, flusso di posta e gestione

**Exchange Online** è il servizio Exchange gestito da Microsoft in Microsoft 365. Fornisce cassette postali, calendari, contatti, gruppi, trasporto SMTP e funzioni di amministrazione. L'amministratore del tenant decide in merito a destinatari, domini, connettori, regole, autorizzazioni e conservazione. Microsoft gestisce invece i server delle cassette postali, le copie del database, le code interne, le patch e le procedure di failover ([Exchange Online service description](https://learn.microsoft.com/en-us/office365/servicedescriptions/exchange-online-service-description/exchange-online-service-description), [Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency)).

Exchange Online assomiglia quindi, dal punto di vista funzionale, a un sistema Exchange proprio, ma non dal punto di vista operativo. Un amministratore locale può analizzare un file di coda o attivare una copia del database. In Exchange Online vede invece gli eventi, gli stati e gli oggetti di configurazione messi a disposizione dal servizio. L'abilità più importante consiste quindi nel ricondurre un reclamo di un utente a un percorso chiaro: identità, accesso client, oggetto destinatario, trasporto, filtro, recapito o conservazione.

## Dal tenant alla cassetta postale

Il tenant costituisce il contesto organizzativo. Al suo interno Exchange Online gestisce destinatari abilitati alla posta: cassette postali utente e condivise, cassette postali di sala e apparecchiatura, liste di distribuzione, gruppi Microsoft 365, contatti e utenti di posta. Il tipo di destinatario determina se i dati vengono archiviati, come avviene il recapito e quali autorizzazioni sono disponibili ([Recipients in Exchange Online](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/recipients-in-exchange-online)).

Un account utente in Microsoft Entra ID e una cassetta postale Exchange sono collegati, ma non sono lo stesso oggetto. L'assegnazione della licenza può attivare il provisioning di una cassetta postale. Exchange aggiunge quindi attributi e servizi correlati alla posta. Se un amministratore rimuove una licenza o elimina un account, si applicano diversi periodi di conservazione ed eliminazione. Per la gestione e l'offboarding, il ciclo di vita dell'identità, quello della cassetta postale e la conservazione per conformità devono pertanto essere pianificati insieme ([Delete or restore user mailboxes](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/delete-or-restore-mailboxes), [Retention policies for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange)).

Per gli esperti, l'origine degli attributi diventa importante. In un tenant esclusivamente cloud, le proprietà di Exchange vengono gestite online. Con identità sincronizzate, l'ambiente locale può continuare a essere la fonte autorevole per determinati attributi dei destinatari. Un valore appare quindi in Exchange Online, ma deve essere modificato localmente e sincronizzato nuovamente. Questo modello appartiene all'articolo [Exchange Hybrid](/kb/exchange-hybrid), poiché non esiste senza sincronizzazione della directory.

## Come un messaggio in ingresso raggiunge la cassetta postale

Una volta compreso il destinatario, è possibile seguire il percorso della posta. Il record MX pubblico di un dominio punta normalmente a Exchange Online Protection, EOP. EOP accetta la connessione SMTP, valuta il mittente e il messaggio, applica le regole di protezione e trasporto e trasferisce un messaggio consentito a Exchange Online. Per i destinatari locali segue quindi il recapito nella cassetta postale ([Exchange Online Protection overview](https://learn.microsoft.com/en-us/defender-office-365/eop-about), [Mail flow in EOP](https://learn.microsoft.com/en-us/defender-office-365/eop-mail-flow)).

Il **dominio accettato** stabilisce come Exchange Online tratta il dominio del destinatario. Con `Authoritative` il servizio si aspetta tutti i destinatari validi nella propria organizzazione e rifiuta gli indirizzi sconosciuti. `Internal Relay` consente di inoltrare destinatari sconosciuti a un altro sistema. Questa impostazione è utile solo se l'hop successivo e la risoluzione dei destinatari sono pianificati in modo affidabile; altrimenti si verificano mancati recapiti o loop ([Manage accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains)).

Un messaggio interno non rimane automaticamente «sullo stesso server». Exchange Online risolve mittente e destinatario, verifica regole e criteri di protezione e registra eventi di trasporto. Per l'amministratore, questa catena di eventi è decisiva: `Delivered` significa che il servizio ha recapitato al destinatario; `Filtered`, `Failed`, `Pending` o `Expanded` descrivono altri passaggi. Message Trace rende visibili questi passaggi, ma non sostituisce la verifica della cassetta postale di destinazione o di una regola successiva ([Trace an email message](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message), [Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq)).

## Messaggi in uscita e connettori

Per i messaggi in uscita viene prima deciso se Exchange Online invia direttamente al sistema di destinazione o utilizza un connettore in uscita configurato. Un connettore può instradare i messaggi verso la propria infrastruttura, un partner o un gateway di posta. La selezione si basa, tra l'altro, sul dominio del destinatario, sulle condizioni del connettore e sulle regole di trasporto ([Set up connectors to route mail](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail)).

I connettori in ingresso descrivono invece le condizioni alle quali Exchange Online si fida di un sistema mittente. I criteri tipici sono l'IP di origine o un certificato TLS. Queste informazioni sono rilevanti per la sicurezza: un intervallo IP troppo ampio o un certificato verificato in modo impreciso può far apparire traffico esterno come traffico interno di un partner.

Se un gateway di posta esterno è posto davanti a EOP, Microsoft vede inizialmente l'IP del gateway. **Enhanced Filtering for Connectors** può includere informazioni sull'hop originale nella valutazione del filtro. La funzione non è un generico «interruttore antispam», ma deve corrispondere al percorso effettivo, ai connettori e agli IP ignorati ([Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)).

La domanda per gli esperti è: quale controparte ha effettivamente accettato un messaggio, quale identità è stata verificata per il connettore e su quale hop è avvenuto l'ultimo filtro del contenuto? Queste tre risposte devono essere incluse in ogni diagramma del flusso di posta.

## Struttura tecnica dal punto di vista dell'amministratore

Exchange Online non pubblica un elenco di server che un amministratore del tenant gestisce come una farm locale. Ciononostante, il servizio dispone di componenti tecnici chiaramente riconoscibili. Diventano visibili tramite protocolli e interfacce di amministrazione.

| Componente | Compito | Ciò che vede l'amministratore del tenant |
|---|---|---|
| Exchange Online Protection | Accettazione SMTP, antimalware, antispam ed elaborazione del trasporto | Quarantena, criteri, report e Message Trace |
| Trasporto Exchange | Risoluzione dei destinatari, regole, routing e recapito | Connettori, domini accettati, regole ed eventi |
| Servizio cassette postali | Archiviazione di e-mail, calendari, contatti e cartelle | Oggetti cassetta postale, quote, autorizzazioni e accesso client |
| Microsoft Entra ID | Identità di utenti, gruppi, applicazioni e accesso | Account, ruoli, Conditional Access e registrazioni di app |
| Exchange Online PowerShell | Amministrazione specifica di Exchange | Cmdlet, RBAC e modifiche verificabili |
| Microsoft Graph | API REST per applicazioni e automazione | Autorizzazioni OAuth, risorse e throttling |

Lo stack tecnologico circostante consiste quindi soprattutto in SMTP e TLS per il trasporto della posta, nonché HTTPS, OAuth, PowerShell e REST per l'accesso client e amministrativo. I dettagli di implementazione interni sono rilevanti per il cliente soltanto nella misura in cui Microsoft li documenta come comportamento del servizio, limite o interfaccia diagnostica ([About the Exchange Online PowerShell module](https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell-v2), [Microsoft Graph mail API](https://learn.microsoft.com/en-us/graph/api/resources/mail-api-overview)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1056" src="/images/kb-interaktiv-exchange-online.svg?v=20260813" title="Interaktive Infografik: Exchange-Online-Pfad von DNS und EOP über Transport und Postfach bis Entra, PowerShell, Graph und Betriebsnachweis" loading="lazy">
  <a href="/images/kb-interaktiv-exchange-online.svg?v=20260813">Apri direttamente la grafica interattiva di Exchange Online</a>.
</iframe>

## Accesso client e autenticazione moderna

Il trasporto della posta termina nella cassetta postale; gli utenti vi accedono successivamente tramite protocolli client. Outlook, Outlook sul Web, client mobili e applicazioni utilizzano endpoint basati su HTTPS. Autodiscover aiuta i client a individuare il servizio appropriato. L'accesso avviene tramite Microsoft Entra ID, mentre Exchange verifica l'autorizzazione sulla cassetta postale ([Clients and mobile in Exchange Online](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/clients-and-mobile-in-exchange-online), [Modern authentication in Exchange Online](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/enable-or-disable-modern-auth-in-exchange-online)).

Questo separa due errori che vengono spesso confusi. Se l'accesso a Entra non riesce, spesso il client non raggiunge affatto Exchange. Se il token è valido, Exchange può comunque rifiutare l'accesso per mancanza di ruolo, autorizzazione della cassetta postale, criterio client o cassetta postale di destinazione errata. Il registro di accesso e la diagnostica di Exchange devono pertanto essere considerati insieme nel tempo.

Le applicazioni accedono preferibilmente tramite Microsoft Graph o interfacce Exchange supportate. Un'autorizzazione dell'applicazione Graph può avere un ambito ampio; Exchange RBAC for Applications può definire in modo più restrittivo l'ambito delle cassette postali accessibili. Un token OAuth valido è quindi soltanto il primo passo. Il servizio della risorsa verifica poi quale azione è consentita su quale cassetta postale ([Role Based Access Control for Applications](https://learn.microsoft.com/en-us/exchange/permissions-exo/application-rbac)).

## Tracciare autorizzazioni e modifiche

Exchange Online dispone di ruoli amministrativi propri. I ruoli Entra possono consentire l'accesso iniziale all'amministrazione di Exchange, ma i Cmdlet Exchange effettivi e il loro ambito sono determinati da Exchange-RBAC ([Permissions in Exchange Online](https://learn.microsoft.com/en-us/exchange/permissions-exo/permissions-exo)).

Esistono inoltre autorizzazioni della cassetta postale quali Full Access, Send As e Send on Behalf. Controllano azioni diverse e non dovrebbero essere inventariate come un unico «diritto di delega». Per le applicazioni si aggiungono ruoli OAuth e ruoli applicativi Exchange ([Manage permissions for recipients](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/manage-permissions-for-recipients)).

Per gli esperti, l'origine di una modifica è importante quanto lo stato finale. I registri di audit, i registri di accesso Entra e le esportazioni di configurazione consentono di stabilire chi ha modificato una regola, un connettore o un'autorizzazione. Un'esportazione notturna degli oggetti centrali del flusso di posta facilita i confronti, ma non sostituisce una fonte di audit protetta.

## Diagnostica: prima DNS, poi eventi di trasporto

Un'analisi del flusso di posta inizia al di fuori del tenant. Il record MX indica quale sistema accetta la posta da Internet. Successivamente, Message Trace verifica se Exchange Online ha visto il messaggio concreto e come lo ha elaborato.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für MX- und Autodiscover-Abfrage">
    <button type="button" role="tab" data-os-tab="windows" aria-selected="true">Windows</button>
    <button type="button" role="tab" data-os-tab="unix" aria-selected="false">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName -Type MX example.com
Resolve-DnsName autodiscover.example.com
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig MX example.com
dig autodiscover.example.com
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) e [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) mostrano pubblicazione e risoluzione. Non indicano ancora se EOP ha accettato il messaggio o se una cassetta postale lo ha ricevuto.

Per il passaggio successivo si sceglie un intervallo temporale ristretto con mittente e destinatario. La stessa Exchange Online PowerShell viene eseguita su Windows e con `pwsh` su sistemi Unix supportati.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Exchange Online PowerShell">
    <button type="button" role="tab" data-os-tab="windows" aria-selected="true">Windows</button>
    <button type="button" role="tab" data-os-tab="unix" aria-selected="false">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Connect-ExchangeOnline
Get-MessageTraceV2 -SenderAddress sender@example.net `
  -StartDate (Get-Date).AddHours(-2) -EndDate (Get-Date)
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```powershell
pwsh
Connect-ExchangeOnline
Get-MessageTraceV2 -SenderAddress sender@example.net `
  -StartDate (Get-Date).AddHours(-2) -EndDate (Get-Date)
```

  </div>
</div>

[`Connect-ExchangeOnline`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/connect-exchangeonline) stabilisce la sessione di amministrazione autenticata. [`Get-MessageTraceV2`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2) cerca eventi di trasporto; [`Get-Date`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-date) limita l'intervallo temporale. Per le tendenze si aggiungono i report, per i problemi Microsoft Service Health. Un singolo segnale verde non risponde a tutte e tre le domande ([Exchange Online monitoring](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-exchange-monitoring?view=o365-worldwide)).

## Conservazione, eliminazione e ripristino

Microsoft protegge il servizio in esecuzione con più copie del database, Shadow Redundancy e Safety Net. Questi meccanismi servono alla disponibilità e all'integrità dei dati del servizio. Non costituiscono l'interfaccia utente per il ripristino di un messaggio eliminato accidentalmente ([Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency)).

Per i casi degli utenti e di conformità intervengono altre funzioni: Deleted Item Retention, Recoverable Items, Single Item Recovery, Retention Policies e Holds. I loro effetti si sovrappongono, ma hanno scopi diversi. Una regola di conservazione può proteggere i contenuti dall'eliminazione definitiva; non fornisce automaticamente un backup separato e indipendente dal tenant con un punto di ripristino liberamente selezionabile ([Recoverable Items folder](https://learn.microsoft.com/en-us/exchange/security-and-compliance/recoverable-items-folder/recoverable-items-folder), [Retention policies for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange)).

Un concetto di ripristino solido specifica pertanto quali eventi sono coperti dalla resilienza del servizio Microsoft, quali contenuti possono essere recuperati tramite la conservazione di Exchange o Purview e per quali requisiti è necessaria una copia indipendente. I test di ripristino dovrebbero utilizzare casi concreti: singolo messaggio, cartella, cassetta postale dopo l'eliminazione dell'utente, elemento conservato per obblighi legali e interruzione a livello di tenant.

## Sicurezza e limiti tipici

Exchange Online combina diversi ambiti di sicurezza: posta Internet, EOP, configurazione del tenant, accesso Entra, diritti della cassetta postale e applicazioni. L'efficacia della protezione dipende dalla corrispondenza tra il percorso reale del messaggio e dell'accesso e la configurazione.

Per il flusso di posta significa che MX, identità del connettore, Enhanced Filtering, SPF/DKIM/DMARC e regole di trasporto devono essere verificati come una catena. Per l'accesso client, autenticazione moderna, Conditional Access, Exchange-RBAC e autorizzazioni della cassetta postale sono controlli separati. Per le applicazioni si aggiungono il consenso OAuth e l'ambito di cassette postali consentito.

La domanda amministrativa più approfondita è sempre la stessa: quale sistema ha preso la decisione, quali dati di input ha visto e dove è stato registrato il risultato? Senza queste tre informazioni, anche un criterio formalmente corretto resta difficile da verificare.

## Evoluzione tecnica e trade-off consapevoli

Exchange Online si è evoluto dalle offerte Exchange ospitate di Microsoft e ha adottato molti concetti del prodotto server: destinatari, database delle cassette postali, trasporto, DAG, Shadow Redundancy e Safety Net. Il servizio automatizza la gestione di questa infrastruttura e fornisce agli amministratori del tenant un livello amministrativo superiore ([Exchange Team: 20 years ago](https://techcommunity.microsoft.com/blog/exchange/20-years-ago-in-a-galaxy-far-away8230/604456), [Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency)).

Il vantaggio consiste nella gestione esternalizzata della piattaforma, nell'integrazione globale dei servizi e nelle interfacce di amministrazione standardizzate. Il prezzo è un minore accesso a singoli server, code e copie del database, nonché una maggiore dipendenza dalle funzioni pubblicate di diagnostica, esportazione e ripristino. Per gli esperti, il compito non consiste quindi nell'indovinare la topologia interna invisibile, ma nell'utilizzare pienamente i controlli del tenant e i segnali del servizio documentati.

## Fonti

- [Microsoft Learn – Exchange Online](https://learn.microsoft.com/en-us/exchange/exchange-online)
- [Microsoft – Exchange Online service description](https://learn.microsoft.com/en-us/office365/servicedescriptions/exchange-online-service-description/exchange-online-service-description)
- [Microsoft Defender – Exchange Online Protection overview](https://learn.microsoft.com/en-us/defender-office-365/eop-about)
- [Microsoft Defender – Mail flow in EOP](https://learn.microsoft.com/en-us/defender-office-365/eop-mail-flow)
- [Microsoft Learn – Recipients in Exchange Online](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/recipients-in-exchange-online)
- [Microsoft Learn – Delete or restore user mailboxes](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/delete-or-restore-mailboxes)
- [Microsoft Learn – Accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains)
- [Microsoft Learn – Set up connectors to route mail](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail)
- [Microsoft Learn – Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)
- [Microsoft Learn – Trace an email message](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message)
- [Microsoft Learn – Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq)
- [Microsoft Learn – Clients and mobile in Exchange Online](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/clients-and-mobile-in-exchange-online)
- [Microsoft Learn – Modern authentication in Exchange Online](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/enable-or-disable-modern-auth-in-exchange-online)
- [Microsoft Learn – Exchange Online PowerShell](https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell)
- [Microsoft Learn – About the Exchange Online PowerShell module](https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell-v2)
- [Microsoft Graph – Mail API overview](https://learn.microsoft.com/en-us/graph/api/resources/mail-api-overview)
- [Microsoft Learn – RBAC for Applications](https://learn.microsoft.com/en-us/exchange/permissions-exo/application-rbac)
- [Microsoft Learn – Permissions in Exchange Online](https://learn.microsoft.com/en-us/exchange/permissions-exo/permissions-exo)
- [Microsoft Learn – Manage permissions for recipients](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/manage-permissions-for-recipients)
- [Microsoft Service Assurance – Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency)
- [Microsoft Learn – Recoverable Items folder](https://learn.microsoft.com/en-us/exchange/security-and-compliance/recoverable-items-folder/recoverable-items-folder)
- [Microsoft Purview – Retention policies for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange)
- [Microsoft Learn – Exchange Online monitoring](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-exchange-monitoring?view=o365-worldwide)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [BIND 9 – dig manual](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility)
- [Microsoft Learn – Connect-ExchangeOnline](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/connect-exchangeonline)
- [Microsoft Learn – Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2)
- [Microsoft Learn – Get-Date](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-date)
- [Exchange Team – 20 years ago](https://techcommunity.microsoft.com/blog/exchange/20-years-ago-in-a-galaxy-far-away8230/604456)
