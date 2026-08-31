---
title: "Determinare il profilo di carico di un server di posta: burst, picchi di velocità e struttura dei destinatari dal Message Tracking"
navTitle: "Determinare il profilo di carico"
description: "Quante e-mail al minuto elabora realmente il vostro server di posta e quali sono i picchi? Come determinare il reale profilo di carico dal Message Tracking di Exchange con PowerShell: velocità al minuto e all’ora, durata dei burst, struttura dei destinatari, dimensioni dei messaggi e i tipici errori di analisi."
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
translationSourceHash: 298fabdf5f8f248539ea8a119681be130cd76f5c8ebc35db5d0c61e1126251b0
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:28:45.969Z
translationReview: automatic
url: https://rafaelpfister.ch/it/blog/determinare-il-profilo-di-carico-di-un-mail-server-burst-picchi-e-struttura-dei-destinatari-dal
---

# Determinare il profilo di carico di un server di posta: burst, picchi di velocità e struttura dei destinatari dal Message Tracking

Che si debba sostituire un gateway, dimensionare un server o pianificare una finestra di manutenzione: prima o poi ogni amministratore di posta deve rispondere alla domanda su quanto elabori effettivamente il proprio sistema. L’intuito è regolarmente fuorviante, perché il traffico e-mail raramente è uniforme. Un sistema che registra in media 20 e-mail al minuto nell’arco della giornata potrebbe doverne elaborare 400 al minuto per un’ora durante un ciclo di fatturazione. Chi conosce solo la media dimensiona il sistema per il problema sbagliato.

Un profilo di carico utile è composto da quattro indicatori: la velocità media (al minuto, all’ora, al giorno), i burst (quanto è alto il picco, quanto dura, quando si verifica), la struttura dei destinatari (quanti destinatari diversi, quali domini di destinazione) e le dimensioni dei messaggi. Tutti e quattro sono disponibili nel Message Tracking e in Exchange si possono calcolare con poche righe di PowerShell.

## La fonte dei dati: Message Tracking

Exchange registra ogni messaggio nel Message Tracking Log. Prima di analizzare i dati, verificate fino a quando risalgono; lo standard è di 30 giorni, ma un limite di dimensione ridotto può accorciare notevolmente la conservazione effettiva:

```powershell
Get-TransportService |
    Select-Object Name, MessageTrackingLogMaxAge,
        MessageTrackingLogMaxDirectorySize, MessageTrackingLogPath
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Get-TransportService` | Elenca tutti i server di trasporto dell’organizzazione; senza parametri tutti i server |
| `Select-Object Name, MessageTrackingLog…` | Riduce l’output alle proprietà indicate: periodo di conservazione, limite di dimensione della directory dei log e percorso del log |

</details>

Per un profilo di carico, il periodo dovrebbe coprire almeno un ciclo batch completo dell’azienda: cicli mensili di fatturazione, elaborazione delle buste paga, newsletter. Una settimana è il minimo, un mese è meglio.

## Raccogliere i dati grezzi: un evento per messaggio

La decisione preliminare più importante: quale evento conta come «una e-mail»? Il Message Tracking scrive più voci per messaggio (RECEIVE all’accettazione, SEND all’inoltro verso l’hop successivo, DELIVER alla consegna nella cassetta postale, oltre a AGENTINFO, HAREDIRECT e altri). Chi conta semplicemente tutte le righe sovrastima il volume di molte volte. Per il carico in ingresso contate RECEIVE, per il carico in uscita verso lo smarthost o Internet SEND.

