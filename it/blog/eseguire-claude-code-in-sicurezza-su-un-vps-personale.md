---
title: "Eseguire Claude Code in sicurezza su un VPS personale"
navTitle: "VPS per Claude"
description: "Un VPS Debian rinforzato mantiene le sessioni di Claude Code sempre accessibili. La guida illustra account utente e chiavi SSH, firewall, igiene dei dati, tmux e accesso sicuro da iPhone."
date: "2026-07-21"
kategorie: "Claude"
timeToRead: "12 min di lettura"
themen:
  - claude
slug: "eseguire-claude-code-in-sicurezza-su-un-vps-personale"
translationOf: "claude-code-vps-debian-absichern"
url: "https://rafaelpfister.ch/it/blog/eseguire-claude-code-in-sicurezza-su-un-vps-personale"
translationId: article-f932e9e537d7704a
translationReview: automatic
translationSourceHash: bd2aac7348c16dbd326ab0c10a063817d88a05cb99ab88a8cde66b885dfd7c3f
translatedAt: 2026-07-29T12:29:38.942Z
---

Su un computer personale, una sessione di Claude Code termina involontariamente al più tardi quando il laptop entra in sospensione o la connessione di rete si interrompe. Un VPS continua a funzionare ed è accessibile da più dispositivi. Al contempo, rimane permanentemente connesso a Internet pubblico e viene sottoposto a scansioni automatizzate già poco dopo l'avvio.

Questa guida combina entrambi i requisiti: Claude Code resta disponibile in una sessione `tmux`, mentre il server Debian offre verso l'esterno solo una connessione SSH protetta da chiavi. Il rafforzamento non è specifico di Claude ed è adatto anche ad altri server Linux pubblicamente raggiungibili.

## Perché un VPS può essere utile

Rispetto a un'installazione esclusivamente locale, il server offre tre vantaggi pratici:

- **Persistenza.** In una sessione `tmux`, Claude continua a funzionare anche se la connessione SSH viene interrotta. Un'attività che richiede dieci minuti o un'ora viene completata senza dover lasciare aperto il laptop.
- **Accessibilità.** La stessa sessione è accessibile da desktop, laptop e iPhone. Si avvia un'attività alla scrivania e si controlla il risultato mentre si è in viaggio.
- **Controllo dei dati.** Si decide autonomamente cosa risiede sul server. Nessun servizio di sincronizzazione, nessuna credenziale inclusa involontariamente nel backup, a condizione di procedere con attenzione durante la migrazione (vedi sotto).

`tmux` è quindi una pura funzionalità di disponibilità e comodità, non una misura di sicurezza. Il vero lavoro consiste nella protezione.

## Situazione di partenza

La base è Debian 13 (Trixie), installato in versione minima, senza desktop né servizi di rete aggiuntivi. Il provider mette a disposizione un firewall a monte, che agisce indipendentemente dal sistema operativo. L'obiettivo è un server sul quale dall'esterno sia raggiungibile esclusivamente SSH, e anche questo solo con chiavi protette da passphrase.

## 1. Aggiornare il sistema

Subito dopo l'installazione, aggiornare l'intero stato dei pacchetti:

```bash
sudo apt update
sudo apt full-upgrade
```

`full-upgrade`, a differenza di `upgrade`, risolve anche le dipendenze che richiedono pacchetti nuovi o rimossi. Su un sistema appena installato, questo è il modo corretto per applicare davvero tutti gli aggiornamenti di sicurezza disponibili. Riavviare una volta dopo gli aggiornamenti del kernel.

## 2. Un utente dedicato invece di root

Lavorare come root è inutilmente rischioso: ogni errore di digitazione ha effetto sull'intero sistema e il login root diretto è la prima cosa che gli attacchi automatizzati tentano. Creare quindi un utente dedicato (qui `claude`) con diritti sudo per i casi in cui sono necessari:

```bash
sudo adduser claude
sudo usermod -aG sudo claude
```

Da questo momento, tutta l'amministrazione avviene tramite `claude` e `sudo`, non più tramite accesso root diretto.

## 3. Chiavi Ed25519 con passphrase, una per dispositivo

L'autenticazione deve avvenire esclusivamente tramite chiavi SSH, non tramite password. Ed25519 è lo standard attuale: breve, veloce e crittograficamente solido. È fondamentale che la chiave venga generata sul client, quindi sul PC e non sul server, e protetta da una passphrase. La passphrase è la seconda linea di difesa nel caso in cui la chiave privata finisca nelle mani sbagliate.

Sul PC:

```bash
ssh-keygen -t ed25519 -C "pc-thinkpad"
```

Il commento (`-C`) identifica il dispositivo. Questo torna utile in seguito: per ogni dispositivo viene generata una chiave dedicata, una per il PC e una separata per l'iPhone. Se un dispositivo viene smarrito, si rimuove in modo mirato la sua chiave pubblica da `~/.ssh/authorized_keys`, senza dover distribuire nuovamente tutti gli altri accessi.

