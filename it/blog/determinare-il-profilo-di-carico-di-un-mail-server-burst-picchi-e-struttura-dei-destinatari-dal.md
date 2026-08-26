---
title: "Determinare il profilo di carico di un mail server: burst, picchi e struttura dei destinatari dal Message Tracking"
navTitle: "Determinare il profilo di carico"
description: "Quante e-mail al minuto elabora realmente il vostro mail server e quanto sono alti i picchi? Come determinare il vero profilo di carico dal Message Tracking di Exchange con PowerShell: tassi al minuto e all'ora, durata dei burst, struttura dei destinatari, dimensioni dei messaggi. Con le tipiche insidie dell'analisi."
date: "2026-08-25"
kategorie: "SMTP e flusso di posta"
timeToRead: "9 min di lettura"
themen:
  - smtp-mailflow
  - exchange-onprem-hybrid
produkte:
  - "exchange-on-premises"
  - "uebergreifend"
protokolle:
  - "smtp"
  - "powershell"
  - "troubleshooting"
slug: "determinare-il-profilo-di-carico-di-un-mail-server-burst-picchi-e-struttura-dei-destinatari-dal"
translationId: "article-1ff17a188d73e289"
aiPrompt: |
  Du bist mein Mailflow-Assistent. Hilf mir Schritt für Schritt, das Lastprofil meines Mailservers zu ermitteln: 1. Die richtige Datenquelle wählen (Message Tracking, Gateway-Logs) und das passende Event pro Nachricht bestimmen. 2. Raten pro Minute, Stunde und Tag berechnen und Bursts mit Dauer und Peak charakterisieren. 3. Empfängerstruktur, Domain-Verteilung und Nachrichtengrössen auswerten. Weise mich auf Doppelzählungen, Export-Limits und Zeitzonen-Fallen hin.
translationOf: mailserver-lastprofil-ermitteln
url: https://rafaelpfister.ch/it/blog/determinare-il-profilo-di-carico-di-un-mail-server-burst-picchi-e-struttura-dei-destinatari-dal
translationSourceHash: 16095cf53ce6f67abe31387ce2f02958eacc3898d3a42b61ad8c7b885ab7ce5d
translationModel: gpt-5.6-terra
translatedAt: 2026-08-26T04:10:01.923Z
translationReview: automatic
---

# Determinare il profilo di carico di un mail server: burst, picchi e struttura dei destinatari dal Message Tracking

Che si debba sostituire un gateway, dimensionare un server o pianificare una finestra di manutenzione: prima o poi ogni amministratore di posta deve rispondere alla domanda su quanto elabori effettivamente il proprio sistema. L'intuito è regolarmente fuorviante, perché il traffico e-mail raramente è uniforme. Un sistema che registra in media 20 e-mail al minuto nell'arco della giornata può doverne elaborare 400 al minuto per un'ora durante l'elaborazione delle fatture. Chi conosce solo la media dimensiona il sistema per il problema sbagliato.

Un profilo di carico utile comprende quattro metriche: il tasso medio (al minuto, all'ora, al giorno), i burst (quanto è alto il picco, quanto dura, quando si verifica), la struttura dei destinatari (quanti destinatari diversi, quali domini di destinazione) e le dimensioni dei messaggi. Tutti e quattro sono disponibili nel Message Tracking e, in Exchange, possono essere calcolati con poche righe di PowerShell.

## La fonte dei dati: Message Tracking

Exchange registra ogni messaggio nel Message Tracking Log. Prima di effettuare l'analisi, verificate fino a quando risalgono i dati; lo standard è di 30 giorni, ma un limite di dimensione ristretto può ridurre sensibilmente la conservazione effettiva:

```powershell
Get-TransportService |
    Select-Object Name, MessageTrackingLogMaxAge,
        MessageTrackingLogMaxDirectorySize, MessageTrackingLogPath
```

Per un profilo di carico, il periodo dovrebbe coprire almeno un ciclo batch completo dell'azienda: elaborazioni mensili delle fatture, conteggi salariali, newsletter. Una settimana è il minimo, un mese è meglio.

## Raccogliere i dati grezzi: un evento per messaggio

La decisione preliminare più importante: quale evento conta come «una e-mail»? Il Message Tracking scrive più voci per messaggio (RECEIVE al momento dell'accettazione, SEND all'inoltro al successivo hop, DELIVER alla consegna nella mailbox, oltre a AGENTINFO, HAREDIRECT e altri). Chi conta semplicemente tutte le righe sovrastima il volume di molte volte. Per il carico in ingresso contate RECEIVE, per il carico in uscita verso lo smarthost o Internet SEND.

```powershell
$start = (Get-Date).AddDays(-7)
$events = Get-TransportService | ForEach-Object {
    Get-MessageTrackingLog -Server $_.Name -ResultSize Unlimited `
        -Start $start -EventId RECEIVE
}
"{0} Nachrichten seit {1:yyyy-MM-dd}" -f $events.Count, $start
```

La query viene eseguita deliberatamente su tutti i server di trasporto, poiché ogni server registra solo la propria parte. Interrogando un solo server, in un cluster si vede solo una frazione del carico.

## Tassi al minuto e all'ora: qui emergono i burst

L'aggregazione avviene con Group-Object sul timestamp arrotondato. I minuti principali sono direttamente i vostri candidati burst:

```powershell
$proMinute = $events |
    Group-Object { $_.Timestamp.ToString("yyyy-MM-dd HH:mm") } |
    Sort-Object Count -Descending

