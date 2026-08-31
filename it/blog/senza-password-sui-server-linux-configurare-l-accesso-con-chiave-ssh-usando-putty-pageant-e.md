---
title: "Senza password sui server Linux: configurare l'accesso con chiave SSH usando PuTTY, Pageant e altri strumenti"
navTitle: "PuTTY senza password"
description: "Chi accede ogni giorno ai server Linux come amministratore inserisce nome utente e password a ogni connessione. Una coppia di chiavi SSH riduce tutto a un doppio clic: generare la chiave con PuTTYgen, memorizzare la chiave pubblica sul server e caricarla in Pageant. La stessa chiave funziona in WinSCP, MobaXterm e OpenSSH, e chi lo desidera può arrivare direttamente nella shell di un account di servizio."
date: "2026-08-28"
kategorie: "Linux e SSH"
timeToRead: "9 min di lettura"
themen:
  - totemomail
  - windows-client
produkte:
  - "totemomail"
  - "uebergreifend"
protokolle:
  - "ssh"
  - "haertung"
slug: "senza-password-sui-server-linux-configurare-l-accesso-con-chiave-ssh-usando-putty-pageant-e"
translationId: "article-9f94fa6eb8b95bcf"
aiPrompt: |
  Du bist mein Linux- und SSH-Assistent. Hilf mir Schritt für Schritt, einen passwortlosen SSH-Login von Windows auf meine Linux-Server einzurichten: 1. Ein Ed25519-Schlüsselpaar mit PuTTYgen erzeugen und den Public Key in authorized_keys eintragen. 2. Die PuTTY-Session mit Schlüsseldatei und Auto-login username vervollständigen und Pageant mit Autostart einrichten. 3. Optional ein Remote command hinterlegen, das mich direkt in die Shell eines Service-Accounts bringt, samt minimaler NOPASSWD-Regel unter /etc/sudoers.d. Weise mich auf typische Fehler hin: Key im falschen Home-Verzeichnis, mehrzeiliger Public Key, falsche Berechtigungen, sudoers-Befehl stimmt nicht exakt mit dem Remote command überein.
translationOf: putty-ssh-login-service-account-shell
url: https://rafaelpfister.ch/it/blog/senza-password-sui-server-linux-configurare-l-accesso-con-chiave-ssh-usando-putty-pageant-e
translationSourceHash: e95b27e9a86f59dfb0808afee63664493f5961b983f807f31ef9ee7a36f6fb3e
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:36:53.742Z
translationReview: automatic
---

# Senza password sui server Linux: configurare l'accesso con chiave SSH usando PuTTY, Pageant e altri strumenti

Chi accede ogni giorno ai server Linux come amministratore ripete gli stessi inserimenti a ogni connessione con l'accesso tramite password: nome utente, password e, per gli account di servizio, anche un comando sudo. Con una coppia di chiavi SSH tutto questo non è più necessario. Dopo la configurazione, un doppio clic sulla sessione salvata apre una shell pronta all'uso e la stessa chiave funziona in PuTTY, WinSCP, MobaXterm e nel client OpenSSH di Windows.

Questa guida configura completamente l'accesso senza password da Windows: generare la coppia di chiavi, memorizzare la chiave pubblica sul server, completare la sessione PuTTY e configurare Pageant come agente delle chiavi. Include inoltre la risoluzione del problema più frequente (il server continua a chiedere la password nonostante la chiave) e, come estensione, l'accesso diretto alla shell di un account di servizio come `totemo`.

## Perché usare le chiavi anziché una password

Il vantaggio in termini di comodità è l'effetto più evidente, ma non il più importante. Una chiave Ed25519 è praticamente immune agli attacchi brute-force, mentre una password è forte solo quanto la sua lunghezza e la disciplina nel non riutilizzarla da nessuna parte. Sui server i cui utenti sono passati interamente alle chiavi, l'autenticazione tramite password può essere disattivata completamente nella configurazione di sshd (`PasswordAuthentication no`), rendendo inefficaci i tentativi di accesso automatizzati da Internet. Disattivate l'autenticazione tramite password solo quando l'accesso con chiave funziona in modo verificabile e disponete di una seconda via di accesso (console, seconda chiave).

