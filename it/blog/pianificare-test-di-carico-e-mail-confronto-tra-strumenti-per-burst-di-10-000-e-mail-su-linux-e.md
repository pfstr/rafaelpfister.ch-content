---
title: "Pianificare test di carico della posta: confronto tra strumenti per burst di 10'000 messaggi sotto Linux e Windows"
navTitle: "Test di carico della posta"
description: "Chi migra un gateway o dimensiona un ambiente di posta ha bisogno di dati affidabili anziché affidarsi all’intuito. Quali strumenti generano burst di diverse decine di migliaia di messaggi, come strutturare un piano di test rigoroso e come valutare i risultati dai log."
date: "2026-08-24"
kategorie: "SMTP e flusso di posta"
timeToRead: "12 min di lettura"
themen:
  - smtp-mailflow
  - testing
produkte:
  - "uebergreifend"
protokolle:
  - "testing"
  - "smtp"
  - "tcp"
  - "tls"
  - "troubleshooting"
slug: "pianificare-test-di-carico-e-mail-confronto-tra-strumenti-per-burst-di-10-000-e-mail-su-linux-e"
translationId: "article-14a98de0cef45565"
aiPrompt: |
  Du bist mein Assistent für Mail-Lasttests. Hilf mir Schritt für Schritt, einen Lasttest gegen mein eigenes Test-Mailgateway zu planen: Zieldefinition (Durchsatz, Latenz, Queue-Verhalten), Wahl des Lastgenerators (smtp-source, Postal, JMeter oder Skript), Aufbau einer Mail-Senke, Testablauf (Baseline, Burst, Soak) und Auswertung der Logs mit Perzentilen. Frage zuerst nach Plattform, Zielsystem und erwartetem Mailvolumen.
translationOf: mail-lasttest-tools-linux-windows-vergleich
translationSourceHash: 2fd0b1bd0748b9fb44be85907a946bbf85604b5eb7c85107170fa7443068efd7
translationModel: gpt-5.6-terra
translatedAt: 2026-08-30T09:26:31.281Z
translationReview: automatic
url: https://rafaelpfister.ch/it/blog/pianificare-test-di-carico-e-mail-confronto-tra-strumenti-per-burst-di-10-000-e-mail-su-linux-e
---

# Pianificare test di carico della posta: confronto tra strumenti per burst di 10'000 messaggi sotto Linux e Windows

È possibile verificare se un nuovo gateway di posta sopporta il carico di picco di un’elaborazione notturna delle fatture solo con un test di carico. Chi sostituisce un’appliance, dimensiona un ambiente Exchange o pianifica l’invio di una newsletter tramite la propria infrastruttura necessita prima di dati affidabili: quanti messaggi al secondo accetta il sistema, come si comporta la coda sotto pressione e da quale punto iniziano i deferral? Questo articolo confronta i comuni generatori di carico sotto Linux e Windows e mostra come pianificare, eseguire e valutare un test con burst di diverse decine di migliaia di messaggi.

Innanzitutto, la regola più importante: i test di carico devono essere eseguiti esclusivamente nella propria infrastruttura o in un ambiente di test espressamente autorizzato. Un burst contro sistemi altrui è un attacco, e un test con indirizzi mittente inventati verso destinazioni di produzione genera backscatter che porta alle blocklist. Una configurazione corretta consiste in un generatore di carico, il sistema da testare e un sink controllato che alla fine accetta e scarta i messaggi.

## Cosa deve misurare un test di carico della posta

Prima di parlare di strumenti, conviene chiedersi quale grandezza interessi effettivamente. Nella pratica sono quattro, e richiedono configurazioni di test differenti:

1. **Tasso di accettazione:** Quanti messaggi al secondo accetta il primo hop via SMTP? È la classica misurazione del throughput e il valore fornito direttamente dai generatori di carico.
2. **Latenza della sessione:** Quanto dura una singola transazione SMTP dall’apertura della connessione fino a `250` dopo `DATA`? Sotto carico, questo valore aumenta spesso molto prima che il tasso di accettazione diminuisca.
3. **Latenza end-to-end:** Quanto tempo impiega un messaggio dalla consegna iniziale fino al recapito al sink, attraverso tutte le stazioni intermedie? È la grandezza percepita dagli utenti.
4. **Comportamento della coda:** Quanto cresce la coda durante il burst e quanto rapidamente si svuota in seguito? Un gateway che accetta 50'000 messaggi e poi li elabora in tre ore supera il test di accettazione, ma fallisce comunque.

