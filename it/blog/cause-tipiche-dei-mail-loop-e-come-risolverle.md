---
title: "Cause tipiche dei mail loop e come risolverle"
navTitle: "Risolvere i mail loop"
description: "Come individuare e risolvere sistematicamente i mail loop SMTP in Exchange Online, ambienti ibridi e mail gateway a monte tramite NDR, header, Message Trace, oggetti destinatario e connettori."
date: "2026-08-07"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "12 min di lettura"
themen:
  - exchange-onprem-hybrid
  - microsoft-365-exchange
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange"
protokolle:
  - "smtp"
  - "powershell"
  - "troubleshooting"
related:
  - exchange-mailboxen-ohne-remote-move-cloud-neu-erstellen
  - totemomail-m365
  - ghost-sender-exchange-online-nebeneingang
slug: "cause-tipiche-dei-mail-loop-e-come-risolverle"
translationId: "article-4c91e7b2a8605fd3"
draft: false
translationOf: typische-ursachen-fuer-mail-loops-und-deren-behebung
translationSourceHash: 6bd8a9a875aa7795711db421084241c47d7b2c1f24abb9ba64bc1b18128d86b7
translationModel: gpt-5.6-terra
translatedAt: 2026-10-30T03:37:12.112Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/cause-tipiche-dei-mail-loop-e-come-risolverle
---

# Cause tipiche dei mail loop e come risolverle

Un mail loop si verifica quando almeno due sistemi di trasporto si scambiano ripetutamente lo stesso messaggio. Nessuno dei sistemi si riconosce come destinazione finale, ma entrambi conoscono un hop successivo apparentemente appropriato. Il ciclo termina solo quando un server rileva il superamento del numero consentito di stazioni di trasporto e genera un NDR.

In Exchange, due messaggi sono particolarmente indicativi:

- `554 5.4.6 Hop count exceeded - possible mail loop` viene generato tipicamente da Exchange locale.
- `554 5.4.14 Hop count exceeded - possible mail loop ATTR34` viene generato da Exchange Online.

Il limite degli hop non è la causa, bensì la protezione contro una ripetizione infinita. Aumentarlo non risolve quindi nulla. Occorre individuare il punto in cui il messaggio viene restituito a un sistema già attraversato, contrariamente all'architettura di destinazione.

## Riconoscere lo schema del ciclo nell'header

L'NDR e gli header completi del messaggio originale devono essere salvati prima di qualsiasi modifica. Le righe `Received` si leggono dal basso verso l'alto: la riga più in basso è l'hop documentato più antico, quella più in alto il più recente.

Un ciclo si manifesta di solito come sequenza ricorrente:

```text
Internet
  → Exchange Online Protection
  → Mailgateway
  → Exchange Online Protection
  → Mailgateway
  → Exchange Online Protection
  → ...
```

Non tutti gli hostname Microsoft che compaiono più volte indicano già un ciclo. Exchange Online elabora internamente i messaggi tramite più ruoli di trasporto. È invece sospetto il ripetuto ritorno tra gli stessi confini amministrativi, ad esempio tra Exchange Online e un gateway locale. Timestamp, IP mittente, host ricevente e `Message-ID` aiutano a riconoscere inequivocabilmente il giro.

Per la prima analisi, rispondere a queste domande:

1. Quale sistema ha generato l'NDR?
2. Quali due o tre hop si ripetono?
3. Quale sistema avrebbe dovuto consegnare definitivamente il messaggio?
4. In base a quale decisione relativa a dominio, destinatario, connettore o regola è stato inoltrato?
5. Quale modifica ha influenzato per ultima il flusso di posta?

## Diagnosi in Exchange Online

Con `Get-MessageTraceV2` è possibile analizzare l'elaborazione degli ultimi 90 giorni; ogni query può coprire al massimo dieci giorni. Una finestra temporale ristretta e l'indirizzo concreto del destinatario forniscono i risultati più utili:

