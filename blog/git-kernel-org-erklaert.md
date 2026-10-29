---
title: "git.kernel.org erklärt: Aufbau der Seite und praktische Nutzung"
navTitle: "git.kernel.org"
description: "git.kernel.org ist das Git-Hosting der Linux-Kernel-Entwicklung. Die Übersicht erklärt die Repository-Liste, das Pfadschema, die Ansichten summary, log, tree und commit, die Suchparameter in der URL und die Kurz-URLs, auf die Kernel-CVEs verweisen."
date: "2026-10-29"
kategorie: "Linux-Kernel"
timeToRead: "9 Min. Lesezeit"
themen:
  - "linux-kernel"
produkte:
  - "linux-kernel"
protokolle:
  - "releases"
  - "troubleshooting"
slug: "git-kernel-org-erklaert"
translationId: "article-91dab903d58c6173"
url: "https://rafaelpfister.ch/blog/git-kernel-org-erklaert"
---

[git.kernel.org](https://git.kernel.org/) ist der Git-Server von kernel.org. Hier liegen der Hauptentwicklungszweig von Linus Torvalds, die Stable-Kernel, linux-next, Hunderte Subsystem- und Entwicklerbäume sowie Werkzeuge wie `git` selbst, `e2fsprogs` oder die Linux-Manpages. Die Weboberfläche ist eine angepasste Version von [cgit](https://git.zx2c4.com/cgit/about/) (im Seitenfuss als `cgit 1.3.1-korg` ausgewiesen). Sie ist rein lesend: Es gibt keine Konten, keine Issues und keine Pull Requests. Die Kernel-Entwicklung läuft über Mailinglisten, deren Archiv unter [lore.kernel.org](https://lore.kernel.org/) liegt; git.kernel.org zeigt das Ergebnis, also die Commits, die Maintainer in ihre Bäume übernommen haben.

Typische Anlässe für einen Besuch sind ein CVE-Eintrag, der auf einen Kernel-Commit verweist, die Frage, ob ein bestimmter Fix in einer Kernel-Version enthalten ist, oder das Nachlesen einer einzelnen Quelldatei, ohne das ganze Repository zu klonen.

## Die Startseite: Repository-Liste

Die Startseite listet alle öffentlichen Repositories in einer Tabelle. Die Gruppierung folgt dem Dateipfad auf dem Server: Jede Zwischenüberschrift wie `pub/scm/fs` oder `pub/scm/linux` ist ein Verzeichnis, darunter stehen die Repositories mit ihrem Pfad relativ zu diesem Verzeichnis.

<details class="options-details">
<summary>Spalten im Überblick</summary>

| Spalte | Inhalt |
|---|---|
| Name | Pfad des Repositorys unterhalb der Gruppe, z. B. `xfs/xfs-linux.git` unter `pub/scm/fs` |
| Description | Inhalt der Datei `description` im Repository. „Unnamed repository; edit this file …" ist der Git-Standardtext und bedeutet nur, dass der Betreiber keine Beschreibung gesetzt hat |
| Owner | verantwortliche Person oder Gruppe (z. B. „XFS FS Group", „Infra Team") |
| Idle | Zeit seit dem letzten Commit auf irgendeinem Branch; ein kleiner Wert zeigt einen aktiv gepflegten Baum |
| Links | Direktsprünge in die Ansichten `summary`, `log` und `tree` des Repositorys |

</details>

Die wichtigsten Gruppen:

| Gruppe | Inhalt |
|---|---|
| `pub/scm/linux` | Kernel-Bäume: Mainline, Stable, linux-next sowie die persönlichen und Subsystem-Bäume der Maintainer |
| `pub/scm/fs` | Dateisysteme samt Userspace-Werkzeugen (ext2/3/4, XFS, fscrypt, fsverity) |
| `pub/scm/libs` | Bibliotheken wie `libcap`, `libgpiod`, `libtraceevent` |
| `pub/scm/git` | das Git-Projekt selbst (`git.git`) mit HTML-Dokumentation und Manpages |
| `pub/scm/docs` | Dokumentationsquellen, darunter `man-pages` und die PGP-Schlüssel der Kernel-Entwickler (`kernel/pgpkeys.git`) |
| `pub/scm/infra` | Infrastruktur von kernel.org, z. B. der cgit-Fork und die Transparenz-Logs des Gitolite-Servers |
| `pub/scm/bluetooth`, `devel`, `editors` | BlueZ, Entwicklerwerkzeuge wie `sparse` und `pahole`, Linus Torvalds' `uemacs` |

Die Liste ist sehr lang, weil allein unter `pub/scm/linux/kernel/git/` jeder Maintainer mit Konto eigene Bäume ablegen kann. Das Suchfeld oben rechts filtert die Liste nach Name, Beschreibung und Owner. Die Suche lässt sich auch direkt als URL aufrufen, z. B. `https://git.kernel.org/?q=xfs`.

## Das Pfadschema der Kernel-Bäume

Kernel-Repositories folgen dem Muster:

```text
https://git.kernel.org/pub/scm/linux/kernel/git/<konto>/<repository>.git/
```

`<konto>` ist der Benutzername des Maintainers auf kernel.org, nicht zwingend sein voller Name (`akpm` ist Andrew Morton, `acme` ist Arnaldo Carvalho de Melo). Für die meisten Fragen genügen drei Bäume:

| Baum | Pfad | Zweck |
|---|---|---|
| Mainline | `torvalds/linux.git` | Entwicklungszweig von Linus Torvalds; hier entstehen die Releases `v7.3`, `v7.4-rc1` usw. |
| Stable | `stable/linux.git` | gepflegte Versionen mit Fixes (`v7.3.1`, `v7.3.2` …) und die Longterm-Kernel; ein Branch pro Versionsreihe, z. B. `linux-7.3.y` |
| linux-next | `next/linux-next.git` | täglich zusammengeführter Stand aller Subsystem-Bäume für das nächste Merge-Window |

Ein Fix durchläuft diese Bäume in der Regel in dieser Reihenfolge: Er landet zuerst im Baum des zuständigen Subsystems, erscheint in linux-next, wird im Merge-Window per Pull-Request-Mail von Linus Torvalds in die Mainline übernommen und danach in die Stable-Zweige zurückportiert. Welcher Subsystem-Baum zuständig ist, steht in der Datei `MAINTAINERS` im Kernel-Quellcode (Zeilen mit `T: git …`).

## Die Ansichten eines Repositorys

Ein Klick auf einen Repository-Namen öffnet die Ansicht `summary`. Die Tab-Leiste darüber führt zu den weiteren Ansichten; links daneben wählt ein Auswahlfeld den Branch, auf den sich alle Ansichten beziehen.

<details class="options-details">
<summary>Ansichten im Überblick</summary>

| Ansicht | Inhalt |
|---|---|
| about | README des Repositorys, sofern hinterlegt |
| summary | Branches mit letztem Commit, die neuesten Tags mit Download-Link, die letzten zehn Commits und die Clone-URLs |
| refs | vollständige Liste aller Branches und Tags |
| log | Commit-Verlauf mit Suchfeld; Spalten Alter, Commit-Nachricht, Autor, geänderte Dateien und Zeilen |
| tree | Dateibaum zum gewählten Branch oder Commit, mit Links auf `plain` (Rohdatei) und `log` (Verlauf der Datei) |
| commit | Metadaten und Diff eines einzelnen Commits |
| diff | Unterschied zwischen zwei Commits oder zwischen einem Commit und seinem Vorgänger |
| stats | Commit-Statistik nach Autor über einen wählbaren Zeitraum |

</details>

In der `summary`-Ansicht ist der Tag-Block nützlich, um die aktuelle Version abzulesen: Die Mainline zeigt Release-Candidates (`v7.3-rc7`) und Releases (`v7.3`). Die Download-Links erzeugen ein Archiv auf Anfrage und sind dafür nicht immer verfügbar; für fertige Releases sind die Tarballs auf [kernel.org](https://kernel.org/) bzw. `cdn.kernel.org` der vorgesehene Bezugsweg, dort auch mit PGP-Signatur.

Am Ende der `summary`-Ansicht stehen die Clone-URLs. Neben `https://` und dem unverschlüsselten `git://` ist ein Spiegel bei Google (`kernel.googlesource.com`) angegeben, der denselben Inhalt liefert.

## Den Verlauf durchsuchen

Das Suchfeld in der `log`-Ansicht hat ein Auswahlfeld mit vier Modi. Jeder Modus ist auch als URL-Parameter verfügbar, sodass sich Suchen als Lesezeichen ablegen oder in Tickets verlinken lassen:

```text
https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?qt=grep&q=CVE
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Parameter | Wirkung |
|---|---|
| `qt=grep` | sucht in den Commit-Nachrichten (Auswahl „log msg") |
| `qt=author` | sucht im Autorenfeld (Name oder E-Mail-Adresse) |
| `qt=committer` | sucht im Committer-Feld, also bei der Person, die den Commit übernommen hat |
| `qt=range` | zeigt einen Bereich in Git-Notation, z. B. `q=v7.3-rc6..v7.3-rc7` für alle Commits zwischen zwei Tags |
| `q=` | Suchbegriff bzw. Bereich |
| `h=` | Branch, auf den sich die Suche bezieht, z. B. `h=linux-7.3.y` im Stable-Baum |
| `ofs=` | Versatz für die Seitenblätterung (git.kernel.org zeigt 200 Commits pro Seite) |

</details>

Der Bereich `v7.3-rc6..v7.3-rc7` beantwortet die Frage „Was hat sich zwischen zwei Versionen geändert?" direkt im Browser. Im Stable-Baum zeigt `qt=range&q=v7.3.1..v7.3.2&h=linux-7.3.y` alle Fixes, die mit einem Stable-Update hinzugekommen sind.

## Einen Commit lesen

Ein einzelner Commit ist über seinen vollständigen SHA-1-Hash adressierbar:

```text
https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=<sha>
```

Die Ansicht zeigt Autor, Committer, Datum, den übergeordneten Commit (`parent`), die vollständige Commit-Nachricht und den Diff. Am Ende einer Kernel-Commit-Nachricht stehen Trailer wie `Signed-off-by:`, `Reviewed-by:` und `Fixes: <sha> ("Titel")`. Die `Fixes:`-Zeile nennt den Commit, der den Fehler eingeführt hat; damit lässt sich eingrenzen, ab welcher Version ein System betroffen ist. `Link:` verweist meist auf die zugehörige Diskussion in lore.kernel.org, `Cc: stable@vger.kernel.org` markiert den Fix zum Rückportieren in die Stable-Kernel.

Ersetzt man `commit` in der URL durch `patch`, liefert der Server den Commit als Patch-Datei im Mail-Format (`text/plain`), die sich mit `git am` oder `patch` anwenden lässt.

## Kurz-URLs in CVE-Einträgen

Seit das Kernel-Projekt selbst CVE-Nummern vergibt, verweisen die Einträge in der CVE-Liste und der NVD auf Kurz-URLs der Form `https://git.kernel.org/stable/c/<sha>`. git.kernel.org leitet diese per HTTP 301 auf die Commit-Ansicht im passenden Baum weiter:

| Kurz-URL | Ziel |
|---|---|
| `https://git.kernel.org/stable/c/<sha>` | Commit im Stable-Baum `stable/linux.git` |
| `https://git.kernel.org/linus/<sha>` | Commit in der Mainline `torvalds/linux.git` |

Ein CVE-Eintrag nennt in der Regel mehrere solcher Links: einen pro Stable-Zweig, in den der Fix zurückportiert wurde, und einen für die Mainline. Die Hashes unterscheiden sich, weil jeder Rückport ein eigener Commit ist. Die Commit-Nachricht des Rückports nennt den Hash des Originals in der Mainline, entweder als Zeile `commit <sha> upstream.` oder als `[ Upstream commit <sha> ]`.

## Prüfen, ob ein Fix in einer Version enthalten ist

Die Weboberfläche zeigt nicht an, in welchen Tags ein Commit enthalten ist. Dafür ist ein lokaler Klon nötig. Für diesen Zweck reicht ein Klon ohne Dateiinhalte, der nur die Commit-Historie lädt:

```bash
git clone --filter=blob:none --no-checkout https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `clone` | legt eine lokale Kopie des Repositorys an |
| `--filter=blob:none` | Partial Clone: lädt Commits und Verzeichnisstrukturen, Dateiinhalte (Blobs) erst bei Bedarf |
| `--no-checkout` | legt kein Arbeitsverzeichnis an; für Abfragen der Historie nicht nötig |
| URL | HTTPS-Clone-URL aus der `summary`-Ansicht des Stable-Baums |

</details>

Anschliessend beantwortet ein Befehl im geklonten Verzeichnis die Frage nach der ersten Version mit dem Fix:

```bash
git describe --contains <sha>
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `describe` | beschreibt einen Commit relativ zu einem Tag |
| `--contains` | sucht den ältesten Tag, der den Commit enthält, statt des jüngsten Tags davor |
| `<sha>` | Hash des Commits, z. B. aus der Kurz-URL im CVE-Eintrag |

</details>

Die Ausgabe hat die Form `v7.3.2~14`: Der Fix ist ab `v7.3.2` enthalten. Alle Tags mit dem Commit listet `git tag --contains <sha>`. Wer nur eine einzelne Datei oder einen Stand zum Lesen braucht, kann statt eines Klons die Ansichten `tree` und `plain` im Browser verwenden.

Für den vollständigen Quellcode zum Bauen genügt ein flacher Klon eines Tags:

```bash
git clone --depth 1 --branch v7.3 https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `--depth 1` | lädt nur den letzten Commit ohne Historie; der Download schrumpft von mehreren GB auf einige hundert MB |
| `--branch v7.3` | wählt den Tag oder Branch, der ausgecheckt wird |
| URL | Clone-URL der Mainline |

</details>

## Änderungen verfolgen

Jedes Repository bietet einen Atom-Feed des Standard-Branches unter `…/<repository>.git/atom/`, für die Mainline also `https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/atom/`. Ein Feed-Reader zeigt damit neue Commits an, ohne dass die Seite regelmässig aufgerufen werden muss. Für die Mainline erzeugt der Feed während des Merge-Windows viele Einträge pro Tag; für Releases und Stable-Updates ist die Versionsübersicht auf [kernel.org](https://kernel.org/) bzw. deren Feed die ruhigere Quelle.

## Quellen

1.  [git.kernel.org](https://git.kernel.org/): Repository-Liste mit Gruppen, Owner- und Idle-Spalte; Suchfeld der Startseite.

2.  [torvalds/linux.git](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/): Mainline mit Ansichten summary, log, tree, Clone-URLs und Atom-Feed.

3.  [stable/linux.git](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/): Stable- und Longterm-Kernel, ein Branch pro Versionsreihe.

4.  [cgit](https://git.zx2c4.com/cgit/about/): Upstream der Weboberfläche, Dokumentation der URL-Parameter.

5.  [Kernel Docs: Everything you ever wanted to know about Linux -stable releases](https://docs.kernel.org/process/stable-kernel-rules.html): Regeln für Rückports, Trailer `Cc: stable` und Verweis auf den Upstream-Commit.

6.  [Kernel Docs: CVEs](https://docs.kernel.org/process/cve.html): CVE-Vergabe durch das Kernel-Projekt und Aufbau der Einträge.

7.  [Kernel Docs: Submitting patches](https://docs.kernel.org/process/submitting-patches.html): Bedeutung der Trailer `Fixes:`, `Link:`, `Signed-off-by:`.

8.  [git-describe](https://git-scm.com/docs/git-describe): Manpage, Option `--contains`.

9.  [git-clone](https://git-scm.com/docs/git-clone): Manpage, Optionen `--filter`, `--depth`, `--branch`, `--no-checkout`.

10.  [lore.kernel.org](https://lore.kernel.org/): Archiv der Kernel-Mailinglisten, auf das die `Link:`-Trailer verweisen.
