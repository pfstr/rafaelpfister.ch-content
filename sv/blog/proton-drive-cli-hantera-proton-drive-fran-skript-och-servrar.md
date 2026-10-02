---
title: "Proton Drive CLI: Hantera Proton Drive från skript och servrar"
navTitle: "Proton Drive CLI"
description: "Sedan juni 2026 erbjuder Proton ett officiellt kommandoradsverktyg för Proton Drive. Artikeln beskriver kommandon, inloggning på servrar utan skrivbordsmiljö, konfliktstrategier för skript och begränsningarna jämfört med Rclone."
date: "2026-10-01"
kategorie: "Proton Drive"
timeToRead: "9 min lästid"
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
slug: "proton-drive-cli-hantera-proton-drive-fran-skript-och-servrar"
translationId: "article-97376b998fdaec4c"
translationOf: proton-drive-cli
url: https://rafaelpfister.ch/sv/blog/proton-drive-cli-hantera-proton-drive-fran-skript-och-servrar
translationSourceHash: e71e82deea6466312d0d95edc2c96890fc0810219a21308b4cb9346df3f32341
translationModel: gpt-5.6-terra
translatedAt: 2026-10-02T10:00:18.235Z
translationReview: automatic
---

Den 9 juni 2026 släppte Proton **Proton Drive CLI**, ett officiellt kommandoradsverktyg för Windows, macOS och Linux. Det bygger på samma SDK som de officiella Drive-apparna, krypterar från ände till ände och finns tillgängligt som en enda körbar fil `proton-drive`. Källkoden finns i det offentliga SDK-repositoriet på `cli/`.

CLI:t är avsett för enskilda, tidsbestämda åtgärder: ladda upp filer efter en build, säkerhetskopiera en mapp enligt schema, kontrollera eller återkalla delningar. Det synkroniserar inte i bakgrunden och monterar inget filsystem. Den aktuella versionen är **0.8.0 från den 13 augusti 2026**; versionsnumret visar att kommandon och alternativ fortfarande kan ändras (version 0.8.0 bytte namn på konfliktstrategierna genom en inkompatibel ändring).

En placering bland övriga Linux-alternativ (Rclone, aviserad skrivbordsklient) finns i statusartikeln [Proton Drive under Linux](/blog/proton-drive-linux-status).

## Kommandoöversikt

Kommandona är organiserade i grupper: `proton-drive <gruppe> <befehl> [optionen] [argumente]`. Gruppnamn kan förkortas så länge de förblir entydiga; för `filesystem` finns dessutom aliaset `fs`. Utan argument startar ett interaktivt skal. Fullständig hjälp fås med `proton-drive help` respektive `proton-drive <gruppe> <befehl> --help`.

<details class="options-details">
<summary>Alternativ i översikt</summary>

| Grupp / kommando | Funktion |
|---|---|
| `auth login` / `auth logout` | Inloggning via webbläsaren; utloggning tar bort lokala inloggningsuppgifter och cachar |
| `filesystem list <pfad>` | Lista innehållet i en mapp; `/` visar rotområdena |
| `filesystem info` / `size` | Metadata för ett objekt respektive storleken på en mapp inklusive innehåll i papperskorgen |
| `filesystem upload` / `download` | Ladda upp respektive ned filer och mappar |
| `filesystem create-folder`, `rename`, `copy`, `move` | Skapa, byta namn på, kopiera och flytta mappar |
| `filesystem trash` / `restore` | Flytta till papperskorgen respektive återställa |
| `filesystem delete` / `empty-trash` | Radera permanent respektive tömma `/trash` |
| `sharing status <pfad>` | Visa medlemmar, öppna inbjudningar och länkinställningar |
| `sharing invite` / `remove` | Bjuda in personer via e-post respektive återkalla åtkomst |
| `sharing set-url` / `remove-url` | Skapa, ändra eller ta bort offentlig länk |
| `sharing leave` / `report` | Lämna en delning som delats med dig respektive rapportera den som missbruk |
| `invitation list` / `accept` / `reject` | Hantera mottagna inbjudningar |
| `album …`, `photo timeline`, `photo upload`, `photo download` | Proton Photos: album och tidslinje |
| `version` | Visa versioner för CLI och SDK |
| `--json` (`-j`) | Maskinläsbar JSON-utdata för alla kommandon |
| `--verbose` (`-v`) | Loggutdata direkt på konsolen |
| `--help` (`-h`) | Hjälp för respektive kommando |

