---
title: "Eseguire Claude Code in sicurezza sul un VPS personale"
navTitle: "VPS per Claude"
description: "Un VPS Debian rafforzato mantiene le sessioni di Claude Code sempre raggiungibili. La guida copre account utente e chiavi SSH, firewall, igiene dei dati, tmux e accesso sicuro da iPhone."
date: "2026-07-21"
kategorie: "Claude"
timeToRead: "12 min di lettura"
themen:
  - claude
slug: "eseguire-claude-code-in-sicurezza-su-un-vps-personale"
translationOf: "claude-code-vps-debian-absichern"
translationId: article-f932e9e537d7704a
translationReview: automatic
translationSourceHash: 011d5e16cec877d14e68e11ff48caee9b6ee849ee6235c889676cfe64ae81628
translatedAt: 2026-09-04T08:44:10.413Z
url: https://rafaelpfister.ch/it/blog/eseguire-claude-code-in-sicurezza-su-un-vps-personale
translationModel: gpt-5.6-terra
---

Su un computer personale, una sessione di Claude Code termina involontariamente al più tardi quando il laptop va in sospensione o la connessione di rete si interrompe. Un VPS continua a funzionare ed è raggiungibile da più dispositivi. Al tempo stesso, resta permanentemente collegato a Internet pubblico e viene sottoposto a scansioni automatizzate già poco dopo l'avvio.

Questa guida combina entrambe le esigenze: Claude Code rimane disponibile in una sessione `tmux`, mentre il server Debian offre verso l'esterno solo una connessione SSH protetta da chiave. Il rafforzamento non è specifico di Claude ed è adatto anche ad altri server Linux raggiungibili pubblicamente.

## Perché un VPS può essere utile

Rispetto a un'installazione esclusivamente locale, il server offre tre vantaggi pratici:

- **Persistenza.** In una sessione `tmux`, Claude continua a funzionare anche se la connessione SSH viene interrotta. Un'attività che richiede dieci minuti o un'ora termina senza dover lasciare aperto il laptop.
- **Raggiungibilità.** La stessa sessione è accessibile dal desktop, dal laptop e dall'iPhone. Si avvia un'attività alla scrivania e se ne controlla il risultato mentre si è in giro.
- **Controllo dei dati.** Si decide autonomamente cosa risiede sul server. Nessun servizio di sincronizzazione, nessuna credenziale inclusa accidentalmente nel backup, a condizione di procedere con attenzione durante la migrazione (vedi sotto).

`tmux` è quindi una funzione esclusivamente di disponibilità e comodità, non una misura di sicurezza. Il vero lavoro è nella protezione.

## Situazione iniziale

La base è Debian 13 (Trixie), installato in versione minimale, senza desktop né servizi di rete aggiuntivi. Il provider mette a disposizione un firewall a monte, che opera indipendentemente dal sistema operativo. L'obiettivo è un server sul quale dall'esterno sia raggiungibile esclusivamente SSH, e anche questo solo con chiavi protette da passphrase.

## 1. Aggiornare il sistema

Subito dopo l'installazione, aggiornare l'intero stato dei pacchetti:

