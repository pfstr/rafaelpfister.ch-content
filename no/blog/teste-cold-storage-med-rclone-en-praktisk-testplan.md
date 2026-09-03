---
title: "Test Cold Storage with Rclone: a practical test plan"
navTitle: "Test Rclone"
description: "Before a service reads its files from the cloud via an Rclone mount, you should check more than just directory access. This test plan covers cold reads, warm reads, write operations, cache behaviour, file integrity and outages."
date: "2026-07-26"
kategorie: "Rclone"
timeToRead: "11 min read"
themen:
  - rclone
  - testing
related:
  - rclone-mount-in-docker-container
  - paperless-dokumente-clouddienst-auslagern
slug: "teste-cold-storage-med-rclone-en-praktisk-testplan"
translationOf: "cloud-mount-testen-dummy-pdfs"
translationId: article-8592f808b2e93cd4
translationModel: gpt-5.6-terra
translatedAt: 2026-09-03T08:23:39.236Z
translationReview: automatic
translationSourceHash: 27bc45a50d8e84fc785eaf04ec6814054e327f516587d0f9f9a101c989ce2aa1
url: https://rafaelpfister.ch/no/blog/teste-cold-storage-med-rclone-en-praktisk-testplan
---

An Rclone mount is quick to set up. The remote appears as a directory, `ls` shows files, and the first functional test has passed. That says little about production use.

As soon as a service accesses the mount, further questions arise: How long does the first access to a file take? Which accesses are served by the local cache? What happens to a file that has not yet been uploaded if Rclone crashes? Does a running container see the remounted mount again? And how does the service react if the cloud is temporarily unavailable?

This article provides a general test plan for this purpose. You can use it for a document archive, media server, photo management system or any other service that retrieves rarely needed files from Cold Storage via Rclone.

## The most important rclone options

For orientation, here are the Rclone options used in this test plan, translated broadly from the documentation:

<details class="options-details">
<summary>Options at a glance</summary>

| Option | Meaning |
|---|---|
| `--seed n` | Initial value for the random number generator in `rclone test makefiles`; the same seed produces the same file tree |
| `--files n` | Number of test files to create |
| `--files-per-directory n` | Average number of files per directory |
| `--min-file-size grösse` | Smallest generated file size (suffixes such as K, M and G are allowed) |
| `--max-file-size grösse` | Largest generated file size |
| `--progress` | Continuous progress display during transfer |
| `--stats dauer` | Interval at which transfer statistics are printed |
| `--log-file datei` | Writes the log to the specified file |
| `--log-level stufe` | Log detail level: DEBUG, INFO, NOTICE or ERROR |
| `--one-way` | With `rclone check`, checks only whether all source files exist in the destination and are identical; additional files in the destination are not considered errors |
| `--download` | Actually downloads the data during comparison instead of comparing hashes only |
| `--vfs-cache-mode modus` | Cache strategy of the VFS layer; `full` buffers reads and writes locally |
| `--cache-dir verzeichnis` | Directory for the local cache |
| `--vfs-cache-max-size grösse` | Soft limit for the total VFS cache size |
| `--vfs-cache-poll-interval dauer` | Interval at which Rclone checks and cleans up the cache |
| `--vfs-write-back dauer` | Delay after closing a file before upload to the remote begins |
| `--vfs-read-ahead grösse` | Additional amount of data read ahead beyond the requested position with `full` |
| `--poll-interval dauer` | Interval at which Rclone polls the remote for changes (only for backends with polling support) |
| `--dir-cache-time dauer` | Validity period for cached directory listings |
| `--allow-other` | Allows users other than the mounting user to access the FUSE mount |

</details>

