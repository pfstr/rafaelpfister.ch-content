---
title: "Proton Drive CLI: bruk Proton Drive fra skript og serveren"
navTitle: "Proton Drive CLI"
description: "Siden juni 2026 har Proton tilbudt et offisielt kommandolinjeverktøy for Proton Drive. Artikkelen beskriver kommandoer, pålogging på servere uten skrivebordsmiljø, konfliktstrategier for skript og begrensningene sammenlignet med Rclone."
date: "2026-10-01"
kategorie: "Proton Drive"
timeToRead: "9 min lesetid"
themen:
  - proton-drive
produkte:
  - "proton-drive"
protokolle:
  - "storage"
  - "backup-dr"
related:
  - proton-drive-linux-status
  - rclone-mount-in-docker-container
slug: "proton-drive-cli-bruk-proton-drive-fra-skript-og-serveren"
translationId: "article-97376b998fdaec4c"
translationOf: proton-drive-cli
url: https://rafaelpfister.ch/no/blog/proton-drive-cli-bruk-proton-drive-fra-skript-og-serveren
translationSourceHash: e71e82deea6466312d0d95edc2c96890fc0810219a21308b4cb9346df3f32341
translationModel: gpt-5.6-terra
translatedAt: 2026-10-02T10:01:06.119Z
translationReview: automatic
---

Den 9. juni 2026 lanserte Proton **Proton Drive CLI**, et offisielt kommandolinjeverktøy for Windows, macOS og Linux. Det er basert på samme SDK som de offisielle Drive-appene, har ende-til-ende-kryptering og er tilgjengelig som én enkelt kjørbar fil `proton-drive`. Kildekoden ligger i det offentlige SDK-repositoriet på `cli/`.

CLI-en er beregnet for enkeltstående, tidsavgrensede oppgaver: laste opp filer etter en build, sikkerhetskopiere en mappe etter en tidsplan, kontrollere eller trekke tilbake delinger. Den synkroniserer ikke i bakgrunnen og monterer ikke noe filsystem. Den nåværende versjonen er **0.8.0 fra 13. august 2026**; versjonsnummeret viser at kommandoer og alternativer fortsatt kan endres (versjon 0.8.0 ga konfliktstrategiene nye navn med en inkompatibel endring).

En plassering blant de øvrige Linux-alternativene (Rclone, annonsert skrivebordsklient) finner du i statusartikkelen [Proton Drive på Linux](/blog/proton-drive-linux-status).

## Kommandooversikt

Kommandoene er organisert i grupper: `proton-drive <gruppe> <befehl> [optionen] [argumente]`. Gruppenavn kan forkortes så lenge de forblir entydige; for `filesystem` finnes det dessuten aliaset `fs`. Uten argumenter starter et interaktivt skall. Fullstendig hjelp får du med `proton-drive help` eller `proton-drive <gruppe> <befehl> --help`.

<details class="options-details">
<summary>Oversikt over alternativer</summary>

| Gruppe / kommando | Virkning |
|---|---|
| `auth login` / `auth logout` | Pålogging via nettleseren; utlogging sletter lokale påloggingsdata og cacher |
| `filesystem list <pfad>` | Vis innholdet i en mappe; `/` viser rotområdene |
| `filesystem info` / `size` | Metadata for et element eller størrelsen på en mappe, inkludert papirkurvinnhold |
| `filesystem upload` / `download` | Last opp eller ned filer og mapper |
| `filesystem create-folder`, `rename`, `copy`, `move` | Opprett, gi nytt navn til, kopier og flytt mapper |
| `filesystem trash` / `restore` | Flytt til papirkurven eller gjenopprett |
| `filesystem delete` / `empty-trash` | Slett permanent eller tøm `/trash` |
| `sharing status <pfad>` | Vis medlemmer, åpne invitasjoner og lenkeinnstillinger |
| `sharing invite` / `remove` | Inviter personer via e-post eller trekk tilbake tilgang |
| `sharing set-url` / `remove-url` | Opprett, endre eller fjern offentlig lenke |
| `sharing leave` / `report` | Forlat en deling som er delt med deg, eller rapporter den som misbruk |
| `invitation list` / `accept` / `reject` | Administrer mottatte invitasjoner |
| `album …`, `photo timeline`, `photo upload`, `photo download` | Proton Photos: album og tidslinje |
| `version` | Vis versjoner av CLI og SDK |
| `--json` (`-j`) | Maskinlesbar JSON-utdata for alle kommandoer |
| `--verbose` (`-v`) | Loggutdata direkte i konsollen |
| `--help` (`-h`) | Hjelp for den aktuelle kommandoen |

