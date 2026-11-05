---
title: "Multipath TCP (MPTCP) su Linux: funzionamento, storia e verifica prima della disattivazione"
navTitle: "MPTCP su Linux"
description: "Multipath TCP distribuisce una connessione TCP su più percorsi di rete. Da Go 1.24, molti servizi Go su Linux sono in ascolto tramite MPTCP senza che nessuno lo abbia configurato. Con CVE-2026-98432 questo è diventato rilevante. L'articolo spiega il protocollo, la sua storia e come verificare se è possibile disattivare MPTCP."
date: "2026-11-02"
kategorie: "Kernel Linux"
timeToRead: "10 min di lettura"
themen:
  - linux-kernel
  - sicherheitsluecken
produkte:
  - "linux-kernel"
  - "sicherheitsluecken"
protokolle:
  - "tcp"
  - "haertung"
hauptthema: "linux-kernel"
slug: "multipath-tcp-mptcp-su-linux-funzionamento-storia-e-verifica-prima-della-disattivazione"
translationId: "article-1caab37cb0d7fce7"
translationOf: multipath-tcp-mptcp-erklaert
url: https://rafaelpfister.ch/it/blog/multipath-tcp-mptcp-su-linux-funzionamento-storia-e-verifica-prima-della-disattivazione
translationSourceHash: 0c0b43321d6db2e6da54028fa5339a41dcd664ca8d2264fec44309b32d7809b1
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:32:27.388Z
translationReview: automatic
---

Durante la verifica del mio server domestico per CVE-2026-98432, ho scoperto che diversi servizi erano in ascolto tramite Multipath TCP, tra cui Prometheus-node-exporter, Pi-hole-Exporter e `tailscaled`, tutti scritti in Go. Non avevo mai configurato nulla del genere. L'exploit root finora descritto pubblicamente per questa vulnerabilità richiede proprio un listener MPTCP.

In breve: MPTCP è incluso nel kernel ufficiale Linux dal kernel 5.6 ed è consentito per impostazione predefinita. Da Go 1.24, i programmi Go aprono automaticamente i propri listener come socket MPTCP. Se sia possibile disattivare MPTCP può essere chiarito con due comandi e alcuni contatori del kernel. Sulla maggior parte dei server nessun client utilizza il protocollo e, dopo la disattivazione, Go ripiega autonomamente sul normale TCP.

## Che cos'è Multipath TCP

Una normale connessione TCP è vincolata a un'esatta coppia di indirizzi: IP e porta sorgente, IP e porta di destinazione. Quando uno smartphone passa dal Wi-Fi alla rete mobile, l'IP sorgente cambia e la connessione si interrompe. Multipath TCP estende TCP in modo che una connessione possa utilizzare più percorsi contemporaneamente o in successione, ad esempio Wi-Fi e rete mobile.

Tecnicamente, una connessione MPTCP è composta da più `Subflows`. Ogni subflow è, per la rete, una normale connessione TCP attraverso un percorso. L'applicazione non se ne accorge: lavora con un unico socket e il kernel distribuisce i dati sui subflow. Affinché i dati arrivino nell'ordine corretto, MPTCP gestisce un proprio numero di sequenza a 64 bit a livello di connessione (Data Sequence Number), in aggiunta ai numeri di sequenza dei singoli subflow.

MPTCP viene negoziato nel normale handshake TCP. SYN, SYN/ACK e il primo ACK contengono l'opzione TCP `MP_CAPABLE` (Option Kind 30), con cui entrambe le parti dichiarano di parlare MPTCP e si scambiano le chiavi. Altri subflow si aggiungono in seguito alla connessione con l'opzione `MP_JOIN`. Se `MP_CAPABLE` manca nella risposta, perché la controparte non conosce MPTCP o un middlebox rimuove l'opzione, la connessione prosegue come normale TCP. Questo fallback fa parte dello standard ed è il motivo per cui i server compatibili con MPTCP funzionano con tutti i client TCP.

In Linux, MPTCP è opt-in per socket: un'applicazione deve creare esplicitamente il socket con il protocollo `IPPROTO_MPTCP` (valore 262). Un programma che non lo fa continua a utilizzare il normale TCP, anche se MPTCP è attivo nel kernel.

