---
title: "Test di carico SMTP con Apache JMeter nella pratica: 10'000 email, cinque percorsi di regole, un report HTML"
navTitle: "Test di carico con JMeter"
description: "Un test di carico eseguito dalla A alla Z: piano di test con un mix di messaggi lungo i percorsi del ruleset di un gateway di crittografia, configurazione portabile senza installazione, 10'000 email in burst e valutazione tramite il report HTML di JMeter, inclusi gli ostacoli effettivamente riscontrati."
date: "2026-08-24"
kategorie: "SMTP e flusso di posta"
timeToRead: "11 min di lettura"
themen:
  - smtp-mailflow
  - testing
  - totemomail
produkte:
  - "uebergreifend"
  - "totemomail"
  - "apache-james"
protokolle:
  - "testing"
  - "smtp"
  - "troubleshooting"
related:
  - mail-lasttest-tools-linux-windows-vergleich
image: "../images/jmeter-report-dashboard.png"
slug: "test-di-carico-smtp-con-apache-jmeter-nella-pratica-10-000-email-cinque-percorsi-di-regole-un"
translationId: "article-fc3f25272e051f92"
aiPrompt: |
  Du bist mein Assistent für Mail-Lasttests mit Apache JMeter. Hilf mir Schritt für Schritt, einen SMTP-Lasttest aufzubauen: portables Setup (JRE + JMeter ohne Installation), lokale SMTP-Senke mit aiosmtpd, Testplan mit Thread Group, Throughput Controllern für den Nachrichtenmix und SMTP Samplern, Lauf im CLI-Modus mit HTML-Report und Auswertung der Perzentile pro Nachrichtenklasse. Frage zuerst nach Zielsystem, Nachrichtenklassen und gewünschtem Volumen.
translationOf: jmeter-smtp-lasttest-html-report
url: https://rafaelpfister.ch/it/blog/test-di-carico-smtp-con-apache-jmeter-nella-pratica-10-000-email-cinque-percorsi-di-regole-un
translationSourceHash: a41d58b7a4a717db179b3fec1ef8fac7961ff3ee12069f65627ddb48338aef0a
translationModel: gpt-5.6-terra
translatedAt: 2026-08-25T04:08:57.408Z
translationReview: required
---

# Test di carico SMTP con Apache JMeter nella pratica: 10'000 email, cinque percorsi di regole, un report HTML

L'[articolo panoramico sui test di carico della posta](/blog/mail-lasttest-tools-linux-windows-vergleich) ha confrontato gli strumenti e delineato il piano di test. Questo articolo mette alla prova il tutto: un test di carico JMeter completamente eseguito con 10'000 email, un mix di messaggi lungo percorsi di regole reali del gateway e il report HTML come valutazione. Tutti i valori mostrati provengono dall'esecuzione effettiva, inclusi gli errori verificatisi lungo il percorso.

Lo scenario riproduce un progetto reale: un gateway di crittografia email basato su Apache James (Totemomail) è collegato come loop di smarthost dietro Exchange Online e decide per ogni messaggio in merito a crittografia, firma e instradamento speciale. Il ruleset Mailet prevede diversi percorsi: trigger nell'oggetto quali (sec), (sign) e (unsec), parole chiave come VERTRAULICH per l'instradamento verso un gateway di settore e il percorso standard con controllo dei certificati e fallback in chiaro. Un test di carico che inviasse un solo tipo di messaggio misurerebbe sempre lo stesso percorso attraverso questo insieme di regole; il piano di test rappresenta quindi cinque classi, la cui proporzione corrisponde al traffico previsto.

## La configurazione: senza dover installare nulla

Il test è stato eseguito su una macchina Windows senza Java né JMeter. Entrambi possono essere utilizzati in modalità portabile, un aspetto decisivo su postazioni di amministrazione con diritti di installazione limitati: JRE Temurin in formato ZIP da Adoptium, JMeter in formato ZIP da apache.org, estrarre entrambi gli archivi, impostare `JAVA_HOME` sulla directory della JRE, fatto.

```bash
export JAVA_HOME="$PWD/jdk-21-jre"
export PATH="$JAVA_HOME/bin:$PATH"
./apache-jmeter-5.6.3/bin/jmeter -n -t gateway-lasttest.jmx -l lauf.jtl -e -o report
```

Come sink è stata usata una black box SMTP locale basata su aiosmtpd, poco più di 40 righe di Python: accetta ogni messaggio con `250`, ne scarta il contenuto, lo conta e assegna ogni email a una classe in base all'oggetto. Questo conteggio indipendente sul lato di ricezione è la verifica di controllo del test; se i numeri del generatore e del sink non coincidono, qualcosa è andato perso lungo il percorso.