```powershell
$start = (Get-Date).AddHours(-2)
$end = Get-Date
$recipient = "user01@contoso.com"

$trace = Get-MessageTraceV2 `
    -RecipientAddress $recipient `
    -StartDate $start `
    -EndDate $end `
    -ResultSize 5000

$trace |
    Select-Object Received,SenderAddress,RecipientAddress,Subject,
        Status,FromIP,ToIP,MessageTraceId,MessageId |
    Sort-Object Received
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-RecipientAddress` | Filtra la traccia in base all'indirizzo del destinatario specificato |
| `-StartDate` / `-EndDate` | Finestra temporale della query; ogni query può coprire al massimo dieci giorni |
| `-ResultSize 5000` | Numero massimo di voci restituite |
| `Select-Object …` | Riduce l'output ai campi rilevanti per l'analisi del loop |
| `Sort-Object Received` | Ordina cronologicamente i risultati in base all'ora di ricezione |

</details>

I dettagli di un risultato mostrano i singoli eventi di trasporto:

```powershell
$trace | ForEach-Object {
    Get-MessageTraceDetailV2 `
        -MessageTraceId $_.MessageTraceId `
        -RecipientAddress $_.RecipientAddress
} | Format-Table Date,Event,Action,Detail -AutoSize
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-MessageTraceId` | ID di traccia univoco dal risultato di `Get-MessageTraceV2` |
| `-RecipientAddress` | Indirizzo del destinatario del risultato; necessario insieme all'ID di traccia per la query di dettaglio |
| `Format-Table … -AutoSize` | Adatta la larghezza delle colonne al contenuto, affinché i dettagli degli eventi rimangano leggibili |

</details>

Successivamente si esaminano insieme dominio, destinatario e connettori:

```powershell
Get-AcceptedDomain |
    Format-Table Name,DomainName,DomainType,MatchSubDomains -AutoSize

Get-EXORecipient -Identity $recipient |
    Format-List DisplayName,RecipientTypeDetails,PrimarySmtpAddress,
        ExternalEmailAddress,EmailAddresses

Get-OutboundConnector -IncludeTestModeConnectors |
    Format-List Name,Enabled,ConnectorType,RecipientDomains,SmartHosts,
        UseMXRecord,RouteAllMessagesViaOnPremises,TlsSettings

Get-InboundConnector |
    Format-List Name,Enabled,ConnectorType,SenderDomains,SenderIPAddresses,
        TlsSenderCertificateName,RequireTls,RestrictDomainsToIPAddresses,
        RestrictDomainsToCertificate
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Identity $recipient` | Seleziona l'oggetto destinatario tramite indirizzo, alias o nome |
| `-IncludeTestModeConnectors` | Include nell'output anche i connettori in modalità di test |
| `Format-Table … -AutoSize` | Visualizzazione tabellare con larghezza delle colonne in base al contenuto |
| `Format-List …` | Visualizzazione a elenco delle proprietà indicate, adatta a valori lunghi come gli elenchi di indirizzi |

</details>

Non è decisivo che un singolo oggetto sembri plausibile. Tipo di dominio, tipo effettivo di destinatario e connettore applicabile devono descrivere tutti la stessa destinazione.

## Diagnosi in Exchange locale

In un ambiente ibrido, lo stesso destinatario viene verificato anche localmente. Le query distinguono tra una vera mailbox locale, una RemoteMailbox e un MailUser:

```powershell
Get-Recipient -Identity $recipient |
    Format-List DisplayName,RecipientType,RecipientTypeDetails,
        PrimarySmtpAddress,EmailAddresses

Get-Mailbox -Identity $recipient -ErrorAction SilentlyContinue |
    Format-List RecipientTypeDetails,ServerName,Database,PrimarySmtpAddress

Get-RemoteMailbox -Identity $recipient -ErrorAction SilentlyContinue |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,RemoteRoutingAddress

Get-MailUser -Identity $recipient -ErrorAction SilentlyContinue |
    Format-List RecipientTypeDetails,PrimarySmtpAddress,ExternalEmailAddress
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Identity $recipient` | Seleziona l'oggetto tramite indirizzo, alias o nome |
| `-ErrorAction SilentlyContinue` | Sopprime il messaggio di errore se l'oggetto non esiste nel tipo specifico; in tal caso la query non restituisce semplicemente alcun risultato |

</details>

Per il percorso di trasporto sono necessari i connettori Send e Receive e i log di tracking:

```powershell
Get-SendConnector |
    Format-List Name,Enabled,AddressSpaces,DNSRoutingEnabled,SmartHosts,
        SourceTransportServers,CloudServicesMailEnabled,TlsDomain

Get-ReceiveConnector |
    Format-List Identity,Enabled,Bindings,RemoteIPRanges,PermissionGroups

$servers = Get-ExchangeServer |
    Where-Object { $_.IsMailboxServer -or $_.IsHubTransportServer }

$servers |
    Get-MessageTrackingLog `
        -Start $start `
        -End $end `
        -Recipients $recipient `
        -ResultSize Unlimited |
    Select-Object Timestamp,ServerHostname,ClientHostname,Source,EventId,
        ConnectorId,Sender,Recipients,MessageId,NetworkMessageId |
    Sort-Object Timestamp
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Where-Object { … }` | Limita l'elenco dei server ai server Mailbox e Hub Transport, ovvero i ruoli con log di tracking |
| `-Start` / `-End` | Finestra temporale per la ricerca nei log |
| `-Recipients $recipient` | Filtra gli eventi di tracking con questo indirizzo destinatario |
| `-ResultSize Unlimited` | Rimuove il limite predefinito di 1000 voci restituite |
| `Select-Object …` | Riduce l'output ai campi rilevanti per l'analisi del percorso |
| `Sort-Object Timestamp` | Ordina cronologicamente gli eventi di tutti i server |

</details>

Un `SEND` verso Exchange Online, seguito da un nuovo `RECEIVE` dello stesso messaggio da Exchange Online, rende visibile la restituzione. Con `MessageId` e `NetworkMessageId` si evita di confondere tra loro messaggi di test diversi.

## Le cause più comuni in sintesi

| Schema | Causa tipica | Risoluzione |
| --- | --- | --- |
| Destinatari sconosciuti oscillano tra due sistemi | Il dominio accettato è impostato su `InternalRelay`, ma entrambe le parti inoltrano i destinatari sconosciuti | Definire una responsabilità univoca; in caso di consegna completa in EXO usare `Authoritative` oppure, per una Split-Domain, definire un unico hop finale |
| EXO invia a Exchange locale, che restituisce subito a EXO | Il connettore ibrido o Centralized Mail Transport non corrisponde più alla posizione della mailbox | Verificare la configurazione HCW e `RouteAllMessagesViaOnPremises`; disattivare il routing centrale obsoleto o correggere la risoluzione locale dei destinatari |
| Il messaggio oscilla tra EXO e un gateway di sicurezza, firma o crittografia | I messaggi restituiti soddisfano nuovamente la regola in uscita | Usare come eccezione l'header impostato dal gateway o il meccanismo documentato di prevenzione dei loop; autenticare chiaramente i connettori in entrata e in uscita |
| È interessato un solo destinatario | `targetAddress` obsoleto o errato, tipo RemoteMailbox errato o indirizzi proxy contraddittori | Determinare la Source of Authority, correggere l'oggetto destinatario in quella sede e sincronizzare |
| Passano nel loop solo i messaggi inoltrati | Una regola di trasporto, l'inoltro della mailbox o una regola Inbox indirizza nuovamente al percorso originale | Disattivare la regola, correggere la destinazione e definire un'eccezione affidabile |
| È interessata solo una sottodominio o un'applicazione | Il dominio padre non copre correttamente la sottodominio nel percorso del connettore previsto | Configurare esplicitamente la sottodominio come dominio accettato e nel Send Connector appropriato |
| Tutti i messaggi entrano nel loop dopo una modifica al gateway o al DNS | Smart Host o MX punta all'ingresso del sistema mittente | Correggere l'hop successivo e verificare separatamente le destinazioni DNS, NAT e Load Balancer |

## Causa 1: Tipo errato del dominio accettato

Un dominio authoritative significa che tutti i destinatari validi di quel dominio sono noti nell'organizzazione Exchange; i destinatari sconosciuti vengono rifiutati. Un dominio Internal Relay significa che una parte dei destinatari si trova in un altro sistema e deve essere inoltrata tramite un connettore Send o Outbound.

La configurazione problematica si verifica quando Exchange Online invia i destinatari sconosciuti a un sistema locale e quest'ultimo non gestisce definitivamente lo stesso dominio, ma lo restituisce a Exchange Online tramite MX o Smart Host.

```powershell
Get-AcceptedDomain -Identity contoso.com |
    Format-List DomainName,DomainType,MatchSubDomains
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Identity contoso.com` | Seleziona il dominio accettato da verificare |
| `Format-List …` | Mostra come elenco nome del dominio, tipo di dominio e copertura delle sottodomini |

</details>

Se, al termine di una migrazione, tutti i destinatari si trovano in Exchange Online, `Authoritative` è di solito lo stato finale corretto:

```powershell
# Eseguire solo dopo aver verificato completamente destinatari e routing.
Set-AcceptedDomain -Identity contoso.com -DomainType Authoritative
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Identity contoso.com` | Il dominio accettato da modificare |
| `-DomainType Authoritative` | Imposta il dominio su authoritative: i destinatari sconosciuti vengono rifiutati anziché inoltrati |

</details>

In una vera Split-Domain, `InternalRelay` può essere corretto. Tuttavia, serve un connettore chiaro verso il sistema che conosce i destinatari rimanenti. Questa destinazione non deve rinviare gli indirizzi sconosciuti al punto di partenza.

## Causa 2: Connettori ibridi sovrapposti e Centralized Mail Transport

Centralized Mail Transport instrada deliberatamente i messaggi in uscita di Exchange Online attraverso Exchange locale. Ciò è utile per determinati requisiti di conformità, ma crea percorsi di trasporto aggiuntivi. Se l'opzione rimane attiva dopo una migrazione, mentre il sistema locale rinvia a Exchange Online tramite il proprio MX, può formarsi un ciclo.

```powershell
Get-OutboundConnector -IncludeTestModeConnectors |
    Format-Table Name,Enabled,ConnectorType,RouteAllMessagesViaOnPremises,
        RecipientDomains,SmartHosts,UseMXRecord -AutoSize
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-IncludeTestModeConnectors` | Include nell'output anche i connettori in modalità di test |
| `Format-Table … -AutoSize` | Visualizzazione tabellare delle proprietà di routing con larghezza delle colonne in base al contenuto |

</details>

Vanno verificati anche più connettori con ambiti sovrapposti. Microsoft raccomanda un connettore On-Premises dedicato per il flusso di posta ibrido; una riparazione tramite Hybrid Configuration Wizard è spesso più sicura di singole modifiche isolate.

Se Centralized Mail Transport non è più necessario in modo dimostrabile, l'impostazione può essere disattivata in modo mirato:

```powershell
# Solo dopo aver verificato i requisiti di conformità e del gateway.
Set-OutboundConnector `
    -Identity "Outbound to On-Premises" `
    -RouteAllMessagesViaOnPremises:$false
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Identity "Outbound to On-Premises"` | Il connettore Outbound da modificare |
| `-RouteAllMessagesViaOnPremises:$false` | Disattiva Centralized Mail Transport: i messaggi in uscita da Exchange Online non passano più attraverso Exchange locale |

</details>

## Causa 3: Un gateway rielabora i messaggi restituiti

In uno scenario in-and-out, Exchange Online invia un messaggio a un servizio aggiuntivo per la firma, la crittografia o l'archiviazione. Questo lo restituisce poi a Exchange Online. La regola in uscita deve riconoscere il messaggio restituito; altrimenti viene inviato nuovamente al servizio.

La verifica inizia da tutte le regole che selezionano connettori, reindirizzano destinatari o valutano header:

```powershell
Get-TransportRule |
    Sort-Object Priority |
    Format-List Name,State,Mode,Priority,FromScope,SentToScope,
        RedirectMessageTo,RouteMessageOutboundConnector,
        SetHeaderName,SetHeaderValue,ExceptIfHeaderContainsMessageHeader,
        ExceptIfHeaderContainsWords
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Sort-Object Priority` | Ordina le regole secondo il relativo ordine di valutazione |
| `Format-List …` | Mostra le proprietà che selezionano connettori, reindirizzano destinatari oppure impostano header o li valutano come eccezione |

</details>

L'eccezione concreta deve seguire la documentazione del produttore del gateway. Di norma si usa un header impostato dal servizio e non falsificabile in modo affidabile da Internet. Inoltre, i connettori in entrata dovrebbero identificare il servizio tramite certificato o IP mittente fisso. Un'eccezione generica per tutti i messaggi apparentemente «interni» è troppo ampia.

## Causa 4: L'oggetto destinatario e la mailbox effettiva non si trovano nello stesso luogo

Un oggetto può apparire in Exchange Online come `MailUser`, anche se la mailbox attiva si trova localmente. In un ambiente ibrido sincronizzato, non si tratta automaticamente di un duplicato. Anche un `ExternalEmailAddress`, corrispondente all'indirizzo SMTP primario, non dimostra di per sé una configurazione errata.

