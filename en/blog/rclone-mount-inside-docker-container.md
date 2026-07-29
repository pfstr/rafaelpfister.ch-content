---
title: "Operating Rclone mounts reliably in Docker"
navTitle: "Rclone in Docker"
description: "For a FUSE mount from a container to work on the host and in other containers, mount propagation, AppArmor and recovery after failures must work together."
date: "2026-07-26"
kategorie: "Rclone"
timeToRead: "9 min read"
themen:
  - rclone
related:
  - paperless-dokumente-clouddienst-auslagern
translationOf: "rclone-mount-in-docker-container"
slug: "rclone-mount-inside-docker-container"
url: "https://rafaelpfister.ch/en/blog/rclone-mount-inside-docker-container"
translationId: article-a08b15399e144547
translationModel: gpt-5.6-terra
translatedAt: 2026-07-29T07:00:06.870Z
translationReview: automatic
translationSourceHash: 9b1f0ebdc53ebc1f61e127ca462d0b92c4e48e717c4ac91778c59fa1f6915823
---

An Rclone mount runs in a Docker container but also needs to be available on the host and in other containers. To achieve this, mount events must traverse several namespaces. A single Compose option is not enough.

In a practical test with Ubuntu 25.10, kernel 6.17 and Docker 29.6, three independent issues arose: Docker silently downgraded `rshared`, AppArmor blocked `fusermount3`, and a consuming container continued to hold on to the old mount after a restart. The specific use case was [cloud storage for Paperless-ngx](/blog/paperless-dokumente-clouddienst-auslagern); the same mechanisms also apply to other FUSE tools such as sshfs.

## 1. The host source must itself be `shared`

For a mount from the container to reach the host, the bind needs `rshared` propagation:

```yaml
    volumes:
      - type: bind
        source: /srv/storage/media
        target: /data
        bind:
          propagation: rshared
```

`rshared` only works if the bind source on the host is **itself a mount point with shared propagation**. An ordinary directory does not meet this requirement. Docker does not report an error, however, and silently uses weaker propagation instead. This can be identified in `/proc/self/mountinfo` inside the container:

```text
1938 2077 8:2 /srv/storage/media /data rw,relatime master:1 - ext4 /dev/sda2 rw
```

`master:1` means slave propagation: mounts enter from the host but never leave it. `shared:N` would be correct. The solution is to bind the source to itself and mark it as shared:

```bash
mount --bind /srv/storage/media /srv/storage/media
mount --make-shared /srv/storage/media
```

For this to survive a reboot, it belongs in a systemd unit with `Before=docker.service`. To check: `findmnt -no PROPAGATION /srv/storage/media` must return `shared`.

## 2. AppArmor also checks `fusermount3` in the container

With propagation configured correctly, the next surprise emerged. Mounting on the shared path still failed:

```text
NOTICE: mount helper error: fusermount3: mount failed: Permission denied
CRITICAL: Fatal error: failed to mount FUSE fs: fusermount: exit status 1
```

The usual additional container permissions made no difference: neither `CAP_SYS_ADMIN` and `/dev/fuse` nor `unconfined` or even `--privileged`. A tmpfs mount worked at the same target, and FUSE worked on other paths. Only the kernel audit log revealed the actual cause:

```text
audit: type=1400 apparmor="DENIED" operation="mount" class="mount"
  info="failed mntpnt match" error=-13 profile="fusermount3"
  name="/data/documents/originals/" fstype="fuse.rclone"
```

Ubuntu includes an **AppArmor profile for the `fusermount3` binary** that only permits FUSE mounts on an allowlist of mount-point patterns. This profile also applies to fusermount3 **inside the container**. What matters is the path as seen by the container:

```text
mount fstype=@{fuse_types} ... -> @{HOME}/**/,
mount fstype=@{fuse_types} ... -> /mnt/{,**/},
mount fstype=@{fuse_types} ... -> /media/**/,
mount fstype=@{fuse_types} ... -> /tmp/**/,
```

`/data` is not on the list, nor is `/srv`. Running the container unconfined does not help: the profile is attached to the executable file, not the container.

The workaround takes advantage of the fact that only fusermount3 is subject to the profile, while an ordinary `mount --bind` is not: mount FUSE on an **allowed** path and publish it from there to the shared path via a bind mount.

```sh
rclone mount remote:path /mnt/inner/documents --allow-other --vfs-cache-mode full &
# wait until the mount responds, then:
mount --bind /mnt/inner/documents /data/documents
```

The bind is a normal mount(2) call and, like any other, propagates to the host via the shared path. This could be verified as far as a second container, which was able to read the files as uid 1000. `--allow-other` is mandatory whenever a user other than the one mounting accesses the files; the Rclone container must have `user_allow_other` in `/etc/fuse.conf` for this (already the case in the official image).

## 3. Consumers need `rslave`

The third pitfall affects the other side. If the Rclone process dies and the mount is rebuilt, the host sees it immediately. However, a container that has included the path as a normal bind does not:

```text
ls: cannot access '/usr/src/app/media': Transport endpoint is not connected
```

Docker uses `rprivate` for bind mounts by default: a mount that is created on the host **after** the container starts never reaches its mount namespace. The container remains attached to the dead FUSE mount until it is recreated. The solution takes one line:

```yaml
    volumes:
      - type: bind
        source: /srv/storage/media
        target: /usr/src/app/media
        bind:
          propagation: rslave
```

With `rslave`, the host passes new mount events into the container. In the test, after a mount was forcibly terminated and rebuilt, the consumer saw all files again **without restarting itself**. Its restart counter remained at zero.

## Recovery without manual intervention

The three building blocks form a robust overall pattern that does not require a watchdog daemon:

1. The mount container checks its mounts in a loop. If one no longer responds, it exits with an error code.
2. `restart: unless-stopped` makes Docker restart the container.
3. On startup, the container first removes **orphaned mounts from its previous incarnation**: a dead bind at the target path would otherwise block republishing, and an unprivileged user cannot remove it from the host. It can be done inside the container, and the unmount propagates outwards:

```sh
while grep -q " /data/documents " /proc/self/mountinfo; do
    umount -l /data/documents 2>/dev/null || break
done
```

4. Then mount and publish normally; consumers with `rslave` automatically pick up the fresh mount.

In the test, the entire chain took 160 seconds: the Rclone process was terminated, the failure detected, the container restarted, the orphaned mount removed and the new mount published again. The consuming container continued running throughout and noticed only a brief interruption.

Those who run Rclone directly **on the host** via systemd avoid the first two issues and only need `rslave` on consuming containers. The additional container is particularly worthwhile when the host should remain free of Rclone installations or several mounts are to be managed consistently. In that case, all three layers must be configured deliberately.

## Sources

1.  [Docker: Bind mounts: configure bind propagation](https://docs.docker.com/engine/storage/bind-mounts/#configure-bind-propagation): the rprivate, rslave and rshared propagation modes and their default behaviour.

2.  [Kernel documentation: Shared Subtrees](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt): Linux kernel mount propagation, on which Docker's bind options are based.

3.  [Rclone mount](https://rclone.org/commands/rclone_mount/): VFS cache modes, --allow-other and the limitations of the FUSE mount.

4.  [AppArmor documentation (Ubuntu)](https://documentation.ubuntu.com/server/how-to/security/apparmor/): how profiles are bound to executable files; the fusermount3 profile is located at /etc/apparmor.d/fusermount3.