```python
from aiosmtpd.controller import Controller

class SinkHandler:
    def __init__(self):
        self.count = 0

    async def handle_DATA(self, server, session, envelope):
        self.count += 1
        # Estrarre l'oggetto dall'header per le statistiche delle classi,
        # Il contenuto viene scartato
        return "250 Message accepted for delivery"

controller = Controller(SinkHandler(), hostname="127.0.0.1", port=2525)
controller.start()
```

Importante per l'interpretazione: generatore e sink erano in esecuzione sulla stessa macchina, senza TLS né rete intermedia. I valori misurati non dicono quindi nulla su un gateway, ma costituiscono l'autotest del generatore dell'articolo panoramico: la prova che la configurazione di carico possa effettivamente produrre il rate target e il limite superiore con cui confrontare in seguito le misurazioni contro il sistema di test reale.

## Il piano di test: cinque classi di messaggi, un rapporto di mix

Il cuore del piano è un Thread Group con 20 thread, 10 secondi di ramp-up e 500 cicli, quindi 10'000 iterazioni. Al suo interno si trovano cinque Throughput Controller in modalità "Percent Executions", ciascuno con esattamente un SMTP Sampler:

| Classe (etichetta del sampler) | Quota | Percorso di regole nel gateway |
|---|---|---|
| 01 Standard senza trigger | 60 % | Controllo AutoGenerated, controllo del certificato, fallback in chiaro |
| 02 Trigger (sec) | 15 % | Busta TRE per destinatari senza certificato |
| 03 Trigger (sign) | 10 % | Certificate Exchange: firmare, inviare la chiave |
| 04 Parola chiave VERTRAULICH | 10 % | Instradamento speciale verso il gateway di settore |
| 05 Trigger (unsec) | 5 % | Testo in chiaro forzato |

La suddivisione in cinque sampler separati anziché in un unico sampler con oggetto variabile ha un motivo concreto: il report HTML raggruppa tutti gli indicatori per etichetta del sampler. Cinque etichette producono cinque righe nelle statistiche, con percentili distinti per classe; un singolo sampler con oggetto alimentato da CSV produrrebbe un'unica riga aggregata e la differenza tra i percorsi di regole resterebbe invisibile nella valutazione.

