---
title: "git.kernel.org förklarat: sidans struktur och praktisk användning"
navTitle: "git.kernel.org"
description: "git.kernel.org är Git-hostingen för Linux-kärnans utveckling. Översikten förklarar listan över repositorier, sökvägsschemat, vyerna summary, log, tree och commit, sökparametrarna i URL:en och kort-URL:erna som kernel-CVE:er hänvisar till."
date: "2026-10-29"
kategorie: "Linux-kärnan"
timeToRead: "9 min. lästid"
themen:
  - linux-kernel
produkte:
  - "linux-kernel"
protokolle:
  - "releases"
  - "troubleshooting"
slug: "git-kernel-org-forklarat-sidans-struktur-och-praktisk-anvandning"
translationId: "article-91dab903d58c6173"
translationOf: git-kernel-org-erklaert
url: https://rafaelpfister.ch/sv/blog/git-kernel-org-forklarat-sidans-struktur-och-praktisk-anvandning
translationSourceHash: 34a3d0d525d79d1ef4c5ff31824dabdd6e910a1dd5a22e60c77b202f97a71b7d
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T14:35:01.365Z
translationReview: automatic
---

[git.kernel.org](https://git.kernel.org/) är Git-servern för kernel.org. Här finns Linus Torvalds huvudutvecklingsgren, stabila kärnor, linux-next, hundratals subsystem- och utvecklarträd samt verktyg som `git` självt, `e2fsprogs` eller Linux-manualsidorna. Webbgränssnittet är en anpassad version av [cgit](https://git.zx2c4.com/cgit/about/) (anges i sidfoten som `cgit 1.3.1-korg`). Det är skrivskyddat: det finns inga konton, inga issues och inga pull requests. Kärnutvecklingen sker via e-postlistor, vars arkiv finns på [lore.kernel.org](https://lore.kernel.org/); git.kernel.org visar resultatet, alltså de commits som maintainerna har tagit in i sina träd.

Vanliga skäl att besöka sidan är en CVE-post som hänvisar till en kernel-commit, frågan om en viss fix ingår i en kernel-version eller att läsa en enskild källfil utan att klona hela repositoriet.

## Startsidan: lista över repositorier

Startsidan listar alla offentliga repositorier i en tabell. Grupperingen följer filsökvägen på servern: varje mellanrubrik som `pub/scm/fs` eller `pub/scm/linux` är en katalog, och under den visas repositorierna med sin sökväg relativt denna katalog.

<details class="options-details">
<summary>Kolumner i översikt</summary>

| Kolumn | Innehåll |
|---|---|
| Namn | Repositoriets sökväg under gruppen, till exempel `xfs/xfs-linux.git` under `pub/scm/fs` |
| Beskrivning | Innehållet i filen `description` i repositoriet. ”Unnamed repository; edit this file …” är Git-standardtexten och betyder endast att operatören inte har angett någon beskrivning |
| Ägare | Ansvarig person eller grupp (t.ex. ”XFS FS Group”, ”Infra Team”) |
| Inaktivitet | Tid sedan senaste commit på någon branch; ett lågt värde visar ett aktivt underhållet träd |
| Länkar | Direktlänkar till repositoriets vyer `summary`, `log` och `tree` |

</details>

De viktigaste grupperna:

| Grupp | Innehåll |
|---|---|
| `pub/scm/linux` | Kernel-träd: mainline, stable, linux-next samt maintainrarnas personliga träd och subsystemträd |
| `pub/scm/fs` | Filsystem med userspace-verktyg (ext2/3/4, XFS, fscrypt, fsverity) |
| `pub/scm/libs` | Bibliotek som `libcap`, `libgpiod`, `libtraceevent` |
| `pub/scm/git` | Själva Git-projektet (`git.git`) med HTML-dokumentation och manualsidor |
| `pub/scm/docs` | Dokumentationskällor, inklusive `man-pages` och PGP-nycklarna för kernel-utvecklarna (`kernel/pgpkeys.git`) |
| `pub/scm/infra` | Infrastruktur för kernel.org, till exempel cgit-forken och Gitolite-serverns transparensloggar |
| `pub/scm/bluetooth`, `devel`, `editors` | BlueZ, utvecklarverktyg som `sparse` och `pahole`, Linus Torvalds `uemacs` |

Listan är mycket lång eftersom varje maintainer med konto kan lägga egna träd enbart under `pub/scm/linux/kernel/git/`. Sökfältet uppe till höger filtrerar listan efter namn, beskrivning och ägare. Sökningen kan även öppnas direkt som URL, till exempel `https://git.kernel.org/?q=xfs`.

## Sökvägsschemat för kernel-träden

Kernel-repositorier följer mönstret:

```text
https://git.kernel.org/pub/scm/linux/kernel/git/<konto>/<repository>.git/
```

`<konto>` är maintainerens användarnamn på kernel.org, inte nödvändigtvis personens fullständiga namn (`akpm` är Andrew Morton, `acme` är Arnaldo Carvalho de Melo). För de flesta frågor räcker tre träd:

| Träd | Sökväg | Syfte |
|---|---|---|
| Mainline | `torvalds/linux.git` | Linus Torvalds utvecklingsgren; här skapas releaserna `v7.3`, `v7.4-rc1` osv. |
| Stable | `stable/linux.git` | Underhållna versioner med fixar (`v7.3.1`, `v7.3.2` …) och longterm-kärnorna; en branch per versionsserie, till exempel `linux-7.3.y` |
| linux-next | `next/linux-next.git` | Dagligen sammanslagen version av alla subsystemträd för nästa merge window |

En fix passerar vanligtvis dessa träd i denna ordning: den hamnar först i det ansvariga subsystemets träd, visas i linux-next, tas under merge window in i mainline av Linus Torvalds via ett pull request-mejl och bakporteras därefter till stable-grenarna. Vilket subsystemträd som ansvarar framgår av filen `MAINTAINERS` i kernel-källkoden (rader med `T: git …`).

## Vyerna i ett repositorium

Ett klick på ett repositorienamn öppnar vyn `summary`. Flikraden ovanför leder till övriga vyer; till vänster om den väljer en rullista den branch som alla vyer avser.

<details class="options-details">
<summary>Vyer i översikt</summary>

| Vy | Innehåll |
|---|---|
| about | Repositoriets README, om den finns |
| summary | Brancher med senaste commit, de senaste taggarna med nedladdningslänk, de senaste tio commits och klon-URL:er |
| refs | Fullständig lista över alla brancher och taggar |
| log | Commit-historik med sökfält; kolumner för ålder, commit-meddelande, författare, ändrade filer och rader |
| tree | Filträd för vald branch eller commit, med länkar till `plain` (råfil) och `log` (filhistorik) |
| commit | Metadata och diff för en enskild commit |
| diff | Skillnad mellan två commits eller mellan en commit och dess föregångare |
| stats | Commit-statistik per författare under en valbar tidsperiod |

</details>

I vyn `summary` är taggblocket användbart för att läsa av den aktuella versionen: mainline visar release candidates (`v7.3-rc7`) och releaser (`v7.3`). Nedladdningslänkarna skapar ett arkiv på begäran och är därför inte alltid tillgängliga; för färdiga releaser är tarbollarna på [kernel.org](https://kernel.org/) respektive `cdn.kernel.org` den avsedda nedladdningsvägen, där även PGP-signaturen finns.

I slutet av vyn `summary` finns klon-URL:erna. Utöver `https://` och det okrypterade `git://` anges en spegel hos Google (`kernel.googlesource.com`) som tillhandahåller samma innehåll.

## Söka i historiken

Sökfältet i vyn `log` har en rullista med fyra lägen. Varje läge finns även som URL-parameter, så att sökningar kan sparas som bokmärken eller länkas i ärenden:

```text
https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?qt=grep&q=CVE
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Parameter | Funktion |
|---|---|
| `qt=grep` | Söker i commit-meddelanden (valet ”log msg”) |
| `qt=author` | Söker i författarfältet (namn eller e-postadress) |
| `qt=committer` | Söker i committer-fältet, alltså efter personen som tog in commiten |
| `qt=range` | Visar ett intervall i Git-notation, till exempel `q=v7.3-rc6..v7.3-rc7` för alla commits mellan två taggar |
| `q=` | Sökterm respektive intervall |
| `h=` | Branch som sökningen avser, till exempel `h=linux-7.3.y` i stable-trädet |
| `ofs=` | Förskjutning för sidindelning (git.kernel.org visar 200 commits per sida) |

</details>

Intervallet `v7.3-rc6..v7.3-rc7` besvarar frågan ”Vad ändrades mellan två versioner?” direkt i webbläsaren. I stable-trädet visar `qt=range&q=v7.3.1..v7.3.2&h=linux-7.3.y` alla fixar som tillkommit med en stable-uppdatering.

## Läsa en commit

En enskild commit kan adresseras med sin fullständiga SHA-1-hash:

```text
https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=<sha>
```

Vyn visar författare, committer, datum, överordnad commit (`parent`), det fullständiga commit-meddelandet och diffen. I slutet av ett kernel-commit-meddelande finns trailers som `Signed-off-by:`, `Reviewed-by:` och `Fixes: <sha> ("Titel")`. Raden `Fixes:` anger commiten som introducerade felet; därmed går det att avgränsa från vilken version ett system är påverkat. `Link:` hänvisar oftast till den tillhörande diskussionen på lore.kernel.org, `Cc: stable@vger.kernel.org` markerar fixen för bakportering till stable-kärnorna.

Om `commit` i URL:en ersätts med `patch`, levererar servern commiten som en patchfil i e-postformat (`text/plain`), som kan tillämpas med `git am` eller `patch`.

## Kort-URL:er i CVE-poster

Sedan kernel-projektet självt började tilldela CVE-nummer hänvisar posterna i CVE-listan och NVD till kort-URL:er av formen `https://git.kernel.org/stable/c/<sha>`. git.kernel.org omdirigerar dessa med HTTP 301 till commit-vyn i rätt träd:

| Kort-URL | Mål |
|---|---|
| `https://git.kernel.org/stable/c/<sha>` | Commit i stable-trädet `stable/linux.git` |
| `https://git.kernel.org/linus/<sha>` | Commit i mainline `torvalds/linux.git` |

En CVE-post anger vanligen flera sådana länkar: en per stable-gren som fixen har bakporterats till och en för mainline. Hasharna skiljer sig åt eftersom varje bakport är en egen commit. Bakportens commit-meddelande anger originalets hash i mainline, antingen som raden `commit <sha> upstream.` eller som `[ Upstream commit <sha> ]`.

## Kontrollera om en fix ingår i en version

Webbgränssnittet visar inte vilka taggar en commit ingår i. För det krävs en lokal klon. För detta ändamål räcker en klon utan filinnehåll, som endast laddar commit-historiken:

```bash
git clone --filter=blob:none --no-checkout https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Funktion |
|---|---|
| `clone` | Skapar en lokal kopia av repositoriet |
| `--filter=blob:none` | Partiell klon: laddar commits och katalogstrukturer, filinnehåll (blobbar) först vid behov |
| `--no-checkout` | Skapar ingen arbetskatalog; behövs inte för frågor om historiken |
| URL | HTTPS-klon-URL från vyn `summary` i stable-trädet |

</details>

Därefter besvarar ett kommando i den klonade katalogen frågan om den första versionen med fixen:

```bash
git describe --contains <sha>
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Funktion |
|---|---|
| `describe` | Beskriver en commit relativt till en tagg |
| `--contains` | Söker efter den äldsta taggen som innehåller commiten i stället för den senaste taggen före den |
| `<sha>` | Commitens hash, till exempel från kort-URL:en i CVE-posten |

</details>

Utdata har formen `v7.3.2~14`: fixen ingår från och med `v7.3.2`. Alla taggar som innehåller commiten listas av `git tag --contains <sha>`. Den som bara behöver en enskild fil eller en version för läsning kan använda vyerna `tree` och `plain` i webbläsaren i stället för en klon.

För den fullständiga källkoden för att bygga räcker en grund klon av en tagg:

```bash
git clone --depth 1 --branch v7.3 https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Funktion |
|---|---|
| `--depth 1` | Laddar endast den senaste commiten utan historik; nedladdningen minskar från flera GB till några hundra MB |
| `--branch v7.3` | Väljer taggen eller branchen som checkas ut |
| URL | Klon-URL för mainline |

</details>

## Följa ändringar

Varje repositorium erbjuder ett Atom-flöde för standardbranchen under `…/<repository>.git/atom/`, för mainline alltså `https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/atom/`. En flödesläsare visar då nya commits utan att sidan behöver besökas regelbundet. För mainline skapar flödet många poster per dag under merge window; för releaser och stable-uppdateringar är versionsöversikten på [kernel.org](https://kernel.org/) respektive dess flöde den lugnare källan.

## Källor

1.  [git.kernel.org](https://git.kernel.org/): Lista över repositorier med grupper, kolumnerna Owner och Idle samt sökfältet på startsidan.

2.  [torvalds/linux.git](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/): Mainline med vyerna summary, log, tree, klon-URL:er och Atom-flöde.

3.  [stable/linux.git](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/): Stable- och longterm-kärnor, en branch per versionsserie.

4.  [cgit](https://git.zx2c4.com/cgit/about/): Webbgränssnittets upstream, dokumentation om URL-parametrarna.

5.  [Kernel Docs: Everything you ever wanted to know about Linux -stable releases](https://docs.kernel.org/process/stable-kernel-rules.html): Regler för bakportering, trailern `Cc: stable` och hänvisning till upstream-commiten.

6.  [Kernel Docs: CVEs](https://docs.kernel.org/process/cve.html): CVE-tilldelning genom kernel-projektet och posternas struktur.

7.  [Kernel Docs: Submitting patches](https://docs.kernel.org/process/submitting-patches.html): Betydelsen av trailerarna `Fixes:`, `Link:`, `Signed-off-by:`.

8.  [git-describe](https://git-scm.com/docs/git-describe): Manualsida, alternativet `--contains`.

9.  [git-clone](https://git-scm.com/docs/git-clone): Manualsida, alternativen `--filter`, `--depth`, `--branch`, `--no-checkout`.

10.  [lore.kernel.org](https://lore.kernel.org/): Arkiv för kernel-e-postlistorna som trailerarna `Link:` hänvisar till.
