---
title: "Proton Drive CLI: utilizzare Proton Drive da script e server"
navTitle: "Proton Drive CLI"
description: "Da giugno 2026, Proton offre uno strumento ufficiale da riga di comando per Proton Drive. L'articolo descrive i comandi, l'accesso su server senza desktop, le strategie di conflitto per gli script e i limiti rispetto a Rclone."
date: "2026-10-01"
kategorie: "Proton Drive"
timeToRead: "9 min di lettura"
themen:
  - proton-drive
produkte:
  - "proton-drive"
protokolle:
  - "storage"
  - "backup-dr"
related:
  - proton-drive-linux-status
  - rclone-mount-in-docker-container
slug: "proton-drive-cli-utilizzare-proton-drive-da-script-e-dal-server"
translationId: "article-97376b998fdaec4c"
translationOf: proton-drive-cli
translationSourceHash: ce73cccdecdcdea611f1ead28872dab00bc33a7106e2eaefd51bee4241d8631c
translationModel: gpt-5.6-terra
translatedAt: 2026-10-27T09:15:39.814Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/proton-drive-cli-utilizzare-proton-drive-da-script-e-dal-server
---

Il 9 giugno 2026 Proton ha pubblicato la **Proton Drive CLI**, uno strumento ufficiale da riga di comando per Windows, macOS e Linux. Si basa sullo stesso SDK delle app ufficiali di Drive, cifra end-to-end ed è disponibile come singolo file eseguibile `proton-drive`. Il codice sorgente si trova nel repository SDK pubblico all'indirizzo `cli/`.

La CLI è pensata per singole operazioni eseguite in momenti specifici: caricare file dopo una build, eseguire il backup pianificato di una cartella, verificare o revocare condivisioni. Non sincronizza in background né monta un filesystem. La versione attuale è la **0.9.0 del 5 ottobre 2026**; il numero di versione indica che comandi e opzioni possono ancora cambiare (la versione 0.8.0 ha rinominato le strategie di conflitto con una modifica incompatibile).

Il posizionamento rispetto alle altre opzioni Linux (Rclone, client desktop annunciato) è illustrato nell'articolo sullo stato [Proton Drive su Linux](/blog/proton-drive-linux-status).

## Panoramica dei comandi

I comandi sono organizzati in gruppi: `proton-drive <gruppe> <befehl> [optionen] [argumente]`. I nomi dei gruppi possono essere abbreviati finché restano univoci; per `filesystem` è disponibile anche l'alias `fs`. Senza argomenti viene avviata una shell interattiva. La guida completa è disponibile con `proton-drive help` oppure `proton-drive <gruppe> <befehl> --help`.

<details class="options-details">
<summary>Panoramica delle opzioni</summary>

| Gruppo / comando | Funzione |
|---|---|
| `auth login` / `auth logout` | Accesso tramite browser; la disconnessione elimina dati di accesso e cache locali |
| `filesystem list <pfad>` | Elencare il contenuto di una cartella; `/` mostra le aree radice |
| `filesystem info` / `size` | Metadati di un elemento o dimensione di una cartella, inclusi i contenuti del cestino |
| `filesystem upload` / `download` | Caricare o scaricare file e cartelle |
| `filesystem create-folder`, `rename`, `copy`, `move` | Creare, rinominare, copiare e spostare cartelle |
| `filesystem trash` / `restore` | Spostare nel cestino o ripristinare |
| `filesystem delete` / `empty-trash` | Eliminare definitivamente o svuotare `/trash` |
| `sharing status <pfad>` | Visualizzare membri, inviti aperti e impostazioni dei link |
| `sharing invite` / `remove` | Invitare persone via e-mail o revocare l'accesso |
| `sharing set-url` / `remove-url` | Creare, modificare o rimuovere un link pubblico |
| `sharing leave` / `report` | Abbandonare una condivisione ricevuta o segnalarla come abuso |
| `invitation list` / `accept` / `reject` | Gestire gli inviti ricevuti |
| `album …`, `photo timeline`, `photo upload`, `photo download` | Proton Photos: album e cronologia |
| `takeout run` | Esportare una copia offline dell'account in una cartella locale (dalla versione 0.9.0) |
| `version` | Visualizzare le versioni della CLI e dell'SDK |
| `--json` (`-j`) | Output JSON leggibile dalle macchine, per ogni comando |
| `--verbose` (`-v`) | Output dei log direttamente nella console |
| `--help` (`-h`) | Guida sul rispettivo comando |