```powershell
$start = (Get-Date).AddDays(-7)
$events = Get-TransportService | ForEach-Object {
    Get-MessageTrackingLog -Server $_.Name -ResultSize Unlimited `
        -Start $start -EventId RECEIVE
}
"{0} Nachrichten seit {1:yyyy-MM-dd}" -f $events.Count, $start
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Server $_.Name` | Interroga il log di tracking del rispettivo server di trasporto dalla pipeline |
| `-ResultSize Unlimited` | Rimuove il limite standard di 1'000 voci restituite |
| `-Start $start` | Limite temporale inferiore della query; qui gli ultimi sette giorni |
| `-EventId RECEIVE` | Filtra esattamente un evento per messaggio, qui l’accettazione da parte del servizio di trasporto |
| `-f` | Operatore di formattazione: inserisce i valori a destra nei segnaposto `{0}` e `{1}` della stringa |

</details>

La query viene intenzionalmente eseguita su tutti i server di trasporto, poiché ogni server registra solo la propria quota. Chi interroga un solo server, in un cluster vede soltanto una frazione del carico.

## Velocità al minuto e all’ora: qui emergono i burst

L’aggregazione è un Group-Object sul timestamp arrotondato. I minuti con i valori più alti sono direttamente i candidati ai burst:

```powershell
$proMinute = $events |
    Group-Object { $_.Timestamp.ToString("yyyy-MM-dd HH:mm") } |
    Sort-Object Count -Descending

$proMinute | Select-Object -First 10 Name, Count
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Group-Object { … }` | Raggruppa in base al valore restituito dal blocco di script, qui il timestamp troncato al minuto |
| `Sort-Object Count -Descending` | Ordina i gruppi in ordine decrescente per numero; i minuti più intensi sono in cima |
| `Select-Object -First 10 Name, Count` | Restituisce solo i dieci gruppi più grandi, ridotti a minuto e numero |

</details>

Lo stesso per ora e come andamento giornaliero (a quale ora il carico è tipicamente di quale intensità):

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

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Group-Object { … ToString("yyyy-MM-dd HH") }` | Raggruppa per ore intere di un giorno specifico |
| `Group-Object { … ToString("HH") }` | Raggruppa solo per ora e aggrega quindi tutti i giorni: l’andamento giornaliero |
| `Sort-Object Count -Descending` | Le ore più intense in cima |
| `Sort-Object Name` | Ordina l’andamento giornaliero cronologicamente per ora invece che per numero |
| `Format-Table Name, Count` | Output tabellare delle due colonne |

</details>

Un burst è caratterizzato solo quando, oltre al picco, se ne conosce anche la durata. Un picco di 400/min che dura due minuti richiede altro rispetto allo stesso picco per un’ora. Contate i minuti sopra una soglia:

```powershell
$schwelle = 100
$burstMinuten = $proMinute | Where-Object Count -ge $schwelle
"{0} Minuten mit >= {1}/min, Peak: {2}/min" -f $burstMinuten.Count,
    $schwelle, ($proMinute | Select-Object -First 1).Count
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Where-Object Count -ge $schwelle` | Filtra i minuti con almeno il numero di messaggi della soglia (sintassi semplificata senza blocco di script) |
| `Select-Object -First 1` | Primo gruppo dell’elenco ordinato in ordine decrescente, quindi il minuto più intenso |
| `-f` | Operatore di formattazione: inserisce numero, soglia e picco nei segnaposto da `{0}` a `{2}` |

</details>

Se i minuti di burst sono consecutivi (visibili direttamente nell’output di `$burstMinuten | Sort-Object Name`), si tratta di un’esecuzione batch. Annotate ora di inizio, durata e schema di ripetizione, perché è proprio questa finestra che l’infrastruttura deve sostenere.

## Struttura dei destinatari: quanti obiettivi, quali domini

Per i gateway, la varietà dei destinatari è spesso più importante della mera velocità, perché per ogni destinatario vengono effettuate ricerche (routing, policy, regole di cifratura). Un’e-mail a una lista di distribuzione con 5'000 membri genera un carico diverso da 5'000 e-mail individuali. Il campo `RecipientCount` e l’elenco dei destinatari forniscono entrambe le prospettive:

```powershell
"Nachrichten: {0}, Empfänger-Zustellungen: {1}" -f $events.Count,
    ($events | Measure-Object RecipientCount -Sum).Sum

$alleEmpfaenger = $events | ForEach-Object { $_.Recipients } |
    ForEach-Object { $_.ToLower() }
"Eindeutige Empfänger: {0}" -f ($alleEmpfaenger | Sort-Object -Unique).Count
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Measure-Object RecipientCount -Sum` | Somma il campo `RecipientCount` su tutti gli eventi: il numero delle consegne ai destinatari |
| `ForEach-Object { $_.Recipients }` | Espande l’elenco dei destinatari di ogni evento in singoli indirizzi |
| `ForEach-Object { $_.ToLower() }` | Normalizza gli indirizzi in minuscolo, affinché i duplicati siano riconosciuti come tali |
| `Sort-Object -Unique` | Ordina e rimuove i duplicati; `Count` restituisce quindi gli indirizzi univoci |

