---
title: "Cause tipiche dei mail loop e come risolverle"
navTitle: "Risolvere i mail loop"
description: "Come individuare e risolvere sistematicamente i mail loop SMTP in Exchange Online, ambienti ibridi e mail gateway a monte mediante NDR, intestazioni, Message Trace, oggetti destinatario e connector."
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
translationSourceHash: c71063cb6e7d05a1f311a5269e4d6805d8b219e8d0fb103485738925ef99f990
translationModel: gpt-5.6-terra
translatedAt: 2026-09-01T09:01:55.513Z
translationReview: automatic
url: https://rafaelpfister.ch/it/blog/cause-tipiche-dei-mail-loop-e-come-risolverle
---

# Cause tipiche dei mail loop e come risolverle

Un mail loop si verifica quando almeno due sistemi di trasporto si passano ripetutamente lo stesso messaggio. Nessuno dei sistemi si riconosce come destinazione finale, ma entrambi conoscono un hop successivo apparentemente adatto. Il ciclo termina solo quando un server rileva il superamento del numero consentito di stazioni di trasporto e genera un NDR.

In Exchange, due messaggi sono particolarmente indicativi:

- `554 5.4.6 Hop count exceeded - possible mail loop` viene in genere generato dall'Exchange locale.
- `554 5.4.14 Hop count exceeded - possible mail loop ATTR34` viene generato da Exchange Online.

Il limite di hop non è la causa, bensì la protezione contro una ripetizione infinita. Aumentarlo non risolve quindi nulla. Occorre individuare il punto in cui il messaggio, contrariamente all'architettura di destinazione, viene restituito a un sistema già attraversato.

## Riconoscere lo schema del ciclo nell'intestazione

L'NDR e le intestazioni complete dei messaggi originali devono essere salvati prima di qualsiasi modifica. Le righe `Received` si leggono dal basso verso l'alto: la riga più in basso è l'hop documentato più antico, quella più in alto il più recente.

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

Non ogni hostname Microsoft che compare più volte indica già un ciclo. Exchange Online elabora internamente i messaggi attraverso più ruoli di trasporto. È invece sospetto il ritorno ripetuto tra gli stessi confini amministrativi, ad esempio tra Exchange Online e un gateway locale. Timestamp, IP mittente, host ricevente e `Message-ID` aiutano a identificare chiaramente il giro.

Per la prima analisi è necessario rispondere a queste domande:

1. Quale sistema ha generato l'NDR?
2. Quali due o tre hop si ripetono?
3. Quale sistema avrebbe dovuto consegnare definitivamente il messaggio?
4. In base a quale decisione relativa a dominio, destinatario, connector o regola è stato inoltrato?
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
| `-RecipientAddress` | Filtra il trace in base all'indirizzo destinatario indicato |
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
| `-MessageTraceId` | ID trace univoco dal risultato di `Get-MessageTraceV2` |
| `-RecipientAddress` | Indirizzo del destinatario del risultato; necessario insieme all'ID trace per la query di dettaglio |
| `Format-Table … -AutoSize` | Adatta la larghezza delle colonne al contenuto affinché i dettagli degli eventi restino leggibili |

</details>

Successivamente vengono rilevati insieme dominio, destinatario e connector:

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
| `-IncludeTestModeConnectors` | Include nell'output anche i connector in modalità test |
| `Format-Table … -AutoSize` | Visualizzazione tabellare con larghezza delle colonne in base al contenuto |
| `Format-List …` | Visualizzazione elenco delle proprietà indicate, adatta a valori lunghi come gli elenchi di indirizzi |

</details>

Non è decisivo se un singolo oggetto sembra plausibile. Il tipo di dominio, il tipo effettivo di destinatario e il connector applicabile devono descrivere insieme la stessa destinazione.

## Diagnosi nell'Exchange locale

In un ambiente ibrido, lo stesso destinatario viene verificato anche localmente. Le query distinguono tra una vera casella di posta locale, una RemoteMailbox e un MailUser:

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
| `-ErrorAction SilentlyContinue` | Sopprime il messaggio di errore se l'oggetto non esiste nel rispettivo tipo; in tal caso la query non restituisce semplicemente alcun risultato |

</details>

Per il percorso di trasporto sono necessari i connector di invio e ricezione, nonché i log di tracking:

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
| `Where-Object { … }` | Limita l'elenco dei server ai server Mailbox e Hub Transport, ossia ai ruoli con log di tracking |
| `-Start` / `-End` | Finestra temporale per la ricerca nei log |
| `-Recipients $recipient` | Filtra gli eventi di tracking con questo indirizzo destinatario |
| `-ResultSize Unlimited` | Rimuove il limite predefinito di 1000 voci restituite |
| `Select-Object …` | Riduce l'output ai campi rilevanti per l'analisi del percorso |
| `Sort-Object Timestamp` | Ordina cronologicamente gli eventi di tutti i server |

