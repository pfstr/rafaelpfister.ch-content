---
title: "Test funzionali e test di carico: verificare l'infrastruttura in condizioni controllate"
blatt: "testing"
description: "Metodologia per test funzionali e di carico in ambienti di messaggistica e infrastruttura: metriche, svolgimento dei test con baseline, burst e soak, generatori di carico e sink, marcatura delle esecuzioni di test e valutazione tramite percentili anziché medie."
fakten:
  - label: Procedura di base
    wert: Test funzionale → Baseline → Burst → Soak
    href: https://jmeter.apache.org/usermanual/best-practices.html
  - label: Quattro metriche
    wert: Tasso di accettazione · Latenza di sessione · Latenza end-to-end · Comportamento della coda
  - label: Valutazione
    wert: Percentili p50 · p95 · p99, non medie
    href: https://sre.google/sre-book/monitoring-distributed-systems/
  - label: Generatori di carico
    wert: smtp-source · Postal · JMeter · script personalizzati
    href: https://www.postfix.org/smtp-source.1.html
  - label: Sink
    wert: smtp-sink · bhm · aiosmtpd · trasporto discard
    href: https://www.postfix.org/smtp-sink.1.html
  - label: Test di controllo
    wert: conteggio indipendente sul lato di ricezione
  - label: Marcatura
    wert: header univoco o marcatore nell'oggetto per ogni esecuzione di test
  - label: Autotest del generatore
    wert: l'esecuzione contro il sink dimostra il tasso obiettivo raggiungibile
  - label: Regola di base
    wert: solo sistemi propri o espressamente autorizzati
werbung:
  - newsletter
ctaThemen:
  - smtp-mailflow
translationSourceHash: f6d1ef180221f41242e1d32b5fb8fb2e631894e3ccd32a3b3322a33b808da210
translationModel: gpt-5.6-terra
translatedAt: 2026-10-14T09:39:46.619Z
translationReview: automatic
---

# Test funzionali e test di carico: verificare l'infrastruttura in condizioni controllate

Se un ambiente regge i picchi di carico, se un insieme di regole si comporta allo stesso modo dopo una modifica o se un mount di storage resiste a modelli di accesso realistici non lo si scopre dalla scheda tecnica, ma con i test. Questa pagina riunisce la metodologia comune a tutti i tipi di test: cosa viene misurato, come è strutturato un test, quali strumenti generano e assorbono il carico e come trasformare dati grezzi in risultati affidabili.

La regola più importante precede ogni metodologia: i test di carico e di perturbazione devono essere eseguiti esclusivamente su sistemi propri o in ambienti espressamente autorizzati a tale scopo. Un burst contro un'infrastruttura altrui è un attacco; un test e-mail con mittenti inventati verso destinazioni produttive genera backscatter e inserimenti in blocklist.

## Separare test funzionali e test di carico

Un test risponde sempre a una sola domanda e le due domande fondamentali richiedono configurazioni diverse. Il **test funzionale** verifica che un percorso funzioni correttamente: una singola transazione SMTP con pieno controllo su ogni passaggio, un singolo accesso a file, un singolo percorso di regole attraverso un gateway. Il **test di carico** verifica come si comporta il sistema sotto volume: throughput, distribuzione della latenza, code, consumo di risorse.

L'ordine è obbligatorio: prima il test funzionale, poi il carico. Un burst su un percorso difettoso non misura nulla; produce soltanto diecimila errori dello stesso tipo. Un contatore di errori al 100 per cento con tempi di risposta prossimi allo zero significa quasi sempre che l'errore avviene prima della rete e che il test non ha mai raggiunto il sistema di destinazione.

## Le quattro metriche

I test di carico in ambienti multistadio misurano quattro aspetti diversi e ciascuno richiede una prospettiva propria:

1. **Tasso di accettazione:** Quante operazioni al secondo accetta il primo hop? È il valore fornito direttamente dai generatori di carico.
2. **Latenza di sessione:** Quanto dura una singola transazione? Sotto carico questo valore spesso aumenta molto prima che il tasso di accettazione diminuisca.
3. **Latenza end-to-end:** Quanto tempo richiede un'operazione attraverso tutti gli stadi intermedi fino alla destinazione? È il valore percepito dagli utenti.
4. **Comportamento della coda:** Quanto cresce la coda durante il picco e quanto rapidamente si svuota in seguito? Un sistema che accetta tutto e lo elabora per ore supera il test di accettazione, ma fallisce comunque.