</details>

Sökvägarna i Proton Drive är alltid POSIX-sökvägar, även i Windows. Roten `/` innehåller virtuella områden: `/my-files` (egna filer), `/devices` (säkerhetskopierade datorer), `/shared-by-me`, `/shared-with-me`, `/trash` samt Photos-områdena `/photos`, `/albums`, `/photos-shared-by-me`, `/photos-shared-with-me` och `/photos-trash`.

## Installation i Linux

Proton tillhandahåller byggen på en egen nedladdningssida, vart och ett med SHA-512-kontrollsumma. För Linux finns fem varianter:

| Build | Användning |
|---|---|
| `linux/x64` | Standard för aktuella x86-64-system |
| `linux/x64-baseline` | x86-64 utan AVX2, t.ex. NAS-enheter och äldre serverprocessorer |
| `linux/arm64` | ARM-servrar och enkortsdatorer med glibc |
| `linux/x64-musl`, `linux/arm64-musl` | Distributioner med musl i stället för glibc, t.ex. Alpine Linux och containeravbildningar som bygger på det |

Om standardbygget avbryts vid start med `Illegal instruction` saknar processorn AVX2-tillägget; då är bygget `x64-baseline` rätt val. Filen innehåller Bun-körtiden och behöver inga ytterligare beroenden:

```bash
chmod +x proton-drive
sudo install -m 0755 proton-drive /usr/local/bin/proton-drive
proton-drive version
```

Utan administratörsrättigheter räcker det att kopiera filen till `~/.local/bin`, om denna katalog finns i `PATH`.

## Inloggning, även på servrar utan skrivbordsmiljö

`auth login` frågar inte efter något lösenord på kommandoraden. CLI:t försöker öppna en webbläsare och visar dessutom inloggnings-URL:en. Denna URL kan öppnas **på en annan enhet**; terminalen väntar tills inloggningen där har slutförts. Tvåfaktorsautentiseringen sker normalt i webbläsaren. Därmed fungerar inloggningen även via SSH på en server utan grafiskt gränssnitt.

```bash
proton-drive auth login
```

Efter lyckad inloggning sparar CLI:t sessionen, inte lösenordet. Var den sparas bestäms av miljövariabeln `PROTON_DRIVE_CREDENTIALS_STORE`:

| Värde | Lagringsplats |
|---|---|
| `keychain` (standard) | Operativsystemets nyckelring: Windows Credential Manager, macOS Keychain, i Linux libsecret (GNOME Keyring, KWallet) |
| `pass` | GPG-krypterad post `ch.proton.drive/drive-sdk-cli/auth-session` i lösenordshanteraren [pass](https://www.passwordstore.org/) |
| `unsafe_file` | Klartextfilen `auth-session.json` i datakatalogen; enligt Proton endast för tester |

På en server utan skrivbordssession saknas vanligen en upplåst libsecret-nyckelring. För detta fall finns alternativet `pass` sedan version 0.6.0. Användaren som kör skripten behöver då en initierad lösenordsbutik och en GPG-nyckel som `gpg-agent` kan låsa upp utan interaktiv inmatning. Sessionen måste hittas via samma variabel vid varje anrop och alltså även vara angiven i Cron-jobb och systemd-enheter:

```bash
export PROTON_DRIVE_CREDENTIALS_STORE=pass
proton-drive auth login
```

Jämfört med Rclone är detta ett framsteg: lösenord och TOTP-nyckel finns inte på servern, och `auth logout` avslutar åtkomsten. Sessionen har dock fortfarande kontots fulla behörighet. Det går inte att begränsa den till enskilda mappar eller skrivskyddad åtkomst. För automatiserade processer är därför ett separat Proton-konto fortfarande det säkrare alternativet.

Cache, applikationsdata och loggar finns i Linux i XDG-katalogerna (`~/.cache/proton-drive-cli`, `~/.local/share/proton-drive-cli`, `~/.local/state/proton-drive-cli`). Med `PROTON_DRIVE_CACHE_DIR` kan alla tre placeras i en enda katalog, exempelvis för en container med en monterad volym. CLI:t skriver som standard loggar med nivån `DEBUG`; `PROTON_DRIVE_LOG_LEVEL=WARNING` minskar mängden.

## Uppladdning och nedladdning i skript

Interaktivt frågar CLI:t vid varje namnkonflikt vad som ska hända. I skript är detta inte möjligt: med `--json` stängs den interaktiva frågan av. Ange därför alltid konfliktstrategin uttryckligen för filer och mappar.

```bash
proton-drive filesystem upload --json \
  --file-conflict-strategy create-new-revision \
  --folder-conflict-strategy merge \
  --skip-thumbnails \
  /srv/export/berichte /my-files/backup
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Funktion |
|---|---|
| `--json` (`-j`) | Ge resultatet som JSON; stänger av interaktiva frågor |
| `--file-conflict-strategy` (`-f`) | Beteende om en fil med samma namn finns: `create-new-revision` (ny version av befintlig fil), `rename` (lägg till suffix), `replace` (fjärrfilen till papperskorgen, ladda upp den lokala), `skip` |
| `--folder-conflict-strategy` (`-d`) | Beteende vid befintlig mapp: `merge` (sammanfoga innehåll), `rename`, `replace`, `skip` |
| `--skip-thumbnails` (`-t`) | Skapa inga miniatyrbilder; sparar beräkningstid för bilder |
| `/srv/export/berichte` | Lokal källa; flera källor är möjliga |
| `/my-files/backup` | Målmapp i Proton Drive (sista argumentet) |

</details>

`create-new-revision` är rätt val för säkerhetskopior: Proton Drive behåller tidigare versioner av en fil, och filer med oförändrat innehåll hoppas automatiskt över av CLI:t sedan version 0.7.0. CLI:t jämför dock inte: lokalt raderade filer blir kvar i Proton Drive. Den som behöver en spegel med raderingar är fortfarande beroende av `rclone sync`.

Nedladdningen fungerar spegelvänt. Strategierna skiljer sig eftersom den lokala sidan skrivs över här:

```bash
proton-drive filesystem download --json \
  --file-conflict-strategy remove \
  --folder-conflict-strategy merge \
  /my-files/backup/berichte /srv/restore
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Funktion |
|---|---|
| `--file-conflict-strategy` (`-f`) | `rename`, `remove` (radera lokal fil och ladda ned fjärrversionen) eller `skip` |
| `--folder-conflict-strategy` (`-d`) | `merge`, `rename`, `remove` eller `skip` |
| `/my-files/backup/berichte` | Källa i Proton Drive; flera källor är möjliga |
| `/srv/restore` | Lokal målmapp (sista argumentet) |

</details>

CLI:t hoppar över Proton Docs och Proton Sheets vid nedladdning; de kan för närvarande inte exporteras som filer.

En regelbunden uppladdning kan planeras med en systemd-timer eller Cron. JSON-utdata kan sedan utvärderas med `jq`, exempelvis för en avisering till övervakningen.

## Hantera delningar

Vid offboarding eller revisioner är hanteringen av delningar ofta mer användbar än filöverföringen. `sharing status` visar alla medlemmar, öppna inbjudningar och inställningarna för en offentlig länk för ett objekt:

```bash
proton-drive sharing status --json /my-files/projekte/kunde-a
```

En inbjudan med läsbehörighet:

```bash
proton-drive sharing invite \
  --user person@example.com \
  --role viewer \
  /my-files/projekte/kunde-a
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Funktion |
|---|---|
| `--user` (`-u`) | E-postadress till den inbjudna personen; kan anges flera gånger |
| `--role` (`-r`) | Roll, standard `viewer`; ytterligare roller enligt `--help` (t.ex. `editor`) |
| `--message` (`-m`) | Meddelande i inbjudningsmejlet; skickas i **klartext** |
| `--include-node-name` (`-n`) | Ta med objektets namn i inbjudningsmejlet; också klartext |
| `/my-files/projekte/kunde-a` | Objekt som ska delas |

</details>

En offentlig länk med lösenord och utgångsdatum:

```bash
proton-drive sharing set-url \
  --role viewer \
  --password 'Linkpasswort' \
  --expiration 2026-12-31 \
  /my-files/projekte/kunde-a/bericht.pdf
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Funktion |
|---|---|
| `--role` | `viewer` (standard) eller `editor` |
| `--password` | Eget lösenord för länken |
| `--expiration` | Utgångsdatum i ISO-format (`JJJJ-MM-TT`) |
| `/my-files/…/bericht.pdf` | Objektet som länken skapas eller ändras för |

</details>

Ett lösenord som anges på kommandoraden hamnar i skalhistoriken och syns i processlistan medan det körs. I skript bör det därför komma från en variabel eller en hemlighetslagring. `sharing remove-url` tar bort länken utan att påverka direkta medlemmar; `sharing remove --user …` återkallar åtkomsten för enskilda personer.

## Under arbete: Takeout

I SDK-repositoriet finns sedan den 10 september 2026 ytterligare ett kommando, `takeout run`. Det exporterar en offlinekopia av kontot till en lokal mapp, valfritt med `--include my-files`, `devices`, `photos` och `revisions` (alla tidigare filversioner). Till varje mapp skriver det en `manifest.json`, som beskriver exporten; kontot ändras inte. Kommandot finns ännu inte i den släppta versionen 0.8.0. När det kommer är det den självklara vägen för en fullständig lokal säkerhetskopia av Proton Drive-innehållet.

## Begränsningar jämfört med Rclone

| Krav | Proton Drive CLI 0.8.0 | Rclone (`protondrive`-backend) |
|---|---|---|
| Officiellt stödd | Ja, av Proton, öppen källkod | Nej, reverse engineering, beta |
| Inloggning | Webbläsare, även på annan enhet; session i nyckelring eller `pass` | Lösenord och TOTP-nyckel i konfigurationsfilen |
| Uppladdning, nedladdning | Ja, med konfliktstrategier och versionshantering | Ja |
| Spegel med raderingar (`sync`) | Nej | Ja |
| Montera filsystem (FUSE) | Nej | Ja |
| Delningar, inbjudningar, länkar | Ja | Nej |
| Proton Photos | Ja | Nej |
| Åtkomst med begränsad omfattning | Nej | Nej |

För säkerhetskopior, buildartefakter och hantering av delningar är CLI:t det bättre valet eftersom det stöds officiellt och fungerar utan sparat lösenord. För en montering, som exempelvis ett [Paperless-dokumentarkiv](/blog/paperless-dokumente-clouddienst-auslagern) behöver, och för speglingar med raderingar är Rclone tills vidare nödvändigt. Det största gapet är detsamma i båda fallen: Proton erbjuder ingen maskinåtkomst som kan begränsas till enskilda mappar eller läsbehörighet.

## Källor

1.  [Proton: Introducing Proton Drive CLI](https://proton.me/blog/proton-drive-cli): tillkännagivandet från den 9 juni 2026 med användningsområde och JSON-utdata.

2.  [Proton Support: Using Proton Drive CLI](https://proton.me/support/drive-cli): guide för nedladdning, inloggning och grundläggande kommandon.

3.  [Proton Drive CLI: Downloads](https://proton.me/download/drive/cli/index.html): aktuell version 0.8.0, alla plattformsbyggen med SHA-512-kontrollsummor.

4.  [ProtonDriveApps/sdk: cli/README.md](https://github.com/ProtonDriveApps/sdk/blob/main/cli/README.md): miljövariabler, lagringsplatser, credential stores och informationen om bygget `x64-baseline`.

5.  [ProtonDriveApps/sdk: cli/CHANGELOG.md](https://github.com/ProtonDriveApps/sdk/blob/main/cli/CHANGELOG.md): versionshistorik från 0.4.2 till 0.8.0, bland annat stöd för `pass` (0.6.0) och att oförändrade filer hoppas över (0.7.0).

6.  [ProtonDriveApps/sdk: cli/src/commands](https://github.com/ProtonDriveApps/sdk/tree/main/cli/src/commands): kommandonas källkod med alternativ, konfliktstrategier och det ännu opublicerade Takeout-kommandot.

7.  [Proton for Business: Proton Drive CLI](https://proton.me/business/drive/cli): Protons användningsscenarier för företag, exempelvis att återkalla delningar när medarbetare slutar.

8.  [Rclone: Proton Drive](https://rclone.org/protondrive/): community-backenden med monterings- och synkfunktion som jämförelse.
