---
title: "Kjøre Rclone-mounts pålitelig i Docker"
navTitle: "Rclone i Docker"
description: "For at en FUSE-mount fra en container også skal fungere på verten og i andre containere, må mount-propagasjon, AppArmor og gjenoppretting etter feil fungere sammen."
date: "2026-07-26"
kategorie: "Rclone"
timeToRead: "9 min lesetid"
themen:
  - rclone
related:
  - paperless-dokumente-clouddienst-auslagern
slug: "drift-rclone-monteringer-palitelig-i-docker"
translationOf: "rclone-mount-in-docker-container"
translationId: article-a08b15399e144547
translationModel: gpt-5.6-terra
translatedAt: 2026-09-03T08:30:30.572Z
translationReview: automatic
translationSourceHash: 5cba6faedde80db33a3f35e999758cb09a93ccb85cfb9021a45026a99173bb26
url: https://rafaelpfister.ch/no/blog/drift-rclone-monteringer-palitelig-i-docker
---

En Rclone-mount kjører i en Docker-container, men skal også være tilgjengelig på verten og i andre containere. For dette må mount-hendelser krysse flere navnerom. Én enkelt Compose-innstilling er ikke nok.

I en praktisk test med Ubuntu 25.10, kjerne 6.17 og Docker 29.6 oppstod tre uavhengige feil: Docker nedgraderte `rshared` uten varsel, AppArmor blokkerte `fusermount3`, og en konsumerende container holdt fast ved den gamle mounten etter en omstart. Det konkrete brukstilfellet var en [skylagring for Paperless-ngx](/blog/paperless-dokumente-clouddienst-auslagern); de samme mekanismene gjelder også for andre FUSE-verktøy som sshfs.

## 1. Vertskilden må selv være `shared`

For at en mount fra containeren skal nå verten, trenger bind-mounten propagasjonen `rshared`:

```yaml
    volumes:
      - type: bind
        source: /srv/storage/media
        target: /data
        bind:
          propagation: rshared
```

`rshared` fungerer bare hvis bind-kilden på verten **selv er et mountpoint med shared-propagasjon**. En vanlig katalog oppfyller ikke dette kravet. Docker melder likevel ingen feil, men bruker i stillhet en svakere propagasjon. Dette kan sees i `/proc/self/mountinfo` inne i containeren:

```text
1938 2077 8:2 /srv/storage/media /data rw,relatime master:1 - ext4 /dev/sda2 rw
```

`master:1` betyr slave-propagasjon: mounte kommer inn fra verten, men aldri ut. Riktig ville vært `shared:N`. Løsningen er å binde kilden til seg selv og markere den som shared:

```bash
mount --bind /srv/storage/media /srv/storage/media
mount --make-shared /srv/storage/media
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `--bind quelle ziel` | Binder en katalog til en annen sti; her til seg selv, slik at katalogen blir et selvstendig mountpoint |
| `--make-shared pfad` | Setter propagasjonen for dette mountpointet til shared, slik at mount-hendelser videresendes i begge retninger |

</details>

For at dette skal overleve en omstart, må det inn i en systemd-unit med `Before=docker.service`. Kontroll: `findmnt -no PROPAGATION /srv/storage/media` må gi `shared`.

## 2. AppArmor kontrollerer `fusermount3` også i containeren

Med korrekt propagasjon oppstod neste problem. Mounten på den delte stien feilet fortsatt:

```text
NOTICE: mount helper error: fusermount3: mount failed: Permission denied
CRITICAL: Fatal error: failed to mount FUSE fs: fusermount: exit status 1
```

De vanlige ekstra container-rettighetene endret ingenting: verken `CAP_SYS_ADMIN` og `/dev/fuse` eller `unconfined` eller til og med `--privileged`. En tmpfs-mount fungerte på samme mål, og FUSE fungerte på andre stier. Først kjerne-auditloggen viste den faktiske årsaken:

```text
audit: type=1400 apparmor="DENIED" operation="mount" class="mount"
  info="failed mntpnt match" error=-13 profile="fusermount3"
  name="/data/documents/originals/" fstype="fuse.rclone"
```

Ubuntu leverer en **AppArmor-profil for binærfilen `fusermount3`**, som bare tillater FUSE-mounter på en positivliste med mountpoint-mønstre. Denne profilen gjelder også for fusermount3 **i containeren**. Det avgjørende er stien slik containeren ser den:

```text
mount fstype=@{fuse_types} ... -> @{HOME}/**/,
mount fstype=@{fuse_types} ... -> /mnt/{,**/},
mount fstype=@{fuse_types} ... -> /media/**/,
mount fstype=@{fuse_types} ... -> /tmp/**/,
```

`/data` står ikke på listen, `/srv` heller ikke. At containeren kjører unconfined, hjelper ikke: Profilen er knyttet til den kjørbare filen, ikke containeren.

Utveien utnytter at bare fusermount3 er underlagt profilen, mens en vanlig `mount --bind` ikke er det: Mount FUSE på en **tillatt** sti og publiser den derfra med bind-mount til den delte stien.

```sh
rclone mount remote:pfad /mnt/inner/dokumente --allow-other --vfs-cache-mode full &
# vent til mounten svarer, deretter:
mount --bind /mnt/inner/dokumente /data/dokumente
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `remote:pfad` | Rclone-remote med sti som skal mountes |
| `/mnt/inner/dokumente` | Mountpoint under `/mnt`, et mønster tillatt av AppArmor-profilen |
| `--allow-other` | Lar andre brukere enn den som mountet, få tilgang til FUSE-mounten |
| `--vfs-cache-mode full` | Bufrer lese- og skriveoperasjoner fullstendig i den lokale cachen |
| `&` | Starter mounten i bakgrunnen, slik at skallet er ledig for bind-mounten |
| `mount --bind quelle ziel` | Publiserer FUSE-mounten med bind-mount på den delte stien; som et mount(2)-kall er den ikke underlagt fusermount3-profilen |

