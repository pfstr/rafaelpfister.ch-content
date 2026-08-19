---
title: "Analizzare il flusso di posta in Exchange: Message Tracking, protocolli SMTP e connettori di ricezione"
navTitle: "Analizzare il flusso di posta"
description: "Come determinare sistematicamente in Exchange OnPrem, Hybrid ed Exchange Online dove si è fermato un messaggio: query con output di esempio, lettura corretta del protocollo SMTP e insidie che portano regolarmente su piste false."
date: "2026-08-11"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "22 min di lettura"
themen:
  - exchange-onprem-hybrid
  - smtp-mailflow
  - microsoft-365-exchange
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange"
  - "exchange-on-premises"
  - "exchange-hybrid"
  - "exchange-online"
protokolle:
  - "smtp"
  - "powershell"
  - "troubleshooting"
related:
  - typische-ursachen-fuer-mail-loops-und-deren-behebung
  - einliefernde-ip-adressen-aggregieren
  - mailflow-fehlersuche-kontrollierte-experimente
slug: "analizzare-il-flusso-di-posta-in-exchange-message-tracking-protocolli-smtp-e-connettori-di"
translationId: "article-ad93c41ab6cd20e6"
draft: false
translationOf: exchange-message-tracking-und-receive-connectoren-analysieren
url: https://rafaelpfister.ch/it/blog/analizzare-il-flusso-di-posta-in-exchange-message-tracking-protocolli-smtp-e-connettori-di
translationSourceHash: 646cb713e4dd97300a2cd068ee8f04953f2e80a99aec63ed11eddc46e1981f13
translationModel: gpt-5.6-terra
translatedAt: 2026-08-12T05:16:25.287Z
translationReview: automatic
---

# Analizzare il flusso di posta in Exchange: Message Tracking, protocolli SMTP e connettori di ricezione

La domanda più frequente nella gestione della posta è: un messaggio non è arrivato, dove si è fermato? Il Message Tracking risponde in modo affidabile, ma solo se sapete cosa **non** vi dice. Questo articolo descrive la procedura nell’ordine che si è dimostrato efficace, mostra l’output tipico per ogni query e indica le insidie che costano regolarmente ore perché suggeriscono conclusioni plausibili, ma errate.