</details>

I percorsi in Proton Drive sono sempre percorsi POSIX, anche su Windows. La radice `/` contiene aree virtuali: `/my-files` (file propri), `/devices` (computer sottoposti a backup), `/shared-by-me`, `/shared-with-me`, `/trash` e le aree Photos `/photos`, `/albums`, `/photos-shared-by-me`, `/photos-shared-with-me` e `/photos-trash`.

## Installazione su Linux

Proton rende disponibili le build in una propria pagina di download, ciascuna con checksum SHA-512. Per Linux sono disponibili cinque varianti:

| Build | Utilizzo |
|---|---|
| `linux/x64` | Standard per sistemi x86-64 recenti |
| `linux/x64-baseline` | x86-64 senza AVX2, ad esempio dispositivi NAS e CPU server meno recenti |
| `linux/arm64` | Server ARM e computer a scheda singola con glibc |
| `linux/x64-musl`, `linux/arm64-musl` | Distribuzioni con musl anziché glibc, ad esempio Alpine Linux e immagini di container basate su di essa |

Se la build standard si interrompe all'avvio con `Illegal instruction`, alla CPU manca l'estensione AVX2; in tal caso la build `x64-baseline` è la scelta giusta. Il file include il runtime Bun e non richiede ulteriori dipendenze:

```bash
chmod +x proton-drive
sudo install -m 0755 proton-drive /usr/local/bin/proton-drive
proton-drive version
```

Senza diritti di amministratore è sufficiente copiare il file in `~/.local/bin`, a condizione che questa directory sia nel `PATH`.

## Accesso, anche su server senza desktop

`auth login` non richiede una password sulla riga di comando. La CLI tenta di aprire un browser e stampa anche l'URL di accesso. Questo URL può essere aperto **su un altro dispositivo**; il terminale attende finché l'accesso non viene completato lì. L'autenticazione a due fattori avviene normalmente nel browser. In questo modo l'accesso funziona anche via SSH su un server senza interfaccia grafica.

```bash
proton-drive auth login
```

Dopo l'accesso riuscito, la CLI salva la sessione, non la password. La sua posizione è determinata dalla variabile d'ambiente `PROTON_DRIVE_CREDENTIALS_STORE`:

| Valore | Posizione di archiviazione |
|---|---|
| `keychain` (predefinito) | Archivio delle chiavi del sistema operativo: Windows Credential Manager, macOS Keychain, su Linux libsecret (GNOME Keyring, KWallet) |
| `pass` | Voce cifrata con GPG `ch.proton.drive/drive-sdk-cli/auth-session` nel gestore di password [pass](https://www.passwordstore.org/) |
| `unsafe_file` | File di testo semplice `auth-session.json` nella directory dei dati; secondo Proton, solo per test |

Su un server senza sessione desktop di norma manca un portachiavi libsecret sbloccato. Per questo caso, dalla versione 0.6.0 è disponibile l'opzione `pass`. L'utente con cui vengono eseguiti gli script necessita di un password store inizializzato e di una chiave GPG che `gpg-agent` possa sbloccare senza input interattivo. La sessione deve essere trovata tramite la stessa variabile a ogni chiamata, quindi la variabile deve essere impostata anche nei job Cron e nelle unità systemd:

```bash
export PROTON_DRIVE_CREDENTIALS_STORE=pass
proton-drive auth login
```

Rispetto a Rclone, questo è un progresso: password e chiave TOTP non risiedono sul server e `auth logout` termina l'accesso. Tuttavia, la sessione continua ad avere l'intera portata dell'account. Non esiste una limitazione a singole cartelle o al solo accesso in lettura. Per i processi automatizzati, un account Proton separato resta quindi l'alternativa più sicura.

Cache, dati dell'applicazione e log si trovano su Linux nelle directory XDG (`~/.cache/proton-drive-cli`, `~/.local/share/proton-drive-cli`, `~/.local/state/proton-drive-cli`). Con `PROTON_DRIVE_CACHE_DIR` è possibile collocare tutti e tre in un'unica directory, ad esempio per un container con volume montato. Per impostazione predefinita, la CLI scrive i log con il livello `DEBUG`; `PROTON_DRIVE_LOG_LEVEL=WARNING` ne riduce la quantità.

## Caricamento e download negli script

In modalità interattiva, la CLI chiede cosa fare a ogni conflitto di nome. Negli script non è possibile: con `--json` la richiesta interattiva viene disattivata. Definite quindi sempre esplicitamente la strategia di conflitto per file e cartelle.

```bash
proton-drive filesystem upload --json \
  --file-conflict-strategy create-new-revision \
  --folder-conflict-strategy merge \
  --skip-thumbnails \
  /srv/export/berichte /my-files/backup
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Funzione |
|---|---|
| `--json` (`-j`) | Restituire il risultato come JSON; disattiva le richieste interattive |
| `--file-conflict-strategy` (`-f`) | Comportamento se esiste un file con lo stesso nome: `create-new-revision` (nuova versione del file esistente), `rename` (aggiungere un suffisso), `replace` (file remoto nel cestino, caricare quello locale), `skip` |
| `--folder-conflict-strategy` (`-d`) | Comportamento se la cartella esiste già: `merge` (unire i contenuti), `rename`, `replace`, `skip` |
| `--skip-thumbnails` (`-t`) | Non generare anteprime; riduce il tempo di elaborazione delle immagini |
| `/srv/export/berichte` | Origine locale; sono possibili più origini |
| `/my-files/backup` | Cartella di destinazione in Proton Drive (ultimo argomento) |

</details>

`create-new-revision` è la scelta adatta per i backup: Proton Drive conserva le versioni precedenti di un file e, dalla versione 0.7.0, la CLI salta automaticamente i file con contenuto invariato. La CLI però non effettua un confronto: i file eliminati localmente restano in Proton Drive. Chi necessita di uno specchio con eliminazioni deve continuare a usare `rclone sync`.

Il download funziona in modo speculare. Le strategie differiscono perché qui viene sovrascritto il lato locale:

```bash
proton-drive filesystem download --json \
  --file-conflict-strategy remove \
  --folder-conflict-strategy merge \
  /my-files/backup/berichte /srv/restore
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Funzione |
|---|---|
| `--file-conflict-strategy` (`-f`) | `rename`, `remove` (eliminare il file locale e scaricare la versione remota) oppure `skip` |
| `--folder-conflict-strategy` (`-d`) | `merge`, `rename`, `remove` oppure `skip` |
| `/my-files/backup/berichte` | Origine in Proton Drive; sono possibili più origini |
| `/srv/restore` | Cartella di destinazione locale (ultimo argomento) |

</details>

La CLI salta Proton Docs e Proton Sheets durante il download; al momento non è possibile esportarli come file.

Un caricamento regolare può essere pianificato con un timer systemd o Cron. L'output JSON può poi essere analizzato con `jq`, ad esempio per inviare una notifica al monitoraggio.

## Gestire le condivisioni

Per l'offboarding o gli audit, la gestione delle condivisioni è spesso più utile del trasferimento di file. `sharing status` mostra per un elemento tutti i membri, gli inviti aperti e le impostazioni di un link pubblico:

```bash
proton-drive sharing status --json /my-files/projekte/kunde-a
```

Un invito con diritti di lettura:

```bash
proton-drive sharing invite \
  --user person@example.com \
  --role viewer \
  /my-files/projekte/kunde-a
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Funzione |
|---|---|
| `--user` (`-u`) | Indirizzo e-mail della persona invitata; può essere specificato più volte |
| `--role` (`-r`) | Ruolo, predefinito `viewer`; altri ruoli secondo `--help` (ad es. `editor`) |
| `--message` (`-m`) | Messaggio nell'e-mail di invito; viene inviato in **testo semplice** |
| `--include-node-name` (`-n`) | Includere il nome dell'elemento nell'e-mail di invito; anch'esso in testo semplice |
| `/my-files/projekte/kunde-a` | Elemento da condividere |

</details>

Un link pubblico con password e data di scadenza:

```bash
proton-drive sharing set-url \
  --role viewer \
  --password 'Linkpasswort' \
  --expiration 2026-12-31 \
  /my-files/projekte/kunde-a/bericht.pdf
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Funzione |
|---|---|
| `--role` | `viewer` (predefinito) oppure `editor` |
| `--password` | Password personalizzata per il link |
| `--expiration` | Data di scadenza in formato ISO (`JJJJ-MM-TT`) |
| `/my-files/…/bericht.pdf` | Elemento per il quale viene creato o modificato il link |

</details>

Una password passata sulla riga di comando viene memorizzata nella cronologia della shell ed è visibile nell'elenco dei processi durante l'esecuzione. Negli script dovrebbe quindi provenire da una variabile o da un archivio di segreti. `sharing remove-url` rimuove nuovamente il link senza interessare i membri diretti; `sharing remove --user …` revoca l'accesso a singole persone.

## Takeout: esportazione dell'account

Dalla versione 0.9.0 del 5 ottobre 2026, la CLI include il comando `takeout run`. Esporta una copia offline dell'account in una cartella locale, opzionalmente con `--include my-files`, `devices`, `photos` e `revisions` (tutte le versioni precedenti dei file). Senza almeno un'indicazione `--include` non esporta nulla e `revisions` richiede `my-files` o `devices`. Per ogni cartella scrive un file `manifest.json` che descrive l'esportazione; non modifica nulla nell'account. Il comando non supporta ancora Proton Docs e Proton Sheets. Per un backup locale completo del contenuto di Proton Drive, è la soluzione più ovvia.

## Limiti rispetto a Rclone

| Requisito | Proton Drive CLI 0.9.0 | Rclone (backend `protondrive`) |
|---|---|---|
| Supporto ufficiale | Sì, da Proton, open source | No, reverse engineering, beta |
| Accesso | Browser, anche su un altro dispositivo; sessione nel portachiavi o in `pass` | Password e chiave TOTP nel file di configurazione |
| Caricamento, download | Sì, con strategie di conflitto e versionamento | Sì |
| Specchio con eliminazioni (`sync`) | No | Sì |
| Montare un filesystem (FUSE) | No | Sì |
| Condivisioni, inviti, link | Sì | No |
| Proton Photos | Sì | No |
| Accesso con portata limitata | No | No |

Per backup, artefatti di build e gestione delle condivisioni, la CLI è la scelta migliore perché è supportata ufficialmente e non richiede una password memorizzata. Per un mount, come quello necessario ad esempio per un [archivio documentale Paperless](/blog/paperless-dokumente-clouddienst-auslagern), e per le sincronizzazioni speculari con eliminazioni, Rclone resta per ora necessario. In entrambi i casi, la lacuna maggiore rimane la stessa: Proton non offre un accesso macchina limitabile a singole cartelle o al solo accesso in lettura.

## Cronologia delle modifiche

- **25 ottobre 2026:** aggiornato alla versione 0.9.0 del 5 ottobre 2026. Aggiunto: il comando takeout run è ora pubblicato, incluse le indicazioni sulle necessarie opzioni --include e sul supporto mancante per Proton Docs e Sheets.

## Fonti

1.  [Proton: Introducing Proton Drive CLI](https://proton.me/blog/proton-drive-cli): l'annuncio del 9 giugno 2026 con finalità d'uso e output JSON.

2.  [Proton Support: Using Proton Drive CLI](https://proton.me/support/drive-cli): guida per download, accesso e comandi di base.

3.  [Proton Drive CLI: Downloads](https://proton.me/download/drive/cli/index.html): versione attuale 0.9.0, tutte le build per piattaforma con checksum SHA-512.

4.  [ProtonDriveApps/sdk: cli/README.md](https://github.com/ProtonDriveApps/sdk/blob/main/cli/README.md): variabili d'ambiente, posizioni di archiviazione, credential store e nota sulla build `x64-baseline`.

5.  [ProtonDriveApps/sdk: cli/CHANGELOG.md](https://github.com/ProtonDriveApps/sdk/blob/main/cli/CHANGELOG.md): cronologia delle versioni dalla 0.4.2 alla 0.9.0, tra cui il supporto per `pass` (0.6.0), il salto dei file invariati (0.7.0) e il comando Takeout (0.9.0).

6.  [ProtonDriveApps/sdk: cli/src/commands](https://github.com/ProtonDriveApps/sdk/tree/main/cli/src/commands): codice sorgente dei comandi con opzioni, strategie di conflitto e comando Takeout.

7.  [Proton for Business: Proton Drive CLI](https://proton.me/business/drive/cli): scenari d'uso aziendali di Proton, ad esempio revocare le condivisioni quando i dipendenti lasciano l'azienda.

8.  [Rclone: Proton Drive](https://rclone.org/protondrive/): il backend della community con funzionalità di mount e sincronizzazione come confronto.
