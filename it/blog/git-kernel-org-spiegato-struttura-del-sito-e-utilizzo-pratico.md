---
title: "git.kernel.org spiegato: struttura del sito e utilizzo pratico"
navTitle: "git.kernel.org"
description: "git.kernel.org è l'hosting Git dello sviluppo del kernel Linux. Questa panoramica spiega l'elenco dei repository, lo schema dei percorsi, le viste summary, log, tree e commit, i parametri di ricerca nell'URL e gli URL brevi a cui fanno riferimento le CVE del kernel."
date: "2026-10-29"
kategorie: "Kernel Linux"
timeToRead: "9 min di lettura"
themen:
  - linux-kernel
produkte:
  - "linux-kernel"
protokolle:
  - "releases"
  - "troubleshooting"
slug: "git-kernel-org-spiegato-struttura-del-sito-e-utilizzo-pratico"
translationId: "article-91dab903d58c6173"
translationOf: git-kernel-org-erklaert
url: https://rafaelpfister.ch/it/blog/git-kernel-org-spiegato-struttura-del-sito-e-utilizzo-pratico
translationSourceHash: 34a3d0d525d79d1ef4c5ff31824dabdd6e910a1dd5a22e60c77b202f97a71b7d
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T14:33:04.580Z
translationReview: automatic
---

