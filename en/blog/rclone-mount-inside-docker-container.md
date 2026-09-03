---
title: "Running Rclone mounts reliably in Docker"
navTitle: "Rclone in Docker"
description: "For a FUSE mount from a container to work on the host and in other containers, mount propagation, AppArmor, and recovery after failures must work together."
date: "2026-07-26"
kategorie: "Rclone"
timeToRead: "9 min read"
themen:
  - rclone
related:
  - paperless-dokumente-clouddienst-auslagern
translationOf: "rclone-mount-in-docker-container"
slug: "rclone-mount-inside-docker-container"
translationId: article-a08b15399e144547
translationModel: gpt-5.6-terra
translatedAt: 2026-09-03T08:28:13.512Z
translationReview: automatic
translationSourceHash: 5cba6faedde80db33a3f35e999758cb09a93ccb85cfb9021a45026a99173bb26
url: https://rafaelpfister.ch/en/blog/rclone-mount-inside-docker-container
---

An Rclone mount runs in a Docker container but also needs to be available on the host and in other containers. To achieve this, mount events must traverse multiple namespaces. A single Compose option is not enough.

In a practical test with Ubuntu 25.10, kernel 6.17, and Docker 29.6, three independent issues occurred: Docker silently downgraded `rshared`, AppArmor blocked `fusermount3`, and a consuming container remained attached to the old mount after a restart. The specific use case was a [cloud storage for Paperless-ngx](/blog/paperless-dokumente-clouddienst-auslagern); the same mechanisms also apply to other FUSE tools such as sshfs.

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

`rshared` works only if the bind source on the host is **itself a mount point with shared propagation**. An ordinary directory does not meet this requirement. Docker still does not report an error, instead silently using weaker propagation. This can be seen in `/proc/self/mountinfo` inside the container:

```text
1938 2077 8:2 /srv/storage/media /data rw,relatime master:1 - ext4 /dev/sda2 rw
```

`master:1` means slave propagation: mounts come in from the host but never go out. It should be `shared:N`. The solution is to bind the source to itself and mark it as shared:

```bash
mount --bind /srv/storage/media /srv/storage/media
mount --make-shared /srv/storage/media
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `--bind quelle ziel` | Binds a directory to a second path; here, to itself, making the directory an independent mount point |
| `--make-shared pfad` | Sets this mount point's propagation to shared so that mount events are passed in both directions |

</details>

For this to survive a reboot, it belongs in a systemd unit with `Before=docker.service`. To verify: `findmnt -no PROPAGATION /srv/storage/media` must return `shared`.

## 2. AppArmor checks `fusermount3` even inside the container

With propagation configured correctly, the next issue appeared. Mounting to the shared path still failed:

```text
NOTICE: mount helper error: fusermount3: mount failed: Permission denied
CRITICAL: Fatal error: failed to mount FUSE fs: fusermount: exit status 1
```

The usual additional container permissions made no difference: neither `CAP_SYS_ADMIN` and `/dev/fuse` nor `unconfined` or even `--privileged`. A tmpfs mount worked at the same destination, and FUSE worked at other paths. Only the kernel audit log revealed the actual cause:

```text
audit: type=1400 apparmor="DENIED" operation="mount" class="mount"
  info="failed mntpnt match" error=-13 profile="fusermount3"
  name="/data/documents/originals/" fstype="fuse.rclone"
```

Ubuntu includes an **AppArmor profile for the `fusermount3` binary** that permits FUSE mounts only on an allowlist of mount point patterns. This profile also applies to fusermount3 **inside the container**. What matters is the path as seen by the container:

```text
mount fstype=@{fuse_types} ... -> @{HOME}/**/,
mount fstype=@{fuse_types} ... -> /mnt/{,**/},
mount fstype=@{fuse_types} ... -> /media/**/,
mount fstype=@{fuse_types} ... -> /tmp/**/,
```

`/data` is not on the list, nor is `/srv`. Running the container unconfined does not help: the profile is attached to the executable, not the container.

The workaround takes advantage of the fact that only fusermount3 is subject to the profile, while an ordinary `mount --bind` is not: mount FUSE on an **allowed** path and publish it from there to the shared path using a bind mount.

```sh
rclone mount remote:pfad /mnt/inner/dokumente --allow-other --vfs-cache-mode full &
# wait until the mount responds, then:
mount --bind /mnt/inner/dokumente /data/dokumente
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `remote:pfad` | Rclone remote and path to mount |
| `/mnt/inner/dokumente` | Mount point under `/mnt`, a pattern permitted by the AppArmor profile |
| `--allow-other` | Allows users other than the one mounting it to access the FUSE mount |
| `--vfs-cache-mode full` | Fully buffers read and write operations in the local cache |
| `&` | Starts the mount in the background so the shell is free for the bind mount |
| `mount --bind quelle ziel` | Publishes the FUSE mount to the shared path using a bind mount; as a mount(2) call, it is not subject to the fusermount3 profile |

