---
title: "Driva Rclone-monteringar tillförlitligt i Docker"
navTitle: "Rclone i Docker"
description: "För att en FUSE-montering från en container ska fungera både på värden och i andra containrar måste monteringspropagering, AppArmor och återställning efter fel samverka."
date: "2026-07-26"
kategorie: "Rclone"
timeToRead: "9 min lästid"
themen:
  - rclone
related:
  - paperless-dokumente-clouddienst-auslagern
slug: "driva-rclone-mountar-i-docker-pa-ett-tillforlitligt-satt"
translationOf: "rclone-mount-in-docker-container"
translationId: article-a08b15399e144547
translationModel: gpt-5.6-terra
translatedAt: 2026-09-03T08:30:00.570Z
translationReview: automatic
translationSourceHash: 5cba6faedde80db33a3f35e999758cb09a93ccb85cfb9021a45026a99173bb26
url: https://rafaelpfister.ch/sv/blog/driva-rclone-mountar-i-docker-pa-ett-tillforlitligt-satt
---

En Rclone-montering körs i en Docker-container men ska även vara tillgänglig på värden och i andra containrar. För detta måste monteringshändelser passera flera namnrymder. Ett enskilt Compose-alternativ räcker inte.

I ett praktiskt test med Ubuntu 25.10, kernel 6.17 och Docker 29.6 uppstod tre oberoende fel: Docker nedgraderade omärkligt `rshared`, AppArmor blockerade `fusermount3`, och en konsumerande container höll efter en omstart fast vid den gamla monteringen. Det konkreta användningsfallet var en [molnlagring för Paperless-ngx](/blog/paperless-dokumente-clouddienst-auslagern); samma mekanismer gäller även för andra FUSE-verktyg som sshfs.

## 1. Värdkällan måste själv vara `shared`

För att en montering från containern ska nå värden behöver bind-monteringen propageringen `rshared`:

```yaml
    volumes:
      - type: bind
        source: /srv/storage/media
        target: /data
        bind:
          propagation: rshared
```

`rshared` fungerar bara om bind-källan på värden **själv är en monteringspunkt med shared-propagering**. En vanlig katalog uppfyller inte detta krav. Docker rapporterar ändå inget fel utan använder i tysthet en svagare propagering. Det syns i `/proc/self/mountinfo` i containern:

```text
1938 2077 8:2 /srv/storage/media /data rw,relatime master:1 - ext4 /dev/sda2 rw
```

`master:1` betyder slave-propagering: monteringarna kommer in från värden, men aldrig ut. Rätt värde vore `shared:N`. Lösningen är att bind-montera källan till sig själv och markera den som shared:

```bash
mount --bind /srv/storage/media /srv/storage/media
mount --make-shared /srv/storage/media
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `--bind quelle ziel` | Bind-monterar en katalog till en andra sökväg; här till sig själv, vilket gör katalogen till en egen monteringspunkt |
| `--make-shared pfad` | Ställer in monteringspunktens propagering till shared så att monteringshändelser vidarebefordras i båda riktningarna |

</details>

För att detta ska överleva en omstart bör det läggas i en systemd-enhet med `Before=docker.service`. Kontroll: `findmnt -no PROPAGATION /srv/storage/media` måste returnera `shared`.

## 2. AppArmor kontrollerar `fusermount3` även i containern

Med korrekt propagering uppstod nästa problem. Monteringen till den delade sökvägen misslyckades fortfarande:

```text
NOTICE: mount helper error: fusermount3: mount failed: Permission denied
CRITICAL: Fatal error: failed to mount FUSE fs: fusermount: exit status 1
```

De vanliga extra containerbehörigheterna ändrade ingenting: varken `CAP_SYS_ADMIN` och `/dev/fuse` eller `unconfined` eller till och med `--privileged`. En tmpfs-montering fungerade på samma mål, och FUSE fungerade på andra sökvägar. Först kernelns audit-logg visade den verkliga orsaken:

```text
audit: type=1400 apparmor="DENIED" operation="mount" class="mount"
  info="failed mntpnt match" error=-13 profile="fusermount3"
  name="/data/documents/originals/" fstype="fuse.rclone"
```

Ubuntu levererar en **AppArmor-profil för binärfilen `fusermount3`**, som endast tillåter FUSE-monteringar till en positivlista med monteringspunktsmönster. Denna profil gäller även för fusermount3 **i containern**. Avgörande är sökvägen så som containern ser den:

```text
mount fstype=@{fuse_types} ... -> @{HOME}/**/,
mount fstype=@{fuse_types} ... -> /mnt/{,**/},
mount fstype=@{fuse_types} ... -> /media/**/,
mount fstype=@{fuse_types} ... -> /tmp/**/,
```

`/data` finns inte i listan, `/srv` inte heller. Att containern körs unconfined hjälper inte: profilen är knuten till den körbara filen, inte till containern.

Utvägen bygger på att endast fusermount3 omfattas av profilen, medan en vanlig `mount --bind` inte gör det: montera FUSE på en **tillåten** sökväg och publicera den därifrån via bind-montering till den delade sökvägen.

```sh
rclone mount remote:pfad /mnt/inner/dokumente --allow-other --vfs-cache-mode full &
# vänta tills monteringen svarar, sedan:
mount --bind /mnt/inner/dokumente /data/dokumente
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `remote:pfad` | Rclone-fjärrlagring inklusive sökväg som ska monteras |
| `/mnt/inner/dokumente` | Monteringspunkt under `/mnt`, ett mönster som tillåts av AppArmor-profilen |
| `--allow-other` | Tillåter andra användare än den som monterade att komma åt FUSE-montering |
| `--vfs-cache-mode full` | Buffrar läs- och skrivåtkomst helt i den lokala cachen |
| `&` | Startar monteringen i bakgrunden så att skalet blir fritt för bind-montering |
| `mount --bind quelle ziel` | Publicerar FUSE-montering via bind-montering till den delade sökvägen; som mount(2)-anrop omfattas den inte av fusermount3-profilen |

