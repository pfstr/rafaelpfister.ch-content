---
title: "git.kernel.org forklart: sidens oppbygning og praktisk bruk"
navTitle: "git.kernel.org"
description: "git.kernel.org er Git-hostingen for utviklingen av Linux-kjernen. Oversikten forklarer repository-listen, stiskjemaet, visningene summary, log, tree og commit, søkeparametrene i URL-en og kort-URL-ene som Kernel-CVE-er viser til."
date: "2026-10-29"
kategorie: "Linux-kjernen"
timeToRead: "9 min lesetid"
themen:
  - linux-kernel
produkte:
  - "linux-kernel"
protokolle:
  - "releases"
  - "troubleshooting"
slug: "git-kernel-org-forklart-sidens-oppbygning-og-praktisk-bruk"
translationId: "article-91dab903d58c6173"
translationOf: git-kernel-org-erklaert
url: https://rafaelpfister.ch/no/blog/git-kernel-org-forklart-sidens-oppbygning-og-praktisk-bruk
translationSourceHash: 34a3d0d525d79d1ef4c5ff31824dabdd6e910a1dd5a22e60c77b202f97a71b7d
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T14:35:46.603Z
translationReview: automatic
---

[git.kernel.org](https://git.kernel.org/) er Git-serveren til kernel.org. Her ligger hovedutviklingsgrenen til Linus Torvalds, Stable-kjernene, linux-next, hundrevis av delsystem- og utviklertrær samt verktøy som `git` selv, `e2fsprogs` eller Linux-man-sidene. Nettgrensesnittet er en tilpasset versjon av [cgit](https://git.zx2c4.com/cgit/about/) (oppgitt som `cgit 1.3.1-korg` i bunnteksten). Det er skrivebeskyttet: Det finnes ingen kontoer, issues eller pull requests. Kjerneutviklingen foregår via e-postlister, med arkiv på [lore.kernel.org](https://lore.kernel.org/); git.kernel.org viser resultatet, altså commitene som vedlikeholderne har tatt inn i trærne sine.

Typiske grunner til å besøke siden er en CVE-oppføring som viser til en kjerne-commit, spørsmålet om en bestemt feilretting er inkludert i en kjerneversjon, eller å lese en enkelt kildefil uten å klone hele repositoryet.

## Startsiden: repository-listen

Startsiden lister opp alle offentlige repositoryer i en tabell. Grupperingen følger filstien på serveren: Hver mellomoverskrift som `pub/scm/fs` eller `pub/scm/linux` er en katalog, og under den står repositoryene med stien sin relativt til denne katalogen.

<details class="options-details">
<summary>Kolonneoversikt</summary>

| Kolonne | Innhold |
|---|---|
| Name | Sti til repositoryet under gruppen, for eksempel `xfs/xfs-linux.git` under `pub/scm/fs` |
| Description | Innholdet i filen `description` i repositoryet. «Unnamed repository; edit this file …» er standardteksten i Git og betyr bare at operatøren ikke har angitt en beskrivelse |
| Owner | Ansvarlig person eller gruppe (for eksempel «XFS FS Group», «Infra Team») |
| Idle | Tid siden siste commit på en hvilken som helst branch; en lav verdi viser et aktivt vedlikeholdt tre |
| Links | Direkte lenker til repositoryets visninger `summary`, `log` og `tree` |

</details>

De viktigste gruppene:

| Gruppe | Innhold |
|---|---|
| `pub/scm/linux` | Kjernetrær: mainline, stable, linux-next samt vedlikeholdernes personlige trær og delsystemtrær |
| `pub/scm/fs` | Filsystemer med Userspace-verktøy (ext2/3/4, XFS, fscrypt, fsverity) |
| `pub/scm/libs` | Biblioteker som `libcap`, `libgpiod`, `libtraceevent` |
| `pub/scm/git` | Selve Git-prosjektet (`git.git`) med HTML-dokumentasjon og man-sider |
| `pub/scm/docs` | Dokumentasjonskilder, blant annet `man-pages` og PGP-nøklene til kjerneutviklerne (`kernel/pgpkeys.git`) |
| `pub/scm/infra` | Infrastruktur for kernel.org, for eksempel cgit-forken og transparensloggene til Gitolite-serveren |
| `pub/scm/bluetooth`, `devel`, `editors` | BlueZ, utviklerverktøy som `sparse` og `pahole`, Linus Torvalds' `uemacs` |

Listen er svært lang fordi hver vedlikeholder med konto kan legge egne trær under `pub/scm/linux/kernel/git/`. Søket øverst til høyre filtrerer listen etter navn, beskrivelse og eier. Søket kan også åpnes direkte som URL, for eksempel `https://git.kernel.org/?q=xfs`.

## Stiskjemaet for kjernetrær

Kjernerepositoryer følger mønsteret:

```text
https://git.kernel.org/pub/scm/linux/kernel/git/<konto>/<repository>.git/
```

`<konto>` er vedlikeholderens brukernavn på kernel.org, ikke nødvendigvis fullt navn (`akpm` er Andrew Morton, `acme` er Arnaldo Carvalho de Melo). For de fleste spørsmål holder det med tre trær:

| Tre | Sti | Formål |
|---|---|---|
| Mainline | `torvalds/linux.git` | Utviklingsgrenen til Linus Torvalds; her oppstår utgivelsene `v7.3`, `v7.4-rc1` osv. |
| Stable | `stable/linux.git` | Vedlikeholdte versjoner med feilrettinger (`v7.3.1`, `v7.3.2` …) og Longterm-kjernene; én branch per versjonsserie, for eksempel `linux-7.3.y` |
| linux-next | `next/linux-next.git` | Daglig sammenslått status for alle delsystemtrær før neste merge-vindu |

En feilretting går vanligvis gjennom disse trærne i denne rekkefølgen: Den havner først i treet til det ansvarlige delsystemet, vises i linux-next, tas inn i mainline av Linus Torvalds i merge-vinduet via en pull request-e-post, og tilbakeporteres deretter til Stable-grenene. Hvilket delsystemtre som er ansvarlig, står i filen `MAINTAINERS` i kjernekildekoden (linjer med `T: git …`).

## Visningene i et repository

Et klikk på et repository-navn åpner visningen `summary`. Fanelinjen over fører til de andre visningene; til venstre for den velger et valgfelt branchen som alle visningene gjelder.

<details class="options-details">
<summary>Oversikt over visninger</summary>

| Visning | Innhold |
|---|---|
| about | Repositoryets README, dersom den finnes |
| summary | Brancher med siste commit, de nyeste taggene med nedlastingslenke, de siste ti commitene og klone-URL-ene |
| refs | Fullstendig liste over alle brancher og tagger |
| log | Commit-historikk med søkefelt; kolonnene alder, commit-melding, forfatter, endrede filer og linjer |
| tree | Filtre for valgt branch eller commit, med lenker til `plain` (råfil) og `log` (filhistorikk) |
| commit | Metadata og diff for én enkelt commit |
| diff | Forskjell mellom to commiter eller mellom en commit og dens forgjenger |
| stats | Commit-statistikk per forfatter over en valgfri periode |

</details>

I visningen `summary` er taggblokken nyttig for å lese av gjeldende versjon: Mainline viser release candidates (`v7.3-rc7`) og utgivelser (`v7.3`). Nedlastingslenkene oppretter et arkiv på forespørsel og er derfor ikke alltid tilgjengelige; for ferdige utgivelser er tarballene på [kernel.org](https://kernel.org/) eller `cdn.kernel.org` den tiltenkte nedlastingsmåten, også med PGP-signatur.

Nederst i visningen `summary` finnes klone-URL-ene. Ved siden av `https://` og den ukrypterte `git://` er det oppgitt et speil hos Google (`kernel.googlesource.com`) som leverer samme innhold.

## Søk i historikken

Søkefeltet i visningen `log` har et valgfelt med fire moduser. Hver modus er også tilgjengelig som URL-parameter, slik at søk kan lagres som bokmerker eller lenkes til i saker:

```text
https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?qt=grep&q=CVE
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Parameter | Virkning |
|---|---|
| `qt=grep` | Søker i commit-meldingene (valget «log msg») |
| `qt=author` | Søker i forfatterfeltet (navn eller e-postadresse) |
| `qt=committer` | Søker i committer-feltet, altså personen som tok inn commiten |
| `qt=range` | Viser et område i Git-notasjon, for eksempel `q=v7.3-rc6..v7.3-rc7` for alle commiter mellom to tagger |
| `q=` | Søkeord eller område |
| `h=` | Branch søket gjelder, for eksempel `h=linux-7.3.y` i Stable-treet |
| `ofs=` | Forskyvning for sideinndeling (git.kernel.org viser 200 commiter per side) |

</details>

Området `v7.3-rc6..v7.3-rc7` besvarer spørsmålet «Hva har endret seg mellom to versjoner?» direkte i nettleseren. I Stable-treet viser `qt=range&q=v7.3.1..v7.3.2&h=linux-7.3.y` alle feilrettinger som er lagt til med en Stable-oppdatering.

## Lese en commit

En enkelt commit kan adresseres med hele SHA-1-hashen:

```text
https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=<sha>
```

Visningen viser forfatter, committer, dato, overordnet commit (`parent`), hele commit-meldingen og diffen. På slutten av en commit-melding for kjernen står trailere som `Signed-off-by:`, `Reviewed-by:` og `Fixes: <sha> ("Titel")`. Linjen `Fixes:` angir commiten som introduserte feilen; dermed kan man avgrense fra hvilken versjon et system er berørt. `Link:` viser som regel til den tilhørende diskusjonen på lore.kernel.org, `Cc: stable@vger.kernel.org` markerer feilrettingen for tilbakeportering til Stable-kjernene.

Hvis man erstatter `commit` i URL-en med `patch`, leverer serveren commiten som en patchfil i e-postformat (`text/plain`), som kan brukes med `git am` eller `patch`.

## Kort-URL-er i CVE-oppføringer

Siden kjerneprosjektet selv begynte å tildele CVE-numre, viser oppføringene i CVE-listen og NVD til kort-URL-er på formen `https://git.kernel.org/stable/c/<sha>`. git.kernel.org videresender dem med HTTP 301 til commit-visningen i det aktuelle treet:

| Kort-URL | Mål |
|---|---|
| `https://git.kernel.org/stable/c/<sha>` | Commit i Stable-treet `stable/linux.git` |
| `https://git.kernel.org/linus/<sha>` | Commit i mainline `torvalds/linux.git` |

En CVE-oppføring angir vanligvis flere slike lenker: én per Stable-gren som feilrettingen er tilbakeportert til, og én for mainline. Hashene er forskjellige fordi hver tilbakeportering er en egen commit. Commit-meldingen for tilbakeporteringen angir hashen til originalen i mainline, enten som linjen `commit <sha> upstream.` eller som `[ Upstream commit <sha> ]`.

## Kontrollere om en feilretting finnes i en versjon

Nettgrensesnittet viser ikke hvilke tagger en commit er inkludert i. Til det trengs en lokal klone. For dette formålet holder det med en klone uten filinnhold, som bare laster ned commit-historikken:

```bash
git clone --filter=blob:none --no-checkout https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `clone` | Oppretter en lokal kopi av repositoryet |
| `--filter=blob:none` | Partial Clone: laster ned commiter og katalogstrukturer, filinnhold (blobs) først ved behov |
| `--no-checkout` | Oppretter ingen arbeidskatalog; ikke nødvendig for spørringer i historikken |
| URL | HTTPS-klone-URL fra visningen `summary` for Stable-treet |

</details>

Deretter besvarer en kommando i den klonede katalogen spørsmålet om den første versjonen med feilrettingen:

```bash
git describe --contains <sha>
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `describe` | Beskriver en commit relativt til en tagg |
| `--contains` | Finner den eldste taggen som inneholder commiten, i stedet for den nyeste taggen før den |
| `<sha>` | Hashen til commiten, for eksempel fra kort-URL-en i CVE-oppføringen |

</details>

Utdataene har formen `v7.3.2~14`: Feilrettingen er inkludert fra `v7.3.2`. Alle tagger med commiten listes av `git tag --contains <sha>`. Den som bare trenger én enkelt fil eller en status for lesing, kan bruke visningene `tree` og `plain` i nettleseren i stedet for en klone.

For fullstendig kildekode til bygging holder det med en grunn klone av en tagg:

```bash
git clone --depth 1 --branch v7.3 https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `--depth 1` | Laster bare ned siste commit uten historikk; nedlastingen reduseres fra flere GB til noen hundre MB |
| `--branch v7.3` | Velger taggen eller branchen som sjekkes ut |
| URL | Klone-URL for mainline |

</details>

## Følge endringer

Hvert repository tilbyr en Atom-feed for standardbranchen under `…/<repository>.git/atom/`, for mainline altså `https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/atom/`. En feed-leser viser dermed nye commiter uten at siden må åpnes regelmessig. For mainline produserer feeden mange oppføringer per dag under merge-vinduet; for utgivelser og Stable-oppdateringer er versjonsoversikten på [kernel.org](https://kernel.org/) eller dens feed den roligere kilden.

## Kilder

1.  [git.kernel.org](https://git.kernel.org/): Repository-liste med grupper, Owner- og Idle-kolonne; søkefelt på startsiden.

2.  [torvalds/linux.git](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/): Mainline med visningene summary, log, tree, klone-URL-er og Atom-feed.

3.  [stable/linux.git](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/): Stable- og Longterm-kjerner, én branch per versjonsserie.

4.  [cgit](https://git.zx2c4.com/cgit/about/): Upstream for nettgrensesnittet, dokumentasjon av URL-parametrene.

5.  [Kernel Docs: Everything you ever wanted to know about Linux -stable releases](https://docs.kernel.org/process/stable-kernel-rules.html): Regler for tilbakeporteringer, traileren `Cc: stable` og henvisning til upstream-commiten.

6.  [Kernel Docs: CVEs](https://docs.kernel.org/process/cve.html): CVE-tildeling gjennom kjerneprosjektet og oppbygningen av oppføringene.

7.  [Kernel Docs: Submitting patches](https://docs.kernel.org/process/submitting-patches.html): Betydningen av trailerne `Fixes:`, `Link:`, `Signed-off-by:`.

8.  [git-describe](https://git-scm.com/docs/git-describe): Man-side, alternativet `--contains`.

9.  [git-clone](https://git-scm.com/docs/git-clone): Man-side, alternativene `--filter`, `--depth`, `--branch`, `--no-checkout`.

10.  [lore.kernel.org](https://lore.kernel.org/): Arkiv for kjernens e-postlister, som trailerne `Link:` viser til.
