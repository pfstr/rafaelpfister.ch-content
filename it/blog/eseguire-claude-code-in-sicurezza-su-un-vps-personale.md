---
title: "Eseguire Claude Code in modo sicuro sul proprio VPS"
navTitle: "VPS per Claude"
description: "Un VPS Debian protetto mantiene le sessioni di Claude Code permanentemente accessibili. La guida copre tutto, dall'account utente e dalle chiavi SSH fino al firewall, all'igiene dei dati, a tmux e all'accesso sicuro da iPhone."
date: "2026-07-21"
kategorie: "Claude"
timeToRead: "12 min di lettura"
themen:
  - claude
slug: "eseguire-claude-code-in-sicurezza-su-un-vps-personale"
translationOf: "claude-code-vps-debian-absichern"
translationId: article-f932e9e537d7704a
translationReview: automatic
translationSourceHash: 0c584eb684bd92b439d97b20aae84cd15468c5c30d1e277d7fde7bd3fab9a37e
translatedAt: 2026-10-31T03:43:00.981Z
translationModel: gpt-5.6-terra
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/eseguire-claude-code-in-sicurezza-su-un-vps-personale
---

Su un computer proprio, una sessione di Claude Code termina involontariamente al più tardi quando il laptop va in sospensione o la connessione di rete si interrompe. Un VPS continua a funzionare ed è raggiungibile da più dispositivi. Al tempo stesso, rimane connesso in modo permanente a Internet pubblico e viene sottoposto a scansioni automatizzate già poco dopo l'avvio.

Questa guida combina entrambe le esigenze: Claude Code resta disponibile in una sessione `tmux`, mentre il server Debian offre verso l'esterno solo una connessione SSH protetta da chiave. L'hardening non è specifico di Claude ed è adatto anche ad altri server Linux raggiungibili pubblicamente.

## Perché un VPS può essere utile

Rispetto a un'installazione esclusivamente locale, il server offre tre vantaggi pratici:

- **Persistenza.** In una sessione `tmux`, Claude continua a funzionare anche se la connessione SSH viene interrotta. Un'attività che richiede dieci minuti o un'ora viene completata senza dover lasciare aperto il laptop.
- **Accessibilità.** La stessa sessione è accessibile dal desktop, dal laptop e dall'iPhone. Si avvia un'attività alla scrivania e se ne controlla il risultato mentre si è in viaggio.
- **Controllo dei dati.** Si decide personalmente cosa risiede sul server. Nessun servizio di sincronizzazione, nessuna credenziale inclusa involontariamente nei backup, a condizione di procedere con attenzione durante la migrazione (vedi sotto).

`tmux` è quindi una funzionalità puramente di disponibilità e comodità, non una misura di sicurezza. Il vero lavoro sta nella protezione.

## Situazione di partenza

La base è Debian 13 (Trixie), installato in versione minimale, senza desktop né servizi di rete aggiuntivi. Il provider mette a disposizione un firewall a monte che interviene indipendentemente dal sistema operativo. L'obiettivo è un server sul quale dall'esterno sia raggiungibile esclusivamente SSH, e solo con chiavi protette da passphrase.

## 1. Aggiornare il sistema

Subito dopo l'installazione, aggiornare l'intero set di pacchetti:

```bash
sudo apt update
sudo apt full-upgrade
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `update` | Rilegge gli elenchi dei pacchetti di tutte le fonti configurate |
| `full-upgrade` | Aggiorna tutti i pacchetti e può anche installare nuovi pacchetti o rimuovere quelli esistenti |

</details>

A differenza di `upgrade`, `full-upgrade` risolve anche le dipendenze che richiedono pacchetti nuovi o rimossi. Su un sistema appena installato, questo è il metodo corretto per installare davvero tutti gli aggiornamenti di sicurezza disponibili. Riavviare una volta dopo gli aggiornamenti del kernel.

## 2. Utente dedicato invece di root

Lavorare come root è inutilmente rischioso: ogni errore di digitazione ha effetto sull'intero sistema e il login root diretto è la prima cosa che gli attacchi automatizzati tentano. Creare quindi un utente dedicato (qui `claude`) con privilegi sudo per i casi in cui siano necessari:

```bash
sudo adduser claude
sudo usermod -aG sudo claude
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-a` | Aggiunge: integra l'elenco dei gruppi dell'utente invece di sostituirlo; valido solo insieme a `-G` |
| `-G sudo` | Gruppo o gruppi supplementari ai quali viene aggiunto l'utente |
| `claude` | L'utente interessato; con `adduser` è il nome dell'account da creare |

</details>

Da ora in poi tutta l'amministrazione avviene tramite `claude` e `sudo`, non più tramite accesso root diretto.

## 3. Chiavi Ed25519 con passphrase, una per dispositivo

L'accesso deve avvenire esclusivamente tramite chiavi SSH, non tramite password. Ed25519 è lo standard attuale: breve, veloce e crittograficamente solido. È fondamentale che la chiave venga generata sul client, cioè sul PC e non sul server, e sia protetta da una passphrase. La passphrase è la seconda linea di difesa nel caso in cui la chiave privata finisca nelle mani sbagliate.

Sul PC:

```bash
ssh-keygen -t ed25519 -C "pc-thinkpad"
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-t ed25519` | Tipo di chiave, qui l'algoritmo ellittico Ed25519 |
| `-C "pc-thinkpad"` | Commento aggiunto alla chiave pubblica |

</details>

Il commento (`-C`) identifica il dispositivo. Questo risulterà utile in seguito: si genera una chiave distinta per ciascun dispositivo, una per il PC e una separata per l'iPhone. Se un dispositivo viene smarrito, si rimuove selettivamente la sua chiave pubblica da `~/.ssh/authorized_keys`, senza dover distribuire nuovamente tutti gli altri accessi.

Sul server deve essere presente soltanto la chiave pubblica. La chiave privata non lascia mai il dispositivo. In `authorized_keys` saranno presenti esclusivamente chiavi pubbliche, ciascuna con il commento del proprio dispositivo:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...pc  pc-thinkpad
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...ios iphone-15
```

Trasferire inizialmente la chiave pubblica del PC. Finché l'autenticazione con password è ancora attiva, il modo più semplice è:

```bash
ssh-copy-id claude@SERVER
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `claude@SERVER` | Utente e host di destinazione; vi viene aggiunta la chiave pubblica standard a `~/.ssh/authorized_keys` |

</details>

Successivamente, verificare che il login con chiave funzioni prima di disattivare l'autenticazione con password nel passaggio seguente. I permessi dei file devono essere corretti, altrimenti sshd ignora il file: `~/.ssh` su `700`, `authorized_keys` su `600`.

## 4. Rafforzare SSH: niente root, niente password

La configurazione del server si trova in `/etc/ssh/sshd_config` e, su Debian 13, nei file drop-in sotto `/etc/ssh/sshd_config.d/`. Le modifiche devono essere inserite in un file drop-in dedicato; così il file principale rimane inalterato e gli aggiornamenti dei pacchetti non sovrascrivono nulla. Creare il file `/etc/ssh/sshd_config.d/99-haertung.conf`:

```text
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
```

Questo disattiva il login root diretto e l'autenticazione con password. Da ora in poi può accedere solo chi possiede una chiave privata idonea. Prima di ricaricare, verificare sintatticamente la configurazione:

```bash
sudo sshd -t
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-t` | Modalità di test: verifica la validità del file di configurazione e delle chiavi senza avviare il servizio |

</details>

Se `sshd -t` non segnala nulla, il file è valido. Solo allora ricaricare:

```bash
sudo systemctl reload ssh
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `reload` | Indica al servizio di ricaricare la configurazione senza interrompere le connessioni esistenti |
| `ssh` | L'unità di destinazione, qui il servizio OpenSSH |

</details>

**Importante:** lasciare aperta la sessione SSH esistente e testare il nuovo accesso in un secondo terminale. Solo quando il login con chiave funziona dimostrabilmente lì, si può chiudere la vecchia sessione. Questa precauzione riduce a quasi zero il rischio di rimanere esclusi. Un errore nella configurazione altrimenti costa l'accesso completo.

## 5. Spostare SSH su una porta non comune

