---
title: "Driva Rclone-mountar i Docker på ett tillförlitligt sätt"
navTitle: "Rclone i Docker"
description: "För att en FUSE-mount från en container även ska fungera på värden och i andra containrar måste mount-propagation, AppArmor och återställning efter fel samverka."
date: "2026-07-26"
kategorie: "Rclone"
timeToRead: "9 min. läsning"
themen:
  - "rclone"
related:
  - "paperless-dokumente-clouddienst-auslagern"
slug: "driva-rclone-mountar-i-docker-pa-ett-tillforlitligt-satt"
translationOf: "rclone-mount-in-docker-container"
url: "https://rafaelpfister.ch/sv/blog/driva-rclone-mountar-i-docker-pa-ett-tillforlitligt-satt"
---

En Rclone-mount körs i en Docker-container, men ska även vara tillgänglig på värden och i andra containrar. För detta måste mount-händelser passera flera namnrymder. Ett enda Compose-alternativ räcker inte.

I ett praktiskt test med Ubuntu 25.10, kärna 6.17 och Docker 29.6 uppstod tre sinsemellan oberoende fel: Docker nedgraderade `rshared` obemärkt, AppArmor blockerade `fusermount3`, och en konsumerande container höll fast vid den gamla mounten efter en omstart. Det konkreta användningsfallet var en [molnlagring för Paperless-ngx](/blog/paperless-dokumente-clouddienst-auslagern); samma mekanismer gäller även för andra FUSE-verktyg som sshfs.

## 1. Värdkällan måste själv vara `shared`

För att en mount från containern ska nå värden behöver bind-mounten propagation `rshared`:

```yaml
    volumes:
      - type: bind
        source: /srv/storage/media
        target: /data
        bind:
          propagation: rshared
```

`rshared` fungerar endast om bind-källan på värden **själv är en mountpoint med Shared-propagation**. En vanlig katalog uppfyller inte detta krav. Docker rapporterar ändå inget fel utan använder tyst en svagare propagation. Detta syns i `/proc/self/mountinfo` inuti containern:

```text
1938 2077 8:2 /srv/storage/media /data rw,relatime master:1 - ext4 /dev/sda2 rw
```

`master:1` betyder slave-propagation: mountar kommer in från värden, men aldrig ut. Rätt vore `shared:N`. Lösningen är att binda källan till sig själv och markera den som shared:

```bash
mount --bind /srv/storage/media /srv/storage/media
mount --make-shared /srv/storage/media
```

För att detta ska överleva en omstart ska det läggas i en systemd-enhet med `Before=docker.service`. Kontroll: `findmnt -no PROPAGATION /srv/storage/media` måste returnera `shared`.

## 2. AppArmor kontrollerar `fusermount3` även i containern

Med korrekt propagation kom nästa överraskning. Mounten på den delade sökvägen misslyckades fortfarande:

```text
NOTICE: mount helper error: fusermount3: mount failed: Permission denied
CRITICAL: Fatal error: failed to mount FUSE fs: fusermount: exit status 1
```

De vanliga extra containerbehörigheterna ändrade inget: varken `CAP_SYS_ADMIN` och `/dev/fuse` eller `unconfined` eller ens `--privileged`. En tmpfs-mount fungerade på samma mål, och FUSE fungerade på andra sökvägar. Först kärnans audit-logg visade den verkliga orsaken:

```text
audit: type=1400 apparmor="DENIED" operation="mount" class="mount"
  info="failed mntpnt match" error=-13 profile="fusermount3"
  name="/data/documents/originals/" fstype="fuse.rclone"
```

Ubuntu levererar en **AppArmor-profil för binärfilen `fusermount3`**, som endast tillåter FUSE-mountar på en positivlista av mountpoint-mönster. Den här profilen gäller även för fusermount3 **i containern**. Avgörande är sökvägen så som containern ser den:

```text
mount fstype=@{fuse_types} ... -> @{HOME}/**/,
mount fstype=@{fuse_types} ... -> /mnt/{,**/},
mount fstype=@{fuse_types} ... -> /media/**/,
mount fstype=@{fuse_types} ... -> /tmp/**/,
```