## La storia

MPTCP è nato nelle università ed è stato introdotto su larga scala da Apple. L'implementazione Linux è rimasta per oltre un decennio al di fuori del kernel ufficiale, prima di essere adottata in una nuova forma.

| Momento | Evento |
|---|---|
| Dal 2009 circa | Ricercatori dell'Université catholique de Louvain (UCLouvain) sviluppano la prima implementazione Linux come ramo del kernel separato (multipath-tcp.org). |
| Marzo 2011 | RFC 6182 descrive l'architettura di Multipath TCP (Informational). |
| Gennaio 2013 | RFC 6824 specifica MPTCP versione 0 come protocollo sperimentale. |
| Settembre 2013 | Esce iOS 7 con MPTCP, il primo impiego commerciale su larga scala. Apple usa MPTCP, tra l'altro, per Siri. |
| Giugno 2015 | KT (Korea Telecom) avvia un servizio commerciale che aggrega Wi-Fi e LTE tramite MPTCP e consente agli smartphone velocità fino a 1 Gbit/s; presentato all'IETF 93. |
| 2017 | iOS 11 rende MPTCP disponibile agli sviluppatori di app tramite `URLSession` (`multipathServiceType`). |
| Marzo 2020 | RFC 8684 definisce MPTCP versione 1 come Proposed Standard e sostituisce RFC 6824. |
| 29 marzo 2020 | Esce Linux 5.6 con un primo supporto MPTCP nel kernel ufficiale, riscritto e solo per la versione 1. I subflow aggiuntivi sono arrivati nelle versioni successive. |
| 2020 | 3GPP specifica in Release 16 ATSSS (Access Traffic Steering, Switching, Splitting) per il 5G; MPTCP vi distribuisce il traffico tra rete mobile e Wi-Fi. |
| Agosto 2023 | Go 1.21 introduce `SetMultipathTCP` per dialer e listener, disattivato per impostazione predefinita. |
| Febbraio 2025 | Go 1.24 attiva MPTCP per impostazione predefinita per i listener (`ListenConfig`), mentre i dialer rimangono su TCP. |
| Ottobre 2026 | CVE-2026-98432: use-after-free nel codice IPv4-TCP; l'exploit root mostrato richiede un listener MPTCP. |

Il ramo out-of-tree di UCLouvain supportava le versioni 0 e 1. Il kernel ufficiale conosce solo la versione 1; oggi il progetto raccomanda esclusivamente questa implementazione.

## Perché MPTCP è attivo su molti server

Coincidono due impostazioni predefinite. Il kernel consente i socket MPTCP per impostazione predefinita: l'impostazione sysctl `net.mptcp.enabled` è impostata su `1`. E i programmi Go dalla versione 1.24 creano ogni listener come socket MPTCP, a condizione che il kernel lo consenta. I servizi Go come gli exporter Prometheus o Tailscale sono quindi in ascolto tramite MPTCP non appena vengono compilati con Go 1.24 o successivo.

Per i client normali questo non cambia nulla: non inviano `MP_CAPABLE` e la connessione procede come TCP. La superficie di attacco aggiuntiva nasce tuttavia lato server, perché ogni client che invia anche `MP_CAPABLE` raggiunge il codice MPTCP nel kernel.

## CVE-2026-98432

CVE-2026-98432 è un use-after-free durante l'instaurazione di connessioni IPv4-TCP con opzioni IP (CVSS 3.1: 9.8). Secondo la voce CVE, Linux è interessato dalla versione 4.4. Il difetto stesso riguarda ogni listener TCP, non solo MPTCP. Tuttavia, per lo sfruttamento fino ai privilegi root è stata finora mostrata solo una catena tramite un listener MPTCP; secondo lilting.ch, Anthropic l'ha dimostrata in un ambiente di laboratorio il 27 ottobre 2026. Non è chiaro se i privilegi root siano raggiungibili anche senza MPTCP.

