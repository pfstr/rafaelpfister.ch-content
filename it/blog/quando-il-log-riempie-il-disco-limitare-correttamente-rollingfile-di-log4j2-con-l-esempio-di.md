---
title: "Quando il log riempie il disco: limitare correttamente RollingFile di log4j2, con l'esempio di totemomail"
navTitle: "Spazio su disco log4j2"
description: "Nel peggiore dei casi, un volume di log che si riempie può bloccare l'intero gateway. Perché la combinazione di rotazione temporale e per dimensione senza %i produce un singolo file enorme, come strategy.max limita la conservazione, quale ruolo svolge il livello di log e dove totemomail nasconde questi valori."
date: "2026-09-04"
kategorie: "Totemomail"
timeToRead: "9 min di lettura"
themen:
  - totemomail
produkte:
  - "totemomail"
protokolle:
  - "troubleshooting"
  - "storage"
slug: "quando-il-log-riempie-il-disco-limitare-correttamente-rollingfile-di-log4j2-con-l-esempio-di"
translationId: "article-c400eee99d90052d"
translationOf: log4j2-rollingfile-plattenplatz-totemomail
url: https://rafaelpfister.ch/it/blog/quando-il-log-riempie-il-disco-limitare-correttamente-rollingfile-di-log4j2-con-l-esempio-di
translationSourceHash: 39952348654f81231356634fc8b434cbfecdea73118db7ff1add02720283792b
translationModel: gpt-5.6-terra
translatedAt: 2026-09-04T08:16:52.783Z
translationReview: automatic
---

# Quando il log riempie il disco: limitare correttamente RollingFile di log4j2, con l'esempio di totemomail

Un gateway di posta basato su Java scrive quantità sorprendenti in modalità DEBUG. Un solo giorno di carico può generare diversi gigabyte di log di trace e, se il volume dei log è dimensionato in modo ridotto, si riempie. La conseguenza è spiacevole: il processo Java non riesce più a scrivere nel proprio log, il framework di logging entra in uno stato di errore e, anche dopo aver liberato spazio, riprende a scrivere solo dopo un riavvio. In un gateway di posta, inoltre, un disco pieno può compromettere lo spooling e la consegna. Il motivo è quasi sempre una rotazione dei log configurata, ma che non funziona come si presume.

Il seguente articolo spiega la rotazione di log4j2 proprio in questo contesto, prima in generale e poi concretamente per totemomail (basato su Apache James e log4j2). Il punto chiave è una singola indicazione nel pattern del file, facile da trascurare.

## Come ruota log4j2

Il `RollingFileAppender` di log4j2 combina due componenti: una o più **TriggeringPolicies** decidono *quando* ruotare, mentre una **RolloverStrategy** decide *come* vengono denominati i file di archivio e quanti ne vengono mantenuti. In genere vengono usate contemporaneamente due policy:

- `TimeBasedTriggeringPolicy`: ruota in base al tempo, di solito ogni giorno.
- `SizeBasedTriggeringPolicy`: ruota non appena il file attivo raggiunge una certa dimensione, ad esempio 100 MB.

Durante il rollover, il file attivo viene rinominato e archiviato. Il nome del file di archivio è determinato dal `filePattern`, che contiene due segnaposto la cui interazione fa la differenza decisiva.

<details class="options-details">
<summary>Panoramica delle opzioni</summary>

| Segnaposto | Significato |
|---|---|
| `%d{...}` | Data/ora del rollover secondo il formato specificato, ad esempio `%d{yyyy-MM-dd}` per il giorno |
| `%i` | L'indice calcolato del file di archivio, un contatore che aumenta a ogni rollover |
| `%03i` | Lo stesso indice, completato con zeri fino a tre cifre |
| `.gz` / `.zip` alla fine del pattern | L'archivio viene compresso durante il rollover |

</details>

La documentazione di log4j2 per il Rolling File Appender contiene il riferimento completo; la tabella sopra elenca solo gli elementi essenziali per la rotazione per dimensione e tempo.

## La trappola di %i

