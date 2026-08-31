---
title: "Analizzare il flusso di posta in Exchange: Message Tracking, protocolli SMTP e connettori di ricezione"
navTitle: "Analizzare il flusso di posta"
description: "Come stabilire sistematicamente, in Exchange OnPrem, Hybrid ed Exchange Online, dove è finito un messaggio: query con output di esempio, come leggere correttamente il protocollo SMTP e gli aspetti che portano regolarmente a conclusioni errate."
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
translationSourceHash: da923f7fa45ee5c38ea52e96d56781f7c3806556245a5f071242e7f02473a71c
translationModel: gpt-5.6-terra
translatedAt: 2026-08-31T10:10:58.725Z
translationReview: automatic
url: https://rafaelpfister.ch/it/blog/analizzare-il-flusso-di-posta-in-exchange-message-tracking-protocolli-smtp-e-connettori-di
---

# Analizzare il flusso di posta in Exchange: Message Tracking, protocolli SMTP e connettori di ricezione

La domanda più frequente nella gestione della posta è: un messaggio non è arrivato, dove è finito? Il Message Tracking risponde in modo affidabile, ma solo se sapete cosa **non** vi dice. Questo articolo descrive il procedimento nell'ordine che si è dimostrato efficace, mostra l'output tipico di ogni query e indica le fonti di errore che fanno perdere regolarmente ore perché suggeriscono conclusioni plausibili ma errate.

