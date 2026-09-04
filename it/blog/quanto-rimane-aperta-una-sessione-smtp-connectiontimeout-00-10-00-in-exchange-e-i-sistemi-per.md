---
title: "Quanto rimane aperta una sessione SMTP? ConnectionTimeout 00:10:00 in Exchange e i sistemi per cui è troppo breve"
navTitle: "Durata della sessione SMTP"
description: "Exchange termina ogni sessione SMTP in ingresso dopo dieci minuti, anche se sta trasferendo dati. Quali mittenti rimangono così a lungo su una connessione, come ricavare la durata effettiva della sessione dal log del protocollo e quando modificare ConnectionTimeout e ConnectionInactivityTimeout su un connettore relay."
date: "2026-09-03"
kategorie: "SMTP e flusso di posta"
timeToRead: "10 min di lettura"
themen:
  - smtp-mailflow
  - exchange-onprem-hybrid
produkte:
  - "exchange-on-premises"
  - "exchange"
protokolle:
  - "smtp"
  - "troubleshooting"
slug: "quanto-rimane-aperta-una-sessione-smtp-connectiontimeout-00-10-00-in-exchange-e-i-sistemi-per"
translationId: "article-b40497933bbe0a88"
aiPrompt: |
  Du bist mein Exchange- und Mailflow-Assistent. Hilf mir, die SMTP-Session-Dauer auf einem Exchange-Receive-Connector zu beurteilen: 1. Frage mich, welche Systeme (Relays, Gateways, Applikationen, Scanner) über den Connector einliefern und ob sie Verbindungen über mehrere Nachrichten hinweg offen halten. 2. Lass dir die Ausgabe der Session-Auswertung aus dem Protokoll-Log geben (IP, Mails, Dauer, Timeout-Kennzeichen) und erkläre mir, welche Sessions am ConnectionTimeout abgebrochen wurden. 3. Empfiehl pro Connector konkrete Werte für ConnectionTimeout und ConnectionInactivityTimeout und begründe, warum der internetseitige Connector unverändert bleibt. 4. Nenne mir, was ich stattdessen auf der Client-Seite ändern kann, damit die Verbindung nach einer festen Anzahl Nachrichten neu aufgebaut wird.
translationOf: smtp-session-dauer-exchange-connectiontimeout
url: https://rafaelpfister.ch/it/blog/quanto-rimane-aperta-una-sessione-smtp-connectiontimeout-00-10-00-in-exchange-e-i-sistemi-per
translationSourceHash: a107c4edd960dabb30ba1b6f263a693808a5edf6815747d81f5d446c103a7e79
translationModel: gpt-5.6-terra
translatedAt: 2026-09-04T08:21:07.047Z
translationReview: automatic
---

# Quanto rimane aperta una sessione SMTP? ConnectionTimeout 00:10:00 in Exchange e i sistemi per cui è troppo breve

In breve: una sessione SMTP non ha una fine naturale. RFC 5321 limita soltanto il tempo di attesa per il passaggio successivo e un client può inviare ulteriori messaggi su una connessione aperta finché il server la mantiene aperta. Exchange la mantiene aperta per impostazione predefinita per dieci minuti sui connettori di ricezione, quindi il server chiude la connessione indipendentemente dal fatto che i dati stiano fluendo. Per il traffico da Exchange a Exchange e per la maggior parte degli MTA questo è irrilevante, perché tali mittenti si riconnettono autonomamente dopo pochi secondi. Per applicazioni, gateway e generatori di carico che utilizzano una sola connessione per un intero ciclo di invio, invece, questo valore è la causa di interruzioni che nel client si manifestano come errori di connessione e nel log del protocollo di Exchange come `421 4.4.1 Connection timed out`.

## Due timeout con significati diversi

Un connettore di ricezione dispone di due limiti di tempo che vengono spesso confusi:

| Parametro | Significato | Predefinito server Mailbox | Predefinito Edge Transport |
|---|---|---|---|
| `ConnectionInactivityTimeout` | tempo massimo di inattività senza attività del client, dopo il quale la connessione viene chiusa | 00:05:00 | 00:01:00 |
| `ConnectionTimeout` | durata totale massima della connessione, anche se sta trasferendo dati | 00:10:00 | 00:05:00 |

Entrambi i valori accettano da 1 secondo a 1 giorno (`1.00:00:00`), e `ConnectionTimeout` deve essere maggiore di `ConnectionInactivityTimeout`. I valori si applicano per connettore, quindi separatamente per il connettore rivolto a Internet `Default Frontend <Server>`, per il connettore del servizio di trasporto `Default <Server>` sulla porta 2525 e per ogni connettore relay creato manualmente.

Il timeout di inattività non è critico: cinque minuti corrispondono esattamente al minimo che RFC 5321 prescrive a un server come tempo di attesa per il comando successivo, e un client che non invia nulla per cinque minuti generalmente ha già dimenticato la connessione. Il timeout complessivo è una particolarità di Exchange: viene calcolato dall'apertura della connessione e continua a scorrere mentre il client recapita messaggio dopo messaggio. Dopo dieci minuti Exchange chiude la connessione nel punto in cui si trova il dialogo, se necessario anche nel mezzo di un blocco `DATA`.

Sul lato di invio non esiste un equivalente: un connettore di invio dispone solo di `ConnectionInactivityTimeOut` (predefinito dieci minuti) e limita invece le sessioni tramite `SmtpMaxMessagesPerConnection`, per impostazione predefinita 20 messaggi. Exchange, in qualità di client, termina quindi ogni connessione al più tardi dopo 20 messaggi e ne stabilisce una nuova. Questo è il motivo per cui il timeout complessivo non si nota mai tra server Exchange: le sessioni durano pochi secondi.

## Cosa prescrive RFC 5321

Lo standard definisce nella sezione 4.5.3.2 i tempi minimi di attesa che un client deve rispettare per ogni passaggio del protocollo prima di abbandonare la connessione:

| Passaggio | Timeout minimo lato client |
|---|---|
| Attesa del banner `220` | 5 minuti |
| Risposta a `MAIL` | 5 minuti |
| Risposta a `RCPT` | 5 minuti |
| Risposta a `DATA` (il `354`) | 2 minuti |
| Invio di un blocco di dati | 3 minuti |
| Risposta al punto finale | 10 minuti |
| Server: attesa del comando successivo | almeno 5 minuti |

L'RFC non prevede un limite massimo per la durata totale di una sessione. Un client che recapita messaggi per trenta minuti sulla stessa connessione senza restare in silenzio per più di qualche secondo si comporta in modo conforme allo standard. È significativo l'ultimo valore lato client: dieci minuti di attesa per la risposta dopo il punto finale, perché in questa fase il server accetta e prende in carico il messaggio. Se il client interrompe troppo presto, il messaggio è già stato recapitato e verrà consegnato una seconda volta al tentativo successivo. La stessa situazione si verifica specularmente quando il server chiude la connessione in quel momento a causa del timeout complessivo.

Se un server chiude la connessione con `421`, il client deve trattare la transazione in corso secondo la sezione 3.8 come se avesse ricevuto un `451`, ovvero come errore temporaneo da ritentare. Un MTA con una coda fa esattamente questo. Un'applicazione senza coda segnala invece un'eccezione e lascia il resto al chiamante.

## Per quanto tempo i mittenti mantengono effettivamente aperte le loro sessioni

La durata della sessione è determinata dal client e le differenze tra i tipi di mittente sono notevoli:

