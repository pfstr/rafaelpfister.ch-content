---
title: "Analizzare il flusso di posta in Exchange: Message Tracking, protocolli SMTP e connettori di ricezione"
navTitle: "Analizzare il flusso di posta"
description: "Come individuare sistematicamente in Exchange OnPrem, Hybrid ed Exchange Online dove è rimasto un messaggio: query con output di esempio, lettura corretta del protocollo SMTP e aspetti che portano regolarmente a conclusioni errate."
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
translationSourceHash: 867f3a9a27b8db05d2721cce6282149f4616c4170576d0ab3f39a3cbde3701e3
translationModel: gpt-5.6-terra
translatedAt: 2026-10-30T03:25:31.730Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/analizzare-il-flusso-di-posta-in-exchange-message-tracking-protocolli-smtp-e-connettori-di
---

# Analizzare il flusso di posta in Exchange: Message Tracking, protocolli SMTP e connettori di ricezione

La domanda più frequente nella gestione della posta è: un messaggio non è arrivato, dove è finito? Il Message Tracking risponde in modo affidabile, ma solo se sapete cosa **non** vi dice. Questo articolo descrive la procedura nell'ordine che si è dimostrato efficace, mostra l'output tipico per ogni query e indica le fonti di errore che regolarmente costano ore perché suggeriscono conclusioni plausibili ma errate.

