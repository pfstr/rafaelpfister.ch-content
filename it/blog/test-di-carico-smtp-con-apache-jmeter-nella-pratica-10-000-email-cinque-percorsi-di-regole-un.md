---
title: "Test di carico SMTP con Apache JMeter nella pratica: 10'000 e-mail, cinque percorsi di regole, un report HTML"
navTitle: "Test di carico con JMeter"
description: "Un test di carico eseguito dalla A alla Z: piano di test con mix di messaggi lungo i percorsi del ruleset di un gateway di crittografia, configurazione portabile senza installazione, 10'000 e-mail in burst e analisi tramite il report HTML di JMeter, compresi i problemi realmente riscontrati."
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
translationSourceHash: 26c09e391d2252b6203dceb5dc45edd23beba797820fe0b95273bf48a9afc181
translationModel: gpt-5.6-terra
translatedAt: 2026-08-30T09:23:18.151Z
translationReview: required
url: https://rafaelpfister.ch/it/blog/test-di-carico-smtp-con-apache-jmeter-nella-pratica-10-000-email-cinque-percorsi-di-regole-un
---

# Test di carico SMTP con Apache JMeter nella pratica: 10'000 e-mail, cinque percorsi di regole, un report HTML

L'[articolo introduttivo sui test di carico della posta](/blog/mail-lasttest-tools-linux-windows-vergleich) ha confrontato gli strumenti e delineato il piano di test. Qui segue l'esecuzione pratica: un test di carico completo con JMeter con 10'000 e-mail, un mix di messaggi lungo percorsi di regole reali del gateway e il report HTML come analisi. Tutti i valori mostrati provengono dall'esecuzione effettiva, inclusi gli errori verificatisi durante il percorso.

Lo scenario riproduce un progetto reale: un gateway di crittografia e-mail basato su Apache James (Totemomail) è collegato come loop di smarthost dietro Exchange Online e decide per ogni messaggio la crittografia, la firma e il routing speciale. Il ruleset Mailet prevede diversi percorsi: trigger nell'oggetto come (sec), (sign) e (unsec), parole chiave come VERTRAULICH per il routing verso un gateway di settore e il percorso standard con verifica del certificato e fallback in chiaro. Un test di carico che invia un solo tipo di messaggio misurerebbe sempre lo stesso percorso attraverso questo set di regole; il piano di test modella quindi cinque classi, la cui proporzione corrisponde al traffico previsto.

Importante per l'interpretazione: questo piano di test genera il carico di molti mittenti indipendenti, poiché JMeter apre una connessione separata per ogni messaggio (i dettagli sono riportati nella delimitazione alla fine). Per dimostrare che un set di regole funziona correttamente e con sufficiente rapidità sotto traffico misto parallelo, questo è il modello adatto. Il piano non riproduce invece il carico di picco di un singolo mittente di massa con sessioni aperte; per questo profilo di carico, `smtp-source` dell'[articolo introduttivo](/blog/mail-lasttest-tools-linux-windows-vergleich) è lo strumento giusto.

## Le principali opzioni di jmeter

Per orientarsi, ecco le opzioni da riga di comando presenti in questo articolo, tradotte liberamente dalla documentazione:

<details class="options-details">
<summary>Panoramica delle opzioni</summary>

| Opzione | Significato |
|---|---|
| `-n` | Modalità CLI (non-GUI): esegue il piano di test senza interfaccia grafica |
| `-t datei` | Percorso del file JMX contenente il piano di test |
| `-l datei` | Percorso del file dei risultati JTL in cui vengono scritti i valori misurati |
| `-e` | Genera direttamente il report HTML Dashboard dopo l'esecuzione |
| `-o verzeichnis` | Directory di destinazione del report; deve essere vuota o non deve ancora esistere |
| `-g datei` | Genera il report successivamente da un file JTL esistente, senza una nuova esecuzione |
| `-J<property>=<wert>` | Imposta una property JMeter solo per questa chiamata |

</details>