Un test che misura solo il tasso di accettazione dice poco su un ambiente a più livelli con gateway, livello di crittografia e server di destinazione. Proprio in questi casi è determinante la visione end-to-end.

## Il profilo di carico determina lo strumento

Oltre alla grandezza da misurare, una seconda domanda determina la scelta dello strumento e viene spesso saltata: quale comportamento di connessione ha il carico da simulare? Occorre distinguere due profili di carico.

Un **mittente bulk con sessioni aperte** è il profilo di carico di elaborazioni di fatture, conteggi salariali e sistemi di newsletter: un singolo sistema apre poche connessioni e invia su di esse centinaia o migliaia di messaggi consecutivi. L’overhead di connessione si presenta una volta per sessione, non una volta per messaggio, e il gateway vede poche connessioni con molte transazioni.

**Molti mittenti indipendenti** rappresentano il profilo di carico di paesaggi applicativi e traffico degli utenti: numerosi sistemi consegnano singoli messaggi, ciascuno tramite la propria connessione. Qui l’apertura della connessione, inclusi TLS e AUTH, fa parte di ogni messaggio.

Per il dimensionamento di un invio di massa conta il primo profilo di carico e, a tale scopo, il generatore di carico deve poter mantenere aperte le sessioni: `smtp-source` lo fa (molti messaggi distribuiti su poche sessioni), così come Postal e gli script propri con connessione persistente. JMeter non può farlo; i motivi sono illustrati nella sezione Windows. Per il carico di picco di un’elaborazione delle fatture, questo criterio di sessione è quindi determinante, non la piattaforma; sotto Windows la soluzione passa dunque da WSL.

## Strumenti sotto Linux

**smtp-source e smtp-sink** del pacchetto Postfix sono lo standard per il carico SMTP grezzo e sono disponibili praticamente su ogni sistema su cui sia installato Postfix. `smtp-source` genera messaggi con dimensione, parallelismo e quantità configurabili, mentre `smtp-sink` è la controparte: un server SMTP che accetta e scarta tutto. Un burst di 10'000 messaggi con 50 sessioni parallele e messaggi da 5 KB richiede una sola riga:

```bash
time smtp-source -s 50 -m 10000 -l 5120 -c -f last@test.example -t senke@test.example gateway.test.example:25
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `time` | Misura la durata complessiva dell’esecuzione; da essa si ricava il tasso in messaggi al secondo |
| `-s 50` | 50 sessioni SMTP parallele |
| `-m 10000` | Numero totale di messaggi, distribuiti tra le sessioni |
| `-l 5120` | Dimensione del corpo del messaggio in byte (senza intestazioni), qui 5 KB |
| `-c` | Contatore progressivo dei messaggi inviati come indicatore di avanzamento |
| `-f last@test.example` | Indirizzo mittente |
| `-t senke@test.example` | Indirizzo destinatario |
| `gateway.test.example:25` | Host e porta di destinazione per la consegna iniziale |

</details>

Limiti importanti: `smtp-source` non misura percentili di latenza e i messaggi sono sinteticamente uniformi. Per la domanda «quanto accetta al massimo il sistema» rimane comunque la prima scelta, perché anche su hardware poco potente genera decine di migliaia di messaggi al minuto e il generatore non diventa praticamente mai il collo di bottiglia.

**Postal** è il classico benchmark dedicato per mail server sotto Linux. Varia automaticamente mittente, destinatario e dimensione del messaggio, mantiene un tasso di destinazione per lunghi periodi e scrive statistiche al minuto. È quindi più adatto di `smtp-source` per i test soak, ossia carico continuo per ore. Il relativo `bhm` (Black Hole Mailer) svolge il ruolo di sink. Postal è datato, ma è costruito proprio per questo scopo ed è incluso nelle fonti dei pacchetti della maggior parte delle distribuzioni.

**swaks** non è un generatore di carico, ma deve far parte di ogni piano di test. Esegue una singola transazione SMTP con il pieno controllo di ogni passaggio: autenticazione, STARTTLS, intestazioni arbitrarie, allegati. Prima di ogni test di carico va eseguito un test funzionale con swaks, affinché il burst non fallisca per un destinatario errato o un problema TLS, rendendo la misurazione inutile. In un ciclo con `xargs -P` è possibile abusare di swaks anche come piccolo generatore di carico, ma per decine di migliaia di messaggi l’overhead dei processi è eccessivo.

**Script propri** in Python (smtplib, aiosmtplib) o Go sono la soluzione quando il test necessita di mix di messaggi realistici: dimensioni differenti, allegati reali, numeri variabili di destinatari per transazione, casi di errore mirati. Lo sforzo è maggiore, ma lo script misura esattamente ciò che il proprio ambiente vedrà in seguito e può registrare timestamp per messaggio per l’analisi della latenza.

## Strumenti sotto Windows

**Apache JMeter** è lo strumento adatto sotto Windows quando il profilo di carico è costituito da molti mittenti indipendenti oppure quando sono prioritari percentili, mix di messaggi e report. L’SMTP Sampler integrato supporta Auth, STARTTLS, allegati e file EML come sorgente dei messaggi, e il meccanismo JMeter offre ciò che manca agli strumenti Postfix: gruppi di thread per profili di carico graduali, percentili dei tempi di risposta, tassi di errore e report. Per burst oltre qualche migliaio di messaggi al minuto vale la consueta regola di JMeter: usare la GUI solo per creare il piano di test, mentre la misurazione vera e propria va eseguita in modalità CLI, altrimenti si misura l’interfaccia.

Occorre però conoscere un limite dell’SMTP Sampler: JMeter non può mantenere aperte le sessioni SMTP. Ogni esecuzione di un sample apre una nuova connessione, percorre il dialogo completo composto da handshake TCP, EHLO, eventualmente STARTTLS e AUTH, invia esattamente un messaggio e chiude la connessione con QUIT. Non è possibile riprodurre più messaggi sulla stessa connessione aperta, come fanno i mittenti di massa con il riutilizzo della sessione; `smtp-source` distribuisce invece molti messaggi su poche sessioni aperte. La ragione risiede nell’architettura: JMeter è un framework di test di carico multi-protocollo, non uno strumento SMTP. Il suo modello di esecuzione tratta ogni sampler come un’unità autonoma e misurata indipendentemente, poiché solo così timer, assertion e valutazione dei percentili funzionano in modo uniforme per tutti i protocolli supportati. L’SMTP Sampler è quindi un sottile livello sopra la libreria JavaMail, che come API client apre e richiude una connessione per ogni operazione di invio; il riutilizzo della connessione tra sample, come quello offerto dall’HTTP Sampler con Keep-Alive, non è mai stato implementato per SMTP. Per la misurazione questo significa che JMeter genera il profilo di carico di molti singoli mittenti, non quello di un mittente bulk con sessione aperta. Il throughput misurato include per ogni messaggio l’intero overhead di connessione e TLS, e i limiti di connessione del gateway intervengono quindi prima rispetto al riutilizzo della sessione. Per il profilo di carico bulk di un’elaborazione delle fatture, JMeter non è quindi lo strumento corretto; sotto Windows, la soluzione WSL con `smtp-source` è la scelta migliore.

**PowerShell con MailKit** è la via degli strumenti integrati. Il precedente `Send-MailMessage` è contrassegnato dalla stessa Microsoft come obsoleto, che raccomanda la migrazione; MailKit può essere caricato tramite NuGet e parallelizzato da PowerShell 7 tramite Runspaces. In modo realistico si raggiungono da alcune centinaia a poche migliaia di messaggi al minuto, sufficienti per test funzionali e di regressione, ma troppo pochi per misurare il carico massimo. Il vantaggio: lo script funziona senza installazioni aggiuntive su ogni postazione di lavoro amministrativa e può scrivere direttamente i risultati come CSV per la valutazione.

**Microsoft Exchange Load Generator (LoadGen)** è stato per anni lo strumento ufficiale per sottoporre gli ambienti Exchange a carico con profili utente simulati (Outlook, ActiveSync, OWA). Microsoft non lo ha più aggiornato dopo Exchange 2013 e ne ha ritirato il download. Per il puro carico SMTP, LoadGen era comunque lo strumento sbagliato; chi oggi vuole simulare il carico delle cassette postali Exchange non dispone di uno strumento ufficiale e farebbe meglio a testare direttamente il percorso SMTP.

**WSL** merita un punto a sé: chi lavora su una macchina Windows ma necessita di strumenti Linux può installare `smtp-source` e Postal in una distribuzione WSL, ottenendo così tutti gli strumenti Linux senza una VM di test separata. Per i carichi qui discussi, il percorso di rete WSL non costituisce un collo di bottiglia rilevante.

## Confronto

| Strumento | Piattaforma | Punto di forza | Limite |
|---|---|---|---|
| smtp-source / smtp-sink | Linux (Postfix) | Massimo carico grezzo con sforzo minimo, generatore e sink dalla stessa fonte | Nessun percentile di latenza, messaggi uniformi |
| Postal / bhm | Linux | Carico continuo con tasso di destinazione, messaggi variabili, statistiche al minuto | Tooling datato, valutazione da realizzare autonomamente |
| swaks | Linux, Windows (Perl) | Test singolo completamente controllabile, ideale come controllo funzionale prima del burst | Non è un generatore di carico |
| JMeter (SMTP Sampler) | Windows, Linux (Java) | Profili di carico, percentili, report, sorgenti di messaggi EML | Overhead Java, modalità GUI inadatta a tassi elevati, una connessione per messaggio (nessun riutilizzo della sessione) |
| PowerShell + MailKit | Windows | Senza installazioni aggiuntive su ogni computer amministrativo, output CSV | Throughput limitato, parallelizzazione da realizzare autonomamente |
| Script proprio (Python/Go) | entrambi | Mix di messaggi realistico, punti di misurazione propri | Sforzo di sviluppo, generatore da validare autonomamente |

## Il sink: dove finiscono i messaggi

La metà sottovalutata della configurazione di test è la destinazione. Si sono affermate tre varianti:

- **smtp-sink** oppure `bhm` come buco nero: accetta tutto, scarta tutto, misura la pura catena di trasporto. `smtp-sink` può, su richiesta, generare ritardi di risposta e codici di errore artificiali, verificando così anche il comportamento del sistema in prova verso una destinazione lenta o che risponde in modo errato.
- **Postfix con trasporto discard** come sink più realistico, se anche la destinazione deve essere un server SMTP completo con queueing.
- **Alcune vere cassette postali seed** in aggiunta al sink, per verificare a campione che i messaggi arrivino integri nel contenuto, incluso il livello di crittografia o firma.

Gli strumenti con interfaccia web come Mailpit sono destinati allo sviluppo e con decine di migliaia di messaggi diventano rapidamente essi stessi il collo di bottiglia. Non sono adatti come sink per un test di carico; la misurazione misurerebbe lo strumento di analisi anziché il sistema in prova.

## Pianificare il test

Un test affidabile si svolge in tre fasi, ciascuna con una propria domanda:

1. **Baseline:** Un carico moderato e noto (circa il 10 percento del picco atteso) per alcuni minuti. Fornisce i valori di riferimento per latenza e consumo di risorse e individua errori di configurazione prima che scompaiano nella misurazione del burst.
2. **Burst:** La vera misurazione del carico di picco, per esempio da 10'000 a 50'000 messaggi il più rapidamente possibile oppure con un tasso di destinazione definito. Più esecuzioni con parallelismo crescente mostrano dove il tasso di accettazione si appiattisce e la latenza peggiora improvvisamente.
3. **Soak:** Il carico giornaliero atteso per più ore. Solo qui emergono memory leak, partizioni spool che si riempiono, rotazione dei log sotto carico e limiti di connessione che un breve burst non raggiunge mai.

Per il mix di messaggi vale: realistico quanto necessario. Una misurazione con soli messaggi di testo da 5 KB sovrastima di molte volte il throughput di un ambiente la cui quotidianità è fatta di allegati PDF. È sensato un mix tratto dal proprio patrimonio, ad esempio 70 percento piccoli, 25 percento con allegato tipico, 5 percento grandi. TLS deve essere incluso nel test se la produzione utilizza TLS: l’handshake costa per connessione sensibilmente più del trasferimento del messaggio stesso, e i generatori che aprono una nuova connessione per messaggio misurano altrimenti soprattutto la terminazione TLS.

Per la successiva valutazione, ogni messaggio di test riceve un marcatore univoco, nel modo più semplice un’intestazione propria come `X-Loadtest-Id` con numero di esecuzione e timestamp, nonché una convenzione di oggetto riconoscibile. In questo modo le esecuzioni di test possono essere separate chiaramente nei log sia tra loro sia dal traffico rimanente, e i messaggi di test possono essere rimossi in modo mirato da quarantene e journal dopo l’esecuzione.

Tre punti che vengono regolarmente dimenticati nella pianificazione: primo, rate limit e tarpitting sul percorso di test; un gateway che rallenta dopo 100 messaggi al minuto per IP sorgente altrimenti testa solo la propria limitazione (escluderli miratamente per la misurazione del carico massimo, mantenerli volutamente per il controllo di realismo). Secondo, DNS: se il sistema in prova risolve i domini dei destinatari o esegue query DNSBL per ogni messaggio, nell’ambiente di test deve essere incluso un resolver, altrimenti il test misura il DNS upstream. Terzo, il generatore stesso: prima della prima esecuzione contro il sistema di destinazione, va eseguito il generatore direttamente contro il sink per dimostrare che il generatore è effettivamente in grado di produrre il tasso di destinazione.

## Valutare

I valori misurati dal generatore di carico sono solo metà della verità, poiché mostrano la prospettiva di chi consegna il messaggio. L’altra metà si trova nei log del sistema in prova.

In Postfix, il maillog fornisce per ogni messaggio i campi `delay` e `delays`, quest’ultimo suddiviso in tempo nella coda, apertura della connessione e trasferimento. Una valutazione su un’esecuzione di test può essere effettuata con gli strumenti integrati:

```bash
grep "status=sent" /var/log/mail.log |
  grep -o "delay=[0-9.]*" | cut -d= -f2 | sort -n |
  awk '{a[NR]=$1} END {print "n="NR, "p50="a[int(NR*0.5)], "p95="a[int(NR*0.95)], "p99="a[int(NR*0.99)], "max="a[NR]}'
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `grep "status=sent" /var/log/mail.log` | Filtra il maillog per messaggi consegnati con successo |
| `grep -o "delay=[0-9.]*"` | `-o` restituisce solo la corrispondenza stessa, qui il campo `delay` con il relativo valore |
| `cut -d= -f2` | Separa a `=` (`-d`) e mantiene il secondo campo (`-f2`), ossia il valore numerico |
| `sort -n` | Ordina numericamente anziché alfabeticamente; prerequisito per il calcolo dei percentili |
| `awk '…'` | Raccoglie i valori ordinati in un array e restituisce numero, p50, p95, p99 e massimo |