Ogni sampler compila i normali campi: host e porta di destinazione come variabili personalizzate (`${zielhost}`, `${zielport}`), affinché lo stesso piano possa essere eseguito senza modifiche contro sink, ambiente di test o PreProd, oltre a mittente, destinatario, oggetto con un marcatore chiaro (qui la parola LOADTEST nell'oggetto) e un corpo di testo di circa 1-2 KB. L'opzione "Include timestamp in subject" aggiunge all'oggetto l'istante di consegna in millisecondi; in una successiva esecuzione contro un sistema reale multilivello, ciò consente di calcolare la latenza end-to-end per messaggio insieme agli orari di ricezione del sink.

Un ostacolo emerso in questa esecuzione, generalizzabile: il primo tentativo è fallito con 10'000 errori in 10 secondi, tutti con `java.lang.ClassCastException ... NullProperty cannot be cast to ... CollectionProperty` invece di una risposta SMTP. La causa era un file JMX creato manualmente, nel quale mancava l'elenco degli header del sampler; il sampler richiede obbligatoriamente questa proprietà, anche se vuota. L'insegnamento non riguarda tanto la proprietà concreta quanto il modello: creare i piani di test nella GUI e salvarli, non scrivere XML a mano, e prima di ogni burst effettuare una piccola esecuzione controllando sul sink che oggetto e contenuto arrivino davvero. Un contatore di errori al 100% con tempo di risposta di 0 ms significa quasi sempre che l'errore avviene prima della rete e che il test non ha quindi mai raggiunto il sistema di destinazione.

## L'esecuzione

La misurazione vera e propria viene eseguita in modalità CLI; la GUI serve solo da editor. Un unico comando produce esecuzione, dati grezzi e report:

```bash
jmeter -n -t gateway-lasttest.jmx -l lauf-10k.jtl -e -o report-10k
```

Il summariser nella console mostra l'andamento in tempo reale, il risultato finale dell'esecuzione:

```text
summary =  10000 in 00:00:13 =  781.8/s Avg: 6 Min: 1 Max: 27 Err: 0 (0.00%)
```

10'000 messaggi in 12.8 secondi, 782 messaggi al secondo in media, nessun errore. Il sink ha confermato in modo indipendente esattamente 10'000 email accettate con il mix 6000 / 1500 / 1000 / 1000 / 500: il rapporto dei Throughput Controller è quindi risultato esatto al singolo messaggio.

## Il report HTML

L'argomento a favore di JMeter rispetto a generatori più snelli come smtp-source è la valutazione, che il report Dashboard fornisce senza lavoro aggiuntivo:

![Dashboard JMeter dell'esecuzione: APDEX 1.000 per tutte e cinque le classi, Requests Summary 100 percento PASS, tabella statistica con percentili per classe di messaggi](../images/jmeter-report-dashboard.png)

La tabella statistica è la parte più importante del report. Per ogni etichetta del sampler, quindi per ogni classe di messaggi, riporta quantità, tasso di errore, media, mediana, percentile 90, 95 e 99, massimo e throughput. Nell'esecuzione concreta: mediana di 7 ms, p95 a 11 ms, p99 a 12 ms, massimo 27 ms, praticamente identici per tutte e cinque le classi. Con un sink locale che tratta ogni messaggio allo stesso modo, questo è esattamente il risultato atteso e allo stesso tempo il valore di riferimento: se in seguito lo stesso piano viene eseguito contro il gateway reale e la classe (sec) mostra improvvisamente un multiplo della mediana standard, quella è l'elaborazione aggiuntiva del percorso di crittografia, isolata correttamente per ogni ramo di regole.

Il blocco APDEX sopra condensa lo stesso risultato in un valore per classe (qui ovunque 1.000, poiché tutte le risposte erano molto al di sotto della soglia di tolleranza di 500 ms); le soglie possono essere adattate ai propri obiettivi di servizio nelle proprietà del report. Il blocco Errors resta vuoto in questa esecuzione, ma nei test contro sistemi reali è il primo punto da consultare: raggruppa gli errori per testo di risposta, consentendo di distinguere immediatamente un throttling `421` del sistema di destinazione dalle interruzioni di connessione.

Un ostacolo anche qui, che riguarda ogni burst breve: per impostazione predefinita, i grafici delle serie temporali del report lavorano con una granularità di un minuto. Un'esecuzione di 13 secondi collassa quindi in un unico punto dati e le curve sotto "Charts" sembrano un errore di misurazione. Il report può essere rigenerato dal file JTL esistente con una risoluzione più fine, senza una nuova esecuzione:

```bash
jmeter -g lauf-10k.jtl -o report-fein -Jjmeter.reportgenerator.overall_granularity=1000
```

Con granularità al secondo, il singolo punto diventa il reale andamento del carico:

![Hits per Second con granularità di 1 secondo: aumento durante il ramp-up di 10 secondi fino a un plateau di circa 840 messaggi al secondo, poi calo ripido alla fine del test](../images/jmeter-report-hits-per-second.png)

La curva mostra il ramp-up di 10 secondi, un plateau attorno a 840 messaggi al secondo e il calo finale, quando i primi thread hanno completato i loro 500 cicli. Per l'interpretazione conta il plateau, non la media sull'intera esecuzione: la media di 782/s include ramp-up e fase di uscita e sottostima il rate sostenuto raggiunto.

## Cosa dimostra questa esecuzione e cosa no

Questa esecuzione dimostra che il piano di test è funzionalmente corretto (piccola esecuzione con controllo del contenuto sul sink), che il rapporto di mix è esatto e che il generatore su questa macchina raggiunge almeno 840 messaggi al secondo senza TLS. Chi desidera testare con esso un gateway progettato per 100 email al secondo dispone di una riserva di un fattore otto e può attribuire con buona sicurezza i colli di bottiglia al sistema di destinazione.

Non dimostra tutto il resto, e questa delimitazione deve far parte di ogni rapporto di test: nessuna affermazione sui costi dell'handshake TLS (il percorso reale usa STARTTLS), nessuna sul comportamento della coda del gateway, nessuna sul tempo di elaborazione dei percorsi di regole. A questo scopo, lo stesso piano con le variabili `zielhost`/`zielport` modificate punta all'ambiente di test del gateway; la valutazione viene quindi eseguita in modo identico, integrata dai log del gateway e dall'osservazione della coda dell'articolo panoramico. Proprio questa riutilizzabilità, un piano per sink, ambiente di test e PreProd con una valutazione identica, è il vero motivo per affrontare una volta lo sforzo di creare un piano JMeter pulito.

## Fonti

1.  [Apache JMeter User's Manual: Component Reference, SMTP Sampler](https://jmeter.apache.org/usermanual/component_reference.html): riferimento ai campi del sampler, inclusi header, opzione timestamp e invio EML.

2.  [Apache JMeter: Generating Dashboard Report](https://jmeter.apache.org/usermanual/generating-dashboard.html): generazione del report HTML dall'esecuzione o successivamente dal JTL, incluse le proprietà di granularità e APDEX.

3.  [Apache JMeter: Test Plan, Logic Controllers](https://jmeter.apache.org/usermanual/component_reference.html#Throughput_Controller): funzionamento del Throughput Controller in modalità Percent Executions per il mix di messaggi.

4.  [aiosmtpd, documentazione](https://aiosmtpd.aio-libs.org/): il server SMTP basato su asyncio, con cui il sink viene creato in poche righe di Python.

5.  [Eclipse Temurin, Adoptium](https://adoptium.net/temurin/releases/): archivi JRE portabili per eseguire JMeter senza installare Java.