| Mittente | Durata tipica della sessione | Limitata da |
|---|---|---|
| Connettore di invio Exchange | Secondi | `SmtpMaxMessagesPerConnection` = 20 |
| Postfix con cache delle connessioni | al massimo 5 minuti | `smtp_connection_reuse_time_limit` = 300s |
| Postfix senza cache delle connessioni | un messaggio per connessione | comportamento predefinito del client `smtp` |
| Applicazione con `.NET SmtpClient`, `JavaMail Transport`, Python `smtplib` | finché l'oggetto resta attivo: per un'esecuzione batch, l'intera esecuzione | solo dal codice del programma |
| Notifiche di quarantena dei gateway di posta | una sessione per esecuzione delle notifiche | comportamento del prodotto, in parte con keepalive `NOOP` |
| Dispositivi multifunzione, scan-to-mail | un messaggio per connessione; per scansioni grandi su linee lente diversi minuti | dimensione del file e larghezza di banda |
| Generatori di carico come `smtp-source -d` | fino alla fine dell'esecuzione | parametri di chiamata |

Le prime due righe spiegano perché in ambienti classici il valore non viene notato per anni: gli MTA mantengono autonomamente brevi le connessioni. Postfix, ad esempio, utilizza una connessione in cache per un massimo di cinque minuti e poi ne apre una nuova, mentre Exchange si disconnette dopo 20 messaggi. Entrambi rimangono quindi al di sotto di qualsiasi valore predefinito di Exchange.

La riga relativa alle applicazioni è il caso problematico più frequente. Un processo batch che invia fatture, buste paga o notifiche di sistema crea tipicamente un oggetto client, richiama il metodo di invio su di esso in un ciclo e lo chiude alla fine. `System.Net.Mail.SmtpClient` utilizza la stessa connessione per chiamate consecutive a `Send` a partire da .NET Framework 4 e invia `QUIT` solo al momento di `Dispose`; JavaMail si comporta allo stesso modo con un `Transport` aperto una sola volta. Se il processo dura più di dieci minuti, a un certo punto nel mezzo si verifica `421` e il processo si interrompe con un'eccezione, in .NET ad esempio con il testo `Service not available, closing transmission channel. The server response was: 4.4.1 Connection timed out`. Il messaggio interessato dipende dalla durata dell'esecuzione, perciò l'errore sembra casuale: talvolta l'interruzione avviene dopo 800 messaggi, altre dopo 1200, a seconda della dimensione dei messaggi e del carico del server.

La riga sui gateway descrive un caso documentato: Symantec (oggi Broadcom) Messaging Gateway invia le notifiche di quarantena antispam tramite un'unica connessione e invia `NOOP` come keepalive tra i messaggi. Exchange risponde a `NOOP` con il ritardo Tarpit di cinque secondi, così in dieci minuti possono passare al massimo circa 120 notifiche prima che la sessione termini con `421 4.4.1` e il gateway debba riconnettersi.

La riga sugli scanner riguarda un problema di dimensioni anziché di quantità: una scansione da 60 MB su una connessione a 2 Mbit/s richiede circa quattro minuti di puro tempo di trasferimento; con 100 MB sono quasi sette minuti. Su un server Edge Transport con timeout complessivo di cinque minuti questo è già sufficiente per un'interruzione; su un server Mailbox resta un margine, ma ridotto.

## Cosa accade in caso di interruzione

Quando scade il timeout complessivo, Exchange scrive la risposta `421 4.4.1 Connection timed out` nel log del protocollo, la invia al client e chiude la connessione. Per la transazione in corso vale quanto segue: se il punto finale non è stato ancora inviato, il messaggio non è stato accettato e deve essere ripetuto integralmente. Se il punto è stato inviato e la connessione viene chiusa prima della risposta `250`, il client non sa se Exchange abbia preso in carico il messaggio; un client implementato correttamente lo ripete e il destinatario potrebbe riceverlo due volte. La probabilità è bassa, ma con migliaia di messaggi per esecuzione non è zero.

Va inoltre considerato il percorso proxy: il servizio di trasporto front-end accetta la connessione sulla porta 25 e la inoltra come sessione SMTP separata al servizio di trasporto sulla porta 2525, dove si applica il connettore `Default <Server>` con gli stessi valori predefiniti. Una sessione lunga compare quindi in entrambi i log e un adeguamento deve includere entrambi i connettori.

## Ricavare la durata effettiva della sessione dal log del protocollo

