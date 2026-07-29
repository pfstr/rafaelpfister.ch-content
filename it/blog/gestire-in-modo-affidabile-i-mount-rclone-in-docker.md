---
title: "Gestire in modo affidabile i mount Rclone in Docker"
navTitle: "Rclone in Docker"
description: "Affinché un mount FUSE da un container funzioni anche sull’host e in altri container, devono collaborare la propagazione dei mount, AppArmor e il ripristino dopo i guasti."
date: "2026-07-26"
kategorie: "Rclone"
timeToRead: "9 min di lettura"
themen:
  - rclone
related:
  - paperless-dokumente-clouddienst-auslagern
slug: "gestire-in-modo-affidabile-i-mount-rclone-in-docker"
translationOf: "rclone-mount-in-docker-container"
url: "https://rafaelpfister.ch/it/blog/gestire-in-modo-affidabile-i-mount-rclone-in-docker"
translationId: article-a08b15399e144547
translationModel: gpt-5.6-terra
translatedAt: 2026-07-29T07:01:53.722Z
translationReview: automatic
translationSourceHash: 9b1f0ebdc53ebc1f61e127ca462d0b92c4e48e717c4ac91778c59fa1f6915823
---

Un mount Rclone viene eseguito in un container Docker, ma deve essere disponibile anche sull’host e in altri container. A questo scopo, gli eventi di mount devono attraversare più namespace. Una singola opzione Compose non basta.

In un test pratico con Ubuntu 25.10, kernel 6.17 e Docker 29.6 si sono verificati tre errori indipendenti: Docker ha declassato silenziosamente `rshared`, AppArmor ha bloccato `fusermount3` e un container consumer è rimasto agganciato al vecchio mount dopo un riavvio. Il caso d’uso concreto era un [archivio cloud per Paperless-ngx](/blog/paperless-dokumente-clouddienst-auslagern); gli stessi meccanismi valgono anche per altri strumenti FUSE come sshfs.

## 1. La sorgente host deve essere essa stessa `shared`

Affinché un mount dal container raggiunga l’host, il bind necessita della propagazione `rshared`:

```yaml
    volumes:
      - type: bind
        source: /srv/storage/media
        target: /data
        bind:
          propagation: rshared
```

`rshared` funziona solo se la sorgente del bind sull’host **è essa stessa un mountpoint con propagazione shared**. Una normale directory non soddisfa questo requisito. Docker non segnala comunque alcun errore, ma usa silenziosamente una propagazione più debole. Lo si riconosce in `/proc/self/mountinfo` all’interno del container:

```text
1938 2077 8:2 /srv/storage/media /data rw,relatime master:1 - ext4 /dev/sda2 rw
```

`master:1` significa propagazione slave: i mount entrano dall’host, ma non escono mai. Corretto sarebbe `shared:N`. La soluzione è eseguire un bind della sorgente su se stessa e contrassegnarlo come shared:

```bash
mount --bind /srv/storage/media /srv/storage/media
mount --make-shared /srv/storage/media
```

Perché sopravviva a un riavvio, va inserito in una unit systemd con `Before=docker.service`. Verifica: `findmnt -no PROPAGATION /srv/storage/media` deve restituire `shared`.

## 2. AppArmor controlla `fusermount3` anche nel container

Con la propagazione corretta è arrivata la sorpresa successiva. Il mount sul percorso condiviso continuava a fallire:

```text
NOTICE: mount helper error: fusermount3: mount failed: Permission denied
CRITICAL: Fatal error: failed to mount FUSE fs: fusermount: exit status 1
```

I consueti privilegi aggiuntivi del container non cambiavano nulla: né `CAP_SYS_ADMIN` e `/dev/fuse` né `unconfined` o persino `--privileged`. Un mount tmpfs funzionava sulla stessa destinazione e FUSE funzionava su altri percorsi. Solo il log di audit del kernel ha mostrato la causa effettiva:

```text
audit: type=1400 apparmor="DENIED" operation="mount" class="mount"
  info="failed mntpnt match" error=-13 profile="fusermount3"
  name="/data/documents/originals/" fstype="fuse.rclone"
```

Ubuntu fornisce un **profilo AppArmor per il binario `fusermount3`** che consente mount FUSE soltanto su una lista positiva di pattern di mountpoint. Questo profilo si applica anche a fusermount3 **nel container**. È decisivo il percorso come lo vede il container:

```text
mount fstype=@{fuse_types} ... -> @{HOME}/**/,
mount fstype=@{fuse_types} ... -> /mnt/{,**/},
mount fstype=@{fuse_types} ... -> /media/**/,
mount fstype=@{fuse_types} ... -> /tmp/**/,
```