Tutti gli esempi utilizzano nomi generici: `SRV-MAIL01` e `SRV-MAIL02` come server di trasporto, `example.com` come dominio. Se volete comporre i comandi per il vostro ambiente anziché digitarli: il [Generatore di comandi](https://rafaelpfister.ch/tools/command-builder) contiene i comuni comandi di Message Tracking e acquisizione per PowerShell e shell Unix affiancati, interamente in locale nel browser.

## Il principio: prima localizzare, poi spiegare

Il riflesso è cercare subito la causa. È più efficiente determinare prima fino a che punto sia arrivato il messaggio. Questo restringe drasticamente lo spazio di ricerca in un solo passaggio, perché poi sapete se dovete cercare nel vostro sistema, presso il gateway a monte o presso la destinazione.

L'ordine è quindi: trovare il messaggio, leggere l'ultimo evento, leggere il motivo dell'errore, determinare se si tratta di un caso singolo o di uno schema e solo allora ricostruire il percorso di inoltro.

## Passaggio 1: trovare il messaggio

Iniziate dal destinatario, perché quasi sempre lo conoscete. È importante eseguire la query su **tutti** i server di trasporto, non soltanto su uno.

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

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-Server` | Server di trasporto di cui viene interrogato il log di tracking; qui entrambi i server in sequenza tramite pipeline |
| `-Start` | Limite temporale inferiore della ricerca, qui le ultime sei ore |
| `-ResultSize Unlimited` | Rimuove il limite predefinito di 1000 voci |
| `-Recipients` | Filtra i messaggi inviati a questo indirizzo del destinatario |
| `Sort-Object Timestamp` | Ordina cronologicamente i risultati combinati dei due server |
| `-AutoSize -Wrap` | Adatta la larghezza delle colonne al contenuto e manda a capo i valori lunghi anziché troncarli |

</details>

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

Se la query non trova nulla, verificate se il destinatario è stato espanso tramite un gruppo di distribuzione. In tal caso è meglio cercare tramite il mittente:

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

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-Server` | Server di trasporto di cui viene interrogato il log di tracking |
| `-Start` | Limite temporale inferiore della ricerca |
| `-ResultSize Unlimited` | Rimuove il limite predefinito di 1000 voci |
| `Where-Object` | Filtra lato client i mittenti del dominio locale, poiché `-Sender` accetta solo indirizzi esatti |
| `@{n=…; e=…}` | Colonna calcolata: riunisce il campo raccolta `Recipients` in una stringa separata da virgole |

</details>

## Passaggio 2: leggere l'ultimo evento

L'intera diagnosi dipende dall'**ultimo** `EventId` del messaggio. Indica quale spazio di ricerca affrontare successivamente.

| Ultimo EventId | Significato | Passaggio successivo |
|---|---|---|
| `RECEIVE`, poi nulla | Il messaggio è bloccato | Controllare le code |
| `SEND` o `SENDEXTERNAL` | Consegnato correttamente | Cercare più avanti al prossimo hop |
| `FAIL` | Fallimento definitivo | Leggere il motivo in `RecipientStatus` |
| `DEFER` | È in corso un nuovo tentativo | Controllare la coda e il sistema di destinazione |
| `DROP` o `POISONMESSAGE` | Scartato | Regola di trasporto o agente |
| `DELIVER` | Consegnato a una casella postale locale | Controllare le regole della casella postale |
| `RESOLVE` | Il destinatario è stato riscritto | Leggere l'indirizzo di destinazione nella voce |

`RESOLVE` è il passaggio intermedio più istruttivo negli ambienti Hybrid, perché qui è visibile la riscrittura nell'indirizzo di routing cloud:

```text
EventId : RESOLVE
Source  : ROUTING
Sender  : absender@example.com
To      : BENUTZER@example.mail.onmicrosoft.com
```

Se vi compare l'indirizzo `onmicrosoft.com` previsto, l'oggetto destinatario è configurato correttamente e potete considerare chiusa la questione. Se continua a comparire l'indirizzo originale, nell'oggetto locale manca l'indirizzo di destinazione e Exchange tenta la consegna locale.

Se il messaggio è bloccato, la coda di solito mostra chiaramente il motivo:

```powershell
Get-Queue -Server SRV-MAIL01 |
    Where-Object { $_.MessageCount -gt 0 } |
    Format-Table Identity, DeliveryType, Status, MessageCount, NextHopDomain, LastError `
        -AutoSize -Wrap
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-Server` | Server di cui vengono interrogate le code di trasporto |
| `Where-Object` | Nasconde le code vuote, mostrando solo quelle con messaggi in attesa |
| `-AutoSize -Wrap` | Impedisce che la lunga colonna `LastError` venga troncata |

</details>

## Fonte di errore 1: il tracking è specifico del server e molte voci sono copie shadow

Se nell'output vedete coppie di `HARECEIVE` e `HADISCARD`, spesso con l'aggiunta `ExplicitlyDiscarded`, quel server **non ha elaborato** il messaggio. Conservava solo una copia shadow nell'ambito della Shadow Redundancy, mentre un altro server gestiva la consegna effettiva. Non appena il server primario segnala il successo, il partner elimina la propria copia.

Ecco come appare se avete interrogato solo il server sbagliato:

```text
Timestamp           EventId    SourceContext
---------           -------    -------------
11.08.2026 09:47:13 HARECEIVE  1a2aae6b-f3a3-4e04-9233-7de460b92223
11.08.2026 09:48:07 HADISCARD  ExplicitlyDiscarded;1a2aae6b-f3a3-4e04-9233-7de460b92223
```

Due righe, nessun errore, nessuna consegna. Chi ne conclude che il messaggio sia sparito cerca nel posto sbagliato. L'elaborazione effettiva si trova nel tracking del server partner.

In pratica questo significa due cose. Primo, tali righe non indicano un problema, ma il normale funzionamento. Secondo, dovete assolutamente interrogare tutti i server di trasporto.

## Fonte di errore 2: `Format-Table` tronca proprio le colonne decisive

Il motivo dell'errore si trova in `RecipientStatus`, e questo campo è lungo. In una tabella scompare del tutto oppure viene troncato. Proprio questo porta a vedere `FAIL` senza il motivo e a iniziare invece a indovinare.

Non appena avete trovato un caso di errore, passate quindi a `Format-List` e risolvete i campi raccolta:

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

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-Server` | Server di trasporto di cui viene interrogato il log di tracking |
| `-Start` | Limite temporale inferiore della ricerca |
| `-ResultSize Unlimited` | Rimuove il limite predefinito di 1000 voci |
| `-Recipients` | Filtra i messaggi inviati a questo indirizzo del destinatario |
| `-EventId FAIL` | Solo voci con errore definitivo di consegna |
| `Format-List` | Mostra ogni campo su una riga separata a lunghezza completa; nulla viene troncato |
| `@{n=…; e=…}` | Campi calcolati: risolvono i campi raccolta `Recipients` e `RecipientStatus` in stringhe leggibili |

</details>

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

La diagnosi è quindi certa, senza aver avuto bisogno di alcuna supposizione: la controparte contesta il mittente. `LED` contiene la risposta SMTP completa, `FQDN` e `IP` indicano il sistema che ha risposto, mentre `LRT` indica il momento dell'ultimo tentativo.

## Passaggio 3: caso singolo o schema?

Prima di approfondire un singolo caso, chiarite l'ampiezza. Questa sola query decide se avete a che fare con una nota marginale o con un incidente:

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

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-Start` | Limite temporale inferiore, qui le ultime otto ore |
| `-EventId FAIL` | Solo consegne definitivamente fallite |
| `-ResultSize Unlimited` | Rimuove il limite predefinito di 1000 voci |
| `Where-Object` | Filtra il codice di stato SMTP analizzato nel campo `RecipientStatus` |
| `Group-Object` | Raggruppa per dominio del mittente, ossia la parte dopo `@` |
| `Sort-Object Count -Descending` | Il dominio più frequente in alto |

</details>

Sostituite `5.1.8` con il codice di stato che state analizzando. L'output risponde alla domanda in una riga:

```text
Count Name
----- ----
    9 example-test.com
```

Un solo dominio mittente significa: problema circoscritto, nessun incidente, potete proseguire con calma. Se vi fossero venti domini diversi, avreste un'interruzione in corso e tutto il resto dovrebbe attendere. Fare questa distinzione così presto fa risparmiare, per esperienza, più tempo di ogni altra cosa.

## Fonte di errore 3: `ConnectorId` non indica il vero connettore di ricezione

Questa è la fonte di errore più costosa, perché l'output sembra affidabile. La posta consegnata da un client o da un sistema esterno sulla porta 25 raggiunge prima il **Front End Transport**. Questo inoltra il messaggio al **Transport Service** sulla porta 2525. Il Message Tracking viene scritto solo lì; il Front End Transport non scrive un proprio tracking.

La conseguenza è visibile in questa riga:

```text
EventId        : RECEIVE
ConnectorId    : SRV-MAIL01\Default SRV-MAIL01
ClientIp       : 10.0.1.11
ClientHostname : srv-mail01.intern.example.com
```

`ConnectorId` indica il connettore interno sulla porta 2525 e `ClientIp` è l'indirizzo del **server proxy**, non quello del mittente originario. Nel tracking non è semplicemente indicato quale dei connettori configurati sulla porta 25 sia stato effettivamente raggiunto da un sistema. Chi crede a questa indicazione cerca l'errore in un connettore che non è nemmeno coinvolto.

Ci sono due modi per ottenere la risposta. Il primo consiste nella ricostruzione tramite la configurazione:

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object { Get-ReceiveConnector -Server $_ } |
    Format-List Identity, Enabled,
        @{n='Bindings';       e={$_.Bindings -join ','}},
        @{n='RemoteIPRanges'; e={$_.RemoteIPRanges -join ','}},
        PermissionGroups, AuthMechanism
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-Server` | Server di cui vengono elencati i connettori di ricezione |
| `Format-List` | Lunghezze complete dei campi; `RemoteIPRanges` e `PermissionGroups` verrebbero troncati nelle tabelle |
| `@{n=…; e=…}` | Campi calcolati: riuniscono i campi raccolta `Bindings` e `RemoteIPRanges` in stringhe separate da virgole |

</details>

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

Determinate l'IP di origine del sistema che invia e cercate il connettore il cui `RemoteIPRanges` lo contiene. Se non rientra in nessuno dei connettori limitati, resta il connettore frontend predefinito, che di norma accetta l'intero spazio di indirizzi. Anche qui utilizzate `Format-List`, poiché `RemoteIPRanges` e `PermissionGroups` vengono regolarmente troncati nelle tabelle.

Il secondo modo è il protocollo SMTP, che merita una sezione dedicata.

## Il protocollo SMTP: l'unica fonte completa

Il protocollo del Front End Transport registra la sessione SMTP completa: quale connettore è stato contattato, quale IP si è connesso, cosa si sono comunicati client e server. È l'unica fonte che risolve il problema descritto sopra con `ConnectorId`.

### Attivare la registrazione

Per impostazione predefinita, la registrazione è **disattivata** sulla maggior parte dei connettori. La attivate per singolo connettore:

```powershell
Set-ReceiveConnector -Identity "SRV-MAIL01\Default Frontend SRV-MAIL01" `
    -ProtocolLoggingLevel Verbose
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-Identity` | Il connettore da modificare nel formato `Server\Connectorname` |
| `-ProtocolLoggingLevel Verbose` | Attiva la registrazione SMTP; `None` la disattiva nuovamente |

</details>

Per le connessioni in uscita, analogamente tramite `Set-SendConnector`. Ricordatevi di reimpostare il valore su `None` dopo l'analisi, perché la registrazione dettagliata consuma spazio disco e, con volumi elevati, scrive quantità considerevoli di dati.

### Dove si trovano i file

Exchange separa i protocolli per servizio e direzione. Non è necessario codificare i percorsi in modo fisso: interrogateli:

```powershell
Get-FrontendTransportService SRV-MAIL01 |
    Format-List ReceiveProtocolLogPath, SendProtocolLogPath,
        ReceiveProtocolLogMaxAge, ReceiveProtocolLogMaxDirectorySize

Get-TransportService SRV-MAIL01 |
    Format-List ReceiveProtocolLogPath, SendProtocolLogPath
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `SRV-MAIL01` | Parametro posizionale `-Identity`: il server da interrogare |
| `ReceiveProtocolLogPath`, `SendProtocolLogPath` | Percorsi di archiviazione dei protocolli per connessioni in entrata e in uscita |
| `ReceiveProtocolLogMaxAge` | Età massima dei file di protocollo; quelli più vecchi vengono eliminati |
| `ReceiveProtocolLogMaxDirectorySize` | Limite massimo dello spazio utilizzato dalla directory dei protocolli |

</details>

Di norma si trovano sotto il percorso di installazione in `TransportRoles\Logs\FrontEnd\ProtocolLog\SmtpReceive` per il Front End Transport e in `TransportRoles\Logs\Hub\ProtocolLog\SmtpReceive` per il Transport Service. **Questo è il punto essenziale:** le connessioni client sulla porta 25 si trovano esclusivamente nel percorso `FrontEnd`; nel percorso `Hub` si trova soltanto il traffico interno di inoltro sulla porta 2525.

Tenete presente la conservazione. `ReceiveProtocolLogMaxAge` è spesso impostato a 30 giorni, mentre `ReceiveProtocolLogMaxDirectorySize` limita ulteriormente lo spazio utilizzato. Con volumi elevati, il limite di dimensione interviene molto prima del limite di età e i vostri protocolli possono avere solo pochi giorni.

### Comprendere il formato

I file sono CSV con righe di intestazione che iniziano con `#`. Le colonne più importanti sono `date-time`, `connector-id`, `session-id`, `sequence-number`, `local-endpoint`, `remote-endpoint`, `event` e `data`.

Decisiva è la colonna `event`, un singolo carattere:

| Carattere | Significato |
|---|---|
| `+` | Connessione stabilita |
| `-` | Connessione terminata |
| `>` | Il server invia al client |
| `<` | Il client invia al server |
| `*` | Informazione del server, nessun traffico SMTP |

Riconoscete una sessione dal comune `session-id`; `sequence-number` indica l'ordine all'interno della sessione. Un estratto tipico appare così:

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

Qui si trova tutto ciò che mancava nel Message Tracking: il connettore **reale** (`smtp-noauth`), l'IP di origine **reale** (`10.0.20.22`) e il nome con cui il sistema si presenta in `EHLO`.

### Cercare in modo mirato

Per i singoli casi, un filtro testuale è molto più rapido di un'analisi di oggetti. Cercate l'indirizzo del mittente o il nome `EHLO` e fatevi restituire l'identificativo della sessione:

```powershell
$pfad = (Get-FrontendTransportService SRV-MAIL01).ReceiveProtocolLogPath
Select-String -Path "$pfad\*.log" -Pattern "dienst@example-test.com" -SimpleMatch |
    Select-Object -First 5 Line
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-Path "$pfad\*.log"` | Cerca in tutti i file di protocollo nel percorso interrogato in precedenza |
| `-Pattern` | Il termine di ricerca, qui l'indirizzo del mittente |
| `-SimpleMatch` | Tratta il modello come testo anziché come espressione regolare; il punto nell'indirizzo non richiede quindi escaping |
| `-First 5` | Limita l'output ai primi cinque risultati |

</details>

Con il `session-id` trovato, recuperate la sessione completa:

```powershell
Select-String -Path "$pfad\*.log" -Pattern "08DEF44EC454A414" -SimpleMatch |
    ForEach-Object { $_.Line } | Select-Object -First 40
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-Pattern` | L'identificativo della sessione dal primo risultato |
| `-SimpleMatch` | Ricerca letterale senza valutazione regex |
| `-First 40` | Limita l'output alle prime 40 righe della sessione |

</details>

Se volete solo sapere quali connettori vedono effettivamente traffico, contate le connessioni stabilite. Su file di grandi dimensioni è di ordini di grandezza più rapido che analizzare ogni riga:

```powershell
Select-String -Path "$pfad\*.log" -Pattern ',\+,' |
    ForEach-Object { ($_.Line -split ',')[1] } |
    Group-Object | Sort-Object Count -Descending |
    Format-Table Count, Name -AutoSize
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-Pattern ',\+,'` | Espressione regolare per l'evento `+` (connessione stabilita) tra due virgole CSV; il più è escaped |
| `ForEach-Object { … -split ',' }` | Divide la riga trovata sulle virgole e accede alla seconda colonna, `connector-id` |
| `Group-Object` | Conta le connessioni stabilite per connettore |
| `Sort-Object Count -Descending` | Il connettore più utilizzato in alto |

</details>

```text
Count Name
----- ----
51479 SRV-MAIL01\Default Frontend SRV-MAIL01
50756 SRV-MAIL01\smtp-auth SRV-MAIL01
19405 SRV-MAIL01\smtp-intern SRV-MAIL01
15789 SRV-MAIL01\smtp-noauth SRV-MAIL01
```

Questa distribuzione risponde a una domanda a cui il Message Tracking non può rispondere: quali percorsi utilizzano davvero le vostre applicazioni? Prima di una modifica dei connettori, questo è il dato più importante in assoluto.

### Se non è stato registrato nulla

Se manca ogni riga nel momento in questione, vi sono tre ragioni comuni: la registrazione era disattivata sul connettore interessato, il limite di conservazione ha già eliminato il file oppure state guardando nel percorso sbagliato, quindi nella directory `Hub` anziché `FrontEnd`. Verificate in quest'ordine.

## Passaggio 4: controllare le autorizzazioni

Se un invio viene rifiutato oppure, al contrario, è consentito più di quanto previsto, il percorso passa attraverso le autorizzazioni del connettore. Vi è una particolarità tecnica: `Get-ADPermission` richiede il **DistinguishedName**. Se passate la consueta identità nel formato `Server\Connectorname`, in una sessione remota la chiamata fallisce con il fuorviante messaggio che l'oggetto non è stato trovato.

```powershell
$dn = (Get-ReceiveConnector "SRV-MAIL01\Default Frontend SRV-MAIL01").DistinguishedName
Get-ADPermission -Identity $dn -User "NT AUTHORITY\ANONYMOUS LOGON" |
    Where-Object { $_.ExtendedRights -like "*ms-Exch-SMTP*" } |
    Format-Table User, @{n='Rights'; e={$_.ExtendedRights}} -AutoSize
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-Identity $dn` | L'oggetto da verificare come DistinguishedName; il formato `Server\Connectorname` fallisce nelle sessioni remote |
| `-User` | Limita l'output alle autorizzazioni di questo principal di sicurezza, qui l'accesso anonimo |
| `Where-Object` | Filtra gli Extended Rights rilevanti per SMTP |

</details>

```text
User                         Rights
----                         ------
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Submit
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Accept-Any-Sender
NT AUTHORITY\ANONYMOUS LOGON ms-Exch-SMTP-Accept-Authoritative-Domain-Sender
```

La valutazione è più semplice di quanto sembri se distinguete quattro diritti:

| Diritto | Significato |
|---|---|
| `ms-Exch-SMTP-Submit` | può effettuare l'invio |
| `ms-Exch-SMTP-Accept-Any-Sender` | può utilizzare indirizzi mittente arbitrari |
| `ms-Exch-SMTP-Accept-Authoritative-Domain-Sender` | può presentarsi come dominio locale |
| `ms-Exch-SMTP-Accept-Any-Recipient` | **può inoltrare a domini esterni** |

I primi tre costituiscono il set standard per l'invio anonimo e sono necessari per ricevere posta Internet. Solo il quarto diritto trasforma un connettore di ingresso in un relay. Su un connettore che accetta dall'intero spazio di indirizzi, costituisce un open relay. Su un connettore con restrizione IP rigorosa, invece, è il metodo usuale e previsto per consentire ai server applicativi di inviare messaggi esterni.

Non confuse `Accept-Any-Sender` con `Accept-Any-Recipient`. Il primo è innocuo e necessario, il secondo è l'impostazione rilevante per la sicurezza.

## Passaggio 5: test di controllo tramite invio proprio

Se l'analisi resta ambigua, effettuate voi stessi un invio. In questo modo controllate completamente mittente, destinatario e punto di ingresso:

```powershell
Send-MailMessage -SmtpServer 10.0.1.11 -Port 25 `
    -From 'test@example.com' `
    -To 'empfaenger@example.net' `
    -Subject 'Diagnose, bitte ignorieren' `
    -Body 'Testeinlieferung' `
    -Encoding UTF8
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-SmtpServer` | Host di destinazione dell'invio, qui volutamente come indirizzo IP per raggiungere un endpoint specifico |
| `-Port 25` | Porta di destinazione; 25 per l'invio server-server non autenticato |
| `-From` | Mittente envelope e header del messaggio di test |
| `-To` | Indirizzo del destinatario |
| `-Subject` | Riga dell'oggetto |
| `-Body` | Testo del messaggio |
| `-Encoding UTF8` | Codifica dei caratteri per oggetto e testo, evita problemi con caratteri accentati |

</details>

`Send-MailMessage` è ufficialmente deprecato, ma per scopi diagnostici resta lo strumento più rapido ed è disponibile su ogni server Windows. In caso di successo non produce output, il che può sembrare insolito.

Se testate un percorso TLS sulla porta 587 e la controparte presenta un certificato che non corrisponde al nome utilizzato, ad esempio perché contattate l'indirizzo IP, la chiamata si interrompe con un errore di certificato. Per il test potete sospendere la verifica nella sessione:

```powershell
[Net.ServicePointManager]::ServerCertificateValidationCallback = { $true }
```

Questo vale solo per la sessione PowerShell corrente. Impostatelo consapevolmente e mai in script eseguiti in produzione.

Se il messaggio di test arriva e volete sapere cosa gli è successo durante il percorso, l'[Analizzatore di intestazioni email](https://rafaelpfister.ch/tools/header-analyzer) è utile: analizza le intestazioni, traccia il percorso tra gli hop e mostra i risultati dei controlli di autenticazione, interamente in locale nel browser, senza che il messaggio lasci il vostro dispositivo.

## Exchange Online: la stessa domanda, uno strumento diverso

Nel tenant vigono regole diverse, ed è qui che le procedure abituali falliscono. Considerate queste differenze:

| | Exchange OnPrem | Exchange Online |
|---|---|---|
| Query | `Get-MessageTrackingLog` | `Get-MessageTraceV2` |
| Granularità | ogni evento di trasporto | una riga per messaggio e destinatario |
| Connettore visibile | sì, con limitazioni, vedi sopra | **no** |
| Riferimento al server | sì, interrogare per server | non applicabile |
| Protocollo SMTP | disponibile | **non disponibile** |
| Conservazione | configurazione dell'utente | 90 giorni tramite il cmdlet, al massimo 10 giorni per query |
| Ritardo | quasi immediato | alcuni minuti |

Le tre conseguenze pratiche più importanti: non esiste **alcuna assegnazione del connettore**, dovete ricorrere a `FromIP` e `ToIP`. Non esiste **alcun protocollo SMTP**, quindi la conversazione SMTP non è ricostruibile. E i dati compaiono **con ritardo**: un messaggio appena inviato non appare subito.

### La query di base

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-4) `
    -EndDate (Get-Date) `
    -RecipientAddress "empfaenger@example.com" `
    -ResultSize 1000 |
  Sort-Object Received |
  Format-Table Received, SenderAddress, RecipientAddress, Status, FromIP, Size -AutoSize
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-StartDate` | Limite temporale inferiore della query, qui le ultime quattro ore |
| `-EndDate` | Limite temporale superiore; il cmdlet richiede entrambi i limiti |
| `-RecipientAddress` | Filtra i messaggi inviati a questo indirizzo del destinatario |
| `-ResultSize 1000` | Numero massimo di righe in questa pagina; il limite massimo è 5000 |

</details>

```text
Received            SenderAddress          RecipientAddress          Status    FromIP
--------            -------------          ----------------          ------    ------
11.08.2026 08:27:16 emma@partner.example   empfaenger@example.com    Delivered 10.0.20.23
11.08.2026 09:05:24 dienst@example-test.com empfaenger@example.com   Failed    10.0.20.23
```

I valori più importanti di `Status` sono: `Delivered`, `Failed`, `Pending`, `Quarantined`, `FilteredAsSpam` e `Expanded` per gruppi di distribuzione espansi. `Pending` significa che sono ancora in corso tentativi di consegna, non che qualcosa sia guasto.

### I dettagli di un messaggio

Il solo stato non dice nulla sul motivo. A questo scopo serve la vista dettagliata, che richiede l'identificativo del messaggio dalla query di base:

```powershell
$n = Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-4) -EndDate (Get-Date) `
        -RecipientAddress "empfaenger@example.com" -ResultSize 100 |
     Where-Object { $_.Status -eq 'Failed' } | Select-Object -First 1

Get-MessageTraceDetailV2 -MessageTraceId $n.MessageTraceId `
    -RecipientAddress $n.RecipientAddress |
  Format-List Date, Event, Action, Detail
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-MessageTraceId` | Identificativo univoco del messaggio dalla query di base; obbligatorio |
| `-RecipientAddress` | Destinatario di cui vengono mostrati i passaggi di elaborazione; anch'esso obbligatorio, poiché un messaggio può avere più destinatari |

</details>

Qui sono riportati i passaggi di elaborazione nel servizio, ad esempio applicazioni di regole, decisioni di filtraggio e il motivo di un rifiuto.

### Oltre dieci giorni

`Get-MessageTraceV2` fornisce dati degli ultimi 90 giorni, ma al massimo 10 giorni per query; suddividete gli intervalli più lunghi in più query. In alternativa è disponibile la ricerca storica, che viene eseguita in modo asincrono e fornisce il risultato come CSV, con un intervallo fino a 90 giorni:

```powershell
Start-HistoricalSearch -ReportTitle "Analyse Nachtlauf" `
    -StartDate (Get-Date).AddDays(-45) `
    -EndDate (Get-Date).AddDays(-30) `
    -ReportType MessageTrace `
    -SenderAddress "dienst@example-test.com" `
    -NotifyAddress "admin@example.com"

Get-HistoricalSearch | Format-Table JobId, ReportTitle, Status, SubmitDate -AutoSize
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-ReportTitle` | Nome liberamente scelto per il processo, con cui il risultato sarà successivamente reperibile |
| `-StartDate`, `-EndDate` | Periodo analizzato, fino a 90 giorni indietro |
| `-ReportType MessageTrace` | Tipo di report; `MessageTrace` fornisce la panoramica dei messaggi come CSV |
| `-SenderAddress` | Filtra questo indirizzo del mittente |
| `-NotifyAddress` | Destinatario della notifica di completamento; deve essere un indirizzo di un Accepted Domain del tenant |

