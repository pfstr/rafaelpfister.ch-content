---
title: "NatJack: spiegazione degli attacchi alle tabelle NAT, chi è interessato e quali misure sono necessarie"
navTitle: "NatJack spiegato"
description: "NatJack è una nuova classe di attacchi contro dispositivi NAT: un aggressore dietro lo stesso router, firewall o bridge Docker può rilevare connessioni TCP altrui, intercettare risposte DNS o paralizzare il NAT. Come funziona l'attacco, quali prodotti sono interessati, quando sono necessarie misure e quali sono efficaci."
date: "2026-10-28"
kategorie: "Vulnerabilità di sicurezza"
timeToRead: "10 min di lettura"
themen:
  - sicherheitsluecken
produkte:
  - "sicherheitsluecken"
protokolle:
  - "tcp"
  - "dns"
  - "container"
  - "haertung"
hauptthema: "sicherheitsluecken"
slug: "natjack-spiegazione-degli-attacchi-alle-tabelle-nat-chi-e-interessato-e-quali-misure-sono"
translationId: "article-223ba44d60e8cbc8"
translationOf: natjack-nat-tabellen-angriff
url: https://rafaelpfister.ch/it/blog/natjack-spiegazione-degli-attacchi-alle-tabelle-nat-chi-e-interessato-e-quali-misure-sono
translationSourceHash: c29a779430bbe591ef42a7db22926ad80fea50613dbc60458b9e88e40edadc02
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T21:30:57.788Z
translationReview: automatic
---

# NatJack: spiegazione degli attacchi alle tabelle NAT, chi è interessato e quali misure sono necessarie