È proprio qui che si trova l'errore che riempie i dischi. Chi assegna un nome basato solo sulla data, quindi `filePattern = trace.log.%d{yyyy-MM-dd}`, e contemporaneamente imposta una policy di dimensione di 100 MB, non ottiene molti file da 100 MB al giorno, ma un unico file che continua a crescere senza limiti. La rotazione per dimensione non ha una destinazione separata in cui scrivere il blocco successivo, poiché il pattern non contiene un contatore. La documentazione di log4j2 è chiara su questo punto:

> When combined with a time-based triggering policy, the filePattern attribute of the Appender should contain an `%i` conversion pattern. Otherwise, the target file will be overwritten on each rollover.

Senza `%i`, la combinazione di rotazione temporale e per dimensione è quindi errata; a seconda della strategia, il file viene sovrascritto oppure cresce oltre la dimensione impostata. In pratica significa che il limite di 100 MB non entra mai in funzione: una giornata di carico scrive tutto in un unico file, che raggiunge diversi gigabyte. La correzione consiste nell'aggiungere il segnaposto al pattern:

```text
filePattern = trace.log.%d{yyyy-MM-dd}.%i
```

In questo modo, ogni rollover da 100 MB crea un file distinto con indice (`trace.log.2026-09-04.1`, `.2`, `.3`) e il limite di dimensione funziona come previsto.

## Conservazione tramite strategy.max

L'indice è al tempo stesso il requisito affinché la conservazione funzioni. La `DefaultRolloverStrategy` dispone dell'attributo `max`, che indica il numero massimo di file di archivio da mantenere; oltre tale limite vengono eliminati i più vecchi. Senza `%i` non esiste alcun indice che `max` possa contare, quindi non viene eliminato nulla e i vecchi file datati si accumulano.

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Attributo | Effetto |
|---|---|
| `max` | Numero massimo di file di archivio conservati; oltre questo valore vengono rimossi i più vecchi |
| `min` | Valore minimo dell'indice (predefinito 1) |
| `fileIndex="min"` | Il file più recente riceve l'indice `min`, il più vecchio `max` |
| `fileIndex="max"` (predefinito) | Il file più vecchio riceve l'indice `min`, il più recente `max` |
| `fileIndex="nomax"` | Non viene mai eliminato nulla, i nuovi archivi ricevono indici progressivamente crescenti |

</details>

Dalla dimensione e dal numero deriva il limite massimo complessivo: 100 MB per file per `max=10` limita il log a circa un gigabyte, indipendentemente da quanto venga scritto. Chi necessita di un controllo più preciso sull'età anziché sul numero può aggiungere alla strategia un'azione `Delete` con `IfLastModified` (età) oppure `IfAccumulatedFileSize` (dimensione totale); nella maggior parte dei casi è sufficiente la combinazione tra dimensione per file e `max`.

## Il livello di log come vero fattore di volume

Rotazione e conservazione limitano il consumo di spazio, ma non cambiano la quantità di dati scritti in partenza. La leva principale è il livello di log. Un gateway in produzione eseguito con DEBUG registra ogni fase di elaborazione di ciascun messaggio e, sotto carico, ciò si somma a gigabyte al giorno. Per il funzionamento normale, il livello deve essere INFO o superiore; DEBUG è uno strumento per analisi limitate nel tempo, non per l'uso continuativo. Se il livello è INFO e la rotazione per dimensione con `%i` è impostata correttamente, le due misure si integrano: INFO mantiene ridotto il volume giornaliero e la rotazione limita anche un picco anomalo di DEBUG.

## Dove totemomail memorizza questi valori

In totemomail queste impostazioni non si trovano in un file locale `log4j2.xml`, il che può facilmente portare fuori strada durante la ricerca degli errori. La configurazione viene generata a runtime da proprietà con il prefisso `totemo.log4j2.*`, gestite centralmente tramite la Management Console (sezione Logging + Tracking). Per questo una ricerca di `log4j2.xml` nel filesystem non porta a nulla; un file `log4j.xml` nella directory di configurazione appartiene a un componente incluso (openjms) e non ha nulla a che vedere con il log di trace.