Prima di modificare un valore, vale la pena osservare le sessioni reali. Il presupposto è il logging dettagliato del protocollo sul connettore interessato; su `Default Frontend <Server>` è già attivo, su tutti gli altri connettori no:

```powershell
Set-ReceiveConnector -Identity 'EX01\Relay Applikationen' -ProtocolLoggingLevel Verbose
```

I log si trovano in `%ExchangeInstallPath%TransportRoles\Logs\FrontEnd\ProtocolLog\SmtpReceive` (front-end) e `%ExchangeInstallPath%TransportRoles\Logs\Hub\ProtocolLog\SmtpReceive` (servizio di trasporto), con nomi basati sull'ora UTC come `RECVyyyyMMddhh-nnnn.log`. Ogni riga rappresenta un evento del protocollo con i campi `date-time`, `connector-id`, `session-id`, `sequence-number`, `local-endpoint`, `remote-endpoint`, `event`, `data` e `context`. Tutte le righe di una sessione hanno lo stesso valore `session-id`, quindi la durata della sessione è la differenza tra il primo e l'ultimo timestamp di tale ID.

Lo script seguente analizza il file di log più recente della giornata per un connettore, raggruppa le righe per sessione e mostra le 15 sessioni più lunghe con numero di messaggi, durata e l'informazione se Exchange le ha terminate con `421 4.4.1`:

```powershell
$logPfad = Join-Path $env:ExchangeInstallPath 'TransportRoles\Logs\FrontEnd\ProtocolLog\SmtpReceive'
$connector = 'Relay Applikationen'
$tag = (Get-Date).ToUniversalTime().ToString('yyyyMMdd')
$datei = Get-ChildItem $logPfad -Filter "RECV$tag*.log" |
    Sort-Object Name -Descending |
    Select-Object -First 1

$sessions = @{}
Get-Content $datei.FullName |
    Where-Object { $_ -notlike '#*' -and $_ -like "*$connector*" } |
    ForEach-Object {
        $c = $_ -split ','
        $s = $c[2]
        if (-not $sessions[$s]) {
            $sessions[$s] = [pscustomobject]@{
                IP = ($c[5] -split ':')[0]; Start = $c[0]; Ende = $c[0]
                Zeilen = 0; Mails = 0; Timeout = $false
            }
        }
        $sessions[$s].Ende = $c[0]
        $sessions[$s].Zeilen++
        if ($c[7] -like 'MAIL FROM*') { $sessions[$s].Mails++ }
        if ($c[7] -like '421 4.4.1*') { $sessions[$s].Timeout = $true }
    }

$sessions.Values |
    Sort-Object Zeilen -Descending |
    Select-Object -First 15 IP, Mails, Zeilen, Timeout,
        @{ n = 'Dauer_s'
           e = { [math]::Round(([datetime]$_.Ende - [datetime]$_.Start).TotalSeconds, 1) } } |
    Format-Table -AutoSize
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Elemento | Effetto |
|---|---|
| `$logPfad` | directory dei log del servizio di trasporto front-end; per il servizio di trasporto usare `Hub` al posto di `FrontEnd` |
| `$connector` | parte del nome del connettore; filtra tramite il campo `connector-id`, registrato come `Server\Name` |
| `$tag` | data UTC, perché i file di log sono denominati in base all'ora UTC |
| `-Filter "RECV$tag*.log"` | solo log di ricezione della giornata odierna |
| `Sort-Object Name -Descending`, `Select-Object -First 1` | il file più recente (ora più alta, numero di istanza più alto) |
| `$_ -notlike '#*'` | salta le intestazioni `#Software`, `#Version`, `#Log-Type`, `#Date`, `#Fields` |
| `$_ -split ','` | suddivide la riga CSV; i campi utilizzati 0, 2, 5 e 7 si trovano prima del primo testo libero e sono quindi stabili |
| `$c[2]` | `session-id`, la chiave di raggruppamento |
| `($c[5] -split ':')[0]` | indirizzo IPv4 da `remote-endpoint` (per gli endpoint IPv6 occorre adattare la suddivisione) |
| `$c[0]` come `Start` e `Ende` | primo e ultimo timestamp della sessione; `Ende` viene sovrascritto a ogni riga |
| `$c[7] -like 'MAIL FROM*'` | conta i messaggi tramite il comando `MAIL FROM` ricevuto |
| `$c[7] -like '421 4.4.1*'` | contrassegna le sessioni che Exchange ha terminato a causa del timeout complessivo |
| `Sort-Object Zeilen -Descending` | prima le sessioni più attive; in alternativa ordinare per `Dauer_s` |
| `Dauer_s` | differenza tra i timestamp ISO 8601 in secondi, arrotondata a una cifra decimale |