Solo la chiave pubblica va sul server. La chiave privata non lascia mai il dispositivo. In `authorized_keys` alla fine sono presenti esclusivamente chiavi pubbliche, ciascuna con il commento del proprio dispositivo:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...pc  pc-thinkpad
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...ios iphone-15
```

Trasferire inizialmente la chiave pubblica del PC. Finché l'autenticazione tramite password è ancora attiva, il modo più semplice è:

```bash
ssh-copy-id claude@SERVER
```

Successivamente, verificare che il login con chiave funzioni prima di disattivare l'autenticazione tramite password nel passo successivo. I permessi del file devono essere corretti, altrimenti sshd ignora il file: `~/.ssh` su `700`, `authorized_keys` su `600`.

## 4. Rafforzare SSH: niente root, niente password

La configurazione del server si trova in `/etc/ssh/sshd_config` e (su Debian 13) nei file drop-in in `/etc/ssh/sshd_config.d/`. Le modifiche vanno inserite in un file drop-in dedicato; in questo modo il file principale rimane invariato e gli aggiornamenti dei pacchetti non sovrascrivono nulla. Creare il file `/etc/ssh/sshd_config.d/99-haertung.conf`:

```text
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
```

Questo disabilita il login root diretto e l'autenticazione tramite password. Da ora può entrare solo chi possiede una chiave privata corrispondente. Prima di ricaricare, verificare sintatticamente la configurazione:

```bash
sudo sshd -t
```

Se `sshd -t` non segnala nulla, il file è valido. Solo allora ricaricare:

```bash
sudo systemctl reload ssh
```

**Importante:** lasciare aperta la sessione SSH esistente e testare il nuovo accesso in un secondo terminale. Solo quando il login con chiave funziona in modo dimostrabile lì, si può chiudere la vecchia sessione. Questa precauzione riduce praticamente a zero il rischio di rimanere bloccati fuori. Un errore nella configurazione altrimenti costa l'intero accesso.

## 5. Spostare SSH su una porta non comune

La porta standard 22 viene provata dai bot ventiquattr'ore su ventiquattro. Passare a una porta alta scelta liberamente (nell'esempio `61417`) fa sì che la maggior parte di questo rumore automatizzato non produca effetti. Non si tratta esplicitamente di un miglioramento della sicurezza in senso stretto: cambiare porta non sostituisce un'autenticazione forte, riduce soltanto il volume dei log e il carico delle scansioni. L'obbligo delle chiavi del passo 4 rimane la vera protezione.

La scelta della porta non è arbitraria. IANA distingue tre zone: **0–1023 (well-known ports)** sono riservate ai servizi standard (SSH stesso sulla 22, HTTP sulla 80, HTTPS sulla 443), richiedono root per il binding e non hanno posto su una porta SSH scelta autonomamente; sono proprio le porte che gli scanner si aspettano, così come i servizi standard eventualmente installati in seguito. **1024–49151 (porte registrate)** sono assegnate su richiesta a singole applicazioni, ad esempio 3306 (MySQL), 5432 (PostgreSQL), 6379 (Redis) oppure 8080/8443 come diffuse alternative HTTP; una porta scelta casualmente in questo intervallo può facilmente entrare in conflitto in seguito con software che si aspetta proprio la propria porta registrata. **49152–65535 (porte dinamiche/private)** non sono assegnate a nessun servizio secondo IANA e sono pensate per scopi temporanei e privati: è l'intervallo corretto per una porta permanente scelta autonomamente.

Resta una riserva: molti sistemi Linux, incluso Debian, usano una parte dello stesso intervallo come porta sorgente per le proprie connessioni in uscita (`net.ipv4.ip_local_port_range`, per impostazione predefinita attorno a 32768–60999). Un servizio in ascolto permanente non entra realmente in conflitto per questo motivo, poiché il kernel non assegna una porta già vincolata, ma una porta superiore a 60999 evita anche questa imprecisione teorica. L'esempio in questo articolo (`61417`) si trova quindi deliberatamente in quell'area. Prima della modifica, verificare inoltre con `ss -lntup` (vedi passo 7) che la porta scelta non sia già occupata sul proprio server.

Su Debian 13 c'è un'insidia: SSH può essere avviato tramite attivazione socket di systemd. In tal caso, l'indicazione `Port` in `sshd_config` viene semplicemente ignorata; la porta deve allora essere impostata nel socket. Verificare prima quale caso si applica:

```bash
systemctl is-enabled ssh.socket
```

Se il comando risponde `enabled`, SSH viene eseguito tramite socket. Modificare quindi la porta lì:

```bash
sudo systemctl edit ssh.socket
```

Inserire le seguenti righe nell'editor. La prima riga `ListenStream=` vuota elimina la porta 22 preimpostata, la seconda imposta quella nuova:

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

Se l'attivazione socket non è attiva (`disabled`), inserire invece `Port 61417` nel file drop-in del passo 4, seguito da `sudo sshd -t` e `sudo systemctl restart ssh`.

Anche qui vale la regola: aprire prima la nuova porta nel firewall (passo successivo), poi collegarsi e testare, lasciando aperta la vecchia sessione finché l'accesso tramite la nuova porta non è confermato.

## 6. Firewall: chiuso per impostazione predefinita

Il firewall a monte del provider è il confine più efficace, perché intercetta i pacchetti prima ancora che raggiungano il sistema operativo. Due regole di base:

- **Azione predefinita in ingresso su DROP.** Tutto ciò che non è esplicitamente consentito viene scartato, senza commenti e senza riscontro al mittente.
- **Un'unica eccezione:** TCP in ingresso sulla porta di destinazione `61417`. Non deve essere raggiungibile altro dall'esterno.

Il traffico in uscita rimane consentito. È una scelta consapevole: il server deve poter scaricare pacchetti, sincronizzare l'ora e raggiungere l'API per Claude Code. Un filtraggio restrittivo in uscita offre poca protezione aggiuntiva su un singolo server, ma rende il funzionamento sensibilmente più complicato.

Chi desidera una difesa in profondità aggiuntiva può duplicare le stesse regole lato host con `nftables` o `ufw`. Per la configurazione descritta è sufficiente il firewall del provider.

## 7. Verificare la superficie d'attacco

Dopo il rafforzamento, controllare ciò che il server offre effettivamente verso l'esterno. Bastano due comandi. Primo: quali servizi sono in ascolto e su quali indirizzi?

```bash
sudo ss -lntup
```

`ss` elenca tutti i socket TCP e UDP in ascolto insieme al processo associato (`sudo` è necessario per vedere i nomi dei processi). Decisiva è la colonna degli indirizzi: un servizio su `0.0.0.0` o `[::]` è raggiungibile dall'esterno, uno su `127.0.0.1` o `[::1]` solo localmente. Nello stato protetto dovrebbe apparire pubblicamente solo SSH. Servizi come `chronyd` (sincronizzazione dell'ora) possono comparire, ma solo vincolati ad indirizzi locali. Se `chronyd` ascolta esclusivamente su `127.0.0.1` e `::1`, non è raggiungibile dall'esterno e quindi non è critico.

Secondo: esistono servizi di sistema non riusciti che indicano un problema di configurazione?

```bash
systemctl --failed
```

La risposta dovrebbe essere `0 loaded units listed`, nessun singolo servizio non riuscito. Le unità difettose non sono solo un problema operativo, ma potenzialmente anche di sicurezza, se dietro si cela un servizio di rete avviato a metà o configurato in modo errato.

## 8. Installare e utilizzare Claude Code

Claude Code richiede un ambiente di esecuzione Node.js aggiornato. Dopo averlo installato, configurare la CLI secondo la guida ufficiale e autenticarsi nuovamente sul server, senza caricare le credenziali locali (maggiori dettagli tra poco).

Per l'esecuzione permanente, `tmux`:

```bash
tmux new -s claude
```

All'interno della sessione avviare Claude. Con `Ctrl-b`, poi `d`, ci si scollega dalla sessione senza terminarla; Claude continua a funzionare. Per rientrare usare:

```bash
tmux attach -t claude
```

In questo modo, un'attività in esecuzione sopravvive a connessioni interrotte, cambi di dispositivo e alla notte del laptop.

## 9. Igiene dei dati durante la migrazione

La parte più delicata del trasferimento sul server non è la tecnica, bensì la domanda su cosa portare con sé. Tre regole:

- **Nessuna chiave privata sul server.** In `authorized_keys` sono presenti esclusivamente chiavi pubbliche. Le chiavi private restano sui dispositivi finali.
- **Non copiare indiscriminatamente le credenziali.** File locali sensibili come `.credentials.json` non vanno trasferiti senza verifica sul VPS. Autenticarsi invece nuovamente sul server.
- **Prima la configurazione in una cartella di migrazione.** Non scrivere direttamente nei percorsi di configurazione attivi le memorie e la configurazione Claude esistenti, ma trasferirle inizialmente in una cartella di migrazione separata e verificare lì cosa deve davvero essere mantenuto. Ciò che non serve più, come vecchie voci MCP o impostazioni orfane, viene lasciato deliberatamente indietro invece di migrare senza controllo.

## 10. Anteprime web tramite tunnel SSH

Per le anteprime web, ad esempio un server di sviluppo locale avviato da Claude, è forte la tentazione di aprire semplicemente un'altra porta. Non andrebbe fatto. Ogni porta aperta aggiuntiva aumenta la superficie d'attacco. L'anteprima passa invece attraverso un tunnel di porta SSH crittografato: il servizio ascolta solo localmente sul server e SSH lo inoltra al client.

Dal PC rendere raggiungibile un servizio in esecuzione locale sulla porta 4321:

```bash
ssh -p 61417 -L 4321:localhost:4321 claude@SERVER
```

Quindi aprire `http://localhost:4321` nel browser locale. Il traffico passa interamente attraverso la connessione SSH esistente e autenticata, senza dover aprire nel firewall nemmeno una porta aggiuntiva.