Le proprietà rilevanti e il loro significato:

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Proprietà | Significato |
|---|---|
| `totemo.log4j2.appender.a1.filePattern` | Il pattern del file; qui deve essere inserito `%i` |
| `totemo.log4j2.appender.a1.policies.size.size` | Dimensione per file per la SizeBasedTriggeringPolicy, ad esempio `100MB` |
| `totemo.log4j2.appender.a1.strategy.max` | Numero di file di archivio conservati |
| `totemo.log4j2.rootLogger.level` | Livello del root logger di log4j2 |
| `totemo.log.priority` | Priorità di logging generale dell'applicazione, ovvero il vero interruttore DEBUG |
| `totemo.tracking` | Livello di dettaglio del tracking dei messaggi; `debug` genera le righe per ogni Mailet |

</details>

È importante la duplice natura: i logger di log4j2 possono essere impostati su `warn` o `error` e generare comunque una valanga di DEBUG nel log di trace, poiché `totemo.log.priority` e `totemo.tracking` agiscono come interruttori propri e di livello superiore. Chi vuole ridurre il volume imposta `totemo.log.priority` su INFO e `totemo.tracking` da `debug` a `on`; in questo modo vengono rimosse le righe dettagliate di elaborazione. Poiché i valori sono gestiti tramite la Console, si applicano all'intero cluster e alcuni richiedono il riavvio dell'istanza per avere effetto, come indicato nella rispettiva proprietà.

## Il riavvio dopo il riempimento del disco

Un dettaglio facile da trascurare: dopo che il disco si è riempito una volta, il logging non riprende automaticamente, nemmeno liberando spazio. L'appender del file rimane nel suo stato di errore finché il processo Java non viene riavviato. Lo si riconosce dal fatto che il gateway continua ad accettare ed elaborare email (il banner SMTP mostra l'ora corretta), mentre il log di trace rimane fermo al momento in cui il disco si è riempito. Un riavvio controllato dell'istanza ripristina il logging e attiva allo stesso tempo le impostazioni dell'appender modificate, come il nuovo `filePattern`.

## Diagnosi in pochi comandi

La partizione piena e la relativa causa possono essere identificate rapidamente. Per prima cosa si può verificare quale filesystem è interessato:

```bash
df -h
```

Se il volume dei log è al 100%, un elenco ordinato per dimensione individua il principale responsabile:

```bash
du -sh /pfad/zu/logs/* | sort -rh | head
```

Se compare un singolo file giornaliero di molti gigabyte invece di numerosi piccoli archivi indicizzati, si tratta della trappola di `%i`. Dopo la correzione e un riavvio, l'elenco dei file conferma che la rotazione funziona:

```bash
ls -laht /pfad/zu/logs/trace.log*
```

Ci si aspettano `trace.log` più archivi indicizzati `trace.log.<datum>.1`, `.2` e così via, ciascuno all'incirca della dimensione massima configurata.

## Riepilogo

Chi utilizza log4j2 con rotazione temporale e per dimensione deve necessariamente includere `%i` nel `filePattern`; altrimenti un singolo file cresce senza limiti e il limite di dimensione rimane inefficace. Tramite `strategy.max` (insieme all'indice), il numero di archivi limita il consumo di spazio, mentre il livello di log determina il volume alla fonte. In totemomail questi valori si trovano nella Management Console in `totemo.log4j2.*`, nonché negli interruttori di livello superiore `totemo.log.priority` e `totemo.tracking`; dopo che il disco si è riempito, è necessario riavviare l'istanza affinché il logging riprenda a scrivere.

## Fonti

1.  [Apache Logging Services: Log4j RollingFileAppender](https://logging.apache.org/log4j/2.x/manual/appenders/rolling-file.html): riferimento a filePattern, alle TriggeringPolicies e alla DefaultRolloverStrategy, inclusa l'indicazione relativa a `%i` nella rotazione basata sul tempo.

2.  [Apache Logging Services: Log4j Architecture](https://logging.apache.org/log4j/2.x/manual/architecture.html): inquadramento di Appender, Layout e gerarchia dei logger, utile per comprendere root logger e livello di log.
