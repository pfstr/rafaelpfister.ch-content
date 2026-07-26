---
title: "rclone-Mounts in Docker zuverlässig betreiben"
navTitle: "rclone in Docker"
description: "Damit ein FUSE-Mount aus einem Container auch auf dem Host und in weiteren Containern funktioniert, müssen Mount-Propagation, AppArmor und die Wiederherstellung nach Ausfällen zusammenspielen."
date: "2026-07-26"
kategorie: "rclone"
timeToRead: "9 Min. Lesezeit"
themen:
  - "rclone"
related:
  - "paperless-dokumente-proton-drive-auslagern"
draft: true
slug: "rclone-mount-in-docker-container"
url: "https://rafaelpfister.ch/blog/rclone-mount-in-docker-container"
---

Ein rclone-Mount läuft in einem Docker-Container, soll aber auch auf dem Host und in weiteren Containern verfügbar sein. Dafür müssen Mount-Ereignisse mehrere Namensräume durchqueren. Eine einzelne Compose-Option genügt nicht.

In einem Praxistest mit Ubuntu 25.10, Kernel 6.17 und Docker 29.6 traten drei voneinander unabhängige Fehler auf: Docker stufte `rshared` unbemerkt herunter, AppArmor blockierte `fusermount3`, und ein konsumierender Container hielt nach einem Neustart am alten Mount fest. Der konkrete Anwendungsfall war eine [Cloud-Ablage für Paperless-ngx](/blog/paperless-dokumente-proton-drive-auslagern); dieselben Mechanismen gelten auch für andere FUSE-Werkzeuge wie sshfs.

## 1. Die Host-Quelle muss selbst `shared` sein

Damit ein Mount aus dem Container den Host erreicht, braucht der Bind die Propagation `rshared`:

```yaml
    volumes:
      - type: bind
        source: /srv/storage/media
        target: /data
        bind:
          propagation: rshared
```

`rshared` funktioniert nur, wenn die Bind-Quelle auf dem Host **selbst ein Mountpoint mit Shared-Propagation** ist. Ein gewöhnliches Verzeichnis erfüllt diese Voraussetzung nicht. Docker meldet trotzdem keinen Fehler, sondern verwendet stillschweigend eine schwächere Propagation. Erkennbar ist das in `/proc/self/mountinfo` innerhalb des Containers:

```text
1938 2077 8:2 /srv/storage/media /data rw,relatime master:1 - ext4 /dev/sda2 rw
```

`master:1` heisst slave-Propagation: Mounts kommen vom Host herein, aber nie hinaus. Richtig wäre `shared:N`. Die Lösung ist ein Bind der Quelle auf sich selbst, als shared markiert:

```bash
mount --bind /srv/storage/media /srv/storage/media
mount --make-shared /srv/storage/media
```

Damit das einen Reboot übersteht, gehört es in eine systemd-Unit mit `Before=docker.service`. Kontrolle: `findmnt -no PROPAGATION /srv/storage/media` muss `shared` liefern.

## 2. AppArmor prüft `fusermount3` auch im Container

Mit korrekter Propagation kam die nächste Überraschung. Der Mount auf den geteilten Pfad scheiterte weiterhin:

```text
NOTICE: mount helper error: fusermount3: mount failed: Permission denied
CRITICAL: Fatal error: failed to mount FUSE fs: fusermount: exit status 1
```

Die üblichen zusätzlichen Container-Rechte änderten nichts: weder `CAP_SYS_ADMIN` und `/dev/fuse` noch `unconfined` oder sogar `--privileged`. Ein tmpfs-Mount funktionierte am selben Ziel, und FUSE funktionierte an anderen Pfaden. Erst das Kernel-Audit-Log zeigte die tatsächliche Ursache:

```text
audit: type=1400 apparmor="DENIED" operation="mount" class="mount"
  info="failed mntpnt match" error=-13 profile="fusermount3"
  name="/data/documents/originals/" fstype="fuse.rclone"
```

