---
title: "Troubleshooting: evidenze, dipendenze e diagnosi controllata"
blatt: "troubleshooting"
description: "Diagnosi sistematica per amministratori di messaggistica e infrastruttura: ambito, timeline delle evidenze, grafi delle dipendenze, stati dei protocolli, log, metriche, trace, acquisizione dei pacchetti, test delle ipotesi, stabilizzazione e risoluzione delle cause."
fakten:
  - label: Metodo di base
    wert: Osservazione + modello di sistema → ipotesi → test discriminante → evidenza
    href: https://sre.google/sre-book/effective-troubleshooting/
  - label: Primo set di dati
    wert: atteso · effettivo · riproducibile · finestra temporale · ambito · impatto
    href: https://sre.google/sre-book/effective-troubleshooting/
  - label: Tre prospettive diagnostiche
    wert: grafo delle dipendenze · stato del protocollo · timeline comune
    href: https://opentelemetry.io/docs/concepts/signals/
  - label: Classi di evidenze
    wert: metriche · log · trace · interrogazioni di stato · acquisizione dei pacchetti
    href: https://sre.google/workbook/monitoring/
  - label: Correlazione importante
    wert: Message-ID, Queue-ID, Trace-ID, Session-ID e host sempre con timestamp
    href: https://www.w3.org/TR/trace-context/
  - label: Triage
    wert: limitare l'impatto e preservare le evidenze; l'analisi della causa può seguire
    href: https://sre.google/sre-book/effective-troubleshooting/
  - label: Soglia di accettazione della posta
    wert: 250 dopo End-of-Data trasferisce la responsabilità SMTP al destinatario
    href: https://datatracker.ietf.org/doc/html/rfc5321
  - label: Classi di errore SMTP
    wert: 4yz temporaneo · 5yz permanente; Enhanced Status Code fornisce la sottoclasse
    href: https://datatracker.ietf.org/doc/html/rfc3463
  - label: Test attivi
    wert: eseguirli dall'origine interessata, con identità, selezione della destinazione e policy identiche
    href: https://sre.google/sre-book/effective-troubleshooting/
  - label: Tempo
    wert: stato NTP, fuso orario e scostamento dell'orologio fanno parte di ogni evidenza distribuita
    href: https://datatracker.ietf.org/doc/html/rfc5905
  - label: Evidenza negativa
    wert: documenta quale ipotesi è stata confutata e in quali condizioni
    href: https://sre.google/sre-book/effective-troubleshooting/
  - label: Chiusura
    wert: verificare servizio, percorso dei dati e monitoraggio; documentare separatamente causa e prevenzione
    href: https://csrc.nist.gov/pubs/sp/800/61/r3/final
werbung:
  - tools
  - newsletter
ctaThemen:
  - troubleshooting
  - messaging
  - betrieb
translationSourceHash: 126551190a29508cedf6aa90a1863b0b8a005ef6d04862bac0feb8d9dca5a769
translationModel: gpt-5.6-terra
translatedAt: 2026-11-13T03:25:17.823Z
translationReview: automatic
---

# Troubleshooting: evidenze, dipendenze e diagnosi controllata