La porta standard 22 viene continuamente sondata dai bot. Passare a una porta alta scelta liberamente (nell'esempio `61417`) fa sì che gran parte di questo rumore automatizzato non produca risultati. Non si tratta esplicitamente di un guadagno di sicurezza in senso stretto: cambiare porta non sostituisce un'autenticazione forte, riduce soltanto la quantità di log e il carico delle scansioni. L'obbligo di usare chiavi del passaggio 4 rimane la vera protezione.

La scelta della porta non è arbitraria. IANA distingue tre zone: **0–1023 (well-known ports)** sono riservate ai servizi standard (SSH stesso sulla 22, HTTP sulla 80, HTTPS sulla 443), richiedono root per il binding e non hanno posto su una porta SSH scelta autonomamente; questi sono proprio i porti che gli scanner e i servizi standard eventualmente installati in seguito si aspettano. **1024–49151 (porte registrate)** sono assegnate su richiesta a singole applicazioni, ad esempio 3306 (MySQL), 5432 (PostgreSQL), 6379 (Redis) o 8080/8443 come diffuse alternative HTTP; una porta scelta casualmente in questo intervallo può facilmente entrare in conflitto in seguito con software che si aspetta esattamente la propria porta registrata. **49152–65535 (porte dinamiche/private)** non sono assegnate da IANA ad alcun servizio e sono destinate a scopi temporanei e privati: è l'intervallo corretto per una porta permanente scelta autonomamente.

Rimane una riserva: molti sistemi Linux, incluso Debian, usano una parte dello stesso intervallo come porta sorgente per le proprie connessioni in uscita (`net.ipv4.ip_local_port_range`, per impostazione predefinita attorno a 32768–60999). Un servizio in ascolto permanente non entra davvero in conflitto per questo, poiché il kernel non assegna una porta già occupata, ma una porta superiore a 60999 evita anche questa ambiguità teorica. L'esempio di questo articolo (`61417`) è quindi intenzionalmente in quell'intervallo. Prima della modifica, verificare inoltre con `ss -lntup` (vedi passaggio 7) che la porta scelta non sia già occupata sul proprio server.

Con Debian 13 c'è una particolarità: SSH può essere avviato tramite attivazione socket di systemd. In tal caso, l'indicazione `Port` in `sshd_config` viene semplicemente ignorata; la porta deve quindi essere impostata sul socket. Innanzitutto verificare quale caso si applica:

```bash
systemctl is-enabled ssh.socket
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `is-enabled` | Mostra se l'unità è abilitata all'avvio del sistema |
| `ssh.socket` | L'unità socket del servizio SSH |

</details>

Se il comando risponde con `enabled`, SSH viene eseguito tramite socket. Modificare quindi la porta lì:

```bash
sudo systemctl edit ssh.socket
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `edit` | Crea un file di override drop-in per l'unità e lo apre nell'editor |
| `ssh.socket` | L'unità socket da sovrascrivere |

</details>

Nell'editor inserire le righe seguenti. La prima riga `ListenStream=` vuota cancella la porta 22 preimpostata, la seconda imposta quella nuova:

```text
[Socket]
ListenStream=
ListenStream=61417
```

Applicare quindi le modifiche:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `daemon-reload` | Rilegge tutti i file delle unità, incluso il file di override appena creato |
| `restart ssh.socket` | Riavvia l'unità socket affinché rimanga in ascolto sulla nuova porta |

</details>

Se l'attivazione socket non è attiva (`disabled`), inserire invece `Port 61417` nel file drop-in del passaggio 4, seguito da `sudo sshd -t` e `sudo systemctl restart ssh`.

Anche qui vale: aprire prima la nuova porta nel firewall (passaggio successivo), poi connettersi e testare, lasciando aperta la vecchia sessione finché l'accesso tramite la nuova porta non è confermato.

## 6. Firewall: chiuso per impostazione predefinita

Il firewall a monte del provider è il confine più efficace, perché intercetta i pacchetti prima ancora che raggiungano il sistema operativo. Due regole fondamentali:

- **Azione standard in entrata: DROP.** Tutto ciò che non è espressamente consentito viene scartato, senza commenti e senza risposta al mittente.
- **Un'unica eccezione:** TCP in entrata sulla porta di destinazione `61417`. Dall'esterno non deve essere raggiungibile altro.

Il traffico in uscita rimane consentito. È una scelta intenzionale: il server deve poter scaricare pacchetti, sincronizzare l'ora e raggiungere l'API per Claude Code. Un filtraggio restrittivo in uscita apporta poca protezione aggiuntiva su un singolo server, ma rende il funzionamento sensibilmente più complicato.

Chi desidera una maggiore difesa in profondità può duplicare le stesse regole lato host con `nftables` o `ufw`. Per l'installazione descritta è sufficiente il firewall del provider.

## 7. Verificare la superficie di attacco

Dopo l'hardening, controllare cosa offre effettivamente il server verso l'esterno. Bastano due comandi. Primo: quali servizi sono in ascolto su quali indirizzi?

```bash
sudo ss -lntup
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-l` | Mostra solo i socket in ascolto |
| `-n` | Output numerico: porte e indirizzi non vengono risolti in nomi |
| `-t` | Include i socket TCP |
| `-u` | Include i socket UDP |
| `-p` | Mostra il processo dietro ogni socket; richiede `sudo` |

</details>

La colonna decisiva è quella degli indirizzi: un servizio su `0.0.0.0` o `[::]` è raggiungibile dall'esterno, mentre uno su `127.0.0.1` o `[::1]` è solo locale. Nello stato protetto, dovrebbe apparire pubblicamente solo SSH. Servizi come `chronyd` (sincronizzazione dell'ora) possono comparire, ma solo associati a indirizzi locali. Se `chronyd` è in ascolto esclusivamente su `127.0.0.1` e `::1`, non è raggiungibile dall'esterno e pertanto non è problematico.

