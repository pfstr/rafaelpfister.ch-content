---
title: "Drift rclone-monteringer pålitelig i Docker"
navTitle: "Rclone i Docker"
description: "For at en FUSE-montering fra en container også skal fungere på verten og i andre containere, må mount-propagation, AppArmor og gjenoppretting etter feil samspille."
date: "2026-07-26"
kategorie: "Rclone"
timeToRead: "9 min. lesetid"
themen:
  - "rclone"
related:
  - "paperless-dokumente-clouddienst-auslagern"
slug: "drift-rclone-monteringer-palitelig-i-docker"
translationOf: "rclone-mount-in-docker-container"
url: "https://rafaelpfister.ch/no/blog/drift-rclone-monteringer-palitelig-i-docker"
---

En rclone-montering kjører i en Docker-container, men skal også være tilgjengelig på verten og i andre containere. Da må monteringshendelser krysse flere navnerom. Ett enkelt Compose-alternativ er ikke nok.

I en praktisk test med Ubuntu 25.10, kjerne 6.17 og Docker 29.6 oppsto tre uavhengige feil: Docker nedgraderte `rshared` uten å varsle, AppArmor blokkerte `fusermount3`, og en konsumerende container holdt fast ved den gamle monteringen etter en omstart. Det konkrete bruksområdet var en [skylagring for Paperless-ngx](/blog/paperless-dokumente-clouddienst-auslagern); de samme mekanismene gjelder også for andre FUSE-verktøy som sshfs.

## 1. Vertskilden må selv være `shared`

For at en montering fra containeren skal nå verten, trenger bind-monteringen propagation `rshared`:

```yaml
    volumes:
      - type: bind
        source: /srv/storage/media
        target: /data
        bind:
          propagation: rshared
```

`rshared` fungerer bare hvis bind-kilden på verten **selv er et monteringspunkt med shared-propagation**. En vanlig katalog oppfyller ikke dette kravet. Docker melder likevel ingen feil, men bruker i stillhet en svakere propagation. Dette kan ses i `/proc/self/mountinfo` inne i containeren:

```text
1938 2077 8:2 /srv/storage/media /data rw,relatime master:1 - ext4 /dev/sda2 rw
```

`master:1` betyr slave-propagation: Monteringer kommer inn fra verten, men aldri ut. Riktig ville vært `shared:N`. Løsningen er å bind-mounte kilden til seg selv og merke den som shared:

```bash
mount --bind /srv/storage/media /srv/storage/media
mount --make-shared /srv/storage/media
```

For at dette skal overleve en omstart, må det inn i en systemd-enhet med `Before=docker.service`. Kontroll: `findmnt -no PROPAGATION /srv/storage/media` må gi `shared`.

## 2. AppArmor kontrollerer `fusermount3` også i containeren

Med korrekt propagation kom neste overraskelse. Monteringen på den delte banen feilet fortsatt:

```text
NOTICE: mount helper error: fusermount3: mount failed: Permission denied
CRITICAL: Fatal error: failed to mount FUSE fs: fusermount: exit status 1
```

De vanlige ekstra containerrettighetene endret ingenting: verken `CAP_SYS_ADMIN` og `/dev/fuse` eller `unconfined` eller til og med `--privileged`. En tmpfs-montering fungerte på samme mål, og FUSE fungerte på andre baner. Først kjernens audit-logg viste den faktiske årsaken:

```text
audit: type=1400 apparmor="DENIED" operation="mount" class="mount"
  info="failed mntpnt match" error=-13 profile="fusermount3"
  name="/data/documents/originals/" fstype="fuse.rclone"
```

Ubuntu leverer en **AppArmor-profil for binærfilen `fusermount3`**, som bare tillater FUSE-monteringer på en positivliste over monteringspunktmønstre. Denne profilen gjelder også for fusermount3 **i containeren**. Det avgjørende er banen slik containeren ser den:

```text
mount fstype=@{fuse_types} ... -> @{HOME}/**/,
mount fstype=@{fuse_types} ... -> /mnt/{,**/},
mount fstype=@{fuse_types} ... -> /media/**/,
mount fstype=@{fuse_types} ... -> /tmp/**/,
```

`/data` står ikke på listen, og det gjør heller ikke `/srv`. At containeren kjører unconfined, hjelper ikke: Profilen er knyttet til den kjørbare filen, ikke til containeren.

