---
title: "Rclone-Mount im Docker-Container: Propagation, AppArmor und der Fehler „Transport endpoint is not connected“"
navTitle: "Rclone in Docker"
description: "Ein FUSE-Mount aus einem Docker-Container soll auf dem Host und in anderen Containern sichtbar sein — und Neustarts überleben. Drei dokumentierte Fallen: stillschweigend degradierte rshared-Propagation, das AppArmor-Profil für fusermount3 und Container, die an toten Mounts festhalten."
date: "2026-07-25"
kategorie: "Rclone"
timeToRead: "9 min to read"
themen:
  - "rclone"
related:
  - "paperless-dokumente-clouddienst-auslagern"
slug: "rclone-mount-in-docker-container"
url: "https://rafaelpfister.ch/blog/rclone-mount-in-docker-container"
---

Ein Rclone-Mount soll in einem Docker-Container laufen, aber ausserhalb nutzbar sein: auf dem Host und in anderen Containern, die die Dateien konsumieren. Das klingt nach einer Zeile Compose — tatsächlich stecken darin drei Fallen, von denen zwei kaum dokumentiert sind. Dieser Artikel zeigt alle drei mit den konkreten Fehlerbildern, wie sie bei einem Praxistest auf Ubuntu 25.10 (Kernel 6.17, Docker 29.6) aufgetreten sind.

Der Anwendungsfall dahinter: eine [Dokumentenablage für Paperless-ngx in einem Clouddienst](/blog/paperless-dokumente-clouddienst-auslagern). Die Erkenntnisse gelten aber für jeden containerisierten FUSE-Mount, auch mit sshfs oder anderen Werkzeugen.

## Falle 1: Docker degradiert rshared stillschweigend

Damit ein Mount aus dem Container den Host erreicht, braucht der Bind die Propagation `rshared`:

```yaml
    volumes:
      - type: bind
        source: /srv/storage/media
        target: /data
        bind:
          propagation: rshared
```

Was die Dokumentation verschweigt: `rshared` funktioniert nur, wenn die Bind-Quelle auf dem Host **selbst ein Mountpoint mit shared-Propagation** ist. Ein gewöhnliches Verzeichnis erfüllt das nicht — und Docker meldet dann keinen Fehler, sondern stuft die Propagation kommentarlos herunter. Sichtbar wird das nur in `/proc/self/mountinfo` im Container:

```text
1938 2077 8:2 /srv/storage/media /data rw,relatime master:1 - ext4 /dev/sda2 rw
```

`master:1` heisst slave-Propagation: Mounts kommen vom Host herein, aber nie hinaus. Richtig wäre `shared:N`. Die Lösung ist ein Bind der Quelle auf sich selbst, als shared markiert:

```bash
mount --bind /srv/storage/media /srv/storage/media
mount --make-shared /srv/storage/media
```

Damit das einen Reboot übersteht, gehört es in eine systemd-Unit mit `Before=docker.service`. Kontrolle: `findmnt -no PROPAGATION /srv/storage/media` muss `shared` liefern.

## Falle 2: das AppArmor-Profil für fusermount3

Mit korrekter Propagation kam die nächste Überraschung. Der Mount auf den geteilten Pfad scheiterte weiterhin:

```text
NOTICE: mount helper error: fusermount3: mount failed: Permission denied
CRITICAL: Fatal error: failed to mount FUSE fs: fusermount: exit status 1
```

Bemerkenswert dabei: Es lag **nicht** an den Container-Rechten. `CAP_SYS_ADMIN`, `/dev/fuse`, AppArmor `unconfined` für den Container, sogar `--privileged` — der Fehler blieb identisch. Gleichzeitig funktionierte auf demselben Pfad ein tmpfs-Mount problemlos, und der FUSE-Mount funktionierte auf anderen Pfaden. Die Auflösung stand im Kernel-Audit-Log:

```text
audit: type=1400 apparmor="DENIED" operation="mount" class="mount"
  info="failed mntpnt match" error=-13 profile="fusermount3"
  name="/data/documents/originals/" fstype="fuse.rclone"
```

Ubuntu liefert ein **AppArmor-Profil für die Binary `fusermount3`** mit, das FUSE-Mounts nur auf eine Positivliste von Mountpunkt-Mustern erlaubt — und dieses Profil greift auch für das fusermount3 **im Container**, gematcht am Pfad, wie ihn der Container sieht:

```text
mount fstype=@{fuse_types} ... -> @{HOME}/**/,
mount fstype=@{fuse_types} ... -> /mnt/{,**/},
mount fstype=@{fuse_types} ... -> /media/**/,
mount fstype=@{fuse_types} ... -> /tmp/**/,
```

