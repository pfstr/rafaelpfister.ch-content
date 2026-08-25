---
title: "Pianificare test di carico e-mail: confronto tra strumenti per burst di 10'000 e-mail su Linux e Windows"
navTitle: "Test di carico e-mail"
description: "Chi migra un gateway o dimensiona un ambiente e-mail ha bisogno di dati affidabili anziché di sensazioni. Quali strumenti generano burst di diverse decine di migliaia di e-mail, come predisporre un piano di test pulito e come valutare i risultati dai log."
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
url: https://rafaelpfister.ch/it/blog/pianificare-test-di-carico-e-mail-confronto-tra-strumenti-per-burst-di-10-000-e-mail-su-linux-e
translationSourceHash: c9b76f3c9887117756e07c71a3dc30d1ee99aeb8a322c50dee994a07df46cb97
translationModel: gpt-5.6-terra
translatedAt: 2026-08-25T04:11:42.071Z
translationReview: automatic
---

# Pianificare test di carico e-mail: confronto tra strumenti per burst di 10'000 e-mail su Linux e Windows

Se un nuovo gateway e-mail regge il carico di picco di una notte di elaborazione delle fatture non si vede dalla scheda tecnica, ma dal test. Chi sostituisce un'appliance, dimensiona un ambiente Exchange o pianifica l'invio di una newsletter tramite la propria infrastruttura ha bisogno prima di numeri affidabili: quante e-mail al secondo accetta il sistema, come si comporta la coda sotto pressione e da quale punto iniziano i deferral? Questo articolo confronta i comuni generatori di carico su Linux e Windows e mostra come pianificare, eseguire e valutare un test con burst di diverse decine di migliaia di e-mail.

Prima di tutto la regola più importante: i test di carico devono essere eseguiti esclusivamente nella propria infrastruttura o in un ambiente di test esplicitamente autorizzato a questo scopo. Un burst contro sistemi altrui è un attacco e un test con indirizzi mittente inventati verso destinazioni produttive genera backscatter che porta alle blocklist. Una configurazione corretta è composta da un generatore di carico, dal sistema da testare e da un sink controllato, che alla fine accetta e scarta le e-mail.

## Cosa deve misurare un test di carico e-mail

Prima di parlare di strumenti, vale la pena chiedersi quale grandezza interessi effettivamente. Nella pratica sono quattro diverse e richiedono configurazioni di test differenti:

1. **Tasso di accettazione:** quante e-mail al secondo accetta il primo hop via SMTP? È la classica misura del throughput e il valore fornito direttamente dai generatori di carico.
2. **Latenza della sessione:** quanto dura una singola transazione SMTP dall'apertura della connessione fino al `250` dopo `DATA`? Sotto carico questo valore aumenta spesso molto prima che il tasso di accettazione crolli.
3. **Latenza end-to-end:** quanto tempo impiega un'e-mail dall'immissione alla consegna al sink, passando per tutte le stazioni intermedie? È la grandezza percepita dagli utenti.
4. **Comportamento della coda:** quanto cresce la coda durante il burst e con quale velocità si svuota in seguito? Un gateway che accetta 50'000 e-mail e poi le elabora per tre ore supera il test di accettazione, ma fallisce comunque.

Un test che misura soltanto il tasso di accettazione dice poco su un ambiente a più livelli con gateway, livello di crittografia e server di destinazione. Proprio in questi casi conta la visione end-to-end.

## Strumenti su Linux

**smtp-source e smtp-sink** del pacchetto Postfix sono lo standard per il carico SMTP grezzo e sono disponibili praticamente su ogni sistema su cui è installato Postfix. `smtp-source` genera e-mail con dimensione, parallelismo e numero configurabili, mentre `smtp-sink` è la controparte: un server SMTP che accetta tutto e scarta tutto. Un burst di 10'000 e-mail con 50 sessioni parallele e messaggi da 5 KB è un one-liner:

```bash
time smtp-source -s 50 -m 10000 -l 5120 -c -f last@test.example -t senke@test.example gateway.test.example:25
```

L'opzione `-c` conta in tempo reale le e-mail inviate, mentre `time` fornisce la durata complessiva e quindi il tasso. Limiti importanti: `smtp-source` non misura i percentili di latenza e le e-mail sono sinteticamente uniformi. Per rispondere alla domanda «quanto accetta al massimo il sistema» resta comunque la prima scelta, perché anche su hardware modesto genera decine di migliaia di e-mail al minuto e il generatore non diventa praticamente mai il collo di bottiglia.