</details>

Sul lato Exchange, il Message Tracking Log è la fonte centrale. Per un’esecuzione di test con convenzione nell’oggetto:

```powershell
$p = @{
    Start          = "24.08.2026 14:00"
    End            = "24.08.2026 15:00"
    MessageSubject = "LOADTEST"
    ResultSize     = "Unlimited"
}
Get-MessageTrackingLog @p | Group-Object EventId |
    Sort-Object Count -Descending | Format-Table Name, Count
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Start` / `End` | Intervallo temporale della ricerca nei log; qui passato tramite splatting (`@p`) |
| `MessageSubject "LOADTEST"` | Filtra i messaggi il cui oggetto contiene il marcatore |
| `ResultSize Unlimited` | Rimuove il limite predefinito di 1000 voci restituite |
| `Group-Object EventId` | Raggruppa gli eventi di tracking per tipo (RECEIVE, DELIVER, DEFER, …) |
| `Sort-Object Count -Descending` | Ordina i gruppi di eventi in ordine decrescente di frequenza |
| `Format-Table Name, Count` | Visualizza il numero per tipo di evento |

</details>

La differenza fra i timestamp degli eventi RECEIVE e DELIVER della stessa MessageId fornisce la latenza end-to-end per messaggio; esportata come CSV, consente di calcolare la distribuzione dei percentili.