Secondo: esistono servizi di sistema non riusciti che indicano un problema di configurazione?

```bash
systemctl --failed
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `--failed` | Elenca esclusivamente le unità in stato di errore |

</details>

La risposta dovrebbe essere `0 loaded units listed`, senza alcun servizio non riuscito. Le unità in errore non sono solo un problema operativo, ma potenzialmente anche di sicurezza, se dietro di esse si nasconde un servizio di rete avviato a metà o configurato in modo errato.

## 8. Installare e utilizzare Claude Code

Claude Code non richiede un ambiente di runtime Node.js separato. Anthropic consiglia l'installer nativo (`curl -fsSL https://claude.ai/install.sh | bash`), che aggiorna autonomamente Claude Code in background; per Debian è disponibile in alternativa un repository apt firmato, aggiornato tramite i normali aggiornamenti di sistema. Dopo l'installazione, autenticare nuovamente la CLI sul server, senza caricare le credenziali locali (maggiori dettagli tra poco).

Per l'uso continuativo, `tmux`:

```bash
tmux new -s claude
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

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
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `attach` | Ricollega il terminale a una sessione in esecuzione |
| `-t claude` | Seleziona la sessione di destinazione in base al nome |

</details>

In questo modo un'attività in corso sopravvive a connessioni interrotte, cambi di dispositivo e al riposo notturno del laptop.

## 9. Igiene dei dati durante la migrazione

La parte più delicata del trasferimento sul server non è la tecnica, ma la domanda su cosa portare con sé. Tre regole:

- **Nessuna chiave privata sul server.** In `authorized_keys` sono presenti esclusivamente chiavi pubbliche. Le chiavi private restano sui dispositivi finali.
- **Non copiare indiscriminatamente le credenziali.** File locali sensibili come `.credentials.json` non devono finire sul VPS senza verifiche. Autenticarsi invece nuovamente sul server.
- **Configurazione prima in una cartella di migrazione.** Non scrivere direttamente nei percorsi di configurazione attivi le memories e la configurazione Claude esistenti; trasferirle dapprima in una cartella di migrazione separata e verificare lì cosa si desidera davvero adottare. Ciò che non serve più, come vecchie voci MCP o impostazioni orfane, rimane intenzionalmente indietro anziché migrare senza controllo.

## 10. Anteprime web tramite un tunnel SSH

Per le anteprime web, ad esempio un server di sviluppo locale avviato da Claude, la tentazione di aprire semplicemente un'altra porta è forte. Non bisogna farlo. Ogni porta aperta in più aumenta la superficie di attacco. L'anteprima passa invece attraverso un tunnel SSH cifrato: il servizio ascolta solo localmente sul server e SSH lo inoltra al client.

Dal PC, rendere raggiungibile un servizio in esecuzione localmente sulla porta 4321:

```bash
ssh -p 61417 -L 4321:localhost:4321 claude@SERVER
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-p 61417` | Porta sulla quale il server SSH è in ascolto, quella scelta nel passaggio 5 |
| `-L 4321:localhost:4321` | Inoltro di porta locale: le connessioni alla porta locale 4321 vengono inoltrate attraverso il tunnel a `localhost:4321` dal punto di vista del server |
| `claude@SERVER` | Utente e host di destinazione della connessione SSH |

</details>

Successivamente si apre `http://localhost:4321` nel browser locale. Il traffico passa interamente attraverso la connessione SSH esistente e autenticata, senza dover aprire nel firewall nemmeno una porta aggiuntiva.

## Accesso dall'iPhone

L'accesso in mobilità funziona con lo stesso modello di sicurezza del PC. Serve solo un client SSH con gestione delle chiavi. Sono diffusi **Termius**, **Blink Shell** e **Secure ShellFish**; tutti possono generare chiavi Ed25519 e memorizzarle nel portachiavi iOS, in parte protette da Face ID.