```bash
sudo apt update
sudo apt full-upgrade
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `update` | Rilegge gli elenchi dei pacchetti di tutte le Fonti configurate |
| `full-upgrade` | Aggiorna tutti i pacchetti e può anche installare nuovi pacchetti o rimuovere quelli esistenti |

</details>

A differenza di `upgrade`, `full-upgrade` risolve anche le dipendenze che richiedono pacchetti nuovi o rimossi. Su un sistema appena installato, è la procedura corretta per installare davvero tutti gli aggiornamenti di sicurezza disponibili. Riavviare una volta dopo gli aggiornamenti del kernel.

## 2. Utente personale invece di root

Lavorare come root è inutilmente rischioso: ogni errore di battitura ha effetto sull'intero sistema e l'accesso diretto come root è la prima cosa che tentano gli attacchi automatizzati. Creare quindi un utente dedicato (qui `claude`) con privilegi sudo per i casi in cui servono:

```bash
sudo adduser claude
sudo usermod -aG sudo claude
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-a` | Aggiunta: completa l'elenco dei gruppi dell'utente invece di sostituirlo; valido solo insieme a `-G` |
| `-G sudo` | Gruppo/i supplementare/i ai quali viene aggiunto l'utente |
| `claude` | L'utente interessato; con `adduser` il nome dell'account da creare |

</details>

Da questo momento tutta l'amministrazione avviene tramite `claude` e `sudo`, non più tramite l'accesso diretto come root.

## 3. Chiavi Ed25519 con passphrase, una per dispositivo

L'accesso deve avvenire esclusivamente tramite chiavi SSH, non tramite password. Ed25519 è lo standard attuale: breve, rapido e crittograficamente solido. È fondamentale che la chiave venga generata sul client, quindi sul PC e non sul server, e sia protetta da una passphrase. La passphrase è la seconda linea di difesa se la chiave privata dovesse mai finire nelle mani sbagliate.

Sul PC:

```bash
ssh-keygen -t ed25519 -C "pc-thinkpad"
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-t ed25519` | Tipo di chiave, qui il metodo ellittico Ed25519 |
| `-C "pc-thinkpad"` | Commento aggiunto alla chiave pubblica |

</details>

Il commento (`-C`) identifica il dispositivo. Questo risulta utile in seguito: per ogni dispositivo viene generata una chiave propria, una per il PC e una separata per l'iPhone. Se un dispositivo viene perso, si rimuove selettivamente la relativa chiave pubblica da `~/.ssh/authorized_keys`, senza dover ridistribuire tutti gli altri accessi.

Sul server deve essere presente solo la chiave pubblica. La chiave privata non lascia mai il dispositivo. In `authorized_keys` alla fine sono presenti esclusivamente chiavi pubbliche, ciascuna con il commento del proprio dispositivo:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...pc  pc-thinkpad
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...ios iphone-15
```

Trasferire inizialmente la chiave pubblica del PC. Finché l'accesso tramite password è ancora attivo, il modo più semplice è:

```bash
ssh-copy-id claude@SERVER
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `claude@SERVER` | Utente e host di destinazione; la chiave pubblica predefinita viene aggiunta lì a `~/.ssh/authorized_keys` |

</details>

Verificare poi che l'accesso con chiave funzioni prima di disattivare l'accesso tramite password nel passaggio successivo. I permessi dei file devono essere corretti, altrimenti sshd ignora il file: `~/.ssh` a `700`, `authorized_keys` a `600`.

## 4. Rafforzare SSH: niente root, niente password

La configurazione del server si trova in `/etc/ssh/sshd_config` e, con Debian 13, nei file drop-in sotto `/etc/ssh/sshd_config.d/`. Le modifiche vanno inserite in un file drop-in dedicato; così il file principale resta invariato e gli aggiornamenti dei pacchetti non sovrascrivono nulla. Creare il file `/etc/ssh/sshd_config.d/99-haertung.conf`:

```text
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
```

Questo disattiva l'accesso diretto come root e l'autenticazione tramite password. D'ora in poi può accedere solo chi possiede una chiave privata corrispondente. Prima di ricaricare, verificare sintatticamente la configurazione:

```bash
sudo sshd -t
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-t` | Modalità di test: verifica la validità del file di configurazione e delle chiavi senza avviare il servizio |

</details>

Se `sshd -t` non segnala nulla, il file è valido. Solo allora ricaricare:

```bash
sudo systemctl reload ssh
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `reload` | Ordina al servizio di ricaricare la propria configurazione senza interrompere le connessioni esistenti |
| `ssh` | L'unità di destinazione, qui il servizio OpenSSH |

</details>

**Importante:** lasciare aperta la sessione SSH esistente e testare il nuovo accesso in un secondo terminale. Solo quando l'accesso con chiave funziona in modo verificabile, si può chiudere la vecchia sessione. Questa precauzione riduce praticamente a zero il rischio di rimanere esclusi. Un errore nella configurazione altrimenti costa l'accesso completo.

## 5. Spostare SSH su una porta non comune