A monte, il difetto è corretto nei kernel stable 5.10.272, 5.15.223, 6.1.190, 6.6.159, 6.12.113, 6.18.56 e 7.2.10, nonché nella mainline 7.3 (commit «ipv4: free inet_opt and ireq_opt after an RCU grace period"). Il 30 ottobre 2026 Ubuntu indicava ancora le versioni LTS 22.04, 24.04 e 26.04 come «Vulnerable, work in progress"; per la 26.04 la correzione è prevista in `linux` 7.0.0-43.43. Le distribuzioni effettuano il backport delle correzioni; fa quindi fede la versione del pacchetto indicata nell'advisory della distribuzione, non la versione principale del kernel.

Disattivare MPTCP elimina il prerequisito dell'exploit mostrato. La vulnerabilità nel codice IPv4-TCP rimane finché il kernel aggiornato non viene installato e avviato.

## Verificare se MPTCP è attivo e utilizzato

### 1. Il kernel consente MPTCP?

```bash
sysctl net.mptcp.enabled
```

`1` significa che le applicazioni possono creare socket MPTCP. Se `sysctl` segnala che la chiave è sconosciuta, MPTCP non è compilato nel kernel e non è disponibile su questo host.

### 2. Quali servizi sono in ascolto tramite MPTCP?

```bash
sudo ss -Mlnp
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-M` | mostra solo i socket MPTCP |
| `-l` | mostra solo i socket in ascolto (listener) |
| `-n` | indirizzi e porte in formato numerico, senza risoluzione dei nomi |
| `-p` | mostra il processo associato al socket; richiede privilegi root per i processi di altri utenti |

</details>

Ogni riga è un listener MPTCP. La stessa porta compare inoltre con `ss -tln` come listener TCP; è normale, poiché il kernel mantiene internamente un socket TCP per ogni listener MPTCP. I nomi dei processi mostrano quali programmi sono interessati. Di norma si tratta di programmi Go.

### 3. Un client ha mai utilizzato MPTCP?

Il kernel conta gli handshake MPTCP dall'ultimo avvio. I contatori rilevanti sono:

```bash
nstat -asz |
  grep -E 'MPTcpExt(MPCapable|MPCurrEstab)'
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-a` | valori assoluti dall'avvio del sistema anziché la differenza rispetto all'ultima chiamata |
| `-s` | non aggiorna il file di cronologia di `nstat` |
| `-z` | visualizza anche i contatori con valore 0 |

</details>

| Contatore | Significato |
|---|---|
| `MPTcpExtMPCapableSYNRX` | SYN in entrata con `MP_CAPABLE` verso un listener MPTCP |
| `MPTcpExtMPCapableACKRX` | connessioni MPTCP in entrata il cui handshake è stato completato |
| `MPTcpExtMPCapableSYNTX` | SYN in uscita con `MP_CAPABLE`, ovvero connessioni che questo host voleva instaurare tramite MPTCP |
| `MPTcpExtMPCapableSYNACKRX` | connessioni in uscita in cui la controparte ha confermato MPTCP |
| `MPTcpExtMPCapableFallbackACK` | connessioni lato server che durante l'handshake sono ripiegate su TCP |
| `MPTcpExtMPCapableFallbackSYNACK` | connessioni lato client che sono ripiegate su TCP |
| `MPTcpExtMPCurrEstab` | connessioni MPTCP attualmente esistenti |

Se `MPCapableSYNRX` e `MPCapableACKRX` sono a 0 dopo un periodo di attività rappresentativo, nessun client ha utilizzato MPTCP. Se `MPCapableSYNTX` è maggiore di 0, un'applicazione sull'host stesso instaura connessioni MPTCP in uscita; è opportuno identificare tale applicazione prima della disattivazione. Le connessioni MPTCP esistenti sono visualizzate da `ss -Mani` con dettagli sui subflow.

I contatori si riferiscono all'ultimo riavvio. Un server avviato solo un'ora prima non fornisce quindi un'indicazione affidabile. Inoltre, il kernel conta per spazio dei nomi di rete, come descritto più avanti.

### 4. Verificare separatamente i container

`net.mptcp.enabled` si applica per spazio dei nomi di rete. Ogni container con una propria rete (bridge Docker, Podman, pod Kubernetes) ha un valore proprio e un nuovo namespace inizia con il valore predefinito `1`, indipendentemente dall'impostazione sull'host. Sul mio server domestico, dopo la disattivazione sull'host, tutti i container bridge restavano impostati su `1`; solo i container con `network_mode: host` adottavano `0`.

```bash
for c in $(docker ps -q); do
  pid=$(docker inspect -f '{{.State.Pid}}' "$c")
  name=$(docker inspect -f '{{.Name}}' "$c")
  echo "$name $(sudo nsenter -t "$pid" -n sysctl -n net.mptcp.enabled)"
done
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `docker ps -q` | ID di tutti i container in esecuzione |
| `docker inspect -f '{{.State.Pid}}'` | PID del processo principale nel container, tramite il quale `nsenter` individua il namespace |
| `nsenter -t <pid>` | entra nei namespace del processo specificato |
| `-n` | entra solo nel namespace di rete; il file system e gli strumenti restano quelli dell'host |
| `sysctl -n` | visualizza solo il valore, senza il nome della chiave |

</details>

`nsenter` ha il vantaggio che nel container non devono essere presenti né `sysctl` né `cat`; molte immagini (distroless, scratch) non contengono una shell. Allo stesso modo è possibile interrogare `ss -Mln` e i contatori di `nstat` nel namespace di un container.

## Quando è possibile disattivare MPTCP

La disattivazione non comporta problemi se i contatori del passaggio 3 non mostrano alcun utilizzo di MPTCP sull'host e nei container. I programmi Go in ascolto non perdono nulla: Go tenta il socket MPTCP, riceve un errore dal kernel e crea silenziosamente un socket TCP. Altri programmi che richiedono esplicitamente `IPPROTO_MPTCP` ricevono l'errore `ENOPROTOOPT`. Se poi ripiegano su TCP o si interrompono dipende dal software; testate tali servizi dopo la modifica.

È necessaria cautela nei seguenti casi:

- **Accesso ibrido e aggregazione di collegamenti:** router o proxy che aggregano DSL, fibra ottica e rete mobile tramite MPTCP (come OpenMPTCProuter o gateway Hybrid Access dei provider) dipendono da MPTCP.
- **Server per app mobili:** un'app iOS proprietaria con `multipathServiceType` beneficia del cambio di rete solo se il server parla MPTCP. I contatori `MPCapableACKRX` mostrano se sono presenti tali client.
- **MPTCP in uscita:** programmi avviati con `mptcpize` o eseguiti in Go con `GODEBUG=multipathtcp=1` o `3` instaurano MPTCP in uscita. Riconoscibile da `MPCapableSYNTX` maggiore di 0.

## Disattivare MPTCP

Esistono tre possibilità, a seconda dell'ambito desiderato.

### Per l'intero host

```bash
echo "net.mptcp.enabled = 0" |
  sudo tee /etc/sysctl.d/99-no-mptcp.conf
sudo sysctl --system
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `/etc/sysctl.d/99-no-mptcp.conf` | file di configurazione letto all'avvio; l'impostazione persiste dopo i riavvii |
| `sysctl --system` | ricarica tutti i file sotto `/etc/sysctl.d/` e gli altri percorsi standard, applicandoli immediatamente |

</details>

L'impostazione impedisce solo la creazione di nuovi socket MPTCP. I listener già aperti restano socket MPTCP finché il servizio non viene riavviato. La soluzione più semplice è riavviare l'host; in seguito `ss -Mln` deve restituire un elenco vuoto.

### Per i container

Nei container con una propria rete, impostate il valore per ciascun container. Docker consente sysctl nell'ambito `net.*`, purché il container non utilizzi la rete dell'host:

```yaml
services:
  app:
    sysctls:
      net.mptcp.enabled: 0
```

Nella chiamata diretta, ciò corrisponde a `docker run --sysctl net.mptcp.enabled=0 …`. La modifica ha effetto alla successiva ricreazione del container (`docker compose up -d`).

### Solo per singoli programmi Go

Invece di disattivare MPTCP nel kernel, è possibile reimpostare il comportamento standard di Go per singolo servizio. La variabile d'ambiente `GODEBUG=multipathtcp=0` disattiva MPTCP per listener e dialer. Per un servizio systemd, ciò avviene tramite un drop-in:

```bash
sudo systemctl edit node_exporter
```

```ini
[Service]
Environment=GODEBUG=multipathtcp=0
```

Se il servizio imposta già una variabile `GODEBUG`, le voci vengono combinate in un'unica variabile separata da virgole. Questa possibilità agisce solo sui programmi Go e solo se il codice non chiama autonomamente `SetMultipathTCP`; una chiamata esplicita nel programma ha la precedenza su `GODEBUG`. Per i propri progetti Go, la stessa impostazione è disponibile come `godebug multipathtcp=0` nel file `go.mod` o come `//go:debug multipathtcp=0` nel pacchetto principale.

## Verifica dopo la modifica

```bash
sysctl net.mptcp.enabled
ss -Mln
```

Sono previsti `0` e nessuna riga sotto l'intestazione della tabella di `ss`. I servizi continuano poi a essere in ascolto sulle stesse porte, visibili con `ss -tln`. È possibile annullare la modifica eliminando il file sotto `/etc/sysctl.d/` e riavviando l'host.

## Fonti

1.  [RFC 8684: TCP Extensions for Multipath Operation with Multiple Addresses](https://datatracker.ietf.org/doc/html/rfc8684): specifica di MPTCP versione 1 (marzo 2020) con subflow, `MP_CAPABLE`, Data Sequence Number e fallback.

2.  [RFC 6824](https://datatracker.ietf.org/doc/rfc6824/): MPTCP versione 0, sperimentale (gennaio 2013), sostituita da RFC 8684.

3.  [RFC 6182: Architectural Guidelines for Multipath TCP Development](https://datatracker.ietf.org/doc/rfc6182/): documento architetturale del marzo 2011.

4.  [Linux Kernel Documentation: Multipath TCP](https://docs.kernel.org/networking/mptcp.html): opt-in tramite `IPPROTO_MPTCP`, codici di errore nella creazione dei socket, Path Manager.

5.  [Linux Kernel Documentation: MPTCP Sysfs variables](https://docs.kernel.org/networking/mptcp-sysctl.html): significato e valore predefinito di `net.mptcp.enabled` e delle altre sysctl MPTCP.

6.  [mptcp.dev](https://www.mptcp.dev/): sito web del progetto upstream con FAQ, guida al debug (`ss -Mani`, `nstat`) e indicazioni su macOS e iOS.

7.  [KernelNewbies: Linux 5.6](https://kernelnewbies.org/Linux_5.6): panoramica del rilascio con il primo supporto MPTCP nel kernel ufficiale.

8.  [Go 1.21 Release Notes](https://go.dev/doc/go1.21): introduzione del supporto MPTCP nel pacchetto `net`, disattivato per impostazione predefinita, con fallback silenzioso su TCP.

9.  [Go 1.24 Release Notes](https://go.dev/doc/go1.24): MPTCP predefinito per `ListenConfig` su Linux.

10.  [Go: impostazioni GODEBUG](https://go.dev/doc/godebug): valori da 0 a 3 di `multipathtcp`, nonché `godebug` in `go.mod` e `//go:debug`.

11.  [Wikipedia: Multipath TCP](https://en.wikipedia.org/wiki/Multipath_TCP): panoramica di implementazioni e utilizzi (UCLouvain, Apple, KT, 3GPP ATSSS).

12.  [Ubuntu Security: CVE-2026-98432](https://ubuntu.com/security/CVE-2026-98432): stato dei kernel Ubuntu per release e versioni di correzione previste.

13.  [lilting.ch: CVE-2026-98432 Linux TCP Use-After-Free](https://lilting.ch/en/articles/linux-ipv4-tcp-options-uaf-mptcp-rce): valutazione della sfruttabilità, ruolo di MPTCP e versioni corrette dei kernel stable.