</details>

La distribuzione dei domini mostra dove fluisce il traffico. Se predominano Gmail e Microsoft, i loro limiti di velocità e la reputazione del proprio IP determinano il throughput raggiungibile, non il proprio hardware:

```powershell
$alleEmpfaenger |
    ForEach-Object { ($_ -split "@")[1] } |
    Group-Object |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `($_ -split "@")[1]` | Divide l’indirizzo in corrispondenza di `@` e conserva la parte del dominio |
| `Group-Object` | Raggruppa senza argomento in base al valore stesso, qui il dominio |
| `Sort-Object Count -Descending` | I domini più frequenti in cima |
| `Select-Object -First 10 Name, Count` | Limita l’output ai primi 10 |

</details>

E nella direzione opposta: quali mittenti (applicazioni, cassette postali funzionali) generano effettivamente il carico? Questo risponde anche alla domanda su quali sistemi debbano essere considerati in una migrazione:

```powershell
$events |
    Group-Object Sender |
    Sort-Object Count -Descending |
    Select-Object -First 10 Name, Count
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Group-Object Sender` | Raggruppa in base al campo `Sender` (parametro posizionale `-Property`) |
| `Sort-Object Count -Descending` | I mittenti con il maggior numero di messaggi in cima |
| `Select-Object -First 10 Name, Count` | Limita l’output ai primi 10 |

</details>

## Dimensioni dei messaggi: byte al secondo invece di e-mail al secondo

Le indicazioni di throughput dei gateway si riferiscono spesso al volume di dati, non al numero di messaggi. Due sistemi con la stessa velocità di e-mail differiscono di un fattore 100 se uno invia notifiche da 50 KB e l’altro PDF di fatture da 5 MB. Il campo `TotalBytes` fornisce la distribuzione:

```powershell
$events | Measure-Object TotalBytes -Average -Maximum -Sum |
    Select-Object @{n = "MittelKB"; e = { [math]::Round($_.Average / 1KB) } },
        @{n = "MaxMB"; e = { [math]::Round($_.Maximum / 1MB, 1) } },
        @{n = "TotalGB"; e = { [math]::Round($_.Sum / 1GB, 1) } }
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Measure-Object TotalBytes -Average -Maximum -Sum` | Calcola media, massimo e somma del campo `TotalBytes` in un unico passaggio |
| `@{n = "…"; e = { … }}` | Proprietà calcolata: `n` denomina la colonna, `e` restituisce il valore tramite blocco di script, qui la conversione in KB, MB e GB |

</details>

Moltiplicate la velocità di burst per la dimensione media nella finestra del burst e otterrete il requisito di larghezza di banda che un nuovo gateway o un collegamento WAN deve sostenere.

## Velocità in tempo reale senza tracking: uno sguardo alle code

Per osservare il momento (il server sta elaborando molto, qualcosa si sta accumulando?) non serve alcun tracking: le code lo mostrano direttamente. `IncomingRate` e `OutgoingRate` sono e-mail al minuto, livellate sugli ultimi minuti:

```powershell
Get-TransportService |
    ForEach-Object { Get-Queue -Server $_.Name } |
    Sort-Object MessageCount -Descending |
    Select-Object Identity, Status, MessageCount, IncomingRate,
        OutgoingRate, NextHopDomain, LastError |
    Format-Table -AutoSize
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Get-Queue -Server $_.Name` | Elenca le code di trasporto del rispettivo server dalla pipeline |
| `Sort-Object MessageCount -Descending` | Le code più piene in cima |
| `Select-Object Identity, Status, …` | Limita l’output ai campi rilevanti per la valutazione del carico |
| `Format-Table -AutoSize` | Adatta la larghezza delle colonne al contenuto invece di troncare le colonne |

</details>

Interpretazione: una coda `Submission` con velocità elevata e profondità 0 significa che il server elabora il carico senza accumularlo. `MessageCount` elevato con `OutgoingRate` vicino a zero indica un arretrato. `Status Retry` con un messaggio 4xx in `LastError` significa che la controparte sta limitando la velocità. Le code `Shadow` con messaggi presenti sono invece normali: sono copie di ridondanza per il server partner, non un arretrato.

Per una curva continua durante una finestra di carico è adatto il Performance Counter delle code di trasporto, qui ogni cinque secondi per un minuto:

```powershell
Get-Counter "\MSExchangeTransport Queues(_total)\Messages Completed Delivery Per Second" `
    -SampleInterval 5 -MaxSamples 12
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `"\MSExchangeTransport Queues(_total)\…"` | Percorso del Performance Counter (parametro posizionale `-Counter`); l’istanza `_total` somma tutte le code |
| `-SampleInterval 5` | Intervallo tra due misurazioni in secondi |
| `-MaxSamples 12` | Numero di misurazioni; 12 misurazioni ogni 5 secondi corrispondono a un minuto |

</details>

## Altri sistemi: lo stesso principio con CSV

Gateway e appliance forniscono generalmente un’esportazione CSV del tracking invece di oggetti PowerShell. La procedura resta identica (scegliere un evento per e-mail, raggruppare per finestre temporali), cambia solo lo strumento, ad esempio Python:

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

## I cinque tipici errori di analisi

**Eventi multipli per e-mail.** La fonte di errore più comune: contare le righe anziché i messaggi. Verificate con `$events | Group-Object EventId` cosa contiene effettivamente il vostro set di dati e filtrate esattamente un evento per messaggio.

**Esportazioni troncate.** Molte funzioni di esportazione restituiscono al massimo 10'000 o 50'000 righe e poi troncono silenziosamente, spesso proprio nel mezzo del burst più grande. Un numero di righe sospettosamente tondo è un segnale d’allarme. Verificate sempre se l’intervallo temporale dei dati corrisponde a quello richiesto.

**Cicli del gateway.** Se il flusso di posta passa attraverso una stazione intermedia (gateway di cifratura, appliance di igiene) e poi ritorna, la stessa e-mail compare più volte nel tracking. Deduplicate tramite il Message-ID o filtrate su un punto univoco della catena.

**Fusi orari.** `Get-MessageTrackingLog` restituisce timestamp nell’ora locale del server, mentre le esportazioni CSV delle appliance sono spesso in UTC. Un burst che sembra verificarsi alle 13 potrebbe in realtà essere il batch delle 15. Chiarite la base temporale prima di interpretare i dati.

**Finestre troppo brevi.** Un profilo di carico basato su due giorni tranquilli è inutile se manca il ciclo mensile di fatturazione. La finestra di analisi deve includere i cicli batch noti; chiedete ai responsabili delle applicazioni i loro piani di invio prima di definirla.

## Cosa fare con il profilo

Alla fine avrete quattro numeri in una pagina: velocità media, burst (picco, durata, momento, schema di ripetizione), struttura dei destinatari (destinatari univoci per esecuzione, domini principali) e distribuzione delle dimensioni. Con questi dati è possibile dimensionare i gateway, collocare le finestre di manutenzione nelle ore notturne con carico reale pari a zero e formulare criteri di accettazione, ad esempio: il nuovo sistema deve elaborare senza errori il doppio del picco misurato. L’articolo [Test di carico SMTP con Apache JMeter nella pratica](/blog/jmeter-smtp-lasttest-html-report) mostra come trasformare un simile profilo in un test di carico riproducibile.

## Fonti

1.  [Microsoft Learn: Get-MessageTrackingLog](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog): riferimento alla query di tracking, inclusi tutti i campi quali EventId, RecipientCount e TotalBytes.

2.  [Microsoft Learn: Message tracking](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): struttura dei log di tracking, tipi di evento e configurazione della conservazione e della dimensione della directory.

3.  [Microsoft Learn: Get-Queue](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue): riferimento alla query delle code, inclusi i campi IncomingRate, OutgoingRate e Velocity.

4.  [Microsoft Learn: Queues and messages in queues](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues): tipi di code, Shadow Redundancy e significato dei valori di stato.