La porta standard 22 viene continuamente tentata dai bot. Il passaggio a una porta alta scelta liberamente (nell'esempio `61417`) fa sì che gran parte di questo rumore automatizzato non trovi nulla. Non è esplicitamente un guadagno di sicurezza in senso stretto: cambiare porta non sostituisce un'autenticazione forte, riduce soltanto la quantità di log e il carico delle scansioni. L'obbligo di usare chiavi del passaggio 4 resta la vera protezione.

La porta da scegliere non è del tutto arbitraria. La IANA distingue tre zone: **0–1023 (well-known ports)** sono riservate ai servizi standard (SSH stesso sulla 22, HTTP sulla 80, HTTPS sulla 443), richiedono root per il binding e non hanno posto su una porta SSH scelta autonomamente; sono proprio le porte che gli scanner, così come eventuali servizi standard installati in futuro, si aspettano. **1024–49151 (porte registrate)** sono assegnate su richiesta a singole applicazioni, per esempio 3306 (MySQL), 5432 (PostgreSQL), 6379 (Redis) o 8080/8443 come alternative HTTP diffuse; una porta scelta casualmente in questo intervallo può facilmente entrare in conflitto in seguito con software che si aspetta proprio la propria porta registrata. **49152–65535 (porte dinamiche/private)** non sono assegnate dalla IANA ad alcun servizio e sono destinate a scopi temporanei e privati: l'intervallo corretto per una porta permanente scelta autonomamente.

Resta una riserva: molti sistemi Linux, incluso Debian, usano una parte dello stesso intervallo come porta sorgente per le proprie connessioni in uscita (`net.ipv4.ip_local_port_range`, per impostazione predefinita attorno a 32768–60999). Un servizio in ascolto permanente non entra realmente in conflitto per questo motivo, poiché il kernel non assegna una porta già associata, ma una porta superiore a 60999 evita anche questa ambiguità teorica. L'esempio di questo articolo (`61417`) si trova quindi deliberatamente lì. Prima della modifica, verificare inoltre con `ss -lntup` (vedi passaggio 7) che la porta scelta non sia già occupata sul proprio server.

Con Debian 13 c'è una particolarità: SSH può essere avviato tramite attivazione socket di systemd. In tal caso, l'indicazione `Port` in `sshd_config` viene semplicemente ignorata; la porta deve quindi essere impostata sul socket. Verificare innanzitutto quale caso si applica:

```bash
systemctl is-enabled ssh.socket
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `is-enabled` | Mostra se l'unità è abilitata per l'avvio del sistema |
| `ssh.socket` | L'unità socket del servizio SSH |

</details>

Se il comando risponde con `enabled`, SSH funziona tramite socket. Modificare quindi la porta lì:

```bash
sudo systemctl edit ssh.socket
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `edit` | Crea un file di override drop-in per l'unità e lo apre nell'editor |
| `ssh.socket` | L'unità socket da sovrascrivere |

</details>

Nell'editor inserire le righe seguenti. La prima riga `ListenStream=` vuota elimina la porta 22 preimpostata, la seconda imposta quella nuova:

```text
[Socket]
ListenStream=
ListenStream=61417
```

Quindi applicare:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `daemon-reload` | Rilegge tutti i file delle unità, incluso il file di override appena creato |
| `restart ssh.socket` | Riavvia l'unità socket affinché resti in ascolto sulla nuova porta |

</details>

Se l'attivazione socket non è attiva (`disabled`), inserire invece `Port 61417` nel file drop-in del passaggio 4, seguito da `sudo sshd -t` e `sudo systemctl restart ssh`.

Anche qui vale quanto segue: prima aprire la nuova porta nel firewall (passaggio successivo), poi connettersi e testare, lasciando aperta la vecchia sessione finché l'accesso tramite la nuova porta non è confermato.

## 6. Firewall: chiuso per impostazione predefinita

Il firewall a monte del provider è il confine più efficace, poiché intercetta i pacchetti prima ancora che raggiungano il sistema operativo. Due regole di base:

- **Azione standard in ingresso su DROP.** Tutto ciò che non è espressamente consentito viene scartato, senza commenti né risposta al mittente.
- **Un'unica eccezione:** TCP in ingresso sulla porta di destinazione `61417`. Dall'esterno non deve essere raggiungibile altro.

Il traffico in uscita resta consentito. È una scelta deliberata: il server deve poter scaricare pacchetti, sincronizzare l'ora e raggiungere l'API per Claude Code. Un filtro restrittivo in uscita offre poca protezione aggiuntiva su un singolo server, ma rende l'utilizzo sensibilmente più scomodo.

Chi desidera ulteriore difesa in profondità può duplicare le stesse regole lato host con `nftables` o `ufw`. Per l'architettura descritta è sufficiente il firewall del provider.

## 7. Verificare la superficie di attacco

Dopo il rafforzamento, controllare ciò che il server offre effettivamente verso l'esterno. Bastano due comandi. Primo: quali servizi sono in ascolto su quali indirizzi?

```bash
sudo ss -lntup
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-l` | Mostra solo i socket in ascolto |
| `-n` | Output numerico: porte e indirizzi non vengono risolti in nomi |
| `-t` | Include i socket TCP |
| `-u` | Include i socket UDP |
| `-p` | Mostra il processo dietro ogni socket; per questo è necessario `sudo` |

</details>

La colonna degli indirizzi è decisiva: un servizio su `0.0.0.0` o `[::]` è raggiungibile dall'esterno, uno su `127.0.0.1` o `[::1]` solo localmente. Nello stato protetto dovrebbe apparire pubblicamente solo SSH. Servizi come `chronyd` (sincronizzazione dell'ora) possono comparire, ma solo associati ad indirizzi locali. Se `chronyd` ascolta esclusivamente su `127.0.0.1` e `::1`, non è raggiungibile dall'esterno e quindi non è problematico.