$proMinute | Select-Object -First 10 Name, Count
```

Lo stesso per ora e come andamento giornaliero (a quale ora il carico è tipicamente più elevato):

```powershell
$events |
    Group-Object { $_.Timestamp.ToString("yyyy-MM-dd HH") } |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count

$events |
    Group-Object { $_.Timestamp.ToString("HH") } |
    Sort-Object Name |
    Format-Table Name, Count
```

Un burst è caratterizzato solo quando, oltre al picco, ne conoscete anche la durata. Un picco di 400/min che dura due minuti è un requisito diverso dallo stesso picco per un'ora. Contate i minuti sopra una soglia:

```powershell
$schwelle = 100
$burstMinuten = $proMinute | Where-Object Count -ge $schwelle
"{0} Minuten mit >= {1}/min, Peak: {2}/min" -f $burstMinuten.Count,
    $schwelle, ($proMinute | Select-Object -First 1).Count
```

Se i minuti di burst sono consecutivi (visibili direttamente nell'output di `$burstMinuten | Sort-Object Name`), si tratta di un'esecuzione batch. Annotate ora di inizio, durata e schema di ripetizione, poiché è esattamente questa finestra che l'infrastruttura deve sostenere.

## Struttura dei destinatari: quanti obiettivi, quali domini

Per i gateway, la varietà dei destinatari è spesso più importante del puro tasso, poiché per ogni destinatario sono necessarie ricerche (routing, policy, regole di crittografia). Un'e-mail a una lista di distribuzione con 5'000 membri comporta un carico diverso da 5'000 e-mail singole. Il campo `RecipientCount` e l'elenco dei destinatari forniscono entrambe le prospettive:

```powershell
"Nachrichten: {0}, Empfänger-Zustellungen: {1}" -f $events.Count,
    ($events | Measure-Object RecipientCount -Sum).Sum

$alleEmpfaenger = $events | ForEach-Object { $_.Recipients } |
    ForEach-Object { $_.ToLower() }
"Eindeutige Empfänger: {0}" -f ($alleEmpfaenger | Sort-Object -Unique).Count
```

La distribuzione per dominio mostra dove fluisce il traffico. Se predominano Gmail e Microsoft, sono i loro rate limit e la reputazione del proprio IP a determinare il throughput raggiungibile, non il proprio hardware:

```powershell
$alleEmpfaenger |
    ForEach-Object { ($_ -split "@")[1] } |
    Group-Object |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count
```

E nella direzione opposta: quali mittenti (applicazioni, mailbox funzionali) generano effettivamente il carico? Questo risponde anche alla domanda su quali sistemi debbano essere considerati durante una migrazione:

```powershell
$events |
    Group-Object Sender |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count
```

## Dimensioni dei messaggi: byte al secondo invece di e-mail al secondo

Le indicazioni di throughput dei gateway spesso si riferiscono al volume di dati, non al numero di messaggi. Due sistemi con lo stesso tasso di e-mail differiscono di un fattore 100 se uno invia notifiche da 50 KB e l'altro PDF di fatture da 5 MB. Il campo `TotalBytes` fornisce la distribuzione:

```powershell
$events | Measure-Object TotalBytes -Average -Maximum -Sum |
    Select-Object @{n = "MittelKB"; e = { [math]::Round($_.Average / 1KB) } },
        @{n = "MaxMB"; e = { [math]::Round($_.Maximum / 1MB, 1) } },
        @{n = "TotalGB"; e = { [math]::Round($_.Sum / 1GB, 1) } }
```

Moltiplicate il tasso di burst per la dimensione media nella finestra di burst e avrete il requisito di larghezza di banda che un nuovo gateway o un collegamento WAN deve sostenere.

## Tassi in tempo reale senza tracking: uno sguardo alle code

Per una visione istantanea (il server sta elaborando molto in questo momento, qualcosa si sta accumulando?) non serve alcun tracking: le code lo mostrano direttamente. `IncomingRate` e `OutgoingRate` sono e-mail al minuto, smussate sugli ultimi minuti:

```powershell
Get-TransportService |
    ForEach-Object { Get-Queue -Server $_.Name } |
    Sort-Object MessageCount -Descending |
    Select-Object Identity, Status, MessageCount, IncomingRate,
        OutgoingRate, NextHopDomain, LastError |
    Format-Table -AutoSize
