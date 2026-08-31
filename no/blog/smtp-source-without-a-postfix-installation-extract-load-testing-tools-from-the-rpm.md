---
title: "smtp-source uten Postfix-installasjon: Pakk ut verktøy for belastningstesting fra RPM-en"
navTitle: "Pakk ut smtp-source"
description: "smtp-source og smtp-sink er en del av Postfix, men kjører også uten en installert e-postserver. Slik pakker du ut de to verktøyene fra pakken på RHEL, hvorfor kjøring fra /tmp kan mislykkes på grunn av monteringsalternativet noexec, og hvilke biblioteker som må følge med."
date: "2026-08-27"
kategorie: "SMTP og e-postflyt"
timeToRead: "7 min lesetid"
themen:
  - smtp-mailflow
  - smtp-lasttests
produkte:
  - "uebergreifend"
protokolle:
  - "smtp"
  - "testing"
  - "troubleshooting"
slug: "smtp-source-without-a-postfix-installation-extract-load-testing-tools-from-the-rpm"
translationId: "article-d0a27da11509d24b"
translationOf: smtp-source-ohne-postfix-installation
translationSourceHash: fd4ad6beb5036817db9b7758653a2b7d015a6adb15d7b4a0b47f94161e34b4e6
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:12:55.128Z
translationReview: automatic
url: https://rafaelpfister.ch/no/blog/smtp-source-without-a-postfix-installation-extract-load-testing-tools-from-the-rpm
---

# smtp-source uten Postfix-installasjon: Pakk ut verktøy for belastningstesting fra RPM-en

For SMTP-belastningstester er `smtp-source` et godt valg: Verktøyet åpner parallelle økter, holder dem åpne over flere meldinger og gjengir dermed tilkoblingsatferden til en masseavsender langt mer realistisk enn testverktøy som oppretter en ny tilkobling for hver e-post. Motstykket `smtp-sink` tar imot e-post og forkaster den uten å levere noe. Begge følger med Postfix.

Det er nettopp der problemet ligger: Systemet du vil teste fra, har ofte ikke Postfix installert. På en e-postgateway-appliance er en installasjon heller ikke ønskelig, fordi en ekstra Postfix medfører sin egen konfigurasjon under `/etc/postfix` og en systemtjeneste som i verste fall opptar port 25 og dermed blokkerer det egentlige e-postsystemet. I tillegg kommer spørsmålet om hva produsentens support mener om etterinstallerte pakker på appliance-en.

Begge verktøyene kan imidlertid brukes uten installasjon: Last ned RPM-en, pakk ut binærfilene sammen med bibliotekene, ferdig. Veien dit har to særegenheter som denne artikkelen viser på et RHEL-8-system. Du trenger ikke root-rettigheter, bare tilgang til pakkekildene.

## Finnes smtp-source allerede?

Kontroller først om verktøyet kanskje allerede finnes på systemet. `smtp-source` ligger, avhengig av distribusjon, utenfor den vanlige PATH-en:

```bash
command -v smtp-source || \
  ls /usr/sbin/smtp-source /usr/lib/postfix/sbin/smtp-source 2>/dev/null
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `command -v smtp-source` | Skriver ut banen dersom programmet ligger i PATH, ellers ingenting |
| `/usr/sbin/... /usr/lib/postfix/sbin/...` | De vanlige plasseringene utenfor PATH (RHEL og Debian/Ubuntu) |
| `2>/dev/null` | Undertrykker feilmeldingene fra `ls` for baner som ikke finnes |

</details>

Forblir utdataene tomme, mangler også den tilhørende pakken. På RPM-systemer kan du bekrefte dette og samtidig kontrollere om repositoriene tilbyr Postfix:

```bash
rpm -qa | grep -i postfix
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-q` | Forespørselsmodus for rpm |
| `-a` | Viser alle installerte pakker |
| `grep -i postfix` | Filtrerer listen uten hensyn til store og små bokstaver |

</details>

```bash
yum list available postfix
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `list available` | Viser bare pakker som finnes i repositoriene, men ikke er installert |
| `postfix` | Begrenser utdataene til pakken det søkes etter |

</details>

På testsystemet var ingen Postfix installert, men BaseOS-repositoriet tilbød `postfix-3.5.8-8.el8_10`. Dermed er veien åpen: Pakken kan lastes ned uten å installeres.

## Last bare ned RPM-en

`yum download` (fra plugin-pakken `dnf-plugins-core`, som vanligvis finnes på RHEL 8) laster ned en pakke til gjeldende katalog uten å installere den. Dette fungerer uten root-rettigheter så lenge målkatalogen er skrivbar:

```bash
cd /tmp && yum download postfix
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `cd /tmp` | Bytter til en skrivbar katalog; `yum download` lagrer RPM-en i gjeldende katalog |
| `download` | Underkommando fra `dnf-plugins-core`: laster ned pakken uten å installere den |
| `postfix` | Navnet på pakken som skal lastes ned |

</details>

Hvis yum melder `No such command: download`, mangler plugin-modulen. Med root-rettigheter oppnår du det samme via installasjonskommandoen med `--downloadonly`:

```bash
sudo yum install --downloadonly --downloaddir=/tmp postfix
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `--downloadonly` | Avbryter etter nedlastingen; ingenting installeres |
| `--downloaddir=/tmp` | Målkatalog for den nedlastede RPM-en |
| `postfix` | Pakkens navn |

</details>

Uten noen av delene gjenstår omveien via et annet system med samme RHEL-versjon: Last ned RPM-en der og kopier den til målsystemet med `scp`.

## Pakk ut binærfiler og biblioteker

`rpm2cpio` konverterer RPM-en til en cpio-arkivstrøm, som `cpio` bruker til å pakke ut bestemte baner. I tillegg til de to binærfilene trenger du også Postfix-bibliotekene, fordi verktøyene på RHEL er dynamisk lenket mot `libpostfix-*.so`:

```bash
cd /tmp && rpm2cpio postfix-*.rpm | \
  cpio -idmv ./usr/sbin/smtp-source ./usr/sbin/smtp-sink \
  './usr/lib64/postfix/*'
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `rpm2cpio postfix-*.rpm` | Konverterer RPM-en til en cpio-arkivstrøm på stdout |
| `-i` | cpio-utpakkingsmodus (copy-in) |
| `-d` | Oppretter manglende kataloger ved utpakking |
| `-m` | Beholder filenes endringstidspunkter |
| `-v` | Lister hver utpakkede fil |
| `./usr/sbin/smtp-source ./usr/sbin/smtp-sink` | De to binærfilene, baner nøyaktig som i arkivet (med innledende `./`) |
| `'./usr/lib64/postfix/*'` | Postfix-bibliotekene; mønsteret er satt i anførselstegn slik at cpio tolker det, ikke skallet |

</details>

Filene ligger deretter under `/tmp/usr/`.

## Problem 1: /tmp er montert med noexec

Det nærliggende forsøket på å starte direkte fra /tmp mislykkes på herdede systemer:

```text
bash: /tmp/usr/sbin/smtp-sink: Permission denied
[1]+  Exit 126
```

Avslutningskode 126 til tross for korrekt satt execute-bit er det typiske tegn på et filsystem med monteringsalternativet `noexec`. Kjernen nekter da all programkjøring fra dette filsystemet, uavhengig av filrettighetene. Dette kan kontrolleres direkte:

```bash
mount | grep ' /tmp '
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `mount` | Lister uten argumenter alle monterte filsystemer med monteringsalternativene deres |
| `' /tmp '` | Søkemønster med mellomrom før og etter, slik at bare monteringspunktet `/tmp` treffes og ikke for eksempel `/var/tmp` |

</details>

Løsningen: Kopier binærfilene og bibliotekene til en katalog der filsystemet tillater kjøring, for eksempel din egen hjemmekatalog:

```bash
mkdir -p ~/bin && \
  cp /tmp/usr/sbin/smtp-source /tmp/usr/sbin/smtp-sink \
     /tmp/usr/lib64/postfix/libpostfix-*.so ~/bin/ && \
  chmod +x ~/bin/smtp-source ~/bin/smtp-sink
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `mkdir -p ~/bin` | Oppretter målkatalogen; uten feil dersom den allerede finnes |
| `cp ... ~/bin/` | Kopierer de to binærfilene og `libpostfix-*.so`-bibliotekene til den kjørbare katalogen |
| `chmod +x` | Setter execute-bit på begge binærfilene |

</details>

Merk at `noexec` også påvirker innlasting av delte biblioteker. Det er derfor ikke nok å bare flytte binærfilene og la bibliotekene ligge i /tmp.

## Problem 2: bibliotekbanen

Uten ytterligere opplysninger leter den dynamiske lenkeren etter Postfix-bibliotekene under `/usr/lib64/postfix`, der de ikke finnes fordi det mangler en installasjon:

```text
smtp-sink: error while loading shared libraries: libpostfix-global.so:
cannot open shared object file: No such file or directory
```

`LD_LIBRARY_PATH` utvider lenkerens søkebane med din egen katalog. Variabelen settes foran hvert kall:

```bash
LD_LIBRARY_PATH=~/bin ~/bin/smtp-source ...
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `LD_LIBRARY_PATH=~/bin` | Utvider den dynamiske lenkerens søkebane for dette ene kallet med `~/bin` |
| `~/bin/smtp-source` | Kall via full bane, siden `~/bin` ikke nødvendigvis ligger i PATH |

