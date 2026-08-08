---
title: "Cause tipiche dei mail loop e come risolverle"
navTitle: "Risolvere i mail loop"
description: "Come individuare e risolvere sistematicamente i mail loop SMTP in Exchange Online, ambienti ibridi e gateway di posta a monte mediante NDR, header, Message Trace, oggetti destinatario e connettori."
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
url: https://rafaelpfister.ch/it/blog/cause-tipiche-dei-mail-loop-e-come-risolverle
translationSourceHash: 5353684681217adafc789a3b28ec218fa927e18d801c82c437ae281e1e1017bd
translationModel: gpt-5.6-terra
translatedAt: 2026-08-08T14:00:11.649Z
translationReview: automatic
---

# Cause tipiche dei mail loop e come risolverle

Un mail loop si verifica quando almeno due sistemi di trasporto si consegnano ripetutamente lo stesso messaggio. Nessuno dei sistemi si riconosce come destinazione finale, ma entrambi conoscono un hop successivo apparentemente adatto. Il ciclo termina soltanto quando un server rileva il superamento del numero consentito di stazioni di trasporto e genera un NDR.

Con Exchange, due messaggi sono particolarmente indicativi:

- `554 5.4.6 Hop count exceeded - possible mail loop` viene tipicamente generato dall'Exchange locale.
- `554 5.4.14 Hop count exceeded - possible mail loop ATTR34` viene generato da Exchange Online.

Il limite degli hop non è la causa, bensì la protezione contro una ripetizione infinita. Aumentarlo non risolve quindi nulla. Occorre individuare il punto in cui il messaggio, contrariamente all'architettura di destinazione, viene restituito a un sistema già attraversato.

## Riconoscere lo schema del ciclo nell'header

L'NDR e gli header completi del messaggio originale devono essere salvati prima di qualsiasi modifica. Le righe `Received` vengono lette dal basso verso l'alto: la riga più in basso è l'hop documentato più antico, quella più in alto il più recente.

Un ciclo si manifesta solitamente come una sequenza ricorrente:

```text
Internet
  → Exchange Online Protection
  → Mailgateway
  → Exchange Online Protection
  → Mailgateway
  → Exchange Online Protection
  → ...
```

Non tutti gli hostname Microsoft che compaiono più volte indicano già un ciclo. Exchange Online elabora internamente i messaggi attraverso diversi ruoli di trasporto. Sospetto è il ritorno ripetuto tra gli stessi confini amministrativi, ad esempio tra Exchange Online e un gateway locale. Timestamp, IP mittente, host ricevente e `Message-ID` aiutano a identificare con certezza il giro.

Per la prima analisi occorre rispondere a queste domande:

1. Quale sistema ha generato l'NDR?
2. Quali due o tre hop si ripetono?
3. Quale sistema avrebbe dovuto consegnare definitivamente il messaggio?
4. In base a quale decisione relativa a dominio, destinatario, connettore o regola è stato inoltrato?
5. Quale modifica ha influenzato per ultima il flusso di posta?

## Diagnosi in Exchange Online

Con `Get-MessageTraceV2` è possibile esaminare l'elaborazione degli ultimi 90 giorni; per ogni query sono consentiti al massimo dieci giorni. Una finestra temporale ristretta e l'indirizzo concreto del destinatario forniscono i risultati più utili:

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

I dettagli di un risultato mostrano i singoli eventi di trasporto:

```powershell
$trace | ForEach-Object {
    Get-MessageTraceDetailV2 `
        -MessageTraceId $_.MessageTraceId `
        -RecipientAddress $_.RecipientAddress
} | Format-Table Date,Event,Action,Detail -AutoSize
```

Successivamente vengono raccolti insieme dominio, destinatario e connettori:

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

Non è decisivo se un singolo oggetto sembra plausibile. Il tipo di dominio, il tipo effettivo di destinatario e il connettore applicabile devono descrivere insieme la stessa destinazione.

## Diagnosi nell'Exchange locale

In un ambiente ibrido, lo stesso destinatario viene verificato anche localmente. Le query distinguono tra una vera cassetta postale locale, una RemoteMailbox e un MailUser:

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

Per il percorso di trasporto sono necessari i connettori di invio e ricezione nonché i log di tracking:

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