```

Interpretazione: una coda `Submission` con tasso elevato e profondità 0 significa che il server elabora il carico senza accumularlo. `MessageCount` elevato con `OutgoingRate` vicino a zero indica un arretrato. `Status Retry` con un messaggio 4xx in `LastError` significa che la controparte sta limitando il traffico. Le code `Shadow` con messaggi presenti, invece, sono normali: sono copie ridondanti per il server partner, non un arretrato.

Per una curva continua durante una finestra di carico è adatto il contatore delle prestazioni delle code di trasporto, qui ogni cinque secondi per un minuto:

```powershell
Get-Counter "\MSExchangeTransport Queues(_total)\Messages Completed Delivery Per Second" `
    -SampleInterval 5 -MaxSamples 12
```

## Altri sistemi: lo stesso principio con CSV

Gateway e appliance forniscono generalmente un'esportazione CSV del tracking anziché oggetti PowerShell. Il procedimento rimane identico (scegliere un evento per e-mail, raggruppare per finestre temporali), cambia solo lo strumento, ad esempio Python:

```python
import csv, collections, datetime

per_min = collections.Counter()
with open("tracking-export.csv", encoding="utf-8") as f:
    reader = csv.reader(f)
    next(reader)
    for row in reader:
        if "response '2" not in row[6]:   # nur finale Zustellungen
            continue
        d = datetime.datetime.strptime(row[0][:16], "%Y-%m-%d %H:%M")
        per_min[d.strftime("%Y-%m-%d %H:%M")] += 1

print(per_min.most_common(10))
```

## Le cinque classiche insidie dell'analisi

**Eventi multipli per e-mail.** La fonte di errore più comune: contare le righe anziché i messaggi. Verificate con `$events | Group-Object EventId` cosa contiene effettivamente il vostro set di dati e filtrate esattamente un evento per messaggio.

**Esportazioni troncate.** Molte funzioni di esportazione forniscono al massimo 10'000 o 50'000 righe e poi le interrompono senza avviso, volentieri nel mezzo del burst più grande. Un numero di righe sospettosamente tondo è un segnale d'allarme. Verificate sempre che il periodo dei dati corrisponda al periodo richiesto.

**Loop del gateway.** Se il flusso di posta passa attraverso una stazione intermedia (gateway di crittografia, appliance di igiene) e poi ritorna, la stessa e-mail compare più volte nel tracking. Deduplicate tramite il Message-ID oppure filtrate su un punto univoco della catena.

**Fusi orari.** `Get-MessageTrackingLog` fornisce timestamp nell'ora locale del server, mentre le esportazioni CSV delle appliance spesso sono in UTC. Un burst che sembra verificarsi alle 13 potrebbe in realtà essere il batch delle 15. Chiarite la base temporale prima di interpretare i dati.

**Finestre troppo brevi.** Un profilo di carico basato su due giorni tranquilli è inutile se manca l'elaborazione mensile delle fatture. La finestra di analisi deve includere i cicli batch noti; chiedete ai responsabili delle applicazioni i loro piani di invio prima di definire la finestra.

## Cosa fare con il profilo

Alla fine avrete quattro valori in una pagina: tasso medio, burst (picco, durata, momento, schema di ripetizione), struttura dei destinatari (destinatari univoci per esecuzione, domini principali) e distribuzione delle dimensioni. Questo consente di dimensionare i gateway, pianificare le finestre di manutenzione nelle ore notturne con carico reale nullo e formulare criteri di accettazione, ad esempio: il nuovo sistema deve elaborare senza errori il doppio del picco misurato. L'articolo [Test di carico SMTP con Apache JMeter nella pratica](/blog/jmeter-smtp-lasttest-html-report) mostra come trasformare tale profilo in un test di carico riproducibile.

## Fonti

1.  [Microsoft Learn: Get-MessageTrackingLog](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog): riferimento per la query di tracking, inclusi tutti i campi quali EventId, RecipientCount e TotalBytes.

2.  [Microsoft Learn: Message tracking](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): struttura dei log di tracking, tipi di evento e configurazione della conservazione e della dimensione della directory.

3.  [Microsoft Learn: Get-Queue](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue): riferimento per la query delle code, inclusi i campi IncomingRate, OutgoingRate e Velocity.

4.  [Microsoft Learn: Queues and messages in queues](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues): tipi di coda, Shadow Redundancy e significato dei valori di stato.