</details>

Med `ldd ~/bin/smtp-source` kan du på forhånd se om alle avhengigheter kan løses. Utover Postfix-bibliotekene er verktøyene bare avhengige av systemets standardbiblioteker.

## Funksjonstest i loopback

Du kan kontrollere at alt fungerer uten én eneste ekte e-post: `smtp-sink` lytter som en engangsmottaker på en høy port, mens `smtp-source` leverer til den. All trafikk forblir på localhost.

```bash
LD_LIBRARY_PATH=~/bin ~/bin/smtp-sink -v 127.0.0.1:2525 100 &
```

```bash
LD_LIBRARY_PATH=~/bin ~/bin/smtp-source -s 2 -m 10 -l 5120 \
  -f test@example.com -t test@example.com 127.0.0.1:2525
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-v` (smtp-sink) | Logger hvert dialogtrinn for de mottatte tilkoblingene |
| `127.0.0.1:2525` | smtp-sink lytter bare på localhost, port 2525 |
| `100` | Backlog: maksimal lengde på køen av ventende tilkoblinger i henhold til listen(2) |
| `-s 2` | To parallelle SMTP-økter |
| `-m 10` | Totalt ti meldinger, fordelt på øktene |
| `-l 5120` | Meldingsstørrelse i byte (uten hodefelt), her 5 KB |
| `-f` / `-t` | Avsender- og mottakeradresse |

</details>

Ved vellykket kjøring gir `smtp-source` ingen utdata, mens smtp-sink skriver ut hele SMTP-dialogen fra `HELO` til `QUIT` for hver melding. Avslutt deretter bakgrunnsprosessen og fjern restene fra /tmp:

```bash
kill %1
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `%1` | Skallets jobbspesifikasjon: avslutter den første bakgrunnsjobben, her smtp-sink |

</details>

```bash
rm -rf /tmp/usr /tmp/postfix-*.rpm
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-r` | Fjerner katalogtreet rekursivt |
| `-f` | Ingen spørsmål, ingen feil ved manglende baner |
| `/tmp/usr /tmp/postfix-*.rpm` | Det utpakkede treet og den nedlastede RPM-en |

</details>

## Merknader for den reelle belastningstesten

For pålitelige målinger av gjennomstrømning bør belastningsgeneratoren stå på en separat maskin i samme nettverkssegment, ikke på selve testobjektet. Hvis `smtp-source` kjører på gatewayen som undersøkes, konkurrerer generatoren og e-postsystemet om CPU og I/O, og målingen viser denne konkurransen i stedet for den faktiske kapasiteten. Lokalt på målsystemet egner det utpakkede verktøyet seg først og fremst til funksjonstester av regelverket og innledende plausibilitetskontroller.

Så snart testen går mot den faktiske port 25, er det reelle e-poster som går gjennom gatewayens regelverk og, avhengig av konfigurasjonen, blir levert. Bruk derfor mottakeradresser som ender kontrollert: en dedikert testpostkasse, en regel som forkaster testavsenderne, eller et forkastingsdomene som leverandøren har beregnet for dette. Produksjonsadresser hører ikke hjemme i en belastningstest.

Fremgangsmåten som er beskrevet, egner seg også utover de to SMTP-verktøyene for ethvert kommandolinjeprogram som følger med en pakke der installasjon på målsystemet ikke er aktuelt. Kombinasjonen av `yum download`, `rpm2cpio` og en kjørbar katalog i hjemmekatalogen er den samme på alle RPM-systemer.

## Kilder

1.  [Postfix: smtp-source(1)](https://www.postfix.org/smtp-source.1.html): Manpage med alle parameterne til belastningsgeneratoren, inkludert økt- og meldingsstyring.

2.  [Postfix: smtp-sink(1)](https://www.postfix.org/smtp-sink.1.html): Manpage for testmottakeren, blant annet med alternativer for kunstige forsinkelser og feilsvar.

3.  [Red Hat: How to download a package without installing it](https://access.redhat.com/solutions/10154): dokumenterer `yum download` og alternativet `--downloadonly`.

4.  [man7.org: mount(8)](https://man7.org/linux/man-pages/man8/mount.8.html): Beskrivelse av monteringsalternativet `noexec` og virkningen det har på programkjøring.