**Postal** è il classico benchmark dedicato per server e-mail su Linux. Varia automaticamente mittente, destinatario e dimensione delle e-mail, mantiene un tasso obiettivo per lunghi periodi e scrive statistiche al minuto. È quindi più adatto di `smtp-source` ai test soak, ovvero al carico continuo per ore. Il relativo `bhm` (Black Hole Mailer) svolge il ruolo di sink. Postal è datato, ma è costruito esattamente per questo scopo ed è incluso nelle fonti di pacchetti della maggior parte delle distribuzioni.

**swaks** non è un generatore di carico, ma dovrebbe far parte di ogni piano di test. Esegue una singola transazione SMTP con controllo completo su ogni fase: autenticazione, STARTTLS, header arbitrari, allegati. Prima di ogni test di carico va eseguito un passaggio con swaks come test funzionale, affinché il burst non fallisca per un destinatario errato o un problema TLS rendendo inutile la misurazione. In un ciclo con `xargs -P` swaks può anche essere abusato come piccolo generatore di carico, ma per decine di migliaia di e-mail l'overhead dei processi è eccessivo.

**Script personalizzati** in Python (smtplib, aiosmtplib) o Go sono la strada giusta quando il test richiede mix di messaggi realistici: dimensioni diverse, allegati reali, numeri variabili di destinatari per transazione, casi di errore mirati. Lo sforzo è maggiore, ma lo script misura esattamente ciò che l'ambiente vedrà in seguito e può scrivere timestamp per ogni e-mail per l'analisi della latenza.

## Strumenti su Windows

**Apache JMeter** è la prima raccomandazione su Windows. Il SMTP Sampler integrato supporta autenticazione, STARTTLS, allegati e file EML come sorgente dei messaggi, mentre il meccanismo di JMeter fornisce ciò che manca agli strumenti Postfix: gruppi di thread per profili di carico graduali, percentili dei tempi di risposta, tassi di errore e report. Per burst oltre qualche migliaio di e-mail al minuto vale la consueta regola di JMeter: GUI soltanto per creare il piano di test; la misurazione vera e propria va eseguita in modalità CLI, altrimenti si misura l'interfaccia.

**PowerShell con MailKit** è la soluzione basata sugli strumenti disponibili. Il `Send-MailMessage` un tempo comune è indicato da Microsoft stessa come obsoleto e ne raccomanda la sostituzione; MailKit può essere caricato tramite NuGet e parallelizzato da PowerShell 7 con Runspaces. In modo realistico si ottengono da alcune centinaia a poche migliaia di e-mail al minuto, sufficienti per test funzionali e di regressione, ma troppo poche per misurare il carico massimo. Il vantaggio: lo script funziona senza installazioni aggiuntive su ogni postazione amministrativa e può scrivere direttamente i risultati in CSV per la valutazione.

**Microsoft Exchange Load Generator (LoadGen)** è stato per anni lo strumento ufficiale per caricare ambienti Exchange con profili utente simulati (Outlook, ActiveSync, OWA). Microsoft non lo ha più aggiornato dopo Exchange 2013 e ne ha ritirato il download. Per il puro carico SMTP LoadGen era comunque lo strumento sbagliato; chi oggi vuole simulare il carico delle cassette postali Exchange non dispone di uno strumento ufficiale e farebbe meglio a testare direttamente il percorso SMTP.

**WSL** merita un punto a sé: chi lavora su una macchina Windows ma necessita degli strumenti Linux installa `smtp-source` e Postal in una distribuzione WSL, ottenendo così l'intera cassetta degli attrezzi Linux senza una VM di test separata. Per i carichi qui discussi, il percorso di rete WSL non rappresenta un collo di bottiglia rilevante.

## Confronto

| Strumento | Piattaforma | Punto di forza | Limite |
|---|---|---|---|
| smtp-source / smtp-sink | Linux (Postfix) | Massimo carico grezzo con sforzo minimo, generatore e sink da un'unica fonte | Nessun percentile di latenza, messaggi uniformi |
| Postal / bhm | Linux | Carico continuo con tasso obiettivo, messaggi variabili, statistiche al minuto | Tooling datato, valutazione da costruire autonomamente |
| swaks | Linux, Windows (Perl) | Test singolo pienamente controllabile, ideale come verifica funzionale prima del burst | Non è un generatore di carico |
| JMeter (SMTP Sampler) | Windows, Linux (Java) | Profili di carico, percentili, report, sorgenti di messaggi EML | Overhead Java, trappola della GUI ad alti tassi |
| PowerShell + MailKit | Windows | Senza installazioni aggiuntive su ogni PC amministrativo, output CSV | Throughput limitato, parallelizzazione da costruire autonomamente |
| Script personalizzato (Python/Go) | entrambi | Mix di messaggi realistico, punti di misurazione personalizzati | Sforzo di sviluppo, generatore da validare autonomamente |