`/data` finns inte på listan, och inte heller `/srv`. Att containern körs unconfined hjälper inte: profilen är knuten till den körbara filen, inte till containern.

Utvägen utnyttjar att endast fusermount3 omfattas av profilen, men inte ett vanligt `mount --bind`: montera FUSE på en **tillåten** sökväg och publicera den därifrån via bind-mount till den delade sökvägen.

```sh
rclone mount remote:sökväg /mnt/inner/dokument --allow-other --vfs-cache-mode full &
# vänta tills mounten svarar, sedan:
mount --bind /mnt/inner/dokument /data/dokument
```

Bind-mounten är ett vanligt mount(2)-anrop och propageras, precis som alla andra, via den Shared-sökvägen till värden. Detta kunde verifieras ända in i en andra container, som kunde läsa filerna som uid 1000. `--allow-other` är obligatoriskt så snart en annan användare än den som monterar får åtkomst till filerna; i Rclone-containern måste därför `user_allow_other` finnas i `/etc/fuse.conf` (vilket redan är fallet i den officiella imagen).

## 3. Konsumenter behöver `rslave`

Den tredje fallgropen gäller andra sidan. Om Rclone-processen dör och mounten byggs upp på nytt ser värden den omedelbart. En container som har monterat sökvägen normalt via bind ser den däremot inte:

```text
ls: cannot access '/usr/src/app/media': Transport endpoint is not connected
```

Docker använder som standard `rprivate` för bind-mountar: En mount som uppstår på värden **efter** att containern startats når aldrig dess mount-namnrymd. Containern förblir kopplad till den döda FUSE-mounten tills den skapas om. Lösningen kräver en rad:

```yaml
    volumes:
      - type: bind
        source: /srv/storage/media
        target: /usr/src/app/media
        bind:
          propagation: rslave
```

Med `rslave` skickar värden vidare nya mount-händelser till containern. I testet såg konsumenten åter alla filer efter en hårt avslutad och återuppbyggd mount **utan egen omstart**. Omstartsräknaren förblev noll.

## Återställning utan manuella ingrepp

De tre byggstenarna ger tillsammans ett robust helhetsmönster som klarar sig utan watchdog-daemon:

1. Mount-containern kontrollerar sina mountar i en slinga. Om en inte längre svarar avslutas den med en felkod.
2. `restart: unless-stopped` låter Docker starta om containern.
3. Vid start rensar containern först bort **överblivna mountar från föregående körning**: en död bind-mount på målsökvägen skulle annars blockera ny publicering, och från värden kan en oprivilegierad användare inte ta bort den. I containern går det, och umount propagateras utåt:

```sh
while grep -q " /data/dokument " /proc/self/mountinfo; do
    umount -l /data/dokument 2>/dev/null || break
done
```

4. Montera och publicera sedan normalt; konsumenter med `rslave` tar automatiskt över den nya mounten.

I testet tog hela kedjan 160 sekunder: Rclone-processen avslutades, felet upptäcktes, containern startades om, den överblivna mounten togs bort och den nya mounten publicerades igen. Den konsumerande containern fortsatte köras under tiden och märkte bara ett kort avbrott.

Den som kör Rclone direkt **på värden** via systemd undviker de två första problemen och behöver endast `rslave` på de konsumerande containrarna. Den extra containern är främst värd det om värden ska hållas fri från Rclone-installationer eller om flera mountar ska hanteras enhetligt. Då måste alla tre nivåer konfigureras medvetet.

## Källor

1.  [Docker: Bind mounts: configure bind propagation](https://docs.docker.com/engine/storage/bind-mounts/#configure-bind-propagation): propagationslägena rprivate, rslave och rshared samt deras standardbeteende.

2.  [Kernel-dokumentation: Shared Subtrees](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt): Linux-kärnans mount-propagation, som Dockers bind-alternativ bygger på.

3.  [Rclone mount](https://rclone.org/commands/rclone_mount/): VFS-cachelägen, --allow-other och begränsningarna för FUSE-mounten.

4.  [AppArmor-dokumentation (Ubuntu)](https://documentation.ubuntu.com/server/how-to/security/apparmor/): hur profiler binds till körbara filer; fusermount3-profilen finns under /etc/apparmor.d/fusermount3.
