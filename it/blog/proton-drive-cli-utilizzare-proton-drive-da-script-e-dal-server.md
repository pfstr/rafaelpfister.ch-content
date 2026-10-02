---
title: "Proton Drive CLI: utilizzare Proton Drive da script e dal server"
navTitle: "Proton Drive CLI"
description: "Dal giugno 2026 Proton offre uno strumento ufficiale da riga di comando per Proton Drive. L’articolo descrive i comandi, l’accesso su server senza desktop, le strategie di conflitto per gli script e i limiti rispetto a Rclone."
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
url: https://rafaelpfister.ch/it/blog/proton-drive-cli-utilizzare-proton-drive-da-script-e-dal-server
translationSourceHash: e71e82deea6466312d0d95edc2c96890fc0810219a21308b4cb9346df3f32341
translationModel: gpt-5.6-terra
translatedAt: 2026-10-02T09:58:54.238Z
translationReview: automatic
---

Il 9 giugno 2026 Proton ha pubblicato la **Proton Drive CLI**, uno strumento ufficiale da riga di comando per Windows, macOS e Linux. Si basa sullo stesso SDK delle app Drive ufficiali, utilizza la crittografia end-to-end ed è disponibile come singolo file eseguibile `proton-drive`. Il codice sorgente si trova nel repository SDK pubblico all’indirizzo `cli/`.

La CLI è pensata per operazioni singole e delimitate nel tempo: caricare file dopo una build, eseguire il backup pianificato di una cartella, verificare o revocare condivisioni. Non sincronizza in background e non monta alcun filesystem. Attualmente è disponibile la versione **0.8.0 del 13 agosto 2026**; il numero di versione indica che comandi e opzioni possono ancora cambiare (la versione 0.8.0 ha rinominato le strategie di conflitto con una modifica incompatibile).

La collocazione rispetto alle altre opzioni Linux (Rclone, client desktop annunciato) è illustrata nell’articolo sullo stato [Proton Drive su Linux](/blog/proton-drive-linux-status).

## Panoramica dei comandi

I comandi sono organizzati in gruppi: `proton-drive <gruppe> <befehl> [optionen] [argumente]`. I nomi dei gruppi possono essere abbreviati purché rimangano univoci; per `filesystem` è disponibile anche l’alias `fs`. Senza argomenti si avvia una shell interattiva. La guida completa è fornita da `proton-drive help` oppure `proton-drive <gruppe> <befehl> --help`.

<details class="options-details">
<summary>Panoramica delle opzioni</summary>

| Gruppo / comando | Funzione |
|---|---|
| `auth login` / `auth logout` | Accesso tramite browser; la disconnessione elimina le credenziali locali e le cache |
| `filesystem list <pfad>` | Elenca il contenuto di una cartella; `/` mostra le aree radice |
| `filesystem info` / `size` | Metadati di un elemento o dimensione di una cartella, inclusi i contenuti del cestino |
| `filesystem upload` / `download` | Carica o scarica file e cartelle |
| `filesystem create-folder`, `rename`, `copy`, `move` | Crea, rinomina, copia e sposta cartelle |
| `filesystem trash` / `restore` | Sposta nel cestino o ripristina |
| `filesystem delete` / `empty-trash` | Elimina definitivamente o svuota `/trash` |
| `sharing status <pfad>` | Mostra membri, inviti aperti e impostazioni dei link |
| `sharing invite` / `remove` | Invita persone via e-mail o revoca l’accesso |
| `sharing set-url` / `remove-url` | Crea, modifica o rimuove un link pubblico |
| `sharing leave` / `report` | Abbandona una condivisione ricevuta o la segnala come abuso |
| `invitation list` / `accept` / `reject` | Gestisce gli inviti ricevuti |
| `album …`, `photo timeline`, `photo upload`, `photo download` | Proton Photos: album e cronologia |
| `version` | Mostra le versioni di CLI e SDK |
| `--json` (`-j`) | Output JSON leggibile dalle macchine, per ogni comando |
| `--verbose` (`-v`) | Output di log direttamente nella console |
| `--help` (`-h`) | Guida al rispettivo comando |

