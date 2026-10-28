---
title: "Proton Drive CLI: använd Proton Drive från skript och servern"
navTitle: "Proton Drive CLI"
description: "Sedan juni 2026 erbjuder Proton ett officiellt kommandoradsverktyg för Proton Drive. Artikeln beskriver kommandon, inloggning på servrar utan skrivbordsmiljö, konfliktstrategier för skript och begränsningarna jämfört med Rclone."
date: "2026-10-01"
kategorie: "Proton Drive"
timeToRead: "9 min läsning"
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
translationSourceHash: ce73cccdecdcdea611f1ead28872dab00bc33a7106e2eaefd51bee4241d8631c
translationModel: gpt-5.6-terra
translatedAt: 2026-10-27T09:16:57.721Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/proton-drive-cli-hantera-proton-drive-fran-skript-och-servrar
---

Den 9 juni 2026 släppte Proton **Proton Drive CLI**, ett officiellt kommandoradsverktyg för Windows, macOS och Linux. Det bygger på samma SDK som de officiella Drive-apparna, krypterar end-to-end och finns som en enskild körbar fil `proton-drive`. Källkoden finns i det offentliga SDK-repositoriet på `cli/`.

CLI:n är avsedd för enskilda, tidsbestämda åtgärder: ladda upp filer efter en build, säkerhetskopiera en mapp enligt ett schema, granska eller återkalla delningar. Den synkroniserar inte i bakgrunden och monterar inget filsystem. Den aktuella versionen är **0.9.0 från 5 oktober 2026**; versionsnumret visar att kommandon och alternativ fortfarande kan ändras (version 0.8.0 döpte om konfliktstrategierna med en inkompatibel ändring).

En jämförelse med övriga Linux-alternativ (Rclone, annonserad skrivbordsklient) finns i statusartikeln [Proton Drive på Linux](/blog/proton-drive-linux-status).

## Kommandoöversikt

Kommandona är organiserade i grupper: `proton-drive <gruppe> <befehl> [optionen] [argumente]`. Gruppnamn kan förkortas så länge de förblir entydiga; för `filesystem` finns dessutom aliaset `fs`. Utan argument startar ett interaktivt skal. Fullständig hjälp fås med `proton-drive help` respektive `proton-drive <gruppe> <befehl> --help`.

<details class="options-details">
<summary>Alternativ i korthet</summary>

| Grupp / kommando | Funktion |
|---|---|
| `auth login` / `auth logout` | Inloggning via webbläsaren; utloggning raderar lokala inloggningsuppgifter och cacheminnen |
| `filesystem list <pfad>` | Lista innehållet i en mapp; `/` visar rotområdena |
| `filesystem info` / `size` | Metadata för ett objekt respektive storleken på en mapp inklusive innehåll i papperskorgen |
| `filesystem upload` / `download` | Ladda upp respektive ned filer och mappar |
| `filesystem create-folder`, `rename`, `copy`, `move` | Skapa, byta namn på, kopiera och flytta mappar |
| `filesystem trash` / `restore` | Flytta till papperskorgen respektive återställa |
| `filesystem delete` / `empty-trash` | Radera permanent respektive tömma `/trash` |
| `sharing status <pfad>` | Visa medlemmar, öppna inbjudningar och länkinställningar |
| `sharing invite` / `remove` | Bjuda in personer via e-post respektive återkalla åtkomst |
| `sharing set-url` / `remove-url` | Skapa, ändra eller ta bort offentlig länk |
| `sharing leave` / `report` | Lämna en delning som har delats med dig respektive rapportera den som missbruk |
| `invitation list` / `accept` / `reject` | Hantera mottagna inbjudningar |
| `album …`, `photo timeline`, `photo upload`, `photo download` | Proton Photos: album och tidslinje |
| `takeout run` | Exportera en offlinekopia av kontot till en lokal mapp (sedan 0.9.0) |
| `version` | Visa versioner av CLI och SDK |
| `--json` (`-j`) | Maskinläsbar JSON-utdata, för varje kommando |
| `--verbose` (`-v`) | Loggutdata direkt i konsolen |
| `--help` (`-h`) | Hjälp för respektive kommando |