Nell’interpretazione contano tre principi fondamentali. Primo: percentili anziché medie. Una media di due secondi può significare che tutto dura due secondi, oppure che il 95 percento viene elaborato in mezzo secondo e il resto rimane nella coda; p50, p95 e p99 distinguono questi casi. Secondo: analizzare i codici di risposta SMTP in forma pivot. La distribuzione nel tempo delle risposte 4xx mostra quando il sistema inizia a rallentare, e quali codici siano (limite di connessione, protezione della coda, greylisting) indica quale meccanismo interviene per primo. Terzo: tracciare nel tempo la profondità della coda, sotto Postfix con `qshape` rispettivamente `postqueue -j`, su Exchange con `Get-Queue` a intervalli di un minuto. L’area sotto questa curva, non il tasso di accettazione, decide se l’ambiente assorbe un burst oppure lo accantona soltanto.

Parallelamente ai maillog, nella valutazione devono rientrare le metriche di sistema del sistema in prova: CPU, tempi di attesa I/O sulla partizione spool, descrittori di file, contatori di connessioni. Il riscontro più frequente in ambienti a più livelli è che non è il processo di posta a limitare, bensì un livello di content inspection (antivirus, modulo di crittografia, DLP) con un numero fisso di worker. Riscontri di questo tipo sono il vero valore del test: indicano la leva di regolazione prima che sia la produzione a trovarla.