Tutti gli esempi utilizzano nomi generici: `SRV-MAIL01` e `SRV-MAIL02` come server di trasporto, `example.com` come dominio. Se volete comporre i comandi per il vostro ambiente anziché digitarli: il [Generatore di comandi](https://rafaelpfister.ch/tools/command-builder) contiene i comuni comandi di Message Tracking e acquisizione per PowerShell e shell Unix affiancati, interamente in locale nel browser.

## Il principio: prima localizzare, poi spiegare

Il riflesso è cercare subito la causa. È più efficiente determinare prima fino a che punto sia arrivato effettivamente il messaggio. Questo restringe drasticamente lo spazio di ricerca in un solo passaggio, perché saprete poi se cercare nel vostro sistema, presso il gateway a monte o presso la destinazione.

L'ordine è quindi: trovare il messaggio, leggere l'ultimo evento, leggere il motivo dell'errore, stabilire se si tratta di un caso isolato o di uno schema ricorrente e solo allora ricostruire il percorso di consegna.

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
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Server` | Server di trasporto di cui viene interrogato il log di tracking; qui entrambi i server vengono interrogati in sequenza tramite pipeline |
| `-Start` | Limite temporale inferiore della ricerca, qui le ultime sei ore |
| `-ResultSize Unlimited` | Rimuove il limite predefinito di 1000 voci |
| `-Recipients` | Filtra i messaggi indirizzati a questo destinatario |
| `Sort-Object Timestamp` | Ordina cronologicamente i risultati riuniti dei due server |
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

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Server` | Server di trasporto di cui viene interrogato il log di tracking |
| `-Start` | Limite temporale inferiore della ricerca |
| `-ResultSize Unlimited` | Rimuove il limite predefinito di 1000 voci |
| `Where-Object` | Filtra lato client i mittenti del dominio interno, poiché `-Sender` accetta soltanto indirizzi esatti |
| `@{n=…; e=…}` | Colonna calcolata: riunisce il campo raccolta `Recipients` in una stringa separata da virgole |

</details>

## Passaggio 2: leggere l'ultimo evento

L'intera diagnosi dipende dall'**ultimo** `EventId` del messaggio. Vi indica quale spazio di ricerca affrontare successivamente.

| Ultimo EventId | Significato | Passaggio successivo |
|---|---|---|
| `RECEIVE`, poi nulla | Il messaggio è bloccato | Controllare le code |
| `SEND` o `SENDEXTERNAL` | Consegnato correttamente | Cercare più avanti, all'hop successivo |
| `FAIL` | Fallito definitivamente | Leggere il motivo in `RecipientStatus` |
| `DEFER` | È in corso un nuovo tentativo | Controllare coda e sistema di destinazione |
| `DROP` o `POISONMESSAGE` | Scartato | Regola di trasporto o agente |
| `DELIVER` | Consegnato a una cassetta postale locale | Controllare le regole della cassetta postale |
| `RESOLVE` | Il destinatario è stato riscritto | Leggere l'indirizzo di destinazione nella voce |

`RESOLVE` è il passaggio intermedio più rivelatore negli ambienti Hybrid, perché rende visibile la riscrittura verso l'indirizzo di routing cloud:

```text
EventId : RESOLVE
Source  : ROUTING
Sender  : absender@example.com
To      : BENUTZER@example.mail.onmicrosoft.com
```

Se appare l'indirizzo `onmicrosoft.com` previsto, l'oggetto destinatario è configurato correttamente e potete chiudere il caso. Se compare ancora l'indirizzo originale, manca l'indirizzo di destinazione nell'oggetto locale e Exchange tenta di consegnare localmente.

Se il messaggio è bloccato, la coda di solito mostra il motivo in chiaro:

```powershell
Get-Queue -Server SRV-MAIL01 |
    Where-Object { $_.MessageCount -gt 0 } |
    Format-Table Identity, DeliveryType, Status, MessageCount, NextHopDomain, LastError `
        -AutoSize -Wrap
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Server` | Server di cui vengono interrogate le code di trasporto |
| `Where-Object` | Nasconde le code vuote, mostrando solo quelle con messaggi in attesa |
| `-AutoSize -Wrap` | Impedisce che la lunga colonna `LastError` venga troncata |

</details>

## Fonte di errore 1: il tracking è legato al server e molte voci sono copie shadow

Se nell'output vedete coppie di `HARECEIVE` e `HADISCARD`, spesso con l'aggiunta `ExplicitlyDiscarded`, quel server **non ha elaborato** il messaggio. Conservava solo una copia shadow nell'ambito della Shadow Redundancy, mentre un altro server eseguiva la consegna effettiva. Non appena il server primario segnala il successo, il partner scarta la propria copia.

Ecco l'aspetto se avete interrogato solo il server sbagliato:

```text
Timestamp           EventId    SourceContext
---------           -------    -------------
11.08.2026 09:47:13 HARECEIVE  1a2aae6b-f3a3-4e04-9233-7de460b92223
11.08.2026 09:48:07 HADISCARD  ExplicitlyDiscarded;1a2aae6b-f3a3-4e04-9233-7de460b92223
```

Due righe, nessun errore, nessuna consegna. Chi ne conclude che il messaggio sia scomparso sta cercando nel posto sbagliato. L'elaborazione effettiva si trova nel tracking del server partner.

In pratica ciò significa due cose. Primo, queste righe non sono un segnale di problema, ma normale funzionamento. Secondo, dovete necessariamente interrogare tutti i server di trasporto.

## Fonte di errore 2: `Format-Table` tronca proprio le colonne decisive

Il motivo dell'errore si trova in `RecipientStatus`, e questo campo è lungo. In una tabella scompare del tutto oppure viene troncato. Proprio questo fa sì che si veda il `FAIL` ma non il motivo, iniziando quindi a indovinare.

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

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Server` | Server di trasporto di cui viene interrogato il log di tracking |
| `-Start` | Limite temporale inferiore della ricerca |
| `-ResultSize Unlimited` | Rimuove il limite predefinito di 1000 voci |
| `-Recipients` | Filtra i messaggi indirizzati a questo destinatario |
| `-EventId FAIL` | Solo voci con errore di consegna definitivo |
| `Format-List` | Mostra ogni campo su una riga separata, a lunghezza intera: nulla viene troncato |
| `@{n=…; e=…}` | Campi calcolati: espandono i campi raccolta `Recipients` e `RecipientStatus` in stringhe leggibili |

</details>

Ecco la differenza. Prima la visualizzazione tabellare, che non spiega nulla:

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

La diagnosi è quindi certa, senza dover formulare una sola ipotesi: il sistema remoto contesta il mittente. `LED` contiene la risposta SMTP completa, `FQDN` e `IP` indicano il sistema che ha risposto, mentre `LRT` indica l'ora dell'ultimo tentativo.

## Passaggio 3: caso isolato o schema ricorrente?

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

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Start` | Limite temporale inferiore, qui le ultime otto ore |
| `-EventId FAIL` | Solo consegne fallite definitivamente |
| `-ResultSize Unlimited` | Rimuove il limite predefinito di 1000 voci |
| `Where-Object` | Filtra in base al codice di stato SMTP esaminato nel campo `RecipientStatus` |
| `Group-Object` | Raggruppa per dominio del mittente, ovvero la parte dopo `@` |
| `Sort-Object Count -Descending` | Il dominio più frequente in cima |

</details>

Sostituite `5.1.8` con il codice di stato che state esaminando. L'output risponde alla domanda in una riga:

```text
Count Name
----- ----
    9 example-test.com
```

Un solo dominio mittente significa: problema circoscritto, non un incidente; potete continuare a cercare con calma. Se fossero indicati venti domini diversi, avreste un'interruzione in corso e tutto il resto dovrebbe attendere. Stabilire questa distinzione così presto fa risparmiare, per esperienza, più tempo di ogni altra cosa.

## Fonte di errore 3: la `ConnectorId` non indica il vero connettore di ricezione

Questa è la fonte di errore più costosa, perché l'output sembra serio. La posta consegnata da un client o da un sistema esterno sulla porta 25 raggiunge prima il **Front End Transport**. Quest'ultimo inoltra il messaggio al **Transport Service** sulla porta 2525. Il Message Tracking viene scritto solo lì; il Front End Transport non scrive un proprio tracking.

La conseguenza è visibile in questa riga:

```text
EventId        : RECEIVE
ConnectorId    : SRV-MAIL01\Default SRV-MAIL01
ClientIp       : 10.0.1.11
ClientHostname : srv-mail01.intern.example.com
```

La `ConnectorId` indica il connettore interno sulla porta 2525 e la `ClientIp` è l'indirizzo del **server proxy**, non quello del mittente originario. Il tracking semplicemente non indica quale dei connettori configurati sulla porta 25 abbia effettivamente ricevuto un sistema. Chi crede a questa indicazione cerca l'errore in un connettore che non è stato affatto coinvolto.

Ci sono due modi per ottenere la risposta. Il primo consiste nella ricostruzione tramite la configurazione:

```powershell
'SRV-MAIL01','SRV-MAIL02' | ForEach-Object { Get-ReceiveConnector -Server $_ } |
    Format-List Identity, Enabled,
        @{n='Bindings';       e={$_.Bindings -join ','}},
        @{n='RemoteIPRanges'; e={$_.RemoteIPRanges -join ','}},
        PermissionGroups, AuthMechanism
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

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

Determinate l'IP sorgente del sistema mittente e cercate il connettore il cui `RemoteIPRanges` la contiene. Se non rientra in nessuno dei connettori limitati, resta il connettore frontend predefinito, che di solito accetta l'intero spazio di indirizzi. Usate anche qui `Format-List`, poiché `RemoteIPRanges` e `PermissionGroups` vengono regolarmente troncati nelle tabelle.

Il secondo modo è il protocollo SMTP, che merita una sezione dedicata.

## Il protocollo SMTP: l'unica fonte completa

Il protocollo del Front End Transport registra la sessione SMTP completa: quale connettore è stato contattato, quale IP si è connesso, cosa si sono detti client e server. È l'unica fonte che risolve il problema della `ConnectorId` descritto sopra.

### Attivare la registrazione

Per impostazione predefinita, la registrazione è **disattivata** sulla maggior parte dei connettori. La attivate per singolo connettore:

```powershell
Set-ReceiveConnector -Identity "SRV-MAIL01\Default Frontend SRV-MAIL01" `
    -ProtocolLoggingLevel Verbose
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Identity` | Il connettore da modificare nel formato `Server\Connectorname` |
| `-ProtocolLoggingLevel Verbose` | Attiva la registrazione SMTP; `None` la disattiva nuovamente |

</details>

Per le connessioni in uscita, procedete analogamente tramite `Set-SendConnector`. Ricordate di riportare il valore a `None` dopo l'analisi, perché la registrazione dettagliata richiede spazio su disco e genera notevoli volumi di dati in caso di traffico elevato.

### Dove si trovano i file

Exchange separa i protocolli per servizio e direzione. Non è necessario codificare i percorsi: interrogateli:

```powershell
Get-FrontendTransportService SRV-MAIL01 |
    Format-List ReceiveProtocolLogPath, SendProtocolLogPath,
        ReceiveProtocolLogMaxAge, ReceiveProtocolLogMaxDirectorySize

Get-TransportService SRV-MAIL01 |
    Format-List ReceiveProtocolLogPath, SendProtocolLogPath
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `SRV-MAIL01` | Parametro posizionale `-Identity`: il server da interrogare |
| `ReceiveProtocolLogPath`, `SendProtocolLogPath` | Percorsi di archiviazione dei protocolli per connessioni in entrata e in uscita |
| `ReceiveProtocolLogMaxAge` | Età massima dei file di protocollo; quelli più vecchi vengono eliminati |
| `ReceiveProtocolLogMaxDirectorySize` | Limite massimo di spazio utilizzato dalla directory dei protocolli |

</details>

In genere si trovano sotto il percorso di installazione in `TransportRoles\Logs\FrontEnd\ProtocolLog\SmtpReceive` per il Front End Transport e in `TransportRoles\Logs\Hub\ProtocolLog\SmtpReceive` per il Transport Service. **Questo è il punto essenziale:** le connessioni client sulla porta 25 si trovano esclusivamente nel percorso `FrontEnd`, mentre il percorso `Hub` contiene solo il traffico di inoltro interno sulla porta 2525.

Prestate attenzione alla conservazione. `ReceiveProtocolLogMaxAge` è spesso impostato su 30 giorni, mentre `ReceiveProtocolLogMaxDirectorySize` limita ulteriormente lo spazio utilizzato. In caso di traffico elevato, il limite di dimensione entra in vigore ben prima di quello di età e i vostri protocolli finiscono per avere solo pochi giorni.

### Comprendere il formato

I file sono CSV con intestazioni che iniziano con `#`. Le colonne più importanti sono `date-time`, `connector-id`, `session-id`, `sequence-number`, `local-endpoint`, `remote-endpoint`, `event` e `data`.

Decisiva è la colonna `event`, costituita da un singolo carattere:

| Carattere | Significato |
|---|---|
| `+` | Connessione stabilita |
| `-` | Connessione terminata |
| `>` | Il server invia al client |
| `<` | Il client invia al server |
| `*` | Informazione del server, nessun traffico SMTP |

Riconoscete una sessione dalla stessa `session-id`; la `sequence-number` indica l'ordine all'interno della sessione. Un estratto tipico appare così:

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

Qui c'è tutto ciò che mancava nel Message Tracking: il connettore **effettivo** (`smtp-noauth`), l'IP sorgente **effettivo** (`10.0.20.22`) e il nome con cui il sistema si presenta nell'`EHLO`.

### Cercare in modo mirato

Per i singoli casi, un filtro testuale è molto più rapido di un'analisi degli oggetti. Cercate l'indirizzo del mittente o il nome `EHLO` e fatevi restituire l'identificativo della sessione:

```powershell
$pfad = (Get-FrontendTransportService SRV-MAIL01).ReceiveProtocolLogPath
Select-String -Path "$pfad\*.log" -Pattern "dienst@example-test.com" -SimpleMatch |
    Select-Object -First 5 Line
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Path "$pfad\*.log"` | Cerca in tutti i file di protocollo nel percorso interrogato in precedenza |
| `-Pattern` | Il termine di ricerca, qui l'indirizzo del mittente |
| `-SimpleMatch` | Tratta il modello come testo anziché come espressione regolare; il punto nell'indirizzo non richiede quindi escape |
| `-First 5` | Limita l'output ai primi cinque risultati |

</details>

Con la `session-id` trovata recuperate la sessione completa:

```powershell
Select-String -Path "$pfad\*.log" -Pattern "08DEF44EC454A414" -SimpleMatch |
    ForEach-Object { $_.Line } | Select-Object -First 40
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Pattern` | L'identificativo della sessione dal primo risultato |
| `-SimpleMatch` | Ricerca letterale senza valutazione regex |
| `-First 40` | Limita l'output alle prime 40 righe della sessione |

</details>

Se volete solo sapere quali connettori vedono effettivamente traffico, contate le connessioni stabilite. Con file di grandi dimensioni, questo è più veloce di ordini di grandezza rispetto al parsing di ogni riga:

```powershell
Select-String -Path "$pfad\*.log" -Pattern ',\+,' |
    ForEach-Object { ($_.Line -split ',')[1] } |
    Group-Object | Sort-Object Count -Descending |
    Format-Table Count, Name -AutoSize
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Pattern ',\+,'` | Espressione regolare per l'evento `+` (connessione stabilita) tra due virgole CSV; il più è sottoposto a escape |
| `ForEach-Object { … -split ',' }` | Divide la riga trovata alle virgole e seleziona la seconda colonna, la `connector-id` |
| `Group-Object` | Conta le connessioni stabilite per connettore |
| `Sort-Object Count -Descending` | Il connettore più utilizzato in cima |

</details>

```text
Count Name
----- ----
51479 SRV-MAIL01\Default Frontend SRV-MAIL01
50756 SRV-MAIL01\smtp-auth SRV-MAIL01
19405 SRV-MAIL01\smtp-intern SRV-MAIL01
15789 SRV-MAIL01\smtp-noauth SRV-MAIL01
```

Questa distribuzione risponde a una domanda a cui il Message Tracking non può rispondere: quali percorsi utilizzano effettivamente le vostre applicazioni? Prima di una modifica dei connettori, questo è il dato più importante in assoluto.

### Se non è stato registrato nulla

Se al momento in questione manca ogni riga, le cause comuni sono tre: la registrazione era disattivata sul connettore interessato, il limite di conservazione ha già eliminato il file oppure state guardando nel percorso sbagliato, cioè nella directory `Hub` anziché `FrontEnd`. Verificate in quest'ordine.

## Passaggio 4: verificare le autorizzazioni

Se una consegna viene rifiutata oppure, al contrario, è consentito più di quanto previsto, dovete esaminare le autorizzazioni del connettore. C'è una particolarità tecnica: `Get-ADPermission` richiede il **DistinguishedName**. Se passate l'identità abituale nel formato `Server\Connectorname`, la chiamata fallisce in una sessione remota con il fuorviante messaggio che l'oggetto non è stato trovato.

```powershell
$dn = (Get-ReceiveConnector "SRV-MAIL01\Default Frontend SRV-MAIL01").DistinguishedName
Get-ADPermission -Identity $dn -User "NT AUTHORITY\ANONYMOUS LOGON" |
    Where-Object { $_.ExtendedRights -like "*ms-Exch-SMTP*" } |
    Format-Table User, @{n='Rights'; e={$_.ExtendedRights}} -AutoSize
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Identity $dn` | L'oggetto da controllare come DistinguishedName; il formato `Server\Connectorname` fallisce nelle sessioni remote |
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
| `ms-Exch-SMTP-Submit` | può effettuare consegne in ingresso |
| `ms-Exch-SMTP-Accept-Any-Sender` | può utilizzare qualsiasi indirizzo mittente |
| `ms-Exch-SMTP-Accept-Authoritative-Domain-Sender` | può presentarsi come dominio interno |
| `ms-Exch-SMTP-Accept-Any-Recipient` | **può inoltrare a domini esterni** |

I primi tre sono il set standard necessario per la consegna anonima e la ricezione della posta Internet. Solo il quarto diritto trasforma un connettore di ingresso in un relay. Su un connettore che accetta l'intero spazio di indirizzi, diventa un open relay. Su un connettore con una restrizione IP rigorosa, rappresenta invece il normale e previsto metodo per consentire ai server applicativi di inviare all'esterno.

Non confuse `Accept-Any-Sender` con `Accept-Any-Recipient`. Il primo è innocuo e necessario, il secondo è l'impostazione rilevante per la sicurezza.

## Passaggio 5: test di controllo con una consegna propria

Se l'analisi rimane ambigua, effettuate voi stessi una consegna. In questo modo controllate completamente mittente, destinatario e punto di consegna:

```powershell
Send-MailMessage -SmtpServer 10.0.1.11 -Port 25 `
    -From 'test@example.com' `
    -To 'empfaenger@example.net' `
    -Subject 'Diagnose, bitte ignorieren' `
    -Body 'Testeinlieferung' `
    -Encoding UTF8
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-SmtpServer` | Host di destinazione della consegna, qui volutamente come indirizzo IP per raggiungere un endpoint specifico |
| `-Port 25` | Porta di destinazione; 25 per consegne server-to-server non autenticate |
| `-From` | Mittente della busta e dell'intestazione del messaggio di prova |
| `-To` | Indirizzo del destinatario |
| `-Subject` | Riga dell'oggetto |
| `-Body` | Testo del messaggio |
| `-Encoding UTF8` | Codifica dei caratteri per oggetto e testo, evita problemi con gli umlaut |

</details>

`Send-MailMessage` è ufficialmente deprecato, ma per scopi diagnostici resta lo strumento più rapido ed è disponibile su ogni server Windows. In caso di successo non produce alcun output, cosa che può risultare insolita.

Se testate un percorso TLS sulla porta 587 e il sistema remoto presenta un certificato che non corrisponde al nome utilizzato, ad esempio perché contattate l'indirizzo IP, la chiamata si interrompe con un errore di certificato. Per il test potete sospendere la verifica nella sessione:

```powershell
[Net.ServicePointManager]::ServerCertificateValidationCallback = { $true }
```

Questo vale solo per la sessione PowerShell in corso. Impostatelo consapevolmente e mai in script eseguiti in produzione.

Se il messaggio di prova arriva e volete sapere cosa gli è accaduto lungo il percorso, l'[Analizzatore di intestazioni email](https://rafaelpfister.ch/tools/header-analyzer) aiuta: scompone le intestazioni, traccia il percorso attraverso gli hop e mostra i risultati dei controlli di autenticazione, interamente in locale nel browser, senza che il messaggio lasci il vostro dispositivo.

## Exchange Online: la stessa domanda, uno strumento diverso

Nel tenant si applicano regole diverse, ed è qui che le procedure abituali falliscono. Considerate queste differenze:

| | Exchange OnPrem | Exchange Online |
|---|---|---|
| Query | `Get-MessageTrackingLog` | `Get-MessageTraceV2` |
| Granularità | ogni evento di trasporto | una riga per messaggio e destinatario |
| Connettore visibile | sì, con limitazioni, vedi sopra | **no** |
| Riferimento al server | sì, interrogare per ciascun server | non applicabile |
| Protocollo SMTP | disponibile | **non disponibile** |
| Conservazione | vostra configurazione | circa 10 giorni tramite il cmdlet |
| Ritardo | quasi immediato | alcuni minuti |

Le tre conseguenze più importanti nella pratica: non esiste **alcuna associazione al connettore**, dovete aiutarvi con `FromIP` e `ToIP`. Non esiste **alcun protocollo SMTP**, quindi la conversazione SMTP non è ricostruibile. E i dati compaiono **con ritardo**, pertanto un messaggio appena inviato non appare immediatamente.

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
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-StartDate` | Limite temporale inferiore della query, qui le ultime quattro ore |
| `-EndDate` | Limite temporale superiore; il cmdlet richiede entrambi i limiti |
| `-RecipientAddress` | Filtra i messaggi indirizzati a questo destinatario |
| `-ResultSize 1000` | Numero massimo di righe di questa pagina; il limite massimo è 5000 |

</details>

```text
Received            SenderAddress          RecipientAddress          Status    FromIP
--------            -------------          ----------------          ------    ------
11.08.2026 08:27:16 emma@partner.example   empfaenger@example.com    Delivered 10.0.20.23
11.08.2026 09:05:24 dienst@example-test.com empfaenger@example.com   Failed    10.0.20.23
```

I valori più importanti di `Status` sono: `Delivered`, `Failed`, `Pending`, `Quarantined`, `FilteredAsSpam` e `Expanded` per le liste di distribuzione espanse. `Pending` significa che sono ancora in corso tentativi di consegna, non che qualcosa sia guasto.

### I dettagli di un messaggio

Il solo stato non dice nulla sul motivo. A tale scopo vi serve la visualizzazione dettagliata, che richiede l'identificativo del messaggio dalla query di base:

```powershell
$n = Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-4) -EndDate (Get-Date) `
        -RecipientAddress "empfaenger@example.com" -ResultSize 100 |
     Where-Object { $_.Status -eq 'Failed' } | Select-Object -First 1

Get-MessageTraceDetailV2 -MessageTraceId $n.MessageTraceId `
    -RecipientAddress $n.RecipientAddress |
  Format-List Date, Event, Action, Detail
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-MessageTraceId` | Identificativo univoco del messaggio dalla query di base; obbligatorio |
| `-RecipientAddress` | Destinatario di cui vengono mostrate le fasi di elaborazione; anch'esso obbligatorio, poiché un messaggio può avere più destinatari |

</details>

Qui sono riportate le fasi di elaborazione nel servizio, come l'applicazione di regole, le decisioni dei filtri e il motivo di un rifiuto.

### Oltre dieci giorni

Il cmdlet risale a circa dieci giorni. Per periodi più vecchi esiste la ricerca storica, che viene eseguita in modo asincrono e rende disponibile il risultato come CSV, per un intervallo fino a 90 giorni:

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
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-ReportTitle` | Nome liberamente scegliibile del processo, con il quale il risultato potrà essere ritrovato in seguito |
| `-StartDate`, `-EndDate` | Periodo esaminato, fino a 90 giorni indietro |
| `-ReportType MessageTrace` | Tipo di report; `MessageTrace` fornisce la panoramica dei messaggi come CSV |
| `-SenderAddress` | Filtra per questo indirizzo del mittente |
| `-NotifyAddress` | Destinatario della notifica di completamento; deve essere un indirizzo di un dominio accettato del tenant |

</details>

Pianificate il tempo necessario: questi processi possono richiedere ore, a seconda della portata.

### Fonte di errore 4: l'assenza di risultati non prova l'assenza di traffico

Questa è la fonte di errore più sottile nel tenant. `Get-MessageTraceV2` restituisce i risultati per pagine, al massimo 5000 righe per chiamata. In caso di traffico elevato, una pagina può coprire solo pochi minuti anche se avete interrogato sette giorni. Se poi filtrate localmente, ad esempio per un IP sorgente, state filtrando un estratto molto ristretto.

Lo riconoscete dall'avviso che indica ulteriori risultati:

```text
WARNING: There are more results, use the following command to get more.
Get-MessageTraceV2 -StartDate "2026-08-11T07:25:19Z" -EndDate "2026-08-11T09:05:46Z"
  -StartingRecipientAddress "naechster@example.com" -ResultSize 5000
```

Se appare, la vostra analisi è **incompleta**. Se non viene restituito alcun risultato, la conclusione corretta è: non trovato nell'estratto. Non è: non esiste.

Ci sono due alternative corrette. Potete ridurre la finestra temporale finché una pagina la copre interamente, riconoscibile dall'assenza dell'avviso. Oppure potete scorrere tutte le pagine utilizzando le indicazioni di continuazione contenute nell'avviso. Per stabilire se qualcosa non si verifica **mai**, un controllo della configurazione è comunque superiore: se un sistema non possiede una route verso una destinazione, non può consegnarvi nulla, indipendentemente da qualsiasi finestra di osservazione.

L'analisi completa di tutti gli indirizzi mittenti è un argomento a sé, con propri aspetti delicati nell'interpretazione. È trattata in [Chi consegna effettivamente nel vostro tenant? Aggregare gli indirizzi IP mittenti](https://rafaelpfister.ch/blog/einliefernde-ip-adressen-aggregieren).

## Un procedimento che si è dimostrato efficace

In sintesi, questa sequenza si è dimostrata la più rapida. Cercate il messaggio su tutti i server e stabilite l'ultimo evento. In caso di errore, passate subito a `Format-List` e leggete la risposta SMTP completa, anziché dedurre dal tipo di evento. Poi chiarite la portata, quindi raggruppate e contate. Solo quando il caso è circoscritto ricostruite il percorso di consegna tramite la configurazione dei connettori e il protocollo SMTP. Infine, se necessario, verificate con una vostra consegna.

I maggiori sprechi di tempo sono invece sempre gli stessi: si legge una tabella troncata anziché il messaggio di errore completo, si scambiano le copie shadow per fasi di elaborazione, si crede alla `ConnectorId` nel tracking e si considera un campione vuoto una prova. Chi conosce questi quattro aspetti arriva di norma al giusto livello in pochi minuti.

## Fonti

1.  [Message tracking in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): descrizione dei campi ed elenco completo dei tipi di evento nel Message Tracking.

2.  [Protocol logging in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/protocol-logging): percorsi di archiviazione, formato e conservazione dei protocolli SMTP, incluso Front End Transport.

3.  [Shadow redundancy in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/shadow-redundancy): spiega gli eventi relativi alle copie shadow e al loro scarto.

4.  [Mail routing in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing): interazione tra Front End Transport e Transport Service, alla base del comportamento proxy.

5.  [Receive connectors in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/receive-connectors): binding, gruppi di autorizzazioni e meccanismi di autenticazione.

6.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2): successore di Get-MessageTrace, inclusi logica di paginazione ed elenco dei campi.

7.  [Start-HistoricalSearch](https://learn.microsoft.com/en-us/powershell/module/exchange/start-historicalsearch): tracciamento asincrono dei messaggi fino a 90 giorni.
