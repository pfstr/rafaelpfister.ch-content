---
title: "Rclone 1.76.0: numerose correzioni di sicurezza, fix VFS e nuovi backend"
navTitle: "Rclone 1.76.0"
description: "Rclone 1.76.0 del 6 novembre 2026 chiude vulnerabilità in serve, rc e vari backend, inclusa una falla nell'Auth Proxy di serve sftp con CVSS 8.1, corregge cinque errori nel layer VFS e blocca per impostazione predefinita i reindirizzamenti da HTTPS a HTTP. Gli operatori di mount, servizi serve e rc dovrebbero aggiornare e verificare le impostazioni predefinite modificate."
date: "2026-11-11"
kategorie: "Rclone"
timeToRead: "7 min di lettura"
themen:
  - rclone
produkte:
  - "rclone"
protokolle:
  - "releases"
  - "storage"
slug: "rclone-1-76-0-numerose-correzioni-di-sicurezza-fix-vfs-e-nuovi-backend"
translationId: article-aa7be1fdff77b8cc
translationOf: rclone-1-76-0
url: https://rafaelpfister.ch/it/blog/rclone-1-76-0-numerose-correzioni-di-sicurezza-fix-vfs-e-nuovi-backend
translationSourceHash: 79bd08de9bbf74eb195b3bdc4093b8e92f84cd358eb2a4b0a950ad3109a08e56
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T03:29:49.607Z
translationReview: automatic
---

# Rclone 1.76.0: numerose correzioni di sicurezza, fix VFS e nuovi backend

Rclone 1.76.0 è stato rilasciato il 6 novembre 2026, circa quattro settimane dopo la versione 1.75.2 (9 ottobre 2026). Il changelog riporta venti voci nella sezione Security, tra cui una vulnerabilità nell'Auth Proxy di `rclone serve sftp` con CVSS 8.1, perdite di credenziali durante i reindirizzamenti nei backend Swift, NetStorage, Seafile e Ulozto, nonché diverse evasioni tramite symlink nel backend `local`. Si aggiungono cinque correzioni nel layer VFS, utilizzato da `rclone mount` e dai comandi serve, ampliamenti del backend S3, due nuovi backend (Dosya e Infomaniak kDrive) e il nuovo comando `index`. Alcune impostazioni predefinite cambiano: i reindirizzamenti da HTTPS a HTTP vengono rifiutati, i listener TCP multipath sono disattivati e i modelli glob nei filtri vengono valutati in modo più rigoroso. La release è rivolta a tutti coloro che usano Rclone per mount, sincronizzazione, backup o come server con `rclone serve` e `rclone rcd`. Per informazioni sui mount nei container, consultare l'articolo [Gestire in modo affidabile i mount Rclone in Docker](/blog/rclone-mount-in-docker-container).

## Panoramica delle versioni

| Versione | Data | Tipo | Punti principali |
|---|---|---|---|
| 1.76.0 | 06.11.2026 | Feature release | Correzioni di sicurezza in serve sftp (CVSS 8.1), rc, lib/http, local e quattro backend; Go 1.27.3; cinque fix VFS; protezione dai reindirizzamenti `--allow-https-downgrade`; nuovi backend Dosya e Infomaniak kDrive; comando `index` |

La versione più recente precedente era la 1.75.2. Fino all'11 novembre 2026 non è stata rilasciata alcuna versione patch 1.76.x.

## Correzioni di sicurezza con advisory