Un `SEND` verso Exchange Online, seguito da un nuovo `RECEIVE` dello stesso messaggio da Exchange Online, rende visibile la restituzione. Con `MessageId` e `NetworkMessageId` è possibile evitare di confondere messaggi di test diversi.

## Le cause più frequenti in sintesi

| Schema | Causa tipica | Risoluzione |
| --- | --- | --- |
| I destinatari sconosciuti rimbalzano tra due sistemi | Il dominio accettato è impostato su `InternalRelay`, ma entrambe le parti inoltrano destinatari sconosciuti | Definire una responsabilità univoca; con consegna completa in EXO usare `Authoritative` oppure, per uno Split Domain, definire un unico hop finale |
| EXO invia all'Exchange locale, che restituisce subito a EXO | Il connettore ibrido o Centralized Mail Transport non corrisponde più alla posizione della mailbox | Verificare la configurazione HCW e `RouteAllMessagesViaOnPremises`; disattivare il routing centrale obsoleto o correggere la risoluzione locale dei destinatari |
| Il messaggio rimbalza tra EXO e un gateway di sicurezza, firma o crittografia | I messaggi restituiti soddisfano nuovamente la regola in uscita | Usare come eccezione l'header impostato dal gateway o il meccanismo documentato di prevenzione dei loop; autenticare in modo univoco i connettori in entrata e in uscita |
| È interessato un solo destinatario | `targetAddress` obsoleto o errato, tipo RemoteMailbox errato oppure indirizzi proxy contraddittori | Determinare la Source of Authority, correggere l'oggetto destinatario in tale posizione e sincronizzare |
| Solo i messaggi inoltrati entrano nel ciclo | Una regola di trasporto, un inoltro della mailbox o una regola Inbox indirizza nuovamente al percorso originario | Disattivare la regola, correggere la destinazione e definire un'eccezione affidabile |
| È interessata solo una sottodomain o un'applicazione | Il dominio padre non copre correttamente la sottodomain nel percorso previsto del connettore | Configurare esplicitamente la sottodomain come dominio accettato e nel Send Connector appropriato |
| Tutti i messaggi entrano nel ciclo dopo una modifica al gateway o al DNS | Smart Host o MX punta all'ingresso del sistema mittente | Correggere il Next Hop e verificare separatamente le destinazioni DNS, NAT e del load balancer |

## Causa 1: tipo errato del dominio accettato

Un dominio authoritative significa che tutti i destinatari validi di tale dominio sono conosciuti nell'organizzazione Exchange; i destinatari sconosciuti vengono rifiutati. Un dominio Internal Relay significa che una parte dei destinatari si trova in un altro sistema e deve essere inoltrata tramite un connettore di invio o in uscita.

La configurazione problematica si verifica quando Exchange Online invia destinatari sconosciuti a un sistema locale e quest'ultimo non gestisce definitivamente lo stesso dominio, ma lo rinvia a Exchange Online tramite MX o Smart Host.

```powershell
Get-AcceptedDomain -Identity contoso.com |
    Format-List DomainName,DomainType,MatchSubDomains
```

Se, al termine di una migrazione, tutti i destinatari si trovano in Exchange Online, `Authoritative` è di solito lo stato finale corretto:

```powershell
# Eseguire solo dopo aver completato la verifica di destinatari e routing.
Set-AcceptedDomain -Identity contoso.com -DomainType Authoritative
```

In un vero Split Domain, `InternalRelay` può essere corretto. È tuttavia necessario un connettore chiaro verso il sistema che conosce i destinatari rimanenti. Questa destinazione non deve rinviare indirizzi sconosciuti al punto di origine.

## Causa 2: connettori ibridi sovrapposti e Centralized Mail Transport

Centralized Mail Transport instrada deliberatamente i messaggi in uscita di Exchange Online attraverso l'Exchange locale. Questo è utile per determinati requisiti di conformità, ma crea percorsi di trasporto aggiuntivi. Se l'opzione rimane attiva dopo una migrazione, mentre il sistema locale invia nuovamente i messaggi a Exchange Online tramite il proprio MX, può formarsi un ciclo.

```powershell
Get-OutboundConnector -IncludeTestModeConnectors |
    Format-Table Name,Enabled,ConnectorType,RouteAllMessagesViaOnPremises,
        RecipientDomains,SmartHosts,UseMXRecord -AutoSize
```