## Conclusione

Per la rapida misurazione del carico massimo sotto Linux non si può prescindere da `smtp-source` con `smtp-sink`; Postal integra il caso di carico continuo. Sotto Windows, JMeter fornisce la misurazione più completa, PowerShell con MailKit copre i test funzionali e di regressione e WSL porta, se necessario, gli strumenti Linux sulla postazione amministrativa. Più importante dello strumento è il piano: misurazione separata di accettazione, latenza e comportamento della coda, un mix di messaggi realistico, un’esecuzione di test marcata e una valutazione che includa percentili e log del sistema di destinazione anziché solo il contatore del generatore.

## Fonti

1.  [smtp-source(1), manuale Postfix](https://www.postfix.org/smtp-source.1.html): Riferimento del generatore di carico con tutte le opzioni per parallelismo, dimensione dei messaggi e TLS.

2.  [smtp-sink(1), manuale Postfix](https://www.postfix.org/smtp-sink.1.html): Riferimento del sink di posta, inclusi ritardi artificiali e risposte di errore.

3.  [Documentazione Postal, Russell Coker](https://doc.coker.com.au/projects/postal/): Descrizione del benchmark per mail server con tasso di destinazione, variazione dei messaggi e sink bhm.

4.  [swaks, John Jetmore](https://www.jetmore.org/john/code/swaks/): Il tester funzionale SMTP per il controllo preliminare di ogni percorso di test.

5.  [Apache JMeter Component Reference: SMTP Sampler](https://jmeter.apache.org/usermanual/component_reference.html): Funzionalità dell’SMTP Sampler, incluse Auth, TLS e sorgenti EML.

6.  [Send-MailMessage, Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/send-mailmessage): Avviso ufficiale di Microsoft che il cmdlet è obsoleto, con riferimento ad alternative come MailKit.

7.  [MailKit, Jeffrey Stedfast](https://github.com/jstedfast/MailKit): La libreria .NET per la posta destinata a script di invio propri sotto PowerShell 7.

8.  [Get-MessageTrackingLog, Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog): Riferimento per la valutazione del Message Tracking Log di Exchange dopo un’esecuzione di test.

9.  [qshape(1), manuale Postfix](https://www.postfix.org/qshape.1.html): Strumento per analizzare la distribuzione della coda durante e dopo il burst.