Il troubleshooting è l'acquisizione controllata di conoscenza su un sistema malfunzionante. Il punto di partenza non è la causa preferita più probabile, bensì una deviazione precisa tra comportamento atteso e comportamento osservato. Dalle osservazioni e da un modello tecnico nascono ipotesi; un test deve quindi distinguere almeno due possibili cause. Google SRE descrive questo approccio come applicazione del metodo ipotetico-deduttivo e sottolinea che una ricerca guasti efficace richiede sia una metodologia generale sia la conoscenza dell'architettura concreta del sistema ([Google SRE – Effective Troubleshooting](https://sre.google/sre-book/effective-troubleshooting/)).

Per i sistemi di messaggistica distribuiti, la formula «verificare dal basso verso l'alto» non è sufficiente. Un messaggio può essere stato accettato tramite DNS, TCP e TLS funzionanti e tuttavia restare bloccato in una coda, in un motore di policy, nel controllo dei contenuti, in un database o al prossimo hop. Al contrario, un errore utente in un'interfaccia web può essere causato da identità, stato della sessione, API, archivio della mailbox o ricerca. La diagnosi diventa affidabile quando l'amministratore combina tre modelli: **il grafo delle dipendenze dell'operazione interessata, gli automi a stati dei protocolli utilizzati e una timeline allineata tra tutti i componenti**.

La spiegazione inizia dal sintomo osservabile e porta, attraverso ambito, timeline ed evidenze, a un'ipotesi verificabile. Modifica, verifica e attività successive seguono solo quando causa ed effetto vengono tenuti distinti.

Il troubleshooting inizia con una deviazione osservabile: chi è interessato, da quando, su quale percorso e con quale risultato? Solo da questo ambito nasce un'ipotesi verificabile. L'articolo guida dalla delimitazione, attraverso evidenze e test, fino alla risoluzione verificata.

## Modello diagnostico: ambito, stato ed evidenze

Un buon report di problema contiene il comportamento atteso, il comportamento effettivo e, ove possibile, un'esecuzione riproducibile. Prima ancora dell'analisi tecnica viene determinato l'ambito: chi o cosa è interessato, da quando, in quale direzione, attraverso quale punto di ingresso e con quale impatto? Questo triage decide se occorre anzitutto analizzare un singolo caso, deviare il traffico, annullare una modifica errata o aprire un incidente più ampio. In caso di grave malfunzionamento, la stabilizzazione ha la priorità sull'analisi completa della causa; al contempo vanno preservati log volatili e altre evidenze ([Google SRE – Effective Troubleshooting](https://sre.google/sre-book/effective-troubleshooting/), [NIST SP 800-61 Rev. 3](https://csrc.nist.gov/pubs/sp/800/61/r3/final)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-troubleshooting.svg?v=20260813" title="Interaktive Infografik: evidenzbasierter Diagnosefluss vom Problembericht über Scope, Zeitachse, Abhängigkeiten, Protokollzustände und Signale bis zum kontrollierten Hypothesentest und verifizierten Abschluss" loading="lazy">
  <a href="/images/kb-interaktiv-troubleshooting.svg?v=20260813">Apri direttamente il grafico interattivo</a>.
</iframe>

### Matrice dell'ambito per il primo riscontro

La prima nota dell'amministratore non dovrebbe essere «la posta non funziona», ma descrivere una transazione riproducibile. Per la messaggistica e l'infrastruttura sono utili almeno i seguenti campi:

| Campo | Contenuto affidabile | Perché riduce lo spazio degli errori |
|---|---|---|
| Atteso / effettivo | «La destinazione SMTP deve restituire 250 dopo DATA; osservato 451 4.4.1» | separa l'aspettativa funzionale dal sintomo |
| Insieme interessato | singolo utente, dominio mittente, tenant, sede, protocollo o tutte le transazioni | distingue errori di oggetto, policy, percorso e piattaforma |
| Direzione e ingresso | inbound/outbound, relay/submission/access, GUI/API | identifica listener, policy e confine di responsabilità |
| Identità e autorizzazione | utente, Service Principal, certificato, IP sorgente, ruolo | rende verificabili autenticazione e autorizzazione |
| Periodo esatto | primo/ultimo errore in UTC, frequenza, durata, fuso orario | allinea log, metriche, trace e modifiche |
| Valori di correlazione | Message-ID, Queue-ID, Trace-ID, Session-ID, host e destinazione | collega gli eventi tra i componenti |
| Riproducibilità | deterministica/intermittente, input concreto, caso di confronto funzionante | consente test discriminanti |
| Impatto e workaround | perso, ritardato, duplicato, solo visualizzazione; aggiramento e rischio | guida triage e priorità |
| Ultime modifiche | deployment, configurazione, certificato, DNS, firewall, migrazione dati, profilo di carico | fornisce ipotesi, ma non ancora prova causale |

La formulazione di ogni voce separa **osservato**, **dedotto** e **sconosciuto**. «La connessione TCP termina dopo 21 secondi con timeout» è osservato. «Il firewall blocca» è un'ipotesi finché non sia dimostrata da un contatore di regole, un riscontro sui pacchetti o un reject amministrativo. Questa disciplina linguistica impedisce che supposizioni diventino fatti senza accorgersene.

## Grafo delle dipendenze anziché scala universale dei livelli

OSI o TCP/IP spiegano l'incapsulamento dei protocolli, ma rappresentano solo in parte le dipendenze operative. Per l'operazione concreta, un amministratore necessita di un grafo diretto: quale componente seleziona la destinazione, quale apre la connessione, quale verifica identità e policy, dove lo stato viene acquisito in modo persistente e quali servizi a valle possono impedire il completamento? Google SRE raccomanda di esaminare input e output noti ai confini di componenti ben definiti e di delimitare sistemi grandi in modo lineare o per dimezzamento ([Google SRE – Effective Troubleshooting](https://sre.google/sre-book/effective-troubleshooting/)).

Per un flusso di posta in uscita, il percorso può essere il seguente:

1. L'applicazione mittente genera envelope e contenuto del messaggio e li consegna a un endpoint Submission o Relay.
2. [DNS](/kb/dns) fornisce dati MX e di indirizzo; resolver, cache, suffissi di ricerca e Split DNS possono produrre risposte diverse.
3. [TCP](/kb/tcp) collega uno specifico indirizzo sorgente a uno specifico indirizzo di destinazione e a una porta.
4. [TLS](/kb/tls) negozia i parametri e verifica, a seconda della Transport Policy, nome, catena di certificati o altre ancore di attendibilità.
5. [SMTP](/kb/smtp) attraversa Greeting, EHLO, autenticazione opzionale, envelope e DATA. Una risposta `250` dopo il punto conclusivo trasferisce la responsabilità al server SMTP ricevente ([RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321)).
6. Il destinatario scrive il messaggio nella coda o nello store, applica regole di routing, malware, DLP, archiviazione e crittografia e tenta l'hop successivo.
7. Il servizio mailbox, l'indice, l'accesso client e le notifiche determinano se il messaggio accettato è visibile all'utente.

La diagnosi chiede a ogni collegamento una **transizione verificabile**. Una connessione TCP riuscita dimostra solo l'handshake TCP verso la destinazione testata. Non dimostra né l'attendibilità TLS né l'accettazione SMTP. Un `250` SMTP dopo End-of-Data dimostra la presa in carico da parte di questo hop, ma non il deposito nella mailbox finale. Un'estrazione riuscita dalla coda non dimostra che un indice di ricerca mostri già il messaggio.

Oltre al percorso diretto, il grafo include dipendenze trasversali: ora, risoluzione dei nomi, identità, certificati e chiavi, configurazione, database, archiviazione di oggetti o file, coda, Load Balancer, proxy, firewall, observability e fornitori esterni. Un errore in una di tali dipendenze può interessare contemporaneamente più componenti apparentemente indipendenti.

## Stati dei protocolli e confini di responsabilità

I protocolli offrono più valore diagnostico della semplice affermazione «porta aperta». Il riscontro rilevante è **quale stato è stato raggiunto, quale parte ha inviato per ultima un messaggio valido e quale risposta o timeout è seguito**.

- **DNS:** Query Name, tipo, resolver interrogato, RCODE, Answer/Authority, TTL e stato DNSSEC fanno parte dello stesso insieme. Il caching è parte del design DNS; per questo resolver diversi possono temporaneamente avere stati dei dati differenti ([RFC 1034](https://datatracker.ietf.org/doc/html/rfc1034)).
- **TCP:** Indirizzo e porta sorgente/destinazione, stato della connessione, ritrasmissioni e chiusura distinguono listener assente, Policy-Drop, Reject attivo e interruzione dopo l'instaurazione riuscita. RFC 9293 specifica stati ed elaborazione dei segmenti; un ICMP Echo non sostituisce il servizio TCP testato ([RFC 9293](https://datatracker.ietf.org/doc/html/rfc9293), [RFC 4443](https://datatracker.ietf.org/doc/html/rfc4443)).
- **TLS:** ClientHello, ServerHello, catena di certificati, nome, Trust Store, protocollo negoziato e Alert localizzano l'errore. TLS 1.3 definisce gli Alert come contenuto di protocollo a sé stante; «Handshake failure» è quindi uno stato, non semplicemente un sinonimo di errore di rete ([RFC 8446](https://datatracker.ietf.org/doc/html/rfc8446)).
- **SMTP:** Ogni risposta appartiene a un comando e a una fase della sessione. `4yz` indica una risposta negativa temporanea, `5yz` una permanente; gli Enhanced Status Codes suddividono lo stato in classe, soggetto e dettaglio ([RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321), [RFC 3463](https://datatracker.ietf.org/doc/html/rfc3463)).
- **LDAP:** Un'instaurazione TCP/TLS riuscita non dice nulla su Bind, Search Base, Filter, Controls o diritti di accesso. LDAPResult trasmette `resultCode`, `matchedDN`, messaggio diagnostico e Referrals opzionali ([RFC 4511](https://datatracker.ietf.org/doc/html/rfc4511)).
- **HTTP:** Codice di stato, metodo, URI di destinazione, hop proxy/gateway, header e Response Body costituiscono un riscontro. Un `403` non significa la stessa cosa di una connessione non riuscita; RFC 9110 definisce la semantica dei codici di stato ([RFC 9110](https://datatracker.ietf.org/doc/html/rfc9110)).
- **IMAP:** Le risposte taggate completano un comando; i dati non taggati possono segnalare modifiche di stato indipendentemente da ciò. Stati come not authenticated, authenticated e selected limitano i comandi consentiti ([RFC 9051](https://datatracker.ietf.org/doc/html/rfc9051)).

In questo modo, la domanda passa da «il server X funziona?» a «la transazione Y, con questi input e su questo percorso, raggiunge quale ultimo stato confermato?».

Quando dipendenze e fase del protocollo sono note, vengono scelti i segnali appropriati. Le metriche mostrano estensione e momento, i log le decisioni, i trace il percorso della chiamata e i dati dei pacchetti il dialogo di rete effettivo.

## Segnali: metriche, log, trace e dati dei pacchetti

Nessun singolo segnale rappresenta la verità dell'intero sistema. Le metriche condensano molti eventi in serie temporali, i log descrivono eventi discreti, i trace collegano operazioni lungo una richiesta, le interrogazioni di stato mostrano lo stato corrente e un'acquisizione dei pacchetti documenta il traffico di rete osservato in un punto di misurazione. Google SRE cita monitoring, log e tracing come strumenti diagnostici centrali; OpenTelemetry distingue tra l'altro Traces, Metrics e Logs come segnali di telemetria ([Google SRE – Monitoring](https://sre.google/workbook/monitoring/), [OpenTelemetry – Signals](https://opentelemetry.io/docs/concepts/signals/)).

| Segnale | Risponde particolarmente bene a | Limite tipico |
|---|---|---|
| Metrica | Quando è iniziata la deviazione, quanto è grande, quale popolazione è interessata? | L'aggregazione nasconde casi singoli; le label ad alta cardinalità sono costose |
| Log strutturato | Quale evento si è verificato in un componente e con quali campi? | contesto mancante, sampling, rotazione o orologio errato rendono difficile la correlazione |
| Trace | Quali hop e tempi ha avuto una singola richiesta? | funziona solo attraverso confini strumentati e che propagano il contesto |
| Interrogazione di stato | Qual è lo stato attuale di listener, coda, replica, certificato o configurazione? | può differire dal momento dell'errore e modificare lo stato |
| Acquisizione dei pacchetti | Quali pacchetti ha visto questo punto di misurazione dell'interfaccia/host? | nessuna visibilità sull'applicazione interna, sul contenuto cifrato o su un altro percorso |

Lo standard W3C Trace Context definisce `traceparent` e `tracestate`, affinché l'identità del trace possa essere propagata tra componenti HTTP. Non si tratta di un ID di posta universale. In una piattaforma di messaggistica è quindi necessario collegare esplicitamente Trace-ID, RFC-Message-ID, Queue-ID dei singoli MTA, Session-ID e host di destinazione ([W3C Trace Context](https://www.w3.org/TR/trace-context/)). Syslog standardizza un formato di evento con dati strutturati, ma da solo non garantisce né completezza né conservazione centralizzata; la gestione dei log comprende generazione, trasmissione, archiviazione, accesso e smaltimento ([RFC 5424](https://datatracker.ietf.org/doc/html/rfc5424), [NIST SP 800-92](https://csrc.nist.gov/pubs/sp/800/92/final)).

### La timeline comune

Una timeline non contiene soltanto il presunto momento dell'errore, ma almeno il primo e l'ultimo successo noti, il primo e l'ultimo errore, deployment e modifiche di configurazione rilevanti, cambi di certificato o token, failover, riavvii nonché la base temporale di ciascuna evidenza. NTP versione 4 serve per la sincronizzazione degli orologi di sistema; tuttavia gli amministratori devono verificare offset, stratum/fonte, stato leap/sync, fuso orario ed eventuali salti manuali ([RFC 5905](https://datatracker.ietf.org/doc/html/rfc5905)).

«Timestamp 10:14» non è un riferimento comune senza fuso orario, host e stato dell'orologio. Nelle code asincrone, inoltre, il tempo dell'evento di ricezione, elaborazione ed esportazione del log può divergere. Una nota robusta conserva il timestamp originale e aggiunge un'ora UTC normalizzata, invece di sovrascrivere il valore originale.

I segnali da soli non spiegano la causa. Una buona ipotesi prevede quale risultato deve verificarsi in un test mirato e in cosa differisce da un'alternativa.

## Ipotesi e test discriminanti

Un'ipotesi è utile quando un test a basso rischio può confutarla o renderla più probabile rispetto a un'alternativa. «Problema di rete» è troppo generico. «L'MTA non riesce a raggiungere sulla porta TCP 25 l'indirizzo IPv6 selezionato dal proprio resolver, mentre IPv4 funziona» indica origine, selezione della destinazione, protocollo e confronto.

Un piano di test definisce prima dell'esecuzione:

1. quali due o più ipotesi distingue il test;
2. da quale host, namespace, interfaccia e utente viene eseguito;
3. quale configurazione DNS, proxy, routing, identità e trust utilizza effettivamente;
4. quale risultato è atteso in ciascun caso;
5. quali effetti collaterali, accessi ai dati o carico vengono prodotti;
6. come viene annullata una modifica di test;
7. quali dati grezzi e timestamp vengono conservati.

Google SRE fa notare che test con sorgenti, policy firewall, credenziali o altri fattori perturbanti diversi possono trarre in inganno. Anche il test stesso può influire sul sistema: logging dettagliato modifica carico e timing; un riavvio cancella stato volatile; un retry può attivare un messaggio due volte. I test dovrebbero quindi essere eseguiti dall'origine interessata e, per quanto possibile, con la stessa selezione della destinazione, identità e policy del processo malfunzionante ([Google SRE – Effective Troubleshooting](https://sre.google/sre-book/effective-troubleshooting/)).

Il confronto con un caso funzionante è particolarmente efficace quando differisce esattamente una variabile indipendente: lo stesso utente su un altro host, un altro utente sullo stesso host, lo stesso messaggio verso un altro dominio o lo stesso host di destinazione tramite un'altra famiglia di indirizzi. Più modifiche di configurazione simultanee producono invece un nuovo stato del sistema sconosciuto.

## Strumenti di amministrazione lungo il percorso reale

I comandi seguenti sono esempi diagnostici, non Health Check universali. I segnaposto devono essere adattati all'origine interessata, al resolver, all'host, alla porta e al periodo. Credenziali, token e contenuti dei messaggi non devono finire senza redazione in ticket o nella Shell History.

### Registrare la risposta DNS e il resolver selezionato

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für DNS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName example.ch -Type MX -DnsOnly -Server 192.0.2.53
Resolve-DnsName mx1.example.ch -Type A -DnsOnly -Server 192.0.2.53
Resolve-DnsName mx1.example.ch -Type AAAA -DnsOnly -Server 192.0.2.53</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig @192.0.2.53 +noall +answer +authority example.ch MX
dig @192.0.2.53 +noall +answer mx1.example.ch A
dig @192.0.2.53 +noall +answer mx1.example.ch AAAA</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) e [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) rendono espliciti resolver e tipo di record. Il confronto con un server pubblico o autorevole può rendere visibili Split DNS e differenze di cache, ma non deve essere confuso con il resolver effettivamente usato dal servizio.

### Verificare connessione TCP e percorso dall'origine interessata

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für TCP- und Pfaddiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection mx1.example.ch -Port 25 -InformationLevel Detailed
tracert -d mx1.example.ch</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">nc -vz -w 5 mx1.example.ch 25
traceroute -n mx1.example.ch</code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) e [`nc`](https://man.openbsd.org/nc) testano l'instaurazione TCP; [`tracert`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/tracert) e [`traceroute`](https://man7.org/linux/man-pages/man8/traceroute.8.html) esaminano limiti di hop e riscontri lungo un percorso. Un asterisco nella rotta non prova un malfunzionamento del traffico utile e un test ICMP Echo riuscito non prova una porta TCP aperta. Lo storico [`ping`](https://ftp.arl.army.mil/~mike/ping.html) resta utile come sonda di latenza/raggiungibilità, ma non è nemmeno un test applicativo.

### Osservare TLS e dialogo applicativo

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für TLS- und Protokolldiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">openssl.exe s_client -connect mx1.example.ch:25 -starttls smtp -servername mx1.example.ch -showcerts
curl.exe --verbose --connect-timeout 5 https://api.example.ch/health</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">openssl s_client -connect mx1.example.ch:25 -starttls smtp -servername mx1.example.ch -showcerts
curl --verbose --connect-timeout 5 https://api.example.ch/health</code></pre>
  </div>
</div>

[`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) mostra handshake TLS, certificati e Alert; [`curl`](https://curl.se/docs/manpage.html) può parlare, tra gli altri, HTTP, SMTP, IMAP, POP3 e LDAP. Entrambi utilizzano configurazioni proprie di build e trust, che possono differire dal prodotto interessato. [`telnet`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/telnet) oppure la [pagina man OpenBSD di `telnet`](https://man.openbsd.org/telnet.1) è storicamente diffuso per dialoghi in chiaro, ma non fornisce verifica TLS; per semplici sonde TCP, `nc` è generalmente più chiaro.

### Rilevare listener locali e stati di connessione

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für lokale Socketdiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-NetTCPConnection -LocalPort 25,443 -ErrorAction SilentlyContinue |
  Sort-Object LocalPort,State,RemoteAddress |
  Format-Table LocalAddress,LocalPort,RemoteAddress,RemotePort,State,OwningProcess</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">ss -lntp '( sport = :25 or sport = :443 )'
ss -nto state syn-sent</code></pre>
  </div>
</div>

[`Get-NetTCPConnection`](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection) e [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) mostrano endpoint locali e stati TCP. Sono determinanti anche indirizzo di bind, namespace/container, processo proprietario e famiglia di indirizzi: un listener su `127.0.0.1` o in un altro Network Namespace non è automaticamente raggiungibile dall'esterno.

### Acquisizione dei pacchetti con filtro ristretto e finestra temporale

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Paketmitschnitt">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">pktmon filter remove
pktmon filter add Mailflow -t TCP SYN ACK RST FIN -p 25
pktmon start --capture --pkt-size 0 --file-name C:\\Temp\\mailflow.etl
pktmon stop
pktmon etl2pcap C:\\Temp\\mailflow.etl --out C:\\Temp\\mailflow.pcapng</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">sudo tcpdump -i any -nn -s 0 -w /tmp/mailflow.pcap \
  'host 192.0.2.25 and tcp port 25'</code></pre>
  </div>
</div>

[`pktmon`](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon) e [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) acquisiscono pacchetti nel punto di misurazione selezionato. Prima devono essere definiti destinazione, filtro, durata, spazio di archiviazione e protezione dei dati. Un'acquisizione può contenere credenziali, contenuti personali e materiale delle chiavi. Inoltre, «non presente nell'acquisizione» dimostra solo che il pacchetto non è stato visto in quel punto di misurazione con quel filtro.

### Salvare i log nell'esatto periodo dell'errore

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Logdiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$start = [datetime]'2026-08-08T10:00:00Z'
$end   = [datetime]'2026-08-08T10:15:00Z'
Get-WinEvent -FilterHashtable @{LogName='System'; StartTime=$start; EndTime=$end} |
  Select-Object TimeCreated,Id,ProviderName,LevelDisplayName,Message</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">journalctl --since '2026-08-08 10:00:00 UTC' \
  --until '2026-08-08 10:15:00 UTC' \
  --output short-iso-precise --no-pager</code></pre>
  </div>
</div>

[`Get-WinEvent`](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent) e [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) delimitano temporalmente la query. Il riscontro deve contenere Event-ID/nomi dei campi e ora grezza; un messaggio di errore tradotto liberamente è meno ricercabile del codice invariato. Prima di una rotazione o di un riavvio, i log rilevanti vengono esportati e ne viene documentata la provenienza.

### Verificare orologio e fonte di sincronizzazione

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Zeitdiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">w32tm /query /status
w32tm /query /source
Get-Date -AsUTC -Format o</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">chronyc tracking
chronyc sources -v
date --utc --iso-8601=ns</code></pre>
  </div>
</div>

[`w32tm`](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings#w32tm-command-line-tool) e [`chronyc`](https://chrony-project.org/doc/4.7/chronyc.html) mostrano fonte e stato di sincronizzazione; l'output UTC locale fornisce un riferimento immediatamente documentabile. Un orologio sincronizzato non garantisce ancora che un'applicazione scriva lo stesso timestamp o lo stesso fuso orario.

Nella messaggistica questo modello viene applicato lungo il messaggio: accettazione, policy, coda, hop successivo, mailbox e accesso client dispongono ciascuno di evidenze proprie.

## Diagnosi della messaggistica come catena di responsabilità

Per un messaggio, la domanda più importante non è «dove manca?», ma **quale componente ha assunto in modo dimostrabile la propria responsabilità e a chi l'ha trasferita?** Ogni hop SMTP possiede una sessione propria e solitamente un Queue-ID proprio. L'header RFC-Message-ID viene trasportato dal contenuto del messaggio, ma non è necessariamente univoco o affidabile; i Queue-ID sono locali e cambiano a ogni hop.

Un tracciamento affidabile documenta pertanto:

1. mittente e destinatario dell'envelope separatamente dagli indirizzi header visibili;
2. Message-ID, dimensione e un fingerprint conforme alla protezione dei dati o un oggetto di test;
3. sessione client/submission e il primo riscontro di accettazione `250`;
4. ogni associazione tra Queue-ID in ingresso e in uscita;
5. decisione di policy, quarantena, rewrite, split o bounce;
6. hop successivo selezionato, risposta DNS, IP sorgente/destinazione, risultato TLS e SMTP;
7. accettazione finale o errore permanente per destinatario;
8. riscontro di mailbox, indice e client, se l'accettazione del trasporto ha avuto successo.

Un `451 4.4.1` SMTP non è la stessa cosa di un generico «5xx». La prima cifra del codice SMTP e l'Enhanced Status Code devono essere mantenuti letteralmente. Gli errori temporanei conducono normalmente a coda e retry; quelli permanenti possono generare un NDR/DSN. La possibilità di rinviare un messaggio dipende dal fatto che sia già stata superata una soglia di accettazione. Ripetere alla cieca dopo una risposta persa può generare duplicati ([RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321), [RFC 3463](https://datatracker.ietf.org/doc/html/rfc3463)).

## Separare triage, risoluzione e analisi della causa

In caso di impatto elevato vanno distinti tre filoni di lavoro:

- **Stabilizzare:** deviare il traffico, limitare la funzione malfunzionante, proteggere la capacità, arrestare la corruzione dei dati o eseguire un rollback sicuro noto.
- **Preservare le evidenze:** salvare log, stato delle code, configurazione, metriche, trace, dati dei pacchetti, stato temporale e riferimenti alle modifiche prima che ruotino o scompaiano a seguito di un riavvio.
- **Diagnosticare:** prioritizzare le ipotesi e confutarle o sostenerle con test il meno rischiosi possibile.

Una risoluzione temporanea non dimostra automaticamente la causa. Se un riavvio pone fine al malfunzionamento, sono ancora possibili perdita di memoria, deadlock, pool di connessioni esauriti, cache danneggiata, timeout esterno e molte altre cause. Il riavvio va registrato come intervento con momento ed effetto; successivamente vengono pianificati prevenzione e ulteriori prove.

Anche «ultima modifica» è una fonte di ipotesi, non una prova di causalità. Una correlazione temporale può derivare da un fattore scatenante comune o dal caso. Google SRE mette esplicitamente in guardia da sintomi irrilevanti, teorie preferite improbabili e correlazioni spurie ([Google SRE – Effective Troubleshooting](https://sre.google/sre-book/effective-troubleshooting/)).

Una modifica conclude l'incidente solo quando il percorso originario funziona di nuovo, gli effetti collaterali sono esclusi e il monitoraggio conferma lo stato stabile.

## Verifica e criteri di chiusura

Un incidente non è concluso quando un singolo test ha successo. L'accettazione verifica l'operazione funzionale originaria, l'ambito interessato e le aree di guasto adiacenti:

- transazioni rappresentative in entrambe le direzioni rilevanti;
- crescita e riduzione della coda, tasso di retry e di errori;
- coerenza dei dati, replica, ricerca e vista client;
- autenticazione, autorizzazione, percorsi di certificati e chiavi;
- monitoraggio, allertamento e dashboard dal punto di vista dell'utente;
- nuova verifica dopo almeno un intervallo rilevante di timer, retry o cache;
- nessuna nuova lacuna nei dati, nessun duplicato inatteso e nessun degrado nascosto.

Il rapporto di chiusura separa sintomo, impatto, timeline, causa immediata, fattori contribuenti, ripristino e misure permanenti. Le evidenze negative vengono conservate: documentano quali ipotesi sono state confutate in quali condizioni ed evitano lavoro ripetuto. NIST SP 800-61 Rev. 3 integra la risposta agli incidenti in rilevamento, risposta, ripristino e miglioramento a livello organizzativo ([NIST SP 800-61 Rev. 3](https://csrc.nist.gov/pubs/sp/800/61/r3/final)).

## Evoluzione tecnica degli strumenti diagnostici

La diagnosi di rete iniziale rendeva direttamente visibili singole proprietà dei protocolli. Mike Muuss scrisse `ping` nel 1983 per 4.2a BSD sulla base di ICMP Echo; in seguito `traceroute` di Van Jacobson utilizzò limiti di hop IP in scadenza e riscontri ICMP per rendere visibili le stazioni del percorso ([Mike Muuss – The Story of the PING Program](https://ftp.arl.army.mil/~mike/ping.html)). Questi strumenti restano preziosi, ma rispondono solo a domande strettamente definite.

Con le applicazioni distribuite, l'esigenza diagnostica si è spostata dalla raggiungibilità dell'host alla correlazione delle transazioni. Syslog ha ricevuto un formato di messaggio standardizzato, il tracing distribuito ha collegato singole operazioni, W3C Trace Context ha standardizzato l'inoltro HTTP dell'identità di trace e OpenTelemetry ha riunito Traces, Metrics e Logs in un framework di observability indipendente dal fornitore ([RFC 5424](https://datatracker.ietf.org/doc/html/rfc5424), [W3C Trace Context](https://www.w3.org/TR/trace-context/), [OpenTelemetry – Signals](https://opentelemetry.io/docs/concepts/signals/)). Il nucleo metodologico è rimasto lo stesso: uno strumento è valido quanto la domanda precisa, il punto di misurazione e l'interpretazione documentata del suo risultato.

## Fonti

- [Google SRE – Effective Troubleshooting](https://sre.google/sre-book/effective-troubleshooting/)
- [NIST – SP 800-61 Rev. 3, Incident Response](https://csrc.nist.gov/pubs/sp/800/61/r3/final)
- [IETF RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321)
- [IETF RFC 1034 – Domain Names: Concepts and Facilities](https://datatracker.ietf.org/doc/html/rfc1034)
- [IETF RFC 9293 – Transmission Control Protocol](https://datatracker.ietf.org/doc/html/rfc9293)
- [IETF RFC 4443 – ICMPv6](https://datatracker.ietf.org/doc/html/rfc4443)
- [IETF RFC 8446 – TLS 1.3](https://datatracker.ietf.org/doc/html/rfc8446)
- [IETF RFC 3463 – Enhanced Mail System Status Codes](https://datatracker.ietf.org/doc/html/rfc3463)
- [IETF RFC 4511 – LDAP Protocol](https://datatracker.ietf.org/doc/html/rfc4511)
- [IETF RFC 9110 – HTTP Semantics](https://datatracker.ietf.org/doc/html/rfc9110)
- [IETF RFC 9051 – IMAP4rev2](https://datatracker.ietf.org/doc/html/rfc9051)
- [Google SRE Workbook – Monitoring](https://sre.google/workbook/monitoring/)
- [OpenTelemetry – Signals](https://opentelemetry.io/docs/concepts/signals/)
- [W3C – Trace Context](https://www.w3.org/TR/trace-context/)
- [IETF RFC 5424 – The Syslog Protocol](https://datatracker.ietf.org/doc/html/rfc5424)
- [NIST – SP 800-92, Log Management](https://csrc.nist.gov/pubs/sp/800/92/final)
- [IETF RFC 5905 – Network Time Protocol Version 4](https://datatracker.ietf.org/doc/html/rfc5905)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [BIND 9 – dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc](https://man.openbsd.org/nc)
- [Microsoft Learn – tracert](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/tracert)
- [Linux man-pages – traceroute](https://man7.org/linux/man-pages/man8/traceroute.8.html)
- [Mike Muuss – The Story of the PING Program](https://ftp.arl.army.mil/~mike/ping.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [curl – command line manpage](https://curl.se/docs/manpage.html)
- [Microsoft Learn – telnet](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/telnet)
- [OpenBSD – telnet](https://man.openbsd.org/telnet.1)
- [Microsoft Learn – Get-NetTCPConnection](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection)
- [Linux man-pages – ss](https://man7.org/linux/man-pages/man8/ss.8.html)
- [Microsoft Learn – Packet Monitor (pktmon)](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon)
- [tcpdump – manual page](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [Microsoft Learn – Get-WinEvent](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent)
- [systemd – journalctl](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
- [Microsoft Learn – Windows Time Service tools](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings)
- [chrony – chronyc](https://chrony-project.org/doc/4.7/chronyc.html)