`/data` non è nella lista, e nemmeno `/srv`. Il fatto che il container venga eseguito senza confinamento non aiuta: il profilo è associato al file eseguibile, non al container.

La soluzione sfrutta il fatto che solo fusermount3 è soggetto al profilo, mentre un normale `mount --bind` non lo è: montare FUSE su un percorso **consentito** e pubblicarlo da lì sul percorso condiviso tramite bind.

```sh
rclone mount remote:percorso /mnt/inner/documenti --allow-other --vfs-cache-mode full &
# attendere finché il mount risponde, poi:
mount --bind /mnt/inner/documenti /data/documenti
```

Il bind è una normale chiamata mount(2) e, come ogni altra, si propaga all’host tramite il percorso shared. È stato possibile verificarlo fino a un secondo container, che poteva leggere i file come uid 1000. `--allow-other` è obbligatorio non appena un utente diverso da quello che effettua il mount accede ai file; nel container Rclone, a tal fine, `user_allow_other` deve essere presente in `/etc/fuse.conf` (già così nell’immagine ufficiale).

## 3. I consumer necessitano di `rslave`

La terza insidia riguarda il lato opposto. Se il processo Rclone muore e il mount viene ricreato, l’host lo vede subito. Un container che ha incluso il percorso normalmente tramite bind, invece, non lo vede:

```text
ls: cannot access '/usr/src/app/media': Transport endpoint is not connected
```

Per impostazione predefinita Docker usa `rprivate` per i bind mount: un mount che viene creato sull’host **dopo** l’avvio del container non raggiunge mai il suo namespace di mount. Il container rimane agganciato al mount FUSE morto finché non viene ricreato. La soluzione richiede una riga:

```yaml
    volumes:
      - type: bind
        source: /srv/storage/media
        target: /usr/src/app/media
        bind:
          propagation: rslave
```

Con `rslave` l’host inoltra al container i nuovi eventi di mount. Nel test, dopo che un mount è stato terminato forzatamente e ricreato, il consumer ha rivisto tutti i file **senza un proprio riavvio**. Il contatore dei riavvii è rimasto a zero.

## Ripristino senza intervento manuale

Dai tre elementi risulta un modello complessivo robusto, che non richiede un demone watchdog:

1. Il container di mount controlla i propri mount in un ciclo. Se uno non risponde più, termina con un codice di errore.
2. `restart: unless-stopped` fa sì che Docker riavvii il container.
3. All’avvio, il container rimuove innanzitutto **i mount orfani della precedente esecuzione**: un bind morto sul percorso di destinazione bloccherebbe altrimenti una nuova pubblicazione e dall’host un utente senza privilegi non può rimuoverlo. Nel container è possibile farlo e l’umount si propaga verso l’esterno:

```sh
while grep -q " /data/documenti " /proc/self/mountinfo; do
    umount -l /data/documenti 2>/dev/null || break
done
```

4. Quindi effettua normalmente il mount e lo pubblica; i consumer con `rslave` adottano automaticamente il mount aggiornato.

Nel test, l’intera catena ha richiesto 160 secondi: il processo Rclone è stato terminato, l’errore rilevato, il container riavviato, il mount orfano rimosso e il nuovo mount nuovamente pubblicato. Nel frattempo il container consumer ha continuato a funzionare e ha notato soltanto una breve interruzione.

Chi esegue Rclone direttamente **sull’host** tramite systemd evita i primi due problemi e necessita soltanto di `rslave` nei container consumer. Il container aggiuntivo conviene soprattutto se l’host deve rimanere libero da installazioni di Rclone oppure se si vogliono gestire più mount in modo uniforme. In tal caso, tutti e tre i livelli devono essere configurati consapevolmente.

## Fonti

1.  [Docker: Bind mounts: configurare la propagazione dei bind](https://docs.docker.com/engine/storage/bind-mounts/#configure-bind-propagation): le modalità di propagazione rprivate, rslave e rshared e il loro comportamento predefinito.

2.  [Documentazione del kernel: Shared Subtrees](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt): la propagazione dei mount del kernel Linux su cui si basano le opzioni bind di Docker.

3.  [Rclone mount](https://rclone.org/commands/rclone_mount/): modalità della cache VFS, --allow-other e i limiti del mount FUSE.

4.  [Documentazione AppArmor (Ubuntu)](https://documentation.ubuntu.com/server/how-to/security/apparmor/): come i profili sono associati ai file eseguibili; il profilo fusermount3 si trova in /etc/apparmor.d/fusermount3.