Tutti gli esempi utilizzano nomi generici: `SRV-MAIL01` e `SRV-MAIL02` come server di trasporto, `example.com` come dominio. Se volete comporre i comandi per il vostro ambiente invece di digitarli: il [Generatore di comandi](https://rafaelpfister.ch/tools/command-builder) contiene i comuni comandi di Message Tracking e cattura per PowerShell e shell Unix affiancati, interamente in locale nel browser.

## Il principio: prima localizzare, poi spiegare

L’istinto è cercare subito la causa. È più efficiente determinare innanzitutto fino a che punto sia arrivato il messaggio. Questo restringe drasticamente lo spazio di ricerca in un solo passaggio, poiché saprete poi se cercare nel vostro sistema, presso il gateway a monte o presso la destinazione.

L’ordine è quindi: trovare il messaggio, leggere l’ultimo evento, leggere il motivo dell’errore, stabilire se si tratta di un caso isolato o di uno schema ricorrente e solo allora ricostruire il percorso di consegna.

## Passo 1: trovare il messaggio

Iniziate dal destinatario, perché quasi sempre lo conoscete. È importante eseguire la query su **tutti** i server di trasporto, non solo su uno.

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object {
    Get-MessageTrackingLog -Server $_ `
        -Start (Get-Date).AddHours(-6) `
        -ResultSize Unlimited `
        -Recipients "empfaenger@example.com"
} | Sort-Object Timestamp |
    Format-Table Timestamp, ServerHostname, EventId, Source, ConnectorId, MessageId `
        -AutoSize -Wrap
```

Un output tipico per un messaggio transitato correttamente:

```text
Timestamp           ServerHostname EventId      Source  ConnectorId
---------           -------------- -------      ------  -----------
11.08.2026 08:27:15 SRV-MAIL02     HARECEIVE    SMTP
11.08.2026 08:27:15 SRV-MAIL01     RECEIVE      SMTP    SRV-MAIL01\Default SRV-MAIL01
11.08.2026 08:27:15 SRV-MAIL01     HAREDIRECT   SMTP
11.08.2026 08:27:15 SRV-MAIL01     RESOLVE      ROUTING
11.08.2026 08:27:15 SRV-MAIL01     AGENTINFO    AGENT
11.08.2026 08:27:16 SRV-MAIL01     SENDEXTERNAL SMTP    Outbound-to-O365
11.08.2026 08:27:53 SRV-MAIL02     HADISCARD    SMTP
```

Se la query non trova nulla, verificate se il destinatario è stato espanso tramite una lista di distribuzione. In tal caso è meglio cercare tramite il mittente:

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object {
    Get-MessageTrackingLog -Server $_ `
        -Start (Get-Date).AddHours(-6) `
        -ResultSize Unlimited
} | Where-Object { $_.Sender -like "*@example.com" } |
    Sort-Object Timestamp |
    Format-Table Timestamp, EventId, Sender,
        @{n='To'; e={$_.Recipients -join ','}}, MessageSubject -AutoSize -Wrap
```

## Passo 2: leggere l’ultimo evento

L’intera diagnosi dipende dall’**ultimo** `EventId` del messaggio. Vi indica quale spazio di ricerca affrontare successivamente.

| Ultimo EventId | Significato | Passo successivo |
|---|---|---|
| `RECEIVE`, poi nulla | Il messaggio è bloccato | Controllare le code |
| `SEND` o `SENDEXTERNAL` | consegnato correttamente | proseguire la ricerca al prossimo hop |
| `FAIL` | errore definitivo | leggere il motivo in `RecipientStatus` |
| `DEFER` | tentativo in corso | controllare coda e sistema di destinazione |
| `DROP` o `POISONMESSAGE` | scartato | regola di trasporto o agente |
| `DELIVER` | consegnato a una casella postale locale | controllare le regole della casella |
| `RESOLVE` | il destinatario è stato riscritto | leggere l’indirizzo di destinazione nella voce |

`RESOLVE` è il passaggio intermedio più significativo negli ambienti Hybrid, perché rende visibile la riscrittura verso l’indirizzo di routing cloud:

```text
EventId : RESOLVE
Source  : ROUTING
Sender  : absender@example.com
To      : BENUTZER@example.mail.onmicrosoft.com
```

Se compare l’indirizzo `onmicrosoft.com` previsto, l’oggetto destinatario è configurato correttamente e potete chiudere il caso. Se compare ancora l’indirizzo originale, nell’oggetto locale manca l’indirizzo di destinazione e Exchange tenta la consegna locale.

Se il messaggio è bloccato, la coda di solito mostra il motivo in chiaro:

```powershell
Get-Queue -Server SRV-MAIL01 |
    Where-Object { $_.MessageCount -gt 0 } |
    Format-Table Identity, DeliveryType, Status, MessageCount, NextHopDomain, LastError `
        -AutoSize -Wrap
```

## Insidia 1: il tracking è basato sul server e molte voci sono copie shadow

Se nell’output vedete coppie di `HARECEIVE` e `HADISCARD`, spesso con l’aggiunta `ExplicitlyDiscarded`, quel server **non ha elaborato** il messaggio. Ha solo mantenuto una copia shadow nel quadro della Shadow Redundancy, mentre un altro server ha eseguito la consegna effettiva. Non appena il server primario segnala il successo, il partner elimina la propria copia.

Ecco come appare se avete interrogato solo il server sbagliato:

```text
Timestamp           EventId    SourceContext
---------           -------    -------------
11.08.2026 09:47:13 HARECEIVE  1a2aae6b-f3a3-4e04-9233-7de460b92223
11.08.2026 09:48:07 HADISCARD  ExplicitlyDiscarded;1a2aae6b-f3a3-4e04-9233-7de460b92223
```

Due righe, nessun errore, nessuna consegna. Chi ne deduce che il messaggio sia scomparso sta cercando nel posto sbagliato. L’elaborazione effettiva si trova nel tracking del server partner.

In pratica ciò significa due cose. Primo, queste righe non indicano un problema, ma il normale funzionamento. Secondo, dovete obbligatoriamente interrogare tutti i server di trasporto.

## Insidia 2: `Format-Table` tronca proprio le colonne decisive

Il motivo dell’errore è in `RecipientStatus`, e questo campo è lungo. In una tabella viene omesso completamente oppure troncato. Proprio questo porta a vedere `FAIL` senza il motivo e a iniziare invece a indovinare.

Non appena trovate un caso di errore, passate quindi a `Format-List` ed espandete i campi raccolta:

```powershell
Get-MessageTrackingLog -Server SRV-MAIL01 `
    -Start (Get-Date).AddHours(-6) `
    -ResultSize Unlimited `
    -Recipients "empfaenger@example.com" `
    -EventId FAIL |
  Format-List Timestamp, Sender,
    @{n='To';     e={$_.Recipients -join ','}},
    @{n='Status'; e={$_.RecipientStatus -join ' | '}},
    MessageSubject, MessageId, SourceContext
```

Ecco la differenza. Prima la vista tabellare, che non spiega nulla:

```text
Timestamp           EventId ConnectorId
---------           ------- -----------
11.08.2026 09:47:13 FAIL    Outbound-to-O365
```

Poi lo stesso messaggio come elenco:

```text
Timestamp      : 11.08.2026 09:47:13
Sender         : dienst@example-test.com
To             : BENUTZER@example.mail.onmicrosoft.com
Status         : [{LED=550 5.1.8 Access denied, bad outbound sender AS(42000001)
                 [XX1PEPF00000000.eurprd02.prod.outlook.com]};{MSG=};
                 {FQDN=10.0.0.40};{IP=10.0.0.40};{LRT=11.08.2026 09:47:13}]
MessageSubject : Statusmeldung Nachtlauf
MessageId      : <1897281176.1319@app01.intern.example.com>
```

La diagnosi è quindi certa, senza dover formulare una sola ipotesi: la controparte contesta il mittente. `LED` contiene la risposta SMTP completa, `FQDN` e `IP` indicano il sistema che ha risposto e `LRT` il momento dell’ultimo tentativo.

## Passo 3: caso isolato o schema ricorrente?

Prima di approfondire un singolo caso, chiarite la portata. Questa singola query decide se avete a che fare con una nota marginale o con un incidente:

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object {
    Get-MessageTrackingLog -Server $_ `
        -Start (Get-Date).AddHours(-8) `
        -EventId FAIL -ResultSize Unlimited
} | Where-Object { ($_.RecipientStatus -join '') -like "*5.1.8*" } |
    Group-Object { ($_.Sender -split '@')[-1] } |
    Sort-Object Count -Descending |
    Format-Table Count, Name -AutoSize
```

Sostituite `5.1.8` con il codice di stato che state esaminando. L’output risponde alla domanda in una riga:

```text
Count Name
----- ----
    9 example-test.com
```

Un solo dominio mittente significa: problema circoscritto, non un incidente; potete continuare a cercare con calma. Se ci fossero venti domini diversi, avreste un’interruzione in corso e tutto il resto dovrebbe aspettare. Fare questa distinzione così presto fa risparmiare, per esperienza, la maggior parte del tempo.

## Insidia 3: `ConnectorId` non rivela il vero connettore di ricezione

Questa è l’insidia più costosa, perché l’output sembra attendibile. La posta consegnata da un client o da un sistema esterno sulla porta 25 raggiunge prima il **Front End Transport**. Quest’ultimo inoltra il messaggio al **Transport Service** sulla porta 2525. Il Message Tracking viene scritto solo lì; il Front End Transport non scrive un tracking proprio.

La conseguenza è visibile in questa riga:

```text
EventId        : RECEIVE
ConnectorId    : SRV-MAIL01\Default SRV-MAIL01
ClientIp       : 10.0.1.11
ClientHostname : srv-mail01.intern.example.com
```

`ConnectorId` indica il connettore interno sulla porta 2525 e `ClientIp` è l’indirizzo del **server proxy**, non quello dell’originatore della consegna. Il tracking semplicemente non indica quale dei connettori configurati sulla porta 25 sia stato effettivamente raggiunto da un sistema. Chi si fida di questa informazione cerca l’errore presso un connettore che non è nemmeno coinvolto.

Esistono due modi per arrivare alla risposta. Il primo è la ricostruzione tramite configurazione:

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object { Get-ReceiveConnector -Server $_ } |
    Format-List Identity, Enabled,
        @{n='Bindings';       e={$_.Bindings -join ','}},
        @{n='RemoteIPRanges'; e={$_.RemoteIPRanges -join ','}},
        PermissionGroups, AuthMechanism
```

```text
Identity         : SRV-MAIL01\Default Frontend SRV-MAIL01
Bindings         : 10.0.1.11:25
RemoteIPRanges   : 0.0.0.0-255.255.255.255
PermissionGroups : AnonymousUsers, ExchangeServers, ExchangeLegacyServers
AuthMechanism    : Tls, Integrated, BasicAuth, BasicAuthRequireTLS, ExchangeServer

Identity         : SRV-MAIL01\smtp-noauth SRV-MAIL01
Bindings         : 10.0.1.13:25
RemoteIPRanges   : 10.0.20.22,10.0.21.11,10.0.21.12
PermissionGroups : AnonymousUsers, Custom
AuthMechanism    : Tls
```

Determinate l’IP sorgente del sistema che consegna e cercate il connettore il cui `RemoteIPRanges` lo contiene. Se non rientra in nessuno dei connettori limitati, resta il connettore frontend predefinito, che di norma accetta l’intero spazio di indirizzi. Anche qui usate `Format-List`, poiché `RemoteIPRanges` e `PermissionGroups` vengono regolarmente troncati nelle tabelle.

Il secondo modo è il protocollo SMTP, che merita una sezione a sé.

## Il protocollo SMTP: l’unico luogo con tutta la verità

Il protocollo del Front End Transport registra la sessione SMTP completa: quale connettore è stato raggiunto, quale IP si è connesso e cosa si sono detti client e server. È l’unica fonte che risolve l’insidia descritta sopra.

### Attivare la registrazione

Per impostazione predefinita, la registrazione è **disattivata** sulla maggior parte dei connettori. La attivate per singolo connettore:

```powershell
Set-ReceiveConnector -Identity "SRV-MAIL01\Default Frontend SRV-MAIL01" `
    -ProtocolLoggingLevel Verbose
```

Per le connessioni in uscita, procedete analogamente tramite `Set-SendConnector`. Ricordate di riportare il valore a `None` dopo l’analisi, perché la registrazione dettagliata richiede spazio su disco e genera quantità notevoli di dati con volumi elevati.

### Dove si trovano i file

Exchange separa i protocolli per servizio e direzione. Non occorre codificare rigidamente i percorsi: interrogateli.

```powershell
Get-FrontendTransportService SRV-MAIL01 |
    Format-List ReceiveProtocolLogPath, SendProtocolLogPath,
        ReceiveProtocolLogMaxAge, ReceiveProtocolLogMaxDirectorySize

Get-TransportService SRV-MAIL01 |
    Format-List ReceiveProtocolLogPath, SendProtocolLogPath
```

In genere si trovano sotto il percorso di installazione in `TransportRoles\Logs\FrontEnd\ProtocolLog\SmtpReceive` per il Front End Transport e in `TransportRoles\Logs\Hub\ProtocolLog\SmtpReceive` per il Transport Service. **Questo è il punto cruciale:** le connessioni client sulla porta 25 si trovano esclusivamente nel percorso `FrontEnd`; il percorso `Hub` contiene solo il traffico di inoltro interno sulla 2525.

Tenete conto della conservazione. `ReceiveProtocolLogMaxAge` è spesso impostato a 30 giorni, mentre `ReceiveProtocolLogMaxDirectorySize` limita ulteriormente l’uso di spazio. Con volumi elevati, il limite di dimensione entra in vigore molto prima di quello temporale e i vostri protocolli risalgono quindi solo a pochi giorni prima.

### Comprendere il formato

I file sono CSV con righe di intestazione che iniziano con `#`. Le colonne più importanti sono `date-time`, `connector-id`, `session-id`, `sequence-number`, `local-endpoint`, `remote-endpoint`, `event` e `data`.

Decisiva è la colonna `event`, un singolo carattere:

| Carattere | Significato |
|---|---|
| `+` | Connessione stabilita |
| `-` | Connessione terminata |
| `>` | Il server invia al client |
| `<` | Il client invia al server |
| `*` | Informazione del server, non traffico SMTP |

Riconoscete una sessione dalla `session-id` comune; `sequence-number` indica l’ordine all’interno della sessione. Un estratto tipico appare così:

```text
2026-08-11T09:47:10.4Z,SRV-MAIL01\smtp-noauth SRV-MAIL01,08DEF44EC454A414,0,
  10.0.1.13:25,10.0.20.22:51244,+,,
2026-08-11T09:47:10.4Z,SRV-MAIL01\smtp-noauth SRV-MAIL01,08DEF44EC454A414,1,
  10.0.1.13:25,10.0.20.22:51244,>,"220 srv-mail01.intern.example.com Microsoft ESMTP",
2026-08-11T09:47:10.5Z,SRV-MAIL01\smtp-noauth SRV-MAIL01,08DEF44EC454A414,2,
  10.0.1.13:25,10.0.20.22:51244,<,EHLO app01.intern.example.com,
2026-08-11T09:47:10.6Z,SRV-MAIL01\smtp-noauth SRV-MAIL01,08DEF44EC454A414,6,
  10.0.1.13:25,10.0.20.22:51244,<,MAIL FROM:<dienst@example-test.com>,
2026-08-11T09:47:10.7Z,SRV-MAIL01\smtp-noauth SRV-MAIL01,08DEF44EC454A414,8,
  10.0.1.13:25,10.0.20.22:51244,>,"250 2.1.5 Recipient OK",
```

Qui c’è tutto ciò che mancava nel Message Tracking: il connettore **reale** (`smtp-noauth`), l’IP sorgente **reale** (`10.0.20.22`) e il nome con cui il sistema si presenta in `EHLO`.

### Cercare in modo mirato

Per i singoli casi, un filtro testuale è molto più rapido di un’analisi a oggetti. Cercate l’indirizzo del mittente o il nome `EHLO` e fatevi restituire l’identificatore di sessione:

```powershell
$pfad = (Get-FrontendTransportService SRV-MAIL01).ReceiveProtocolLogPath
Select-String -Path "$pfad\*.log" -Pattern "dienst@example-test.com" -SimpleMatch |
    Select-Object -First 5 Line
```

Con la `session-id` trovata, recuperate la sessione completa:

```powershell
Select-String -Path "$pfad\*.log" -Pattern "08DEF44EC454A414" -SimpleMatch |
    ForEach-Object { $_.Line } | Select-Object -First 40
```

Se volete solo sapere quali connettori ricevono effettivamente traffico, contate le connessioni stabilite. Con file di grandi dimensioni è più veloce di ordini di grandezza rispetto all’analisi di ogni riga:

```powershell
Select-String -Path "$pfad\*.log" -Pattern ',\+,' |
    ForEach-Object { ($_.Line -split ',')[1] } |
    Group-Object | Sort-Object Count -Descending |
    Format-Table Count, Name -AutoSize
```

```text
Count Name
----- ----
51479 SRV-MAIL01\Default Frontend SRV-MAIL01
50756 SRV-MAIL01\smtp-auth SRV-MAIL01
19405 SRV-MAIL01\smtp-intern SRV-MAIL01
15789 SRV-MAIL01\smtp-noauth SRV-MAIL01
```

Questa distribuzione risponde a una domanda a cui il Message Tracking non può rispondere: quali percorsi usano realmente le vostre applicazioni? Prima di modificare un connettore, è il dato più importante in assoluto.

### Quando non è stato registrato nulla

Se manca qualsiasi riga nell’orario in questione, ci sono tre ragioni comuni: la registrazione era disattivata sul connettore interessato, il limite di conservazione ha già eliminato il file oppure state guardando nel percorso sbagliato, cioè nella directory `Hub` anziché in `FrontEnd`. Verificate in questo ordine.

## Passo 4: verificare le autorizzazioni

Se una consegna viene rifiutata o, al contrario, è consentito più di quanto previsto, occorre verificare le autorizzazioni del connettore. Qui si cela un’insidia tecnica: `Get-ADPermission` richiede il **DistinguishedName**. Se passate la consueta identità nel formato `Server\Connectorname`, la chiamata fallisce in una sessione remota con il fuorviante messaggio che l’oggetto non può essere trovato.

```powershell
$dn = (Get-ReceiveConnector "SRV-MAIL01\Default Frontend SRV-MAIL01").DistinguishedName
Get-ADPermission -Identity $dn -User "NT AUTHORITY\ANONYMOUS LOGON" |
    Where-Object { $_.ExtendedRights -like "*ms-Exch-SMTP*" } |
    Format-Table User, @{n='Rights'; e={$_.ExtendedRights}} -AutoSize
```

```text
User                         Rights
----                         ------
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Submit
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Accept-Any-Sender
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Accept-Authoritative-Domain-Sender
```

L’interpretazione è più semplice di quanto sembri, se distinguete quattro diritti:

| Diritto | Significato |
|---|---|
| `ms-Exch-SMTP-Submit` | può effettuare consegne |
| `ms-Exch-SMTP-Accept-Any-Sender` | può usare indirizzi mittente arbitrari |
| `ms-Exch-SMTP-Accept-Authoritative-Domain-Sender` | può presentarsi come dominio proprio |
| `ms-Exch-SMTP-Accept-Any-Recipient` | **può inoltrare verso domini esterni** |

I primi tre sono il set standard necessario per le consegne anonime e la ricezione di posta Internet. Solo il quarto diritto trasforma un connettore in ingresso in un relay. Su un connettore che accetta l’intero spazio di indirizzi, si tratta di un open relay. Su un connettore con restrizione IP rigorosa, è invece il percorso usuale e previsto affinché i server applicativi possano inviare posta esterna.

Non confuse `Accept-Any-Sender` con `Accept-Any-Recipient`. Il primo è innocuo e necessario, il secondo è l’impostazione rilevante per la sicurezza.

## Passo 5: controprova con una consegna propria

Se l’analisi rimane ambigua, effettuate voi stessi una consegna. In questo modo controllate completamente mittente, destinatario e punto di consegna:

```powershell
Send-MailMessage -SmtpServer 10.0.1.11 -Port 25 `
    -From 'test@example.com' `
    -To 'empfaenger@example.net' `
    -Subject 'Diagnose, bitte ignorieren' `
    -Body 'Testeinlieferung' `
    -Encoding UTF8
```

`Send-MailMessage` è ufficialmente deprecato, ma per scopi diagnostici resta lo strumento più rapido ed è disponibile su ogni server Windows. In caso di successo non produce output, cosa che può risultare insolita.

Se testate un percorso TLS sulla porta 587 e la controparte presenta un certificato che non corrisponde al nome utilizzato, ad esempio perché contattate l’indirizzo IP, la chiamata si interrompe con un errore di certificato. Per il test potete sospendere la verifica nella sessione:

```powershell
[Net.ServicePointManager]::ServerCertificateValidationCallback = { $true }
```

Questo vale solo per la sessione PowerShell corrente. Impostatelo consapevolmente e mai negli script in esecuzione operativa.

Se il messaggio di test arriva e volete sapere cosa gli è accaduto lungo il percorso, l’[Analizzatore di intestazioni e-mail](https://rafaelpfister.ch/tools/header-analyzer) è utile: scompone le intestazioni, traccia il percorso attraverso gli hop e mostra gli esiti delle verifiche di autenticazione, interamente in locale nel browser, senza che il messaggio lasci il vostro dispositivo.

## Exchange Online: la stessa domanda, uno strumento diverso

Nel tenant valgono regole diverse, ed è qui che le procedure abituali falliscono. Considerate queste differenze:

| | Exchange OnPrem | Exchange Online |
|---|---|---|
| Query | `Get-MessageTrackingLog` | `Get-MessageTraceV2` |
| Granularità | ogni evento di trasporto | una riga per messaggio e destinatario |
| Connettore visibile | sì (con limitazioni, vedi sopra) | **no** |
| Riferimento al server | sì, interrogare per server | non applicabile |
| Protocollo SMTP | disponibile | **non disponibile** |
| Conservazione | la vostra configurazione | circa 10 giorni tramite il cmdlet |
| Ritardo | quasi immediato | alcuni minuti |

Le tre conseguenze pratiche più importanti: non esiste **alcuna associazione al connettore**, per cui vi affidate a `FromIP` e `ToIP`. Non esiste **alcun protocollo SMTP**, quindi la conversazione SMTP non è ricostruibile. E i dati appaiono **in ritardo**: un messaggio appena inviato non compare subito.

### La query di base

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-4) `
    -EndDate (Get-Date) `
    -RecipientAddress "empfaenger@example.com" `
    -ResultSize 1000 |
  Sort-Object Received |
  Format-Table Received, SenderAddress, RecipientAddress, Status, FromIP, Size -AutoSize
```

```text
Received            SenderAddress          RecipientAddress          Status    FromIP
--------            -------------          ----------------          ------    ------
11.08.2026 08:27:16 emma@partner.example   empfaenger@example.com    Delivered 10.0.20.23
11.08.2026 09:05:24 dienst@example-test.com empfaenger@example.com   Failed    10.0.20.23
```

I valori più importanti di `Status`: `Delivered`, `Failed`, `Pending`, `Quarantined`, `FilteredAsSpam` e `Expanded` per le liste di distribuzione espanse. `Pending` significa che i tentativi di consegna sono ancora in corso, non che qualcosa sia guasto.

### I dettagli di un messaggio

Lo stato da solo non dice nulla sul motivo. A questo serve la vista dettagliata, che richiede l’identificatore del messaggio dalla query di base:

```powershell
$n = Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-4) -EndDate (Get-Date) `
        -RecipientAddress "empfaenger@example.com" -ResultSize 100 |
     Where-Object { $_.Status -eq 'Failed' } | Select-Object -First 1

Get-MessageTraceDetailV2 -MessageTraceId $n.MessageTraceId `
    -RecipientAddress $n.RecipientAddress |
  Format-List Date, Event, Action, Detail
```

Qui sono riportati i passaggi di elaborazione nel servizio, ad esempio applicazioni di regole, decisioni di filtro e il motivo di un rifiuto.

### Oltre dieci giorni

Il cmdlet risale a circa dieci giorni. Per periodi più vecchi esiste la ricerca storica, che viene eseguita in modo asincrono e fornisce il risultato in formato CSV, con un intervallo fino a 90 giorni:

```powershell
Start-HistoricalSearch -ReportTitle "Analyse Nachtlauf" `
    -StartDate (Get-Date).AddDays(-45) `
    -EndDate (Get-Date).AddDays(-30) `
    -ReportType MessageTrace `
    -SenderAddress "dienst@example-test.com" `
    -NotifyAddress "admin@example.com"

Get-HistoricalSearch | Format-Table JobId, ReportTitle, Status, SubmitDate -AutoSize
```

Prevedete tempo a sufficienza: questi lavori possono richiedere ore, a seconda del volume.

### Insidia 4: l’assenza di risultati non prova l’assenza di traffico

Questa è l’insidia più sottile nel tenant. `Get-MessageTraceV2` restituisce risultati a pagine, con un massimo di 5000 righe per chiamata. Con volumi elevati, una pagina può coprire solo pochi minuti anche se avete interrogato sette giorni. Se poi filtrate localmente, ad esempio per un IP sorgente, state filtrando su un estratto minuscolo.

Lo riconoscete dall’avviso che segnala ulteriori risultati:

```text
WARNING: There are more results, use the following command to get more.
Get-MessageTraceV2 -StartDate "2026-08-11T07:25:19Z" -EndDate "2026-08-11T09:05:46Z"
  -StartingRecipientAddress "naechster@example.com" -ResultSize 5000
```

Se appare, la vostra analisi è **incompleta**. Se non viene restituito alcun risultato, la conclusione corretta è: non trovato nell’estratto. Non è: non esiste.

Esistono due soluzioni corrette. Potete ridurre la finestra temporale fino a quando una pagina la copre completamente, riconoscibile dall’assenza dell’avviso. Oppure potete procedere attraverso tutte le pagine usando le indicazioni di continuazione contenute nell’avviso. Per stabilire se qualcosa non si verifica **mai**, è comunque preferibile una verifica della configurazione: se un sistema non possiede una route verso una destinazione, non può consegnarvi nulla, indipendentemente da qualsiasi finestra di osservazione.

L’analisi completa di tutti gli indirizzi che effettuano consegne è un argomento a sé, con insidie specifiche nell’interpretazione. È trattato in [Chi effettua effettivamente consegne nel vostro tenant? Aggregare gli indirizzi IP di consegna](https://rafaelpfister.ch/blog/einliefernde-ip-adressen-aggregieren).

## Una procedura che si è dimostrata efficace

In sintesi, questa sequenza si è dimostrata la più rapida. Cercate il messaggio su tutti i server e determinate l’ultimo evento. In caso di errore, passate immediatamente a `Format-List` e leggete la risposta SMTP completa, invece di dedurre dal tipo di evento. Poi chiarite la portata, ovvero raggruppate e contate. Solo se il caso è circoscritto ricostruite il percorso di consegna tramite configurazione dei connettori e protocollo SMTP. Infine, se necessario, eseguite una controprova con una consegna propria.

I principali sprechi di tempo sono sempre gli stessi: si legge una tabella troncata invece del messaggio di errore completo, si scambiano le copie shadow per passaggi di elaborazione, si crede alla `ConnectorId` nel tracking e si considera un campione vuoto una prova. Chi conosce questi quattro aspetti arriva di norma al livello corretto in pochi minuti.

## Fonti

1.  [Message tracking in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): descrizione dei campi ed elenco completo dei tipi di evento nel Message Tracking.

2.  [Protocol logging in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/protocol-logging): percorsi di archiviazione, formato e conservazione dei protocolli SMTP, incluso il Front End Transport.

3.  [Shadow redundancy in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/shadow-redundancy): spiega gli eventi relativi alle copie shadow e alla loro eliminazione.

4.  [Mail routing in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing): interazione tra Front End Transport e Transport Service, base del comportamento proxy.

5.  [Receive connectors in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/receive-connectors): associazioni, gruppi di autorizzazioni e meccanismi di autenticazione.

6.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2): successore di Get-MessageTrace, incluse logica di paginazione ed elenco dei campi.

7.  [Start-HistoricalSearch](https://learn.microsoft.com/en-us/powershell/module/exchange/start-historicalsearch): tracciamento asincrono dei messaggi fino a 90 giorni.