## Il sink: dove inviare le e-mail

La metà sottovalutata della configurazione di test è la destinazione. Tre varianti si sono dimostrate valide:

- **smtp-sink** o `bhm` come buco nero: accetta tutto, scarta tutto e misura la pura catena di trasporto. `smtp-sink` può generare su richiesta ritardi di risposta e codici di errore artificiali, verificando così anche il comportamento del sistema in test verso una destinazione lenta o ostinata.
- **Postfix con trasporto discard** come sink più realistico, se anche la destinazione deve essere un server SMTP completo con queueing.
- **Alcune vere cassette postali seed** oltre al sink, per verificare a campione che le e-mail arrivino integre nel contenuto, incluso il livello di crittografia o firma.

Strumenti con interfaccia web come Mailpit sono pensati per lo sviluppo e con decine di migliaia di e-mail diventano rapidamente essi stessi il collo di bottiglia. Non sono adatti come sink per un test di carico; la misurazione finirebbe per misurare lo strumento di analisi anziché il sistema in test.

## Pianificare il test

Un test affidabile si svolge in tre fasi, ciascuna con una domanda distinta:

1. **Baseline:** un carico moderato e noto, circa il 10 per cento del picco previsto, per alcuni minuti. Fornisce valori di riferimento per latenza e consumo di risorse e rileva errori di configurazione prima che scompaiano nella misurazione del burst.
2. **Burst:** la vera misurazione del carico di picco, ad esempio da 10'000 a 50'000 e-mail il più rapidamente possibile o con un tasso obiettivo definito. Più esecuzioni con parallelismo crescente mostrano dove il tasso di accettazione si appiattisce e la latenza peggiora.
3. **Soak:** il carico giornaliero previsto per più ore. Solo qui emergono memory leak, partizioni spool che si riempiono, rotazione dei log sotto carico e limiti di connessione che un breve burst non raggiunge mai.

Per il mix di messaggi vale: realistico quanto necessario. Una misurazione esclusivamente con e-mail di testo da 5 KB sovrastima di molte volte il throughput di un ambiente la cui quotidianità consiste in allegati PDF. È sensato usare un mix tratto dal proprio parco e-mail, ad esempio 70 per cento piccoli, 25 per cento con allegato tipico, 5 per cento grandi. Anche TLS deve essere incluso nel test se in produzione viene usato TLS: l'handshake costa per connessione molto più del trasferimento delle e-mail stesso, e generatori che aprono una nuova connessione per ogni e-mail misurano altrimenti soprattutto la terminazione TLS.

Per la successiva valutazione, ogni e-mail di test riceve un marcatore univoco, preferibilmente un header dedicato come `X-Loadtest-Id` con numero di esecuzione e timestamp, oltre a una convenzione riconoscibile per l'oggetto. In questo modo le esecuzioni di test possono essere separate pulitamente nei log sia tra loro sia dal restante traffico, e le e-mail di test possono essere rimosse in modo mirato da quarantene e journal dopo l'esecuzione.

Tre aspetti vengono regolarmente dimenticati durante la pianificazione. Primo: rate limit e tarpitting nel percorso di test; un gateway che limita dopo 100 e-mail al minuto per IP sorgente testerebbe altrimenti soltanto la propria limitazione (escluderlo appositamente per la misurazione del carico massimo, lasciarlo volutamente attivo per il controllo di realismo). Secondo: DNS. Se il sistema in test risolve domini destinatari o effettua interrogazioni DNSBL per ogni e-mail, nell'ambiente di test deve essere incluso un resolver, altrimenti il test misura il DNS upstream. Terzo: il generatore stesso. Prima della prima esecuzione verso il sistema di destinazione, il generatore va eseguito direttamente contro il sink per dimostrare che può effettivamente generare il tasso obiettivo.

## Valutazione

I valori misurati dal generatore di carico sono soltanto metà della verità, poiché mostrano la prospettiva di chi immette le e-mail. L'altra metà si trova nei log del sistema in test.