</details>

Bind-montering är ett vanligt mount(2)-anrop och propageras, som alla andra, via den delade sökvägen till värden. Detta kunde verifieras ända till en andra container som kunde läsa filerna som uid 1000. `--allow-other` är obligatoriskt så snart en annan användare än den monterande får åtkomst till filerna; i Rclone-containern måste därför `user_allow_other` finnas i `/etc/fuse.conf` (vilket redan är fallet i den officiella imagen).

## 3. Konsumenter behöver `rslave`

Det tredje problemet gäller andra sidan. Om Rclone-processen kraschar och monteringen byggs upp på nytt ser värden den omedelbart. En container som har monterat in sökvägen på vanligt sätt via bind ser den däremot inte:

```text
ls: cannot access '/usr/src/app/media': Transport endpoint is not connected
```

Docker använder som standard `rprivate` för bind-monteringar: en montering som uppstår på värden **efter** att containern har startat når aldrig dess monteringsnamnrymd. Containern förblir hängande vid den frånkopplade FUSE-montering tills den återskapas. Lösningen kräver en rad:

```yaml
    volumes:
      - type: bind
        source: /srv/storage/media
        target: /usr/src/app/media
        bind:
          propagation: rslave
```

Med `rslave` vidarebefordrar värden nya monteringshändelser till containern. I testet såg konsumenten efter en hårt avslutad och nyuppbyggd montering åter alla filer **utan egen omstart**. Omstartsräknaren förblev noll.

## Återställning utan manuella ingrepp

De tre byggstenarna ger tillsammans ett robust helhetsmönster som klarar sig utan en watchdog-daemon:

1. Monteringscontainern kontrollerar sina monteringar i en slinga. Om någon inte längre svarar avslutas den med en felkod.
2. `restart: unless-stopped` låter Docker starta om containern.
3. Vid start rensar containern först bort **överblivna monteringar från föregående körning**: en överbliven bind-montering på målsökvägen skulle annars blockera ny publicering, och från värden kan en oprivilegierad användare inte ta bort den. I containern går det, och umount propageras utåt:

```sh
while grep -q " /data/dokumente " /proc/self/mountinfo; do
    umount -l /data/dokumente 2>/dev/null || break
done
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `grep -q` | Ingen utdata; endast slutkoden anger om sökvägen fortfarande finns som montering i `/proc/self/mountinfo` |
| `umount -l` | Lazy unmount: kopplar omedelbart bort monteringen från trädet och rensar referenser först när den inte längre används |
| `2>/dev/null` | Undertrycker felmeddelanden från umount |
| `\|\| break` | Avslutar slingan om en umount misslyckas i stället för att fortsätta i all oändlighet |

</details>

4. Montera och publicera sedan normalt; konsumenter med `rslave` tar automatiskt över den nya monteringen.

I testet tog hela kedjan 160 sekunder: Rclone-processen avslutades, felet upptäcktes, containern startades om, den överblivna monteringen togs bort och den nya monteringen publicerades igen. Den konsumerande containern fortsatte att köra under tiden och märkte bara ett kort avbrott.

Den som kör Rclone direkt **på värden** via systemd undviker de första två problemen och behöver bara `rslave` på de konsumerande containrarna. Den extra containern är främst värd det om värden ska hållas fri från Rclone-installationer eller om flera monteringar ska hanteras enhetligt. Då måste alla tre nivåer konfigureras medvetet.

## Källor

1.  [Docker: Bind mounts: configure bind propagation](https://docs.docker.com/engine/storage/bind-mounts/#configure-bind-propagation): propageringslägena rprivate, rslave och rshared samt deras standardbeteende.

2.  [Kernel-dokumentation: Shared Subtrees](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt): Linux-kärnans monteringspropagering som Dockers bind-alternativ bygger på.

3.  [Rclone mount](https://rclone.org/commands/rclone_mount/): VFS-cachelägen, --allow-other och begränsningarna för FUSE-montering.

4.  [AppArmor-dokumentation (Ubuntu)](https://documentation.ubuntu.com/server/how-to/security/apparmor/): hur profiler knyts till körbara filer; fusermount3-profilen finns under /etc/apparmor.d/fusermount3.