The complete lists are available in the Rclone documentation, especially under [rclone mount](https://rclone.org/commands/rclone_mount/) and in the overview of [global flags](https://rclone.org/flags/).

## First decide what you want to achieve

Cold Storage does not automatically mean the same thing for every application. A media server usually reads large files sequentially. A photo management system loads many small preview files and jumps to different positions. A document archive opens comparatively small files, but often only once.

Before testing, note the most important characteristics of your real dataset:

- typical file size and largest file encountered
- number of files per directory
- complete reading or random access to individual regions
- ratio between reads and writes
- number of simultaneous users or processes
- changes that take place directly in the remote outside the mount
- acceptable wait time for a cold read
- maximum available space for the local cache

Only then do meaningful success criteria emerge. Opening a single file in 1.2 seconds may be perfectly acceptable for an archive and unusable for an interactive application.

## Create a reproducible test dataset

Rclone already includes a suitable tool. `rclone test makefiles` creates the same file tree every time with a fixed seed:

```bash
rclone test makefiles ./testdata \
  --seed 42 \
  --files 250 \
  --files-per-directory 25 \
  --min-file-size 16K \
  --max-file-size 32M
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `./testdata` | Destination directory where the test tree is created |
| `--seed 42` | Fixed initial value for the random number generator; every run creates the same dataset |
| `--files 250` | 250 test files in total |
| `--files-per-directory 25` | An average of 25 files per directory |
| `--min-file-size 16K` | Smallest file: 16 KiB |
| `--max-file-size 32M` | Largest file: 32 MiB |

</details>

Adapt the number and sizes to your real dataset. Do not test only average files. A few very small files show how expensive metadata accesses are; a few large files make throughput, read-ahead and cache behaviour visible.

Also add file names that may cause problems in practice:

```bash
mkdir -p "testdata/Sonderfälle/Unterordner"
printf 'Leerzeichen\n' > "testdata/Sonderfälle/Datei mit Leerzeichen.txt"
printf 'Umlaute\n' > "testdata/Sonderfälle/Grösse und Änderung.txt"
printf 'Grossschreibung\n' > "testdata/Sonderfälle/Test.txt"
printf 'Kleinschreibung\n' > "testdata/Sonderfälle/test.txt"
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `mkdir -p` | Also creates missing parent directories and does not report an error if the directory exists |
| `printf '…\n' > datei` | Writes the specified text as file content; the redirection creates the file with the problematic name |

</details>

The last test is particularly important if the local file system and cloud backend handle upper and lower case differently.

If your service accepts only certain formats, arbitrary binary files are not sufficient. In that case, also create synthetic files in precisely those formats. With Paperless-ngx, these were PDFs with a real text layer so that the test would not accidentally measure OCR performance rather than the storage path. For a photo management system, include different image sizes and formats; for a media server, short files with different codecs.

## A baseline measurement without a mount

Before FUSE and the VFS cache come into play, you should measure the backend directly. Copy the dataset to the test remote with Rclone and save a detailed log:

```bash
rclone copy ./testdata remote:cold-storage-test \
  --progress \
  --stats 5s \
  --log-file rclone-copy.log \
  --log-level INFO
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `./testdata` | Copy source: the locally generated test dataset |
| `remote:cold-storage-test` | Destination: path in the configured remote |
| `--progress` | Continuous progress display in the terminal |
| `--stats 5s` | Transfer statistics every 5 seconds instead of at the default interval |
| `--log-file rclone-copy.log` | Complete log in a file for later analysis |
| `--log-level INFO` | Logs transfers, retries and errors without the volume of DEBUG logging |

</details>

Then check whether source and destination match:

```bash
rclone check ./testdata remote:cold-storage-test \
  --one-way \
  --download
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `./testdata` | Reference: the local original dataset |
| `remote:cold-storage-test` | Subject under test: the freshly uploaded dataset in the remote |
| `--one-way` | Checks only whether all source files exist in the destination and are identical; additional files in the destination are not flagged |
| `--download` | Downloads the data and compares contents instead of relying on hashes |

</details>

`--download` is crucial here because some backends do not provide suitable hashes. The comparison takes longer, but provides a useful baseline for the later integrity test.

Record upload time, transfer rate, number of retries and API errors. If direct access is already unstable, the mount cannot fix it.

## Separate the test mount from the production cache

Use a dedicated mount point and a dedicated cache directory for measurement:

```bash
rclone mount remote:cold-storage-test /mnt/rclone-test \
  --vfs-cache-mode full \
  --cache-dir /var/cache/rclone-test \
  --vfs-cache-max-size 10G \
  --vfs-cache-poll-interval 1m \
  --allow-other \
  --log-file /var/log/rclone-test.log \
  --log-level INFO
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `remote:cold-storage-test` | Remote and path to mount |
| `/mnt/rclone-test` | Mount point on the test system |
| `--vfs-cache-mode full` | Fully buffers reads and writes in the local cache |
| `--cache-dir /var/cache/rclone-test` | Dedicated cache directory, separate from the production cache |
| `--vfs-cache-max-size 10G` | Soft limit of 10 GiB for the VFS cache |
| `--vfs-cache-poll-interval 1m` | Cache checking and cleanup every minute |
| `--allow-other` | Users other than the mounting user may also access it; required for services and containers |
| `--log-file /var/log/rclone-test.log` | Log to a file to trace access during tests |
| `--log-level INFO` | Medium log detail level |

</details>

The values are an example, not a general recommendation. Separation is what matters: an empty test cache makes cold reads reproducible without requiring you to delete files from an active production cache.

`--vfs-cache-mode full` is usually the most informative test mode for applications. Rclone buffers reads and writes locally and can better emulate file accesses that would not be possible with pure object storage. The additional compatibility costs local storage space.

## Always check from the perspective of the real service

A mount may work for your user but still be unusable for the service. Common causes include a different user ID, missing `--allow-other`, container boundaries or incorrect mount propagation.

Therefore, perform at least one complete read with the same identity under which the application will later run:

```bash
sudo -u <service-user> sha256sum /mnt/rclone-test/pfad/zur/datei
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-u <service-user>` | Runs the command as the specified user, not as root |
| `/mnt/rclone-test/pfad/zur/datei` | File to read; `sha256sum` forces a complete read |

</details>

If the service runs in Docker, the test belongs inside the container:

```bash
docker exec --user <uid>:<gid> <app-container> \
  sha256sum /pfad/im/container/datei
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `--user <uid>:<gid>` | Runs the command in the container with this user and group ID, regardless of the image's default user |
| `<app-container>` | Name or ID of the running application container |
| `sha256sum /pfad/im/container/datei` | Command to run; the path is the mount from the container's perspective |

</details>

An actual application test is even better. Open the file through the service's web interface or API. Only this reveals whether the application, for example, starts several parallel reads, jumps to the end of the file or expects additional metadata.

## Measure cold reads and warm reads separately

With `--vfs-cache-mode full`, there are three layers between the application and the cloud:

| Layer | What it contains |
|---|---|
| Remote | the complete file in the cloud service |
| VFS cache | locally stored regions of files that have already been read |
| Linux page cache | recently used data in RAM |

For a cold read, choose a file whose contents have never been read through the test mount. In the immediately following warm read, it is in the VFS cache and usually also in RAM.

```bash
measure_read() {
  file="$1"
  label="$2"
  start=$(date +%s%3N)
  cat "$file" > /dev/null
  end=$(date +%s%3N)
  printf '%s: %s ms\n' "$label" "$((end - start))"
}

measure_read "/mnt/rclone-test/grosse-datei.bin" "Cold Read"
measure_read "/mnt/rclone-test/grosse-datei.bin" "Warm Read"
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `date +%s%3N` | Timestamp in milliseconds: Unix seconds immediately followed by the first three digits of the nanosecond component (GNU date) |
| `cat "$file" > /dev/null` | Reads the file completely and discards output; only read time is measured |
| `"$1"`, `"$2"` | Shell function arguments: file path and label for the measurement line |

</details>

Do not measure only one file. Use at least ten previously unread files of different sizes and record the median, slowest value and file size. A single best result is not a basis for a decision.

A warm read is not a pure disk test because the kernel may keep parts of the file in RAM. For most Cold Storage scenarios, that is not a problem. What matters is what a user experiences when opening a file for the first time and again later. If you want to evaluate RAM and local disk separately, you must additionally control and demonstrably clear the page cache.

## Test more than complete reads

`cat` reads a file from beginning to end. Many applications behave differently:

- A video player initially reads headers and the index, later jumps to another position and then continues reading sequentially.
- An image management system reads metadata and then generates a preview image.
- An archive program may read the end of the file first.
- Several workers may access different files at the same time.

Test these workflows with the actual application. Observe the Rclone log and cache in parallel. With large files, it is interesting how much Rclone actually stores locally and whether `--vfs-read-ahead` matches the access pattern.

An Rclone mount is also not a sensible storage location for databases or other files that require reliable locking and frequent changes within the same file. The VFS layer compensates for differences between file systems and object storage, but does not turn the backend into a local file system.

## Validate the write path separately

If your service only reads, mount the remote read-only whenever possible. If it must write, test creating, overwriting, renaming and deleting individually.

A written file does not necessarily appear in the remote immediately. With the VFS cache enabled, upload starts only after the file has been closed and `--vfs-write-back` has elapsed. Therefore check both states:

1. The application has successfully closed the file.
2. The file can subsequently be read in the remote through direct Rclone access.

```bash
printf 'writeback-test\n' > /mnt/rclone-test/writeback-test.txt

# After --vfs-write-back has elapsed:
rclone cat remote:cold-storage-test/writeback-test.txt
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `/mnt/rclone-test/writeback-test.txt` | Destination file in the mount; redirection writes via the VFS cache |
| `remote:cold-storage-test/writeback-test.txt` | Direct access bypassing the mount: `rclone cat` reads the file from the remote and writes it to stdout |

</details>

Repeat the test with a large file and stop Rclone while the upload is still running. Then restart it using the same cache directory and check whether the upload resumes. This exact time window determines how much data is at risk during a server failure.

Also test renaming and deleting. Many cloud backends map these operations differently from a local file system. What matters is not only whether the command finishes successfully, but when the change becomes visible through direct access to the remote and to other clients.

## Test changes outside the mount

Files can be changed through the provider's web interface, a second Rclone process or another server. The mount does not always see such changes immediately because directory information is cached.

Therefore, create a file directly in the remote with a second Rclone invocation:

```bash
printf 'external-change\n' > external-change.txt
rclone copyto external-change.txt \
  remote:cold-storage-test/external-change.txt
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `external-change.txt` | Source: the locally created file |
| `remote:cold-storage-test/external-change.txt` | Destination with the exact file name; `copyto` copies a single file under that exact name, rather than copying it into a directory like `copy` |

</details>

Measure when the file appears in the mount. Repeat the test for modification and deletion. The result depends on the backend, its polling support, and `--poll-interval` and `--dir-cache-time`. If the application must see current changes immediately, this behaviour must explicitly be included in the acceptance criteria.

With the Remote Control interface enabled, you can specifically clear the directory cache:

```bash
rclone rc vfs/forget
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `vfs/forget` | Remote Control command to execute: clears the VFS cached directory tree so that the next access queries the remote again |

</details>

This is useful for a manual test, but is no substitute for an appropriate operating strategy.

## Put the cache under pressure

A nearly empty cache is the simplest case. In a second test round, deliberately set `--vfs-cache-max-size` low and read more data than it can hold.

```bash
du -sh /var/cache/rclone-test/vfs
du -sh --apparent-size /var/cache/rclone-test/vfs
find /var/cache/rclone-test/vfs -type f | wc -l
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `du -s` | Summarises disk usage in one total line instead of listing every subdirectory |
| `du -h` | Output in human-readable units (K, M, G) |
| `du --apparent-size` | Shows logical file size instead of actual occupied disk space |
| `find … -type f` | Includes only regular files, not directories |
| `wc -l` | Counts output lines, here the number of cache files |

</details>

The two sizes can differ greatly. In full mode, Rclone uses sparse files: a file shows its complete logical size even though only the read regions occupy local space.

The cache limit is also soft. Rclone checks it at the interval set by `--vfs-cache-poll-interval`, and open files cannot be removed. The cache may therefore temporarily exceed the limit. However, it should decrease again after the files are closed and the next cleanup run takes place.

Record the peak value, the value after cleanup and the time required. This makes it possible to size the required local storage sensibly.

## Simulate two different failures

An unreachable cloud and a crashed Rclone process are two different errors:

| Failure | What this tests |
|---|---|
| Backend or network unavailable, Rclone continues running | Behaviour during retries, timeouts and for already cached files |
| Rclone process terminated | Behaviour of the FUSE mount and restoration of the mount point |

Simulate both only in the test environment. You can forcefully terminate an Rclone container for the second case:

```bash
docker kill --signal KILL <rclone-container>
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `--signal KILL` | Sends SIGKILL instead of the default SIGTERM signal; the process has no opportunity for cleanup |
| `<rclone-container>` | Name or ID of the Rclone container |

</details>

During the failure, check the application rather than only the mount point:

- Which functions remain available?
- How long does an access wait before an error appears?
- Are already fully cached files still accessible?
- Does the application stop new write operations?
- Is there an understandable error message or merely a hanging process?
- Does your monitoring trigger?

When the mount is missing, a writing service must not silently write to the underlying local directory. These files would be obscured after the mount returns. A simple safeguard before every write job is:

```bash
mountpoint -q /mnt/rclone-test || exit 1
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-q` | No output; the result is available only in the exit code |
| `/mnt/rclone-test` | Path to check; exit code 0 only if a mount is actually active there |
| `\|\| exit 1` | Stops the script if the path is not a mount point |

</details>

After restarting Rclone, check the mount on the host and from every consuming container. A remounted mount reaches an already running container only with appropriate mount propagation. For Docker, `rslave` is usually required on the consuming side. Details are in the article [Run Rclone mounts reliably in Docker](/blog/rclone-mount-in-docker-container).

## A concrete example from Paperless-ngx

For my Paperless test, I created 40 PDFs totalling 13.9 MB. A previously unopened document took around 1.8 seconds, while an immediately repeated access took 19 to 24 milliseconds. A VFS cache limited to 4 MB briefly grew to 12.7 MiB and was cleaned up again during the next run.

While the remote was unavailable, the document list, full-text search and preview images continued to work because that data was stored locally. Only the original could not be opened. After remounting, the running Paperless container could access the files again without having to be restarted itself.

These figures are not a benchmark for Rclone or Proton Drive. The behaviour is what matters: Hot Storage remained available locally, cold reads were slower but predictable, and the service recovered after the failure.

## What belongs in the test log

A result that can be understood later includes at least:

- Rclone version and backend used
- operating system, FUSE variant and file system of the cache directory
- complete mount command without credentials
- number, size distribution and structure of the test files
- cold-read and warm-read values for several files
- write duration until visibility in the remote
- cache peak value and cleanup duration
- result of `rclone check --download`
- behaviour during backend failure and Rclone process termination
- recovery time from the application's perspective
- retries, timeouts, throttling and authentication errors from the log

Define a threshold for every item in advance. Then the test ends with a decision rather than just a collection of interesting figures.

## When the setup is ready

A Cold Storage mount is ready for use when you can answer these questions with yes:

- Are cold reads fast enough for the intended service?
- Does the cache speed up repeated accesses as expected?
- Does local storage use remain manageable even under load?
- Do all files match after a complete download?
- Do all required file operations work with the selected backend?
- Does the application behave in a controlled manner during a cloud outage?
- Are write operations safely stopped when the mount is missing?
- Does a remounted mount reach all running consumers?
- Does monitoring show the failure before a user reports it?

If an answer is missing, you at least know exactly what you need to continue working on. That is much more helpful than a mount that looked good at the first `ls` and only reveals its limits in operation.

## Kilder

1.  [Rclone test makefiles](https://rclone.org/commands/rclone_test_makefiles/): reproducible test files and directory structures with configurable sizes.

2.  [Rclone mount](https://rclone.org/commands/rclone_mount/): VFS cache modes, writeback, sparse files, cache limits and directory cache.

3.  [Rclone check](https://rclone.org/commands/rclone_check/): comparison of source and destination, including complete verification with `--download`.

4.  [Rclone Remote Control](https://rclone.org/rc/): targeted clearing of the VFS directory cache with `vfs/forget`.

5.  [Rclone Global Flags](https://rclone.org/flags/): complete reference for global options, including logging, statistics and VFS parameters.