</details>

Un `SEND` verso Exchange Online, seguito da un nuovo `RECEIVE` dello stesso messaggio da Exchange Online, rende visibile il ritorno. Con `MessageId` e `NetworkMessageId` si evita di confondere tra loro diversi messaggi di test.

## Le cause più frequenti in sintesi

| Schema | Causa tipica | Risoluzione |
| --- | --- | --- |
| I destinatari sconosciuti rimbalzano tra due sistemi | Il dominio accettato è impostato su `InternalRelay`, ma entrambe le parti inoltrano i destinatari sconosciuti | Definire una responsabilità univoca; in caso di consegna completa in EXO usare `Authoritative` oppure, per una split domain, definire un unico hop finale |
| EXO invia all'Exchange locale, che lo rinvia subito a EXO | Il connector ibrido o Centralized Mail Transport non corrisponde più alla posizione della casella di posta | Verificare la configurazione HCW e `RouteAllMessagesViaOnPremises`; disattivare il routing centrale obsoleto o correggere la risoluzione locale dei destinatari |
| Il messaggio rimbalza tra EXO e un gateway di sicurezza, firma o crittografia | I messaggi di ritorno soddisfano nuovamente la regola in uscita | Usare come eccezione l'header impostato dal gateway o il meccanismo documentato di prevenzione dei loop; autenticare in modo univoco i connector in entrata e in uscita |
| È interessato un solo destinatario | `targetAddress` obsoleto o errato, tipo RemoteMailbox errato o indirizzi proxy in conflitto | Determinare la Source of Authority, correggere l'oggetto destinatario lì e sincronizzarlo |
| Vengono coinvolti solo messaggi inoltrati | Una regola di trasporto, un inoltro della casella di posta o una regola Inbox indirizza nuovamente al percorso originale | Disattivare la regola, correggere la destinazione e definire un'eccezione affidabile |
| È interessato solo un sottodominio o un'applicazione | Il dominio padre non copre correttamente il sottodominio nel percorso del connector previsto | Configurare esplicitamente il sottodominio come dominio accettato e nel Send Connector appropriato |
| Tutti i messaggi vanno in loop dopo una modifica al gateway o al DNS | Lo Smart Host o l'MX punta all'ingresso del sistema mittente | Correggere il next hop e verificare separatamente le destinazioni DNS, NAT e Load Balancer |

## Causa 1: tipo errato del dominio accettato

Un dominio authoritative significa: tutti i destinatari validi di questo dominio sono noti nell'organizzazione Exchange; i destinatari sconosciuti vengono rifiutati. Un dominio Internal Relay significa: una parte dei destinatari si trova in un altro sistema e deve essere inoltrata tramite un Send Connector o un Outbound Connector.

La configurazione problematica si verifica quando Exchange Online invia destinatari sconosciuti a un sistema locale e quest'ultimo non gestisce definitivamente lo stesso dominio, ma lo restituisce a Exchange Online tramite MX o Smart Host.

```powershell
Get-AcceptedDomain -Identity contoso.com |
    Format-List DomainName,DomainType,MatchSubDomains
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Identity contoso.com` | Seleziona il dominio accettato da verificare |
| `Format-List …` | Mostra in elenco nome del dominio, tipo di dominio e copertura dei sottodomini |

</details>

Se, al termine di una migrazione, tutti i destinatari si trovano in Exchange Online, `Authoritative` è in genere lo stato di destinazione corretto:

```powershell
# Eseguire solo dopo aver completato la verifica dei destinatari e del routing.
Set-AcceptedDomain -Identity contoso.com -DomainType Authoritative
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Identity contoso.com` | Il dominio accettato da modificare |
| `-DomainType Authoritative` | Imposta il dominio su authoritative: i destinatari sconosciuti vengono rifiutati anziché inoltrati |

</details>

In una vera split domain, `InternalRelay` può essere corretto. Tuttavia, è allora necessario un connector chiaro verso il sistema che conosce i destinatari rimanenti. Questa destinazione non deve rinviare gli indirizzi sconosciuti al punto di partenza.

## Causa 2: connector ibridi sovrapposti e Centralized Mail Transport

Centralized Mail Transport instrada intenzionalmente i messaggi in uscita da Exchange Online attraverso l'Exchange locale. Ciò è utile per determinati requisiti di compliance, ma crea ulteriori percorsi di trasporto. Se l'opzione rimane attiva dopo una migrazione, benché il sistema locale invii messaggi di nuovo a Exchange Online tramite il proprio MX, può formarsi un circuito.