Secondo: ci sono servizi di sistema non riusciti che indicano un problema di configurazione?

```bash
systemctl --failed
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `--failed` | Elenca esclusivamente le unità nello stato di errore |

</details>

La risposta dovrebbe essere `0 loaded units listed`, senza neppure un servizio non riuscito. Le unità difettose non sono solo un problema operativo, ma potenzialmente anche di sicurezza se dietro di esse c'è un servizio di rete avviato a metà o configurato in modo errato.

## 8. Installare e utilizzare Claude Code

Claude Code richiede un ambiente di runtime Node.js aggiornato. Dopo averlo installato, configurare la CLI secondo la guida ufficiale e autenticarsi nuovamente sul server, senza caricare le credenziali locali (maggiori dettagli tra poco).

Per il funzionamento permanente, `tmux`:

```bash
tmux new -s claude
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `new` | Crea una nuova sessione |
| `-s claude` | Assegna il nome della sessione con cui potrà essere ripresa in seguito |

</details>

All'interno della sessione, avviare Claude. Con `Ctrl-b`, poi `d`, ci si stacca dalla sessione senza terminarla; Claude continua a funzionare. Per rientrare:

```bash
tmux attach -t claude
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `attach` | Ricollega il terminale a una sessione in esecuzione |
| `-t claude` | Seleziona la sessione di destinazione in base al suo nome |

</details>

In questo modo un'attività in esecuzione sopravvive alle connessioni interrotte, ai cambi di dispositivo e al riposo notturno del laptop.

## 9. Igiene dei dati durante la migrazione

La parte più delicata del trasferimento al server non è la tecnica, bensì la scelta di cosa portare con sé. Tre regole:

- **Nessuna chiave privata sul server.** In `authorized_keys` sono presenti esclusivamente chiavi pubbliche. Le chiavi private restano sui dispositivi finali.
- **Non copiare indiscriminatamente le credenziali.** I file locali sensibili, come un `.credentials.json`, non devono finire sul VPS senza verifica. Autenticarsi invece nuovamente sul server.
- **Prima la configurazione in una cartella di migrazione.** Non scrivere direttamente le memorie e la configurazione di Claude esistenti nei percorsi di configurazione attivi; trasferirle dapprima in una cartella di migrazione separata e verificare lì cosa debba essere realmente mantenuto. Ciò che non serve più, come vecchie voci MCP o impostazioni orfane, viene deliberatamente lasciato indietro invece di migrare senza controllo.

## 10. Anteprime web tramite un tunnel SSH

Per le anteprime web, per esempio un server di sviluppo locale avviato da Claude, è forte la tentazione di aprire semplicemente un'altra porta. Non bisogna farlo. Ogni porta aperta aggiuntiva aumenta la superficie di attacco. L'anteprima passa invece attraverso un tunnel di porta SSH cifrato: il servizio ascolta solo localmente sul server e SSH lo inoltra al client.

Dal PC, rendere raggiungibile un servizio in esecuzione localmente sulla porta 4321:

```bash
ssh -p 61417 -L 4321:localhost:4321 claude@SERVER
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-p 61417` | Porta sulla quale il server SSH è in ascolto (quella scelta nel passaggio 5) |
| `-L 4321:localhost:4321` | Inoltro della porta locale: le connessioni alla porta locale 4321 vengono inoltrate dal tunnel a `localhost:4321` dal punto di vista del server |
| `claude@SERVER` | Utente e host di destinazione della connessione SSH |

</details>

Aprire quindi `http://localhost:4321` nel browser locale. Il traffico passa interamente attraverso la connessione SSH esistente e autenticata, senza dover aprire nel firewall nemmeno una porta aggiuntiva.