`/data` steht nicht auf der Liste, `/srv` auch nicht. Dass der Container unconfined läuft, hilft nichts: Das Profil hängt an der ausführbaren Datei, nicht am Container.

Der Ausweg nutzt aus, dass nur fusermount3 dem Profil unterliegt, ein gewöhnlicher `mount --bind` aber nicht: FUSE auf einen **erlaubten** Pfad mounten und von dort per Bind auf den geteilten Pfad publizieren.

```sh
rclone mount remote:pfad /mnt/inner/dokumente --allow-other --vfs-cache-mode full &
# warten bis der Mount antwortet, dann:
mount --bind /mnt/inner/dokumente /data/dokumente
```

Der Bind ist ein normaler mount(2)-Aufruf, propagiert wie jeder andere über den shared Pfad zum Host — verifiziert bis in einen zweiten Container, der die Dateien als uid 1000 lesen konnte. `--allow-other` ist dabei Pflicht, sobald ein anderer Benutzer als der mountende auf die Dateien zugreift; im Rclone-Container muss dafür `user_allow_other` in `/etc/fuse.conf` stehen (im offiziellen Image bereits der Fall).

## Falle 3: Konsumenten halten an toten Mounts fest

Die dritte Falle betrifft die Gegenseite. Stirbt der Rclone-Prozess und wird der Mount neu aufgebaut, sieht ihn der Host sofort — ein Container, der den Pfad ganz normal per Bind eingebunden hat, aber nicht:

```text
ls: cannot access '/usr/src/app/media': Transport endpoint is not connected
```

Docker verwendet für Bind-Mounts standardmässig `rprivate`: Ein Mount, der auf dem Host **nach** dem Container-Start entsteht, erreicht dessen Mount-Namensraum nie. Der Container bleibt am toten FUSE-Mount hängen, bis er neu erstellt wird. Die Lösung kostet eine Zeile:

```yaml
    volumes:
      - type: bind
        source: /srv/storage/media
        target: /usr/src/app/media
        bind:
          propagation: rslave
```

Mit `rslave` reicht der Host neue Mount-Ereignisse in den Container weiter. Im Test sah der Konsument nach einem hart beendeten und neu aufgebauten Mount **ohne eigenen Neustart** wieder alle Dateien — der Neustartzähler blieb bei null.

## Selbstheilung als Muster

Aus den drei Bausteinen ergibt sich ein robustes Gesamtmuster, das ohne Watchdog-Daemon auskommt:

1. Der Mount-Container prüft seine Mounts in einer Schleife. Antwortet einer nicht mehr, beendet er sich mit Fehlercode.
2. `restart: unless-stopped` lässt Docker den Container neu starten.
3. Beim Start räumt der Container zuerst **verwaiste Mounts aus dem Vorleben** ab — ein toter Bind auf dem Zielpfad würde das erneute Publizieren sonst blockieren, und vom Host aus kann ein unprivilegierter Benutzer ihn nicht entfernen. Im Container geht es, und der umount propagiert nach draussen:

```sh
while grep -q " /data/dokumente " /proc/self/mountinfo; do
    umount -l /data/dokumente 2>/dev/null || break
done
```

4. Danach normal mounten und publizieren; Konsumenten mit `rslave` übernehmen den frischen Mount von selbst.

Im Test war die komplette Kette — Rclone-Prozess getötet, Erkennung, Container-Neustart, Aufräumen, Remount, Republish — nach 160 Sekunden durch, ohne dass der Konsumenten-Container etwas davon mitbekam ausser einer kurzen Lücke.

Ein Wort zur Einordnung: Wer den Mount **auf dem Host** per systemd betreibt, umgeht Falle 1 und 2 vollständig und braucht nur `rslave` auf der Konsumentenseite. Der Container-Weg lohnt sich, wenn der Host frei von Rclone-Installationen bleiben soll oder mehrere Mounts zentral verwaltet werden — dann führt an den drei Fallen kein Weg vorbei.

## Quellen

1.  [Docker: Bind mounts — configure bind propagation](https://docs.docker.com/engine/storage/bind-mounts/#configure-bind-propagation) — die Propagationsmodi rprivate, rslave und rshared und ihr Standardverhalten.

2.  [Kernel-Dokumentation: Shared Subtrees](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt) — die Mount-Propagation des Linux-Kernels, auf der Dockers Bind-Optionen aufsetzen.

3.  [Rclone mount](https://rclone.org/commands/rclone_mount/) — VFS-Cache-Modi, --allow-other und die Grenzen des FUSE-Mounts.

4.  [AppArmor-Dokumentation (Ubuntu)](https://documentation.ubuntu.com/server/how-to/security/apparmor/) — wie Profile an ausführbare Dateien gebunden werden; das fusermount3-Profil liegt unter /etc/apparmor.d/fusermount3.