## Accesso da iPhone

L'accesso in mobilità funziona con lo stesso modello di sicurezza del PC. Serve solo un client SSH con gestione delle chiavi. Diffusi sono **Termius**, **Blink Shell** e **Secure ShellFish**; tutti possono generare chiavi Ed25519 e memorizzarle nel portachiavi iOS, in parte protette da Face ID.

La procedura corrisponde al passo 3, solo sull'iPhone:

1. Generare nel client SSH una chiave Ed25519 dedicata all'iPhone, senza copiare la chiave del PC. La chiave privata rimane nel portachiavi del dispositivo.
2. Aggiungere la chiave pubblica dell'iPhone come riga aggiuntiva in `~/.ssh/authorized_keys` sul server, con un commento descrittivo (`iphone-15`).
3. Creare la connessione nel client: indirizzo del server, utente `claude`, porta `61417`, chiave dell'iPhone per l'autenticazione.

Proprio per questo conviene avere una chiave separata per ogni dispositivo: se l'iPhone viene smarrito, si elimina sul server l'unica riga `iphone-15` da `authorized_keys`, e il dispositivo resta escluso mentre l'accesso dal PC e tutte le altre chiavi continuano a funzionare senza modifiche.

Dopo la connessione, recuperare la sessione Claude in esecuzione con `tmux attach -t claude` e continuare a lavorare dal punto in cui ci si era fermati alla scrivania. Anche il tunnel di porta del passo 10 funziona da iOS; Termius e Secure ShellFish supportano l'inoltro di porte.