È determinante la combinazione di tutte le query:

- `Get-Mailbox` locale restituisce un risultato: la mailbox attiva è locale.
- `Get-RemoteMailbox` locale restituisce un risultato: la destinazione gestita si trova in Exchange Online.
- `Get-EXOMailbox` restituisce un risultato: nel cloud esiste una mailbox effettiva.
- `Get-EXORecipient` restituisce solo un MailUser: l'oggetto è una destinazione di routing, non una mailbox cloud.

Sono problematici gli oggetti obsoleti dopo una migrazione, domini di routing remoto errati o valori `targetAddress` impostati manualmente, il cui dominio conduce nuovamente indietro tramite lo stesso percorso di trasporto. Le modifiche vanno effettuate presso la Source of Authority: negli ambienti sincronizzati, quindi, con gli strumenti di gestione Exchange locali e non modificando direttamente singoli attributi in Exchange Online.

## Causa 5: Inoltri e regole di trasporto formano un cerchio

Una regola può reindirizzare dall'indirizzo A a B, mentre B invia nuovamente ad A tramite una seconda regola, un inoltro della mailbox o un sistema esterno. Questi cicli spesso interessano solo singoli destinatari o tipi di messaggio.

```powershell
Get-TransportRule |
    Sort-Object Priority |
    Select-Object Name,State,Mode,Priority,RedirectMessageTo,
        BlindCopyTo,AddToRecipients,RouteMessageOutboundConnector

Get-Mailbox -ResultSize Unlimited |
    Select-Object DisplayName,PrimarySmtpAddress,
        ForwardingAddress,ForwardingSmtpAddress,DeliverToMailboxAndForward

Get-InboxRule -Mailbox user01@contoso.com |
    Select-Object Name,Enabled,Priority,ForwardTo,RedirectTo,ForwardAsAttachmentTo
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Sort-Object Priority` | Ordina le regole di trasporto secondo il relativo ordine di valutazione |
| `-ResultSize Unlimited` | Rimuove il limite predefinito di 1000 mailbox restituite |
| `-Mailbox user01@contoso.com` | Mailbox di cui vengono interrogate le regole Inbox |
| `Select-Object …` | Riduce l'output alle destinazioni di inoltro e reindirizzamento |

</details>

La risoluzione non consiste soltanto nel disattivare temporaneamente una regola. Occorre eliminare l'intera catena e le regole per servizi esterni necessitano di un'eccezione che riconosca in modo sicuro i messaggi già elaborati.

## Causa 6: MX, Smart Host o sottodominio punta indietro

Un gateway può richiedere internamente un hop successivo diverso rispetto ai mittenti esterni. Se per l'inoltro usa semplicemente l'MX pubblico, quest'ultimo potrebbe puntare nuovamente al gateway stesso. Lo stesso problema si verifica quando uno Smart Host riconduce al proprio listener tramite NAT o Load Balancing.

```powershell
Resolve-DnsName -Type MX contoso.com
Resolve-DnsName -Type MX app.contoso.com

Get-SendConnector |
    Format-List Name,AddressSpaces,DNSRoutingEnabled,SmartHosts
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Type MX` | Interroga i record MX anziché i record A predefiniti |
| `contoso.com` / `app.contoso.com` | Dominio da interrogare come argomento posizionale (parametro `-Name`) |
| `Format-List …` | Mostra per ogni Send Connector spazi di indirizzi, modalità di routing e Smart Host |

</details>

Le sottodomini meritano una verifica separata. Microsoft ha documentato, in un articolo di supporto nel frattempo ritirato, casi in cui una sottodominio applicativa deve essere creata esplicitamente come dominio Internal Relay e sincronizzata con i sistemi Edge:

```powershell
New-AcceptedDomain `
    -Name "app.contoso.com" `
    -DomainName app.contoso.com `
    -DomainType InternalRelay