Il principio è semplice: la chiave privata rimane sul vostro computer Windows, mentre quella pubblica viene memorizzata sul server. All'avvio della connessione, il server verifica che la controparte possieda la chiave privata corrispondente, senza che quest'ultima lasci mai il computer.

## Passaggio 1: generare una coppia di chiavi con PuTTYgen

1. Avviate **PuTTYgen** (parte del pacchetto PuTTY), selezionate **Ed25519** come tipo, fate clic su **Generate** e muovete il mouse sull'area indicata.
2. Inserite una **passphrase** in entrambi i campi e salvate la chiave privata con **Save private key** come file `.ppk`.
3. Copiate integralmente il campo di testo in alto (la riga che inizia con `ssh-ed25519 AAAA...`). È la chiave pubblica nel formato previsto dal server.

Salvate la chiave privata con una passphrase. Senza passphrase, qualsiasi copia del file costituisce un accesso immediato al server; con una passphrase, il file da solo è inutile. Lo svantaggio in termini di comodità viene quasi completamente eliminato da Pageant (passaggio 4). Una chiave senza passphrase è accettabile solo per l'automazione non presidiata, non per l'accesso interattivo.

## Passaggio 2: memorizzare la chiave pubblica sul server

Sul server, dopo aver effettuato l'accesso come utente con cui vi collegherete in futuro:

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo 'ssh-ed25519 AAAA... kommentar' >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
chmod go-w ~
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Comando / Opzione | Effetto |
|---|---|
| `mkdir -p ~/.ssh` | Crea la directory SSH; `-p` evita l'errore se esiste già |
| `chmod 700 ~/.ssh` | Solo il proprietario può leggere, scrivere e accedere alla directory |
| `echo '...' >> ~/.ssh/authorized_keys` | Aggiunge la chiave pubblica come nuova riga al file (`>>` anziché `>`, altrimenti sovrascrivereste le chiavi esistenti) |
| `chmod 600 ~/.ssh/authorized_keys` | Solo il proprietario può leggere e scrivere il file delle chiavi |
| `chmod go-w ~` | Rimuove il permesso di scrittura al gruppo e agli altri dalla directory home |

</details>

L'ultima riga sembra poco appariscente, ma è necessaria: una directory home scrivibile dal gruppo o da chiunque fa sì che il demone SSH ignori silenziosamente la chiave e il server continui a chiedere la password, senza indicarne il motivo.

## Passaggio 3: completare la sessione PuTTY

1. Aprite PuTTY, selezionate la sessione salvata e caricatela con **Load**.
2. In **Connection → SSH → Auth → Credentials**, selezionate il file `.ppk` in **Private key file for authentication**.
3. In **Connection → Data**, inserite il nome utente in **Auto-login username**, altrimenti PuTTY continuerà a richiederlo a ogni connessione.
4. Tornate alla categoria **Session**, selezionate nuovamente il nome della sessione e fate clic su **Save**.

L'errore operativo più frequente: omettere **Load** prima della modifica o **Save** dopo. Senza Load modificate solo le impostazioni predefinite; senza Save la modifica andrà persa al successivo avvio di PuTTY.

## Passaggio 4: Pageant, passphrase una volta per sessione Windows

Pageant è l'agente delle chiavi di PuTTY. Mantiene in memoria la chiave privata decrittata, per cui la passphrase è richiesta solo una volta per sessione Windows:

1. Avviate Pageant (l'icona appare nella barra delle applicazioni).
2. Fate clic con il pulsante destro sull'icona, selezionate **Add Key**, scegliete il file `.ppk` e inserite la passphrase.
3. Da quel momento tutte le connessioni funzioneranno senza richieste fino al riavvio successivo.

Per avviare automaticamente Pageant, create un collegamento nella cartella di avvio automatico (`Win+R`, quindi `shell:startup`) e passate la chiave come argomento:

```text
"C:\Program Files\PuTTY\pageant.exe" "C:\Pfad\zum\schluessel.ppk"
```

Windows chiederà quindi la passphrase una volta dopo l'accesso e il resto della giornata lavorativa proseguirà senza richieste.

## Se il server continua a chiedere la password

La ricerca del problema inizia nell'**Event Log** di PuTTY (clic destro sulla barra del titolo della sessione terminale). Qui viene indicato se la chiave è stata effettivamente proposta:

| Riscontro nell'Event Log | Causa e soluzione |
|---|---|
| Nessuna voce relativa a una chiave pubblica | Il file `.ppk` non è memorizzato nella sessione salvata, oppure è stata modificata la sessione sbagliata. Caricate la sessione, impostate la chiave e salvate. |
| `Server refused our key` | Il server non trova o non accetta la chiave: home errata, formato errato o permessi errati (vedere sotto). |
| `Access granted`, seguito dalla richiesta della password | L'accesso con chiave ha funzionato; la richiesta proviene da un programma successivo, tipicamente sudo. Consultate l'estensione più sotto. |

Le tre cause più frequenti di `Server refused our key`:

- **Chiave nella home errata.** La chiave pubblica deve trovarsi nel file `authorized_keys` dell'utente con cui viene stabilita la connessione. Chi, durante l'inserimento, è già passato a un altro account con `sudo -u` o `su`, crea il file nella sua home anziché nella propria. `whoami` prima dell'inserimento mostra nella home di chi verrà salvata la chiave.
- **Formato errato.** La chiave pubblica deve essere contenuta in un'unica riga in `authorized_keys`, nel formato del campo di testo superiore di PuTTYgen. Il file creato con **Save public key** ha un altro formato, su più righe (`---- BEGIN SSH2 PUBLIC KEY ----`), e non funziona in `authorized_keys`.
- **Permessi.** `~/.ssh` su `700`, `authorized_keys` su `600`, directory home non scrivibile dal gruppo o da chiunque.

Se il riscontro resta poco chiaro, lato server aiuta consultare `/var/log/secure` oppure `journalctl -u sshd`; lì il demone SSH motiva il rifiuto.

## La stessa chiave in altri strumenti

La configurazione sul server è indipendente dallo strumento e la chiave può essere riutilizzata ovunque:

| Strumento | Configurazione |
|---|---|
| **WinSCP** | Usa direttamente i file `.ppk` (Accesso → Avanzate → SSH → Autenticazione) e utilizza automaticamente Pageant se la chiave è caricata lì |
| **MobaXterm** | File `.ppk` in Session settings → SSH → Advanced → Use private key; supporta anche il formato OpenSSH |
| **FileZilla** | Inserite il file `.ppk` in Impostazioni → SFTP oppure lasciate Pageant in esecuzione |
| **OpenSSH (Windows Terminal, `ssh`)** | Richiede il formato OpenSSH: in PuTTYgen esportate tramite **Conversions → Export OpenSSH key** e collocatelo in `~/.ssh/` |

Per il client OpenSSH, un accesso comodo richiede una voce in `~/.ssh/config` sul computer Windows:

```text
Host mailgw
    HostName server.example.com
    User mmuster
    IdentityFile ~/.ssh/id_ed25519
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Host mailgw` | Alias liberamente selezionabile; in seguito è sufficiente il comando `ssh mailgw` |
| `HostName` | Nome effettivo del server o indirizzo IP |
| `User` | Nome utente, corrisponde ad Auto-login username in PuTTY |
| `IdentityFile` | Percorso della chiave privata nel formato OpenSSH |

</details>

## Estensione: arrivare direttamente nella shell dell'account di servizio

Molti server Linux nell'ambito delle applicazioni e della posta non vengono amministrati con l'account personale, bensì tramite un account di servizio: Totemomail viene eseguito con `totemo`, mentre altri gateway e applicazioni dispongono dei propri account funzionali. Dopo l'accesso segue quindi abitualmente lo stesso comando:

```bash
sudo -u totemo /bin/bash -l
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `sudo` | Esegue il comando seguente con altri privilegi e registra la chiamata |
| `-u totemo` | L'utente di destinazione è `totemo` anziché quello predefinito `root` |
| `/bin/bash` | Il comando da eseguire: una nuova shell Bash |
| `-l` | Avvia Bash come login shell; legge il profilo dell'utente di destinazione (`.bash_profile`, variabili d'ambiente, percorsi) |

</details>

L'opzione `-l` è decisiva per gli account di servizio: senza una login shell mancano le variabili d'ambiente del profilo dell'account funzionale, ad esempio i percorsi alle directory applicative o alle installazioni Java, e i comandi specifici dell'applicazione non riescono con messaggi di errore fuorvianti.

L'accesso SSH diretto come account di servizio sarebbe ancora più breve, ma per una buona ragione nella maggior parte dei casi non è nemmeno possibile: gli account funzionali spesso non hanno una password o hanno una login shell bloccata, e un accesso diretto condiviso da più persone eliminerebbe la traccia di audit personalizzata. Passando da sudo, rimane tracciabile quale persona e quando è passata alla shell dell'account di servizio. L'automazione seguente non cambia questo aspetto, ma risparmia solo la digitazione.

### Remote command nella sessione PuTTY

PuTTY può memorizzare per ogni sessione salvata un comando da eseguire dopo l'accesso al posto della normale shell:

1. Caricate la sessione con **Load**.
2. Nel riquadro a sinistra, navigate a **Connection → SSH** (il nodo principale, non un sottopunto).
3. Nel campo **Remote command**, inserite: `sudo -u totemo /bin/bash -l`
4. Salvate in **Session**.

Dovreste conoscere tre particolarità del Remote command:

- Un `exit` nella shell dell'account di servizio chiude completamente la connessione invece di tornare alla vostra shell personale. Il comando sostituisce la login shell, non la annida.
- Se occasionalmente lavorate sul server con l'account personale, salvate una seconda sessione senza Remote command (caricate la sessione, svuotate il campo e salvatela con un nuovo nome).
- Gli strumenti di trasferimento file come WinSCP o `pscp` non sono interessati. Stabiliscono connessioni proprie e ignorano il Remote command della sessione PuTTY.

Se la connessione si chiude subito dopo l'avvio o sudo segnala l'assenza di un terminale: in **Connection → SSH → TTY**, verificate che **Don't allocate a pseudo-terminal** non sia selezionato. Per impostazione predefinita non lo è. Importante per il passaggio 2 sopra: con Remote command attivo, durante l'inserimento della chiave pubblica siete già l'account di servizio; la chiave appartiene invece alla home dell'utente personale.

Nel client OpenSSH, due righe nella voce `Host` svolgono la stessa funzione: `RequestTTY yes` e `RemoteCommand sudo -u totemo /bin/bash -l`.

### sudo senza richiesta della password

Dopo la chiave e il Remote command rimane un solo inserimento: la richiesta della password di sudo. Scompare solo tramite la configurazione sudoers sul server, per la quale sono necessari privilegi root. Su un server aziendale gestito, si tratta di una richiesta all'amministratore del server, non di un'impostazione di PuTTY.

La regola va inserita in un file separato in `/etc/sudoers.d/` e modificata con `visudo`:

```bash
visudo -f /etc/sudoers.d/totemo-shell
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `visudo` | Apre il file sudoers nell'editor e verifica la sintassi prima del salvataggio |
| `-f /etc/sudoers.d/totemo-shell` | Modifica il file indicato anziché il file centrale `/etc/sudoers` |

</details>

Contenuto del file, con il nome utente personale (qui `mmuster` come esempio):

```text
mmuster ALL=(totemo) NOPASSWD: /bin/bash -l
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Componente | Effetto |
|---|---|
| `mmuster` | La regola si applica solo a questo utente |
| `ALL=` | Su tutti gli host (rilevante per file sudoers distribuiti centralmente) |
| `(totemo)` | Solo per comandi come utente di destinazione `totemo`, non come root |
| `NOPASSWD:` | Nessuna richiesta della password per i comandi seguenti |
| `/bin/bash -l` | Esattamente questo comando con esattamente questo argomento, nient'altro |

</details>

Due aspetti determinano se la regola viene applicata e se è giustificabile:

- **Corrispondenza esatta.** Il comando nella regola deve corrispondere al Remote command, argomento incluso. Se in PuTTY è impostato `sudo -u totemo /bin/bash -l`, la regola deve consentire `/bin/bash -l`. Una regola per `/bin/bash` senza `-l` non copre la chiamata con `-l` e sudo continuerà a chiedere la password.
- **Ambito minimo.** La regola consente un solo comando per un solo utente di destinazione. Non concede né privilegi root né comandi arbitrari. In questa forma è una richiesta usuale e giustificabile anche sui server gestiti. La registrazione di sudo resta completamente invariata: ogni passaggio alla shell dell'account di servizio continua a essere presente nel log.

`visudo` non è facoltativo: verifica la sintassi prima del salvataggio. Un errore di battitura scritto direttamente nel file può rendere sudo inutilizzabile per tutti gli utenti del sistema. Per lo stesso motivo, è preferibile usare un file separato in `/etc/sudoers.d/` anziché modificare il file centrale `/etc/sudoers`; resiste agli aggiornamenti dei pacchetti e può essere rimosso senza rischi.

## Il risultato

Dopo la configurazione, l'accesso è così: doppio clic sulla sessione PuTTY e la shell è pronta. Nessun nome utente, nessuna password e, con l'estensione, neppure una richiesta sudo. La situazione della sicurezza non è peggiorata e, sotto un aspetto, è persino migliorata:

| Aspetto | Prima | Dopo |
|---|---|---|
| Autenticazione | Password a ogni accesso | Chiave Ed25519 con passphrase, conservata in Pageant |
| Identità di accesso | Account personale | Account personale invariato |
| Registrazione sudo | Ogni passaggio nel log | Ogni passaggio invariato nel log |
| Ambito NOPASSWD | Nessuno | Un comando, un utente di destinazione, nessun root |

## Fonti

1.  [PuTTY User Manual, capitolo 4: Configuring PuTTY](https://the.earth.li/~sgtatham/putty/0.83/htmldoc/Chapter4.html): Documentazione delle impostazioni della sessione, tra cui Remote command (pannello SSH), Auto-login username (pannello Data) e pseudo-terminale (pannello TTY).

2.  [PuTTY User Manual, capitolo 8: Using public keys for SSH authentication](https://the.earth.li/~sgtatham/putty/0.83/htmldoc/Chapter8.html): PuTTYgen, tipi di chiavi, passphrase, esportazione OpenSSH e inserimento della chiave pubblica sul server.

3.  [PuTTY User Manual, capitolo 9: Using Pageant for authentication](https://the.earth.li/~sgtatham/putty/0.83/htmldoc/Chapter9.html): Funzionamento dell'agente, caricamento delle chiavi e avvio con una chiave come argomento della riga di comando.

4.  [Manuale ssh_config(5)](https://man.openbsd.org/ssh_config.5): Configurazione client del client OpenSSH, inclusi alias host, IdentityFile, RequestTTY e RemoteCommand.

5.  [Manuale sudoers(5)](https://www.sudo.ws/docs/man/sudoers.man/): Sintassi delle regole sudoers, specifica Runas e tag NOPASSWD.

6.  [Manuale sshd(8), sezione AUTHORIZED_KEYS FILE FORMAT](https://man.openbsd.org/sshd.8): Formato del file authorized_keys e requisiti relativi ai permessi dei file.
