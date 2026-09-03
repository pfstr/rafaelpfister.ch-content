---
title: "Eseguire mount Rclone in Docker in modo affidabile"
navTitle: "Rclone in Docker"
description: "Affinché un mount FUSE da un container funzioni anche sull'host e in altri container, devono collaborare la propagazione dei mount, AppArmor e il ripristino dopo i guasti."
date: "2026-07-26"
kategorie: "Rclone"
timeToRead: "9 min di lettura"
themen:
  - rclone
related:
  - paperless-dokumente-clouddienst-auslagern
slug: "gestire-in-modo-affidabile-i-mount-rclone-in-docker"
translationOf: "rclone-mount-in-docker-container"
translationId: article-a08b15399e144547
translationModel: gpt-5.6-terra
translatedAt: 2026-09-03T08:29:05.281Z
translationReview: automatic
translationSourceHash: 5cba6faedde80db33a3f35e999758cb09a93ccb85cfb9021a45026a99173bb26
url: https://rafaelpfister.ch/it/blog/gestire-in-modo-affidabile-i-mount-rclone-in-docker
---

Un mount Rclone viene eseguito in un container Docker, ma deve essere disponibile anche sull'host e in altri container. Per questo, gli eventi di mount devono attraversare più namespace. Una sola opzione Compose non è sufficiente.

In un test pratico con Ubuntu 25.10, kernel 6.17 e Docker 29.6 si sono verificati tre errori indipendenti: Docker ha degradato `rshared` senza segnalarlo, AppArmor ha bloccato `fusermount3`, e un container consumer è rimasto collegato al vecchio mount dopo un riavvio. Il caso d'uso concreto era un [archivio cloud per Paperless-ngx](/blog/paperless-dokumente-clouddienst-auslagern); gli stessi meccanismi valgono anche per altri strumenti FUSE come sshfs.

## 1. La sorgente host deve essere essa stessa `shared`

Affinché un mount dal container raggiunga l'host, il bind necessita della propagazione `rshared`:

```yaml
    volumes:
      - type: bind
        source: /srv/storage/media
        target: /data
        bind:
          propagation: rshared
```

`rshared` funziona solo se la sorgente del bind sull'host è **essa stessa un mountpoint con propagazione Shared**. Una normale directory non soddisfa questo requisito. Docker non segnala comunque alcun errore, ma usa silenziosamente una propagazione più debole. Lo si riconosce in `/proc/self/mountinfo` all'interno del container:

```text
1938 2077 8:2 /srv/storage/media /data rw,relatime master:1 - ext4 /dev/sda2 rw
```

`master:1` significa propagazione slave: i mount entrano dall'host, ma non escono mai. Corretto sarebbe `shared:N`. La soluzione è un bind della sorgente su sé stessa, contrassegnato come shared:

```bash
mount --bind /srv/storage/media /srv/storage/media
mount --make-shared /srv/storage/media
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `--bind quelle ziel` | Collega una directory a un secondo percorso; qui a sé stessa, rendendo la directory un mountpoint indipendente |
| `--make-shared pfad` | Imposta la propagazione di questo mountpoint su shared, in modo che gli eventi di mount vengano inoltrati in entrambe le direzioni |

</details>

Perché sopravviva a un riavvio, va inserito in una unit systemd con `Before=docker.service`. Verifica: `findmnt -no PROPAGATION /srv/storage/media` deve restituire `shared`.

## 2. AppArmor verifica `fusermount3` anche nel container

Con la propagazione corretta è comparso il problema successivo. Il mount sul percorso condiviso continuava a fallire:

```text
NOTICE: mount helper error: fusermount3: mount failed: Permission denied
CRITICAL: Fatal error: failed to mount FUSE fs: fusermount: exit status 1
```

I consueti permessi aggiuntivi del container non cambiavano nulla: né `CAP_SYS_ADMIN` e `/dev/fuse` né `unconfined` o persino `--privileged`. Un mount tmpfs funzionava sulla stessa destinazione, e FUSE funzionava su altri percorsi. Solo il log di audit del kernel ha mostrato la vera causa:

```text
audit: type=1400 apparmor="DENIED" operation="mount" class="mount"
  info="failed mntpnt match" error=-13 profile="fusermount3"
  name="/data/documents/originals/" fstype="fuse.rclone"
```

Ubuntu include un **profilo AppArmor per il binario `fusermount3`** che consente mount FUSE solo su una lista positiva di pattern di mountpoint. Questo profilo si applica anche a fusermount3 **nel container**. Determinante è il percorso come lo vede il container:

```text
mount fstype=@{fuse_types} ... -> @{HOME}/**/,
mount fstype=@{fuse_types} ... -> /mnt/{,**/},
mount fstype=@{fuse_types} ... -> /media/**/,
mount fstype=@{fuse_types} ... -> /tmp/**/,
```

`/data` non è nell'elenco, così come `/srv`. Il fatto che il container venga eseguito senza confinamento non aiuta: il profilo è associato al file eseguibile, non al container.

La soluzione sfrutta il fatto che solo fusermount3 è soggetto al profilo, mentre un normale `mount --bind` non lo è: montare FUSE su un percorso **consentito** e pubblicarlo da lì sul percorso condiviso tramite bind.

```sh
rclone mount remote:pfad /mnt/inner/dokumente --allow-other --vfs-cache-mode full &
# attendere che il mount risponda, quindi:
mount --bind /mnt/inner/dokumente /data/dokumente
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `remote:pfad` | Remote Rclone e relativo percorso da montare |
| `/mnt/inner/dokumente` | Mountpoint sotto `/mnt`, un pattern consentito dal profilo AppArmor |
| `--allow-other` | Consente a utenti diversi da quello che effettua il mount di accedere al mount FUSE |
| `--vfs-cache-mode full` | Memorizza completamente nel cache locale gli accessi in lettura e scrittura |
| `&` | Avvia il mount in background, lasciando libera la shell per il bind |
| `mount --bind quelle ziel` | Pubblica il mount FUSE tramite bind sul percorso condiviso; essendo una chiamata mount(2), non è soggetto al profilo fusermount3 |