```powershell
Get-OutboundConnector -IncludeTestModeConnectors |
    Format-Table Name,Enabled,ConnectorType,RouteAllMessagesViaOnPremises,
        RecipientDomains,SmartHosts,UseMXRecord -AutoSize
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-IncludeTestModeConnectors` | Include nell'output anche i connector in modalità test |
| `Format-Table … -AutoSize` | Visualizzazione tabellare delle proprietà di routing con larghezza delle colonne in base al contenuto |

</details>

Devono essere verificati anche più connector con ambiti sovrapposti. Microsoft raccomanda un connector on-premises dedicato per il flusso di posta ibrido; una riparazione tramite Hybrid Configuration Wizard è spesso più sicura rispetto a singole modifiche isolate.

Se Centralized Mail Transport non è più comprovabilmente necessario, l'impostazione può essere disattivata in modo mirato:

```powershell
# Solo dopo aver verificato i requisiti di compliance e del gateway.
Set-OutboundConnector `
    -Identity "Outbound to On-Premises" `
    -RouteAllMessagesViaOnPremises:$false
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Identity "Outbound to On-Premises"` | L'Outbound Connector da modificare |
| `-RouteAllMessagesViaOnPremises:$false` | Disattiva Centralized Mail Transport: i messaggi in uscita da Exchange Online non passano più attraverso l'Exchange locale |

</details>

## Causa 3: un gateway elabora nuovamente i propri messaggi di ritorno

In uno scenario in-and-out, Exchange Online invia un messaggio a un servizio aggiuntivo per la firma, la crittografia o l'archiviazione. Quest'ultimo lo restituisce poi a Exchange Online. La regola in uscita deve riconoscere il messaggio di ritorno; altrimenti verrà inviato nuovamente al servizio.

La verifica inizia con tutte le regole che selezionano connector, reindirizzano destinatari o valutano header:

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
| `Sort-Object Priority` | Ordina le regole in base alla loro sequenza di valutazione |
| `Format-List …` | Mostra le proprietà che selezionano connector, reindirizzano destinatari o impostano header oppure li valutano come eccezione |

</details>

L'eccezione concreta deve seguire la documentazione del produttore del gateway. È comune un header impostato dal servizio che non possa essere falsificato in modo affidabile da Internet. Inoltre, gli Inbound Connector dovrebbero identificare il servizio tramite certificato o IP mittente fisso. Un'eccezione generalizzata per tutti i messaggi apparentemente «interni» è troppo ampia.

## Causa 4: l'oggetto destinatario e la casella di posta effettiva non si trovano nello stesso luogo

Un oggetto può apparire in Exchange Online come `MailUser`, anche se la casella di posta attiva si trova localmente. In un ambiente ibrido sincronizzato, questo non è automaticamente un duplicato. Neppure una `ExternalEmailAddress`, che corrisponde all'indirizzo SMTP primario, dimostra da sola una configurazione errata.

È determinante la combinazione di tutte le query:

- `Get-Mailbox` restituisce un risultato localmente: la casella di posta attiva si trova localmente.
- `Get-RemoteMailbox` restituisce un risultato localmente: la destinazione gestita si trova in Exchange Online.
- `Get-EXOMailbox` restituisce un risultato: nel cloud esiste una vera casella di posta.
- `Get-EXORecipient` restituisce solo un MailUser: l'oggetto è una destinazione di routing, non una casella di posta cloud.

Sono problematici gli oggetti obsoleti dopo una migrazione, domini di routing remoto errati o valori `targetAddress` impostati manualmente, il cui dominio riconduce allo stesso percorso di trasporto. Le modifiche vengono effettuate nella Source of Authority: negli ambienti sincronizzati, quindi con gli strumenti di gestione Exchange localmente e non modificando direttamente singoli attributi in Exchange Online.

## Causa 5: inoltri e regole di trasporto formano un circuito

Una regola può reindirizzare dall'indirizzo A a B, mentre B invia nuovamente ad A tramite una seconda regola, un inoltro della casella di posta o un sistema esterno. Questi loop riguardano spesso solo singoli destinatari o tipi di messaggio.

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
| `Sort-Object Priority` | Ordina le regole di trasporto in base alla loro sequenza di valutazione |
| `-ResultSize Unlimited` | Rimuove il limite predefinito di 1000 caselle di posta restituite |
| `-Mailbox user01@contoso.com` | Casella di posta di cui vengono interrogate le regole Inbox |
| `Select-Object …` | Riduce l'output alle destinazioni di inoltro e reindirizzamento |

</details>

La risoluzione non consiste solo nel disattivare temporaneamente una regola. L'intera catena deve essere eliminata e le regole per servizi esterni necessitano di un'eccezione che riconosca con certezza i messaggi già elaborati.

## Causa 6: MX, Smart Host o sottodominio punta indietro