</details>

I percorsi in Proton Drive sono sempre percorsi POSIX, anche su Windows. La radice `/` contiene aree virtuali: `/my-files` (file propri), `/devices` (computer sottoposti a backup), `/shared-by-me`, `/shared-with-me`, `/trash` e le aree Photos `/photos`, `/albums`, `/photos-shared-by-me`, `/photos-shared-with-me` e `/photos-trash`.

## Installazione su Linux

Proton mette a disposizione le build su una pagina di download dedicata, ciascuna con checksum SHA-512. Per Linux sono disponibili cinque varianti:

| Build | Utilizzo |
|---|---|
| `linux/x64` | Standard per sistemi x86-64 attuali |
| `linux/x64-baseline` | x86-64 senza AVX2, ad esempio dispositivi NAS e CPU server meno recenti |
| `linux/arm64` | Server ARM e computer a scheda singola con glibc |
| `linux/x64-musl`, `linux/arm64-musl` | Distribuzioni con musl anziché glibc, ad esempio Alpine Linux e immagini container basate su di essa |

Se la build standard si interrompe all’avvio con `Illegal instruction`, alla CPU manca l’estensione AVX2; in tal caso la scelta corretta è la build `x64-baseline`. Il file include il runtime Bun e non richiede ulteriori dipendenze:

```bash
chmod +x proton-drive
sudo install -m 0755 proton-drive /usr/local/bin/proton-drive
proton-drive version
```

Senza diritti di amministratore basta copiare il file in `~/.local/bin`, purché questa directory sia nel `PATH`.

## Accesso, anche su server senza desktop

`auth login` non richiede alcuna password nella riga di comando. La CLI tenta di aprire un browser e mostra inoltre l’URL di accesso. Questo URL può essere aperto **su un altro dispositivo**; il terminale attende finché l’accesso non viene completato lì. L’autenticazione a due fattori avviene normalmente nel browser. In questo modo l’accesso funziona anche tramite SSH su un server senza interfaccia grafica.

```bash
proton-drive auth login
```

Dopo l’accesso riuscito, la CLI salva la sessione, non la password. La sua posizione è determinata dalla variabile d’ambiente `PROTON_DRIVE_CREDENTIALS_STORE`:

| Valore | Posizione di archiviazione |
|---|---|
| `keychain` (predefinito) | Archivio credenziali del sistema operativo: Windows Credential Manager, macOS Keychain, su Linux libsecret (GNOME Keyring, KWallet) |
| `pass` | Voce crittografata GPG `ch.proton.drive/drive-sdk-cli/auth-session` nel gestore di password [pass](https://www.passwordstore.org/) |
| `unsafe_file` | File in chiaro `auth-session.json` nella directory dei dati; secondo Proton, solo per test |

Su un server senza sessione desktop generalmente manca un portachiavi libsecret sbloccato. Per questo caso, dalla versione 0.6.0 esiste l’opzione `pass`. L’utente con cui vengono eseguiti gli script necessita di un password store inizializzato e di una chiave GPG che il `gpg-agent` possa sbloccare senza input interattivo. La sessione deve essere trovata tramite la stessa variabile a ogni esecuzione, quindi deve essere impostata anche nei job Cron e nelle unità systemd:

```bash
export PROTON_DRIVE_CREDENTIALS_STORE=pass
proton-drive auth login
```

Rispetto a Rclone è un progresso: password e chiave TOTP non risiedono sul server e `auth logout` revoca l’accesso. La sessione mantiene tuttavia l’intera portata dell’account. Non esistono limitazioni a singole cartelle o al solo accesso in lettura. Per i processi automatizzati, un account Proton dedicato resta quindi la variante più sicura.

Cache, dati dell’applicazione e log si trovano su Linux nelle directory XDG (`~/.cache/proton-drive-cli`, `~/.local/share/proton-drive-cli`, `~/.local/state/proton-drive-cli`). Con `PROTON_DRIVE_CACHE_DIR` è possibile collocare tutti e tre in un’unica directory, ad esempio per un container con volume montato. Per impostazione predefinita, la CLI scrive i log con livello `DEBUG`; `PROTON_DRIVE_LOG_LEVEL=WARNING` ne riduce la quantità.

## Caricare e scaricare negli script

In modalità interattiva, la CLI chiede cosa fare per ogni conflitto di nome. Negli script ciò non è possibile: con `--json` la richiesta interattiva viene disattivata. Definite quindi sempre esplicitamente la strategia di conflitto per file e cartelle.

```bash
proton-drive filesystem upload --json \
  --file-conflict-strategy create-new-revision \
  --folder-conflict-strategy merge \
  --skip-thumbnails \
  /srv/export/berichte /my-files/backup
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Funzione |
|---|---|
| `--json` (`-j`) | Restituisce il risultato come JSON; disattiva le richieste interattive |
| `--file-conflict-strategy` (`-f`) | Comportamento se esiste un file con lo stesso nome: `create-new-revision` (nuova versione del file esistente), `rename` (aggiunge un suffisso), `replace` (file remoto nel cestino, carica quello locale), `skip` |
| `--folder-conflict-strategy` (`-d`) | Comportamento se la cartella esiste: `merge` (unisce i contenuti), `rename`, `replace`, `skip` |
| `--skip-thumbnails` (`-t`) | Non genera anteprime; riduce il tempo di elaborazione delle immagini |
| `/srv/export/berichte` | Sorgente locale; sono possibili più sorgenti |
| `/my-files/backup` | Cartella di destinazione in Proton Drive (ultimo argomento) |

</details>

`create-new-revision` è la scelta adatta per i backup: Proton Drive conserva le versioni precedenti di un file e, dalla versione 0.7.0, la CLI salta automaticamente i file dal contenuto invariato. La CLI non effettua però un confronto: i file eliminati localmente restano in Proton Drive. Chi necessita di un mirror con eliminazioni dipende ancora da `rclone sync`.

Il download funziona in modo speculare. Le strategie differiscono perché qui viene sovrascritta la parte locale:

```bash
proton-drive filesystem download --json \
  --file-conflict-strategy remove \
  --folder-conflict-strategy merge \
  /my-files/backup/berichte /srv/restore
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Funzione |
|---|---|
| `--file-conflict-strategy` (`-f`) | `rename`, `remove` (elimina il file locale e scarica la versione remota) o `skip` |
| `--folder-conflict-strategy` (`-d`) | `merge`, `rename`, `remove` o `skip` |
| `/my-files/backup/berichte` | Sorgente in Proton Drive; sono possibili più sorgenti |
| `/srv/restore` | Cartella di destinazione locale (ultimo argomento) |

</details>

La CLI salta Proton Docs e Proton Sheets durante il download; al momento non possono essere esportati come file.

Un upload regolare può essere pianificato con un timer systemd o Cron. L’output JSON può quindi essere analizzato con `jq`, ad esempio per inviare una notifica al monitoraggio.

## Gestire le condivisioni

Per l’offboarding o gli audit, la gestione delle condivisioni è spesso più utile del trasferimento file. `sharing status` mostra per un elemento tutti i membri, gli inviti aperti e le impostazioni di un link pubblico:

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
<summary>Spiegazione delle opzioni</summary>

| Opzione | Funzione |
|---|---|
| `--user` (`-u`) | Indirizzo e-mail della persona invitata; può essere specificato più volte |
| `--role` (`-r`) | Ruolo, predefinito `viewer`; altri ruoli secondo `--help` (ad es. `editor`) |
| `--message` (`-m`) | Messaggio nell’e-mail di invito; viene inviato in **testo semplice** |
| `--include-node-name` (`-n`) | Include il nome dell’elemento nell’e-mail di invito; anch’esso in testo semplice |
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
<summary>Spiegazione delle opzioni</summary>

| Opzione | Funzione |
|---|---|
| `--role` | `viewer` (predefinito) o `editor` |
| `--password` | Password propria per il link |
| `--expiration` | Data di scadenza in formato ISO (`JJJJ-MM-TT`) |
| `/my-files/…/bericht.pdf` | Elemento per il quale viene creato o modificato il link |

</details>

Una password passata sulla riga di comando rimane nella cronologia della shell ed è visibile nell’elenco dei processi durante l’esecuzione. Negli script dovrebbe quindi provenire da una variabile o da un archivio di segreti. `sharing remove-url` rimuove nuovamente il link senza interessare i membri diretti; `sharing remove --user …` revoca l’accesso a singole persone.

## In lavorazione: Takeout

Dal 10 settembre 2026, il repository SDK contiene un ulteriore comando `takeout run`. Esporta una copia offline dell’account in una cartella locale, a scelta con `--include my-files`, `devices`, `photos` e `revisions` (tutte le versioni precedenti dei file). Per ogni cartella scrive un file `manifest.json` che descrive l’esportazione; non modifica nulla nell’account. Il comando non è ancora incluso nella versione pubblicata 0.8.0. Non appena sarà disponibile, rappresenterà il modo più ovvio per un backup locale completo dei contenuti di Proton Drive.

## Limiti rispetto a Rclone

| Requisito | Proton Drive CLI 0.8.0 | Rclone (backend `protondrive`) |
|---|---|---|
| Supportato ufficialmente | Sì, da Proton, open source | No, reverse engineering, beta |
| Accesso | Browser, anche su un altro dispositivo; sessione nell’archivio credenziali o in `pass` | Password e chiave TOTP nel file di configurazione |
| Caricare, scaricare | Sì, con strategie di conflitto e versionamento | Sì |
| Mirror con eliminazioni (`sync`) | No | Sì |
| Montare filesystem (FUSE) | No | Sì |
| Condivisioni, inviti, link | Sì | No |
| Proton Photos | Sì | No |
| Accesso con portata limitata | No | No |

Per backup, artefatti di build e gestione delle condivisioni, la CLI è la scelta migliore perché è supportata ufficialmente e non richiede una password memorizzata. Per un mount, come quello necessario ad esempio per un [archivio documentale Paperless](/blog/paperless-dokumente-clouddienst-auslagern), e per i mirror con eliminazioni, Rclone resta per ora necessario. In entrambi i casi, la maggiore lacuna rimane la stessa: Proton non offre alcun accesso macchina limitabile a singole cartelle o alla sola lettura.

## Fonti

1.  [Proton: Introducing Proton Drive CLI](https://proton.me/blog/proton-drive-cli): l’annuncio del 9 giugno 2026 con casi d’uso e output JSON.

2.  [Proton Support: Using Proton Drive CLI](https://proton.me/support/drive-cli): guida per download, accesso e comandi di base.

3.  [Proton Drive CLI: Downloads](https://proton.me/download/drive/cli/index.html): versione attuale 0.8.0, tutte le build per piattaforma con checksum SHA-512.

4.  [ProtonDriveApps/sdk: cli/README.md](https://github.com/ProtonDriveApps/sdk/blob/main/cli/README.md): variabili d’ambiente, percorsi di archiviazione, credential store e nota sulla build `x64-baseline`.

5.  [ProtonDriveApps/sdk: cli/CHANGELOG.md](https://github.com/ProtonDriveApps/sdk/blob/main/cli/CHANGELOG.md): cronologia delle versioni da 0.4.2 a 0.8.0, inclusi il supporto per `pass` (0.6.0) e il salto dei file invariati (0.7.0).

6.  [ProtonDriveApps/sdk: cli/src/commands](https://github.com/ProtonDriveApps/sdk/tree/main/cli/src/commands): codice sorgente dei comandi con opzioni, strategie di conflitto e il comando Takeout non ancora pubblicato.

7.  [Proton for Business: Proton Drive CLI](https://proton.me/business/drive/cli): scenari d’uso aziendali di Proton, ad esempio revocare le condivisioni alla partenza dei dipendenti.

8.  [Rclone: Proton Drive](https://rclone.org/protondrive/): il backend della community con funzioni di mount e sincronizzazione come confronto.
