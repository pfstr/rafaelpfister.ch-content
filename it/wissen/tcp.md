---
title: "TCP: stati, flusso di dati e diagnostica"
blatt: "tcp"
description: "TCP come base di trasporto per servizi di messaggistica e directory: modello dei socket, stati di connessione, numeri di sequenza, conferme, controllo di flusso e congestione, timer, errori tipici e strumenti diagnostici per amministratori."
fakten:
  - label: Nome completo
    wert: Transmission Control Protocol
    href: https://datatracker.ietf.org/doc/html/rfc9293
  - label: Standard
    wert: RFC 9293 · STD 7
    href: https://www.rfc-editor.org/info/rfc9293
  - label: Modello di servizio
    wert: flusso di byte affidabile, ordinato e bidirezionale
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-2
  - label: Endpoint
    wert: indirizzo IP e numero di porta a 16 bit
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-3.1
  - label: Connessione
    wert: coppia univoca di socket locale e remoto
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-3.1
  - label: Header
    wert: almeno 20 byte più opzioni
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-3.1
  - label: Affidabilità
    wert: numeri di sequenza, ACK cumulativi e ritrasmissione
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-3.3
  - label: Controllo di flusso
    wert: finestra di ricezione rwnd
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-3.8.4
  - label: Controllo della congestione
    wert: cwnd · Slow Start · Congestion Avoidance
    href: https://datatracker.ietf.org/doc/html/rfc5681
  - label: Intervalli di porte
    wert: 0–1023 · 1024–49151 · 49152–65535
    href: https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml
  - label: Crittografia
    wert: nessuna; protezione tramite TLS, IPsec o TCP-AO
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-7
  - label: Diagnostica
    wert: test della porta · stato del socket · cattura dei pacchetti
    href: https://www.tcpdump.org/manpages/tcpdump.1.html
werbung:
  - newsletter
ctaThemen:
  - smtp-mailflow
translationSourceHash: 6c749be6d80debccf17ceb4386ba0e298c11deedf855db326e7d20da9593fc05
translationModel: gpt-5.6-terra
translatedAt: 2026-10-14T09:35:14.030Z
translationReview: automatic
---

# TCP: stati, flusso di dati e diagnostica

TCP trasforma il trasporto inaffidabile dei pacchetti di IP in un flusso di dati ordinato e bidirezionale per le applicazioni. L'applicazione scrive byte in un socket; TCP li segmenta, ne numera la posizione, conferma gli intervalli ricevuti e ritrasmette le lacune. SMTP, IMAP, POP3, LDAP e HTTPS si basano su questa garanzia. Quando un'applicazione di questo tipo «si blocca», l'amministratore deve quindi anzitutto distinguere se TCP attende ancora una conferma oppure se è già bloccato il protocollo applicativo.

L'articolo segue il ciclo di vita di una connessione: socket e listener, handshake a tre vie, trasferimento dati, controllo di flusso e congestione, chiusura e TIME-WAIT. Da questo normale svolgimento derivano poi diagnostica, pianificazione della capacità, monitoraggio e hardening.

TCP non conosce tuttavia né messaggi né utenti, certificati o indirizzi e-mail. Non preserva neppure i confini dell'applicazione: due operazioni di scrittura consecutive possono essere riunite in un segmento o distribuite su più segmenti. Un handshake TCP riuscito dimostra pertanto soltanto che due endpoint TCP hanno potuto stabilire una connessione. Se successivamente appare un banner SMTP, viene negoziato TLS o riesce un bind LDAP, lo decide l'applicazione sovrastante.

Per gli amministratori, TCP è rilevante meno come modello a livelli che come **sistema di stati e segnali**. SYN, ACK, FIN e RST indicano in quale punto di passaggio una connessione fallisce. Stati dei socket, ritrasmissioni, finestre di ricezione e timer spiegano perché un servizio possa essere raggiungibile ma lento o apparentemente bloccato.

La spiegazione segue una connessione TCP dal primo SYN attraverso il flusso di byte ordinato fino alla chiusura o al reset. Finestre, ritrasmissioni, congestione e diagnostica vengono illustrati nelle rispettive fasi.

## Modello di socket e connessione