Il progetto Rclone ha pubblicato le seguenti vulnerabilità come GitHub Security Advisory il 6 novembre 2026. Tutte indicano la versione 1.76.0 come corretta; nessuna dispone di un ID CVE (stato all'11 novembre 2026, contrassegnate nel changelog come «CVE-PENDING»).

- **GHSA-66qv-66mg-9h74, CVSS 8.1 (High):** In `rclone serve sftp` con `--auth-proxy`, la chiave della cache dell'Auth Proxy non conteneva il tipo di autenticazione. Una chiave pubblica nota poteva quindi essere riutilizzata come password se l'Auth Proxy trattava in modo diverso l'autenticazione tramite chiave e password, l'attaccante e la vittima usavano lo stesso nome utente e lo stesso IP client e la voce in cache era ancora valida (impostazione predefinita: 300 secondi di inattività). Gli stessi IP client possono verificarsi, ad esempio, dietro NAT, un bastion host o un load balancer. Secondo l'advisory, gli altri protocolli serve non sono interessati. Il campo delle versioni interessate indica «>= 1.51.0, <= 1.76.0», mentre quello della versione corretta indica 1.76.0; fa fede la voce relativa alla correzione nel changelog della versione 1.76.0.
- **GHSA-vmgx-g9c5-669p, CVSS 4.2:** In `serve sftp`, `serve ftp`, `serve http` e `serve webdav` con `--auth-proxy`, una revoca nell'Auth Proxy non aveva effetto finché esisteva una risposta memorizzata nella cache. Dalla versione 1.76.0, una revoca ha effetto entro cinque minuti. `serve s3` non è interessato.
- **GHSA-hwj7-wp9g-wrv9, CVSS 5.3:** Attraverso il tempo di risposta della verifica bcrypt, con `--htpasswd` e `--user` era possibile distinguere i nomi utente validi da quelli non validi. Sono interessati, tra gli altri, `serve http`, `serve webdav`, `serve restic`, `serve s3`, `rcd` con `--rc-htpasswd` e l'endpoint Metrics con `--metrics-htpasswd`, nelle versioni da 1.56.0 a 1.75.2. Le password non vengono tuttavia divulgate.
- **GHSA-f8p4-8g4j-q3m2, CVSS 5.5:** Se il server rc veniva eseguito senza autenticazione HTTP, un client con accesso alla porta poteva esaurire la memoria del processo inviando una richiesta JSON di grandi dimensioni a un metodo non autorizzato (Denial of Service, versioni da 1.46.0 a 1.75.2). La versione 1.76.0 limita la dimensione delle richieste e rifiuta le chiamate non autorizzate prima di leggere il body. Come workaround, l'advisory indica l'autenticazione HTTP per rc e un limite al body sul reverse proxy; secondo l'advisory, il binding a localhost non separa i conti utente locali tra loro.
- **GHSA-m97j-7x65-wh93, CVSS 4.8:** I backend Swift, NetStorage e Seafile trasmettevano credenziali nei reindirizzamenti verso un altro host o da HTTPS a HTTP. Lo sfruttamento è possibile solo se l'endpoint configurato reindirizza a sua volta, ad esempio tramite un proxy. Come workaround, l'advisory indica esclusivamente endpoint `https://` e nessun proxy che effettui il downgrade a HTTP.
- **GHSA-3xpx-hjg2-vrjh, CVSS 4.8:** Il backend Ulozto inviava gli header `X-Auth-Token` e `X-User-Token` durante i reindirizzamenti dai link di download. A questo scopo, un attaccante necessita di una posizione di rete dalla quale possa deviare il traffico CDN.
- **GHSA-r5wv-vqqh-2r9c, CVSS 4.0:** Con `--links` e `--metadata`, il backend `local` poteva scrivere attributi `user.*` al di fuori della directory di destinazione tramite un symlink iniettato. Fino all'aggiornamento, l'advisory raccomanda di non usare insieme le due opzioni se terzi possono influenzare la destinazione.

Inoltre, la versione 1.76.0 corregge nel backend `local` altri casi senza advisory dedicato, nei quali `MkdirMetadata`, `Remove`, `Rmdir` e `DirMove` potevano agire tramite symlink al di fuori della directory radice.

## Dipendenze e impostazioni predefinite modificate

Rclone 1.76.0 è compilato con Go 1.27.3. Il changelog cita CVE-2026-111943: Go aggiunge in `crypto/x509` una lista di blocco per certificati intermedi compromessi (CVSS 6.8 secondo la valutazione CISA-ADP; interessate le versioni di Go precedenti alla 1.26.10 e dalla 1.27.0 fino alla 1.27.3 esclusa). Inoltre, `golang.org/x/net` è stato aggiornato alla v0.62.0 e `github.com/andybalholm/brotli` alla v1.2.7, quest'ultimo a causa di CVE-2026-111497; il contesto di questa vulnerabilità è disponibile nell'articolo su [CVE-2026-111497 nel decoder Brotli](/blog/brotli-cve-2026-111497).

I listener TCP multipath sono disattivati per impostazione predefinita dalla versione 1.76.0. Il changelog motiva la scelta con CVE-2026-98432; la voce CVE descrive un use-after-free nello stack TCP del kernel Linux (CVSS 9.8, corretto tra l'altro nelle versioni 6.1.190, 6.6.159, 6.12.113 e 6.18.56). Le correzioni sono fornite dagli aggiornamenti del kernel della distribuzione; Rclone riduce soltanto la propria superficie di attacco. Il changelog non indica un flag per riattivarli.

Panoramica delle modifiche al comportamento predefinito:

- **Reindirizzamenti:** Rclone rifiuta i reindirizzamenti da HTTPS a HTTP. Il nuovo flag `--allow-https-downgrade` permette nuovamente di accettarli.
- **Filtri:** `[!...]` nei modelli glob è ora una classe di caratteri negata, `[^...]` esclude `/`. Prima dell'uso in produzione, verificate con `--dry-run` i file di filtri e le regole `--include`/`--exclude` che utilizzano tali classi.
- **local:** `--copy-links` non segue più i loop di symlink. Senza privilegi root, Rclone imposta ora soltanto i proprietari che un utente normale può impostare. La mancanza di spazio interrompe ora i trasferimenti multi-thread con un errore fatale.
- **Convalida dei parametri:** Vengono rifiutati valori di dimensione troppo grandi, NaN e durate con overflow, così come indicazioni temporali `--bwlimit` senza due punti o con segni.
- **bisync:** `--max-delete` considera `--track-renames`; i valori superiori a 100 vengono rifiutati.
- **operations:** `--immutable` ha ora effetto anche con `copyto`, `moveto` e durante la copia di singoli file.
- **Deprecati:** `--rc-web-gui` e, in `serve s3`, l'opzione `--no-cleanup`, che ora è un alias del nuovo `--directory-markers`.

Dopo l'aggiornamento, è consigliabile eseguire una volta script e unità che avviano Rclone con parametri fissi e verificare eventuali messaggi di errore relativi a valori rifiutati.

## Mount e cache VFS

Per `rclone mount`, il changelog indica una correzione: le istanze VFS non venivano rilasciate in caso di mount non riusciti e durante l'unmount. Nel layer VFS, usato in comune da mount e dai comandi serve, la versione 1.76.0 corregge cinque errori:

- Un truncate riduttivo su un file non memorizzato nella cache andava perso.
- La ridenominazione di una directory durante la scadenza della cache delle directory causava un arresto anomalo.
- Le voci virtuali rimanevano nelle directory eliminate.
- Dopo una ridenominazione che differiva soltanto per maiuscole/minuscole o normalizzazione Unicode, le voci apparivano due volte.
- Gli upload di un file modificato non iniziavano finché il file continuava a essere letto.

L'ultimo punto riguarda i mount con cache VFS nei quali un file modificato continua a essere letto mentre deve essere caricato. In tali configurazioni, dopo l'aggiornamento monitorate la coda di upload ed eseguite un test di scrittura e lettura, come descritto nel [piano di test per i mount Rclone](/blog/cloud-mount-testen-dummy-pdfs).

## Backend S3 e Proton Drive

Nel backend S3, Rclone non mette più in buffer gli upload con Object Lock se il valore MD5 della sorgente è noto. È nuovo il comando backend `link` con override firmati per gli header di risposta. Si aggiungono le regioni Fastly EU Central 1 e Scaleway IT-MIL, diverse regioni Huawei OBS e i provider Autonomys Auto Drive, Clever Cloud Cellar, Infomaniak Swiss Backup, Neon Object Storage e RelAix.Storage. Gli upload multi-thread calcolano i checksum in operazioni di lettura da 1 MiB e, dopo un restore, viene visualizzata l'effettiva Storage Class. Sono stati corretti `rmdir` a livello root con `--s3-directory-markers` e gli URL prefirmati per GCS, Linode e Ceph.

Per `rclone serve s3`, la versione 1.76.0 introduce il flag `--directory-markers` e modifiche a tag, ETag, ETag multipart e scritture condizionali. È stato inoltre risolto il problema per cui i bucket non potevano essere eliminati dopo un upload multipart interrotto.

Il changelog della versione 1.76.0 non contiene alcuna voce relativa al backend Proton Drive. Le ultime correzioni risalgono alla versione 1.75.0 (logica di retry, blocchi in caso di errori di convalida, tempi di modifica errati) e alla 1.75.1 (i file caricati con la 1.75.0 non erano leggibili nelle app Proton). Lo stato del supporto Linux di Proton Drive è descritto nell'articolo [Proton Drive su Linux](/blog/proton-drive-linux-status).

## Ulteriori novità

I due nuovi backend sono Dosya (Kaya) e Infomaniak kDrive. Il nuovo comando `index` scrive elenchi statici di directory in un remote e, nelle esecuzioni successive, può reindicizzare soltanto le directory modificate. Rclone supporta CRC-32C come hash, `sync` può controllare l'ordine dei trasferimenti in base a modelli e la visualizzazione delle directory di `serve http` e `serve webdav` è ordinata e ridisegnata. Secondo il changelog, la GUI non è più considerata beta. Le statistiche non contano più due volte `totalBytes` nelle copie lato server e `serve sftp` segnala un errore con `rmdir` su un file, anziché eliminare il file.

## Procedura consigliata

La versione di destinazione è la 1.76.0. L'aggiornamento è particolarmente urgente sui sistemi con `rclone serve sftp --auth-proxy`, con `rclone rcd` o `--rc` senza autenticazione HTTP, con servizi serve protetti da `--htpasswd` e con remote di tipo Swift, NetStorage, Seafile o Ulozto. Nelle installazioni dai binari ufficiali, `rclone selfupdate` aggiorna il file eseguibile e verifica il download tramite hash e firma; con `rclone selfupdate --check`, Rclone mostra in anticipo quale versione verrebbe installata. Per le installazioni da pacchetti deb o rpm, la documentazione indica `--package deb` oppure `--package rpm`. I mount e i processi serve in esecuzione useranno la nuova versione solo dopo un riavvio. Verificate quindi le regole di filtro con classi di caratteri, le pianificazioni temporali `--bwlimit` e le configurazioni che dipendono da reindirizzamenti HTTP. Il changelog e la pagina di release non indicano Known Issues per la versione 1.76.0 (stato all'11 novembre 2026).

## Fonti

1.  [Rclone: Changelog](https://rclone.org/changelog/): voci relative alla versione 1.76.0 (6 novembre 2026) e alle versioni da 1.75.0 a 1.75.2 con correzioni di sicurezza, modifiche a mount, VFS, S3 e Proton Drive.

2.  [GitHub: rclone v1.76.0](https://github.com/rclone/rclone/releases/tag/v1.76.0): pagina della release con binari e rimando al changelog.

3.  [GitHub: Security Advisories di rclone](https://github.com/rclone/rclone/security/advisories): panoramica degli advisory pubblicati il 6 novembre 2026.

4.  [GHSA-66qv-66mg-9h74](https://github.com/rclone/rclone/security/advisories/GHSA-66qv-66mg-9h74): serve sftp, cache dell'Auth Proxy senza tipo di autenticazione, CVSS 8.1.

5.  [GHSA-vmgx-g9c5-669p](https://github.com/rclone/rclone/security/advisories/GHSA-vmgx-g9c5-669p): la revoca nell'Auth Proxy non ha effetto in presenza di una risposta in cache, CVSS 4.2.

6.  [GHSA-hwj7-wp9g-wrv9](https://github.com/rclone/rclone/security/advisories/GHSA-hwj7-wp9g-wrv9): rilevamento di nomi utente tramite timing bcrypt con `--htpasswd`, CVSS 5.3.

7.  [GHSA-f8p4-8g4j-q3m2](https://github.com/rclone/rclone/security/advisories/GHSA-f8p4-8g4j-q3m2): esaurimento della memoria nel server rc tramite richieste JSON eccessivamente grandi, CVSS 5.5.

8.  [GHSA-m97j-7x65-wh93](https://github.com/rclone/rclone/security/advisories/GHSA-m97j-7x65-wh93): perdite di credenziali nei reindirizzamenti in Swift, NetStorage e Seafile, CVSS 4.8.

9.  [GHSA-3xpx-hjg2-vrjh](https://github.com/rclone/rclone/security/advisories/GHSA-3xpx-hjg2-vrjh): perdita di token nei reindirizzamenti nel backend Ulozto, CVSS 4.8.

10.  [GHSA-r5wv-vqqh-2r9c](https://github.com/rclone/rclone/security/advisories/GHSA-r5wv-vqqh-2r9c): evasione tramite symlink con `--links --metadata` nel backend local, CVSS 4.0.

11.  [CVE.org: CVE-2026-111943](https://www.cve.org/CVERecord?id=CVE-2026-111943): lista di blocco per certificati intermedi compromessi in Go `crypto/x509`, corretto in Go 1.27.3.

12.  [CVE.org: CVE-2026-98432](https://www.cve.org/CVERecord?id=CVE-2026-98432): use-after-free nello stack TCP del kernel Linux, CVSS 9.8, motivo della disattivazione dei listener TCP multipath.

13.  [Rclone: rclone selfupdate](https://rclone.org/commands/rclone_selfupdate/): documentazione sull'autoaggiornamento, `--check` e `--package`.

14.  [CVE-2026-111497: vulnerabilità nel decoder Brotli](/blog/brotli-cve-2026-111497): contesto della vulnerabilità Brotli affrontata dalla versione 1.76.0 tramite l'aggiornamento delle dipendenze.