Un gateway può richiedere internamente un next hop diverso da quello degli mittenti esterni. Se per l'inoltro usa semplicemente l'MX pubblico, quest'ultimo potrebbe puntare nuovamente al gateway stesso. Lo stesso problema si verifica quando uno Smart Host, tramite NAT o Load Balancing, riconduce al proprio listener.

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
| `Format-List …` | Mostra per ogni Send Connector gli spazi di indirizzi, la modalità di routing e gli Smart Host |

</details>

I sottodomini meritano una verifica separata. Microsoft documenta casi in cui un sottodominio applicativo deve essere creato esplicitamente come dominio Internal Relay e sincronizzato con i sistemi Edge:

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
| `-DomainName app.contoso.com` | Il dominio SMTP per cui Exchange accetta messaggi |
| `-DomainType InternalRelay` | Una parte dei destinatari si trova al di fuori dell'organizzazione; i destinatari sconosciuti vengono inoltrati tramite un Send Connector anziché rifiutati |

</details>

Questi comandi non sono una correzione universale. Sono adatti solo se `app.contoso.com` viene effettivamente consegnato al di fuori dell'organizzazione Exchange e il Send Connector dispone di un next hop univoco.

## Procedura sicura in caso di ciclo attivo

Durante l'interruzione occorre innanzitutto fermare la moltiplicazione. A seconda dell'architettura, la regola di trasporto che lo provoca o il connector specifico viene disattivato in modo controllato, oppure il gateway trattiene la coda interessata. Prima di farlo, vengono esportati configurazione ed esempi di messaggi.

Segue quindi un test con esattamente un mittente, un destinatario e un oggetto chiaramente riconoscibile. Il messaggio viene tracciato senza interruzioni attraverso header, Message Trace e log di tracking locali. Solo quando termina alla destinazione prevista, il flusso di posta viene riaperto gradualmente.

Non sono consigliati:

- aumentare i limiti di hop
- modificare contemporaneamente più connector
- commutare per tentativi i domini accettati tra `Authoritative` e `InternalRelay`
- reinserire ripetutamente una coda problematica senza verificarla
- correggere direttamente in AD o Exchange Online gli attributi Exchange sincronizzati
- disattivare i controlli TLS, IP o certificato come presunta soluzione rapida

## Controllo finale

Dopo la correzione, la documentazione dovrebbe contenere esattamente un'affermazione per ogni dominio rilevante: quale sistema conosce il destinatario, quale connector è applicabile e quale host è il next hop finale?

Il collaudo tecnico comprende almeno:

- messaggio di test esterno e interno
- destinatario sconosciuto dello stesso dominio
- destinatario su ciascun lato di una vera split domain
- messaggio in uscita con gateway o Centralized Mail Transport attivo
- header senza sequenza di hop ricorrente
- Message Trace con `Delivered` oppure il passaggio previsto
- tracking locale senza un nuovo `RECEIVE` dopo un `SEND` verso la stessa destinazione
- convalida dei connector per tutti i connector ancora necessari

Un mail loop risolto è concluso solo quando non soltanto l'e-mail di test arriva, ma anche i destinatari sconosciuti e i percorsi alternativi del flusso di posta terminano in modo definito. È proprio qui che si verificano la maggior parte delle ricadute.

## Fonti

1. [Microsoft Learn – Fix NDR error 5.4.6 or 5.4.14 in Exchange Online](https://learn.microsoft.com/en-us/troubleshoot/exchange/email-delivery/ndr/fix-error-code-5-4-6-through-5-4-20-in-exchange-online): significato degli NDR di Exchange e cause tipiche nei domini accettati e nei connector ibridi.

2. [Microsoft Learn – Manage accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains): differenze tra `Authoritative` e `InternalRelay`.

3. [Microsoft Learn – Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains): responsabilità, domini relay e ricerca dei destinatari nell'Exchange locale.

4. [Microsoft Learn – Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing): percorsi di trasporto previsti con e senza Centralized Mail Transport.

5. [Microsoft Learn – Validate connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/validate-connectors): convalida dei connector e indicazioni su più connector applicabili contemporaneamente.

6. [Microsoft Learn – Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud): schemi di flusso di posta supportati con servizi cloud di terze parti a monte.

7. [Microsoft Learn – Mail flow rules in Exchange Online](https://learn.microsoft.com/en-us/exchange/security-and-compliance/mail-flow-rules/mail-flow-rules): elaborazione, priorità, azioni ed eccezioni delle regole di trasporto.

8. [Microsoft Learn – Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2?view=exchange-ps): ricerca di messaggi nel trasporto di Exchange Online.

9. [Microsoft Learn – Search message tracking logs](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/search-message-tracking-logs): tracciamento locale dei messaggi su tutti i server Exchange.

10. [Microsoft Learn – Hop count exceeded for an on-premises application subdomain](https://learn.microsoft.com/en-us/troubleshoot/exchange/mailflow/hop-count-exceeded-possible-mail-loop): scenario documentato di sottodominio/EdgeSync con dominio Internal Relay esplicito.