</details>

Bind-mounten er et vanlig mount(2)-kall og propageres, som alle andre, via shared-stien til verten. Dette kunne verifiseres helt inn i en annen container, som kunne lese filene som uid 1000. `--allow-other` er obligatorisk så snart en annen bruker enn den som mountet, får tilgang til filene; i Rclone-containeren må derfor `user_allow_other` stå i `/etc/fuse.conf` (allerede tilfelle i det offisielle imaget).

## 3. Konsumenter trenger `rslave`

Det tredje problemet gjelder motsatt side. Hvis Rclone-prosessen stopper og mounten bygges opp på nytt, ser verten den umiddelbart. En container som har fått stien bundet inn helt normalt, ser den derimot ikke:

```text
ls: cannot access '/usr/src/app/media': Transport endpoint is not connected
```

Docker bruker som standard `rprivate` for bind-mounter: En mount som oppstår på verten **etter** at containeren har startet, når aldri dens mount-navnerom. Containeren blir hengende på den ikke lenger tilkoblede FUSE-mounten til den opprettes på nytt. Løsningen koster én linje:

```yaml
    volumes:
      - type: bind
        source: /srv/storage/media
        target: /usr/src/app/media
        bind:
          propagation: rslave
```

Med `rslave` videresender verten nye mount-hendelser til containeren. I testen så konsumenten igjen alle filene etter en hardt avsluttet og gjenoppbygd mount **uten egen omstart**. Omstartstelleren forble på null.

## Gjenoppretting uten manuelt inngrep

De tre byggesteinene gir sammen et robust helhetsmønster som klarer seg uten en watchdog-daemon:

1. Mount-containeren kontrollerer mountene sine i en løkke. Hvis én ikke lenger svarer, avslutter den med feilkode.
2. `restart: unless-stopped` lar Docker starte containeren på nytt.
3. Ved oppstart rydder containeren først opp i **foreldreløse mounter fra forrige kjøring**: En foreldreløs bind-mount på målstien ville ellers blokkere ny publisering, og fra verten kan en uprivilegert bruker ikke fjerne den. I containeren går det, og umount-en propageres utover:

```sh
while grep -q " /data/dokumente " /proc/self/mountinfo; do
    umount -l /data/dokumente 2>/dev/null || break
done
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `grep -q` | Ingen utdata; bare exit-koden angir om stien fortsatt finnes som mount i `/proc/self/mountinfo` |
| `umount -l` | Lazy unmount: kobler mounten umiddelbart fra treet og rydder først opp i referanser når den ikke lenger er i bruk |
| `2>/dev/null` | Undertrykker feilmeldinger fra umount |
| `\|\| break` | Avslutter løkken hvis en umount feiler, i stedet for å fortsette uendelig |

</details>

4. Mount og publiser deretter som normalt; konsumenter med `rslave` overtar den nye mounten automatisk.

I testen tok hele kjeden 160 sekunder: Rclone-prosessen ble avsluttet, feilen oppdaget, containeren startet på nytt, den foreldreløse mounten fjernet og den nye mounten publisert igjen. Den konsumerende containeren fortsatte å kjøre og merket bare et kort avbrudd.

De som kjører Rclone direkte **på verten** med systemd, unngår de to første problemene og trenger bare `rslave` på de konsumerende containerne. Den ekstra containeren er først og fremst verdt det hvis verten skal holdes fri for Rclone-installasjoner eller flere mounter skal administreres konsekvent. Da må alle tre nivåene konfigureres bevisst.

## Kilder

1.  [Docker: Bind mounts: configure bind propagation](https://docs.docker.com/engine/storage/bind-mounts/#configure-bind-propagation): propagasjonsmodusene rprivate, rslave og rshared, og standardoppførselen deres.

2.  [Kjernedokumentasjon: Shared Subtrees](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt): mount-propagasjonen i Linux-kjernen som Dockers bind-alternativer bygger på.

3.  [Rclone mount](https://rclone.org/commands/rclone_mount/): VFS-cachemoduser, --allow-other og begrensningene for FUSE-mounten.

4.  [AppArmor-dokumentasjon (Ubuntu)](https://documentation.ubuntu.com/server/how-to/security/apparmor/): hvordan profiler knyttes til kjørbare filer; fusermount3-profilen ligger under /etc/apparmor.d/fusermount3.