In Postfix, il maillog fornisce per ogni e-mail i campi `delay` e `delays`, il secondo suddiviso in tempo nella coda, apertura della connessione e trasferimento. Una valutazione di un'esecuzione di test è realizzabile con gli strumenti integrati:

```bash
grep "status=sent" /var/log/mail.log |
  grep -o "delay=[0-9.]*" | cut -d= -f2 | sort -n |
  awk '{a[NR]=$1} END {print "n="NR, "p50="a[int(NR*0.5)], "p95="a[int(NR*0.95)], "p99="a[int(NR*0.99)], "max="a[NR]}'
```

Sul lato Exchange, il Message Tracking Log è la fonte centrale. Per un'esecuzione di test con convenzione dell'oggetto:

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

La differenza tra i timestamp degli eventi RECEIVE e DELIVER dello stesso MessageId fornisce la latenza end-to-end per e-mail; esportando in CSV si può calcolare la distribuzione dei percentili.

Nell'interpretazione contano tre principi di base. Primo: percentili anziché medie. Una media di due secondi può significare che tutto dura due secondi, oppure che il 95 per cento viene completato in mezzo secondo e il resto resta nella coda; p50, p95 e p99 distinguono questi casi. Secondo: analizzare i codici di risposta SMTP. La distribuzione delle risposte 4xx nel tempo mostra quando il sistema inizia a limitare, mentre i codici stessi (limite di connessioni, protezione della coda, greylisting) indicano quale meccanismo interviene per primo. Terzo: tracciare la profondità della coda nel tempo, su Postfix tramite `qshape` oppure `postqueue -j`, su Exchange tramite `Get-Queue` a intervalli di un minuto. L'area sotto questa curva, non il tasso di accettazione, decide se l'ambiente assorbe un burst o lo immagazzina soltanto.

Parallelamente ai maillog, nella valutazione devono rientrare le metriche di sistema del sistema in test: CPU, tempi di attesa I/O sulla partizione spool, descrittori di file, contatori delle connessioni. Il riscontro più frequente negli ambienti a più livelli è che non è il processo e-mail a limitare, bensì uno stadio di content inspection (scanner antivirus, modulo di crittografia, DLP) con un numero fisso di worker. Sono proprio questi riscontri il vero valore del test: indicano la leva su cui agire prima che sia la produzione a scoprirla.

## Conclusione

Per la rapida misurazione del carico massimo su Linux non si può prescindere da `smtp-source` con `smtp-sink`; Postal integra il caso del carico continuo. Su Windows, JMeter offre la misurazione più completa, PowerShell con MailKit copre i test funzionali e di regressione e, se necessario, WSL porta gli strumenti Linux sulla postazione amministrativa. Più importante dello strumento è il piano: misurazione separata di accettazione, latenza e comportamento della coda, mix di messaggi realistico, esecuzione di test marcata e valutazione che includa percentili e log del sistema di destinazione invece di considerare soltanto il contatore del generatore.

## Fonti

1.  [smtp-source(1), manuale Postfix](https://www.postfix.org/smtp-source.1.html): riferimento del generatore di carico con tutte le opzioni per parallelismo, dimensione delle e-mail e TLS.

2.  [smtp-sink(1), manuale Postfix](https://www.postfix.org/smtp-sink.1.html): riferimento del sink e-mail, inclusi ritardi artificiali e risposte di errore.

3.  [Documentazione Postal, Russell Coker](https://doc.coker.com.au/projects/postal/): descrizione del benchmark per server e-mail con tasso obiettivo, variazione dei messaggi e sink bhm.

4.  [swaks, John Jetmore](https://www.jetmore.org/john/code/swaks/): il tester funzionale SMTP per il controllo preliminare di ogni percorso di test.

5.  [Apache JMeter Component Reference: SMTP Sampler](https://jmeter.apache.org/usermanual/component_reference.html): funzionalità del SMTP Sampler, incluse autenticazione, TLS e sorgenti EML.

6.  [Send-MailMessage, Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/send-mailmessage): avviso ufficiale di Microsoft che il cmdlet è obsoleto, con riferimento ad alternative come MailKit.

7.  [MailKit, Jeffrey Stedfast](https://github.com/jstedfast/MailKit): la libreria e-mail .NET per script di invio personalizzati in PowerShell 7.

8.  [Get-MessageTrackingLog, Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog): riferimento per la valutazione del Message Tracking Log di Exchange dopo un'esecuzione di test.

9.  [qshape(1), manuale Postfix](https://www.postfix.org/qshape.1.html): strumento per analizzare la distribuzione della coda durante e dopo il burst.