</details>

Sökvägarna i Proton Drive är alltid POSIX-sökvägar, även i Windows. Roten `/` innehåller virtuella områden: `/my-files` (egna filer), `/devices` (säkerhetskopierade datorer), `/shared-by-me`, `/shared-with-me`, `/trash` samt Photos-områdena `/photos`, `/albums`, `/photos-shared-by-me`, `/photos-shared-with-me` och `/photos-trash`.

## Installation på Linux

Proton tillhandahåller byggen på en egen nedladdningssida, vart och ett med SHA-512-kontrollsumma. För Linux finns fem varianter:

| Build | Användning |
|---|---|
| `linux/x64` | Standard för moderna x86-64-system |
| `linux/x64-baseline` | x86-64 utan AVX2, till exempel NAS-enheter och äldre server-CPU:er |
| `linux/arm64` | ARM-servrar och enkortsdatorer med glibc |
| `linux/x64-musl`, `linux/arm64-musl` | Distributioner med musl i stället för glibc, till exempel Alpine Linux och containeravbildningar baserade på det |

Om standardbygget avbryts vid start med `Illegal instruction` saknar CPU:n AVX2-tillägget; då är bygget `x64-baseline` rätt val. Filen har Bun-körtiden inbäddad och behöver inga ytterligare beroenden:

```bash
chmod +x proton-drive
sudo install -m 0755 proton-drive /usr/local/bin/proton-drive
proton-drive version
```

Utan administratörsrättigheter räcker det att kopiera filen till `~/.local/bin`, förutsatt att denna katalog finns i `PATH`.

## Inloggning, även på servrar utan skrivbordsmiljö

`auth login` frågar inte efter något lösenord på kommandoraden. CLI:n försöker öppna en webbläsare och visar även inloggnings-URL:en. Denna URL kan öppnas **på en annan enhet**; terminalen väntar tills inloggningen där är slutförd. Tvåfaktorsautentiseringen sker som vanligt i webbläsaren. Därmed fungerar inloggningen även via SSH på en server utan grafiskt gränssnitt.

```bash
proton-drive auth login
```

Efter lyckad inloggning sparar CLI:n sessionen, inte lösenordet. Var den lagras bestäms av miljövariabeln `PROTON_DRIVE_CREDENTIALS_STORE`:

| Värde | Lagringsplats |
|---|---|
| `keychain` (standard) | Operativsystemets nyckellager: Windows Credential Manager, macOS Keychain, i Linux libsecret (GNOME Keyring, KWallet) |
| `pass` | GPG-krypterad post `ch.proton.drive/drive-sdk-cli/auth-session` i lösenordshanteraren [pass](https://www.passwordstore.org/) |
| `unsafe_file` | Klartextfilen `auth-session.json` i datakatalogen; enligt Proton endast för tester |

På en server utan skrivbordssession saknas normalt en upplåst libsecret-nyckelring. För detta fall finns alternativet `pass` sedan version 0.6.0. Användaren som skripten körs under behöver för detta en initierad lösenordsbutik och en GPG-nyckel som `gpg-agent` kan låsa upp utan interaktiv inmatning. Sessionen måste hittas via samma variabel vid varje anrop, alltså även vara angiven i Cron-jobb och systemd-enheter:

```bash
export PROTON_DRIVE_CREDENTIALS_STORE=pass
proton-drive auth login
```

Jämfört med Rclone är detta ett framsteg: lösenord och TOTP-nyckel finns inte på servern, och `auth logout` avslutar åtkomsten. Sessionen har dock fortfarande hela kontots omfattning. Det finns ingen begränsning till enskilda mappar eller skrivskyddad åtkomst. För automatiserade flöden förblir därför ett separat Proton-konto det säkrare alternativet.

Cache, applikationsdata och loggar finns i Linux i XDG-katalogerna (`~/.cache/proton-drive-cli`, `~/.local/share/proton-drive-cli`, `~/.local/state/proton-drive-cli`). Med `PROTON_DRIVE_CACHE_DIR` kan alla tre placeras i en enda katalog, exempelvis för en container med monterad volym. CLI:n skriver som standard loggar med nivån `DEBUG`; `PROTON_DRIVE_LOG_LEVEL=WARNING` minskar mängden.

## Uppladdning och nedladdning i skript

Interaktivt frågar CLI:n vid varje namnkonflikt vad som ska ske. I skript är det inte möjligt: med `--json` är den interaktiva frågan avstängd. Ange därför alltid konfliktstrategin för filer och mappar uttryckligen.

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
| `--file-conflict-strategy` (`-f`) | Beteende när en fil med samma namn finns: `create-new-revision` (ny version av den befintliga filen), `rename` (lägg till suffix), `replace` (fjärrfilen till papperskorgen, ladda upp den lokala), `skip` |
| `--folder-conflict-strategy` (`-d`) | Beteende för befintlig mapp: `merge` (sammanfoga innehåll), `rename`, `replace`, `skip` |
| `--skip-thumbnails` (`-t`) | Skapa inga miniatyrbilder; sparar beräkningstid för bilder |
| `/srv/export/berichte` | Lokal källa; flera källor är möjliga |
| `/my-files/backup` | Målkatalog i Proton Drive (sista argumentet) |

</details>

`create-new-revision` är rätt val för säkerhetskopior: Proton Drive behåller tidigare versioner av en fil, och CLI:n hoppar automatiskt över filer med oförändrat innehåll sedan version 0.7.0. CLI:n synkroniserar dock inte: lokalt raderade filer finns kvar i Proton Drive. Den som behöver en spegel med raderingar är fortfarande hänvisad till `rclone sync`.

Nedladdning fungerar spegelvänt. Strategierna skiljer sig åt eftersom den lokala sidan skrivs över här:

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
| `/srv/restore` | Lokal målkatalog (sista argumentet) |

</details>

CLI:n hoppar över Proton Docs och Proton Sheets vid nedladdning; de kan för närvarande inte exporteras som filer.

En regelbunden uppladdning kan schemaläggas med en systemd-timer eller Cron. JSON-utdata kan sedan utvärderas med `jq`, exempelvis för ett meddelande till övervakningen.

## Hantera delningar

För offboarding eller revisioner är delningshanteringen ofta mer användbar än filöverföring. `sharing status` visar alla medlemmar, öppna inbjudningar och inställningarna för en offentlig länk för ett objekt:

```bash
proton-drive sharing status --json /my-files/projekte/kunde-a
```

En inbjudan med läsrättigheter:

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
| `--role` (`-r`) | Roll, standard `viewer`; fler roller enligt `--help` (t.ex. `editor`) |
| `--message` (`-m`) | Meddelande i e-postinbjudan; skickas i **klartext** |
| `--include-node-name` (`-n`) | Ta med objektets namn i e-postinbjudan; också klartext |
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
| `/my-files/…/bericht.pdf` | Objekt som länken skapas eller ändras för |

</details>

Ett lösenord som anges på kommandoraden finns i skalhistoriken och syns i processlistan medan körningen pågår. I skript bör det därför komma från en variabel eller ett hemlighetslager. `sharing remove-url` tar bort länken utan att påverka direkta medlemmar; `sharing remove --user …` återkallar åtkomsten för enskilda personer.

## Takeout: export av kontot

Sedan version 0.9.0 från 5 oktober 2026 innehåller CLI:n kommandot `takeout run`. Det exporterar en offlinekopia av kontot till en lokal mapp, valfritt med `--include my-files`, `devices`, `photos` och `revisions` (alla tidigare filversioner). Utan minst en `--include`-angivelse exporterar det ingenting, och `revisions` förutsätter `my-files` eller `devices`. För varje mapp skriver det en `manifest.json` som beskriver exporten; kontot ändras inte. Kommandot stöder ännu inte Proton Docs och Proton Sheets. För en fullständig lokal säkerhetskopia av Proton Drive-innehållet är det den naturliga vägen.

## Begränsningar jämfört med Rclone

| Krav | Proton Drive CLI 0.9.0 | Rclone (`protondrive`-backend) |
|---|---|---|
| Officiellt stödd | Ja, av Proton, öppen källkod | Nej, reverse engineering, beta |
| Inloggning | Webbläsare, även på en annan enhet; session i nyckellagret eller `pass` | Lösenord och TOTP-nyckel i konfigurationsfilen |
| Uppladdning, nedladdning | Ja, med konfliktstrategier och versionshantering | Ja |
| Spegel med raderingar (`sync`) | Nej | Ja |
| Montera filsystem (FUSE) | Nej | Ja |
| Delningar, inbjudningar, länkar | Ja | Nej |
| Proton Photos | Ja | Nej |
| Åtkomst med begränsad omfattning | Nej | Nej |

För säkerhetskopior, buildartefakter och delningshantering är CLI:n det bättre valet, eftersom den stöds officiellt och inte kräver något sparat lösenord. För en montering, som exempelvis ett [Paperless-dokumentarkiv](/blog/paperless-dokumente-clouddienst-auslagern) behöver, och för speglingar med raderingar är Rclone tills vidare nödvändigt. Den största luckan är densamma i båda fallen: Proton erbjuder ingen maskinåtkomst som kan begränsas till enskilda mappar eller läsbehörighet.

## Ändringshistorik

- **25 oktober 2026:** Uppdaterat till version 0.9.0 från 5 oktober 2026. Tillagt: Kommandot takeout run har därmed släppts, inklusive information om nödvändiga --include-angivelser och avsaknaden av stöd för Proton Docs och Sheets.

## Källor

1.  [Proton: Introducing Proton Drive CLI](https://proton.me/blog/proton-drive-cli): tillkännagivandet från 9 juni 2026 med användningsområden och JSON-utdata.

2.  [Proton Support: Using Proton Drive CLI](https://proton.me/support/drive-cli): anvisningar för nedladdning, inloggning och grundläggande kommandon.

3.  [Proton Drive CLI: Downloads](https://proton.me/download/drive/cli/index.html): aktuell version 0.9.0, alla plattformsbyggen med SHA-512-kontrollsummor.

4.  [ProtonDriveApps/sdk: cli/README.md](https://github.com/ProtonDriveApps/sdk/blob/main/cli/README.md): miljövariabler, lagringsplatser, credential-stores och anmärkningen om bygget `x64-baseline`.

5.  [ProtonDriveApps/sdk: cli/CHANGELOG.md](https://github.com/ProtonDriveApps/sdk/blob/main/cli/CHANGELOG.md): versionshistorik från 0.4.2 till 0.9.0, bland annat stöd för `pass` (0.6.0), överhoppning av oförändrade filer (0.7.0) och Takeout-kommandot (0.9.0).

6.  [ProtonDriveApps/sdk: cli/src/commands](https://github.com/ProtonDriveApps/sdk/tree/main/cli/src/commands): kommandonas källkod med alternativ, konfliktstrategier och Takeout-kommandot.

7.  [Proton for Business: Proton Drive CLI](https://proton.me/business/drive/cli): Protons användningsscenarier för företag, exempelvis att återkalla delningar när medarbetare slutar.

8.  [Rclone: Proton Drive](https://rclone.org/protondrive/): community-backenden med monterings- och synkfunktion som jämförelse.