## Accesso dall'iPhone

L'accesso in mobilità utilizza lo stesso modello di sicurezza del PC. Serve soltanto un client SSH con gestione delle chiavi. Sono diffusi **Termius**, **Blink Shell** e **Secure ShellFish**; tutti possono generare chiavi Ed25519 e salvarle nel portachiavi iOS, talvolta protette da Face ID.

La procedura corrisponde al passaggio 3, solo sull'iPhone:

1. Generare nel client SSH una chiave Ed25519 separata per l'iPhone, senza copiare la chiave del PC. La chiave privata resta nel portachiavi del dispositivo.
2. Inserire la chiave pubblica dell'iPhone come riga aggiuntiva in `~/.ssh/authorized_keys` sul server, con un commento descrittivo (`iphone-15`).
3. Creare la connessione nel client: indirizzo del server, utente `claude`, porta `61417`, chiave dell'iPhone come autenticazione.

È proprio per questo che vale la pena avere una chiave separata per dispositivo: se l'iPhone viene perso, si elimina sul server l'unica riga `iphone-15` da `authorized_keys`, e il dispositivo non può più accedere, mentre l'accesso dal PC e tutte le altre chiavi continuano a funzionare senza modifiche.

Dopo la connessione, riprendere la sessione Claude in esecuzione con `tmux attach -t claude` e continuare da dove si era rimasti alla scrivania. Anche il tunnel di porta del passaggio 10 funziona da iOS; Termius e Secure ShellFish supportano l'inoltro delle porte.

## Checklist

In sintesi, l'intera procedura:

1. Debian 13 installato e completamente aggiornato con `apt full-upgrade`.
2. Utente dedicato `claude` con privilegi sudo; l'accesso diretto come root non viene più usato.
3. Chiavi Ed25519 protette da passphrase, una per dispositivo, solo chiavi pubbliche in `authorized_keys`.
4. sshd rafforzato: `PermitRootLogin no`, `PasswordAuthentication no`; verificato prima del ricaricamento con `sshd -t`, sessione esistente mantenuta aperta fino al test.
5. SSH sulla porta 61417, impostata su `ssh.socket` in caso di attivazione socket, altrimenti nella configurazione di sshd.
6. Firewall del provider: DROP predefinito in ingresso, unica eccezione TCP 61417; traffico in uscita consentito.
7. Superficie di attacco verificata con `ss -lntup` (solo SSH pubblico, `chronyd` locale) e `systemctl --failed` (nessun errore).
8. Claude Code autenticato nuovamente sul server, esecuzione in una sessione `tmux`.
9. Igiene dei dati: nessuna chiave privata né credenziale sul server, configurazione verificata dapprima tramite una cartella di migrazione.
10. Nessuna porta aggiuntiva; le anteprime web passano attraverso un tunnel SSH.

Dopo questa configurazione, dall'esterno è raggiungibile solo SSH sulla porta stabilita, e anche lì esclusivamente tramite una chiave protetta da passphrase. Claude Code funziona indipendentemente dal dispositivo in `tmux`; le anteprime web restano accessibili tramite tunnel SSH senza aprire una porta aggiuntiva.

## Fonti

1.  [Manuale OpenSSH – sshd_config(5)](https://man.openbsd.org/sshd_config): riferimento per tutte le direttive di sshd, incluse `PermitRootLogin`, `PasswordAuthentication` e `PubkeyAuthentication`.

2.  [Wiki Debian – SSH](https://wiki.debian.org/SSH): indicazioni specifiche per Debian sulla configurazione SSH, inclusi i file drop-in sotto `/etc/ssh/sshd_config.d/`.

3.  [systemd.socket(5) – freedesktop.org](https://www.freedesktop.org/software/systemd/man/latest/systemd.socket.html): funzionamento dell'attivazione socket e della direttiva `ListenStream=`, rilevante per il cambio della porta SSH in Debian 13.

4.  [ss(8) – pagina man di iproute2](https://man7.org/linux/man-pages/man8/ss.8.html): opzioni di `ss` per elencare i socket in ascolto insieme a processo e indirizzo di binding.

5.  [Claude Code – documentazione ufficiale](https://docs.claude.com/en/docs/claude-code/overview): installazione, autenticazione e utilizzo di Claude Code.