Prima che i pacchetti fluiscano, un'applicazione associa un socket a indirizzo e porta. Un server ascolta, ad esempio, su TCP 25; ogni connessione accettata è quindi identificata univocamente da indirizzo sorgente, porta sorgente, indirizzo di destinazione e porta di destinazione. Per questo migliaia di client possono usare lo stesso listener, purché differiscano almeno per l'indirizzo sorgente o la porta effimera. Per la diagnostica, «porta 25» non è quindi sufficiente: occorre sempre la coppia di connessione completa ([RFC 9293, sezione 3.1](https://datatracker.ietf.org/doc/html/rfc9293#section-3.1)).

Su un server, un processo associa un socket a un indirizzo locale e lo porta nello stato **LISTEN**. La scelta dell'indirizzo di bind è operativamente importante:

| Binding | Visibilità | Conseguenza tipica |
|---|---|---|
| `127.0.0.1:25` o `::1:25` | solo host locale | Il monitoraggio sul server è normale, i sistemi remoti non ottengono una connessione |
| `0.0.0.0:25` | tutti gli indirizzi IPv4 locali | Il listener segue gli indirizzi IPv4 aggiuntivi, se firewall e routing lo consentono |
| `[::]:25` | wildcard IPv6 | Il comportamento dual stack dipende dal sistema operativo e dall'opzione `IPV6_V6ONLY` |
| IP di servizio concreto | solo questo indirizzo | Separazione controllata, ma sensibile a cambiamenti di IP o cluster |

La porta del server è generalmente definita da una [registrazione IANA](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml); il client sceglie solitamente una porta effimera nell'intervallo gestito dal sistema operativo. Firewall, gateway NAT e bilanciatori di carico devono quindi consentire non soltanto la nota porta di destinazione, ma l'intero percorso di ritorno verso la porta sorgente dinamica.

### TCP Control Block

Per ogni connessione, lo stack TCP mantiene un **Transmission Control Block** (TCB). Contiene, tra l'altro, stato, socket locali e remoti, sequenze di invio e ricezione, finestre, timer e parametri di sicurezza. Il TCB è il motivo per cui una connessione mantiene lo stato e per cui i bilanciatori di carico o i firewall devono operare con un percorso di ritorno simmetrico: se un segmento di risposta raggiunge un altro nodo senza uno stato corrispondente, quest'ultimo può scartarlo o rispondere con RST.

## Stabilimento della connessione, trasferimento dati e chiusura

Con l'apertura attiva inizia lo stato di connessione visibile. Il client invia SYN con il numero di sequenza iniziale, il listener risponde con SYN+ACK e il proprio numero di sequenza, quindi il client conferma con ACK. Già questi tre segmenti negoziano opzioni quali Maximum Segment Size, Window Scale, Selective Acknowledgment e Timestamps. Se manca un pacchetto, la risposta arriva da un'altra strada o un prodotto intermedio modifica le opzioni, l'handshake mostra già dove raggiungibilità e prestazioni successive si separano.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 976" src="/images/kb-interaktiv-tcp-verbindung.svg?v=20260813" title="Interaktive Infografik: TCP-Verbindungslebenszyklus und Diagnosepunkte" loading="lazy">
  <a href="/images/kb-interaktiv-tcp-verbindung.svg?v=20260813">Apri l'infografica sul ciclo di vita della connessione TCP</a>
</iframe>

Dopo l'handshake, la connessione si trova in **ESTABLISHED**. Entrambi i lati possono inviare dati indipendentemente l'uno dall'altro e chiudere la propria direzione di invio con FIN. Una chiusura ordinata richiede quindi normalmente un FIN e un ACK per ogni direzione. Il lato che invia l'ultimo ACK rimane in **TIME-WAIT**, affinché segmenti ritardati di una vecchia connessione non finiscano in una nuova connessione con la stessa coppia di socket ([RFC 9293, sezioni 3.3.2 e 3.6](https://datatracker.ietf.org/doc/html/rfc9293#section-3.3.2)).

Un **RST** non è una chiusura ordinata. Interrompe la connessione e scarta gli stati in sospeso. Durante lo stabilimento della connessione, un RST valido significa solitamente che l'host di destinazione è raggiungibile, ma il socket non viene accettato. Dopo che sono già stati trasferiti dati, un RST può provenire dall'applicazione, dal sistema operativo o da un nodo intermedio stateful. La posizione del RST nella cattura dei pacchetti è quindi più importante del messaggio d'errore generico «connection reset».

### Gli stati come indicatori operativi

| Stato | Significato tecnico | Cosa suggerisce un accumulo |
|---|---|---|
| `LISTEN` | socket passivo in attesa di SYN | stato server previsto; l'assenza di listener indica un problema di servizio o binding |
| `SYN-SENT` | SYN locale inviato, manca una risposta corrispondente | filtro, route, indirizzo errato o controparte sovraccarica |
| `SYN-RECEIVED` | SYN accettato, manca l'ultimo ACK | SYN flood, percorso di ritorno asimmetrico, problema del client o del firewall |
| `ESTABLISHED` | spazi di sequenza sincronizzati | normale; molte connessioni persistenti possono essere volute, ad esempio con IMAP o HTTP/2 |
| `FIN-WAIT-1/2` | il lato locale ha chiuso | la controparte o l'applicazione termina la propria direzione con ritardo |
| `CLOSE-WAIT` | FIN remoto ricevuto, l'applicazione locale non ha ancora chiuso | spesso una perdita di risorse o un thread applicativo bloccato |
| `LAST-ACK` | FIN locale inviato dopo la chiusura remota | manca l'ACK della controparte o il percorso di ritorno è disturbato |
| `TIME-WAIT` | la vecchia quadrupla è protetta prima del riutilizzo | normale sul lato che chiude attivamente; problematico solo insieme a pressione sulle porte o sul TCB |

I nomi degli stati sono definiti in [RFC 9293](https://datatracker.ietf.org/doc/html/rfc9293#section-3.3.2). Un numero elevato non è di per sé un errore. Sono determinanti età, tasso di crescita, controparti interessate e relazione con tasso di connessioni, descrittori di file, intervallo di porte effimere e limiti dell'applicazione.

## Numeri di sequenza, ACK e ritrasmissione

Dopo l'handshake, TCP conta il flusso dati byte per byte. Il numero di sequenza indica il primo byte di un segmento; l'ACK indica il byte successivo atteso e conferma quindi tutto ciò che lo precede. I pacchetti possono andare persi, arrivare duplicati o cambiare ordine. Il destinatario può memorizzarli nel buffer, ma consegna all'applicazione soltanto un flusso senza lacune. Proprio per questo una singola area mancante può bloccare un'applicazione apparentemente inattiva, benché pacchetti successivi siano già arrivati all'host.

Se un segmento o il suo ACK viene perso, il mittente può rilevare l'area mancante in due modi:

1. **Retransmission Timeout (RTO):** da Round-Trip Time smussato e relativa variazione viene derivato un timer. Alla sua scadenza viene ritrasmesso il segmento non confermato più vecchio. [RFC 6298](https://datatracker.ietf.org/doc/html/rfc6298) definisce calcolo, valori minimi e backoff esponenziale.
2. **Fast Retransmit:** più ACK duplicati indicano una lacuna nel flusso di dati. Il mittente può ritrasmettere l'area presumibilmente persa senza attendere l'RTO ([RFC 5681](https://datatracker.ietf.org/doc/html/rfc5681#section-3.2)).

Una ritrasmissione non dimostra automaticamente una perdita di pacchetti nella rete. Una cattura può essere stata effettuata su un solo lato, possono mancare pacchetti a causa di capture drop oppure può mostrare una Spurious Retransmission. Wireshark indica esplicitamente che la sua analisi TCP è un'interpretazione derivata dal flusso di pacchetti osservato ([Wireshark User’s Guide – TCP Analysis](https://www.wireshark.org/docs/wsug_html_chunked/ChAdvTCPAnalysis.html)). Per un'affermazione attendibile, è quindi necessario correlare, se possibile, catture presso client, server e un punto intermedio rilevante.

## Controllo di flusso e controllo della congestione

La sola affidabilità non impedisce ancora che un mittente veloce sovraccarichi il destinatario o la rete. TCP limita quindi i byte non confermati simultaneamente con due finestre, delle quali alla fine è efficace la più piccola:

- La **finestra di ricezione** `rwnd` protegge il destinatario. Descrive il buffer attualmente disponibile. Se il destinatario segnala Zero Window, il mittente deve attendere, salvo Window Probes periodici. Uno Zero Window persistente spesso non è un problema di rete, ma significa che l'applicazione ricevente non legge i dati del socket abbastanza velocemente.
- La **Congestion Window** `cwnd` protegge il percorso. Viene gestita localmente dal mittente e adattata in base a conferme e segnali di perdita. [RFC 5681](https://datatracker.ietf.org/doc/html/rfc5681) descrive Slow Start, Congestion Avoidance, Fast Retransmit e Fast Recovery.

In linea di massima, è efficace il minore tra `rwnd` e `cwnd`. Un'elevata larghezza di banda da sola non garantisce quindi un throughput elevato. Su un percorso con grande prodotto banda-latenza deve essere consentita una quantità sufficiente di dati non confermati. L'opzione Window Scale estende la finestra a 16 bit; Timestamps supportano, tra l'altro, la misurazione dell'RTT e la protezione da vecchi numeri di sequenza ([RFC 7323](https://datatracker.ietf.org/doc/html/rfc7323)). Entrambe le opzioni vengono negoziate durante l'handshake e non possono essere aggiunte successivamente alla stessa connessione.

Per brevi transazioni SMTP domina spesso il numero di round trip: handshake TCP, handshake TLS opzionale e varie risposte di protocollo. Per allegati di grandi dimensioni e flussi di replica diventano inoltre rilevanti RTT, perdita, finestre, offloading e algoritmo di controllo della congestione. Un problema di throughput è quindi «larghezza di banda» solo dopo aver escluso tempo di handshake, pause applicative, Zero Windows e ritrasmissioni.

## Timer, keepalive e timeout di inattività

Laddove non arriva una risposta immediata, i timer decidono il passo successivo. Il Retransmission Timeout attiva le ritrasmissioni dei dati, il Persist Timer mantiene una connessione nonostante Zero Window e TIME-WAIT impedisce che vecchi segmenti finiscano in una nuova connessione. Sistemi operativi e applicazioni aggiungono keepalive e User Timeout. Chi registra soltanto «timeout» perde queste differenze e quindi la vera causa dell'errore.

I TCP keepalive sono spesso disabilitati per impostazione predefinita oppure iniziano soltanto dopo lunghi periodi di inattività. Non sostituiscono un health check a livello applicativo. Un keepalive riuscito indica soltanto che il peer TCP risponde; non prova che un worker SMTP elabori messaggi o che una query LDAP possa essere servita. Viceversa, firewall e bilanciatori di carico interrompono spesso connessioni inattive prima dei sistemi finali. Un lato conserva quindi un socket apparentemente valido finché il successivo tentativo di invio dati non attiva un timeout o RST.

Per connessioni IMAP, pool LDAP o API di lunga durata, quattro valori devono quindi essere coerenti: timeout di inattività dell'applicazione, intervallo TCP keepalive, timeout di inattività di ogni nodo intermedio e logica di riconnessione del client. Il più breve timeout non annunciato determina il comportamento osservato.

## Diagnostica dal socket al pacchetto

La ricerca guasti diventa affidabile non appena descrive la stessa connessione dell'applicazione: host sorgente, nome di destinazione, indirizzo risolto, porta di destinazione, momento e protocollo atteso. Un ping non sostituisce queste indicazioni. ICMP può essere bloccato benché TCP funzioni; viceversa, una risposta Echo non prova né un listener né lo stato di un firewall stateful.

### Raggiungibilità e listener

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für den TCP-Verbindungstest">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection -ComputerName mail.example.ch -Port 25 -InformationLevel Detailed

Get-NetTCPConnection -State Listen -LocalPort 25 |
  Select-Object LocalAddress, LocalPort, OwningProcess</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">nc -vz -w 5 mail.example.ch 25

sudo ss -ltnp 'sport = :25'</code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) combina in Windows risoluzione dei nomi, indirizzo di destinazione, test della porta e diagnostica di routing opzionale. In Unix, [`nc`](https://man.openbsd.org/nc.1) apre in modo mirato connessioni TCP o UDP; [`telnet`](https://man.openbsd.org/telnet.1) può anch'esso aprire una porta TCP, ma gestisce il protocollo Telnet e non è un sostituto completamente trasparente. [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) legge le informazioni dei socket del kernel Linux e può filtrare stati, timer, code e processi. Un test di connessione riuscito apre e chiude una reale connessione TCP; sui servizi produttivi dovrebbe quindi essere impiegato in modo mirato e non come port scan ad alta frequenza.

### Cattura dei pacchetti e filtri di stato

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für den TCP-Paketmitschnitt">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">pktmon filter remove
pktmon filter add MailTcp -t TCP -p 25
pktmon start --capture --pkt-size 0 --file-name C:\Temp\mail-tcp.etl
# Problem reproduzieren, danach:
pktmon stop
pktmon etl2pcap C:\Temp\mail-tcp.etl --out C:\Temp\mail-tcp.pcapng</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">sudo tcpdump -i any -nn -s 0 -w /tmp/mail-tcp.pcap \
  'host 192.0.2.25 and tcp port 25'

ss -tin state established '( dport = :25 or sport = :25 )'</code></pre>
  </div>
</div>

[`Pktmon`](https://learn.microsoft.com/windows-server/networking/technologies/pktmon/pktmon) acquisisce pacchetti nello stack di rete Windows e può convertire il formato ETL in PCAPNG. In Unix, [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) usa un'espressione di cattura e qui scrive un file PCAP per l'analisi successiva. Una cattura completa conserva il payload e può contenere credenziali o contenuti dei messaggi. Va trattata come un'esportazione di e-mail: filtrarla strettamente, acquisirla per breve tempo, archiviarla in modo protetto e cancellarla dopo l'analisi. Con TLS restano visibili indirizzi, porte, dimensioni dei pacchetti, timing, metadati dell'handshake e segnali TCP, ma non il contenuto applicativo crittografato.

### Quadro dei segnali e punto di passaggio più probabile

| Osservazione | Prospettiva TCP | Prossima verifica più probabile |
|---|---|---|
| SYN viene ripetuto, nessuna risposta | lo stabilimento della connessione non raggiunge un peer rispondente | destinazione DNS, route, Security Group, firewall drop, percorso di ritorno asimmetrico |
| SYN riceve subito RST | il percorso di destinazione funziona, il socket viene rifiutato | listener, indirizzo di bind, destinazione NAT, configurazione della porta |
| handshake riuscito, nessun banner | TCP è stabilito, l'applicazione non fornisce dati | worker, thread pool, protocollo proxy, log applicativo, dipendenza backend |
| RST subito dopo i dati client | peer o middlebox scarta la sessione | protocollo errato, ACL, TLS su porta in chiaro, dispositivo di ispezione |
| ritrasmissioni e Dup ACK in aumento | perdita, riordinamento o pacchetti di cattura mancanti | errori dell'interfaccia, sovraccarico, MTU/PMTUD, cambio di percorso, cattura dalla controparte |
| Zero Window | il destinatario segnala buffer pieno | CPU, thread dell'applicazione, storage, garbage collection, buffer del socket |
| molti `CLOSE-WAIT` vecchi | il peer ha terminato, l'app locale non chiude | perdita di descrittori di file, percorso di codice bloccato, connection pool |
| le connessioni falliscono solo sotto carico | limite di stato o capacità | listen backlog, SYN cookies, NAT/conntrack, porte effimere, limite FD |
| i dati piccoli funzionano, quelli grandi si bloccano | errore di percorso dipendente dalla dimensione | MTU, filtro PMTUD/ICMP, MSS clamping, limiti di record TLS/proxy |

## Capacità e aree di guasto

Dopo la singola connessione segue la questione della capacità. Ogni socket occupa stato nel client e nel server, spesso anche in firewall, NAT, bilanciatore di carico o proxy. Il numero sostenibile di connessioni non è quindi determinato soltanto da CPU o larghezza di banda, ma dalla tabella, coda o quantità di porte più piccola lungo il percorso reale.

Gli amministratori dovrebbero distinguere almeno le seguenti risorse:

- **Listen backlog e SYN backlog:** limitano rispettivamente connessioni completamente e parzialmente aperte;
- **Descrittori di file e limiti di processo:** una porta libera non basta se l'applicazione non può più accettare socket;
- **Porte effimere:** relay e proxy in uscita possono raggiungere in parallelo lo stesso socket di destinazione solo con socket locali diversi;
- **Tabelle NAT e conntrack:** mantengono timer propri e possono esaurirsi nonostante risorse libere sui sistemi finali;
- **Memoria per socket:** buffer di invio/ricezione, stato TLS e contesto applicativo si moltiplicano con il numero di connessioni;
- **Distribuzione TIME-WAIT:** il lato che chiude attivamente sostiene questo stato; le decisioni architetturali spostano quindi il carico;
- **Affinità del bilanciatore di carico e percorso di ritorno:** i dispositivi stateful devono vedere entrambe le direzioni della stessa connessione.

Ridurre alla cieca i timer TIME-WAIT, keepalive o ritrasmissione raramente risolve la causa. Modifica la protezione del protocollo e la semantica degli errori. Occorre prima misurare tasso di connessioni, durata di mantenimento, distribuzione delle quadruple e la tabella effettivamente esaurita.

## Monitoraggio per ambienti di messaggistica

Da questi stati deriva il monitoraggio. Il solo utilizzo dell'interfaccia non rileva handshake lenti, ritrasmissioni, tempeste di reset e un destinatario con finestra chiusa. Una dashboard utile unisce quindi prospettive di host, rete e applicazione e affianca almeno le seguenti serie temporali:

| Metrica | Indicazione | Allarme solo in relazione a |
|---|---|---|
| tasso di successo delle connessioni e p95/p99 | raggiungibilità e tempo di stabilimento | destinazione, famiglia IP, porta e sede |
| nuove connessioni al secondo | carico e churn | connessioni stabilite e durata media di mantenimento |
| `SYN-SENT` / `SYN-RECEIVED` | connessioni semiaperte | età, backlog drop, SYN cookies |
| RST per direzione e fase di vita | interruzioni attive | prima/dopo handshake, nodo mittente, codice applicativo |
| tasso di ritrasmissione | segnale di perdita o riordinamento | RTT, drop dell'interfaccia, posizione della cattura |
| RTT e RTO | ritardo del percorso e comportamento dei timer | regione, controparte, ora del giorno |
| durata Zero Window | il destinatario non legge | CPU, storage, JVM/GC, thread pool |
| età di `CLOSE-WAIT` | l'applicazione locale non chiude | processo, release e connection pool |
| utilizzo NAT/conntrack | stato nel nodo intermedio | timeout, uso delle porte, percorso di ritorno |

I test sintetici delle porte dovrebbero essere abbinati a un test di protocollo. Per SMTP significa: TCP connect, banner, EHLO, StartTLS opzionale e una transazione controllata sono punti di misurazione separati. Solo così resta riconoscibile se siano il percorso di rete, TLS o l'applicazione di posta a causare la latenza.

## Sicurezza e hardening

TCP fornisce ordine e correzione degli errori, ma né riservatezza né identità crittografica. Le applicazioni aggiungono a questo scopo solitamente [TLS](/kb/tls), le reti eventualmente IPsec e le connessioni TCP speciali TCP-AO. I numeri di sequenza rendono più difficile l'iniezione alla cieca, ma non sostituiscono l'autenticazione. L'hardening deve inoltre considerare SYN flood, reset falsificati, code troppo grandi e listener abusabili ([RFC 9293, sezione 7](https://datatracker.ietf.org/doc/html/rfc9293#section-7)).

Per servizi di messaggistica esposti sono rilevanti i seguenti controlli:

- pubblicare solo i listener e le famiglie di indirizzi effettivamente necessari;
- modellare separatamente i ruoli server e client nei firewall, anziché definire genericamente «TCP consentito»;
- monitorare SYN backlog, SYN cookies e tassi di connessione prima che rate limit rigidi colpiscano server di posta legittimi;
- accettare Proxy Protocol solo tra peer esplicitamente affidabili;
- documentare la terminazione TCP e TLS, affinché IP sorgente, timeout e responsabilità RST restino tracciabili;
- proteggere catture di pacchetti e diagnostica dei socket come dati operativi potenzialmente sensibili;
- evitare manipolazioni middlebox non necessarie a MSS, opzioni e idle timer e testare ogni deviazione necessaria.

## Storia tecnica

La separazione oggi scontata tra IP e TCP nacque soltanto durante il primo sviluppo di Internet. [RFC 675](https://datatracker.ietf.org/doc/html/rfc675) descriveva nel 1974 ancora un unico Internet Transmission Control Program. La successiva suddivisione assegnò a IP il trasporto a commutazione di pacchetto tra reti e a TCP la sessione affidabile end-to-end. [RFC 793](https://datatracker.ietf.org/doc/html/rfc793) standardizzò questo TCP nel 1981.

| Anno | Sviluppo |
|---:|---|
| 1974 | RFC 675 descrive Internet Transmission Control Program come predecessore |
| 1981 | RFC 793 definisce TCP; RFC 791 descrive l'Internet Protocol separato |
| 1989 | RFC 1122 consolida i requisiti degli host e precisa numerose regole TCP |
| 1992 | RFC 1323 introduce finestre grandi e Timestamps per collegamenti ad alte prestazioni |
| 2009 | RFC 5681 consolida Slow Start, Congestion Avoidance, Fast Retransmit e Fast Recovery |
| 2011 | RFC 6298 standardizza il calcolo del Retransmission Timeout |
| 2014 | RFC 7323 sostituisce RFC 1323 e aggiorna Window Scale e Timestamps |
| 2022 | RFC 9293 sostituisce RFC 793 e riunisce nuovamente lo standard di base distribuito nel corso dei decenni |

Le implementazioni hanno continuato a evolversi parallelamente: Selective Acknowledgment, Explicit Congestion Notification, moderni algoritmi di controllo della congestione, TCP Fast Open e offloading del sistema operativo estendono il nucleo. Per la ricerca guasti, tuttavia, il modello originario di coppia di socket, spazio di sequenza, finestre, stati e timer resta valido.

## Fonti

- [RFC Editor – RFC 9293: Transmission Control Protocol](https://www.rfc-editor.org/info/rfc9293)
- [RFC 9293 – Transmission Control Protocol](https://datatracker.ietf.org/doc/html/rfc9293) – specifica TCP consolidata, header, stati, modello di sequenza, interfacce e sicurezza.
- [IANA – Service Name and Transport Protocol Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) – servizi registrati e intervalli di porte.
- [RFC 6298 – Computing TCP's Retransmission Timer](https://datatracker.ietf.org/doc/html/rfc6298) – algoritmo RTT, RTO e backoff.
- [RFC 5681 – TCP Congestion Control](https://datatracker.ietf.org/doc/html/rfc5681) – Slow Start, Congestion Avoidance, Fast Retransmit e Fast Recovery.
- [Wireshark User’s Guide – TCP Analysis](https://www.wireshark.org/docs/wsug_html_chunked/ChAdvTCPAnalysis.html) – derivazione di ritrasmissione, Zero Window, Out-of-Order e ulteriori segnali di analisi.
- [RFC 7323 – TCP Extensions for High Performance](https://datatracker.ietf.org/doc/html/rfc7323) – Window Scale, Timestamps, RTTM e PAWS.
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) – diagnostica di porta TCP, risoluzione dei nomi e route in Windows.
- [OpenBSD Manual – nc(1)](https://man.openbsd.org/nc.1) – connessioni TCP/UDP, listener e test delle porte con Netcat.
- [OpenBSD Manual – telnet(1)](https://man.openbsd.org/telnet.1) – client Telnet, stabilimento della connessione e comportamento del protocollo.
- [iproute2 – ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html) – stati dei socket, timer, code e filtri.
- [Microsoft Learn – Pktmon](https://learn.microsoft.com/windows-server/networking/technologies/pktmon/pktmon) – monitor dei pacchetti Windows e conversione PCAPNG.
- [tcpdump(1) Manual](https://www.tcpdump.org/manpages/tcpdump.1.html) – opzioni di cattura, filtri e file di output.
- [RFC 675 – Specification of Internet Transmission Control Program](https://datatracker.ietf.org/doc/html/rfc675) – predecessore tecnico di TCP e IP.
- [RFC 793 – Transmission Control Protocol](https://datatracker.ietf.org/doc/html/rfc793) – specifica TCP originaria del 1981.