</details>

Il bind è una normale chiamata mount(2) e, come ogni altra, si propaga all'host attraverso il percorso Shared. Questo è stato verificato fino a un secondo container, che poteva leggere i file come uid 1000. `--allow-other` è obbligatorio non appena un utente diverso da quello che ha effettuato il mount accede ai file; nel container Rclone, `user_allow_other` deve essere presente in `/etc/fuse.conf` (già così nell'immagine ufficiale).

## 3. I consumer necessitano di `rslave`

Il terzo problema riguarda l'altra parte. Se il processo Rclone si interrompe e il mount viene ricostruito, l'host lo vede immediatamente. Un container che ha incluso normalmente il percorso tramite bind, invece, non lo vede:

```text
ls: cannot access '/usr/src/app/media': Transport endpoint is not connected
```

Per impostazione predefinita Docker usa `rprivate` per i bind mount: un mount creato sull'host **dopo** l'avvio del container non raggiunge mai il suo namespace di mount. Il container rimane collegato al mount FUSE non più connesso finché non viene ricreato. La soluzione richiede una sola riga:

```yaml
    volumes:
      - type: bind
        source: /srv/storage/media
        target: /usr/src/app/media
        bind:
          propagation: rslave
```

Con `rslave`, l'host inoltra al container i nuovi eventi di mount. Nel test, dopo un mount terminato forzatamente e ricostruito, il consumer ha rivisto tutti i file **senza un proprio riavvio**. Il contatore dei riavvii è rimasto a zero.

## Ripristino senza intervento manuale

Dai tre elementi risulta un modello complessivo robusto che non richiede un demone watchdog:

1. Il container di mount controlla i propri mount in un ciclo. Se uno non risponde più, termina con un codice di errore.
2. `restart: unless-stopped` fa riavviare il container da Docker.
3. All'avvio, il container rimuove innanzitutto **i mount orfani dell'esecuzione precedente**: un bind orfano sul percorso di destinazione altrimenti bloccherebbe una nuova pubblicazione e un utente non privilegiato non può rimuoverlo dall'host. Nel container è possibile, e l'umount si propaga verso l'esterno:

```sh
while grep -q " /data/dokumente " /proc/self/mountinfo; do
    umount -l /data/dokumente 2>/dev/null || break
done
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `grep -q` | Nessun output; solo il codice di uscita segnala se il percorso è ancora presente come mount in `/proc/self/mountinfo` |
| `umount -l` | Lazy unmount: rimuove immediatamente il mount dall'albero e pulisce i riferimenti solo quando non viene più usato |
| `2>/dev/null` | Sopprime i messaggi di errore di umount |
| `\|\| break` | Termina il ciclo se un umount fallisce, invece di proseguire all'infinito |

</details>

4. Poi monta e pubblica normalmente; i consumer con `rslave` adottano automaticamente il mount aggiornato.

Nel test, l'intera catena ha richiesto 160 secondi: il processo Rclone è stato terminato, l'errore rilevato, il container riavviato, il mount orfano rimosso e il nuovo mount nuovamente pubblicato. Il container consumer è rimasto in esecuzione nel frattempo e ha notato solo una breve interruzione.

Chi esegue Rclone direttamente **sull'host** tramite systemd evita i primi due problemi e necessita solo di `rslave` nei container consumer. Il container aggiuntivo conviene soprattutto se l'host deve rimanere privo di installazioni Rclone o se si vogliono gestire più mount in modo uniforme. In tal caso, tutti e tre i livelli devono essere configurati consapevolmente.

## Fonti

1.  [Docker: Bind mounts: configure bind propagation](https://docs.docker.com/engine/storage/bind-mounts/#configure-bind-propagation): le modalità di propagazione rprivate, rslave e rshared e il loro comportamento predefinito.

2.  [Documentazione del kernel: Shared Subtrees](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt): la propagazione dei mount del kernel Linux su cui si basano le opzioni bind di Docker.

3.  [Rclone mount](https://rclone.org/commands/rclone_mount/): modalità della cache VFS, --allow-other e i limiti del mount FUSE.

4.  [Documentazione AppArmor (Ubuntu)](https://documentation.ubuntu.com/server/how-to/security/apparmor/): come i profili vengono associati ai file eseguibili; il profilo fusermount3 si trova in /etc/apparmor.d/fusermount3.