Start-EdgeSynchronization
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Name "app.contoso.com"` | Nome visualizzato del nuovo oggetto dominio accettato |
| `-DomainName app.contoso.com` | Il dominio SMTP per il quale Exchange accetta messaggi |
| `-DomainType InternalRelay` | Una parte dei destinatari si trova al di fuori dell'organizzazione; i destinatari sconosciuti vengono inoltrati tramite un Send Connector invece di essere rifiutati |

</details>

Questi comandi non sono una correzione universale. Sono adatti solo se `app.contoso.com` viene effettivamente consegnato al di fuori dell'organizzazione Exchange e il Send Connector dispone di un hop successivo univoco.

## Procedura sicura in caso di ciclo attivo

Durante l'interruzione, occorre innanzitutto fermare la moltiplicazione. A seconda dell'architettura, la regola di trasporto che lo causa o il connettore specifico viene disattivato in modo controllato, oppure il gateway trattiene la coda interessata. Prima vengono esportati configurazione ed esempi di messaggi.

Segue quindi un test con esattamente un mittente, un destinatario e un oggetto facilmente riconoscibile. Il messaggio viene seguito senza interruzioni tramite header, Message Trace e log di tracking locali. Solo quando termina alla destinazione prevista, il flusso di posta viene riaperto gradualmente.

Non sono consigliabili:

- aumentare i limiti degli hop
- modificare più connettori contemporaneamente
- commutare per tentativi i domini accettati tra `Authoritative` e `InternalRelay`
- reinserire ripetutamente una coda problematica senza verificarla
- correggere direttamente in AD o Exchange Online gli attributi Exchange sincronizzati
- disattivare le verifiche TLS, IP o dei certificati come presunta soluzione rapida

## Controllo finale

Dopo la correzione, la documentazione dovrebbe contenere per ogni dominio rilevante un'unica indicazione: quale sistema conosce il destinatario, quale connettore è applicabile e quale host è l'hop successivo finale?

L'accettazione tecnica comprende almeno:

- messaggio di test esterno e interno
- destinatario sconosciuto nello stesso dominio
- destinatario su ciascun lato di una vera Split-Domain
- messaggio in uscita con gateway o Centralized Mail Transport attivo
- header senza sequenza di hop ricorrente
- Message Trace con `Delivered` o consegna prevista
- tracking locale senza un nuovo `RECEIVE` dopo un `SEND` verso la stessa destinazione
- convalida dei connettori per tutti i connettori ancora necessari

Un mail loop risolto è davvero concluso solo quando non arriva soltanto il messaggio di test, ma anche i destinatari sconosciuti e i percorsi di mailflow alternativi terminano in modo definito. È proprio qui che si verificano la maggior parte delle ricadute.

## Cronologia delle modifiche

- **25 ottobre 2026:** Microsoft ha ritirato l'articolo di supporto relativo allo scenario di sottodominio e EdgeSync; la fonte rimanda ora alla versione archiviata del 19 aprile 2026.

## Fonti

1. [Microsoft Learn – Fix NDR error 5.4.6 or 5.4.14 in Exchange Online](https://learn.microsoft.com/en-us/troubleshoot/exchange/email-delivery/ndr/fix-error-code-5-4-6-through-5-4-20-in-exchange-online): significato degli NDR di Exchange e cause tipiche nei domini accettati e nei connettori ibridi.

2. [Microsoft Learn – Manage accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains): differenze tra `Authoritative` e `InternalRelay`.

3. [Microsoft Learn – Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains): responsabilità, domini relay e Recipient Lookup in Exchange locale.

4. [Microsoft Learn – Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing): percorsi di trasporto previsti con e senza Centralized Mail Transport.

5. [Microsoft Learn – Validate connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/validate-connectors): convalida dei connettori e indicazioni su più connettori applicabili contemporaneamente.

6. [Microsoft Learn – Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud): schemi di mailflow supportati con servizi di terze parti a monte.

7. [Microsoft Learn – Mail flow rules in Exchange Online](https://learn.microsoft.com/en-us/exchange/security-and-compliance/mail-flow-rules/mail-flow-rules): elaborazione, priorità, azioni ed eccezioni delle regole di trasporto.

8. [Microsoft Learn – Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2?view=exchange-ps): ricerca di messaggi nel trasporto di Exchange Online.

9. [Microsoft Learn – Search message tracking logs](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/search-message-tracking-logs): tracciamento locale dei messaggi su tutti i server Exchange.

10. [Microsoft Learn – Hop count exceeded for an on-premises application subdomain (versione archiviata del 19 aprile 2026)](https://web.archive.org/web/20260419203247/https://learn.microsoft.com/en-us/troubleshoot/exchange/mailflow/hop-count-exceeded-possible-mail-loop): scenario documentato di sottodominio/EdgeSync con dominio Internal Relay esplicito; Microsoft ha nel frattempo ritirato l'articolo.