</details>

Stiene i Proton Drive er alltid POSIX-stier, også i Windows. Roten `/` inneholder virtuelle områder: `/my-files` (egne filer), `/devices` (sikkerhetskopierte datamaskiner), `/shared-by-me`, `/shared-with-me`, `/trash` samt Photos-områdene `/photos`, `/albums`, `/photos-shared-by-me`, `/photos-shared-with-me` og `/photos-trash`.

## Installasjon på Linux

Proton tilbyr buildene på en egen nedlastingsside, hver med SHA-512-sjekksum. For Linux finnes fem varianter:

| Build | Bruksområde |
|---|---|
| `linux/x64` | Standard for moderne x86-64-systemer |
| `linux/x64-baseline` | x86-64 uten AVX2, for eksempel NAS-enheter og eldre server-CPU-er |
| `linux/arm64` | ARM-servere og enkelkortdatamaskiner med glibc |
| `linux/x64-musl`, `linux/arm64-musl` | Distribusjoner med musl i stedet for glibc, for eksempel Alpine Linux og containeravbildninger basert på det |

Hvis standard-builden avsluttes ved oppstart med `Illegal instruction`, mangler CPU-en AVX2-utvidelsen; da er `x64-baseline`-builden det riktige valget. Filen inkluderer Bun-kjøretiden og trenger ingen andre avhengigheter:

```bash
chmod +x proton-drive
sudo install -m 0755 proton-drive /usr/local/bin/proton-drive
proton-drive version
```

Uten administratorrettigheter holder det å kopiere filen til `~/.local/bin`, dersom denne katalogen er i `PATH`.

## Pålogging, også på servere uten skrivebordsmiljø

`auth login` ber ikke om passord på kommandolinjen. CLI-en prøver å åpne en nettleser og skriver også ut påloggings-URL-en. Denne URL-en kan åpnes **på en annen enhet**; terminalen venter til påloggingen er fullført der. Tofaktorautentiseringen foregår som normalt i nettleseren. Dermed fungerer påloggingen også via SSH på en server uten grafisk brukergrensesnitt.

```bash
proton-drive auth login
```

Etter vellykket pålogging lagrer CLI-en økten, ikke passordet. Hvor den lagres, bestemmes av miljøvariabelen `PROTON_DRIVE_CREDENTIALS_STORE`:

| Verdi | Lagringssted |
|---|---|
| `keychain` (standard) | Operativsystemets nøkkellager: Windows Credential Manager, macOS Keychain, på Linux libsecret (GNOME Keyring, KWallet) |
| `pass` | GPG-kryptert oppføring `ch.proton.drive/drive-sdk-cli/auth-session` i passordbehandleren [pass](https://www.passwordstore.org/) |
| `unsafe_file` | Klartekstfilen `auth-session.json` i datakatalogen; ifølge Proton kun for testing |

På en server uten skrivebordsøkt mangler det normalt en ulåst libsecret-nøkkelring. For dette tilfellet har alternativet `pass` eksistert siden versjon 0.6.0. Brukeren som kjører skriptene, trenger da et initialisert passordlager og en GPG-nøkkel som `gpg-agent` kan låse opp uten interaktiv inntasting. Økten må finnes via samme variabel ved hvert kall, og variabelen må derfor også være angitt i Cron-jobber og systemd-enheter:

```bash
export PROTON_DRIVE_CREDENTIALS_STORE=pass
proton-drive auth login
```

Sammenlignet med Rclone er dette et fremskritt: Passord og TOTP-nøkkel ligger ikke på serveren, og `auth logout` avslutter tilgangen. Økten har imidlertid fortsatt full tilgang til kontoen. Det finnes ingen begrensning til bestemte mapper eller skrivebeskyttet tilgang. For automatiserte prosesser er derfor en egen Proton-konto fortsatt det sikrere alternativet.

Cache, applikasjonsdata og logger ligger på Linux i XDG-katalogene (`~/.cache/proton-drive-cli`, `~/.local/share/proton-drive-cli`, `~/.local/state/proton-drive-cli`). Med `PROTON_DRIVE_CACHE_DIR` kan alle tre legges i én enkelt katalog, for eksempel for en container med et montert volum. CLI-en skriver logger med nivået `DEBUG` som standard; `PROTON_DRIVE_LOG_LEVEL=WARNING` reduserer mengden.

## Opplasting og nedlasting i skript

Interaktivt spør CLI-en ved hver navnekonflikt hva som skal skje. I skript er dette ikke mulig: Med `--json` er den interaktive forespørselen deaktivert. Angi derfor alltid konfliktstrategien for filer og mapper uttrykkelig.

```bash
proton-drive filesystem upload --json \
  --file-conflict-strategy create-new-revision \
  --folder-conflict-strategy merge \
  --skip-thumbnails \
  /srv/export/berichte /my-files/backup
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `--json` (`-j`) | Skriv resultatet som JSON; deaktiverer interaktive forespørsler |
| `--file-conflict-strategy` (`-f`) | Atferd når en fil med samme navn finnes: `create-new-revision` (ny versjon av den eksisterende filen), `rename` (legg til suffiks), `replace` (flytt ekstern fil til papirkurven, last opp lokal fil), `skip` |
| `--folder-conflict-strategy` (`-d`) | Atferd ved eksisterende mappe: `merge` (slå sammen innhold), `rename`, `replace`, `skip` |
| `--skip-thumbnails` (`-t`) | Ikke opprett miniatyrbilder; sparer tid ved bilder |
| `/srv/export/berichte` | Lokal kilde; flere kilder er mulig |
| `/my-files/backup` | Målmappe i Proton Drive (siste argument) |

</details>

`create-new-revision` er det riktige valget for sikkerhetskopier: Proton Drive beholder tidligere versjoner av en fil, og CLI-en hopper automatisk over filer med uendret innhold siden versjon 0.7.0. CLI-en sammenligner imidlertid ikke: Lokalt slettede filer forblir i Proton Drive. De som trenger et speil med slettinger, er fortsatt avhengige av `rclone sync`.

Nedlasting fungerer tilsvarende motsatt vei. Strategiene skiller seg fordi den lokale siden overskrives her:

```bash
proton-drive filesystem download --json \
  --file-conflict-strategy remove \
  --folder-conflict-strategy merge \
  /my-files/backup/berichte /srv/restore
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `--file-conflict-strategy` (`-f`) | `rename`, `remove` (slett lokal fil og last ned ekstern versjon) eller `skip` |
| `--folder-conflict-strategy` (`-d`) | `merge`, `rename`, `remove` eller `skip` |
| `/my-files/backup/berichte` | Kilde i Proton Drive; flere kilder er mulig |
| `/srv/restore` | Lokal målmappe (siste argument) |

</details>

CLI-en hopper over Proton Docs og Proton Sheets ved nedlasting; de kan for tiden ikke eksporteres som filer.

En regelmessig opplasting kan planlegges med en systemd-timer eller Cron. JSON-utdataene kan deretter evalueres med `jq`, for eksempel for en melding til overvåkingen.

## Administrere delinger

For offboarding eller revisjoner er administrasjon av delinger ofte mer nyttig enn filoverføring. `sharing status` viser alle medlemmer, åpne invitasjoner og innstillingene for en offentlig lenke for et element:

```bash
proton-drive sharing status --json /my-files/projekte/kunde-a
```

En invitasjon med lesetilgang:

```bash
proton-drive sharing invite \
  --user person@example.com \
  --role viewer \
  /my-files/projekte/kunde-a
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `--user` (`-u`) | E-postadresse til den inviterte personen; kan angis flere ganger |
| `--role` (`-r`) | Rolle, standard `viewer`; flere roller i henhold til `--help` (f.eks. `editor`) |
| `--message` (`-m`) | Melding i invitasjons-e-posten; sendes i **klartekst** |
| `--include-node-name` (`-n`) | Ta med navnet på elementet i invitasjons-e-posten; også klartekst |
| `/my-files/projekte/kunde-a` | Elementet som skal deles |

</details>

En offentlig lenke med passord og utløpsdato:

```bash
proton-drive sharing set-url \
  --role viewer \
  --password 'Linkpasswort' \
  --expiration 2026-12-31 \
  /my-files/projekte/kunde-a/bericht.pdf
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `--role` | `viewer` (standard) eller `editor` |
| `--password` | Eget passord for lenken |
| `--expiration` | Utløpsdato i ISO-format (`JJJJ-MM-TT`) |
| `/my-files/…/bericht.pdf` | Elementet lenken opprettes eller endres for |

</details>

Et passord som angis på kommandolinjen, lagres i shell-historikken og er synlig i prosesslisten mens kommandoen kjører. I skript bør det derfor hentes fra en variabel eller en hemmelighetslagring. `sharing remove-url` fjerner lenken igjen uten å påvirke direkte medlemmer; `sharing remove --user …` trekker tilbake tilgangen for enkeltpersoner.

## Under arbeid: Takeout

Siden 10. september 2026 inneholder SDK-repositoriet en ytterligere kommando, `takeout run`. Den eksporterer en offline-kopi av kontoen til en lokal mappe, valgfritt med `--include my-files`, `devices`, `photos` og `revisions` (alle tidligere filversjoner). For hver mappe skriver den en `manifest.json`, som beskriver eksporten; den endrer ingenting på kontoen. Kommandoen finnes ennå ikke i den utgitte versjonen 0.8.0. Når den kommer, vil den være den nærliggende veien til en fullstendig lokal sikkerhetskopi av Proton Drive-innholdet.

## Begrensninger sammenlignet med Rclone

| Krav | Proton Drive CLI 0.8.0 | Rclone (`protondrive`-backend) |
|---|---|---|
| Offisielt støttet | Ja, av Proton, åpen kildekode | Nei, reverse engineering, beta |
| Pålogging | Nettleser, også på en annen enhet; økt i nøkkellageret eller `pass` | Passord og TOTP-nøkkel i konfigurasjonsfilen |
| Opplasting, nedlasting | Ja, med konfliktstrategier og versjonering | Ja |
| Speil med slettinger (`sync`) | Nei | Ja |
| Montere filsystem (FUSE) | Nei | Ja |
| Delinger, invitasjoner, lenker | Ja | Nei |
| Proton Photos | Ja | Nei |
| Tilgang med begrenset omfang | Nei | Nei |

For sikkerhetskopier, build-artefakter og administrasjon av delinger er CLI-en det bedre valget, fordi den er offisielt støttet og ikke krever et lagret passord. For en montering, slik et [Paperless-dokumentarkiv](/blog/paperless-dokumente-clouddienst-auslagern) trenger, og for speilinger med slettinger, er Rclone foreløpig fortsatt nødvendig. Det største gapet er det samme i begge tilfeller: Proton tilbyr ingen maskintilgang som kan begrenses til bestemte mapper eller lesetilgang.

## Kilder

1.  [Proton: Introducing Proton Drive CLI](https://proton.me/blog/proton-drive-cli): kunngjøringen fra 9. juni 2026 med bruksområde og JSON-utdata.

2.  [Proton Support: Using Proton Drive CLI](https://proton.me/support/drive-cli): veiledning for nedlasting, pålogging og grunnleggende kommandoer.

3.  [Proton Drive CLI: Downloads](https://proton.me/download/drive/cli/index.html): nåværende versjon 0.8.0, alle plattform-builds med SHA-512-sjekksummer.

4.  [ProtonDriveApps/sdk: cli/README.md](https://github.com/ProtonDriveApps/sdk/blob/main/cli/README.md): miljøvariabler, lagringssteder, credential-stores og merknaden om `x64-baseline`-builden.

5.  [ProtonDriveApps/sdk: cli/CHANGELOG.md](https://github.com/ProtonDriveApps/sdk/blob/main/cli/CHANGELOG.md): versjonshistorikk fra 0.4.2 til 0.8.0, blant annet støtte for `pass` (0.6.0) og hopping over uendrede filer (0.7.0).

6.  [ProtonDriveApps/sdk: cli/src/commands](https://github.com/ProtonDriveApps/sdk/tree/main/cli/src/commands): kildekode for kommandoene med alternativer, konfliktstrategier og den ennå ikke utgitte Takeout-kommandoen.

7.  [Proton for Business: Proton Drive CLI](https://proton.me/business/drive/cli): Protons bruksscenarier for virksomheter, for eksempel å trekke tilbake delinger når ansatte slutter.

8.  [Rclone: Proton Drive](https://rclone.org/protondrive/): community-backenden med mount- og sync-funksjon til sammenligning.