</details>

Pianificate del tempo: a seconda dell'ampiezza, questi processi possono richiedere ore.

### Fonte di errore 4: risultati mancanti non sono prova di assenza di traffico

Questa è la fonte di errore più sottile nel tenant. `Get-MessageTraceV2` restituisce risultati paginati, al massimo 5000 righe per chiamata. Con volumi elevati, una pagina può coprire solo pochi minuti anche se avete interrogato sette giorni. Se poi filtrate localmente, ad esempio per un IP di origine, filtrate su un estratto minuscolo.

Lo riconoscete dall'avviso che segnala ulteriori risultati:

```text
WARNING: There are more results, use the following command to get more.
Get-MessageTraceV2 -StartDate "2026-08-11T07:25:19Z" -EndDate "2026-08-11T09:05:46Z"
  -StartingRecipientAddress "naechster@example.com" -ResultSize 5000
```

Se appare, la vostra analisi è **incompleta**. Se non viene restituito alcun risultato, la conclusione corretta è: non trovato nell'estratto. Non: non esiste.

Ci sono due soluzioni corrette. Potete ridurre la finestra temporale finché una pagina la copre completamente, riconoscibile dall'assenza dell'avviso. Oppure potete seguire le indicazioni di continuazione dell'avviso attraverso tutte le pagine. Per la domanda se qualcosa non accada **mai**, una verifica della configurazione è comunque superiore: se un sistema non possiede una route verso una destinazione, non può consegnarvi messaggi, indipendentemente da ogni finestra di osservazione.