</details>

The bind mount is a regular mount(2) call and, like any other, propagates to the host through the shared path. This was verified as far as a second container, which could read the files as uid 1000. `--allow-other` is mandatory whenever a user other than the one mounting accesses the files; in the Rclone container, `user_allow_other` must therefore be present in `/etc/fuse.conf` (which is already the case in the official image).

## 3. Consumers need `rslave`

The third issue affects the other side. If the Rclone process crashes and the mount is rebuilt, the host sees it immediately. However, a container that has included the path as a normal bind mount does not:

```text
ls: cannot access '/usr/src/app/media': Transport endpoint is not connected
```

Docker uses `rprivate` by default for bind mounts: a mount created on the host **after** the container starts never reaches its mount namespace. The container remains attached to the no-longer-connected FUSE mount until it is recreated. The solution takes one line:

```yaml
    volumes:
      - type: bind
        source: /srv/storage/media
        target: /usr/src/app/media
        bind:
          propagation: rslave
```

With `rslave`, the host passes new mount events into the container. In the test, after a forcibly terminated and rebuilt mount, the consumer saw all files again **without restarting itself**. The restart counter remained at zero.

## Recovery without manual intervention

The three building blocks result in a robust overall pattern that does not require a watchdog daemon:

1. The mount container checks its mounts in a loop. If one no longer responds, it exits with an error code.
2. `restart: unless-stopped` lets Docker restart the container.
3. When starting, the container first removes **orphaned mounts from the previous run**: an orphaned bind mount at the destination path would otherwise prevent republishing, and an unprivileged user cannot remove it from the host. It can be done in the container, and the unmount propagates outward:

```sh
while grep -q " /data/dokumente " /proc/self/mountinfo; do
    umount -l /data/dokumente 2>/dev/null || break
done
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `grep -q` | No output; only the exit code indicates whether the path is still a mount in `/proc/self/mountinfo` |
| `umount -l` | Lazy unmount: immediately detaches the mount from the tree and cleans up references only when it is no longer in use |
| `2>/dev/null` | Suppresses umount error messages |
| `\|\| break` | Ends the loop if an umount fails instead of continuing indefinitely |

</details>

4. Then mount and publish normally; consumers with `rslave` automatically receive the fresh mount.

In the test, the entire chain took 160 seconds: the Rclone process was terminated, the error was detected, the container was restarted, the orphaned mount was removed, and the new mount was published again. The consuming container kept running throughout and noticed only a brief interruption.

Those running Rclone directly **on the host** through systemd avoid the first two problems and need only `rslave` on consuming containers. The additional container is particularly worthwhile when the host should remain free of Rclone installations or when multiple mounts need to be managed consistently. In that case, all three layers must be configured deliberately.

## Sources

1.  [Docker: Bind mounts: configure bind propagation](https://docs.docker.com/engine/storage/bind-mounts/#configure-bind-propagation): the propagation modes rprivate, rslave, and rshared and their default behavior.

2.  [Kernel documentation: Shared Subtrees](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt): Linux kernel mount propagation, on which Docker's bind options are based.

3.  [Rclone mount](https://rclone.org/commands/rclone_mount/): VFS cache modes, --allow-other, and the limits of the FUSE mount.

4.  [AppArmor documentation (Ubuntu)](https://documentation.ubuntu.com/server/how-to/security/apparmor/): how profiles are bound to executable files; the fusermount3 profile is located at /etc/apparmor.d/fusermount3.