L'elenco completo è mostrato da `jmeter -?`; le opzioni sono descritte nel capitolo sul funzionamento non-GUI del [JMeter User's Manual](https://jmeter.apache.org/usermanual/get-started.html).

## La configurazione: senza dover installare nulla

Il test è stato eseguito su una macchina Windows senza Java né JMeter. Entrambi possono essere usati in modo portabile, un aspetto decisivo sulle workstation amministrative con diritti di installazione limitati: Temurin JRE come ZIP da Adoptium, JMeter come ZIP da apache.org, estrarre entrambi, impostare `JAVA_HOME` sulla directory della JRE, fatto.

```bash
export JAVA_HOME="$PWD/jdk-21-jre"
export PATH="$JAVA_HOME/bin:$PATH"
./apache-jmeter-5.6.3/bin/jmeter -n -t gateway-lasttest.jmx -l lauf.jtl -e -o report
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `export JAVA_HOME=…` | Indica la directory JRE estratta; JMeter individua così il runtime Java senza installazione |
| `export PATH=…` | Porta i binari della JRE all'inizio del percorso di ricerca |
| `-n` | Modalità CLI senza interfaccia grafica |
| `-t gateway-lasttest.jmx` | Il piano di test da eseguire |
| `-l lauf.jtl` | File dei risultati con i valori misurati da ogni sampler |
| `-e` | Genera il report HTML direttamente dopo l'esecuzione |
| `-o report` | Directory di destinazione del report |

</details>

Come sink è stata utilizzata una blackbox SMTP locale basata su aiosmtpd, poco più di 40 righe di Python: accetta ogni messaggio con `250`, scarta il contenuto, conta e assegna ogni e-mail a una classe in base all'oggetto. Questo conteggio indipendente sul lato ricevente è il controllo del test; se i conteggi del generatore e del sink non coincidono, qualcosa è andato perso lungo il percorso.

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

Importante per l'interpretazione: generatore e sink erano in esecuzione sulla stessa macchina, senza TLS e senza rete intermedia. I valori misurati non costituiscono quindi un'affermazione su un gateway, bensì l'autotest del generatore dell'articolo introduttivo: la prova che la struttura di carico può effettivamente generare la velocità target e il limite superiore con cui confrontare le successive misurazioni sul sistema di test reale.

## Il piano di test: cinque classi di messaggi, una proporzione mista

Il cuore del piano è un Thread Group con 20 thread, 10 secondi di ramp-up e 500 cicli, quindi 10'000 iterazioni. Sotto di esso si trovano cinque Throughput Controller in modalità "Percent Executions", ciascuno con esattamente un SMTP Sampler:

| Classe (etichetta sampler) | Quota | Percorso di regole nel gateway |
|---|---|---|
| 01 Standard senza trigger | 60 % | Verifica AutoGenerated, verifica del certificato, fallback in chiaro |
| 02 Trigger (sec) | 15 % | Envelope TRE per destinatari senza certificato |
| 03 Trigger (sign) | 10 % | Certificate Exchange: firmare, inviare la chiave |
| 04 Parola chiave VERTRAULICH | 10 % | Routing speciale verso il gateway di settore |
| 05 Trigger (unsec) | 5 % | Testo in chiaro forzato |

La suddivisione in cinque sampler distinti anziché in un solo sampler con oggetto variabile ha una ragione concreta: il report HTML raggruppa tutti gli indicatori per etichetta del sampler. Cinque etichette producono cinque righe nelle statistiche con percentili propri per classe; un singolo sampler con oggetto alimentato da CSV produrrebbe un'unica riga aggregata e la differenza tra i percorsi di regole sarebbe invisibile nell'analisi.

Ogni sampler compila i consueti campi: host e porta di destinazione come variabili definite dall'utente (`${zielhost}`, `${zielport}`), così che lo stesso piano possa essere eseguito senza modifiche contro sink, ambiente di test o PreProd, oltre a mittente, destinatario, oggetto con un marcatore chiaro (qui la parola LOADTEST nell'oggetto) e un corpo di testo di circa 1-2 KB. L'opzione "Include timestamp in subject" aggiunge all'oggetto il momento dell'invio in millisecondi; in una successiva esecuzione contro un vero sistema a più livelli, questo consente di calcolare, insieme agli orari di ricezione del sink, la latenza end-to-end per messaggio.

Un errore di questa esecuzione che può essere generalizzato: il primo tentativo è fallito con 10'000 errori in 10 secondi, tutti con `java.lang.ClassCastException ... NullProperty cannot be cast to ... CollectionProperty` anziché con una risposta SMTP. La causa era un file JMX costruito a mano, in cui mancava l'elenco degli header del sampler; il sampler richiede obbligatoriamente questa property, anche se vuota. La lezione non riguarda tanto la property specifica quanto il modello: creare e salvare i piani di test nella GUI, non scrivere l'XML a mano, ed eseguire un test minimo prima di ogni burst, verificando sul sink che oggetto e contenuto arrivino davvero. Un contatore di errori del 100 percento con tempo di risposta di 0 ms significa quasi sempre che l'errore si verifica prima della rete e che il test non ha quindi mai raggiunto il sistema di destinazione.

## L'esecuzione

La misurazione stessa viene eseguita in modalità CLI; la GUI serve solo da editor. Una singola chiamata genera esecuzione, dati grezzi e report:

```bash
jmeter -n -t gateway-lasttest.jmx -l lauf-10k.jtl -e -o report-10k
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-n` | Modalità CLI: il piano di test viene eseguito senza GUI, solo il summariser scrive nella console |
| `-t gateway-lasttest.jmx` | Il piano di test creato nella GUI |
| `-l lauf-10k.jtl` | Dati grezzi dell'esecuzione; da questo file il report può essere generato nuovamente in seguito |
| `-e` | Genera il report immediatamente dopo l'esecuzione |
| `-o report-10k` | Directory di destinazione per il report HTML |

</details>

Il summariser nella console mostra l'andamento in tempo reale, il risultato finale dell'esecuzione:

```text
summary =  10000 in 00:00:13 =  781.8/s Avg: 6 Min: 1 Max: 27 Err: 0 (0.00%)
```

10'000 messaggi in 12.8 secondi, 782 messaggi al secondo in media, nessun errore. Il sink ha confermato indipendentemente esattamente 10'000 e-mail accettate con il mix 6000 / 1500 / 1000 / 1000 / 500; la proporzione mista dei Throughput Controller corrispondeva dunque al messaggio esatto.

## Il report HTML

L'argomento a favore di JMeter rispetto a generatori più snelli come smtp-source è l'analisi, e il report Dashboard la fornisce senza lavoro aggiuntivo:

![Dashboard JMeter dell'esecuzione: APDEX 1.000 per tutte e cinque le classi, Requests Summary 100 percento PASS, tabella statistica con percentili per classe di messaggi](../images/jmeter-report-dashboard.png)

La tabella statistica è la parte più importante del report. Per ogni etichetta del sampler, ovvero per ogni classe di messaggi, riporta numero, tasso di errore, media, mediana, 90°, 95° e 99° percentile, massimo e throughput. Nell'esecuzione concreta: mediana di 7 ms, p95 a 11 ms, p99 a 12 ms, massimo 27 ms, praticamente identico per tutte e cinque le classi. Con un sink locale che tratta ogni messaggio allo stesso modo, questo è esattamente il risultato atteso e al tempo stesso il valore di riferimento: se lo stesso piano viene eseguito successivamente contro il gateway reale e la classe (sec) mostra improvvisamente un multiplo della mediana standard, si tratta del lavoro aggiuntivo del percorso di crittografia, isolato in modo pulito per ogni ramo di regole.

Il blocco APDEX soprastante riassume lo stesso dato in un valore per classe (qui ovunque 1.000, perché tutte le risposte erano ben al di sotto della soglia di tolleranza di 500 ms); le soglie possono essere adattate ai propri obiettivi di servizio nelle property del report. Il blocco Errors rimane vuoto in questa esecuzione, ma nei test contro sistemi reali è il primo punto da consultare: raggruppa gli errori per testo di risposta, in modo che una limitazione `421` del sistema di destinazione sia immediatamente distinguibile dalle interruzioni di connessione.

Anche qui c'è un tipico errore di analisi, che riguarda ogni breve burst: per impostazione predefinita, i grafici delle serie temporali del report usano una granularità di un minuto. Un'esecuzione di 13 secondi si riduce quindi a un unico punto dati e le curve sotto "Charts" sembrano un errore di misurazione. Il report può essere rigenerato dal file JTL esistente senza una nuova esecuzione, con una risoluzione più fine:

```bash
jmeter -g lauf-10k.jtl -o report-fein -Jjmeter.reportgenerator.overall_granularity=1000
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-g lauf-10k.jtl` | Genera il report dal file JTL esistente, senza eseguire di nuovo il test |
| `-o report-fein` | Nuova directory di destinazione; la directory del report esistente rimane invariata |
| `-Jjmeter.reportgenerator.overall_granularity=1000` | Imposta la granularità dei grafici per questa chiamata a 1'000 ms anziché al minuto predefinito |

</details>

Con granularità al secondo, dal singolo punto emerge l'effettivo andamento del carico:

![Hits per Second con granularità di 1 secondo: aumento durante i 10 secondi di ramp-up fino a un plateau di circa 840 messaggi al secondo, quindi brusco calo alla fine del test](../images/jmeter-report-hits-per-second.png)

La curva mostra il ramp-up di 10 secondi, un plateau di circa 840 messaggi al secondo e il calo finale, quando i primi thread hanno completato i loro 500 cicli. Per l'interpretazione conta il plateau, non la media dell'intera esecuzione: la media di 782/s include ramp-up e fase finale e sottostima la velocità sostenuta raggiunta.

## Cosa dimostra questa esecuzione e cosa no

Dopo questa esecuzione è dimostrato quanto segue: il piano di test è funzionalmente corretto (test minimo con controllo dei contenuti sul sink), il mix corrisponde esattamente e il generatore raggiunge su questa macchina almeno 840 messaggi al secondo senza TLS. Chi desidera testare con esso un gateway progettato per 100 e-mail al secondo dispone di un margine di un fattore otto e può attribuire con serenità i colli di bottiglia al sistema di destinazione.

Non è dimostrato tutto il resto, e questa delimitazione deve far parte di ogni rapporto di test: nessuna affermazione sui costi dell'handshake TLS (il percorso reale utilizza STARTTLS), nessuna sul comportamento della coda del gateway, nessuna sul tempo di elaborazione dei percorsi di regole. A tale scopo, lo stesso piano, con le variabili `zielhost`/`zielport` impostate sull'ambiente di test del gateway, punta a quest'ultimo; l'analisi procede quindi in modo identico, integrata dai log del gateway e dall'osservazione della coda dell'articolo introduttivo. Proprio questa riutilizzabilità, un piano per sink, ambiente di test e PreProd con analisi identica, è il vero motivo per affrontare una volta lo sforzo di realizzare un piano JMeter pulito.

Anche un limite dello strumento stesso va incluso nella delimitazione: JMeter non può mantenere aperte le sessioni SMTP. L'SMTP Sampler apre una nuova connessione per ogni messaggio, esegue EHLO, eventualmente STARTTLS e AUTH, e la chiude dopo esattamente una transazione con QUIT. Gli 840 messaggi al secondo includono quindi una creazione completa della connessione per ciascun messaggio. Un mittente di massa che invia centinaia di messaggi tramite una sessione aperta genera sul gateway un profilo di carico diverso, con meno connessioni e più transazioni per connessione; di conseguenza, con il carico di JMeter i limiti di connessione intervengono prima. Il motivo risiede nell'architettura del framework: JMeter misura ogni sampler come unità autonoma e indipendente, affinché timer, assertion e percentili funzionino allo stesso modo per tutti i protocolli supportati, e l'SMTP Sampler è basato sulla libreria JavaMail, che come API client connette e disconnette per ogni operazione di invio. Per SMTP non esiste un riutilizzo della connessione come il Keep-Alive dell'HTTP Sampler. Per il profilo di carico di un mittente bulk con sessione aperta sono più adatti `smtp-source` o uno script dedicato; il confronto degli strumenti nell'articolo introduttivo lo contestualizza.

## Fonti

1.  [Apache JMeter User's Manual: Component Reference, SMTP Sampler](https://jmeter.apache.org/usermanual/component_reference.html): riferimento dei campi del sampler, inclusi header, opzione timestamp e invio EML.

2.  [Apache JMeter: Generating Dashboard Report](https://jmeter.apache.org/usermanual/generating-dashboard.html): generazione del report HTML dall'esecuzione o successivamente dal JTL, incluse le property di granularità e APDEX.

3.  [Apache JMeter: Test Plan, Logic Controllers](https://jmeter.apache.org/usermanual/component_reference.html#Throughput_Controller): funzionamento del Throughput Controller in modalità Percent Executions per il mix di messaggi.

4.  [aiosmtpd, documentazione](https://aiosmtpd.aio-libs.org/): il server SMTP basato su asyncio con cui si realizza il sink in poche righe di Python.

5.  [Eclipse Temurin, Adoptium](https://adoptium.net/temurin/releases/): archivi JRE portabili per usare JMeter senza installare Java.

6.  [Apache JMeter: Getting Started, Non-GUI Mode](https://jmeter.apache.org/usermanual/get-started.html): panoramica delle opzioni da riga di comando per il funzionamento CLI, incluse `-n`, `-t`, `-l`, `-e`, `-o`, `-g` e `-J`.