L'analisi completa di tutti gli indirizzi mittenti è un argomento a sé, con propri aspetti delicati nell'interpretazione. Si trova in [Chi invia effettivamente nel vostro tenant? Aggregare gli indirizzi IP di invio](https://rafaelpfister.ch/blog/einliefernde-ip-adressen-aggregieren).

## Una procedura che si è dimostrata efficace

In sintesi, questa sequenza si è dimostrata la più rapida. Cercare il messaggio su tutti i server e determinare l'ultimo evento. In caso di fallimento, passare subito a `Format-List` e leggere la risposta SMTP completa anziché dedurre dal tipo di evento. Poi chiarire l'ampiezza, quindi raggruppare e contare. Solo quando il caso è circoscritto, ricostruire il percorso di ingresso tramite configurazione dei connettori e protocollo SMTP. Infine, se necessario, verificare con un invio proprio.

I maggiori sprechi di tempo sono sempre gli stessi: si legge una tabella troncata anziché il messaggio di errore completo, si scambiano le copie shadow per passaggi di elaborazione, si crede a `ConnectorId` nel tracking e si considera un campione vuoto come prova. Chi conosce questi quattro aspetti arriva di norma al livello corretto in pochi minuti.

## Cronologia delle modifiche

- **25 ottobre 2026:** Corretto: Get-MessageTraceV2 fornisce dati degli ultimi 90 giorni, ma al massimo 10 giorni per query. Aggiornati i link delle fonti ai nuovi indirizzi di Microsoft Learn.

## Fonti

1.  [Message tracking in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): descrizione dei campi e elenco completo dei tipi di evento nel Message Tracking.

2.  [Protocol logging in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/protocol-logging): percorsi di archiviazione, formato e conservazione dei protocolli SMTP, incluso Front End Transport.

3.  [Shadow redundancy in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/shadow-redundancy): spiega gli eventi relativi alle copie shadow e alla loro eliminazione.

4.  [Mail routing in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing): interazione tra Front End Transport e Transport Service, base del comportamento proxy.

5.  [Receive connectors in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/receive-connectors): binding, gruppi di autorizzazione e meccanismi di autenticazione.

6.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2): successore di Get-MessageTrace, inclusi logica di paginazione, elenco dei campi e intervallo di query (90 giorni, massimo 10 giorni per query).

7.  [Start-HistoricalSearch](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/start-historicalsearch): tracciamento asincrono dei messaggi fino a 90 giorni.