</details>

Nell'output è possibile riconoscere i sistemi interessati dal fatto che `Timeout` è impostato su `True` e `Dauer_s` è vicino a 600: la sessione è durata esattamente quanto consentito dal connettore. Le sessioni con molti messaggi e una durata nettamente inferiore a 600 secondi non sono critiche, anche se al momento sono le più lunghe. Per una panoramica delle fonti interessate, è sufficiente raggruppare le sessioni contrassegnate:

```powershell
$sessions.Values |
    Where-Object { $_.Timeout } |
    Group-Object IP |
    Select-Object Name, Count |
    Sort-Object Count -Descending
```

Due limitazioni dell'approccio: una sessione che attraversa il cambio d'ora si distribuisce su due file di log e appare accorciata nel singolo file; per una valutazione giornaliera occorre leggere tutti i file della giornata. Inoltre, il valore `Mails` conta i comandi `MAIL FROM`, quindi i tentativi, non i messaggi accettati.

## Adeguare i valori: su quale connettore e di quanto

I valori predefiniti proteggono il connettore rivolto a Internet, sul quale controparti arbitrarie possono occupare connessioni. Lì restano invariati; un MTA esterno legittimo si riconnette comunque. Va invece adeguato il connettore dedicato tramite il quale inviano i sistemi interni identificati. Se tale connettore non esiste, è possibile crearlo limitandolo agli IP mittenti con `RemoteIPRanges`; è preferibile ad aumentare il valore su `Default Frontend`. Lo stato attuale di tutti i connettori si ottiene con:

```powershell
Get-ReceiveConnector |
    Format-Table Name, TransportRole, ConnectionTimeout, ConnectionInactivityTimeout, TarpitInterval -AutoSize
```

L'adeguamento vero e proprio, qui con una durata complessiva di un'ora e timeout di inattività invariato:

```powershell
$werte = @{
    ConnectionTimeout           = '01:00:00'
    ConnectionInactivityTimeout = '00:05:00'
}
Set-ReceiveConnector -Identity 'EX01\Relay Applikationen' @werte
Set-ReceiveConnector -Identity 'EX01\Default EX01' @werte
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Parametro | Effetto |
|---|---|
| `ConnectionTimeout` | durata complessiva di una connessione; consentita da 00:00:01 a 1.00:00:00, deve essere superiore a `ConnectionInactivityTimeout` |
| `ConnectionInactivityTimeout` | tempo di inattività prima della chiusura; cinque minuti corrispondono al minimo RFC e possono restare invariati |
| `-Identity 'EX01\Relay Applikationen'` | il connettore front-end dei mittenti interni |
| `-Identity 'EX01\Default EX01'` | il connettore del servizio di trasporto sulla porta 2525 a cui il front-end inoltra la sessione |
| `@werte` | splatting: passa entrambi i parametri dalla tabella hash al cmdlet |

</details>

Per il valore vale quanto segue: deve essere superiore alla sessione legittima più lunga mostrata dall'analisi, con una riserva per i picchi di carico. Un'ora copre la maggior parte delle esecuzioni batch; per un'esecuzione notturna di due ore è necessario proporzionalmente di più, fino al massimo di un giorno. Tuttavia, nemmeno su un connettore interno il valore dovrebbe essere arbitrariamente elevato, perché `MaxInboundConnectionPerSource` (predefinito 20) e `MaxInboundConnection` (predefinito 5000) vengono anch'essi conteggiati: un client che, oltre a una connessione bloccata, continua ad aprirne di nuove raggiunge il limite per fonte tanto prima quanto più a lungo restano aperte le vecchie connessioni.

Per i mittenti che inviano `NOOP` tra i messaggi, `TarpitInterval` dovrebbe essere impostato su `00:00:00` sullo stesso connettore. Il ritardo Tarpit non è utile per mittenti interni autenticati o limitati per IP e allunga artificialmente ogni sessione.

La modifica sul lato Exchange risolve il sintomo. La soluzione più stabile è lato client: ristabilisce la connessione dopo un numero fisso di messaggi, come fanno Exchange con 20 e Postfix con cinque minuti. Con `.NET SmtpClient` ciò significa creare e scartare l'oggetto per blocchi di, ad esempio, 100 messaggi; con JavaMail, `Transport` viene chiuso e riaperto di conseguenza. In questo modo l'invio funziona anche verso destinazioni i cui timeout non possono essere adattati, in particolare Exchange Online, i cui connettori in ingresso non dispongono di parametri di timeout.

## Ulteriori limiti di tempo lungo il percorso

Il valore di Exchange non è l'unico limite. Firewall e load balancer dispongono di propri timer di inattività per le connessioni TCP: un profilo FastL4 su un F5 BIG-IP è impostato per impostazione predefinita su 300 secondi, un Azure Load Balancer su quattro minuti. Questi timer misurano l'inattività, non la durata complessiva, e intervengono quindi durante le pause nell'invio, ad esempio quando un processo batch legge dati dal database tra due blocchi. È sempre determinante il valore più basso dell'intero percorso. L'articolo [F5 BIG-IP come proxy outbound per l'invio massivo di posta](https://rafaelpfister.ch/blog/f5-big-ip-outbound-smtp-massenversand) descrive come dimensionare i timeout su un load balancer per connessioni SMTP persistenti.

## Fonti

1.  [Microsoft Learn: Set-ReceiveConnector](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-receiveconnector): riferimento con i valori predefiniti e gli intervalli ammessi di `ConnectionTimeout`, `ConnectionInactivityTimeout`, `TarpitInterval`, `MaxInboundConnection` e `MaxInboundConnectionPerSource` per server Mailbox ed Edge Transport.

2.  [Microsoft Learn: Set-SendConnector](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-sendconnector): `ConnectionInactivityTimeOut` e `SmtpMaxMessagesPerConnection` sul lato di invio.

3.  [Microsoft Learn: Protocol logging](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/protocol-logging): posizioni di archiviazione, nomi dei file e struttura dei campi dei log del protocollo SMTP per front-end e servizio di trasporto.

4.  [Microsoft Learn: Mail routing in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing): il servizio di trasporto front-end come proxy senza stato davanti al servizio di trasporto.

5.  [RFC 5321, sezione 4.5.3.2 Timeouts](https://www.rfc-editor.org/rfc/rfc5321.html#section-4.5.3.2): tempi minimi di attesa per ogni passaggio del protocollo, motivazione dei dieci minuti dopo il punto finale e comportamento in caso di `421` nella sezione 3.8.

6.  [Postfix: postconf(5)](https://www.postfix.org/postconf.5.html): `smtp_connection_reuse_time_limit` (300s) e `smtpd_timeout` come esempio di MTA che mantiene autonomamente brevi le sessioni.

7.  [Broadcom Knowledge Base: Quarantine notification process appears to be failing, logs may show 421 4.4.1 Connection timed out](https://knowledge.broadcom.com/external/article/154389/quarantine-notification-process-appears.html): caso documentato di un gateway che, con keepalive `NOOP` e Tarpit, raggiunge il timeout complessivo di Exchange.

8.  [Microsoft Learn: SmtpClient Class](https://learn.microsoft.com/en-us/dotnet/api/system.net.mail.smtpclient): riutilizzo della connessione attraverso più chiamate a `Send` e `QUIT` solo al momento di `Dispose`.
