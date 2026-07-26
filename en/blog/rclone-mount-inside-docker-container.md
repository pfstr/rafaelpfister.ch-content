---
title: "rclone mounts inside Docker containers: propagation, AppArmor and the error \"Transport endpoint is not connected\""
navTitle: "rclone in Docker"
description: "A FUSE mount from inside a Docker container should be visible on the host and in other containers — and survive restarts. Three documented traps: silently downgraded rshared propagation, the fusermount3 AppArmor profile, and containers clinging to dead mounts."
date: "2026-07-26"
kategorie: "rclone"
timeToRead: "9 min to read"
themen:
  - "rclone"
related:
  - "offloading-paperless-documents-to-proton-drive"
translationOf: "rclone-mount-in-docker-container"
slug: "rclone-mount-inside-docker-container"
url: "https://rafaelpfister.ch/en/blog/rclone-mount-inside-docker-container"
---

An rclone mount is supposed to run in a Docker container but be usable outside of it: on the host and in other containers that consume the files. That sounds like one line of compose — in reality it hides three traps, two of which are barely documented. This article shows all three with the concrete error signatures, as they occurred during a hands-on test on Ubuntu 25.10 (kernel 6.17, Docker 29.6).

The use case behind it: a [document store for Paperless-ngx in a cloud service](/en/blog/offloading-paperless-documents-to-proton-drive). The findings apply to any containerised FUSE mount, though, including sshfs and similar tools.

## Trap 1: Docker silently downgrades rshared

For a mount from the container to reach the host, the bind needs `rshared` propagation:

```yaml
    volumes:
      - type: bind
        source: /srv/storage/media
        target: /data
        bind:
          propagation: rshared
```

What the documentation glosses over: `rshared` only works if the bind source on the host is **itself a mount point with shared propagation**. A plain directory does not qualify — and Docker then reports no error, but silently downgrades the propagation. It only becomes visible in `/proc/self/mountinfo` inside the container:

```text
1938 2077 8:2 /srv/storage/media /data rw,relatime master:1 - ext4 /dev/sda2 rw
```

`master:1` means slave propagation: mounts come in from the host, but never out. Correct would be `shared:N`. The fix is a bind of the source onto itself, marked shared:

```bash
mount --bind /srv/storage/media /srv/storage/media
mount --make-shared /srv/storage/media
```

To survive a reboot, this belongs in a systemd unit with `Before=docker.service`. Check: `findmnt -no PROPAGATION /srv/storage/media` must print `shared`.

## Trap 2: the AppArmor profile for fusermount3

With propagation fixed, the next surprise followed. The mount onto the shared path kept failing:

```text
NOTICE: mount helper error: fusermount3: mount failed: Permission denied
CRITICAL: Fatal error: failed to mount FUSE fs: fusermount: exit status 1
```

Remarkably, container privileges were **not** the cause. `CAP_SYS_ADMIN`, `/dev/fuse`, AppArmor `unconfined` for the container, even `--privileged` — the error stayed identical. At the same time a tmpfs mount worked fine on the very same path, and the FUSE mount worked on other paths. The resolution sat in the kernel audit log:

```text
audit: type=1400 apparmor="DENIED" operation="mount" class="mount"
  info="failed mntpnt match" error=-13 profile="fusermount3"
  name="/data/documents/originals/" fstype="fuse.rclone"
```

Ubuntu ships an **AppArmor profile for the `fusermount3` binary** that only permits FUSE mounts on an allow-list of mount point patterns — and this profile also applies to the fusermount3 **inside the container**, matched against the path as the container sees it:

```text
mount fstype=@{fuse_types} ... -> @{HOME}/**/,
mount fstype=@{fuse_types} ... -> /mnt/{,**/},
mount fstype=@{fuse_types} ... -> /media/**/,
mount fstype=@{fuse_types} ... -> /tmp/**/,
```

`/data` is not on the list, `/srv` is not either. The container running unconfined does not help: the profile attaches to the executable, not to the container.

The way out exploits the fact that only fusermount3 is subject to the profile, while a plain `mount --bind` is not: mount FUSE on an **allowed** path and publish it from there via bind onto the shared path.

```sh
rclone mount remote:path /mnt/inner/documents --allow-other --vfs-cache-mode full &
# wait until the mount answers, then:
mount --bind /mnt/inner/documents /data/documents
```

The bind is a regular mount(2) call and propagates like any other through the shared path to the host — verified all the way into a second container that could read the files as uid 1000. `--allow-other` is mandatory as soon as any user other than the mounting one accesses the files; the rclone container needs `user_allow_other` in `/etc/fuse.conf` for that (already the case in the official image).

## Trap 3: consumers cling to dead mounts

The third trap concerns the other side. If the rclone process dies and the mount is re-created, the host sees it immediately — a container that binds the path the ordinary way does not:

```text
ls: cannot access '/usr/src/app/media': Transport endpoint is not connected
```

Docker uses `rprivate` for bind mounts by default: a mount created on the host **after** the container started never reaches its mount namespace. The container stays stuck on the dead FUSE mount until it is recreated. The fix costs one line:

```yaml
    volumes:
      - type: bind
        source: /srv/storage/media
        target: /usr/src/app/media
        bind:
          propagation: rslave
```

With `rslave` the host passes new mount events into the container. In the test, the consumer saw all files again after a hard-killed and re-created mount **without a restart of its own** — the restart counter stayed at zero.

## Self-healing as a pattern

The three building blocks combine into a robust overall pattern that needs no watchdog daemon:

1. The mount container checks its mounts in a loop. If one stops answering, it exits with an error code.
2. `restart: unless-stopped` makes Docker restart the container.
3. At startup the container first clears **orphaned mounts from its previous life** — a dead bind on the target path would otherwise block republishing, and an unprivileged user cannot remove it from the host. Inside the container it works, and the umount propagates outwards:

```sh
while grep -q " /data/documents " /proc/self/mountinfo; do
    umount -l /data/documents 2>/dev/null || break
done
```

4. Then mount and publish normally; consumers with `rslave` pick the fresh mount up by themselves.

In the test the complete chain — rclone process killed, detection, container restart, cleanup, remount, republish — took 160 seconds, with the consumer container noticing nothing beyond a short gap.

For perspective: whoever runs the mount **on the host** via systemd avoids traps 1 and 2 entirely and only needs `rslave` on the consumer side. The container route pays off when the host should stay free of rclone installations or several mounts are managed centrally — then there is no way around the three traps.

## Sources

1.  [Docker: Bind mounts — configure bind propagation](https://docs.docker.com/engine/storage/bind-mounts/#configure-bind-propagation) — the propagation modes rprivate, rslave and rshared and their default behaviour.

2.  [Kernel documentation: Shared Subtrees](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt) — the Linux kernel's mount propagation that Docker's bind options build on.

3.  [rclone mount](https://rclone.org/commands/rclone_mount/) — VFS cache modes, --allow-other and the limits of the FUSE mount.

4.  [AppArmor documentation (Ubuntu)](https://documentation.ubuntu.com/server/how-to/security/apparmor/) — how profiles attach to executables; the fusermount3 profile lives at /etc/apparmor.d/fusermount3.