Utveien utnytter at bare fusermount3 er underlagt profilen, mens et vanlig `mount --bind` ikke er det: Monter FUSE på en **tillatt** bane, og publiser den derfra via bind-mount til den delte banen.

```sh
rclone mount remote:sti /mnt/inner/dokumenter --allow-other --vfs-cache-mode full &
# vent til monteringen svarer, deretter:
mount --bind /mnt/inner/dokumenter /data/dokumenter
```

Bind-monteringen er et vanlig mount(2)-kall og propagerer som alle andre via den shared banen til verten. Dette kunne verifiseres helt inn i en andre container som kunne lese filene som uid 1000. `--allow-other` er obligatorisk så snart en annen bruker enn den som monterer, skal få tilgang til filene; i rclone-containeren må `user_allow_other` stå i `/etc/fuse.conf` (allerede tilfelle i det offisielle imaget).

## 3. Konsumenter trenger `rslave`

Den tredje fellen gjelder motsatt side. Hvis rclone-prosessen dør og monteringen bygges opp på nytt, ser verten den straks. En container som har fått banen bind-mountet på vanlig måte, ser den derimot ikke:

```text
ls: cannot access '/usr/src/app/media': Transport endpoint is not connected
```

Docker bruker som standard `rprivate` for bind-monteringer: En montering som oppstår på verten **etter** at containeren er startet, når aldri dens mount-navnerom. Containeren blir hengende på den døde FUSE-monteringen til den opprettes på nytt. Løsningen koster én linje:

```yaml
    volumes:
      - type: bind
        source: /srv/storage/media
        target: /usr/src/app/media
        bind:
          propagation: rslave
```

Med `rslave` videresender verten nye monteringshendelser til containeren. I testen så konsumenten igjen alle filene etter en hardt avsluttet og nyopprettet montering **uten egen omstart**. Omstartstelleren forble på null.

## Gjenoppretting uten manuell inngripen

De tre byggeklossene gir sammen et robust mønster som ikke krever en watchdog-daemon:

1. Monteringscontaineren kontrollerer monteringene sine i en løkke. Hvis en ikke lenger svarer, avslutter den med feilkode.
2. `restart: unless-stopped` lar Docker starte containeren på nytt.
3. Ved oppstart rydder containeren først opp i **foreldreløse monteringer fra forrige instans**: En død bind-montering på målbanen ville ellers blokkere ny publisering, og fra verten kan en uprivilegert bruker ikke fjerne den. I containeren går det, og umount propagerer ut:

```sh
while grep -q " /data/dokumenter " /proc/self/mountinfo; do
    umount -l /data/dokumenter 2>/dev/null || break
done
```

4. Deretter monteres og publiseres det normalt; konsumenter med `rslave` overtar den nye monteringen av seg selv.

I testen tok hele kjeden 160 sekunder: rclone-prosessen ble avsluttet, feilen ble oppdaget, containeren startet på nytt, den foreldreløse monteringen ble fjernet og den nye monteringen ble publisert igjen. Den konsumerende containeren fortsatte å kjøre imens og merket bare et kort avbrudd.

De som kjører rclone direkte **på verten** via systemd, unngår de to første problemene og trenger bare `rslave` på de konsumerende containerne. Den ekstra containeren er først og fremst verdt det hvis verten skal holdes fri for rclone-installasjoner eller flere monteringer skal administreres enhetlig. Da må alle tre nivåene konfigureres bevisst.

## Kilder

1.  [Docker: Bind mounts: konfigurer bind propagation](https://docs.docker.com/engine/storage/bind-mounts/#configure-bind-propagation): propagation-modusene rprivate, rslave og rshared og deres standardoppførsel.

2.  [Kjernedokumentasjon: Shared Subtrees](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt): Linux-kjernens mount-propagation, som Dockers bind-alternativer bygger på.

3.  [Rclone mount](https://rclone.org/commands/rclone_mount/): VFS-cache-moduser, --allow-other og begrensningene ved FUSE-montering.

4.  [AppArmor-dokumentasjon (Ubuntu)](https://documentation.ubuntu.com/server/how-to/security/apparmor/): hvordan profiler knyttes til kjørbare filer; fusermount3-profilen ligger under /etc/apparmor.d/fusermount3.
