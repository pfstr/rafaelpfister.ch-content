---
title: "git.kernel.org explained: site structure and practical use"
navTitle: "git.kernel.org"
description: "git.kernel.org is the Git hosting service for Linux kernel development. This overview explains the repository list, path scheme, summary, log, tree, and commit views, URL search parameters, and the short URLs referenced by kernel CVEs."
date: "2026-10-29"
kategorie: "Linux kernel"
timeToRead: "9 min read"
themen:
  - linux-kernel
produkte:
  - "linux-kernel"
protokolle:
  - "releases"
  - "troubleshooting"
slug: "git-kernel-org-explained-site-structure-and-practical-use"
translationId: "article-91dab903d58c6173"
translationOf: git-kernel-org-erklaert
url: https://rafaelpfister.ch/en/blog/git-kernel-org-explained-site-structure-and-practical-use
translationSourceHash: 34a3d0d525d79d1ef4c5ff31824dabdd6e910a1dd5a22e60c77b202f97a71b7d
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T14:31:44.909Z
translationReview: automatic
---

[git.kernel.org](https://git.kernel.org/) is kernel.org's Git server. It hosts Linus Torvalds' main development branch, the stable kernels, linux-next, hundreds of subsystem and developer trees, as well as tools such as `git` itself, `e2fsprogs` or the Linux man pages. The web interface is a customized version of [cgit](https://git.zx2c4.com/cgit/about/) (identified in the footer as `cgit 1.3.1-korg`). It is read-only: there are no accounts, issues, or pull requests. Kernel development takes place through mailing lists, whose archive is at [lore.kernel.org](https://lore.kernel.org/); git.kernel.org shows the result: the commits that maintainers have incorporated into their trees.

Typical reasons to visit include a CVE entry that refers to a kernel commit, determining whether a particular fix is included in a kernel version, or reading an individual source file without cloning the entire repository.

## The home page: repository list

The home page lists all public repositories in a table. The grouping follows the file path on the server: each intermediate heading such as `pub/scm/fs` or `pub/scm/linux` is a directory, with the repositories listed below it using paths relative to that directory.

<details class="options-details">
<summary>Column overview</summary>

| Column | Content |
|---|---|
| Name | Repository path below the group, for example `xfs/xfs-linux.git` under `pub/scm/fs` |
| Description | Contents of the `description` file in the repository. “Unnamed repository; edit this file …” is Git's default text and only means that the operator has not set a description |
| Owner | Responsible person or group (for example, “XFS FS Group,” “Infra Team”) |
| Idle | Time since the last commit on any branch; a low value indicates an actively maintained tree |
| Links | Direct links to the repository's `summary`, `log`, and `tree` views |

</details>

The most important groups:

| Group | Content |
|---|---|
| `pub/scm/linux` | Kernel trees: mainline, stable, linux-next, and the personal and subsystem trees of maintainers |
| `pub/scm/fs` | File systems and user-space tools (ext2/3/4, XFS, fscrypt, fsverity) |
| `pub/scm/libs` | Libraries such as `libcap`, `libgpiod`, `libtraceevent` |
| `pub/scm/git` | The Git project itself (`git.git`) with HTML documentation and man pages |
| `pub/scm/docs` | Documentation sources, including `man-pages` and the PGP keys of kernel developers (`kernel/pgpkeys.git`) |
| `pub/scm/infra` | kernel.org infrastructure, such as the cgit fork and the Gitolite server's transparency logs |
| `pub/scm/bluetooth`, `devel`, `editors` | BlueZ, developer tools such as `sparse` and `pahole`, Linus Torvalds' `uemacs` |

The list is very long because every maintainer with an account can store their own trees under `pub/scm/linux/kernel/git/`. The search field in the upper right filters the list by name, description, and owner. The search can also be called directly as a URL, for example `https://git.kernel.org/?q=xfs`.

## The path scheme of kernel trees

Kernel repositories follow this pattern:

```text
https://git.kernel.org/pub/scm/linux/kernel/git/<konto>/<repository>.git/
```

`<konto>` is the maintainer's username on kernel.org, not necessarily their full name (`akpm` is Andrew Morton, `acme` is Arnaldo Carvalho de Melo). For most questions, three trees are sufficient:

| Tree | Path | Purpose |
|---|---|---|
| Mainline | `torvalds/linux.git` | Linus Torvalds' development branch; releases `v7.3`, `v7.4-rc1`, and so on are created here |
| Stable | `stable/linux.git` | Maintained versions with fixes (`v7.3.1`, `v7.3.2` …) and the long-term kernels; one branch per version series, for example `linux-7.3.y` |
| linux-next | `next/linux-next.git` | Daily merged state of all subsystem trees for the next merge window |

A fix generally moves through these trees in this order: it first lands in the responsible subsystem's tree, appears in linux-next, is incorporated into mainline during the merge window by Linus Torvalds via pull request email, and is then backported to the stable branches. The responsible subsystem tree is specified in the `MAINTAINERS` file in the kernel source code (lines with `T: git …`).

## Repository views

Clicking a repository name opens the `summary` view. The tab bar above it leads to the other views; a selection field to its left chooses the branch to which all views apply.

<details class="options-details">
<summary>View overview</summary>

| View | Content |
|---|---|
| about | Repository README, if available |
| summary | Branches with the latest commit, the newest tags with download links, the last ten commits, and clone URLs |
| refs | Complete list of all branches and tags |
| log | Commit history with search field; columns for age, commit message, author, changed files, and lines |
| tree | File tree for the selected branch or commit, with links to `plain` (raw file) and `log` (file history) |
| commit | Metadata and diff for a single commit |
| diff | Difference between two commits or between a commit and its parent |
| stats | Commit statistics by author over a selectable period |

</details>

In the `summary` view, the tag block is useful for reading the current version: mainline shows release candidates (`v7.3-rc7`) and releases (`v7.3`). The download links generate an archive on request and are therefore not always available; for completed releases, the tarballs at [kernel.org](https://kernel.org/) or `cdn.kernel.org` are the intended download source, including PGP signatures.

The clone URLs are at the bottom of the `summary` view. In addition to `https://` and the unencrypted `git://`, a mirror at Google (`kernel.googlesource.com`) is listed and provides the same content.

## Searching the history

The search field in the `log` view has a selection field with four modes. Each mode is also available as a URL parameter, allowing searches to be bookmarked or linked in tickets:

```text
https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?qt=grep&q=CVE
```

<details class="options-details">
<summary>Options explained</summary>

| Parameter | Effect |
|---|---|
| `qt=grep` | Searches commit messages (“log msg” selection) |
| `qt=author` | Searches the author field (name or email address) |
| `qt=committer` | Searches the committer field, meaning the person who incorporated the commit |
| `qt=range` | Shows a range in Git notation, for example `q=v7.3-rc6..v7.3-rc7` for all commits between two tags |
| `q=` | Search term or range |
| `h=` | Branch to which the search applies, for example `h=linux-7.3.y` in the stable tree |
| `ofs=` | Offset for pagination (git.kernel.org shows 200 commits per page) |

</details>

The range `v7.3-rc6..v7.3-rc7` directly answers the question, “What changed between two versions?” in the browser. In the stable tree, `qt=range&q=v7.3.1..v7.3.2&h=linux-7.3.y` shows all fixes added with a stable update.

## Reading a commit

An individual commit can be addressed through its full SHA-1 hash:

```text
https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=<sha>
```

The view shows the author, committer, date, parent commit (`parent`), complete commit message, and diff. At the end of a kernel commit message are trailers such as `Signed-off-by:`, `Reviewed-by:`, and `Fixes: <sha> ("Titel")`. The `Fixes:` line identifies the commit that introduced the bug; this can be used to determine from which version a system is affected. `Link:` usually refers to the related discussion on lore.kernel.org, while `Cc: stable@vger.kernel.org` marks the fix for backporting to stable kernels.

If `commit` in the URL is replaced with `patch`, the server returns the commit as a patch file in mail format (`text/plain`), which can be applied with `git am` or `patch`.

## Short URLs in CVE entries

Since the kernel project itself began assigning CVE numbers, entries in the CVE list and NVD refer to short URLs in the form `https://git.kernel.org/stable/c/<sha>`. git.kernel.org redirects these via HTTP 301 to the commit view in the appropriate tree:

| Short URL | Destination |
|---|---|
| `https://git.kernel.org/stable/c/<sha>` | Commit in the stable tree `stable/linux.git` |
| `https://git.kernel.org/linus/<sha>` | Commit in mainline `torvalds/linux.git` |

A CVE entry generally lists several such links: one for each stable branch to which the fix was backported, and one for mainline. The hashes differ because each backport is its own commit. The backport's commit message identifies the hash of the original in mainline, either as a `commit <sha> upstream.` line or as `[ Upstream commit <sha> ]`.

## Checking whether a fix is included in a version

The web interface does not show which tags contain a commit. A local clone is required for that. For this purpose, a clone without file contents that downloads only the commit history is sufficient:

```bash
git clone --filter=blob:none --no-checkout https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `clone` | Creates a local copy of the repository |
| `--filter=blob:none` | Partial clone: downloads commits and directory structures, with file contents (blobs) only as needed |
| `--no-checkout` | Does not create a working directory; not needed for history queries |
| URL | HTTPS clone URL from the `summary` view of the stable tree |

</details>

Then, a command in the cloned directory answers the question of the first version containing the fix:

```bash
git describe --contains <sha>
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `describe` | Describes a commit relative to a tag |
| `--contains` | Finds the oldest tag that contains the commit instead of the newest preceding tag |
| `<sha>` | Commit hash, for example from the short URL in the CVE entry |

</details>

The output takes the form `v7.3.2~14`: the fix is included starting with `v7.3.2`. `git tag --contains <sha>` lists all tags containing the commit. If only an individual file or a revision is needed for reading, the `tree` and `plain` views can be used in the browser instead of a clone.

For the full source code for building, a shallow clone of a tag is sufficient:

```bash
git clone --depth 1 --branch v7.3 https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `--depth 1` | Downloads only the latest commit without history; the download shrinks from several GB to a few hundred MB |
| `--branch v7.3` | Selects the tag or branch to check out |
| URL | Clone URL for mainline |

</details>

## Tracking changes

Every repository provides an Atom feed for the default branch at `…/<repository>.git/atom/`, and for mainline this is `https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/atom/`. A feed reader can therefore display new commits without requiring regular visits to the page. During the merge window, the mainline feed generates many entries per day; for releases and stable updates, the version overview at [kernel.org](https://kernel.org/) or its feed is the quieter source.

## Sources

1.  [git.kernel.org](https://git.kernel.org/): Repository list with groups, Owner and Idle columns; home page search field.

2.  [torvalds/linux.git](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/): Mainline with summary, log, tree views, clone URLs, and Atom feed.

3.  [stable/linux.git](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/): Stable and long-term kernels, one branch per version series.

4.  [cgit](https://git.zx2c4.com/cgit/about/): Upstream of the web interface, documentation for URL parameters.

5.  [Kernel Docs: Everything you ever wanted to know about Linux -stable releases](https://docs.kernel.org/process/stable-kernel-rules.html): Rules for backports, `Cc: stable` trailer, and reference to the upstream commit.

6.  [Kernel Docs: CVEs](https://docs.kernel.org/process/cve.html): CVE assignment by the kernel project and entry structure.

7.  [Kernel Docs: Submitting patches](https://docs.kernel.org/process/submitting-patches.html): Meaning of the `Fixes:`, `Link:`, and `Signed-off-by:` trailers.

8.  [git-describe](https://git-scm.com/docs/git-describe): Man page, `--contains` option.

9.  [git-clone](https://git-scm.com/docs/git-clone): Man page, `--filter`, `--depth`, `--branch`, and `--no-checkout` options.

10.  [lore.kernel.org](https://lore.kernel.org/): Archive of the kernel mailing lists referenced by the `Link:` trailers.