NatJack è una classe di attacchi contro dispositivi che eseguono Network Address Translation (NAT): router, firewall, reti di hypervisor, bridge di container e gateway NAT cloud. Un aggressore che si trova dietro lo stesso dispositivo NAT della vittima manipola la relativa tabella NAT affinché le risposte da Internet non arrivino più alla vittima, bensì a lui. In questo modo è possibile rilevare connessioni TCP esistenti, intercettare e falsificare risposte DNS oppure bloccare il dispositivo NAT per tutti gli altri utenti. La ricerca è stata pubblicata da Malcolm Stagg (SODIUM-24, LLC) il 6 agosto 2026 alla Black Hat USA, coordinata tramite CERT/CC (VU#268870) con 13 produttori. In 23 prodotti testati sono stati segnalati 95 singoli risultati; secondo il white paper, ogni dispositivo testato con NAT era vulnerabile ad almeno una parte degli attacchi.

In breve: per una rete domestica con dispositivi propri non è necessario intervenire. Le misure sono necessarie dove utenti o workload estranei o non attendibili usano lo stesso dispositivo NAT dei sistemi produttivi: Wi-Fi ospiti sul firewall aziendale, host Docker e Kubernetes con immagini di terze parti, VM di test nella rete NAT di un host Hyper-V, ambienti di hosting e co-working. La misura più efficace è la crittografia (TLS anche internamente, DNS protetto), la seconda più efficace è separare le zone di fiducia su dispositivi NAT distinti.

<div class="notfall-hinweis">
<p class="notfall-hinweis__titel">Supporto nella valutazione</p>
<p>Se desidera verificare se il suo ambiente firewall, container o hypervisor è interessato, utilizzi il <a href="https://adeptio.ch/en">modulo di contatto su adeptio.ch</a>.</p>
</div>

## Cosa fa una tabella NAT

Dietro un dispositivo NAT, i dispositivi dispongono di indirizzi privati secondo RFC 1918, ad esempio `10.0.10.21`. Quando un dispositivo stabilisce una connessione verso l'esterno, il dispositivo NAT sostituisce indirizzo e porta del mittente con il proprio indirizzo esterno e una porta esterna e memorizza l'associazione nella tabella NAT. Quando arriva la risposta del server, il dispositivo consulta la tabella per determinare a quale dispositivo interno appartiene. In Linux se ne occupa Netfilter con il suo monitoraggio delle connessioni `conntrack`; la maggior parte dei router commerciali, OpenWRT, Docker e Kubernetes usa questo codice.

In origine il NAT era una soluzione transitoria alla scarsità di indirizzi IPv4 e non un confine di sicurezza. Nella pratica, tuttavia, lo è diventato: a livello 2, VLAN, isolamento delle porte, Dynamic ARP Inspection e isolamento vSwitch separano nettamente i dispositivi tra loro, e l'ARP spoofing è ormai quasi impossibile in reti ben gestite. A livello 3/4, però, le reti separate spesso confluiscono nuovamente nello stesso dispositivo NAT. È proprio qui che interviene NatJack.

![Dispositivo NAT condiviso con due VLAN separate: la voce NAT della vittima viene sostituita da una voce dell'aggressore, e le risposte del server di destinazione vengono quindi inviate all'aggressore](../images/natjack-nat-tabellen-angriff/geteiltes-nat.svg)

## Come funziona l'attacco

Tutte le varianti di NatJack seguono lo stesso schema: l'aggressore induce il dispositivo NAT a rimuovere la voce della tabella della vittima e crea immediatamente una propria voce con gli stessi parametri esterni. Il dispositivo NAT inoltra quindi all'aggressore la risposta successiva del server. Né il server né la vittima ricevono un messaggio di errore.

Ciò è reso possibile da comportamenti progettati per una rete cooperativa e sensati in tale contesto:

- **Loose mode:** per impostazione predefinita, Netfilter inserisce nella tabella anche connessioni TCP di cui non ha visto l'handshake. Questo serve a mantenere le connessioni dopo il riavvio di un router o un cambio di percorso, ma consente anche di creare voci senza un valido avvio della connessione.
- **Verifica permissiva dei pacchetti di reset:** il dispositivo NAT verifica i numeri di sequenza meno rigorosamente degli endpoint stessi, per non scartare pacchetti validi. Pacchetti che un endpoint ignorerebbe possono ancora essere interpretati dal dispositivo NAT come fine della connessione.
- **Port Preservation e Endpoint-Independent Mapping:** quando possibile, Netfilter utilizza la porta interna come porta esterna e continua a usare un'associazione già assegnata per tutte le destinazioni. Dal comportamento in caso di conflitti di porta si può dedurre quali porte siano attualmente usate da altri dispositivi.
- **Tabella finita:** se la tabella è piena, Netfilter scarta casualmente voci non ancora confermate, ad esempio richieste DNS aperte. Le connessioni confermate rimangono fino a cinque giorni.

Per la maggior parte delle varianti, l'aggressore deve poter inviare pacchetti con indirizzo del mittente falsificato. Se aggressore e vittima si trovano nella stessa sottorete, gli basta un dispositivo o un container con diritti di raw socket (`NET_RAW`). La variante upstream funziona anche oltre i confini di sottorete e VLAN; a tal fine l'aggressore necessita inoltre di un proprio server su Internet in grado di inviare pacchetti con indirizzo del mittente falsificato. Il white paper descrive le sequenze di pacchetti e il codice proof of concept; il Technical Report annunciato con framework di test non è ancora stato pubblicato.

## Panoramica delle cinque vulnerabilità

Il white paper descrive cinque vulnerabilità. Le valutazioni sono valori tipici; possono variare a seconda del prodotto.

| Vulnerabilità | Impatto | CVSS 3.1 (tipico) |
|---|---|---|
| Denial of Service | L'aggressore riempie la tabella NAT con voci confermate. Nessun altro dispositivo può stabilire nuove connessioni, con Netfilter fino a cinque giorni o fino al riavvio. | 7.4 (High) |
| Divulgazione di porta e IP | L'aggressore rileva che un altro dispositivo è connesso a un determinato server e ne determina la porta e l'indirizzo interno. Serve soprattutto come fase preliminare per gli altri attacchi. | 4.7 (Medium) |
| DNS hijacking tramite UDP | L'aggressore intercetta con una certa probabilità la risposta DNS destinata a un altro dispositivo e invia una risposta falsificata, ad esempio con l'indirizzo di un proprio server. | 10.0 (Critical) |
| TCP hijacking (downstream) | L'aggressore rileva una connessione TCP esistente di un dispositivo nella stessa sottorete, legge le risposte del server e invia dati a nome della vittima. | 9.6 (Critical) |
| TCP hijacking (upstream) | Come sopra, ma oltre i confini di sottorete e VLAN con l'aiuto di un server esterno dell'aggressore. | 10.0 (Critical) |

Sono interessati tutti i protocolli senza crittografia robusta: HTTP, DNS tramite UDP, protocolli di amministrazione basati su Telnet nonché protocolli di posta come SMTP e IMAP, se utilizzati senza TLS. Con una connessione TLS l'aggressore può sì interrompere la connessione, ma non ottiene dati leggibili e non può iniettare nulla. Il white paper sottolinea inoltre che i pacchetti di attacco possono essere inviati con TTL ridotto in modo che raggiungano il dispositivo NAT ma non proseguano nella rete. Ciò rende più difficile il rilevamento.

Rispetto all'ARP spoofing, NatJack ha un effetto simile, ma aggira i meccanismi di protezione consolidati a livello 2. Lavori precedenti di Yang et al. (CVE-2023-30305 fino a CVE-2023-30314) hanno mostrato il TCP hijacking tramite tabelle NAT su router Wi-Fi con opzioni non standard non sicure; NatJack funziona nella configurazione standard.

## Chi è interessato

Secondo il white paper, sono vulnerabili praticamente tutti i router basati su Linux, così come sistemi con una base di codice completamente diversa. La seguente panoramica riassume lo stato della matrice delle patch su natjack.io (ultimo aggiornamento: 19 agosto 2026).

| Area | Prodotti | Stato |
|---|---|---|
| Linux / Netfilter | Kernel Linux, OpenWRT, DD-WRT, quasi tutti i router basati su Linux | Correzione parziale con CVE-2026-63913 da Linux 7.1; aumenta la difficoltà, non offre protezione completa. OpenWRT e DD-WRT necessitano delle modifiche di Netfilter. |
| BSD | FreeBSD pf, OpenBSD pf | FreeBSD da 15.0.0 fortemente mitigato da una modifica proveniente da OpenBSD, versioni precedenti interessate. OpenBSD è protetto in modo significativamente migliore dal 2016. |
| Windows | Windows NAT (Hyper-V vSwitch con NAT) | CVE-2026-56181 (luglio 2026) e CVE-2026-56179 (agosto 2026), entrambe classificate come moderate. La mitigazione per CVE-2026-56179 è disattivata per impostazione predefinita e viene attivata tramite Registro di sistema. Windows Internet Connection Sharing è in fase di valutazione. |
| Dispositivi di rete | Cisco Firepower e Meraki, NETGEAR Nighthawk, Google Nest WiFi Pro, SpaceX Starlink, Amazon eero | A seconda del produttore: protezione parziale, raccomandazioni di configurazione o nessun aggiornamento. eero ha pubblicato una propria comunicazione. |
| Hotspot | Hotspot personale iOS, Condivisione Internet macOS, hotspot Android | interessati |
| Virtualizzazione e container | Hyper-V, KVM/QEMU, Apple Virtualization Framework, Docker (bridge), FreeBSD Jails, GKE, MicroK8s con Calico o Flannel, Azure AKS | interessati; protezione soprattutto tramite configurazione (vedi sotto) |
| Cloud | AWS NAT Gateway e NLB, Google Cloud NAT, Cloud Run, Azure Firewall, Azure NAT Gateway, Azure Functions | AWS ha rafforzato il servizio lato server, i clienti non devono fare nulla. AWS Lambda è protetto per l'assenza di `NET_RAW`. Per gli altri servizi, verificare lo stato su natjack.io. |

## Sono necessarie misure?

È decisivo chi si trova dietro lo stesso dispositivo NAT; il fatto che il modello del router compaia nell'elenco è secondario. Ogni variante presuppone che l'aggressore controlli già un dispositivo, una VM o un container dietro il NAT. Dove si tratta solo di dispositivi propri e attendibili, un aggressore con tale accesso avrebbe comunque vie più semplici. Il white paper afferma quindi che la maggior parte degli utenti domestici non è seriamente interessata, mentre le conseguenze per aziende e fornitori di servizi sono sensibilmente più gravi.

![Tre livelli: nessuna urgenza per soli dispositivi propri, verifica e rafforzamento per reti ospiti separate sullo stesso firewall, misure necessarie per utenti o workload estranei dietro lo stesso NAT](../images/natjack-nat-tabellen-angriff/massnahmen-noetig.svg)

Nelle PMI svizzere, il livello intermedio è il caso più comune: il Wi-Fi per gli ospiti è separato tramite VLAN, ma accede a Internet attraverso lo stesso firewall della rete aziendale. La separazione VLAN impedisce ARP spoofing e accesso diretto, ma entrambe le reti condividono comunque la tabella NAT. Per la variante downstream, l'aggressore dovrebbe trovarsi nella stessa sottorete della vittima; la variante upstream funziona oltre il confine VLAN, ma è più complessa e richiede un server esterno. Le misure sono chiaramente necessarie laddove estranei si trovano nella stessa sottorete (Wi-Fi di hotel, studi medici o scuole senza client isolation) e su host di container e Kubernetes in cui vengono eseguiti workload con diversi livelli di fiducia. Un pacchetto npm o PyPI compromesso in un container è sufficiente in questi casi come punto di partenza.

Per i servizi di posta, vale la pena esaminare specificamente le connessioni interne senza TLS: scanner, applicazioni specialistiche e sistemi di monitoraggio che consegnano senza crittografia a un relay interno tramite porta 25, nonché IMAP senza TLS. L'articolo [Invio di posta tramite un relay: verificare TLS e autenticazione](/blog/mail-relay-tls-authentisierung-pruefen) descrive come verificare TLS e autenticazione sul relay.

## Misure

Le raccomandazioni del white paper possono essere organizzate in cinque livelli. Differiscono per le vulnerabilità contro cui aiutano.

![Cinque livelli di protezione: crittografia, separazione delle zone di fiducia, anti-spoofing, rafforzamento NAT e monitoraggio, ciascuno con il proprio effetto su hijacking, divulgazione delle porte e Denial of Service](../images/natjack-nat-tabellen-angriff/schutzschichten.svg)

### Crittografia

TLS su tutte le connessioni, anche tra sistemi interni, rende inutili le connessioni TCP rilevate: l'aggressore può interromperle, ma non può né leggerle né iniettare dati. Per il DNS, il white paper raccomanda DNSSEC contro risposte falsificate e DNS over HTTPS o DNS over TLS contro l'intercettazione. La crittografia non aiuta contro la divulgazione delle porte e il Denial of Service.

### Separazione delle zone di fiducia

Utilizzate dispositivi e workload non attendibili in sottoreti e VLAN dedicate, con ACL L3/L4 che impediscano ogni accesso agli host attendibili. Ancora più efficace è un dispositivo NAT o un indirizzo esterno dedicato per ogni zona di fiducia, poiché senza una tabella NAT condivisa manca la base per tutte le varianti. Nel cloud ciò significa non eseguire workload attendibili e di terze parti dietro lo stesso NAT Gateway e utilizzare indirizzi IP dedicati anziché condivisi per i servizi serverless. In Kubernetes, i pod non attendibili vengono eseguiti su nodi dedicati.

### Anti-spoofing

Attivate IP Source Guard o una funzione equivalente sullo switch o sull'access point e uRPF (Reverse Path Filtering) sul router, affinché i client possano inviare solo con il proprio indirizzo. Nei container, la rimozione della capability `NET_RAW` impedisce a un processo compromesso di generare pacchetti falsificati. Docker assegna questa capability per impostazione predefinita.

```bash
docker run --cap-drop=NET_RAW --user 1000:1000 example/app:1.4
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `run` | Avvia un nuovo container da un'immagine |
| `--cap-drop=NET_RAW` | Revoca al container il diritto ai raw socket; non è quindi più possibile generare pacchetti IP falsificati |
| `--user 1000:1000` | Esegue il processo con ID utente e gruppo 1000 anziché come root |
| `example/app:1.4` | Immagine e tag del container (esempio) |

</details>

`ping` richiede `NET_RAW` in alcune immagini; sui kernel attuali funziona generalmente anche senza, tramite socket ICMP non privilegiati. In Kubernetes se ne occupa il `securityContext` del container:

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - NET_RAW
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `runAsNonRoot: true` | Kubernetes non avvia il container se verrebbe eseguito come root |
| `allowPrivilegeEscalation: false` | Impedisce al processo di acquisire diritti aggiuntivi, ad esempio tramite programmi setuid |
| `capabilities.drop: [NET_RAW]` | Revoca la capability dei raw socket, come `--cap-drop=NET_RAW` in Docker |

</details>

Il Pod Security Standard `restricted` richiede che i container rinuncino a tutte le capability (`drop: [ALL]`), coprendo così `NET_RAW`.

### Rafforzamento del NAT

Installate gli aggiornamenti disponibili: kernel o firmware con la correzione Netfilter, gli aggiornamenti Windows di luglio e agosto 2026 sugli host Hyper-V con rete NAT e FreeBSD dalla versione 15.0.0. Il changelog del pacchetto kernel indica se il kernel della distribuzione contiene la correzione per CVE-2026-63913.

Su router Linux e host di container è inoltre possibile disattivare il loose mode. Netfilter accetterà quindi nella tabella solo connessioni TCP con handshake completo.

```bash
sysctl net.netfilter.nf_conntrack_tcp_loose
sudo sysctl -w net.netfilter.nf_conntrack_tcp_loose=0
echo "net.netfilter.nf_conntrack_tcp_loose = 0" | sudo tee /etc/sysctl.d/90-natjack.conf
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `sysctl <schlüssel>` | Mostra il valore attuale; `1` significa loose mode attivo (impostazione predefinita) |
| `-w` | Scrive immediatamente il valore nel kernel in esecuzione, senza riavvio |
| `nf_conntrack_tcp_loose=0` | Disattiva l'accettazione di connessioni senza handshake osservato |
| `tee /etc/sysctl.d/90-natjack.conf` | Salva l'impostazione in modo permanente affinché persista al riavvio |

</details>

Il loose mode esiste per un motivo: dopo il riavvio del router o un failover in un cluster con `conntrackd`, senza di esso le connessioni esistenti si interrompono. Verificatelo prima della modifica, soprattutto con firewall ad alta disponibilità. Il white paper sottolinea espressamente che diverse misure richiedono di bilanciare sicurezza, disponibilità e compatibilità dei protocolli.

Altre impostazioni, se disponibili nel prodotto:

- **Limite di connessioni per client:** il white paper raccomanda un limite massimo di circa 10 000 connessioni simultanee per client. Nei firewall basati su pf, come OPNsense e pfSense, l'impostazione si trova nelle opzioni avanzate della regola firewall (pf: `max-src-states`).
- **Disattivare Port Preservation e Endpoint-Independent Mapping:** rende più difficile la determinazione delle porte. Con pf, una regola NAT senza `static-port` sostituisce casualmente la porta sorgente. Tenete presente che alcune applicazioni VoIP e peer-to-peer dipendono da associazioni fisse.
- **Randomizzazione ISN:** in pf tramite `modulate state` invece di `keep state`. Evitate regole con `flags any`, perché consentono connessioni senza handshake valido.

### Monitoraggio

Il white paper indica diversi segnali di tentativi di attacco: una tabella NAT piena o quasi piena, flussi di pacchetti da un client su un ampio intervallo di porte sorgente, lo stesso indirizzo IP associato a due indirizzi MAC o un indirizzo MAC con più indirizzi IP, flussi di pacchetti RST con numeri di sequenza variabili, numeri di sequenza molto al di fuori della finestra TCP e pacchetti con TTL insolitamente basso. In Linux, il livello di riempimento della tabella è indicato dal confronto fra due valori:

```bash
cat /proc/sys/net/netfilter/nf_conntrack_count /proc/sys/net/netfilter/nf_conntrack_max
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `nf_conntrack_count` | Numero di voci attualmente occupate nella tabella |
| `nf_conntrack_max` | Dimensione massima della tabella; dipende dalla memoria disponibile (con 4 GB o più: 262 144 voci) |

</details>

Se il primo valore rimane stabilmente vicino al secondo, è un motivo per indagare, indipendentemente da NatJack. Molti firewall visualizzano lo stesso indicatore come "State Table" o "Sessions" nella dashboard e possono generare un avviso al superamento di una soglia.

## Inquadramento

NatJack sfrutta assunzioni presenti da decenni nelle implementazioni NAT, non un singolo errore di programmazione. Per questo non esiste una patch che elimini completamente questa classe di attacchi: Linux e Windows hanno aumentato la difficoltà, AWS ha apportato miglioramenti lato server, molti dispositivi restano senza aggiornamento. Le misure durature risiedono nella propria architettura: protocolli crittografati anche nella rete interna e nessun dispositivo NAT condiviso per sistemi cui si attribuiscono livelli di fiducia diversi. Chi ha implementato entrambi può affrontare NatJack con il normale ritmo di patch.

## Fonti

1.  [NatJack (natjack.io)](https://natjack.io/): pagina del progetto del ricercatore con matrice delle patch per prodotto, elenco CVE e raccomandazioni; stato della matrice al 19 agosto 2026.

2.  [NatJack White Paper 1.0 (PDF)](https://natjack.io/natjack_white_paper_1.0.pdf): Malcolm Stagg, SODIUM-24, LLC, agosto 2026; descrizione tecnica delle cinque vulnerabilità, valutazioni CVSS, mitigazioni e indicatori.

3.  [Black Hat USA 2026: Breaking Trust Boundaries](https://blackhat.com/us-26/briefings/schedule/#breaking-trust-boundaries-exploiting-design-assumptions-in-network-infrastructure-53311): presentazione del 6 agosto 2026, con cui la classe di attacchi è stata resa pubblica.

4.  [Microsoft: CVE-2026-56181](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-56181): Windows NAT, variante downstream con Hyper-V; aggiornamento luglio 2026.

5.  [Microsoft: CVE-2026-56179](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-56179): Windows NAT, variante upstream con Hyper-V; aggiornamento agosto 2026, mitigazione da attivare tramite Registro di sistema.

6.  [Kernel Linux: commit Netfilter per CVE-2026-63913](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/net/netfilter/nf_conntrack_proto_tcp.c?id=2bb6d82b586ea5a4cb73bbdd6b7432e96096bc77): mitigazione parziale nel tracking TCP di Netfilter.

7.  [FreeBSD: commit pf](https://github.com/freebsd/freebsd-src/commit/9485d22286603ef2ac2e2009dd7dfcc6a59e2283): modifica adottata da OpenBSD, che in FreeBSD 15.0.0 rende molto più difficili la maggior parte delle varianti.

8.  [eero: Protecting eero customers from NatJack](https://eero.com/blog/articles/protecting-eero-customers-from-natjack): comunicazione del produttore sui propri dispositivi.

9.  [AWS: Troubleshoot NAT gateways](https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-troubleshooting.html): indicazioni sull'utilizzo delle porte dopo il rafforzamento lato server, a cui AWS rimanda nella propria dichiarazione.

10.  [Documentazione del kernel Linux: Netfilter Conntrack Sysctl](https://docs.kernel.org/networking/nf_conntrack-sysctl.html): descrizione di `nf_conntrack_tcp_loose`, `nf_conntrack_max` e degli altri parametri Conntrack.