La procedura corrisponde al passaggio 3, solo sull'iPhone:

1. Nel client SSH, generare una chiave Ed25519 dedicata per l'iPhone, senza copiare la chiave del PC. La chiave privata resta nel portachiavi del dispositivo.
2. Inserire la chiave pubblica dell'iPhone come riga aggiuntiva in `~/.ssh/authorized_keys` sul server, con un commento descrittivo (`iphone-15`).
3. Creare la connessione nel client: indirizzo del server, utente `claude`, porta `61417`, chiave dell'iPhone come autenticazione.

Proprio per questo vale la pena avere una chiave separata per dispositivo: se l'iPhone viene smarrito, sul server si elimina l'unica riga `iphone-15` da `authorized_keys`, e il dispositivo non può più accedere, mentre l'accesso dal PC e tutte le altre chiavi continuano a funzionare senza modifiche.

Dopo la connessione, si riprende la sessione Claude in esecuzione con `tmux attach -t claude` e si continua da dove si era rimasti alla scrivania. Anche il tunnel di porta del passaggio 10 funziona da iOS; Termius e Secure ShellFish supportano l'inoltro di porta.

## Checklist

In sintesi, la procedura completa:

1. Debian 13 installato e completamente aggiornato con `apt full-upgrade`.
2. Utente dedicato `claude` con privilegi sudo; il login root diretto non viene più usato.
3. Chiavi Ed25519 protette da passphrase, una per dispositivo, solo chiavi pubbliche in `authorized_keys`.
4. sshd rafforzato: `PermitRootLogin no`, `PasswordAuthentication no`; verificato con `sshd -t` prima della ricarica, sessione esistente lasciata aperta fino al test.
5. SSH sulla porta 61417, impostata in `ssh.socket` in caso di attivazione socket, altrimenti nella configurazione sshd.
6. Firewall del provider: DROP predefinito in entrata, unica eccezione TCP 61417; traffico in uscita consentito.
7. Superficie di attacco verificata con `ss -lntup` (solo SSH pubblico, `chronyd` locale) e `systemctl --failed` (nessun errore).
8. Claude Code autenticato nuovamente sul server, eseguito in una sessione `tmux`.
9. Igiene dei dati: nessuna chiave privata e nessuna credenziale sul server; configurazione prima verificata tramite una cartella di migrazione.
10. Nessuna porta aggiuntiva; le anteprime web passano attraverso un tunnel SSH.

Dopo questa configurazione, dall'esterno è raggiungibile solo SSH sulla porta definita, e anche in quel caso esclusivamente con una chiave protetta da passphrase. Claude Code viene eseguito indipendentemente dal dispositivo finale in `tmux`; le anteprime web restano accessibili tramite tunnel SSH senza aprire una porta aggiuntiva.

## Cronologia delle modifiche

- **25 ottobre 2026:** Corretto: Claude Code non richiede un ambiente di runtime Node.js separato; aggiunti l'installer nativo consigliato e il repository apt firmato per Debian. Aggiornati i link alle fonti con i nuovi indirizzi della documentazione di Claude Code.

## Fonti

1.  [OpenSSH Manual – sshd_config(5)](https://man.openbsd.org/sshd_config): riferimento per tutte le direttive sshd, incluse `PermitRootLogin`, `PasswordAuthentication` e `PubkeyAuthentication`.

2.  [Debian Wiki – SSH](https://wiki.debian.org/SSH): indicazioni specifiche per Debian sulla configurazione SSH, inclusi i file drop-in sotto `/etc/ssh/sshd_config.d/`.

3.  [systemd.socket(5) – freedesktop.org](https://www.freedesktop.org/software/systemd/man/latest/systemd.socket.html): funzionamento dell'attivazione socket e della direttiva `ListenStream=`, rilevante per il cambio della porta SSH su Debian 13.

4.  [ss(8) – iproute2 Manpage](https://man7.org/linux/man-pages/man8/ss.8.html): opzioni di `ss` per elencare i socket in ascolto con processo e indirizzo di binding.

5.  [Claude Code – Documentazione ufficiale](https://code.claude.com/docs/en/overview): installazione, autenticazione e utilizzo di Claude Code.

6.  [Claude Code – Advanced setup](https://code.claude.com/docs/en/setup): requisiti di sistema, installer nativo, repository apt e aggiornamenti automatici.