Anche più connettori con ambiti sovrapposti sono sospetti. Microsoft raccomanda un connettore On-Premises dedicato per il flusso di posta ibrido; una riparazione tramite Hybrid Configuration Wizard è spesso più sicura di singole modifiche isolate.

Se Centralized Mail Transport non è più dimostrabilmente necessario, l'impostazione può essere disattivata in modo mirato:

```powershell
# Solo dopo aver verificato i requisiti di conformità e del gateway.
Set-OutboundConnector `
    -Identity "Outbound to On-Premises" `
    -RouteAllMessagesViaOnPremises:$false
```

## Causa 3: un gateway elabora nuovamente i propri messaggi restituiti

In uno scenario in-and-out, Exchange Online invia un messaggio a un servizio aggiuntivo per firma, crittografia o archiviazione. Questo lo restituisce poi a Exchange Online. La regola in uscita deve riconoscere il messaggio restituito; altrimenti viene inviato nuovamente al servizio.

La verifica inizia con tutte le regole che selezionano connettori, reindirizzano destinatari o valutano header:

```powershell
Get-TransportRule |
    Sort-Object Priority |
    Format-List Name,State,Mode,Priority,FromScope,SentToScope,
        RedirectMessageTo,RouteMessageOutboundConnector,
        SetHeaderName,SetHeaderValue,ExceptIfHeaderContainsMessageHeader,
        ExceptIfHeaderContainsWords
```

L'eccezione concreta deve seguire la documentazione del produttore del gateway. È comune un header impostato dal servizio e non falsificabile in modo affidabile da Internet. Inoltre, i connettori in entrata dovrebbero identificare il servizio tramite certificato o IP mittente fisso. Un'eccezione generica per tutti i messaggi apparentemente «interni» è troppo ampia.

## Causa 4: l'oggetto destinatario e la mailbox effettiva non si trovano nello stesso posto

Un oggetto può comparire in Exchange Online come `MailUser`, benché la mailbox attiva si trovi localmente. In un ambiente ibrido sincronizzato, ciò non costituisce automaticamente un duplicato. Nemmeno un `ExternalEmailAddress`, corrispondente all'indirizzo SMTP primario, dimostra da solo una configurazione errata.

È determinante la combinazione di tutte le query:

- `Get-Mailbox` in locale restituisce un risultato: la mailbox attiva si trova localmente.
- `Get-RemoteMailbox` in locale restituisce un risultato: la destinazione gestita si trova in Exchange Online.
- `Get-EXOMailbox` restituisce un risultato: nel cloud esiste una vera mailbox.
- `Get-EXORecipient` restituisce solo un MailUser: l'oggetto è una destinazione di routing, non una mailbox cloud.

Sono problematici gli oggetti obsoleti dopo una migrazione, domini di routing remoto errati o valori `targetAddress` impostati manualmente, il cui dominio riconduce indietro attraverso lo stesso percorso di trasporto. Le modifiche vengono effettuate nella Source of Authority: negli ambienti sincronizzati quindi con gli strumenti di gestione Exchange locali e non modificando direttamente singoli attributi in Exchange Online.

## Causa 5: inoltri e regole di trasporto formano un cerchio

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

La risoluzione non consiste solo nel disattivare temporaneamente una regola. L'intera catena deve essere sciolta e le regole per i servizi esterni richiedono un'eccezione che riconosca in modo sicuro i messaggi già elaborati.

## Causa 6: MX, Smart Host o sottodomain punta indietro

Un gateway può richiedere internamente un Next Hop diverso rispetto ai mittenti esterni. Se per l'inoltro usa semplicemente l'MX pubblico, questo potrebbe puntare di nuovo al gateway stesso. Lo stesso problema si verifica quando uno Smart Host, tramite NAT o load balancing, riconduce al proprio listener.

```powershell
Resolve-DnsName -Type MX contoso.com
Resolve-DnsName -Type MX app.contoso.com

Get-SendConnector |
    Format-List Name,AddressSpaces,DNSRoutingEnabled,SmartHosts
```

Le sottodomain meritano una verifica separata. Microsoft documenta casi in cui una sottodomain applicativa deve essere creata esplicitamente come dominio Internal Relay e sincronizzata con i sistemi Edge:

```powershell
New-AcceptedDomain `
    -Name "app.contoso.com" `
    -DomainName app.contoso.com `
    -DomainType InternalRelay

Start-EdgeSynchronization
```

Questi comandi non sono una soluzione universale. Sono adatti solo se `app.contoso.com` viene effettivamente consegnato al di fuori dell'organizzazione Exchange e il Send Connector possiede un hop successivo univoco.

## Procedura sicura in presenza di un ciclo attivo

Durante il guasto, occorre innanzitutto fermare la moltiplicazione. A seconda dell'architettura, la regola di trasporto che lo innesca o il connettore specifico viene disattivato in modo controllato, oppure il gateway trattiene la coda interessata. Prima vengono esportate la configurazione e gli esempi di messaggi.

Segue quindi un test con esattamente un mittente, un destinatario e un oggetto facilmente riconoscibile. Il messaggio viene tracciato senza interruzioni tramite header, Message Trace e log di tracking locali. Solo quando termina alla destinazione prevista, il flusso di posta viene riaperto gradualmente.

Non sono consigliabili:

- aumentare i limiti degli hop
- modificare più connettori contemporaneamente
- commutare per tentativi i domini accettati tra `Authoritative` e `InternalRelay`
- reinserire ripetutamente una coda problematica senza verifica
- correggere direttamente in AD o Exchange Online gli attributi Exchange sincronizzati
- disattivare verifiche TLS, IP o dei certificati come presunta soluzione rapida

## Controllo finale

Dopo la correzione, la documentazione dovrebbe contenere esattamente una dichiarazione per ogni dominio rilevante: quale sistema conosce il destinatario, quale connettore è applicabile e quale host è l'hop successivo finale?

L'accettazione tecnica include almeno:

- messaggio di test esterno e interno
- destinatario sconosciuto dello stesso dominio
- destinatario su ciascun lato di un vero Split Domain
- messaggio in uscita con gateway o Centralized Mail Transport attivato
- header senza sequenza di hop ricorrente
- Message Trace con `Delivered` o consegna prevista
- tracking locale senza un nuovo `RECEIVE` dopo un `SEND` verso la stessa destinazione
- convalida dei connettori per tutti i connettori ancora necessari

Un mail loop risolto è concluso solo quando non arriva soltanto il messaggio di test, ma anche i destinatari sconosciuti e i percorsi alternativi del flusso di posta terminano in modo definito. È proprio qui che si verificano la maggior parte delle ricadute.

## Fonti

1. [Microsoft Learn – Fix NDR error 5.4.6 or 5.4.14 in Exchange Online](https://learn.microsoft.com/en-us/troubleshoot/exchange/email-delivery/ndr/fix-error-code-5-4-6-through-5-4-20-in-exchange-online): significato degli NDR di Exchange e cause tipiche nei domini accettati e nei connettori ibridi.

2. [Microsoft Learn – Manage accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains): differenze tra `Authoritative` e `InternalRelay`.

3. [Microsoft Learn – Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains): responsabilità, domini relay e Recipient Lookup nell'Exchange locale.

4. [Microsoft Learn – Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing): percorsi di trasporto previsti con e senza Centralized Mail Transport.

5. [Microsoft Learn – Validate connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/validate-connectors): convalida dei connettori e indicazioni su più connettori contemporaneamente applicabili.

6. [Microsoft Learn – Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud): modelli di flusso di posta supportati con servizi di terze parti a monte.

7. [Microsoft Learn – Mail flow rules in Exchange Online](https://learn.microsoft.com/en-us/exchange/security-and-compliance/mail-flow-rules/mail-flow-rules): elaborazione, priorità, azioni ed eccezioni delle regole di trasporto.

8. [Microsoft Learn – Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2?view=exchange-ps): ricerca di messaggi nel trasporto di Exchange Online.

9. [Microsoft Learn – Search message tracking logs](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/search-message-tracking-logs): tracciamento locale dei messaggi su tutti i server Exchange.

10. [Microsoft Learn – Hop count exceeded for an on-premises application subdomain](https://learn.microsoft.com/en-us/troubleshoot/exchange/mailflow/hop-count-exceeded-possible-mail-loop): scenario documentato di sottodomain/EdgeSync con dominio Internal Relay esplicito.