[git.kernel.org](https://git.kernel.org/) è il server Git di kernel.org. Qui si trovano il ramo di sviluppo principale di Linus Torvalds, i kernel Stable, linux-next, centinaia di alberi di sottosistemi e sviluppatori, nonché strumenti come `git` stesso, `e2fsprogs` o le pagine man di Linux. L'interfaccia web è una versione adattata di [cgit](https://git.zx2c4.com/cgit/about/) (indicata nel piè di pagina come `cgit 1.3.1-korg`). È in sola lettura: non esistono account, issue né pull request. Lo sviluppo del kernel avviene tramite mailing list, il cui archivio si trova su [lore.kernel.org](https://lore.kernel.org/); git.kernel.org mostra il risultato, ovvero i commit che i maintainer hanno integrato nei loro alberi.

I motivi tipici per visitarlo sono una voce CVE che rimanda a un commit del kernel, la verifica che una determinata correzione sia inclusa in una versione del kernel oppure la lettura di un singolo file sorgente senza clonare l'intero repository.

## La pagina iniziale: elenco dei repository

La pagina iniziale elenca tutti i repository pubblici in una tabella. Il raggruppamento segue il percorso dei file sul server: ogni sottotitolo come `pub/scm/fs` o `pub/scm/linux` è una directory; sotto di esso sono elencati i repository con il relativo percorso rispetto a tale directory.

<details class="options-details">
<summary>Panoramica delle colonne</summary>

| Colonna | Contenuto |
|---|---|
| Name | Percorso del repository sotto il gruppo, ad es. `xfs/xfs-linux.git` sotto `pub/scm/fs` |
| Description | Contenuto del file `description` nel repository. «Unnamed repository; edit this file …» è il testo predefinito di Git e significa soltanto che il gestore non ha impostato una descrizione |
| Owner | persona o gruppo responsabile, ad es. «XFS FS Group», «Infra Team» |
| Idle | Tempo trascorso dall'ultimo commit su un qualsiasi branch; un valore basso indica un albero mantenuto attivamente |
| Links | Collegamenti diretti alle viste `summary`, `log` e `tree` del repository |

</details>

I gruppi più importanti:

| Gruppo | Contenuto |
|---|---|
| `pub/scm/linux` | Alberi del kernel: mainline, Stable, linux-next nonché gli alberi personali e dei sottosistemi dei maintainer |
| `pub/scm/fs` | File system e strumenti userspace (ext2/3/4, XFS, fscrypt, fsverity) |
| `pub/scm/libs` | Librerie come `libcap`, `libgpiod`, `libtraceevent` |
| `pub/scm/git` | Il progetto Git stesso (`git.git`) con documentazione HTML e pagine man |
| `pub/scm/docs` | Fonti della documentazione, tra cui `man-pages` e le chiavi PGP degli sviluppatori del kernel (`kernel/pgpkeys.git`) |
| `pub/scm/infra` | Infrastruttura di kernel.org, ad es. il fork di cgit e i log di trasparenza del server Gitolite |
| `pub/scm/bluetooth`, `devel`, `editors` | BlueZ, strumenti per sviluppatori come `sparse` e `pahole`, `uemacs` di Linus Torvalds |

L'elenco è molto lungo perché solo sotto `pub/scm/linux/kernel/git/` ogni maintainer con un account può archiviare i propri alberi. Il campo di ricerca in alto a destra filtra l'elenco per nome, descrizione e owner. La ricerca può essere richiamata anche direttamente come URL, ad es. `https://git.kernel.org/?q=xfs`.

## Lo schema dei percorsi degli alberi del kernel

I repository del kernel seguono il modello:

```text
https://git.kernel.org/pub/scm/linux/kernel/git/<konto>/<repository>.git/
```

`<konto>` è il nome utente del maintainer su kernel.org, non necessariamente il suo nome completo (`akpm` è Andrew Morton, `acme` è Arnaldo Carvalho de Melo). Per la maggior parte delle domande sono sufficienti tre alberi:

| Albero | Percorso | Scopo |
|---|---|---|
| Mainline | `torvalds/linux.git` | Ramo di sviluppo di Linus Torvalds; qui nascono le release `v7.3`, `v7.4-rc1` ecc. |
| Stable | `stable/linux.git` | Versioni mantenute con correzioni (`v7.3.1`, `v7.3.2` …) e kernel Longterm; un branch per ogni serie di versioni, ad es. `linux-7.3.y` |
| linux-next | `next/linux-next.git` | Stato integrato quotidianamente di tutti gli alberi dei sottosistemi per la prossima merge window |

In genere una correzione attraversa questi alberi in quest'ordine: arriva prima nell'albero del sottosistema responsabile, appare in linux-next, viene integrata nella mainline da Linus Torvalds durante la merge window tramite e-mail di pull request e successivamente viene sottoposta a backport nei rami Stable. L'albero del sottosistema responsabile è indicato nel file `MAINTAINERS` nel codice sorgente del kernel (righe con `T: git …`).

## Le viste di un repository

Un clic sul nome di un repository apre la vista `summary`. La barra delle schede sopra porta alle altre viste; a sinistra un menu a tendina seleziona il branch a cui si riferiscono tutte le viste.

<details class="options-details">
<summary>Panoramica delle viste</summary>

| Vista | Contenuto |
|---|---|
| about | README del repository, se presente |
| summary | Branch con l'ultimo commit, tag più recenti con link per il download, gli ultimi dieci commit e gli URL di clonazione |
| refs | Elenco completo di tutti i branch e tag |
| log | Cronologia dei commit con campo di ricerca; colonne età, messaggio di commit, autore, file e righe modificati |
| tree | Albero dei file del branch o commit selezionato, con link a `plain` (file grezzo) e `log` (cronologia del file) |
| commit | Metadati e diff di un singolo commit |
| diff | Differenza tra due commit oppure tra un commit e il suo predecessore |
| stats | Statistiche dei commit per autore in un periodo selezionabile |

</details>

Nella vista `summary`, il blocco dei tag è utile per leggere la versione attuale: la mainline mostra release candidate (`v7.3-rc7`) e release (`v7.3`). I link di download generano un archivio su richiesta e quindi non sono sempre disponibili; per le release definitive, i tarball su [kernel.org](https://kernel.org/) oppure `cdn.kernel.org` sono il canale previsto per il download, anche con firma PGP.

Alla fine della vista `summary` si trovano gli URL di clonazione. Oltre a `https://` e al non cifrato `git://`, viene indicato un mirror su Google (`kernel.googlesource.com`) che fornisce lo stesso contenuto.

## Cercare nella cronologia

Il campo di ricerca nella vista `log` dispone di un menu a tendina con quattro modalità. Ogni modalità è disponibile anche come parametro URL, così le ricerche possono essere salvate nei segnalibri o collegate nei ticket:

```text
https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?qt=grep&q=CVE
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Parametro | Effetto |
|---|---|
| `qt=grep` | Cerca nei messaggi di commit (selezione «log msg») |
| `qt=author` | Cerca nel campo autore (nome o indirizzo e-mail) |
| `qt=committer` | Cerca nel campo committer, quindi nella persona che ha integrato il commit |
| `qt=range` | Mostra un intervallo nella notazione Git, ad es. `q=v7.3-rc6..v7.3-rc7` per tutti i commit tra due tag |
| `q=` | Termine di ricerca o intervallo |
| `h=` | Branch a cui si riferisce la ricerca, ad es. `h=linux-7.3.y` nell'albero Stable |
| `ofs=` | Offset per la paginazione (git.kernel.org mostra 200 commit per pagina) |

</details>

L'intervallo `v7.3-rc6..v7.3-rc7` risponde direttamente nel browser alla domanda «Che cosa è cambiato tra due versioni?». Nell'albero Stable, `qt=range&q=v7.3.1..v7.3.2&h=linux-7.3.y` mostra tutte le correzioni aggiunte con un aggiornamento Stable.

## Leggere un commit

Un singolo commit è indirizzabile tramite il suo hash SHA-1 completo:

```text
https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=<sha>
```

La vista mostra autore, committer, data, il commit padre (`parent`), il messaggio di commit completo e il diff. Alla fine di un messaggio di commit del kernel sono presenti trailer come `Signed-off-by:`, `Reviewed-by:` e `Fixes: <sha> ("Titel")`. La riga `Fixes:` indica il commit che ha introdotto il difetto; consente quindi di restringere l'intervallo per stabilire da quale versione un sistema è interessato. `Link:` rimanda di norma alla discussione corrispondente su lore.kernel.org, mentre `Cc: stable@vger.kernel.org` contrassegna la correzione per il backport nei kernel Stable.

Sostituendo `commit` nell'URL con `patch`, il server fornisce il commit come file patch in formato e-mail (`text/plain`), applicabile con `git am` oppure `patch`.

## URL brevi nelle voci CVE

Da quando il progetto kernel assegna direttamente i numeri CVE, le voci nell'elenco CVE e nell'NVD rimandano a URL brevi del formato `https://git.kernel.org/stable/c/<sha>`. git.kernel.org li reindirizza con HTTP 301 alla vista del commit nell'albero appropriato:

| URL breve | Destinazione |
|---|---|
| `https://git.kernel.org/stable/c/<sha>` | Commit nell'albero Stable `stable/linux.git` |
| `https://git.kernel.org/linus/<sha>` | Commit nella mainline `torvalds/linux.git` |

Una voce CVE indica di norma più link di questo tipo: uno per ogni ramo Stable in cui la correzione è stata sottoposta a backport e uno per la mainline. Gli hash differiscono perché ogni backport è un commit distinto. Il messaggio di commit del backport indica l'hash dell'originale nella mainline, come riga `commit <sha> upstream.` oppure come `[ Upstream commit <sha> ]`.

## Verificare se una correzione è inclusa in una versione

L'interfaccia web non mostra in quali tag è contenuto un commit. A questo scopo è necessario un clone locale. Per questo utilizzo basta un clone senza contenuti dei file, che scarica soltanto la cronologia dei commit:

```bash
git clone --filter=blob:none --no-checkout https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `clone` | Crea una copia locale del repository |
| `--filter=blob:none` | Clone parziale: scarica commit e strutture delle directory, i contenuti dei file (blob) solo quando necessario |
| `--no-checkout` | Non crea una directory di lavoro; non è necessaria per interrogare la cronologia |
| URL | URL di clonazione HTTPS dalla vista `summary` dell'albero Stable |

</details>

Successivamente, un comando nella directory clonata risponde alla domanda sulla prima versione contenente la correzione:

```bash
git describe --contains <sha>
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `describe` | Descrive un commit rispetto a un tag |
| `--contains` | Cerca il tag più vecchio che contiene il commit, invece del tag più recente precedente |
| `<sha>` | Hash del commit, ad es. dall'URL breve nella voce CVE |

</details>

L'output ha la forma `v7.3.2~14`: la correzione è inclusa a partire da `v7.3.2`. Tutti i tag contenenti il commit sono elencati da `git tag --contains <sha>`. Chi ha bisogno soltanto di un singolo file o di uno stato da leggere può usare nel browser le viste `tree` e `plain` invece di un clone.

Per il codice sorgente completo da compilare è sufficiente un clone superficiale di un tag:

```bash
git clone --depth 1 --branch v7.3 https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `--depth 1` | Scarica soltanto l'ultimo commit senza cronologia; il download si riduce da diversi GB a qualche centinaio di MB |
| `--branch v7.3` | Seleziona il tag o il branch da estrarre |
| URL | URL di clonazione della mainline |

</details>

## Monitorare le modifiche

Ogni repository offre un feed Atom del branch predefinito su `…/<repository>.git/atom/`, per la mainline quindi `https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/atom/`. Un lettore di feed mostra così i nuovi commit senza dover visitare regolarmente la pagina. Per la mainline, il feed genera molte voci al giorno durante la merge window; per release e aggiornamenti Stable, la panoramica delle versioni su [kernel.org](https://kernel.org/) oppure il suo feed è la fonte più tranquilla.

## Fonti

1.  [git.kernel.org](https://git.kernel.org/): Elenco dei repository con gruppi, colonne Owner e Idle; campo di ricerca della pagina iniziale.

2.  [torvalds/linux.git](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/): Mainline con viste summary, log, tree, URL di clonazione e feed Atom.

3.  [stable/linux.git](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/): Kernel Stable e Longterm, un branch per ogni serie di versioni.

4.  [cgit](https://git.zx2c4.com/cgit/about/): Upstream dell'interfaccia web, documentazione dei parametri URL.

5.  [Kernel Docs: Everything you ever wanted to know about Linux -stable releases](https://docs.kernel.org/process/stable-kernel-rules.html): Regole per i backport, trailer `Cc: stable` e riferimento al commit upstream.

6.  [Kernel Docs: CVEs](https://docs.kernel.org/process/cve.html): Assegnazione delle CVE da parte del progetto kernel e struttura delle voci.

7.  [Kernel Docs: Submitting patches](https://docs.kernel.org/process/submitting-patches.html): Significato dei trailer `Fixes:`, `Link:`, `Signed-off-by:`.

8.  [git-describe](https://git-scm.com/docs/git-describe): Pagina man, opzione `--contains`.

9.  [git-clone](https://git-scm.com/docs/git-clone): Pagina man, opzioni `--filter`, `--depth`, `--branch`, `--no-checkout`.

10.  [lore.kernel.org](https://lore.kernel.org/): Archivio delle mailing list del kernel, a cui rimandano i trailer `Link:`.