## Checklist

In sintesi, l'intera procedura:

1. Debian 13 installato e completamente aggiornato con `apt full-upgrade`.
2. Utente dedicato `claude` con diritti sudo; il login root diretto non viene più utilizzato.
3. Chiavi Ed25519 protette da passphrase, una per dispositivo, solo chiavi pubbliche in `authorized_keys`.
4. sshd rafforzato: `PermitRootLogin no`, `PasswordAuthentication no`; verificato con `sshd -t` prima della ricarica, sessione esistente lasciata aperta fino al test.
5. SSH sulla porta 61417, impostata su `ssh.socket` con attivazione socket, altrimenti nella configurazione sshd.
6. Firewall del provider: DROP predefinito in ingresso, unica eccezione TCP 61417; traffico in uscita consentito.
7. Superficie d'attacco verificata con `ss -lntup` (solo SSH pubblico, `chronyd` locale) e `systemctl --failed` (nessun errore).
8. Claude Code autenticato nuovamente sul server, eseguito in una sessione `tmux`.
9. Igiene dei dati: nessuna chiave privata né credenziale sul server, configurazione verificata prima tramite una cartella di migrazione.
10. Nessuna porta aggiuntiva; le anteprime web passano attraverso un tunnel SSH.

Dopo questa configurazione, dall'esterno è raggiungibile solo SSH sulla porta definita, ed esclusivamente con una chiave protetta da passphrase. Claude Code viene eseguito indipendentemente dal dispositivo finale in `tmux`; le anteprime web rimangono accessibili tramite tunnel SSH senza aprire una porta aggiuntiva.

## Fonti

1.  [Manuale OpenSSH – sshd_config(5)](https://man.openbsd.org/sshd_config): riferimento per tutte le direttive sshd, incluse `PermitRootLogin`, `PasswordAuthentication` e `PubkeyAuthentication`.

2.  [Wiki Debian – SSH](https://wiki.debian.org/SSH): indicazioni specifiche per Debian sulla configurazione SSH, inclusi i file drop-in in `/etc/ssh/sshd_config.d/`.

3.  [systemd.socket(5) – freedesktop.org](https://www.freedesktop.org/software/systemd/man/latest/systemd.socket.html): funzionamento dell'attivazione socket e della direttiva `ListenStream=`, rilevante per il cambio della porta SSH su Debian 13.

4.  [ss(8) – Manpage iproute2](https://man7.org/linux/man-pages/man8/ss.8.html): opzioni di `ss` per elencare i socket in ascolto con processo e indirizzo di binding.

5.  [Claude Code – Documentazione ufficiale](https://docs.claude.com/en/docs/claude-code/overview): installazione, autenticazione e utilizzo di Claude Code.