Ubuntu liefert ein **AppArmor-Profil für die Binary `fusermount3`** mit, das FUSE-Mounts nur auf eine Positivliste von Mountpunkt-Mustern erlaubt. Dieses Profil greift auch für das fusermount3 **im Container**. Entscheidend ist der Pfad, wie ihn der Container sieht:

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

Der Bind ist ein normaler mount(2)-Aufruf und propagiert wie jeder andere über den Shared-Pfad zum Host. Das liess sich bis in einen zweiten Container verifizieren, der die Dateien als uid 1000 lesen konnte. `--allow-other` ist dabei Pflicht, sobald ein anderer Benutzer als der mountende auf die Dateien zugreift; im rclone-Container muss dafür `user_allow_other` in `/etc/fuse.conf` stehen (im offiziellen Image bereits der Fall).

## 3. Konsumenten brauchen `rslave`

Die dritte Falle betrifft die Gegenseite. Stirbt der rclone-Prozess und wird der Mount neu aufgebaut, sieht ihn der Host sofort. Ein Container, der den Pfad ganz normal per Bind eingebunden hat, sieht ihn dagegen nicht:

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

Mit `rslave` reicht der Host neue Mount-Ereignisse in den Container weiter. Im Test sah der Konsument nach einem hart beendeten und neu aufgebauten Mount **ohne eigenen Neustart** wieder alle Dateien. Der Neustartzähler blieb bei null.

## Wiederherstellung ohne manuellen Eingriff

Aus den drei Bausteinen ergibt sich ein robustes Gesamtmuster, das ohne Watchdog-Daemon auskommt:

1. Der Mount-Container prüft seine Mounts in einer Schleife. Antwortet einer nicht mehr, beendet er sich mit Fehlercode.
2. `restart: unless-stopped` lässt Docker den Container neu starten.
3. Beim Start räumt der Container zuerst **verwaiste Mounts aus dem Vorleben** ab: ein toter Bind auf dem Zielpfad würde das erneute Publizieren sonst blockieren, und vom Host aus kann ein unprivilegierter Benutzer ihn nicht entfernen. Im Container geht es, und der umount propagiert nach draussen:

```sh
while grep -q " /data/dokumente " /proc/self/mountinfo; do
    umount -l /data/dokumente 2>/dev/null || break
done
```

4. Danach normal mounten und publizieren; Konsumenten mit `rslave` übernehmen den frischen Mount von selbst.

Im Test dauerte die gesamte Kette 160 Sekunden: Der rclone-Prozess wurde beendet, der Fehler erkannt, der Container neu gestartet, der verwaiste Mount entfernt und der neue Mount wieder veröffentlicht. Der konsumierende Container lief währenddessen weiter und bemerkte nur eine kurze Unterbrechung.

Wer rclone direkt **auf dem Host** per systemd betreibt, umgeht die ersten beiden Probleme und benötigt nur `rslave` an den konsumierenden Containern. Der zusätzliche Container lohnt sich vor allem, wenn der Host frei von rclone-Installationen bleiben oder mehrere Mounts einheitlich verwalten soll. Dann müssen alle drei Ebenen bewusst konfiguriert werden.

## Quellen

1.  [Docker: Bind mounts: configure bind propagation](https://docs.docker.com/engine/storage/bind-mounts/#configure-bind-propagation): die Propagationsmodi rprivate, rslave und rshared und ihr Standardverhalten.

2.  [Kernel-Dokumentation: Shared Subtrees](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt): die Mount-Propagation des Linux-Kernels, auf der Dockers Bind-Optionen aufsetzen.

3.  [rclone mount](https://rclone.org/commands/rclone_mount/): VFS-Cache-Modi, --allow-other und die Grenzen des FUSE-Mounts.

4.  [AppArmor-Dokumentation (Ubuntu)](https://documentation.ubuntu.com/server/how-to/security/apparmor/): wie Profile an ausführbare Dateien gebunden werden; das fusermount3-Profil liegt unter /etc/apparmor.d/fusermount3.