## Svolgimento del test: baseline, burst, soak

Un test affidabile si svolge in tre fasi, ciascuna con una propria domanda. La **baseline** definisce, con un carico moderato e noto, i valori di riferimento per latenza e consumo di risorse e individua gli errori di configurazione prima che scompaiano nella misurazione del picco. Il **burst** è la vera misurazione del carico di picco; più esecuzioni con parallelismo crescente mostrano dove il tasso di accettazione si appiattisce e la latenza aumenta sensibilmente. Il **soak** applica il carico continuo previsto per ore; solo qui emergono perdite di memoria, partizioni spool che si riempiono e limiti di connessione che un breve burst non raggiunge mai.

Vi sono inoltre tre regole pratiche. Primo, l'**autotest del generatore**: prima della prima esecuzione contro il sistema di destinazione, il generatore viene eseguito direttamente contro il sink; questo dimostra che la configurazione può generare il tasso obiettivo e fornisce il limite superiore con cui confrontare le misurazioni successive. Secondo, la **marcatura**: ogni operazione di test porta un marcatore univoco (header dedicato, convenzione nell'oggetto, numero di esecuzione), così da separare chiaramente le esecuzioni nei log e rimuovere completamente i dati di test in seguito. Terzo, il **test di controllo**: un conteggio indipendente sul lato di ricezione conferma che è arrivato ciò che il generatore ha inviato; se i numeri non coincidono, qualcosa è andato perso lungo il percorso.

## Strumenti: generatore e sink

Un test di carico è sempre composto da tre parti: generatore di carico, sistema in prova e sink controllato. Per il carico SMTP, i generatori più comuni sono [smtp-source](https://www.postfix.org/smtp-source.1.html) del pacchetto Postfix (massimo carico grezzo con sforzo minimo), [Postal](https://doc.coker.com.au/projects/postal/) (carico continuo con tasso obiettivo e messaggi variabili), [Apache JMeter](https://jmeter.apache.org/) (profili di carico, percentili e report HTML, prima scelta su Windows) e script personalizzati quando il test richiede mix di operazioni realistici. [swaks](https://www.jetmore.org/john/code/swaks/) non è un generatore di carico, ma come tester funzionale deve precedere ogni burst.

Sul lato di ricezione, un sink accetta tutto e lo scarta: [smtp-sink](https://www.postfix.org/smtp-sink.1.html), `bhm` incluso in Postal oppure un piccolo server [aiosmtpd](https://aiosmtpd.aio-libs.org/) in poche righe di Python, che si occupa anche del conteggio di controllo. Gli strumenti per sviluppatori con interfaccia web non sono adatti come sink per test di carico; con decine di migliaia di operazioni si finirebbe altrimenti per misurare lo strumento di analisi anziché il sistema in prova.

## Valutazione: percentili, pivot, andamento

I dati grezzi diventano risultati solo attraverso tre passaggi di valutazione. **Percentili anziché medie:** una media di due secondi può significare che tutto dura due secondi, oppure che il 95 per cento si conclude in mezzo secondo e il resto richiede molto più tempo; p50, p95 e p99 distinguono questi casi. **Eseguire il pivot degli errori:** la distribuzione dei codici di errore nel tempo mostra quando un sistema inizia a limitare e quale meccanismo di protezione interviene per primo. **Andamento anziché stato finale:** throughput e profondità della coda rappresentati nel tempo mostrano ramp-up, plateau e fase di esaurimento; per l'interpretazione conta il plateau, non la media sull'intera esecuzione, poiché quest'ultima include avvio e conclusione e sottostima il tasso continuo raggiunto.

La valutazione comprende infine una delimitazione onesta: ogni rapporto di test specifica cosa dimostra l'esecuzione e cosa no. Una misurazione senza TLS non dice nulla sui costi dell'handshake TLS; un sink locale nulla sul comportamento della coda della vera destinazione; un burst nulla sulle perdite di memoria nel funzionamento continuo. Questi limiti indicano le successive esecuzioni di test.

Gli articoli seguenti applicano la metodologia a casi concreti: confronto degli strumenti e pianificazione dei test per test di carico e-mail, un'esecuzione JMeter completa con report HTML, diagnosi SMTP a strati con strumenti integrati, un piano di test per mount di cloud storage e il metodo scientifico alla base di esperimenti controllati nel flusso di posta.
